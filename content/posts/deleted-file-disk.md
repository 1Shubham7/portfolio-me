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
