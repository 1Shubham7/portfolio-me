---
title: "Kubernetes Local Storage: PVs, Nodes, and the WaitForFirstConsumer Trap"
description: "Why your pod is stuck in Pending, why it lands on the wrong node, and how WaitForFirstConsumer fixes local storage scheduling for good."
dateString: May 2026
draft: false
tags: ["Kubernetes", "OpenEBS", "DevOps", "Storage", "ZFS", "Helm"]
weight: 1
cover:
    image: "/blog/pv-shift/k8s.png"
---

## Local Storage vs Network Storage

In Kubernetes, not all storage is equal. Network-attached storage (Ceph, NFS, AWS EBS) can be accessed from any node in the cluster. Your pod can run on node A while the volume lives somewhere else entirely — the storage layer handles the network transport.

Local storage is different. With local storage (like OpenEBS ZFS LocalPV), the data physically lives on a specific node's disk. There is no network replication, no remote access. The pod and the PersistentVolume (PV) **must** be on the same node, without exception. If they aren't, the pod will fail to start.

This single constraint is the root cause of most local storage headaches in Kubernetes.

---

## How OpenEBS ZFS LocalPV Works

When you create a PersistentVolumeClaim (PVC) using the `zfs-localpv` StorageClass, the OpenEBS ZFS CSI driver:

1. Picks a node that has a ZFS pool available
2. Creates a ZFS dataset on that node
3. Creates a PV bound to that PVC
4. Automatically sets a `nodeAffinity` on the PV:

```yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
    - matchExpressions:
      - key: openebs.io/nodeid
        operator: In
        values:
        - <node-name>
```

This nodeAffinity is what pins the PV to its node permanently. Kubernetes uses it to ensure any pod mounting that PV is scheduled on the same node. You don't set this — the provisioner sets it for you after the volume is created.

---

## The Trap: `volumeBindingMode: Immediate`

Here's where things go wrong. StorageClasses have a `volumeBindingMode` field with two options:

- **`Immediate`** (default): The PVC is provisioned the moment it's created, before any pod is scheduled
- **`WaitForFirstConsumer`**: The PVC stays `Pending` until a pod that needs it is actually being scheduled

With `Immediate`, the sequence looks like this:

1. PVC is created
2. Provisioner immediately picks **any available node** with ZFS storage
3. PV is created on that node, PVC goes `Bound`
4. Pod tries to schedule with your `nodeSelector` pointing to a specific node
5. Conflict: pod wants node A, but PV's nodeAffinity says node B
6. **PV's nodeAffinity wins** — pod gets scheduled on node B, your `nodeSelector` is ignored

Your carefully configured `nodeSelector` on the pod does nothing when the PVC is already bound to a PV on a different node. The PV's nodeAffinity always takes precedence.

---

## The Fix: `WaitForFirstConsumer`

With `WaitForFirstConsumer`, the sequence becomes correct:

1. PVC is created → stays `Pending`
2. Pod is scheduled → Kubernetes picks the right node based on your `nodeSelector`
3. Provisioner sees which node the pod is going to → creates the PV **on that node**
4. PVC goes `Bound`, pod starts — both on the same intended node

This is the recommended binding mode for any local storage. `Immediate` is technically wrong for local storage because it makes pod placement non-deterministic.

To update your StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zfs-localpv
provisioner: zfs.csi.openebs.io
parameters:
  compression: "off"
  dedup: "off"
  fstype: zfs
  poolname: primary
  recordsize: 4k
  thinprovision: "no"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # <-- this is the key change
```

**Is this change safe cluster-wide?** Yes. Changing `volumeBindingMode` on an existing StorageClass only affects **new** PVC creations. Already-bound PVCs are completely unaffected. Existing workloads keep running without any disruption.

---

## Pinning a Pod to a Node (for Local Storage)

To control which node your pod — and therefore your PV — lands on, add a `nodeSelector` to the pod spec. For OpenEBS ZFS LocalPV, the node label to use is `openebs.io/nodeid`:

```yaml
# For a Helm chart like Bitnami Redis:
master:
  nodeSelector:
    openebs.io/nodeid: your-node-name

replica:
  nodeSelector:
    openebs.io/nodeid: your-node-name
```

Or using full `nodeAffinity` if you need more control:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: openebs.io/nodeid
          operator: In
          values:
          - your-node-name
```

With `WaitForFirstConsumer` on the StorageClass, this nodeSelector is what the provisioner reads to decide where to create the PV.

---

## How to Shift a PV from One Node to Another

Since PVs are pinned to their node by nodeAffinity, you cannot move a ZFS LocalPV volume to a different node in-place. The data is physically on the original node's ZFS pool. The migration process is:

### Step 1 — Update the pod's nodeSelector

Change the `nodeSelector` in your deployment/StatefulSet to point to the target node. With `WaitForFirstConsumer`, new PVCs will be created on the correct node.

### Step 2 — Back up the data (if needed)

If the PV has data you care about, back it up before proceeding. For stateless or cache workloads (like Redis used as a cache), this may not be necessary.

### Step 3 — Delete the PVC

```bash
kubectl delete pvc <pvc-name> -n <namespace>
```

This also deletes the PV (if `reclaimPolicy: Delete`) and the underlying ZFS dataset on the old node.

### Step 4 — Let it recreate

For a Deployment, the pod restart triggers PVC recreation. For a StatefulSet, you may need to delete the pod:

```bash
kubectl delete pod <pod-name> -n <namespace>
```

The StatefulSet controller recreates the pod, which triggers PVC creation, which (with `WaitForFirstConsumer`) creates the PV on the node your `nodeSelector` points to.

### Step 5 — Verify

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Check that the PV's nodeAffinity now shows the correct target node.

---

## Key Takeaways

- **Pod and PV must be on the same node** for local storage — this is a hard requirement, not a preference
- **The provisioner sets nodeAffinity on the PV automatically** — you don't configure it, but it controls pod scheduling
- **`volumeBindingMode: Immediate` is wrong for local storage** — it creates PVs before pods are scheduled, leading to node mismatches
- **`volumeBindingMode: WaitForFirstConsumer` is the correct mode** — provisioning waits for pod scheduling so the PV is always created on the right node
- **Changing `volumeBindingMode` is safe cluster-wide** — it only affects new PVCs, not existing ones
- **You cannot move a local PV** — you can only delete and recreate it on the target node
- **`nodeSelector` on the pod only works correctly with `WaitForFirstConsumer`** — with `Immediate`, the PV's nodeAffinity overrides it