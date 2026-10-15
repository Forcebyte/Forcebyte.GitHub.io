---
title: "ShikataLab v2.5 - Putting the free-tier box on a diet"
date: 2026-10-15T14:00:00-05:00
tags:
    - homelab
    - oci
    - k3s
draft: true
---

My homelab lives on a free OCI Ampere instance with 2 vCPUs, and for the last few months it's been sitting at a load average between 4 and 10. Sustained. The kind of load where everything technically works, but `kubectl` takes a second to think about it, and you start wondering which one of your containers is going to fall over first.

I finally got annoyed enough to spend a couple of evenings on it. This is what I did, roughly in the order I did it.

## Finding the hog

First thing I did was look at what was actually eating CPU, and the answer was a little embarrassing: Datadog's system-probe, sitting at around 26% on its own, just polling eBPF things in a loop. On a 2-vCPU box, that's more than a quarter of the machine gone to watching the machine.

I didn't uninstall it — I still want the capability around for the next time I need to dig into something weird — but I trimmed the system-probe modules down to the ones I might actually use someday. There was also a brief moment of confusion about how many agents were running, because I kept seeing old-style `/opt/datadog-agent` paths. Turns out those are just processes inside the DaemonSet pod, not a second host install. One Fleet install on the host, one DaemonSet pod in k3s. Fine.

## The service massacre

Then I went through both hosts and turned off everything that didn't need to be on. On azamilab0: fwupd, iscsid, multipathd, udisks2, rpcbind, the serial getty, postfix — all disabled. snapd got purged along with every snap it was holding (the lxd, helm, and kubectx snaps), and I replaced helm and kubectx with standalone ARM64 binaries in /usr/local/bin. The standalone docker.io and containerd packages got purged too, since k3s ships its own containerd and they were just sitting there taking up space. aide got purged. And /var/log went from 4.9G to 2.2G with a manual rotation, which felt great for about ten seconds.

The funny one was packagekit. Masking it made every `apt update` print a GDBus error about the masked unit, because of the hook it leaves in /etc/apt/apt.conf.d/20packagekit. Purging packagekit outright would have dragged software-properties-common and the ubuntu-server metapackage out with it, which felt like too much collateral damage for the sake of tidiness. So I just deleted the hook file. `apt update` runs clean now, and I got to feel clever about it.

shikatalab0 is the little 1GB x86 box that mostly runs NFS these days. CrowdSec and its firewall bouncer came off, the canonical-livepatch snap came off, AdGuardHome came off (DNS there is Tailscale-managed anyway, so nothing noticed). I put fail2ban on the SSH port, capped journald at 50M, and added 2GB of swap, because 1GB of RAM in 2026 is a joke and the OOM killer was starting to get ideas.

End result on azamilab0: 37G used, 112G free, and the load is merely bad instead of catastrophic.

## k3s tuning

Two small things for k3s itself. I added a weekly image prune timer (Sundays at 03:00 UTC, persistent so it catches up if the box was down) so old container images stop piling up forever, and I lowered the kubelet's image garbage collection thresholds to 75/65 in /etc/rancher/k3s/config.yaml. That second one needs a k3s restart to take effect, and I am not restarting k3s while I'm traveling, so it's sitting there staged for whenever I'm home and feeling brave.

## Storage, the cursed part

Separate from the diet: I had two free OCI resources sitting idle and it was bothering me. A MySQL HeatWave instance with 50GB of storage doing absolutely nothing, and 10GB of Object Storage with one lonely qcow2 in it. Neither of them is a filesystem, which is the interesting part.

For MySQL, there's this thing called mysqlfs — a FUSE filesystem that stores files as rows in a MySQL database. I built v1.0.4 for ARM64 from source, set up a systemd mount at /mnt/mysqlfs backed by the HeatWave instance, and got about 3MB/s on writes in testing. It's slow, and I wouldn't keep anything I actually care about on it as the only copy, but for cold storage it's free real estate. The gotchas, for anyone else considering this particular bad idea: the mountpoint has to be empty and owned by the mounting user, you need `user_allow_other` in /etc/fuse.conf, the credentials live in a 600-mode file and nowhere else, and do not NFS-export a FUSE mount. The kernel does not enjoy that.

For Object Storage, it's rclone with a VFS cache, mounted at /mnt/oci-bucket. Good for backups and media, terrible for lots of small random writes, which is fine because that's not what it's for.

Then, because I apparently can't leave well enough alone, I wrote two tiny local-path provisioner deployments following the same pattern as k3s's built-in one, exposing both mounts as StorageClasses: `local-path-mysqlfs` and `local-path-oci`. So now I can claim a dynamic PVC in Kubernetes and have it land in a MySQL database or an S3-compatible bucket. This is deeply silly and I'm a little proud of it.

![Storage on azamilab0](/assets/2026-15-10-shikata-slimdown/storage-arch.svg)

*Diagram generated with AI assistance.*

## Datadog on a diet, part two

A while later I looked at the in-cluster Datadog again and realized it was running a cluster agent. The cluster agent exists to fan out to many node agents and run cluster-level checks. I have one node. So I removed it — about 11 millicores and 140MB of RAM back, which on this box is real money. The DaemonSet stayed, since that's what does the container log collection and the vulnerability scanning that actually sends me useful emails.

While I was in there I found two near-duplicate DatadogAgent configs in the git repo. Turns out one of them wasn't referenced by any Kustomization and had never been applied to the cluster at all — it was just sitting there, waiting to confuse future me. Deleted the orphan.

![Where the savings came from](/assets/2026-15-10-shikata-slimdown/diet-savings.svg)

*Diagram generated with AI assistance.*

## The Audiobookshelf scare

Somewhere in the middle of all this I found four Audiobookshelf PVCs stuck in Terminating since late September, with the pod still running and holding 2.2GB of data. If that pod had ever restarted, it would never have come back — you can't remount a Terminating PVC. Backed the data up from the running pod, deleted everything, recreated the PVCs, restored from backup. All fine in the end, but it was an unpleasant twenty minutes, and those PVCs are still not in git, which I should fix.

## What's next

The box is breathing again. Still to do: restart k3s when I'm home so the GC thresholds actually kick in, build the actual backup solution (the important one, and half the reason I wired up the object storage in the first place), and eventually write an Ansible playbook for all of this host config so it stops living exclusively in my head. That last one is a when-I'm-home project — I'm not testing a bootstrap playbook against the live box from a hotel room.
