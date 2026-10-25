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

`OnCalendar=daily` is shorthand for midnight, local time. `RandomizedDelaySec=1h` adds a random delay of up to an hour, picked fresh each time, so a fleet of machines does not all gzip their logs at 00:00:00. `Persistent=true` is the one that matters most. systemd stores the time of the last trigger on disk, under `/var/lib/systemd/timers/`, and when the timer starts it checks whether a scheduled run was missed while it was inactive. If the machine was off or rebooting at midnight, the run fires shortly after boot instead of being skipped. Without that line a server rebooted at the wrong moment would just not rotate that day.

The commands to inspect all of this:

```bash
systemctl list-timers logrotate.timer
systemctl status logrotate.timer
systemctl cat logrotate.timer
systemctl status logrotate.service
journalctl -u logrotate.service
```

`list-timers` gives you the next and last trigger times in one row; `status` on the service shows the exit code of the last run; `journalctl` shows what logrotate printed, which on a healthy run is nothing. Debian still installs `/etc/cron.daily/logrotate` too, but it carries a patch that makes the script exit immediately when systemd is running, so the timer is the only thing that fires.

## Does it come installed?

Mostly yes, which is why the previous question was a mystery to me for so long.

On Debian and Ubuntu the package has priority `important`, so a standard install includes it, and rsyslog recommends it, so a box that logs to `/var/log/syslog` almost certainly has it. The RHEL family (RHEL, Rocky, Alma, Fedora) ships it with the timer enabled by the distro's systemd preset when the package is installed, and rsyslog recommends it there as well. openSUSE and SLES ship the timer; I have not checked a fresh install, so run the check below. Arch is the odd one out: logrotate is in `core` but not in the `base` metapackage, so a fresh Arch install does not have it unless you add it; once you do, the package ships the `timers.target.wants` symlink, so the timer is enabled without a separate step.

The check is the same everywhere:

```bash
systemctl is-enabled logrotate.timer
systemctl is-active logrotate.timer
```

## Where does the configuration live?

Two places, and the split is the thing to understand.

`/etc/logrotate.conf` is the global file. It sets defaults for every log, then includes a directory. The upstream example is short: `weekly`, `rotate 4`, `create`, `dateext`, `compress` commented out, and `include /etc/logrotate.d`. Distros tweak it. Debian carries a patch that removes `dateext` from the default, which is why rotated files on Debian end in `.1` and `.2.gz` while the same files on Rocky end in a date.

`/etc/logrotate.d/` is where packages drop their own rules. Install nginx and you get `/etc/logrotate.d/nginx`; install rsyslog and you get `/etc/logrotate.d/rsyslog`. Each file is one or more blocks of the form `path { directives }`, and because the directory is included from the main file, every block inherits the global defaults and overrides what it needs to. The file belongs to the package that installed it, which you can confirm:

```bash
dpkg -S /etc/logrotate.d/rsyslog
rpm -qf /etc/logrotate.d/rsyslog
```

Why ownership matters comes up under the sharp edges. The short version is that both package managers treat these as configuration files, so if you edit one by hand, the next upgrade either prompts you or leaves the new version beside yours with a `.dpkg-dist` or `.rpmnew` suffix, and you now have two files to reason about.

## What do the directives actually mean?

The man page lists a lot of them. These are the ones that show up in nearly every rule, grouped by the decision each one makes.
