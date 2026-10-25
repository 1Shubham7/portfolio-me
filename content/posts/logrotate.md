---
title: "logrotate from Scratch: Rotate, Compress, Prune, Repeat"
description: "What logrotate actually does, what runs it and when, where its configuration and state live, how to write and test a rule for your own application, and the handful of sharp edges that stay hidden until it runs at a time you did not plan."
dateString: October 2026
draft: false
tags: ["Linux", "logrotate", "systemd", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/logrotate/logrotate.png"
---

Every Linux server I have ever logged into already had logrotate on it. It was installed before I got there, it ran every night, and the dated or numbered files next to each log in `/var/log` were proof that it worked. In years of that I never opened `/etc/logrotate.conf`. I could not have told you what started logrotate, where it kept track of what it had already done, or what it would do if it ran twice in one day.
