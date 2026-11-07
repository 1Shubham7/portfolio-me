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
