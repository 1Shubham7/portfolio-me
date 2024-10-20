---
title: "Beginners guide to Kubernetes namespaces"
description: ""
dateString: August 2024
draft: false
tags: ["Kubernetes", "KubeEdge", "Golang", "E2E Testing", "Ginkgo", "Gomega", "Testify", "Behavior Driven Development (BDD)", "Test Driven Development (TDD)"]
weight: 100
cover:
    image: "experience/lfx.webp"
---

Namespaces are Kubernetes Objects used to divide a Cluster into smaller organized sub clusters as per different needs. Namespaces enables users to isolate group of resources within a cluster, this is useful when multiple teams are working on a shared cluster and each team is independent of others. In this article, we will discuss how to change or switch namespaces in a Kubernetes cluster, but before that let's start with the basics.

Kubernetes  Namespaces
Namespaces are Kubernetes Objects used to divide a Cluster into smaller organized sub clusters as per different needs. Namespaces enables users to isolate group of resources within a cluster, this is useful when multiple teams are working on a shared cluster and each team is independent of others. In simpler words, Namespaces are used to organize resources. You can have multiple Namespaces in a Cluster And these Namespaces are kind of virtual Clusters of their own. Within a Kubernetes Namespace, resources must have unique names, but across different Namespaces, you can have resources with the same name.

Why do we need Namespaces
Now Let's first discuss what is the need for Namespaces. when should you create them? And how you should use it? 

1. Structuring Resources in Groups
The biggest use case of creating your Namespaces is this that instead of cluttering the Default Namespace with various components, Namespaces allow grouping resources logically. For Example, you can have a database Namespace where you deploy your database and all its required resources and you can have a Monitoring Namespace where you deploy monitoring related stuff.

2. Preventing Conflicts between Multiple Teams
When multiple teams share a cluster, Namespaces prevent conflicts by isolating team deployments. This ensures that each team can work independently without interfering with others. For example, one team deploys an application which is called "my-app-deployment" and that deployment has its certain configuration. Now if another team had a Deployment that accidentally had the same name but a different configuration and they created that Deployment or they applied it, they would overwrite the first team's Deployment. But with different namespaces assigned to different teams, they can have same Deployment name without any conflicts.

3. Resource Sharing
Lets say you have one Cluster and you want to host both Staging and Development environment in the same Cluster. and the reason for that is if you're using Nginx Controller or Elastic Stack for logging, you can deploy it in one Cluster and use it for both environments. In that way you don't have to deploy these common resources twice in two different clusters. So now the staging can use both resources as well as the development environment.

4. Blue/Green Deployment
Namespaces facilitate Blue/Green Deployment by allowing different versions of an application to coexist in the same cluster, utilizing shared resources efficiently. Which means that in the same cluster you can have two different versions of Production - The one that is in production now and another one that is going to be the next production version.

5. Access Control and Resource Limitation
Another reason for using Namespaces is to restrict access and resource consumption in multi-team environments. With each team having its Namespace, access can be limited to only their Namespace, preventing accidental interference with other teams' work. This ensures a secure and isolated environment for each team. Additionally, resource quotas per Namespace prevent one team from consuming excessive resources, ensuring fair resource allocation across the cluster.

How to Change Namespace in Kubernetes - Tutorial
Step 1: Starting a Kubernetes cluster
You can skip this step if you already have a Kubernetes cluster running on your machine. But in case you don't have a cluster running, enter the following command to create Kubernetes locally using Minikube:

minikube start



Screenshot-2024-03-27-085106

Step 2. Creating Namespaces
Let us create a Namespace so that we can later switch between the default and new namespace, before that let us check the list of namespaces we have currently in our cluster by entering the following command:

kubectl get ns
and you will get the same results:





Screenshot-2024-03-27-013427


"default" is the default namespace by Kubernetes and rest of the namespaces are created by Minikube. To create a namespace called namespace-one, enter the following command:

kubectl create namespace namespace-one



Screenshot-2024-03-27-082134

Step 3. Switching between namespaces
Now that we have a  new namespace created, let us learn how to switch between them. By default the currently active namespace is "default" namespace. You can check the currently active namespace by the following command:

kubectl config view --minify | grep namespace



Screenshot-2024-03-27-083208

You can change or switch Namespaces in Kubernetes using the following command:

kubectl config set-context --current --namespace=namespace-one
And now the namespace-one will be the currently active namespace. And is we check the currently active namespace:

kubectl config view --minify | grep namespace
You will see namespace-one as the currently active namespace:




Screenshot-2024-03-27-083757

This is how you can change namespace or switch between namespace in a Kubernetes Cluster. Pat yourself on the back because we have just learned how to switch between Namespaces in a Kubernetes cluster.

Operations on Kubernetes Namespaces
To Create a Kubernetes Namespace enter the following command:
kubectl create namespace <namespace-name>
To List Namespaces in the Cluster enter the following command:
kubectl get namespaces
To Delete a Namespace enter the following command:
kubectl delete namespace <namespace-name>
To View Namespace Detail enter the following command:
kubectl describe namespace <namespace-name>
To Set Current Namespace enter the following command:
kubectl config set-context --current --namespace=<namespace-name>
To Edit a Namespace enter the following command:
kubectl edit namespace <namespace-name>
To List Pods in a Namespace enter the following command:
kubectl get pods --namespace=<namespace-name>
To List Services in a Namespace enter the following command:
kubectl get services --namespace=<namespace-name>
To Apply a Resource Configuration to a Namespace enter the following command:
kubectl apply -f <resource-config.yaml> --namespace=<namespace-name>
To Export Namespace Configuration enter the following command:
kubectl get namespace <namespace-name> -o yaml > namespace-config.yaml
To Import Namespace Configuration enter the following command:
kubectl apply -f namespace-config.yaml
Conclusion
Kubernetes Namespaces are a way to group resources in a Kubernetes Cluster. In this article, we discussed how Namespaces work, What are the default Namespaces present already in a Kubernetes Cluster. From need of Namespaces to it's characteristics, we discussed everything. Do follow the tutorial and try creating resources in a Namespace by your own because that's the best way to learn. Make sure to learn more about kubectx and its features. We hope that this article helped you improve your understanding about Namespaces and you learned something valuable out of it.

Kubernetes Namespaces - FAQ's
What are the out of the box Namespaces provided by Kubernetes?
There are three out of the box Namespaces provided by Kubernetes:

default
kube-system
kube-public
How to change Namespace in Kubernetes?
You can change or switch Namespaces in Kubernetes using the following command:

kubectl config set-context --current --namespace=namespace-one
How do I export Namespace configuration?
You can export the Namespace Configuration in Kubernetes by entering the following command:

kubectl get namespace <namespace-name> -o yaml > namespace-config.yaml
What is the purpose of a namespace?
Purpose of Namespaces is to group resources of a Kubernetes Cluster in order to manage them in efficient manner.

What are some unique characteristics of Namespaces?
Some unique characteristics of Namespaces are as following:

You can not access most of the resources from another namespace.
Service can be shared across Namespaces
Volumes and Nodes cannot be created within a Namespace