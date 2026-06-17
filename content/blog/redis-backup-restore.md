---
title: "Backing Up and Restoring Redis on Kubernetes"
description: "How to backup and restore Redis on Kubernetes using RDB snapshots - and why just copying dump.rdb is not enough."
dateString: June 2026
draft: false
tags: ["Redis", "Kubernetes", "DevOps", "Bitnami", "Storage"]
weight: 1
cover:
    image: "/blog/redis-backup/redis.png"
---

This doc will help you backup redis, and restore it.

## Important Things to Know

- Always backup from the **master** pod, not replicas
- Redis startup priority: `appendonly.aof` **always wins over** `dump.rdb` - if AOF exists on disk, Redis loads it no matter what
- AOF (Append Only File) logs every write command in real time. On restart, Redis replays all those commands to rebuild the dataset. Think of it as a transaction log
- RDB (`dump.rdb`) is a point-in-time snapshot
- Setting `appendonly no` in config only stops **writing** to AOF - it does NOT stop Redis from loading an existing AOF file on startup. Wasted a lot of time on this one.

---

## Taking a Backup

### 1. Trigger a fresh snapshot on the master

Exec into the master pod and run:

```bash
redis-cli BGSAVE
```

Output:
```
Background saving started
```

### 2. Wait for it to finish

Keep running `LASTSAVE` until the integer changes:

```bash
redis-cli LASTSAVE
```

Output:
```
(integer) 1780038443
```

Convert to human-readable to confirm:
```bash
date -d @1780038443
# Fri May 29 07:07:23 UTC 2026
```

Run `LASTSAVE` again after a few seconds - once the number increases, the snapshot is done:
```
(integer) 1780038606  ← increased, backup complete
```

### 3. Copy dump.rdb to local machine

```bash
kubectl cp <namespace>/<redis-master-pod>:/data/dump.rdb ./redis-backup-$(date +%Y%m%d).rdb
```

### 4. Verify the backup is not empty

```bash
ls -lh redis-backup-20260529.rdb
# -rw-rw-r-- 1 user user 3.6M May 29 12:42 redis-backup-20260529.rdb
```

3.6M, non-zero - good to go.

You can also peek at keys inside:
```bash
strings redis-backup-20260529.rdb | head -50
```

If you see key names in the output, the backup has real data.

---

## Wiping the PVC and Recreating Redis

1. Delete the StatefulSet via your GitOps tool (or `kubectl delete sts`)
2. Delete all Redis PVCs manually:
```bash
kubectl get pvc -n <namespace>
kubectl delete pvc <master-pvc> <replica-pvcs> -n <namespace>
```
3. Recreate - new PVCs and pods come back fresh

---

## Restoring the Backup

After a fresh PVC, Redis immediately creates a new empty `appendonly.aof`. If you just copy `dump.rdb` in and restart, Redis will load the empty AOF and ignore your backup completely. You end up with way fewer keys than expected instead of your full dataset.

**The fix: delete the AOF file first, then copy the backup, then restart.**

### Step 1 - Delete the AOF files inside the pod

```bash
kubectl exec -it <redis-master-pod> -n <namespace> -- rm -f /data/appendonly.aof /data/appendonly.aof.bak
```

### Step 2 - Copy the backup into the pod

```bash
kubectl cp ./redis-backup-20260529.rdb <namespace>/<redis-master-pod>:/data/dump.rdb
```

### Step 3 - Verify the file size matches

```bash
kubectl exec -it <redis-master-pod> -n <namespace> -- ls -lh /data/
```

Expected:
```
total 3.6M
-rw-r--r-- 1 1001 1001 3.6M May 29 09:43 dump.rdb
```

If you see `175 bytes` - Redis already overwrote it with an empty snapshot. Go back to step 1.

### Step 4 - Restart the pod

```bash
kubectl delete pod <redis-master-pod> -n <namespace>
```

### Step 5 - Verify data is restored

```bash
kubectl exec -it <redis-master-pod> -n <namespace> -- redis-cli DBSIZE
# (integer) 35  ← your keys are back

kubectl exec -it <redis-master-pod> -n <namespace> -- redis-cli get <known-key>
# your data is there
```

### Step 6 - Confirm AOF comes back

After the pod restarts with `appendonly yes` in the ConfigMap, Redis will recreate the AOF file automatically:

```bash
kubectl exec -it <redis-master-pod> -n <namespace> -- ls -lh /data/
```

```
-rw-rw-r-- 1 1001 1001  77K May 29 09:01 appendonly.aof
-rw-r--r-- 1 1001 1001 3.6M May 29 09:43 dump.rdb
```

Both files present - persistence is working normally again.

---

## What AOF Actually Does (and Why It Matters)

AOF logs every write command (`SET`, `HSET`, `DEL`, etc.) to a file in real time. When Redis restarts, it replays all those commands from top to bottom to reconstruct the full dataset.

RDB is just a snapshot - it captures the state at a point in time but misses any writes that happened between snapshots.

With both enabled, you get the best of both worlds. Without AOF, if Redis crashes between RDB snapshots, you lose all writes since the last snapshot.
