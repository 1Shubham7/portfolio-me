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

Containers inherit the rule. The first process in a container is PID 1 of its own PID namespace, and `pid_namespaces(7)` says only signals for which that process has established a handler can be sent to it, even by privileged processes. Two signals are forcibly delivered anyway when they come from an ancestor namespace, which is where the container runtime lives: SIGKILL and SIGSTOP. That is the whole mechanism. The runtime sends SIGTERM, the kernel checks whether PID 1 has a handler for it, finds none, and drops the signal. Thirty seconds later the runtime sends SIGKILL, which the kernel cannot be talked out of. And when the init of a PID namespace dies, the kernel SIGKILLs every other process in the namespace, which is why nothing survives the end of the grace period.

Whether your app has a handler depends on the runtime more than on your code. The Go runtime installs its own: `runtime/sigtab_linux_generic.go` marks SIGTERM `_SigNotify + _SigKill`, `initsig` installs a handler for every signal not marked `_SigDefault`, and that handler exits the program with the status a shell expects. So a Go binary with no signal code still dies on SIGTERM as PID 1, just not gracefully. Python installs handlers only for SIGPIPE and SIGINT, per its [signal docs](https://docs.python.org/3/library/signal.html), so a Python service as PID 1 with no handler is the textbook 30-second pod. A C program that never calls `sigaction` is the same.

PID 1 has a second job, which is why people reach for an init even when signals are fine. Orphaned processes are reparented to it, and something has to `wait()` on them when they exit, or they stay as zombies holding a PID each.

## The shell-form trap

The other way to lose SIGTERM is to have it land on a process that is not your app. These two Dockerfiles are the same container:

```dockerfile
FROM ubuntu
CMD sleep 1000
```

```dockerfile
FROM ubuntu
CMD ["sh", "-c", "sleep 1000"]
```

The first is the shell form, and the [Dockerfile reference](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form) says it is wrapped in `/bin/sh -c`, which makes it identical to the second. In both, `sh` is PID 1 and `sleep` is its child. The docs are direct about the cost: the executable is not PID 1, does not receive Unix signals, and does not receive SIGTERM from `docker stop`, so stop is forced to send SIGKILL after its timeout.

Walk it through. SIGTERM goes to process 1, which is `sh`. A non-interactive shell has no handler for TERM, so as PID 1 the kernel never delivers it. Even if it did, `sh` is blocked in `wait()` on its child and does not forward signals; that is not something a shell does. `sleep` is not PID 1, so it would die instantly on SIGTERM, but nobody sends it one. Thirty seconds pass. SIGKILL from the ancestor namespace is forced onto `sh`, `sh` dies, the namespace's init is gone, and the kernel SIGKILLs `sleep` to clear the namespace. The sleep never stopped because it never heard anything; the shell never stopped because it could not.

One wrinkle if you try this on a laptop: bash optimises a lone `-c` command into an `exec`, so `bash -c "sleep 1000"` can leave `sleep` as PID 1 and the trap does not show. The image's `/bin/sh` decides, so do not count on it either way.

The check is cheap. PID 1's command line is in `/proc`:

```bash
$ kubectl exec api-7d4c9-x2k1p -- cat /proc/1/cmdline | tr '\0' ' '
sh -c sleep 1000
```
