---
title: "Designing a Highly Available EKS Cluster: Why the Disk Is the Hard Part"
description: "What multi-AZ actually means on EKS, drawn out one failure domain at a time: the control plane AWS already spreads for you, node groups per AZ and why the Cluster Autoscaler needs them once pods use EBS, topology spread constraints and PodDisruptionBudgets, and the storage questions I got stuck on. Why an EBS volume survives its node but not its AZ, why deleting a pod does not delete the volume, what WaitForFirstConsumer is for, how a StatefulSet spans AZs on AZ-local disks, and when EFS is the answer and when it is not."
dateString: November 2026
draft: false
tags: ["Kubernetes", "AWS", "EKS", "Storage", "SRE", "DevOps"]
weight: 2
cover:
    image: "/blog/ha-eks/ha-eks.png"
---


Ask me a month ago what a highly available EKS cluster looked like and I would have said: nodes in three Availability Zones, replicas spread across them, a load balancer in front. That is the right shape, and it is the easy half. I was working through the topic for an interview when the hard half arrived as a question I could not answer: if a node dies, why does its EBS volume go down with it? EBS is network storage. It is not inside the instance. So why does every discussion of HA treat a stateful pod as something that dies with its zone?

The answer is that the volume does not die with its node, and it does die with its AZ, and the gap between those two sentences is the whole subject. "Highly available" is only ever a claim about failure domains: which things can disappear together, and what you have arranged to survive it. This post walks the EKS stack by failure domain, quickly for the parts that just reschedule and slowly for the storage, since that is where I was stuck.

## The control plane is AWS's problem

You do not design this part. The [EKS resilience page](https://docs.aws.amazon.com/eks/latest/userguide/disaster-recovery-resiliency.html) says the control plane is at least two API server instances and three `etcd` instances across three Availability Zones in the region, that EKS replaces unhealthy instances across those zones, and that this is what backs the SLA on the API endpoint. There is no setting that puts it in one zone. Everything below is the data plane, and the data plane is yours.

## Nodes: three AZs, and room for one to be missing

A node's AZ is decided by its subnet. Managed node groups and Karpenter NodeClasses both take a list of subnets, so give them subnets in three zones and nodes land in all three. Then size for the loss of one. If a zone is gone, the other two have to hold everything, which means either a third of your capacity idle or an autoscaler you trust to replace it fast enough. Neither is free, and choosing between them is the actual design decision.

EBS adds a catch. The [EKS data plane guidance](https://docs.aws.amazon.com/eks/latest/best-practices/data-plane.html) puts it plainly: the scheduler will always place a pod that needs an EBS volume in the volume's zone, and if there is no node in that zone the pod cannot be scheduled. The Cluster Autoscaler's [AWS README](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md) draws the consequence: one Auto Scaling group spanning several AZs can scale up in the wrong one, so for EBS workloads you run either a single-AZ group or one group per AZ with `--balance-similar-node-groups`. Karpenter sidesteps this by reading the volume's zone from the claim and launching the node where the disk already is. It does not balance zones on its own, though; its docs leave that to the pod's spread constraints.

## Pods: spread them, then budget the disruptions

Three replicas on three nodes in one AZ is one failure domain with extra steps. The fix is a topology spread constraint keyed on the zone label:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: api
```

`maxSkew: 1` means no zone may hold more than one pod above the emptiest zone. `ScheduleAnyway` is the safer setting: `DoNotSchedule` leaves pods Pending when the spread cannot be met, and the EKS guidance says to use it only if you would rather have fewer pods than an uneven spread. During a zone outage, uneven is exactly what you want. The older way is pod anti-affinity on the same key, and the same guidance warns against making it `required`, since then a Deployment can never have more replicas than you have zones.

Spreading covers the zone failing. A PodDisruptionBudget covers you failing the zone yourself. `minAvailable: 2` on a three-replica service means a node drain, a managed node group upgrade or an autoscaler scale-down cannot take a second replica until the first is back, because all of those go through the Eviction API and honour the budget. The [disruptions docs](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) are equally clear about what it does not do: involuntary disruptions cannot be prevented by PDBs, they only count against the budget. A PDB will not save you from an AZ outage. It will stop you draining a node in the surviving zones while one is down, which is the moment it earns its keep.

## Storage: where I got stuck

The [EBS documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html) settles my question in two sentences. EBS volumes persist independently from the running life of an EC2 instance. The volume and instance must be in the same Availability Zone. My instinct was right, the disk is not in the node, and it was incomplete: the disk is in the zone.

So when a node dies, the volume is still there. The pod reschedules, the volume detaches from the dead node and attaches to the new one, as long as the new one is in the same AZ. If the node vanished without a clean shutdown, the attach/detach controller waits six minutes before force-detaching, according to the [EBS CSI driver FAQ](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/faq.md), so a stateful pod on a dead node is back in six-plus minutes. That delay is the controller choosing a slow recovery over a corrupted disk. When the whole zone dies, the volume is in it. Nothing is deleted, but there is nothing in that zone to attach it to, and the data is unreachable until the zone comes back.

Deleting the pod does not touch the volume either. The pod is a consumer of a PersistentVolumeClaim; the claim is bound to a PersistentVolume; the EBS volume sits behind that. Delete the pod and both stay. Delete the claim and the StorageClass's `reclaimPolicy` decides: `Delete` is the default, `Retain` keeps the EBS volume for you to deal with by hand. The other StorageClass field that matters here is `volumeBindingMode`. The default, `Immediate`, provisions the volume the moment the claim is created, which the [StorageClass docs](https://kubernetes.io/docs/concepts/storage/storage-classes/) say happens without knowledge of the Pod's scheduling requirements and may result in unschedulable Pods: a disk in zone A and a pod that only fits in zone B. `WaitForFirstConsumer` waits until the pod is scheduled and creates the volume in whichever zone it landed, which is why the EBS CSI driver's own example StorageClass sets it:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

Put that together with the spread constraint above and a StatefulSet does something useful. Three replicas spread across three zones each get their own claim, so each gets its own EBS volume, created in its pod's zone. Lose a zone and you lose one replica and one disk; the other two keep serving. EBS replicated nothing across zones. The application did, and it has to be an application that does: Postgres with replicas spread across the zones, or Kafka with rack awareness set to the AZ so each partition's replicas land in different zones, which is [the Kafka post](/kafka/) in one line. The managed version of the same idea is RDS Multi-AZ, which the [RDS docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html) describe as a synchronous standby in a different Availability Zone, with the honest footnote that synchronous replication raises write and commit latency.

## EFS is regional, which is the point and the catch
