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

Recently I sat down and read it properly, the man page first and then the source. This post is what I found, in the order I asked the questions. Most of it is not exciting, and that is sort of the point. logrotate comes preconfigured on every mainstream distro, so almost nobody touches it until they have to, and that is exactly when its few sharp edges show up: `size` versus time, what `dateext` can and cannot name, and what `copytruncate` costs.

## What is it actually doing, and why does a log need rotating?

A log file that nothing ever rotates grows until the disk is full. Before it gets there it becomes useless in a quieter way: a multi-gigabyte text file takes forever to `grep`, `less` struggles with it, and anything that ships logs elsewhere has to reread or tail one enormous file. And you cannot just delete it. A process that has the file open keeps the inode alive, so `rm` on a live log frees nothing; `df` does not move until the process closes the descriptor. I wrote about that in [the storage section of the Linux concepts post](https://1shubham7.blog/linux-concepts/), and logrotate is the tool that exists so you never have to do it by hand.

Rotation is the four verbs in the title. Rename the current file to an archive name and start a fresh one (rotate). Gzip the archives, usually one cycle later (compress). Delete archives beyond a count or an age (prune). Do it again on a schedule (repeat).

One thing I had wrong: logrotate is not a daemon. It is a program that runs, reads its configuration, compares every configured log against its rule and against a state file, does whatever is due, and exits. Nothing watches the file in between runs. That has consequences for every question below.

## What runs it, and when?

For most of its life the answer was a script in `/etc/cron.daily`. The upstream project still ships one in its `examples` directory: it runs `/usr/sbin/logrotate /etc/logrotate.conf`, and if the exit code is non-zero it writes a line to syslog through `logger`. That is the whole thing.

On a systemd distro today the answer is a timer. The package drops two units: `logrotate.service`, which is a oneshot that runs the same command, and `logrotate.timer`, which decides when. The timer upstream ships looks like this, and both Debian and the RHEL family install it unchanged:

```ini
[Unit]
Description=Daily rotation of log files
Documentation=man:logrotate(8) man:logrotate.conf(5)

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

If you have only used cron, a timer is two units instead of one line. The timer unit owns the schedule and nothing else. The service unit owns the command, the user it runs as, its environment, and in this case a list of hardening options that keep logrotate away from kernel tunables and the hostname. The two are paired by name: `logrotate.timer` activates `logrotate.service` unless told otherwise. What you gain over cron is that the run is a real unit, so its output goes to the journal, `systemctl status` tells you whether the last run failed, and you get the two `[Timer]` lines after `OnCalendar`, which a crontab line cannot express.
