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

