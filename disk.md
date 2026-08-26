# Disk & Storage Troubleshooting

Disk problems on EC2 split into two categories that need completely different tools: **"the volume is full"** (a capacity problem, boring but urgent) and **"I/O is slow"** (a performance problem, where EBS's credit/burst mechanics are usually the actual cause). This guide covers both, plus the inode-exhaustion trap that `df -h` alone will never show you.

## 1-Minute Summary

- `df -h` **and** `df -i` — space and inodes are separate resources; a disk can have plenty of space and still refuse to create files because inodes ran out.
- `cat /proc/pressure/io` tells you stall time, not busy-ness — a device can be 100% utilized and fine, or moderately utilized and genuinely stalling.
- `iostat -xz 1` — read `%util`, `await`, and `aqu-sz` together: high utilization *with* climbing `await`/queue size is real saturation, not just a busy device keeping up.
- A process in `D` state (`ps -eo pid,stat,wchan,comm | awk '$2 ~ /D/'`) is stuck in the kernel, almost always on I/O, and can't be killed until the I/O resolves.
- **On EBS `gp2`:** burst balance hitting zero causes a sudden, uniform slowdown with no error message — check CloudWatch `BurstBalance` before assuming an application-side cause. `gp3` avoids this entirely.

## Methodology

1. **Full or slow?** These are different investigations — don't run `iostat` for a full disk, or `df` for a slow one.
2. **If full: space or inodes?** `df -h` only tells you one of these.
3. **If slow: is it actually I/O, or something waiting on I/O for an unrelated reason?** PSI for I/O, then `iostat` for the device-level truth.
4. **Which process, and is it blocked in the kernel right now?** `D`-state processes are stuck in an uninterruptible wait — usually I/O.
5. **On EBS specifically:** check burst balance and IOPS/throughput limits before assuming it's an application problem.

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
A filesystem can be nowhere near full on space (`df -h` looks fine) and still be **completely unable to create a new file** because it's out of inodes — every file, however small, consumes exactly one inode, so a directory full of millions of tiny files (logs, session files, a queue directory nobody's cleaning up) exhausts inodes long before it exhausts space. `df -h` alone will tell you the disk is fine while `mkdir`/`touch` fail with "No space left on device" — always check both.

```bash
du -xh --max-depth=1 / | sort -h
```
`-x` stays on one filesystem (won't wander into `/proc`, mounted volumes, etc.) — the fastest way to find *which top-level directory* is actually holding the space, before you go hunting file by file.

```bash
lsblk -f ; blkid
mount | column -t
```
Confirms what's actually mounted where, and with what filesystem — useful when a volume was attached but never mounted, or mounted somewhere unexpected (a common cause of "I attached a bigger EBS volume and `df` still shows the old size").

### PSI and `/proc/diskstats` — is this actually I/O

```bash
cat /proc/pressure/io
```
```
some avg10=34.20 avg60=12.10 avg300=4.02
```
Same pattern as CPU/memory PSI: this is stall time, not utilization — a device can show 100% utilization in `iostat` while serving requests fast enough that nothing stalls (fine), or show moderate utilization while stalling badly (not fine, e.g. many small random I/Os). PSI tells you which situation you're in without reading tea leaves from `iostat` alone.

```bash
cat /proc/diskstats            # raw counters iostat computes its numbers from — useful when scripting your own checks
```

### `iostat -xz 1` — the workhorse

```bash
iostat -xz 1
```
```
Device            r/s     w/s   rkB/s   wkB/s  await  r_await  w_await  aqu-sz  %util
nvme0n1          12.00  850.00   48.00 108800.00  42.30    2.10    45.80    18.40  98.50
```
Read this device-by-device: `%util` near 100 with `await` (average time per I/O, milliseconds) still low means the device is busy but keeping up — fine. `%util` near 100 **and** `await` climbing (here, 42ms — on NVMe-backed EBS, healthy is usually low single digits) means requests are queueing faster than the device can drain them: genuine saturation. `aqu-sz` (average queue size) rising alongside `await` confirms it — the queue is backing up, not just one slow outlier request.

### Who's actually blocked

```bash
ps -eo pid,stat,wchan,comm | awk '$2 ~ /D/'
```
A process in `D` state (uninterruptible sleep) is waiting on the kernel — almost always I/O — and **cannot be killed** with a normal signal until the I/O completes or times out, which is itself useful diagnostic information (a `kill -9` that does nothing confirms you're looking at a real kernel-level I/O stall, not a hung userspace process).

```bash
cat /proc/PID/io               # read_bytes / write_bytes for one process — confirms which process is generating the load iostat is showing
cat /proc/PID/stack            # where in the kernel it's actually stuck
```

### Device-level configuration

```bash
cat /sys/block/nvme0n1/queue/scheduler
cat /proc/sys/vm/dirty_ratio /proc/sys/vm/dirty_background_ratio
```
`dirty_ratio`/`dirty_background_ratio` control how much written-but-not-yet-flushed data the kernel lets accumulate in page cache before forcing writes to disk — a workload with heavy, bursty writes hitting these thresholds shows up as sudden `await` spikes that look like "the disk got slow" but are actually "the kernel just started forcing a large flush it had been deferring."

### Install (RHEL/Amazon Linux)

```bash
dnf install sysstat        # iostat, pidstat -d, sar -d
dnf install iotop
dnf install fio            # benchmark / reproduce, not just observe
dnf install smartmontools  # smartctl -a /dev/sda — mostly for instance-store/bare metal, not EBS
dnf install bcc-tools      # biolatency, biosnoop, ext4slower, xfsslower
dnf install lvm2 xfsprogs
```

```bash
pidstat -d 1                       # per-process I/O over time
iotop -oPa                         # only active processes, accumulated totals
biolatency-bpfcc 10 1              # I/O latency histogram — shows the *distribution*, not just the average iostat gives you
biosnoop-bpfcc                     # per-I/O trace with PID and file — the "who did this specific slow read" tool
fio --name=t --rw=randread --bs=4k --iodepth=32 --runtime=30 --filename=/dev/nvme1n1   # reproduce/benchmark, doesn't touch a mounted filesystem's data if pointed at a raw unmounted device
```

## AWS-specific gotchas

- **`gp2` burst balance runs out — and then everything gets slow at once.** `gp2` volumes earn I/O credits and burst above their baseline IOPS (3 IOPS/GB, minimum 100) using a credit balance, exactly like `t3` CPU credits. When the balance hits zero, IOPS are hard-capped to baseline — a 100GB `gp2` volume (300 baseline IOPS) that's been bursting to 3000 IOPS all day will suddenly, uniformly get slower with **no error, no warning in `iostat` beyond `await` climbing**. Check CloudWatch `BurstBalance` on the volume before chasing an application-side cause.
- **`gp3` doesn't have this problem** — it has a flat, provisioned baseline (3000 IOPS / 125 MiB/s by default, independently adjustable) with no credit mechanism. If burst exhaustion is a recurring incident, migrating `gp2` → `gp3` (an online operation) usually resolves it outright.
- **Instance-store (ephemeral, NVMe) volumes on the instance itself are physically fast but disappear on stop/terminate** — if `lsblk` shows a fast local NVMe device that isn't in `blkid`/`fstab`, someone put transient data on it; that's a data-durability conversation, not a performance one.
- **Multi-attach and shared filesystems (EFS) add network latency to every I/O** — `iostat`'s `await` on an EFS mount includes a round trip over the network, so the healthy baseline is fundamentally different (single-digit milliseconds isn't the bar; check the EFS CloudWatch metrics instead of comparing directly to local NVMe numbers).

## Worked example: "Batch job that ran fine all month suddenly takes 4x longer"

**Symptom:** a nightly ETL job on a `gp2`-backed instance, normally 40 minutes, took 3 hours last night. No code changes, no deploy.

**Step 1 — rule out full disk first (cheap check):**
```bash
df -h
```
```
/dev/nvme0n1p1   500G  210G  290G  43% /data
```
Plenty of space. Not this.

**Step 2 — check I/O saturation during the slow run:**
```bash
iostat -xz 1
```
```
Device     r/s     w/s   rkB/s   wkB/s   await  aqu-sz  %util
nvme0n1   295.0   180.0  4200.0  9800.0  185.40   22.10   99.80
```
`%util` pinned near 100, `await` at 185ms (versus a normal night's ~8ms baseline for this workload) — genuine, severe I/O saturation, not a red herring.

**Step 3 — this is `gp2`, so check burst balance before assuming an application regression:**

CloudWatch → `BurstBalance` for the volume: 0% for the entire run, having been at ~40% at the same time the previous night. The dataset grew past the point where the job's I/O pattern fits inside a normal night's available burst credits, and the volume fell back to baseline IOPS mid-job.

**Root cause:** dataset growth pushed the job's I/O demand above what burst credits could sustain for the job's full duration; baseline IOPS on this volume size (300 IOPS) is far below what the job actually needs.

**Fix (immediate):** migrate to `gp3` and provision IOPS/throughput to match the job's real requirement (measured from `iostat` during a healthy run, with headroom). **Prevention:** alarm on `BurstBalance` trending toward zero, and size volumes from measured baseline I/O needs rather than from capacity alone — a volume can have "plenty of space" and still be badly undersized for IOPS.

## Cheat sheet

```bash
df -h ; df -i                  # space AND inodes
du -xh --max-depth=1 / | sort -h
lsblk -f ; blkid
mount | column -t
cat /proc/diskstats            # raw counters behind iostat
cat /proc/pressure/io          # PSI — I/O stall time
ps -eo pid,stat,wchan,comm | awk '$2 ~ /D/'   # who is blocked
cat /proc/PID/io               # read_bytes, write_bytes per process
cat /proc/PID/stack            # where in kernel it's stuck
cat /sys/block/nvme0n1/queue/scheduler
cat /proc/sys/vm/dirty_ratio /proc/sys/vm/dirty_background_ratio

dnf install sysstat        # iostat, pidstat -d, sar -d
dnf install iotop
dnf install fio            # benchmark / reproduce
dnf install smartmontools  # smartctl -a /dev/sda
dnf install bcc-tools      # biolatency, biosnoop, ext4slower, xfsslower
dnf install lvm2 xfsprogs

iostat -xz 1                       # the workhorse
pidstat -d 1                       # per-process I/O
iotop -oPa                         # only active, accumulated
biolatency-bpfcc 10 1              # latency histogram
biosnoop-bpfcc                     # per-I/O trace with PID and file
fio --name=t --rw=randread --bs=4k --iodepth=32 --runtime=30 --filename=/dev/nvme1n1

# AWS: check EBS CloudWatch BurstBalance (gp2) before assuming an application-side regression
```
