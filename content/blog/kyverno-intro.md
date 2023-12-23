---
title: "Introduction to Kyverno"
description: ""
dateString: December 2023
draft: false
tags: ["Kubernetes", "Kyverno", "Kubernetes API"]
weight: 99
cover:
    image: "blog/kyverno.png"
---

Kyverno is admission controller that can be used to govern and enforce how a Kubernetes cluster is used.

## Admission Controllers

Let's first start with admission controllers since Kyverno is an admission controller. In Kubernetes admission controllers are used to intercept requests going to the API servier and one of the two or both the things with them - mutate or/and validate.

Admission control phas

Kyverno is an admission controller for a Kubernetes cluster. But it is more than a admission controller. Kyverno comes from the Greek word meaning "to govern".

The primary job of a admission controller is to validate the resources. If a resource is passed to an admission controller it will tell if it is allowed or not. And that is all what an admission controller does. But as we said Kyverno is more than just an admission controller. Kyverno acts as a admission controller and validates the resources but it also can change the resources transparently.

In this beginner friendly article, we will discuss what Kyverno is and how it works on theory, but before that let's have a look at some basic terms:

Custom resources
Custom resources are the extensions of the Kubernetes API that are not available in the default Kubernetes installation. These are like additional stuff or customization we can add on Kubernetes. However, many Kubernetes core functions are now build using custom resources, this make Kubernetes very modular. A Kyverno policy is a standard Kubernetes custom resource.

Custom Controllers
The custom resources lets us store and retrieve structured data. But when we combine a custom resource with a custom controller, it provides us a declarative API. Custom controllers are used to encode domain knowledge for specific applications into an extension of Kubernetes API.

Admission Controllers
In simplest terms, If a resource is passed to an admission controller it will tell if it is allowed or not. To explain further, the Kubernetes official documentation defines admission controller as - "An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized." Admission controllers can either validate resources or mutate them, or do both.

Why Kyverno
Kyverno is not just another admission controller, it can do much more than that. Where an ordinary admission controller can only validate the resources in the cluster. Kyverno not just validates the resources but can also change the resources transparently. when Kyverno receives the API requests, it can validate them, this can also be done in blocking mode.

Blocking mode means it won't prevent or allow cluster to run but specify that in a policy report.

Other than this Kyverno also has the ability to mutate the resources transparently. Through Kyverno you can also generate Kubernetes resources. This means you can create new Kubernetes resources of any type or configuration based upon a policy that you define and install in the cluster.

Kyverno Policy
A Kyverno policy can be one of: 

Cluster scoped
Namespace scoped 
Inside the Kyverno policy we have rules. A single policy has one or multiple rules. Each rule has a match block and can optionally also have an exclude block. 



Kyverno

You can define scope of the resource you want to apply your policy to, using the following options. You can off-course choose one or multiple of these options:

Resource Kinds
Resource Names
Labels
Annotations
Namespaces
Namespace Labels
(Cluster)Roles
Users
Groups
ServiceAccounts
You can also declare what you want from the Policy, do you want to use it for Validation, mutating or something else. Here are the options you can choose in Kyverno:

Validate the resource
Mutate the resource
Generate a resource
Verify images 
Example of Kyverno Policy
Here is a sample Kyverno Policy, A "ClusterPolicy" applies to the entire cluster. you can see we have a single rules block inside which we have a match block. We have set validationFailureAction to Audit therefore, it will be allowed even after the resource validation failure what but we should get a report if the resource validation fails.

This policy will perform validation of the pods and check the label. If the label is not provided with any value then it will send a message saying "The label `app.kubernetes.io/name` is must" .



Screenshot-2023-09-29-072343

You can copy the same policy from the following code:

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
        kinds:
          -Pod
    validate:
      message: "The label `app.kubernetes.io/name` is must"
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"

Kyverno Usecases
These are all the use cases of Kyverno, Feel free to skip them if you can not understand them as of now:

Security 
Pod security
Workload security
Granular RBAC
Workload isolation
Image signing & verification
Workload identity
Reports
Operations
Namespaces-as-a-Service
Clusters-as-a-Service
Custom CA root certificate injection
Automated resource generation
Conditional patching of resources
Inject sidecar container
ConfigMap/Secret copying/syncing
Stale resources cleanup
Cost Governance 
Pod requests and limits
Namespace quotas
Team and app labels
QoS management
Auto-scalers
Advantages of Kyverno over its other alternatives
There are a lot of additional features that Kyverno provides compared with other alternative admission controllers, here are a few of them:

1. Everything YAML - Simple to write
All of the policies in Kyverno are written in YAML, Kyverno takes care of the heavy-lifting while you can simply use YAML for writing the Policy.

1. Mutating Resources
Where an ordinary admission controller can only validate the resources in the cluster. Kyverno not just validates the resources but can also change the resources transparently.

1. Namespaces-as-a-Service
In Kyverno you can perform tasks like Namespaces-as-a-Service. You can simply create a namespace with a bunch of other resources that are in it. Kyverno can then generate all of those resources for you based on the manifest. You don't have to run bash scripts or spit out resource quotas and limit ranges. You just have to simply  write a Kyverno policy in YAML, install it in the cluster and version it. Kyverno will perform all the tasks for you after that.

1. Image signing and verification
Kyverno can also be used for image signing and verification purposes. Kyverno can do image verification without needing other bolt-on components. The Kyverno policy will need to know which images you want to check. Whenever that image gets built, it will get signed and pushed to the OCI registry.

1. Some more features of Kyverno
Some more features of Kyverno includes:

Matching resources using label selectors and wildcards.
Validating and mutating resources using overlays.
Synchronizing configurations across namespaces.
Blocking any non-conformant resources using admission controls, or report policy violations.
Kyverno CLI - for testing policies and validating  resources in the CI/CD pipeline.
managing policies as code using familiar tools like git and kustomize.
Conclusion
As we discussed in previous sections, Kyverno is not just any admission controller, it is much more than that. An ordinary admission controller can only validate the resources in the cluster. Kyverno not just validates the resources but can also change the resources transparently. We discussed about Kyverno is this article we started off with an introduction to Kyverno, later we dicusses the terms like custom resources and custom controllers. Custom resources are the extensions of the Kubernetes API that are not available in the default Kubernetes installation. While custom controllers are used to encode domain knowledge for specific applications into an extension of Kubernetes API. We then say what Kyverno does? and an example of a Kyverno policy. At the end we discussed the advantages of Kyverno over other admission controllers and various usecases of Kyverno.

This marks the end of this article and we hope that this article helped you improve your understanding about Kyverno and policy management in Kubernetes.

`minikube start`


`kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml`


`kubectl explain policy.spec.rules.preconditions`


Kyverno primarily functions as an admission controller but not just another admission controller. Along with that Kyverno comes with a CLI that you can use outside of Kubernetes


Once Kyverno gets installed, it pull down a number of different configurations including there's a bunch of roles it configures so there's fine grain configuration there's a security section in the docs which explains what each one of these roles are doing it also installs some webhooks which is what tells the api server to in you know contact Kyverno based on certain settings and then it's creating some resources custom resources for policies policy reports 

Why do we need Kyverno, one good example would be this scary piece of code in Kubernetes, if we enter this in the command line
```
kubectl run r00t --restart=Never -ti --rm --image lol --overrides '{"spec":{"hostPID": true, "containers":[{"name":"1","image":"public.ecr.aws/h1a5s9h8/alpine:latest","command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"],"stdin":true,"tty":true,"securityContext":{"privileged":true}}]}}'
```
What this is doing is it's running a simple image but then it's doing an nsenter and also elevating privileges for the Pod. So if we run this, the interesting thing that happens if we have no policies or no guardrails in place it that all we need is permissions to run a container in any namespace and we will have access to bash shell. With that we can look at all the containers running.
`cd var/log/containers`

`ls`

`exit`
