---
title: "ShikataLab - Backups: a Bucket, a Database Filesystem, and a Binlog Surprise"
date: 2026-10-20T09:00:00-04:00
tags:
    - homelab
    - kubernetes
    - backups
---

For a long time, my homelab backup strategy was "nothing here is irreplaceable," which is a fine strategy right up until the day it isn't. In my case the day came during a routine rebuild: a PersistentVolumeClaim got recreated, the volume underneath it went away with it, and my web analytics database — *years* of history — was just gone. Kubernetes defaults did exactly what they're documented to do. The PVC had a Delete reclaim policy, the storage class agreed, and the data left. Nobody was negligent. The system worked as configured, which was the problem: it had never occurred to me to configure it.

So: backups. With a management UI, because a backup system I can't glance at from my phone is a backup system I'll never check.

One constraint up front, because it explains every weird decision below: **the budget is $0.** This homelab runs on free tiers and scavenged parts — Oracle's Always Free instances, Backblaze's free 10 GB, a Raspberry Pi doing DNS. Half the fun is Frankensteining things together that were never meant to be storage and seeing how far they go. Sometimes the answer is "surprisingly far." Sometimes it's "into a wall, at speed."

## The shape of it

The cluster is a single k3s node, and most app data lives in local-path volumes on that node's disk. That makes the backup itself simple: one pod running [Backrest](https://github.com/garethgeorge/backrest) — a web UI over restic — mounts the node's volume directory and snapshots the whole thing, about 6 GB all in:

```yaml
image: garethgeorge/backrest:v1.14.1
env:
  - { name: TZ, value: America/Toronto }
  - { name: BACKREST_PORT, value: ":9898" }
volumeMounts:
  - { name: pvdata,  mountPath: /pvdata }        # the node's local-path volumes, read in place
  - { name: b2,      mountPath: /mnt/b2 }         # rclone mount of the Backblaze bucket
  - { name: mysqlfs, mountPath: /mnt/mysqlfs }    # the MySQL filesystem (more below)
  - { name: config,  mountPath: /config }        # Backrest's own config, on a PVC
```

The interesting part was never the backup. It was where to put it, because I had two "free" targets lying around, and they turned out to have very different personalities. Target one: **Backblaze B2** — 10 free gigabytes, S3-compatible, mounted with rclone. Target two: **Oracle's Always Free MySQL** — which, long story (earlier post), I run a FUSE filesystem on top of, giving me a 50 GB database pretending to be a disk:

```ini
# /etc/systemd/system/mysqlfs.service — a MySQL database, mounted like a folder.
# The password lives in ~/.my.cnf under a [mysqlfs] section, not here.
ExecStart=/usr/local/bin/mysqlfs -oallow_other -odefault_permissions \
    -ohost=10.0.1.170 -odatabase=mysqlfs -ouser=mysqlfs -obig_writes /mnt/mysqlfs
```

Both were already plumbed into the cluster as storage classes. Both looked like free backup space.

## B2: the straightforward one, with one sharp edge

The full nightly backup goes to B2: everything, every night at 3 AM Toronto time, keeping 7 daily, 4 weekly, and 3 monthly snapshots. Restic over an rclone mount is not fast, but 6 GB is 6 GB; it finished in minutes and has been boring ever since, which is the highest praise a backup can earn.

The sharp edge: B2 keeps old *versions* of files by default. Restic constantly rewrites small files in its repository, and every rewrite quietly kept the previous version around. A few days in, the bucket hit 100% of the free tier — half of it ghosts. The fix is a bucket lifecycle rule (keep only the last version), but it doesn't apply retroactively, so cleanup meant deleting 285 old versions — 4.54 GB — from the command line with rclone. Set the lifecycle rule *first*. That's the whole lesson of this section and it cost me an afternoon.

## MySQL: the clever idea that filled a database with its own diary

The MySQL filesystem felt like the better target on paper. Fifty gigabytes! Already mounted! Transactions underneath!

Here's what I failed to think through: a filesystem implemented in MySQL tables doesn't just store your writes — it *journals* them. MySQL's binary log records every change so it can be replayed later:

```
mysql> SHOW GLOBAL VARIABLES WHERE Variable_name IN ('log_bin','binlog_expire_logs_seconds');
+----------------------------+-------+
| log_bin                    | ON    |
| binlog_expire_logs_seconds | 3600  |
+----------------------------+-------+
```

Restic is a worst-case client for this. A backup is thousands of tiny files: constant small writes, deletes, rewrites. Each one lands in the tables *and* in the binlog. The binlogs grew roughly ten times faster than the data they described, and an hour of expiry couldn't drain them. MySQL's own error log tells it plainly — the datadir filesystem filled to 91% (46,627,625,728 of 51,044,687,872 bytes), InnoDB started reporting low disk space, and then the instance declared itself out of space and went offline. This database also serves as a storage backend for live cluster workloads, so the blast radius was not theoretical. It happened twice, because the first time I diagnosed it as a fluke.

![Chart: restic writes ~6 GB while MySQL disk usage climbs to 91% and dies; the hourly DB-only plan is flat](/assets/2026-20-10-shikata-backups/binlog-chart.png)

*Reconstructed from the incident — 6 GB in, ~45 GB of disk gone. The shape is the point.*

The workaround was not a setting. I looked: `binlog_expire_logs_seconds` at 3600 was already aggressive, and Oracle's free tier manages the instance — you don't get to just turn replication logging off and call it a day. The actual fix was to stop feeding it the workload it was choking on. More on that in a second.

There was a bonus round during recovery: after wiping the database, only the *initial* schema got reapplied, and mysqlfs quietly needs all ten of its incremental update scripts (`initial_schema.sql`, then `00000000.sql` through `00000009.sql`) on top. Miss them and every filesystem write fails — missing columns, a missing statistics table — in a way that looks exactly like a corrupted backup repository for an embarrassing length of time.

Databases are wonderful things. Their change logs are not your friend when your "files" are someone else's backup chunks.

## Where it landed

![Diagram of the final backup design: Backrest pod snapshots /pvdata to B2 nightly and critical databases to the MySQL filesystem hourly, with failure alerts to Gotify and Discord](/assets/2026-20-10-shikata-backups/backup-flow.svg)

*The final design. Everything important in two places, nothing important in one.*

Split the targets by write pattern:

- **Everything → B2, nightly.** The full ~6 GB `/pvdata` snapshot at 3 AM, retention 7 daily / 4 weekly / 3 monthly. Object storage is built for exactly this: write blobs, keep versions under control, go home.
- **Just the databases that matter → MySQL, hourly.** Immich's Postgres (~334 MB), Rybbit's Postgres (~158 MB), Readarr (~89 MB) — call it 580 MB that is small, precious, and changes constantly. At that volume the hourly cadence is worth it and the binlogs stay a rounding error. Backrest's own config goes along too, so the backup system can restore itself.
- **Both plans alert on failure**, to a self-hosted Gotify instance (on the separate little VM, so it survives the cluster being down) *and* Discord. An unmonitored backup is a hope, not a backup.
- **Belt and suspenders underneath it all:** every volume's reclaim policy is now Retain, and the four PVCs I actually care about carry a `kustomize.toolkit.fluxcd.io/prune: disabled` annotation so GitOps pruning can't delete them either. Backups are the second line of defense. The first is not deleting the data in the first place.

And the total is still $0 — which, again, was the point. Not "cheap." *Free*, assembled from a database pretending to be a disk, a storage bucket's free tier, and mount daemons held together by systemd. Frankensteined, but honest work.

The lesson I'd stencil on the wall: **match the backup target to the write pattern, and know your storage's failure modes before your data does.** A backup target is itself a workload. Treat it like one — give it an error budget, watch its disk, and read its error log before it reads you the riot act.

*Drafted with Jett's help, like the last one; edited by me. The binlogs, regrettably, were real.*
