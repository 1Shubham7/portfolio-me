---
title: "Kubernetes Controllers: The Loop, the Informer, and the Queue"
description: "How a Kubernetes controller turns a record in etcd into something real: the reconcile loop in Go, the informer that feeds it, the queue that removes duplicates and retries, and the parts that controller-runtime hides from you."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Go", "Controllers", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/kubernetes-controllers/kubernetes-controllers.png"
---

## `kubectl apply` returns before anything runs

You run `kubectl apply -f deployment.yaml`. kubectl sends the Deployment to the API server. The API server validates the object and writes it to etcd. kubectl prints `deployment.apps/web created`. At this point no Pod exists, and no node has pulled an image. The only change is one record in etcd.

The Pods appear because a controller makes them appear. The Deployment controller runs inside kube-controller-manager and holds an open watch on Deployments. It sees the new object, compares the spec with the ReplicaSets that exist, and creates one. The ReplicaSet controller sees the new ReplicaSet and creates the Pods. The scheduler sees Pods with no node and assigns one. The kubelet on that node sees a Pod with its node name and starts the containers.

The API server does not call any of these components. Each one connects to the API server, reads the objects that it watches, and acts on what it finds. This is a pull model. A controller is a client of the API server, with the same `list` and `watch` verbs that kubectl uses.

The Kubernetes docs describe the Job controller in the same way. It does not run Pods itself. It tells the API server to create or remove them.

Everything in Kubernetes that looks automatic is one of these loops. When you see how the informer feeds the loop, you can read the source of any controller, and you can write your own.

## Level-triggered, not edge-triggered

A controller does not react to "what changed". It reacts to "this object may need work". The reconcile reads the whole object and the whole state of the world, and then it corrects the difference. The controller-runtime godoc for `Reconciler` says this directly: reconciliation is level-based, and individual events do not drive the action. The `Request` that a reconcile receives contains a namespace and a name, and nothing about the event.

There are good reasons for this design. When the controller restarts, the informer lists every object again, and each object arrives as an add with no history. When a watch stays down for longer than the API server keeps history, the client gets `410 Gone` and must list again. The queue merges ten edits into one key, so the reconcile sees the last state and not the ten steps. By default, controller-runtime also sends every cached object to the handlers again every ten hours, as an update with no change in it.

Because of this, the reconcile cannot trust a diff. It must be idempotent. If you run it twice on the same state, the second run changes nothing. This is what makes the retry safe. A reconcile that fails halfway can run again from the start, because the start is always "read everything, then fix the difference".

