# CPU Troubleshooting

"High CPU" is often the *symptom* reported, rarely the actual diagnosis. The real question is always one of: too much work, work stuck waiting on something else (which shows up as CPU pressure but isn't a CPU problem), one runaway thread, or — on AWS specifically — a burstable instance that simply ran out of credit. This guide walks through telling those apart.

## 1-Minute Summary

- `uptime` + `nproc` first — load average is a count of runnable/waiting processes, not a percentage; compare it to your core count before deciding anything is "high."
- `cat /proc/pressure/cpu` is the fastest honest answer to "is CPU actually the bottleneck" — no math against core count required.
- One hot core vs. all of them changes the fix entirely — check with `mpstat -P ALL 1`, not just aggregate `top`.
- `strace -c -p PID` / `perf top -p PID` tell you *what* the hot thread is actually doing, once you've found it.
- **On `t3`/`t4g`:** CPU can be hard-throttled by credit exhaustion while `top` shows normal/low usage. Check CloudWatch `CPUCreditBalance` before assuming a code regression.

## Methodology

1. **Is the system actually loaded, relative to its core count?** `uptime` + `nproc`.
2. **Is it one core pegged, or all of them?** Aggregate `top` numbers hide this — go per-core.
3. **Is it user code, kernel time, or I/O wait masquerading as "CPU"?** `us` vs `sy` vs `wa` vs `st`.
4. **Which process, which thread?** Narrow from system → process → thread.
5. **Why is that thread hot?** `perf`/`strace` to see what it's actually doing.
6. **On a burstable instance:** rule out CPU credit exhaustion before you go any further — it produces symptoms that look exactly like a code regression.

## Commands, explained

### `uptime` and `nproc` — the first 5 seconds

```bash
uptime
```
```
14:32:01 up 12 days,  3:41,  2 users,  load average: 8.42, 6.15, 3.20
```
```bash
nproc
```
```
4
```
Load average is *runnable + uninterruptible-sleep* processes, averaged over 1/5/15 minutes — **not** a percentage. A load of 8.42 on a 4-core box means, on average, roughly twice as many processes wanted a CPU as you have cores. The three numbers trending up (3.20 → 6.15 → 8.42) tell you the load is *building right now*, not a spike that already passed.

### `vmstat` — the run queue, without the noise

```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free  buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 812340  40212 2211200    0    0     2    18  980 2400 71  9  0  0 20
```
The `r` column is the run queue *right now* (processes ready to run but waiting for a core) — 6 runnable on a 4-core box confirms real CPU contention, not just a noisy `uptime` average. Watch `us`/`sy`/`wa`/`st` together: high `us` is your application; high `sy` is the kernel (often syscalls or context switching); high `wa` means CPU looks busy but is actually stuck on I/O (go to the [disk guide](disk.md) instead); **`st` (steal time) is CPU your instance was *promised* but the hypervisor gave to a noisy neighbor or throttled away — this is an AWS-specific signal, see below.**

### `top` / `top -H -p PID` — who, and which thread

```bash
top -H -p 4821
```
Press `1` in plain `top` to see per-core breakdown instead of the aggregate — this is how you catch "one thread pegging one core" versus "genuinely load-balanced across all four," which look identical in the aggregate view but have completely different fixes (fix the code vs. add capacity).

### `/proc/pressure/cpu` — PSI, the fast path

```bash
cat /proc/pressure/cpu
```
```
some avg10=42.10 avg60=38.55 avg300=21.03 total=98234123
```
`some avg10=42.10` means: over the last 10 seconds, *some* process was stalled waiting for a CPU 42% of the time. This is the most direct answer to "is CPU actually the bottleneck" — no math against core count required, no guessing from load average. If PSI-CPU is low but users report slowness, stop looking at CPU and check `/proc/pressure/memory` and `/proc/pressure/io` instead.

### Per-core and per-process breakdowns

```bash
mpstat -P ALL 1          # per-core %usr/%sys/%iowait/%steal — one hot core, or all of them?
pidstat -u 1             # per-process CPU over time, sampled (top is a snapshot)
pidstat -t -p PID 1      # per-thread, once you've found the process
```

### Going deeper: what is the hot code actually doing?

```bash
perf top -p PID                                   # live, sampled, hottest functions right now
perf record -F 99 -g -p PID -- sleep 30; perf report   # capture 30s, analyze after — build a flame graph from this
strace -c -p PID         # syscall counts + time — high sy time with a specific syscall dominating is your answer
```

### Response actions

```bash
taskset -cp PID           # check/set CPU affinity — pin a noisy process off shared cores
renice -n 10 -p PID       # deprioritize a background job without killing it
```

### Install (RHEL/Amazon Linux)

```bash
dnf install sysstat      # mpstat, pidstat, iostat, sar
dnf install perf
dnf install htop          # optional, nicer top
dnf install bcc-tools     # runqlat, profile, cpudist — eBPF, low overhead
```

## AWS-specific gotchas

- **CPU credit exhaustion on burstable instances (`t3`/`t4g`).** These instance types earn CPU credits at a baseline rate and spend them to burst above baseline. When the credit balance hits zero, the instance is hard-throttled to its baseline performance — CPU usage in `top` can look completely normal (even low!) while the application is starving, because the vCPU itself is capped. `mpstat`/`top` won't show this directly on their own — you need CloudWatch:
  ```
  CPUCreditBalance   — trending to zero is your early warning
  CPUSurplusCreditBalance / CPUSurplusCreditsCharged  — if "unlimited" mode, you're paying for the burst
  ```
  If load looks normal but throughput cratered on a `t3.*`, check credit balance before you touch the code.
- **Steal time (`%st`) means the hypervisor, not your app.** On shared/older instance families, sustained non-zero steal means you're contending with a noisy neighbor or you're oversubscribed for the instance size. On modern Nitro-based instances this is rare but not impossible — if you see it, it's a capacity/instance-type conversation, not a code review.
- **`nproc` inside a container isn't always the host's core count**, but cgroup CPU quota still throttles you even if `nproc` reports the full host. On ECS/EKS, check `cat /sys/fs/cgroup/cpu.max` (cgroup v2) — a process can be throttled by its container's CPU limit while `top` on the host shows plenty of idle capacity.

## Worked example: "Checkout API got slower, but CPU usage looks fine"

**Symptom:** p99 latency on a `t3.medium` fleet doubled over two hours. `top` shows CPU around 35% — nowhere near saturated. On-call's first instinct is "it's not CPU," and moves on to the database.

**Step 1 — check PSI, since "35% average" can still hide stalls:**
```bash
cat /proc/pressure/cpu
```
```
some avg10=61.40 avg60=58.02 avg300=40.11
```
Contradiction: aggregate usage is 35%, but processes are stalled waiting for CPU 61% of the time over the last 10s. That gap is the tell — something is capping available CPU below what `top`'s percentage suggests.

**Step 2 — this is a `t3`, so check the credit balance before anything else:**

CloudWatch → `CPUCreditBalance` for the instance: flat at 0 for the last 90 minutes, `CPUSurplusCreditsCharged` climbing (instance is in `unlimited` mode, burning real money to stay above baseline and *still* getting throttled at peak).

**Root cause:** the fleet's baseline traffic grew past what `t3.medium`'s baseline vCPU performance supports; instances have been running in permanent burst, which is fine until sustained load exhausts the credit mechanism's ability to keep up, at which point the hypervisor throttles regardless of "unlimited" billing mode.

**Fix (immediate):** move the fleet to `m6i`/`m7i` (fixed, non-burstable performance) sized for actual baseline load. **Fix (if bursty traffic is expected and genuinely intermittent):** stay on `t3` but right-size so baseline covers steady-state, and alarm on `CPUCreditBalance` well before it hits zero. **Prevention:** a CloudWatch alarm on `CPUCreditBalance < (2 × hours-to-page) × credits-per-hour` catches this days before it pages anyone.

## Cheat sheet

```bash
uptime; nproc                    # load vs cores
top / top -H -p PID              # per-thread view
vmstat 1 5                       # r (runqueue), cs, us/sy/wa/st
ps -eo pid,ppid,%cpu,stat,comm --sort=-%cpu | head
cat /proc/pressure/cpu           # PSI — real CPU starvation (kernel 4.20+)
cat /proc/loadavg /proc/softirqs
taskset -cp PID                  # CPU affinity
renice -n 10 -p PID              # deprioritise

dnf install sysstat      # mpstat, pidstat, iostat, sar
dnf install perf         # perf top / perf record
dnf install htop         # optional, nicer top
dnf install bcc-tools    # runqlat, profile, cpudist (eBPF)

mpstat -P ALL 1          # per-core — is it one core pegged or all?
pidstat -u 1             # per-process CPU over time
pidstat -t -p PID 1      # per-thread
perf top -p PID          # live hot functions
perf record -F 99 -g -p PID -- sleep 30; perf report
strace -c -p PID         # syscall count/time (sy high)

# AWS: check CloudWatch CPUCreditBalance / CPUSurplusCreditsCharged on t3/t4g before assuming code regression
```
