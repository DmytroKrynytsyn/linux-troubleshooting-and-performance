# Disk & Storage Troubleshooting

Disk problems on EC2 split into two categories that need completely different tools: **"the volume is full"** (a capacity problem, boring but urgent) and **"I/O is slow"** (a performance problem, where EBS's credit and throughput mechanics are usually the actual cause). This guide covers both, plus the two traps that `df -h` alone will never show you: inode exhaustion, and space held by files that no longer exist.

## 1-Minute Summary

- `df -h` **and** `df -i` — space and inodes are separate resources; a disk can have plenty of space and still refuse to create files.
- **`df` and `du` disagree?** A deleted file is still held open by a running process. `lsof +L1`, or `ls -l /proc/*/fd | grep deleted`.
- `cat /proc/pressure/io` tells you stall time, not busy-ness — a device can be 100% utilized and fine, or moderately utilized and genuinely stalling.
- `iostat -xz 1` — read `%util`, `await`, and `aqu-sz` together: high utilization *with* climbing `await`/queue size is real saturation.
- A process in `D` state is stuck in the kernel, almost always on I/O, and can't be killed until the I/O resolves.
- **Grew an EBS volume and `df` didn't change?** Three layers: volume → partition (`growpart`) → filesystem (`xfs_growfs` / `resize2fs`).
- **On EBS `gp2`:** burst balance hitting zero causes a sudden, uniform slowdown with no error message. `gp3` has no credit mechanism at all.

## Methodology

1. **Full or slow?** These are different investigations — don't run `iostat` for a full disk, or `df` for a slow one.
2. **If full: space, inodes, or ghosts?** `df -h` only tells you one of the three.
3. **If slow: is it actually I/O, or something waiting on I/O for an unrelated reason?** PSI for I/O, then `iostat` for device-level truth.
4. **Which process, and is it blocked in the kernel right now?** `D`-state processes are in an uninterruptible wait — usually I/O.
5. **On EBS: are you hitting a *limit* rather than a device?** Volume IOPS/throughput, instance-level EBS bandwidth, or burst balance. All three are invisible from inside the guest.

## Commands, explained

### `df -h` and `df -i` — space *and* inodes are two different resources

```bash
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1   20G   18G  1.1G  95% /
```
```bash
df -i
```
```
Filesystem      Inodes IUsed IFree IUse% Mounted on
/dev/nvme0n1p1   1.3M  1.3M     0  100%  /
```
A filesystem can be nowhere near full on space and still be **completely unable to create a new file** because it's out of inodes — every file, however small, consumes exactly one inode, so a directory full of millions of tiny files (session files, a queue directory nobody's cleaning up, an unrotated maildir) exhausts inodes long before space. `df -h` will tell you the disk is fine while `touch` fails with "No space left on device."

**Distro note:** ext4 fixes the inode count at `mkfs` time and it **cannot be changed** without recreating the filesystem — this is the Ubuntu default and the reason this failure mode is more common there. XFS allocates inodes dynamically and effectively never runs out; it's the Amazon Linux 2/2023 root default. Knowing which filesystem you're on tells you immediately whether `df -i` is even worth checking.

```bash
lsblk -f            # filesystem type, label, UUID, mountpoint — one view
findmnt             # the mount tree, including bind mounts and propagation
```

Finding the offender:
```bash
du -xh --max-depth=1 / | sort -h        # -x stays on one filesystem, won't wander into /proc
find /var -xdev -type f -printf '%s %p\n' | sort -rn | head -20      # biggest files
find /var -xdev -type d -size +1M | head                             # dirs with huge entry counts (inode hunt)
```
`-x` / `-xdev` is the flag that matters: without it, `du /` descends into `/proc`, `/sys`, and every other mounted volume, and gives you a number that means nothing.

### When `df` and `du` disagree — deleted but open files

This is the single most common "the disk is full and I can't find why" incident, and the fastest to fix once you recognise it.

**Symptom:** `df -h` says 100% used; `du -xsh /` accounts for 40%.

A file that's been `rm`'d is unlinked from the directory tree — so `du`, which walks the directory tree, can't see it. But its inode and data blocks are not freed until the **last open file descriptor closes**. A logfile rotated with `rm` instead of `copytruncate`, while the application still holds it open and keeps appending, will consume disk forever and be invisible to `du`.

```bash
lsof +L1                        # files whose link count is < 1: deleted but still open
lsof -nP | grep '(deleted)'
```
```
COMMAND  PID USER  FD  TYPE DEVICE   SIZE/OFF NLINK NODE NAME
java    4821 app   7w  REG  259,1  8589934592     0 1234 /var/log/app/app.log (deleted)
```
`NLINK 0` and 8 GB of size is the whole answer.

**`/proc` alternative — works with no `lsof` installed, which matters on a minimal AMI:**
```bash
ls -l /proc/*/fd 2>/dev/null | grep '(deleted)'
# narrow to one process:
ls -l /proc/4821/fd | grep deleted
```

**Fix without restarting the process.** You can write through the file descriptor even though the path is gone:
```bash
: > /proc/4821/fd/7
```
This truncates the file to zero and returns the blocks immediately, while the process keeps its fd and keeps logging. Restarting the service also works and is often simpler — but on a service you can't bounce mid-incident, the `/proc/PID/fd` trick is the one that gets you out of trouble.

**Prevention:** `logrotate` with `copytruncate`, or a `postrotate` script that signals the app to reopen (`kill -HUP`, `systemctl reload`). Never `rm` an active logfile.

**The other cause of a `df`/`du` mismatch: files hidden under a mountpoint.** If something wrote to `/data` *before* the volume was mounted there, those files still exist on the root filesystem, consuming space, invisible under the mount.
```bash
mkdir -p /mnt/root-view
mount --bind / /mnt/root-view
du -xh --max-depth=1 /mnt/root-view/data
umount /mnt/root-view
```

### PSI and `/proc/diskstats` — is this actually I/O

```bash
cat /proc/pressure/io
```
```
some avg10=34.20 avg60=12.10 avg300=4.02
full avg10=18.10 avg60=6.02 avg300=1.11
```
Same pattern as CPU/memory PSI: stall time, not utilization. A device can show 100% utilization in `iostat` while serving requests fast enough that nothing stalls (fine), or show moderate utilization while stalling badly (not fine — typically many small random I/Os). PSI tells you which situation you're in without reading tea leaves from `iostat`.

Not present on Amazon Linux 2 (4.14 kernel). Fallback: `await` climbing in `iostat`, plus a rising count of `D`-state processes.

```bash
cat /proc/diskstats
```
The raw counters `iostat` is computed from. The fields that matter, per device: 4 = reads completed, 6 = sectors read, 8 = writes completed, 10 = sectors written, **12 = I/Os currently in flight**, 13 = ms spent doing I/O, **14 = weighted ms spent doing I/O** (this is what `%util` and `aqu-sz` are derived from). Field 12 being persistently non-zero is a queue that never drains.

### `iostat -xz 1` — the workhorse

```bash
iostat -xz 1
```
```
Device            r/s     w/s   rkB/s     wkB/s  await  r_await  w_await  aqu-sz  %util
nvme0n1         12.00  850.00   48.00 108800.00  42.30     2.10    45.80   18.40   98.50
```
Read device-by-device, and read three columns together:

- `%util` near 100 with `await` still low → the device is busy but keeping up. **Fine.**
- `%util` near 100 **and** `await` climbing → requests are queueing faster than the device drains them. **Genuine saturation.**
- `aqu-sz` rising alongside `await` confirms it — the queue is backing up, not just one slow outlier.

For EBS on NVMe, healthy `await` is low single-digit milliseconds. 42ms means something is capping you.

> **`%util` is misleading on NVMe.** It measures "time at least one request was in flight," which on a device with deep parallelism can hit 100% at a fraction of real capacity. On NVMe-backed EBS, treat `await` and `aqu-sz` as the primary signals and `%util` as supporting evidence only. This trips up people carrying habits from spinning disks.

**Version note:** the column is `aqu-sz` in sysstat 12+ (AL2023, Ubuntu 22.04+) and `avgqu-sz` in sysstat 11 (Amazon Linux 2, Ubuntu 18.04). Same number, different header — worth knowing before you write a parser.

```bash
iostat -xmd 1                     # megabytes instead of kilobytes
iostat -xz 1 | grep -v ' 0.00 '   # -z already suppresses idle devices
```

### Who's actually blocked

```bash
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'
```
```
 4102 D    wait_on_page_bit    postgres
```
A process in `D` state (uninterruptible sleep) is waiting on the kernel — almost always I/O — and **cannot be killed** with a normal signal until the I/O completes or times out. That's itself diagnostic: a `kill -9` that does nothing confirms a real kernel-level I/O stall rather than a hung userspace process.

`wchan` names the kernel function it's parked in. `wait_on_page_bit` = waiting for a page read. `nfs_wait_bit_killable` or anything `rpc_*` = a hung NFS/EFS mount, which is an entirely different problem from a slow local disk.

**`/proc` alternatives:**
```bash
cat /proc/PID/io          # read_bytes / write_bytes — confirms WHICH process generates iostat's load
cat /proc/PID/stack       # exact kernel stack — where it's stuck
cat /proc/PID/wchan       # the single function, in one word
```
`/proc/PID/io`'s `read_bytes`/`write_bytes` count actual block-device traffic (unlike `rchar`/`wchar`, which count syscall bytes including cache hits). Diff it over 10 seconds and you have a per-process I/O rate with no tooling at all.

```bash
pidstat -d 1                # per-process I/O over time
iotop -oPa                  # -o only active, -P processes not threads, -a accumulated
biolatency-bpfcc 10 1       # latency HISTOGRAM — shows the distribution iostat's average hides
biosnoop-bpfcc              # per-I/O trace with PID and file — "who did this specific slow read"
```
The histogram matters more than it sounds. An `await` average of 10ms could be everything at 10ms (a uniformly slow device) or 95% at 1ms and 5% at 200ms (a tail-latency problem hitting your p99). Those need completely different fixes and `iostat` cannot tell them apart.

### Growing a volume — the three-layer problem

You expanded the EBS volume in the console and `df` still shows the old size. That's expected: **three independent layers each need growing.**

```bash
# 1. Does the kernel see the new device size?
lsblk
```
```
NAME          SIZE
nvme0n1       200G          ← new size, good
└─nvme0n1p1   100G          ← partition still old
```
If `lsblk` still shows the *old* device size, the kernel hasn't rescanned:
```bash
echo 1 > /sys/class/block/nvme0n1/device/rescan_controller     # NVMe (Nitro)
echo 1 > /sys/class/block/xvdf/device/rescan                   # Xen
```

```bash
# 2. Grow the partition — note the SPACE before the partition number
growpart /dev/nvme0n1 1
```

```bash
# 3. Grow the filesystem — this one is filesystem-specific, and both can run ONLINE
xfs_growfs -d /                    # XFS: takes the MOUNTPOINT
resize2fs /dev/nvme0n1p1           # ext4: takes the DEVICE
```
The argument type differing between the two is a genuinely common mistake under pressure. **XFS takes a mountpoint, ext4 takes a device.**

**Which will you be on?**

| AMI | Root filesystem |
|---|---|
| Amazon Linux 2 / 2023 | XFS |
| RHEL 8/9, Fedora | XFS |
| Ubuntu cloud images | ext4 |
| Debian cloud images | ext4 |

**XFS cannot shrink. Ever.** If you over-provisioned, the only path is create a smaller volume, copy, swap. Say this out loud in a design review before someone provisions a 16 TB volume "to be safe."

**Install:** `dnf install cloud-utils-growpart xfsprogs e2fsprogs` / `apt install cloud-guest-utils xfsprogs e2fsprogs`

### Device-level configuration

```bash
cat /sys/block/nvme0n1/queue/scheduler          # [none] is correct for NVMe — don't "fix" it
cat /sys/block/nvme0n1/queue/nr_requests
cat /sys/block/nvme0n1/queue/read_ahead_kb      # default 128; raise for large sequential reads
cat /proc/sys/vm/dirty_ratio /proc/sys/vm/dirty_background_ratio
```
`dirty_ratio` / `dirty_background_ratio` control how much written-but-not-flushed data the kernel lets accumulate in page cache before forcing writes. A workload with heavy, bursty writes hitting these thresholds shows up as sudden `await` spikes that look like "the disk got slow" but are actually "the kernel just started forcing a large flush it had been deferring." On a large-memory instance the default percentage-based limits can represent tens of gigabytes of dirty pages — the `dirty_bytes`/`dirty_background_bytes` variants let you cap it absolutely instead.

### Filesystem repair and integrity

```bash
xfs_repair -n /dev/nvme1n1p1     # -n = dry run. MUST be unmounted. XFS has no online fsck
fsck -n /dev/nvme1n1p1           # ext4, also unmounted
xfs_db -c frag -r /dev/nvme1n1p1 # fragmentation report
tune2fs -l /dev/nvme1n1p1        # ext4 superblock: mount count, check interval, features
xfs_info /mnt/data               # XFS geometry
```
Never run a repair on a mounted filesystem. On EC2, the practical sequence is: stop the instance, detach the volume, attach it to a rescue instance, repair there. See [recovery.md](recovery.md).

### Install

| Tool | RHEL-family (`dnf`) | Debian-family (`apt`) |
|---|---|---|
| `iostat`, `pidstat -d`, `sar -d` | `sysstat` | `sysstat` |
| `iotop` | `iotop` | `iotop` |
| `lsof` | `lsof` | `lsof` |
| `fio` (benchmark/reproduce) | `fio` | `fio` |
| `growpart` | `cloud-utils-growpart` | `cloud-guest-utils` ← **name differs** |
| XFS tools | `xfsprogs` | `xfsprogs` |
| ext4 tools | `e2fsprogs` | `e2fsprogs` |
| LVM | `lvm2` | `lvm2` |
| eBPF `biolatency`, `biosnoop` | `bcc-tools` | `bpfcc-tools` |
| `smartctl` (instance-store/bare metal only) | `smartmontools` | `smartmontools` |
| `nvme` CLI | `nvme-cli` | `nvme-cli` |

## AWS-specific gotchas

### NVMe device naming does not match what you asked for

On Nitro instances, an EBS volume you attached as `/dev/sdf` appears in the guest as `/dev/nvme1n1` — and **the numbering is not guaranteed stable across reboots**, because it follows enumeration order, not your attachment order. Mounting by device path in `/etc/fstab` is how instances fail to boot after a reboot.

Map a device back to its requested name:
```bash
# Amazon Linux (ships the helper):
/sbin/ebsnvme-id /dev/nvme1n1

# Anywhere with nvme-cli:
nvme id-ctrl -v /dev/nvme1n1 | grep -A1 '^0000'
```
Amazon Linux and recent Ubuntu images also ship udev rules that create `/dev/sdf` symlinks pointing at the right NVMe device — but don't rely on their presence in a custom AMI.

**Always mount by UUID.** Always add `nofail`:
```
UUID=6d9e...  /data  xfs  defaults,nofail,x-systemd.device-timeout=5  0 2
```
Without `nofail`, a volume that fails to attach drops the instance into emergency mode at boot with no SSH — see [recovery.md](recovery.md) for how much worse that day gets.

### `gp2` burst balance runs out, and then everything gets slow at once

`gp2` volumes earn I/O credits and burst above baseline using a credit balance, exactly like `t3` CPU credits.

- Baseline: **3 IOPS per GiB**, minimum 100
- Burst: up to **3000 IOPS**, funded by credits
- Volumes **≥ 1000 GiB** have a baseline of 3000+ and never burst at all

So a 100 GiB `gp2` volume has a 300 IOPS baseline; a 500 GiB volume has 1500. When the balance hits zero, IOPS are hard-capped to baseline with **no error and no warning in `iostat` beyond `await` climbing**. Check CloudWatch `BurstBalance` on the volume before chasing an application-side cause.

**`gp3` has no credit mechanism** — a flat 3000 IOPS / 125 MiB/s baseline, with IOPS and throughput independently provisionable. If burst exhaustion is a recurring incident, `gp2` → `gp3` is an online modification and usually resolves it outright. It's also typically cheaper.

### The instance has its own EBS bandwidth limit, separate from the volume's

This is the one people miss. Each instance type has a maximum aggregate EBS throughput and IOPS. A `m5.large` caps around 4750 Mbps of EBS bandwidth regardless of how many `io2` volumes you attach. If `iostat` shows you plateauing well below the sum of your volumes' provisioned performance, you're hitting the **instance** ceiling, not the volume's — the fix is a bigger instance, not a faster volume. Smaller instance types also have *burstable* EBS bandwidth with the same credit-exhaustion behaviour as CPU.

### Other traps

- **Instance-store (ephemeral NVMe) is physically fast but disappears on stop/terminate.** If `lsblk` shows a fast local NVMe device not present in `blkid`/`fstab`, someone put transient data on it. That's a durability conversation, not a performance one.
- **EFS adds a network round trip to every I/O.** `iostat`'s `await` on an EFS mount includes network latency, so single-digit milliseconds isn't the bar. Compare against EFS CloudWatch metrics, not local NVMe. A hung EFS mount shows as `D`-state processes with `rpc_*` in `wchan`.
- **Newly restored volumes from a snapshot are slow on first read.** Blocks are lazily loaded from S3 on first access. A benchmark run immediately after restore measures S3 hydration, not EBS. Use Fast Snapshot Restore, or pre-warm with `fio --rw=read`, before drawing conclusions.

## Worked example: "Batch job that ran fine all month suddenly takes 4x longer"

**Symptom:** a nightly ETL job on a `gp2`-backed instance, normally 40 minutes, took 3 hours last night. No code changes, no deploy.

**Step 1 — rule out a full disk first (cheap, and eliminates a whole class of cause):**
```bash
df -h /data ; df -i /data
```
```
/dev/nvme1n1   500G  210G  290G  43% /data
/dev/nvme1n1   262M  1.1M  261M   1% /data
```
Plenty of both. Not a capacity problem.

**Step 2 — check for saturation during the slow run:**
```bash
iostat -xz 1
```
```
Device     r/s     w/s   rkB/s   wkB/s   await  aqu-sz  %util
nvme1n1   295.0   180.0  4200.0  9800.0  185.40   22.10   99.80
```
`await` at 185ms against a normal night's ~8ms baseline for this workload, with `aqu-sz` at 22 — genuine, severe saturation. Combined IOPS is ~475, which is well *under* what this volume should sustain, so the device isn't being asked to do more than usual. Something has lowered the ceiling.

**Step 3 — confirm the shape with a histogram, to rule out a tail-latency artefact:**
```bash
biolatency-bpfcc 10 1
```
Uniform distribution centred around 180ms, not a bimodal split. Everything is slow, not a slow minority — consistent with a hard rate cap rather than a few pathological requests.

**Step 4 — this is `gp2`, so check burst balance before assuming an application regression.**

CloudWatch → `BurstBalance` for the volume: 0% for the entire run, having been at ~40% at the same point the previous night. This is a 500 GiB `gp2` volume, so its baseline is 1500 IOPS and it bursts to 3000. The job's I/O pattern had been fitting inside a night's available credits; dataset growth pushed it past, and mid-job the volume fell back to 1500 baseline while the job's random-read pattern needed sustained ~2800.

**Root cause:** dataset growth pushed the job's sustained I/O demand above what burst credits could fund for the job's full duration. The volume dropped to baseline part-way through, and everything after that ran at roughly half the required rate.

**Fix (immediate):** migrate to `gp3` (online modification, no downtime) and provision IOPS and throughput to the job's measured requirement plus headroom — measured from `iostat` during a healthy run, not guessed.
**Also verify:** that the instance type's aggregate EBS bandwidth can actually deliver the newly provisioned volume performance. Provisioning 16,000 IOPS on a volume attached to an instance capped at 4750 Mbps buys nothing.
**Prevention:** alarm on `BurstBalance` trending toward zero, and size volumes from measured baseline I/O needs rather than capacity alone — a volume can have plenty of space and still be badly undersized for IOPS. Longer term, `gp2` in a production fleet is a standing liability; `gp3` removes the failure mode entirely.

## Cheat sheet

```bash
# Full?
df -h ; df -i                                   # space AND inodes (ext4 fixed at mkfs; XFS dynamic)
du -xh --max-depth=1 / | sort -h                # -x = stay on one filesystem
find /var -xdev -type f -printf '%s %p\n' | sort -rn | head -20

# df and du disagree → deleted-but-open file
lsof +L1                                        # link count < 1
ls -l /proc/*/fd 2>/dev/null | grep deleted     # /proc alternative, no lsof needed
: > /proc/PID/fd/N                              # reclaim space WITHOUT restarting the process
mount --bind / /mnt/x && du -xh /mnt/x/data     # or: files hidden under a mountpoint

# Slow?
cat /proc/pressure/io                           # PSI stall time (not on AL2)
iostat -xz 1                                    # await + aqu-sz together; %util lies on NVMe
                                                # aqu-sz (sysstat 12+) == avgqu-sz (sysstat 11)
biolatency-bpfcc 10 1                           # the DISTRIBUTION iostat's average hides
biosnoop-bpfcc                                  # per-I/O with PID and filename
pidstat -d 1 ; iotop -oPa

# Who's blocked
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'  # D state = uninterruptible, unkillable
cat /proc/PID/io                                # read_bytes/write_bytes = real block I/O
cat /proc/PID/stack ; cat /proc/PID/wchan       # exactly where in the kernel
cat /proc/diskstats                             # field 12 = in-flight, 14 = weighted ms

# Grew the volume, df unchanged → three layers
lsblk                                           # 1. does the kernel see the new size?
echo 1 > /sys/class/block/nvme0n1/device/rescan_controller
growpart /dev/nvme0n1 1                         # 2. partition (note the SPACE)
xfs_growfs -d /                                 # 3a. XFS takes a MOUNTPOINT
resize2fs /dev/nvme0n1p1                        # 3b. ext4 takes a DEVICE
# XFS cannot shrink, ever.

# Device config
cat /sys/block/nvme0n1/queue/{scheduler,nr_requests,read_ahead_kb}
cat /proc/sys/vm/dirty_ratio /proc/sys/vm/dirty_background_ratio

# Repair (UNMOUNTED only — no online fsck for XFS)
xfs_repair -n /dev/nvme1n1p1 ; fsck -n /dev/nvme1n1p1
xfs_info /mnt/data ; tune2fs -l /dev/nvme1n1p1

# NVMe naming
/sbin/ebsnvme-id /dev/nvme1n1                   # Amazon Linux
nvme id-ctrl -v /dev/nvme1n1 | grep -A1 '^0000'
# ALWAYS fstab by UUID, ALWAYS with nofail

# Install
dnf install sysstat iotop lsof fio cloud-utils-growpart xfsprogs e2fsprogs lvm2 bcc-tools nvme-cli
apt  install sysstat iotop lsof fio cloud-guest-utils    xfsprogs e2fsprogs lvm2 bpfcc-tools nvme-cli

# AWS
# gp2 baseline = 3 IOPS/GiB (min 100), bursts to 3000; >=1000 GiB never bursts. Check BurstBalance.
# gp3 = flat 3000/125 MiB/s, no credits, online migration from gp2.
# The INSTANCE has its own EBS bandwidth cap, separate from the volume's provisioned IOPS.
# Snapshot-restored volumes are slow on first read (lazy S3 hydration) — pre-warm before benchmarking.
```