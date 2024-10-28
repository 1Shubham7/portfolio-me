---
title: "Kubernetes Statefulsets"
description: "In simplest terms StatefulSets are Kubernetes component that is used specifically for stateful applications. These are workload API object used to manage stateful applications. They manage the deployment and scaling of a set of Pods (Creating more replicas or deleting them), and StatefulSets are also responsible for the ordering and uniqueness of these Pods. StatefulSet was released in the Kubernetes 1.9 release..."
dateString: October 2024
draft: false
tags: ["Kubernetes", "Statefulsets", "K8s", "DevOps"]
weight: 100
cover:
    image: "blog/jwt-logo.png"
---

StatefulSets are API objects in Kubernetes that are used to manage stateful applications. There are two types of applications in Kubernetes, Stateful applications and stateless applications. There are two ways to deploy these applications:

- Deployment (for stateless applications)
- StatefulSets (for stateful applications)

## What are Stateful Applications?

Those applications that maintain some form of persistent state or data are called stateful applications. The key characteristic that differentiates them from stateless applications is that these applications don't rely on storing data locally and they don't treat each request as independent. They manage data between interactions. Sometimes stateless applications connect to the stateful application to forward the requests to a database.

## What are Stateless Applications?

Those applications that do not maintain any form of persistent state or data locally are called stateless applications. In stateless applications, each request or interaction is treated independently. These applications are designed to be highly scalable, easy to manage, and fault-tolerant because, unlike Stateful applications, they don't have to track past interactions or requests.

Stateless applications are deployed using deployment component. Deployment is an abstraction of pods and allows you to replicate the application, meaning it allows you to run to 1, 5, 10 or n identical pods of the same stateless application.

## What are StatefulSets?

In simplest terms StatefulSets are Kubernetes component that is used specifically for stateful applications. These are workload API object used to manage stateful applications. They manage the deployment and scaling of a set of Pods (Creating more replicas or deleting them), and StatefulSets are also responsible for the ordering and uniqueness of these Pods. StatefulSet was released in the Kubernetes 1.9 release.

StatefulSets will represent the set of pods with different (unique), persistent identities, and elastic hostnames (stable). It makes you assure about the ordering of scaling and deployments. Before understanding StatefulSets, you must understand Kubernetes Deployment.

Here is an example of a StatefulSet named web:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

## When to Use StatefulSets

StatefulSets in Kubernetes are ideal for deploying stateful applications that require stable, unique network identifiers, persistent storage, and ordered, graceful deployment and scaling. They are suitable for applications like databases, key-value stores, and messaging queues that require consistent identity and storage.

### Example of Stateful and Stateless Applications

Consider a node.js application connected to a MongoDB database. When a  request comes to the node.js application it handles the request independently and does not depend on previous data to do that. It handles the request based on the payload in the request itself. This node.js application is an example of Stateless application. Now the request will either update some data in the database or query some data from the database. When node.js forwards that request to MongoDB, MongoDB updates the data based on the previous state of the data or query the data from its storage. For each request it needs to handle data and it  depends upon the most up-to-date data or state to be available while node.js is just a pass-through for data updates or queries and it just processes code. Hence the node.js application should be a stateless application while the MongoDB application must be a stateful application.

Sometimes stateless applications connect to the stateful application to forward the requests to a database. This is a good example of a stateless application forwarding request to a stateful application.

How to Create a StatefulSet in Kubernetes
Here is a step by step tutorial on how to use StatefulSets and some basic operations on StatefulSets.

Create a Nginx StatefulSet Application
**Step 1.** Create a StatefulSet file. you can do that by entering the following command:

touch example-statefulset.yaml
**Step 2.** Open this file in a code-editor and write the following code into it:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gfg-example-statefulset
  annotations:
    description: "This is an example statefulset"
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "gfg-example-service"
  replicas: 3 # remember this, we will have 3 identical pods running
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: myclaim
```

**Step 3.** Now we have to create a service file and a PersistentVolumeClaim file.

```sh
touch example-service.yaml
touch example-persistentVolumeChain.yaml
```

Create a Service for the StatefulSet Application

**Step 4.** Enter the following code into the service file:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gfg-example-service
  annotations:
    description: "this is an example service"
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
Create a PersistentVolumeClaim for Application
Step 5. Enter the following code into the PersistentVolumeClaim file:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 8Gi # This means we are requesting for 8 GB of storage
```

Now lets apply these changes.

**Step 6.** Enter the following command in your terminal to create the gfg-example-statefulset:

```sh
kubectl create -f example-statefulset.yaml
```

This will create our gfg-example-statefulset, you will get a similar result:

Kubectl- apply 

now if we search our StatefulSets in our terminal by the command

 kubectl get statefulsets
we will find our gfg-example-statefulset in the list.




kubectlget stateful set 

**Step 7.** Enter the following command in your terminal to create the gfg-example-service.

kubectl apply -f example-service.yaml
this will create a service with the name "gfg-example-service"

**Step 8.** Let's check our pods and services, for getting the list of pods enter the following command in your terminal:

kubectl get pods
You will get the list of the three gfg- pods that we create though defining three replicas in the example-stateful-set.yaml file. You will get a similar output:




Kubectl get pods 

for checking the list of services, enter the following command in your terminal:

kubectl get services
This will give you similar output:




Kubectl get services

Some Operations On StatefulSets
Adding a StatefulSet: To add a StatefulSet to your Kubernetes cluster, use the command kubectl create -f [StatefulSet file name], replacing [StatefulSet file name] with the name of your StatefulSet manifest file.

kubectl create -f [StatefulSet file name]

Deleting a StatefulSet: To delete a StatefulSet in Kubernetes, you can use the kubectl delete statefulset [name] command, where [name] is the name of the StatefulSet you want to delete.

kubectl delete statefulset [name]

Editing a StatefulSet: The command kubectl edit statefulset [name] allows you to modify the configuration of a StatefulSet directly from the command line by opening an editor.

kubectl edit statefulset [name]

Scaling of Replicas: The kubectl scale command scales the number of replicas in a StatefulSet named [StatefulSet name] to the specified [Number of replicas].

kubectl scale statefulset [StatefulSet name] --replicas=[Number of replicas]

**Step 9.** Now let's scale up our pods and check if it works! for scaling up the pods to 6 pods, enter the following command:

kubectl scale statefulset gfg-example-statefulset --replicas=6
This will create 3 more pods and number of pods are now 6, to get list of pods enter the following command:

kubectl get pods
You will get a similar output:




Kubectl get pods 

**Step 10.** Now let's scale down pods to 3, for that enter the same command, just change the number of replicas back to 3:

kubectl scale statefulset gfg-example-statefulset --replicas=3
now if we check the list of pods by 

kubectl get pods
you will see only 3 pods running:




pods 

In this way we can create StatefulSets, scale them up and then scale them down as well. Make sure to delete the StatefulSet and the service before closing the terminal.To know more commands of of kubectl refer to Kubectl Command Cheat Sheet.

## How Stateful Applications Work?

In stateful applications like MySQL, multiple pods cannot simultaneously read and write data to avoid data inconsistency.
One pod is designated as the master pod, responsible for writing and changing data, while others are designated as slave pods, only allowed to read data.
Each pod has its own replica of the data storage, ensuring data isolation and independence.
Synchronization mechanisms are employed to ensure that all pods have the same data state, with slave pods updating their data storage when the master pod changes data.
Continuous synchronization is necessary to maintain data consistency among all pods in the stateful applications.

Example:
Let's say we have one master and two slave pods of MySQL. Now what happens when a new pod replica joins the existing setup? because now that new pod also needs to create its own storage and take care of synchronizing it what happens is that it first clones the data from the previous pod and then it starts continuous synchronization to listen for any updates by master pod. since each pod has its own data storage (persistent volume) that is backed up by its own physical storage which includes the synchronized data and the state of the pod. Each pod has its own state which has information about whether it's a master pod or a slave pod and other individual characteristics. All of this gets stored in the pods own storage. Therefore when a pod dies and gets replaced the persistent pod. Identifiers make sure that the storage volume gets reattached to the replacement pod. In this way even if the cluster crashes, it is made sure that data is not lost.

## Conclusion

In this article we discussed about how to use Kubernetes StatefulSets. StatefulSets are Kubenetes components used to deploy stateful applications. Stateful applications are those applications that maintain some form of persistent state or data. A good example would be any application with a database. We discussed about how to deploy an stateful application using StatefulSets. After that we discussed how Stateful applications work? At the end we discussed the difference between StatefulSet and deployment which basically moves around the point that deployment are used to deploy stateless application and StatefulSets are used to deploy statefull applications. We will end this article by addressing some FAQs.

## Kubernetes StatefulSets - FAQs

#### How do I increase volume size in Kubernetes?

To increase volume size in Kubernetes, you need to modify the PersistentVolumeClaim (PVC) specification by changing the storage size. Then, Kubernetes automatically provisions additional storage to meet the new size requirement, provided the underlying storage class supports dynamic provisioning.

#### When to use hostPath in Kubernetes?

The hostPath volume type in Kubernetes is suitable for scenarios where you need to access files or directories on the node's filesystem directly within a pod. It's commonly used for accessing node-specific resources or for sharing data between containers on the same node.

#### What is the difference between PVC StatefulSet and deployment?

PersistentVolumeClaims (PVCs) in StatefulSets are used to provide stable, persistent storage for individual pods, ensuring data persistence and identity across pod restarts. In contrast, deployments typically utilize ephemeral volumes, suitable for stateless applications where data persistence is not a requirement.

#### Can We Deploy A Stateless Application Using StatefulSets?

While technically possible, it's not recommended to deploy stateless applications using StatefulSets. StatefulSets are specifically designed for stateful applications requiring stable, unique identifiers and persistent storage. Deploying stateless applications with StatefulSets can introduce unnecessary complexity and resource overhead.

#### Is stateless better than stateful?

It depends on the specific requirements of the application. Stateless applications are simpler to manage and scale horizontally, while stateful applications maintain data integrity and are better suited for certain workloads like databases or messaging systems.