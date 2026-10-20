---
title: "ShikataLab - Backups: a Bucket, a Database Filesystem, and a Binlog Surprise"
date: 2026-10-20T09:00:00-04:00
tags:
    - homelab
    - kubernetes
    - backups
---

For a long time, my homelab backup strategy was "nothing here is irreplaceable," which is a fine strategy right up until the day it isn't. In my case the day came during a routine rebuild: a PersistentVolumeClaim got recreated, the volume underneath it went away with it, and my web analytics database — a few weeks of history — was just gone. Kubernetes defaults did exactly what they're documented to do. The PVC had a Delete reclaim policy, the storage class agreed, and the data left. Nobody was negligent. The system worked as configured, which was the problem: it had never occurred to me to configure it.

So: backups. With a management UI, because a backup system I can't glance at from my phone is a backup system I'll never check.

## The shape of it

The cluster is a single k3s node, and most app data lives in local-path volumes on that node's disk. That makes the backup itself simple: one pod running [Backrest](https://github.com/garethgeorge/backrest) (a web UI over restic) mounts the node's volume directory read-only and snapshots the whole thing — about 6 GB all in. The interesting part was never the backup. It was where to put it, because I had two "free" targets lying around, and they turned out to have very different personalities.

Target one: **Backblaze B2.** Ten free gigabytes, S3-compatible, mounted on the node with rclone. Target two: **Oracle's Always Free MySQL**, which — long story, earlier post — I run a FUSE filesystem on top of (`mysqlfs`), giving me a 50 GB database pretending to be a disk. Both were already plumbed into the cluster as storage. Both looked like free backup space.

## B2: the straightforward one, with one sharp edge

The full nightly backup goes to B2: everything, every night at 3 AM Toronto time, keeping 7 daily, 4 weekly, and 3 monthly snapshots. Restic over an rclone mount is not fast, but 6 GB is 6 GB; it finished in minutes and has been boring ever since, which is the highest praise a backup can earn.

The sharp edge: B2 keeps old *versions* of files by default. Restic constantly rewrites small files in its repository, and every rewrite quietly kept the previous version around. A few days in, the bucket hit 100% of the free tier — half of it ghosts. The fix is a lifecycle rule (keep only the last version), but it doesn't apply retroactively, so cleanup meant deleting 285 old versions, about 4.5 GB, from the command line. Set the lifecycle rule *first*. That's the whole lesson of this section and it cost me an afternoon.

## MySQL: the clever idea that filled a database with its own diary

The MySQL filesystem felt like the better target on paper. Fifty gigabytes! Already mounted! Versioning and transactions underneath!

Here's what I failed to think through: a filesystem implemented in MySQL tables doesn't just store your writes — it *journals* them. Every file write is row changes, and MySQL's binary log records those changes so replicas and point-in-time recovery can replay them. Restic is a worst-case client for this: a backup is thousands of tiny files, constant small writes, deletes, rewrites. The binlogs grew roughly ten times faster than the data they described. Binlog expiry was set to an hour and still couldn't keep up. The 50 GB disk filled to 91%, MySQL declared low disk space, and the database — which also serves as a storage backend for actual cluster workloads — went offline. Twice, because the first time I diagnosed it as a fluke.

There was a bonus round: after wiping the database to recover, only the initial schema got reapplied, and mysqlfs quietly needs *all ten* of its incremental update scripts on top. Without them every write failed with I/O errors, in a way that looked exactly like a corrupt backup repository for an embarrassing length of time.

Databases are wonderful things. Their change logs are not your friend when your "files" are someone else's backup chunks.

## Where it landed

Split the targets by write pattern:

- **Everything → B2, nightly.** The full 6 GB snapshot. Object storage is built for exactly this: write blobs, keep versions under control, go home.
- **Just the databases that matter → MySQL, hourly.** The Immich, Rybbit, and Readarr databases total about 580 MB. Small, precious, and changing constantly — the one workload where an hourly cadence is worth it and the binlogs stay a rounding error. Backrest's own config goes along too, so the backup system can restore itself.
- **Both plans alert on failure**, to a self-hosted Gotify instance (on a separate little VM, so it survives the cluster being down) *and* Discord. An unmonitored backup is a hope, not a backup.
- **Belt and suspenders underneath it all:** every volume's reclaim policy is now Retain, and the four PVCs I actually care about are annotated so GitOps pruning can't delete them. Backups are the second line of defense. The first is not deleting the data in the first place.

The total cost is still $0, which was the actual constraint all along — this is a homelab, the budget is "free tiers and stubbornness." The lesson I'd stencil on the wall: **match the backup target to the write pattern, and know your storage's failure modes before your data does.** A backup target is itself a workload. Treat it like one.

*Drafted with Jett's help, like the last one; edited by me. The binlogs, regrettably, were real.*
