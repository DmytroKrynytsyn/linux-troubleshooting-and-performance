# Interview Drills

Thirty scenarios, structured the way you'd actually talk through them out loud. Each has the same shape:

- **The question** as an interviewer would pose it
- **What they're testing** — the thing behind the question
- **How to open** — your first sentence, which matters more than the rest
- **The walkthrough** — the sequence, with the commands
- **The differentiator** — the detail that separates a good answer from a great one

**The single most important habit:** never answer a troubleshooting question by listing commands. Open with the *branch* — the question you're trying to answer and how you'd narrow it. "First I'd establish whether this is a capacity problem or a performance problem, because they need completely different tools" is worth more than any ten commands. Interviewers are listening for structured elimination, not recall.

**Say what you'd rule out, not just what you'd check.** "I'd check `df -i` too, because a full inode table looks identical to a full disk from the application's side" shows you know the failure modes, not just the tools.

---

## A. First-response and methodology

### 1. "You get paged: an EC2 instance is slow. Walk me through your first two minutes."

**Testing:** Do you have a repeatable method, or do you flail at whatever tool you remember first?

**Open with:** "I'd start by identifying *which* resource is contended before touching any specific tool, because the answer determines everything after."

**Walkthrough:**
```bash
uptime ; nproc                          # is load actually high relative to cores?
cat /proc/pressure/{cpu,memory,io}      # which resource is stalling — one read, three answers
dmesg -T | tail -50                     # did the kernel already tell me? OOM, driver resets, EBS errors
systemctl list-units --failed
top                                     # only now, once I know where to look
```
Then branch to the relevant guide.

**Differentiator:** Volunteer the PSI caveat unprompted. "PSI needs kernel 4.20+, so on Amazon Linux 2 with the 4.14 kernel that file doesn't exist and I'd fall back to `vmstat`'s `r` column versus `nproc` for CPU and `si`/`so` for memory." Knowing where your favourite tool *isn't available* is the strongest signal in this entire list.

Also worth adding: "And in parallel I'd check whether this is one instance or the whole fleet, because that changes it from a host problem to a dependency or deploy problem."

---

### 2. "Load average is 40 on a 4-core box, but `top` shows the CPUs mostly idle. Explain."

**Testing:** Do you actually know what load average measures?

**Answer:** Linux load average counts processes in **runnable *or* uninterruptible sleep (`D`) state** — unlike most other Unixes, which count only runnable. So a load of 40 with idle CPUs means ~40 processes are blocked in the kernel, almost always on I/O.

```bash
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'
```
The `wchan` column names the kernel function. `nfs_wait_bit_killable` or anything `rpc_*` means a hung NFS/EFS mount. `wait_on_page_bit` means a slow block device.

**Differentiator:** "On AWS the most common version of this is an EFS or NFS mount whose target became unreachable — every process that touches that path piles into `D` state and can't even be killed, because you can't signal a process in uninterruptible sleep. The load number is a symptom of the mount, not of CPU."

---

### 3. "How do you know whether a problem is your instance or AWS?"

**Testing:** Do you understand the boundary of the guest OS?

**Answer:** Work outward and be explicit about what each layer can and can't see.

1. **Guest OS** — `ss`, `iostat`, `free`, PSI. If a resource is genuinely saturated here, it's yours.
2. **Instance-level shaping** — `ethtool -S eth0 | grep exceeded` for network, CloudWatch `CPUCreditBalance` and `BurstBalance` for compute and storage. These are AWS limits enforced above the guest and **invisible to every generic Linux tool**.
3. **Network layers** — security groups, NACLs, route tables. VPC Flow Logs show `REJECT` records for packets that never reached the guest.
4. **EC2 status checks** — system check = the hypervisor and host hardware; instance check = network reachability to your instance.

**Differentiator:** "The clean tell is that `tcpdump` showing *nothing* is itself evidence — if the packet never arrived, the drop happened above me, and I'd go to flow logs and the ENA allowance counters rather than keep digging in the guest."

---

## B. CPU

### 4. "`t3.large`, throughput halved, `top` shows 30% CPU. What's happening?"

**Testing:** Burstable instance mechanics.

**Answer:** Almost certainly CPU credits — but **which failure it is depends on the credit mode**, and they produce opposite symptoms:

- **Standard mode:** credits hit zero, the vCPU is hard-throttled to baseline. `top` shows normal or *low* usage because the CPU itself got slower, not busier. Throughput craters.
- **Unlimited mode** (the default for T3/T3a/T4g): you are **not** throttled. You keep bursting and get billed for surplus credits. Performance stays fine; the bill moves.

Since throughput actually dropped, this is standard mode. Confirm:
```bash
aws ec2 describe-instance-credit-specifications --instance-ids i-...
```
CloudWatch → `CPUCreditBalance` flat at zero.

**Differentiator:** Two things. First, PSI-CPU will read high while `top` reads 30% — that contradiction is the fingerprint of an externally capped vCPU. Second: "In unlimited mode there's a confusing second-order effect where `CPUCreditBalance` stays at zero even after load drops, because earned credits pay down `CPUSurplusCreditBalance` first. An instance that looks permanently out of credits at 5% CPU is usually paying off a debt, not broken."

---

### 5. "A container is slow. The node has plenty of idle CPU. Why?"

**Testing:** cgroups.

**Answer:** CPU quota. `nproc` inside a container respects affinity but **not** the cgroup quota, so the app thinks it has the whole node.

```bash
cat /sys/fs/cgroup/cpu.max      # "50000 100000" = 50ms per 100ms window = 0.5 CPU
cat /sys/fs/cgroup/cpu.stat     # nr_throttled ← the whole answer
```
`nr_throttled` climbing means the app is being stopped, not slowed.

**Differentiator:** "In Kubernetes this is often caused by setting a CPU *limit* at all. Because the CFS quota is enforced per 100ms period, a multi-threaded app can exhaust its whole quota in the first 20ms of each period and sit idle for 80ms — you get throttling even at average utilisation well below the limit. For latency-sensitive services the usual advice is to set requests and drop the limit."

Add the cgroup v1 path (`cpu.cfs_quota_us`, `cpu.stat`) since Amazon Linux 2 nodes are still v1.

---

### 6. "One core is at 100%, the other seven are idle. What now?"

**Answer:** Single-threaded work, or interrupts pinned to one core.

```bash
mpstat -P ALL 1                 # confirm which core, and whether it's %usr or %sys
top -H -p PID                   # which thread
cat /proc/interrupts | grep eth # all NIC IRQs on core 0?
cat /proc/softirqs | grep NET_RX
```
If it's `%usr` on one thread, it's application design. If it's `%sys` or `%soft` on core 0, it's interrupt handling — check `irqbalance`, and on ENA use `ethtool -l` to confirm multiple queues are configured and RPS/RFS is spreading softirq work.

**Differentiator:** "The distinction matters because they have opposite fixes: a single hot application thread means adding cores buys you nothing, whereas an interrupt bottleneck is fixed by spreading, not scaling."

---

## C. Memory

### 7. "`free -h` shows 200MB free out of 32GB. Is this a problem?"

**Testing:** The single most common Linux misconception.

**Answer:** Almost certainly not. Read **`available`**, not `free`. The kernel uses spare RAM for page cache and reclaims it instantly under pressure — high cache is the kernel doing its job, not a leak. `free` being near zero on a long-running box is normal and healthy.

Real signals: `available` shrinking over time, `si`/`so` non-zero in `vmstat` (actively swapping right now), or `/proc/pressure/memory`'s `full` line rising.

**Differentiator:** "One trap here: `swpd` being non-zero in `vmstat` isn't a problem either — those are pages swapped out during some past spike and never needed since. The *rate*, `si`/`so`, is what matters, not the total."

---

### 8. "A process disappeared overnight. No error in the application log."

**Answer:** Check for a kill before hunting live — if the OOM killer fired, everything looks healthy now.

```bash
dmesg -T | grep -iE 'oom|killed process'
```
Read the *wording*, because it names the level:
- `Out of memory: Killed process` → global exhaustion
- `Memory cgroup out of memory` → a container limit, and the host may have had gigabytes free

Then check the other two killers people forget:
```bash
cat /sys/fs/cgroup/memory.events        # oom_kill count, survives a wrapped dmesg ring buffer
journalctl -u systemd-oomd ; oomctl     # userspace PSI-based killer, leaves NOTHING in dmesg
```

**Differentiator:** "`systemd-oomd` is the one that catches people. It's on by default in Ubuntu 22.04+ and Fedora, kills based on PSI before the kernel would, and logs only to its own unit. If a service keeps dying with no kernel OOM message, that's usually why." Then add the leading indicator: "In `memory.events`, `high` climbing with `oom_kill` at zero means the cgroup is being throttled into constant reclaim without dying — a slow performance problem that no OOM alarm will ever catch."

---

### 9. "Sum of RSS across processes is 40GB on a 16GB box. Explain."

**Answer:** RSS counts shared pages against every process that maps them. Ten workers sharing one 200MB library each report +200MB. Summing RSS double-counts by design.

```bash
cat /proc/PID/smaps_rollup      # Pss = fair share of shared pages; Private_Dirty = truly this process's
```

**Differentiator:** "For leak hunting I actually watch `Private_Dirty` rather than `Pss`, because it's the memory that belongs only to this process and cannot be reclaimed without killing it. Cache and shared library text can be evicted; `Private_Dirty` growing monotonically over hours is the leak."

---

### 10. "Java service on EKS gets OOMKilled nightly. Node memory is fine."

**Answer:** cgroup limit, not host exhaustion — confirmed by the "Memory cgroup out of memory" wording and by the node metrics being clean.

The usual root cause is heap sized without margin for non-heap memory: thread stacks, metaspace, code cache, direct byte buffers, GC structures. `-Xmx896m` in a 1 GiB container leaves ~128 MiB for all of that, which is tight and gets tighter as thread count rises under load.

**Fix:** `-XX:MaxRAMPercentage=70` instead of a hardcoded `-Xmx`, so the heap tracks the container limit and can't drift out of sync when someone resizes the pod.

**Differentiator:** "The prevention answer is alarming on the container's `memory.events` `high` counter rather than on OOM kills — `high` was rising for hours before the first kill. One is a leading indicator, the other is a post-mortem." Also mention the equivalent traps for other runtimes (`--max-old-space-size` for Node, `GOMEMLIMIT` for Go), which shows the principle rather than one memorised JVM flag.

---

## D. Disk

### 11. "Disk is 100% full. `du` only accounts for 40%. Where's the rest?"

**Testing:** Do you know how unlinking works? This is the classic.

**Answer:** A deleted file still held open by a running process. `rm` unlinks it from the directory tree, so `du` (which walks the tree) can't see it — but the blocks aren't freed until the last file descriptor closes.

```bash
lsof +L1                                    # link count < 1
ls -l /proc/*/fd 2>/dev/null | grep deleted # /proc alternative, no lsof needed
```
```
java 4821 app 7w REG 259,1 8589934592 0 1234 /var/log/app/app.log (deleted)
```

**The fix without restarting the service:**
```bash
: > /proc/4821/fd/7
```
You can still write through the fd even though the path is gone — this truncates and returns the blocks immediately while the process keeps logging.

**Differentiator:** Two things. The `/proc/*/fd` route, because minimal AMIs often don't have `lsof` and reaching for a package install mid-incident is bad. And the second cause: "If that's clean, check for files hidden *under* a mountpoint — something wrote to `/data` before the volume was mounted there. `mount --bind / /mnt/x` then `du` the shadowed path."

**Prevention:** `logrotate` with `copytruncate`, or a `postrotate` that signals the app to reopen. Never `rm` an active logfile.

---

### 12. "`touch` fails with 'No space left on device' but `df -h` shows 40% free."

**Answer:** Inode exhaustion. `df -i`. Every file consumes exactly one inode regardless of size, so millions of tiny files exhaust inodes long before space.

**Differentiator:** "This is mostly an ext4 problem, so it's mostly an Ubuntu problem — ext4 fixes the inode count at `mkfs` time and it cannot be changed without recreating the filesystem. XFS, which is the Amazon Linux and RHEL root default, allocates inodes dynamically and effectively never runs out. So the first thing I'd check is which filesystem I'm on, because that tells me whether `df -i` is even worth running."

Knowing *when a check is unnecessary* is a stronger signal than knowing the check.

---

### 13. "You grew the EBS volume in the console. `df` hasn't changed."

**Answer:** Three independent layers, each needs growing:

```bash
lsblk                                    # 1. does the kernel see the new device size?
growpart /dev/nvme0n1 1                  # 2. the partition — note the SPACE
xfs_growfs -d /                          # 3a. XFS takes a MOUNTPOINT
resize2fs /dev/nvme0n1p1                 # 3b. ext4 takes a DEVICE
```

**Differentiator:** The argument-type difference (mountpoint vs. device) is a real trip hazard under pressure — naming it shows you've done this rather than read about it. Then: "Both grow online, no downtime. But **XFS cannot shrink, ever** — if you over-provision, the only path is create, copy, swap. That's worth raising in a design review before someone provisions 16TB 'to be safe'."

If `lsblk` still shows the old size: `echo 1 > /sys/class/block/nvme0n1/device/rescan_controller`.

---

### 14. "A nightly batch job that took 40 minutes now takes 3 hours. No code changes."

**Answer:** Rule out capacity first (`df -h`, `df -i` — cheap, eliminates a whole class), then look for saturation:
```bash
iostat -xz 1     # await + aqu-sz climbing together = genuine saturation
```
If `await` is at 185ms against a normal 8ms baseline while total IOPS is *lower* than usual, something has lowered the ceiling rather than raised the demand. On `gp2`, check CloudWatch `BurstBalance`.

`gp2` baseline is 3 IOPS/GiB (minimum 100), bursting to 3000 on credits. Dataset growth pushes sustained demand past what credits can fund for the job's full duration, and it falls back to baseline mid-run.

**Differentiator:** Three additions. "`%util` is misleading on NVMe — it measures time with at least one request in flight, so a device with deep parallelism hits 100% well below capacity. I'd weight `await` and `aqu-sz` over `%util`." Then: "`biolatency` gives the distribution that `iostat`'s average hides — uniform slowness means a rate cap, a bimodal split means tail latency." And: "Before provisioning more volume IOPS I'd check the *instance's* aggregate EBS bandwidth ceiling — provisioning 16,000 IOPS on an instance capped at 4750 Mbps buys nothing."

---

## E. Network

### 15. "The app works from `curl localhost` but the load balancer health check fails."

**Answer:** Almost always bound to `127.0.0.1` instead of `0.0.0.0`.
```bash
ss -tulpn | grep 8080
```
Identical from inside the box, refuses every external connection.

If it *is* on `0.0.0.0`, work outward: guest firewall → security group (does it allow the ALB's SG on the health-check port?) → the health check's own path and expected status code.

**Differentiator:** "The `Recv-Q` and `Send-Q` columns on a LISTEN socket are worth reading while I'm there — they're the current and maximum backlog. If `Recv-Q` is approaching `Send-Q`, the app isn't calling `accept()` fast enough and I'm looking at a different problem entirely."

---

### 16. "Health checks fail intermittently. The application log is completely clean."

**Answer:** Clean app log during failures means connections aren't reaching the app.
```bash
ss -tan state syn-recv | wc -l
cat /proc/net/netstat | grep -i listen        # ListenOverflows / ListenDrops climbing
```
Backlog exhaustion: SYNs arriving faster than the app calls `accept()`, so the kernel drops them silently.

**Fix:** raise `net.core.somaxconn` **and** the application's own `listen()` backlog argument — the effective value is `min(somaxconn, app_backlog)`, so raising only the sysctl does nothing. That detail alone is worth stating.

**Differentiator:** "I'd rule out AWS-layer shaping in the same breath, because it produces an identical symptom: `ethtool -S eth0 | grep exceeded`, sampled twice and diffed since the counters are cumulative since boot." Then the capture logic: "A capture distinguishes them cleanly — SYN arriving with no SYN-ACK sent means *I* dropped it; SYN never arriving means it was blocked above me."

---

### 17. "Connections to an internal service fail at peak with 'Cannot assign requested address'."

**Answer:** `EADDRNOTAVAIL` — ephemeral port exhaustion, not a firewall. A single source IP can hold ~28,000 concurrent connections to the same destination IP:port tuple, and `TIME_WAIT` sockets count against it for 60 seconds.

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
ss -tan state time-wait | wc -l
cat /proc/net/sockstat
```

**Mitigations in order:** fix the app (connection pooling, keep-alive) → `tcp_tw_reuse=1` → widen the port range → add source IPs.

**Differentiator:** "I'd specifically *not* use `tcp_tw_recycle`, which every pre-2018 blog post recommends. It broke connections from clients behind NAT and was removed from the kernel in 4.12." Knowing which advice is stale is a much better signal than knowing the advice.

---

### 18. "Everything in the guest looks clean, but the fleet drops connections under load. Where do you look?"

**Testing:** Do you know AWS shapes traffic above the guest?

**Answer:**
```bash
ethtool -S eth0 | grep -E 'exceeded|allowance'
```
Every Nitro instance type has hard ceilings on aggregate bandwidth, PPS, tracked connections, and link-local traffic. Exceed one and packets are queued or dropped **above the guest** — `ip -s link` shows nothing, `tcpdump` shows nothing because the packet never arrived, the app just sees timeouts.

- `bw_in/bw_out_allowance_exceeded` → bandwidth cap
- `pps_allowance_exceeded` → packet-rate cap (small packets hit this before bandwidth)
- `conntrack_allowance_exceeded` → **new connections refused at the instance level**, because security groups are stateful and AWS tracks every connection per ENI
- `linklocal_allowance_exceeded` → DNS, IMDS, and Time Sync throttled

**Differentiator:** "`conntrack_allowance_available` is the leading indicator and the one I'd alarm on — it tells you how much headroom is left before connections start failing, rather than telling you afterwards. It needs ENA driver 2.8.1+, so its absence means an old driver, not that you're safe." Then: "I'd also push these into CloudWatch via the agent's ethtool plugin, because at fleet scale this is the difference between a five-minute diagnosis and a five-hour one."

---

### 19. "Intermittent, unexplainable DNS timeouts on a busy EKS node."

**Answer:** The VPC resolver enforces **1024 packets per second per network interface**. Exceed it and queries are silently dropped — timeouts, not errors, which is why nothing logs an error.

```bash
ethtool -S eth0 | grep linklocal_allowance_exceeded
```
Non-zero confirms it. Fix with a node-local DNS cache (NodeLocal DNSCache on EKS), by raising `ndots` behaviour so you're not making five failed lookups per resolution, or by reducing query volume.

**Differentiator:** "The Kubernetes default `ndots:5` makes this dramatically worse — every external hostname generates several failed search-domain lookups before the real one. Fixing `ndots` often removes the problem without touching infrastructure." Also: "This is the same allowance bucket as IMDS, so an aggressive metadata-polling sidecar can cause DNS failures, which is a genuinely counterintuitive link."

---

### 20. "TLS handshakes complete but large uploads hang. Small requests are fine."

**Answer:** Path MTU mismatch. The handshake uses small packets and succeeds; the first full-size packet exceeds the path MTU, can't be fragmented (DF bit set), and is dropped.

```bash
ip link show eth0 | grep mtu
tracepath <host>
ping -M do -s 8972 <host>        # 8972 + 28 = 9000
```
AWS MTUs: **9001 within a VPC**, **1500 over an internet gateway**, 8500 over Transit Gateway.

**Differentiator:** "PMTUD depends on ICMP 'fragmentation needed' packets getting back to the sender. If a security group or NACL blocks ICMP — which people do reflexively for 'security' — PMTUD breaks and connections *hang* instead of failing cleanly. That's the best concrete argument for allowing ICMP type 3 code 4, and it's a good example of a security decision with a non-obvious availability cost."

---

### 21. "Explain security groups versus NACLs, and give me a failure only one of them causes."

**Answer:**

| | Security group | NACL |
|---|---|---|
| Attaches to | ENI | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow **and deny**, in number order, first match wins |

**The failure only NACLs cause:** because they're stateless, return traffic needs its own explicit rule. A connection can pass the security group outbound and stall because the inbound NACL has no rule for the ephemeral port range (`1024–65535`) the response arrives on. Security groups handle that automatically; NACLs don't.

**Differentiator:** "Practically I check the security group first because it's the more common misconfiguration and cheaper to inspect — but for inbound traffic AWS actually evaluates the NACL first. And when both look right, VPC Flow Logs show `REJECT` records for packets that never reached the guest, which is my proof that the problem isn't the instance."

---

## F. Boot and recovery

### 22. "An instance won't boot after a config change. Walk me through recovery."

**Testing:** Composure and sequence under the highest-stakes scenario.

**Open with:** "First I'd read the system log, because it tells me which stage failed and that determines everything after."

**Walkthrough:**
1. **"Get system log"** in the console — no setup needed. If it reaches `Reached target Multi-User System`, it *booted* and I have a network or sshd problem, not a boot problem.
2. **Serial Console** if enabled — interrupt GRUB, boot with `systemd.unit=emergency.target`, `mount -o remount,rw /`, fix in place. Two minutes.
3. **Otherwise the universal fallback:** stop the instance (never terminate), **snapshot the root volume**, detach it, attach to a rescue instance in the **same AZ**, mount, fix, reattach at the **exact original device name**, start.

**Differentiator (this is the answer that lands):** "The reason step 2 usually isn't available is that Serial Console needs an OS user with a password actually set, and cloud AMIs ship with password login disabled. So the practical answer is to do that in advance on every instance — it's a five-minute change that turns a twenty-minute recovery into a two-minute one."

Then the fstab detail: "The cause is `/etc/fstab` more often than everything else combined, which is why every non-root mount on a cloud instance should carry `nofail` and `x-systemd.device-timeout`, and be referenced by UUID rather than device path — NVMe enumeration order isn't stable across reboots."

Two more if they're still listening: chroot into the volume for package or kernel work, but pass the kernel version explicitly (`dracut -f --kver $KVER`), because `uname -r` inside a chroot reports the *rescue host's* kernel. And on Nitro, verify the initramfs contains the `ena` and `nvme` modules before reattaching, or you get a `VFS: Unable to mount root fs` panic.

---

### 23. "A new AMI passes both status checks but SSH times out."

**Answer:** Status checks passing means the hypervisor link and basic reachability are fine — so it's inside the guest. After a kernel or AMI change, the most likely cause is NIC renaming.

```bash
ip -br a                   # no eth0?
dmesg | grep -iE 'renamed|ens'
```
```
ens5: renamed from eth0
```
Predictable interface naming derives the name from PCI bus location; the network config still refers to `eth0`, so the interface is up but unconfigured.

**Fix:** update the network config to the new name (preferred), or force legacy naming with `net.ifnames=0 biosdevname=0` on the cmdline.

**Differentiator:** "I'd fix the config rather than pin legacy naming, unless we've deliberately standardised on `eth0` fleet-wide — pinning is a workaround that hides the same problem next time." Then prevention: "This is exactly what AMI validation in the pipeline is for: launch from the candidate AMI, wait for the reachability check, SSH in, run a smoke test, and only then promote it to known-good. This class of bug should never reach production."

---

### 24. "`systemd-analyze blame` says a unit took 20 seconds. Is that your problem?"

**Answer:** Not necessarily. `blame` lists units by their own startup time, but most start in parallel and finish long before boot completes — they were never on the critical path. `critical-chain` walks the actual dependency graph and shows what was *waited on*.

If `blame` shows a 20s unit that `critical-chain` doesn't mention, it cost you nothing.

**Differentiator:** "The related trap is `journalctl -b -1` for the previous boot. It only works with a persistent journal, and Amazon Linux ships volatile by default — so after a crash-reboot the evidence is simply gone. I'd check `journalctl --list-boots` before relying on it, and make the journal persistent across the fleet, because losing the log of the boot that failed is a self-inflicted wound."

---

## G. Beyond troubleshooting — the parts of the JD people under-prepare

### 25. "Tell me about a time you improved an operational procedure."

**Testing:** This maps directly to a *stated basic qualification* ("leading the creation, revision, and/or improvement of SOPs"). It's a guaranteed question and it's not a Linux question.

**Structure (STAR, with the emphasis on R):**
- **Situation:** which recurring incident or gap
- **Task:** what you specifically owned
- **Action:** what you wrote or changed, and — critically — **how you got other people to adopt it**
- **Result:** a number. MTTR, page volume, incidents avoided, onboarding time

**Differentiator:** Adoption is the part people skip and interviewers listen for. "I wrote a runbook" is weak. "I wrote it, walked two on-call engineers through it during a game day, found three steps that were wrong under real conditions, and revised it — and the next occurrence was resolved by someone who'd never seen the failure before" is the answer.

Have a measurable outcome ready even if it's approximate, and be honest that it's approximate.

---

### 26. "You've written a diagnostic guide. How do you know it's correct?"

**Testing:** Intellectual honesty. This is a *Dive Deep* and *Are Right, A Lot* probe.

**Answer:** Verify claims against primary sources, test commands on the actual distros, and be specific about version and platform boundaries rather than writing advice that's true "in general." Then say what you'd get wrong: burstable credit behaviour, EBS baseline arithmetic, and which kernel features exist on which AMI are all easy to state confidently and incorrectly.

**Differentiator:** Give a real example of something you corrected. "I'd written that a `t3` in unlimited mode gets throttled at zero credits. It doesn't — that's standard mode. In unlimited you keep bursting and get billed. Getting it backwards would send someone down completely the wrong path during an incident, so I now check credit-mode claims against the AWS docs rather than memory." Admitting a specific correction is far stronger than claiming you don't make them.

---

### 27. "How do you decide what to alarm on?"

**Answer:** Alarm on things that are **leading** and **actionable**, not on things that are merely visible.

| Weak alarm | Better alarm | Why |
|---|---|---|
| Low free memory | `memory.events` `high` counter | `free` is meaningless; `high` rises hours before the kill |
| High CPU | `CPUCreditBalance` trending down | The symptom you care about is throttling, not utilisation |
| Disk > 80% | `BurstBalance` + inode usage | Space is rarely what actually breaks first |
| Network errors | `conntrack_allowance_available` | Tells you before connections start failing |
| Host `MemoryUtilization` | Container-level memory | They diverge exactly when it matters |

**Differentiator:** "The pattern across all of these is that the trailing indicator is the one that's easy to graph and the leading one is the one that's useful. `ListenDrops` caught a health-check problem for us before a single application log line did — and that's the test I'd apply: would this alarm have fired before the last incident, or only during it?"

---

### 28. "This role has an on-call rotation. How do you keep it sustainable?"

**Testing:** Operational maturity, and the JD's stated 24x7 high-availability context.

**Answer:** Three things: every page must be actionable (a page that has no action is an alarm that should be a dashboard), every recurring page gets a root-cause fix or an automation rather than a better runbook, and every incident produces a written follow-up with an owner. Runbooks reduce MTTR; removing the failure removes the page.

**Differentiator:** "I'd track page volume per rotation as a first-class metric alongside MTTR, because MTTR alone can look great while the team burns out. And I'd be specific that the goal isn't zero pages — it's that every page is one a human genuinely needed to see."

---

### 29. "The JD lists Python or scripting. What have you automated?"

**Testing:** It's a **basic** qualification, so it will come up. People preparing for a Linux interview reliably under-prepare here.

**Have ready:** one concrete thing you automated, why it was worth automating, and how you handled failure — retries, idempotency, what happens when it runs twice, how you knew it worked.

**Differentiator:** Idempotency and blast radius. "It was designed so re-running it was safe, it ran against one host first, and it wrote what it *would* do before it did anything." That's the difference between someone who writes scripts and someone who writes scripts that run against a fleet.

If your automation is mostly shell rather than Python, say so plainly and describe where you'd reach for each. Overclaiming is worse than a narrower honest answer.

---

### 30. "Do you have questions for us?"

Not optional. Have three, and make them operational rather than perks:

- "What does the on-call rotation actually look like day to day — page volume, and how much of it is recurring versus novel?"
- "How much of the fleet is Amazon Linux versus other distributions? I'm curious how much variance the runbooks have to absorb."
- "European Sovereign Cloud means separate infrastructure and separate operational boundaries. What's genuinely different about operating it compared to a standard AWS region?"

That last one is specific to this role, shows you read past the job title, and gets you a real conversation instead of a recruiting answer.

---

## Quick self-test

Cover the answers. If you can't answer these in two sentences each, go back to the relevant guide.

1. Why does load average count `D`-state processes, and what does that change?
2. `t3` standard vs. unlimited at zero credits — what happens in each?
3. `df` and `du` disagree. Two possible causes and one command for each.
4. Which is stateful, security groups or NACLs, and name a failure the other one causes.
5. Which command reveals AWS shaping your traffic above the guest OS?
6. What overrides `/etc/security/limits.conf` for a systemd service?
7. XFS or ext4 — which can run out of inodes, and which can't shrink?
8. `blame` vs. `critical-chain` — which one do you trust?
9. Three different things can OOM-kill a process. Name them and where each logs.
10. Why might `journalctl -b -1` return nothing on Amazon Linux?
11. Which sysctl for TIME_WAIT reuse is safe, and which was removed from the kernel?
12. VPC MTU, internet gateway MTU — and what breaks PMTUD?
13. `nr_throttled` — which file, and what does it prove?
14. What's the hard packets-per-second limit on the VPC DNS resolver, and per what?
15. Serial Console needs one thing you must configure in advance. What?