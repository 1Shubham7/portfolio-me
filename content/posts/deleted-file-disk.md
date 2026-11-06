---
title: "I deleted a huge file but the disk usage didn't go down."
description: "Why rm on an open file frees nothing: a file is an inode, a name is only a link to it, and the blocks stay allocated until the link count is zero and every file descriptor is closed. Where df and du part ways, how to find the process holding the file, how to make it let go, and what the same problem looks like on a Kubernetes node."
dateString: November 2026
draft: false
tags: ["Linux", "Filesystems", "SRE", "DevOps", "Kubernetes", "Troubleshooting"]
weight: 2
cover:
    image: "/blog/deleted-file-disk/deleted-file-disk.png"
---


The sequence is familiar to anyone who has run a Linux box for long enough. `df -h` says the root filesystem is at 95%. `du` finds a log file the size of a small database, you `rm` it, you run `df` again, and the number has not moved. Run `du` now and it adds up to 60%. The two tools disagree by a third of the disk, and the file you just deleted is nowhere.

I knew the sequence long before I knew the reason, and the reason is one sentence: `rm` removes a name, not a file. Everything else in this post follows from that.

## A file is an inode, a name is a link to it

On a Linux filesystem the thing you think of as a file is two separate things. The inode holds the data blocks and the metadata: size, owner, permissions, timestamps, a link count, and the pointers (or extents) to where the data lives. The filename is not in there. Names live in directories, and a directory entry is nothing more than a name pointing at an inode number. `ls -li` shows both the number and, in the column after the permissions, the link count: how many names point at that inode.

```bash
$ ls -li /var/log/app.log
1572893 -rw-r--r-- 1 app app 31457280000 Nov  5 09:12 /var/log/app.log
```

`rm` calls `unlink()`, and the man page is exact about what that does. It removes the name and decrements the link count. If that was the last name and no process has the file open, the file is deleted and its space is made available. If any process still has it open, the file remains in existence until the last file descriptor referring to it is closed. Two counters have to reach zero: names, and open descriptors across every process on the machine. `rm` only touches the first.

So a process that opened the log before you deleted it still holds a working descriptor to a perfectly good inode. It keeps appending, the inode keeps acquiring blocks, and the filesystem keeps counting them. The only thing that changed is that no path leads there any more.

This is deliberate. Decoupling names from data is what lets a package manager replace a running binary on disk while the old copy keeps running from its now-nameless inode, and it is how temporary files have always worked: open, unlink, keep using, and the kernel cleans up when the process exits. Windows takes the other route and locks files while they are open.

## Why df and du disagree

They measure different things. `du` walks the directory tree, so it can only count files that have a name. `df` asks the filesystem itself, through `statfs`, and the filesystem answers from the free-block counters in its superblock, which know nothing about names. A deleted-but-open file has no name and still has blocks, so `du` cannot see it and `df` cannot miss it.

That is the signature. `df` at 95% and `du` adding up to 60% with nothing else to explain the gap means something is holding a deleted file open, and the gap is roughly how much.

## Finding what is holding it

`lsof` can list open files whose link count is below one, which is exactly the set of files that have been unlinked:

```bash
$ sudo lsof +L1
COMMAND  PID USER  FD   TYPE DEVICE    SIZE/OFF NLINK    NODE NAME
app     4211  app   3w   REG  259,2 31457280000     0 1572893 /var/log/app.log (deleted)
```

PID, descriptor number, the size still held, and the path it was opened under. `lsof -nP | grep '(deleted)'` gets you the same rows. On a minimal host without `lsof`, go straight to where `lsof` reads it from:

```bash
$ sudo find /proc/*/fd -ls 2>/dev/null | grep '(deleted)'
 38817  0 l-wx------  1 app  app  64 Nov  5 09:12 /proc/4211/fd/3 -> /var/log/app.log (deleted)
```

Each entry in `/proc/<pid>/fd/` is a symlink to whatever the descriptor points at, and the kernel appends ` (deleted)` to the target when the name has been unlinked. That suffix is the whole trick behind both commands.

## Making the process let go

In order of preference.

Tell the process to reopen its log. Most daemons reopen on a signal; `SIGHUP` is the convention, and nginx uses `SIGUSR1`, with [its docs](https://nginx.org/en/docs/control.html) describing the master process re-opening every log file and then asking the workers to do the same. The old descriptor closes, the second counter reaches zero, and the blocks are freed. Nothing restarts.

Restart it. Every descriptor closes with the process. Blunter, and fine when a restart is cheap.

If you cannot restart it, truncate the file through the descriptor:
