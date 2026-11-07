---
title: "Why Does Your Kubernetes Pod Always Take 30 Seconds to Die?"
description: "A pod that takes exactly 30 seconds to terminate is a pod that never heard SIGTERM. Why the kernel refuses to deliver a default-action signal to PID 1, why a shell-form CMD puts sh in that seat and your app behind it, how to check what PID 1 actually is in a running container, and the three fixes: exec form, a real signal handler, or tini."
dateString: November 2026
draft: false
tags: ["Kubernetes", "Linux", "Docker", "Containers", "SRE", "DevOps"]
weight: 2
cover:
    image: "/blog/pod-30-seconds/pod-30-seconds.png"
---


`kubectl delete pod` returns, the pod sits in `Terminating`, and 30 seconds later it is gone. Not 3 seconds, not 12, always 30. A rolling deploy of ten replicas turns into a five-minute wait, and the number is far too round to be the application doing anything. It is a timeout. Something was asked to stop, did not hear, and was eventually shot.

I started reading about this for an interview, where it is a well-worn question, and found that the usual one-line answer, "PID 1 ignores SIGTERM", is close but not what Linux actually does. The real rule is more specific and explains more, including the case where your app is not even PID 1.

## What Kubernetes sends, and when

The [pod lifecycle docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) lay the sequence out. `kubectl delete` stamps the pod with a deletion time and the grace period, which is `spec.terminationGracePeriodSeconds` and defaults to 30. The kubelet sees the pod marked terminating and, if a container has a `preStop` hook, runs that first; the hook runs inside the same grace period, and if it is still going when the period ends it gets a one-off extension of 2 seconds. Then the kubelet asks the container runtime to send TERM to process 1 in each container. If anything is still running when the grace period expires, the runtime sends SIGKILL, the kubelet moves the pod to a terminal phase, and the object disappears from the API.

So the second signal is SIGKILL, not SIGINT, and it is the one that cannot be caught or ignored. A pod that takes exactly 30 seconds is a pod where SIGTERM achieved nothing and SIGKILL did all the work. The question is why SIGTERM achieved nothing, and there are two usual answers.

## PID 1 does not get the default action

For an ordinary process, SIGTERM needs no code at all. The default action is to terminate, so a process that never mentions signals dies the moment one arrives. PID 1 is the exception, and `kill(2)` states it in one sentence: the only signals that can be sent to process ID 1 are those for which init has explicitly installed signal handlers. The man page adds that this is to make sure the system is not brought down by accident.
