# CPU Troubleshooting

"High CPU" is often the *symptom* reported, rarely the actual diagnosis. The real question is always one of: too much work, work stuck waiting on something else (which shows up as CPU pressure but isn't a CPU problem), one runaway thread, a cgroup quota capping you, or — on AWS specifically — a burstable instance that ran out of credit. This guide walks through telling those apart.

## 1-Minute Summary

- `uptime` + `nproc` first — load average is a count of runnable/waiting processes, not a percentage; compare it to your core count before deciding anything is "high."
- `cat /proc/pressure/cpu` is the fastest honest answer to "is CPU actually the bottleneck" — but confirm PSI exists on this kernel first (Amazon Linux 2 doesn't have it).
- One hot core vs. all of them changes the fix entirely — check with `mpstat -P ALL 1`, not just aggregate `top`.
- `strace -c -p PID` / `perf top -p PID` tell you *what* the hot thread is actually doing, once you've found it.
- **In a container:** `cat /sys/fs/cgroup/cpu.stat` → `nr_throttled` climbing means the cgroup quota is the ceiling, not the hardware.
- **On `t3`/`t4g`:** in **standard** mode a zero credit balance hard-throttles you to baseline while `top` looks normal. In **unlimited** mode (the default) you are *not* throttled — you're billed. Two different problems, two different fixes.

## Methodology

1. **Is the system actually loaded, relative to its core count?** `uptime` + `nproc`.
2. **Is it one core pegged, or all of them?** Aggregate `top` numbers hide this — go per-core.
3. **Is it user code, kernel time, or I/O wait masquerading as "CPU"?** `us` vs `sy` vs `wa` vs `st`.
4. **Is something capping you below the hardware?** cgroup quota, or burstable credit exhaustion.
5. **Which process, which thread?** Narrow from system → process → thread.
6. **Why is that thread hot?** `perf`/`strace` to see what it's actually doing.

Steps 4 and 5 are the ones people skip. A capped CPU and a busy CPU look identical in `top`; the difference is entirely in counters `top` doesn't read.

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

Because Linux counts uninterruptible sleep (`D` state) in load average, **a box with load 40 and idle CPUs is usually an I/O problem, not a CPU problem** — that's an NFS mount that went away, or an EBS volume that stopped responding. Load average alone can't distinguish them; that's what step 3 is for.

**`/proc` alternative:**
```bash
cat /proc/loadavg
```
```
8.42 6.15 3.20 9/1832 44021
```
Fields 4 and 5 are the bonus: `9/1832` is *running/total* tasks, and `44021` is the last PID allocated. A last-PID that jumps by thousands between two reads is a fork storm — something is spawning processes in a loop, which `top` will show as diffuse CPU with no obvious owner.

```bash
nproc                          # respects CPU affinity
cat /proc/cpuinfo | grep -c ^processor   # what the kernel sees, ignoring affinity
lscpu                          # sockets, cores, threads, cache, NUMA nodes, flags
```

### `vmstat` — the run queue, without the noise

```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free  buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 812340  40212 2211200    0    0     2    18  980 2400 71  9  0  0 20
```
The `r` column is the run queue *right now* (processes ready to run but waiting for a core) — 6 runnable on a 4-core box confirms real CPU contention, not just a noisy `uptime` average. The `b` column is processes blocked in uninterruptible sleep; if `b` is high and `r` is near zero, stop reading this guide and go to [disk](disk.md).

Read `us`/`sy`/`wa`/`st` together:

| Column | Meaning | Where to go next |
|---|---|---|
| `us` high | Your application code | `perf top`, profiling |
| `sy` high | Kernel — syscalls, context switching, softirqs | `strace -c`, check `cs` and `in` |
| `wa` high | CPU idle but blocked on I/O | [disk.md](disk.md) — this is not a CPU problem |
| `st` high | **Steal** — CPU the hypervisor didn't give you | AWS-specific, see below |

`cs` (context switches) growing far faster than `in` (interrupts) usually means thread thrash — too many runnable threads for the core count, or heavy lock contention where threads keep sleeping and waking.

**`/proc` alternative:** `vmstat` computes everything from `/proc/stat` and `/proc/vmstat`.
```bash
cat /proc/stat | head -5
```
```
cpu  2314 0 1201 981234 412 0 88 4012 0 0
```
Fields after `cpu` are, in order, jiffies spent in: user, nice, system, idle, iowait, irq, softirq, **steal**, guest, guest_nice. Take two reads a second apart and diff them and you've reimplemented `mpstat` — which is exactly what you do when writing a monitoring check on a box with no `sysstat` installed.

### `top` / `top -H -p PID` — who, and which thread

```bash
top -H -p 4821          # -H shows individual threads instead of aggregated processes
```
Press `1` in plain `top` to see per-core breakdown instead of the aggregate — this is how you catch "one thread pegging one core" versus "genuinely load-balanced across all four," which look identical in the aggregate view but have completely different fixes (fix the code vs. add capacity).

Other keys worth knowing under pressure: `P` sorts by CPU, `M` by memory, `f` opens the field selector (add `P` = last-used-core, and `nTH` = thread count).

```bash
ps -eo pid,ppid,%cpu,stat,nlwp,comm --sort=-%cpu | head
```
`nlwp` is the thread count — a process at 400% CPU on a 4-core box with 200 threads is a different conversation than one with 4.

### `/proc/pressure/cpu` — PSI, the fast path

```bash
cat /proc/pressure/cpu
```
```
some avg10=42.10 avg60=38.55 avg300=21.03 total=98234123
```
`some avg10=42.10` means: over the last 10 seconds, *some* process was stalled waiting for a CPU 42% of the time. This is the most direct answer to "is CPU actually the bottleneck" — no math against core count required, no guessing from load average. If PSI-CPU is low but users report slowness, stop looking at CPU and check `/proc/pressure/memory` and `/proc/pressure/io` instead.

> **Availability check first.** PSI needs kernel 4.20+ with `CONFIG_PSI=y`. **Amazon Linux 2 ships 4.14 by default and has no `/proc/pressure` at all.** AL2023, Ubuntu 20.04+, and RHEL 9 have it. If the kernel is new enough but the path is missing, the build set `CONFIG_PSI_DEFAULT_DISABLED=y` — add `psi=1` to the cmdline and reboot.
> ```bash
> ls /proc/pressure/ 2>/dev/null || echo "PSI unavailable — fall back to vmstat r vs nproc"
> ```

Note there is no `full` line for CPU (unlike memory and I/O) — by definition, if every task were stalled on CPU there'd be nothing running to stall.

### Per-core and per-process breakdowns

```bash
mpstat -P ALL 1          # per-core %usr/%sys/%iowait/%steal — one hot core, or all of them?
pidstat -u 1             # per-process CPU over time, sampled (top is a snapshot)
pidstat -t -p PID 1      # per-thread, once you've found the process
sar -u 1 5               # same data; sar -u -f /var/log/sa/saNN reads *historical* data
```
`sar` deserves a specific mention: it's the only tool here that answers "what did CPU look like at 3am when the alarm fired," because `sysstat` writes samples to disk on a cron/timer. Verify the collector is actually enabled, since it usually isn't by default:

**RHEL-family:**
```bash
dnf install sysstat
systemctl enable --now sysstat        # writes to /var/log/sa/
```
**Debian-family:**
```bash
apt install sysstat
sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
systemctl enable --now sysstat        # writes to /var/log/sysstat/
```
Note the log path differs between families — a small thing that wastes five minutes at 3am.

### Going deeper: what is the hot code actually doing?

```bash
perf top -p PID                                        # live, sampled, hottest functions right now
perf record -F 99 -g -p PID -- sleep 30; perf report   # capture 30s, analyze after — flame graph source
perf stat -p PID -- sleep 5                            # IPC, cache misses, branch misses
strace -c -p PID                                       # syscall counts + time — high sy time, one syscall dominating
```
`perf stat`'s **IPC** (instructions per cycle) is the number that separates "genuinely CPU-bound, needs algorithmic work" (IPC high, ~2+) from "stalling on memory access" (IPC low, <1, with high cache-miss rate) — two problems that look identical in `top` and have completely different fixes.

### Response actions

```bash
taskset -cp PID                 # READ current affinity
taskset -cp 0-3 PID             # SET affinity — pin a noisy process off shared cores
renice -n 10 -p PID             # deprioritize a background job without killing it
chrt -p PID                     # scheduling policy and priority
systemctl set-property foo.service CPUQuota=50%    # cap a unit, live, persists to drop-in
```
`systemd-run` is the underrated one for ad-hoc containment of a runaway batch job:
```bash
systemd-run --scope -p CPUQuota=25% -p MemoryMax=1G ./import-job.sh
```

### Install

| Tool | RHEL-family (`dnf`) | Debian-family (`apt`) |
|---|---|---|
| `mpstat`, `pidstat`, `sar` | `sysstat` | `sysstat` |
| `perf` | `perf` | `linux-tools-common linux-tools-$(uname -r)` |
| `htop` | `htop` | `htop` |
| eBPF tools | `bcc-tools` | `bpfcc-tools` |
| `lscpu`, `taskset`, `chrt` | `util-linux` (preinstalled) | `util-linux` (preinstalled) |
| `numactl` | `numactl` | `numactl` |

The `perf` split is the one that catches people: on Debian-family, `perf` is versioned against the running kernel, and on EC2 you need the AWS-specific package:
```bash
apt install linux-tools-aws linux-tools-$(uname -r)   # Ubuntu on EC2
```
Without that exact match, `perf` prints a warning and refuses to run.

## AWS-specific gotchas

### Burstable instances: know which mode you're in *before* you diagnose

`t2`/`t3`/`t3a`/`t4g` earn CPU credits at a fixed rate per hour and spend them to run above their baseline. What happens at zero depends entirely on the credit mode, and the two produce opposite symptoms:

| Mode | Behaviour at zero credits | Symptom | The metric that proves it |
|---|---|---|---|
| **standard** | Hard-throttled to baseline vCPU performance | Throughput cratered, `top` looks *normal or low* | `CPUCreditBalance` flat at 0 |
| **unlimited** (default for T3/T3a/T4g) | Keeps bursting; you're billed for surplus | Performance fine, **bill** climbing | `CPUSurplusCreditBalance` rising, `CPUSurplusCreditsCharged` non-zero |

In unlimited mode the instance can spend surplus credits up to the maximum it could earn in 24 hours without immediate charge; beyond that, the surplus is billed at a flat rate per vCPU-hour. If the 24-hour average CPU utilisation stays above baseline, the instance can never earn enough to pay the surplus down, so the charge is continuous rather than a one-off.

There is a second-order trap in unlimited mode: after a long period above baseline, `CPUCreditBalance` sits at zero even once load drops, because earned credits go to paying down `CPUSurplusCreditBalance` first. An instance that looks "permanently out of credits" at 5% CPU is usually paying off a debt, not broken.

Check the mode from the instance itself, not just the console:
```bash
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
aws ec2 describe-instance-credit-specifications --instance-ids "$IID"
```

**None of this is visible from inside the guest.** There is no `/proc` file, no `dmesg` line, no `%steal` spike for standard-mode throttling — the vCPU is simply slower. If throughput dropped on a `t*` instance and the guest looks healthy, check CloudWatch before you read a single line of application code.

### Steal time (`%st`) means the hypervisor, not your app

Sustained non-zero steal on shared/older instance families means you're contending with a noisy neighbour or oversubscribed for the instance size. On modern Nitro-based instances this is rare but not impossible — if you see it, it's a capacity/instance-type conversation, not a code review. Read it from field 8 of `/proc/stat` if `mpstat` isn't installed.

### Containers: cgroup quota is the real ceiling

`nproc` respects CPU affinity but **not** cgroup CPU quota — a container can report 64 CPUs and still be limited to the equivalent of 0.5.

**cgroup v2** (AL2023, Ubuntu 22.04+, EKS 1.25+ on modern AMIs):
```bash
cat /sys/fs/cgroup/cpu.max
```
```
50000 100000
```
Read as `quota period` in microseconds: 50ms of CPU per 100ms window = **0.5 CPU**. `max 100000` means unlimited.

```bash
cat /sys/fs/cgroup/cpu.stat
```
```
usage_usec 84210000
nr_periods 91238
nr_throttled 41022
throttled_usec 190034000
```
**`nr_throttled` climbing is the whole answer.** The application isn't slow; it's being stopped ~45% of enforcement periods. In Kubernetes terms: the CPU *limit* is too low (or set at all — CPU limits cause throttling even below the limit due to burst behaviour within the 100ms window).

**cgroup v1** (Amazon Linux 2, older EKS):
```bash
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us      # -1 = unlimited
cat /sys/fs/cgroup/cpu/cpu.cfs_period_us
cat /sys/fs/cgroup/cpu/cpu.stat              # nr_throttled, throttled_time (nanoseconds)
```
Check which version you're on before hunting for paths:
```bash
stat -fc %T /sys/fs/cgroup/     # cgroup2fs = v2, tmpfs = v1
```

## Worked example: "Checkout API got slower, but CPU usage looks fine"

**Symptom:** p99 latency on a `t3.medium` fleet doubled over two hours. `top` shows CPU around 35% — nowhere near saturated. On-call's first instinct is "it's not CPU," and they move on to the database.

**Step 1 — check PSI, since "35% average" can hide stalls.**
```bash
cat /proc/pressure/cpu
```
```
some avg10=61.40 avg60=58.02 avg300=40.11
```
Contradiction: aggregate usage is 35%, but processes were stalled waiting for CPU 61% of the time over the last 10s. That gap is the tell — something is capping available CPU below what `top`'s percentage suggests. (This box runs AL2023; on an AL2 node this step is unavailable and you'd substitute `vmstat 1` and look for `r` consistently above `nproc`.)

**Step 2 — rule out the cgroup, since these run as containers.**
```bash
cat /sys/fs/cgroup/cpu.max        # max 100000  → no quota
cat /sys/fs/cgroup/cpu.stat       # nr_throttled: 0
```
Not container throttling.

**Step 3 — this is a `t3`, so establish the credit mode before anything else.**
```bash
aws ec2 describe-instance-credit-specifications --instance-ids i-0abc...
```
```
"CpuCredits": "standard"
```
**Standard** mode — so a zero balance means hard throttling, and the PSI/utilisation contradiction fits exactly.

CloudWatch → `CPUCreditBalance`: flat at 0 for the last 90 minutes, `CPUCreditUsage` pinned at the earn rate.

**Root cause:** the fleet's baseline traffic grew past what `t3.medium`'s baseline vCPU performance supports. Instances had been running in permanent burst; once sustained load exhausted the balance, the hypervisor capped each vCPU at baseline. The guest sees a slower CPU, not a busier one, which is why `top` never looked alarming.

**Fix (immediate):** move the fleet to `m6i`/`m7i` — fixed, non-burstable performance sized for actual baseline load.
**Fix (if traffic is genuinely bursty):** stay on `t3` but right-size so baseline covers steady-state, and consider `unlimited` mode with a cost alarm — accepting a predictable bill instead of unpredictable throttling.
**Prevention:** a CloudWatch alarm on `CPUCreditBalance` trending toward zero catches this days before it pages anyone. Add `CPUSurplusCreditsCharged` to the same dashboard so a later switch to unlimited doesn't silently turn a latency problem into a billing one.

## Cheat sheet

```bash
# First 5 seconds
uptime; nproc                    # load vs cores
cat /proc/loadavg                # + running/total tasks, last PID (fork storms)
vmstat 1 5                       # r (runqueue), b (blocked), cs, us/sy/wa/st
cat /proc/pressure/cpu           # PSI — real CPU starvation (kernel 4.20+, NOT on AL2)
ls /proc/pressure/ || echo "no PSI"

# Narrow down
top / top -H -p PID              # per-thread view; press 1 for per-core
ps -eo pid,ppid,%cpu,stat,nlwp,comm --sort=-%cpu | head
mpstat -P ALL 1                  # per-core — one pegged or all?
pidstat -u 1                     # per-process over time
pidstat -t -p PID 1              # per-thread
sar -u 1 5                       # + historical: sar -u -f /var/log/sa/saNN

# /proc alternatives
cat /proc/stat                   # user nice system idle iowait irq softirq STEAL
cat /proc/PID/stat               # per-process utime/stime (fields 14,15)
cat /proc/PID/schedstat          # time on cpu, time waiting on runqueue, timeslices
cat /proc/softirqs               # NET_RX imbalance across cores
cat /proc/interrupts

# Why is it hot
perf top -p PID
perf record -F 99 -g -p PID -- sleep 30; perf report
perf stat -p PID -- sleep 5      # IPC: <1 = memory-stalled, >2 = genuinely compute-bound
strace -c -p PID                 # syscall count/time (when sy is high)

# Containers / cgroups
stat -fc %T /sys/fs/cgroup/                  # cgroup2fs vs tmpfs
cat /sys/fs/cgroup/cpu.max                   # v2: "quota period" µs
cat /sys/fs/cgroup/cpu.stat                  # v2: nr_throttled ← the answer
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us      # v1
cat /sys/fs/cgroup/cpu/cpu.stat              # v1: nr_throttled, throttled_time

# Response
taskset -cp PID                  # read affinity
taskset -cp 0-3 PID              # set affinity
renice -n 10 -p PID
systemd-run --scope -p CPUQuota=25% ./job.sh

# Install
dnf install sysstat perf htop bcc-tools numactl          # RHEL-family
apt  install sysstat linux-tools-aws linux-tools-$(uname -r) htop bpfcc-tools numactl   # Debian-family
systemctl enable --now sysstat   # sar history: /var/log/sa (RHEL) vs /var/log/sysstat (Debian)

# AWS
# t3/t4g STANDARD  → CPUCreditBalance == 0 means THROTTLED (top looks fine)
# t3/t4g UNLIMITED → not throttled; CPUSurplusCreditsCharged means you're PAYING
aws ec2 describe-instance-credit-specifications --instance-ids i-...
# %st sustained → noisy neighbour / oversubscribed instance type
```