# Memory (RAM) Troubleshooting

Memory problems on Linux have a uniquely misleading first symptom: `free -h` almost always shows "low free memory," even on a perfectly healthy box, because the kernel aggressively uses spare RAM for page cache. The actual skill here is telling *reclaimable* cache from *real* pressure, finding which process actually owns the memory, and knowing which of the three things that can kill a process actually did it.

## 1-Minute Summary

- `free -h` always looks alarming — read **`available`**, not `free`. `cache` is reclaimable disk pages, not a leak.
- `vmstat`'s `si`/`so` non-zero means the kernel is *actively* swapping right now — the clearest real signal of pressure, expensive because every page is a disk round-trip.
- `cat /proc/pressure/memory` — `full avg10` rising means *every* runnable process is stalled. Not available on Amazon Linux 2 (kernel 4.14).
- RSS overcounts shared memory across processes — use `/proc/PID/smaps_rollup` (`Pss`) for the honest per-process number.
- Process vanished? Check `dmesg -T | grep -iE 'oom|killed'` *first* — if the OOM killer already fired, live tools will show a perfectly healthy system.
- **Three different things can kill a process for memory:** the global OOM killer, the **cgroup** OOM killer, and **`systemd-oomd`** (PSI-driven, userspace, on by default in Ubuntu 22.04+ and Fedora). They log in different places.
- **Containers:** a cgroup limit can OOM-kill a task while the host's `free -h` and CloudWatch `MemoryUtilization` look completely fine.

## Methodology

1. **Is "low free" actually a problem, or just cache doing its job?** `free -h -w`, look at `available` not `free`.
2. **Did something already die?** Check `dmesg` and the cgroup OOM counter before you spend time hunting live — it may already be over.
3. **Is there real memory pressure right now?** `/proc/pressure/memory`, or `si`/`so` in `vmstat` as the fallback.
4. **At which level?** Host-wide, or one cgroup? These have completely different fixes and only one of them shows in host metrics.
5. **Which process, honestly?** RSS overcounts shared memory — use PSS.
6. **Anonymous memory, or something else?** A "leak" in page cache, slab, or mmap'd files isn't the same bug as a heap leak.

## Commands, explained

### `free -h -w` — start here, but read it correctly

```bash
free -h -w
```
```
              total        used        free      shared     buffers       cache   available
Mem:           7.8G        2.1G        180M         45M        210M        5.3G        5.4G
Swap:            0B          0B          0B
```
`free` (180M) looks alarming. **`available`** (5.4G) is the number that matters — it's the kernel's own estimate of memory it can hand to a new process right now, including cache it would happily evict. `cache` (5.3G) is disk pages the kernel is holding onto opportunistically because nothing else needed the RAM; it's reclaimed instantly under pressure, not a leak. The `-w` flag splits `buffers` from `cache` (older `free` merges them into one confusing column).

**Actual signal of trouble:** `available` shrinking over time, or approaching zero. A single reading tells you almost nothing; a trend tells you everything.

**`/proc` alternative:**
```bash
grep -E 'MemTotal|MemFree|MemAvailable|Cached|Dirty|Writeback|Slab|Committed_AS' /proc/meminfo
```
`free` is a thin formatter over `/proc/meminfo`. Fields worth knowing beyond what `free` prints:

| Field | Means |
|---|---|
| `MemAvailable` | The kernel's own estimate — the number `free`'s `available` column shows |
| `Committed_AS` | Total memory *promised* to processes. Exceeding `CommitLimit` means overcommit is carrying you |
| `Dirty` / `Writeback` | Written-but-not-flushed pages. Large `Dirty` + slow disk = write stalls that look like memory problems |
| `Slab` / `SReclaimable` | Kernel data structures. Large and *not* reclaimable = a kernel-side leak (dentries, inodes, conntrack) |
| `AnonHugePages` | THP in use — a common source of "RSS is much larger than expected" |
| `KReclaimable` | Kernel memory the shrinkers can give back under pressure |

### `vmstat 1 5` — is the system actually swapping

```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io----
 r  b   swpd   free  buff  cache   si   so    bi    bo
 0  1  102400  95200  1200 512000  340  180    50    20
```
`si`/`so` (swap in/out) non-zero means the kernel is actively moving pages to/from swap *right now* — the clearest "you're out of memory" signal there is, and expensive: every swapped page is a disk round-trip on the critical path of whatever process needed it.

Note `swpd` (total swap used) being non-zero while `si`/`so` are zero is **not** a problem — those are pages swapped out long ago during a transient spike and never needed since. People page-fault on this constantly. The rate matters, not the total.

**`/proc` alternative:**
```bash
grep -E 'pswpin|pswpout|pgscan|pgsteal|pgmajfault|oom_kill' /proc/vmstat
```
`pgscan_*` climbing far faster than `pgsteal_*` means reclaim is working hard and finding little to free — the kernel is scanning pages it can't reclaim. That's the signature of pressure from anonymous memory (which can only go to swap) rather than cache. `oom_kill` here is a simple cumulative counter of global OOM kills, useful for a monitoring check.

### `/proc/pressure/memory` — PSI, the fast path

```bash
cat /proc/pressure/memory
```
```
some avg10=15.20 avg60=8.44 avg300=2.10
full avg10=3.10 avg60=1.02 avg300=0.20
```
`some` = at least one process stalled waiting on memory; `full` = *all* runnable processes stalled simultaneously (much worse — the whole system is blocked, not just one task). A rising `full` number is a much stronger "act now" signal than any `free -h` output.

Same availability caveat as CPU: **no `/proc/pressure` on Amazon Linux 2's 4.14 kernel.** Fall back to `si`/`so` and `pgscan`/`pgsteal` ratios there.

Per-cgroup PSI also exists on cgroup v2, and it's the better signal for a container:
```bash
cat /sys/fs/cgroup/memory.pressure
```

### Finding the actual owner — RSS lies, PSS tells the truth

```bash
ps -eo pid,rss,vsz,comm --sort=-rss | head
```
RSS (Resident Set Size) counts shared library pages against *every* process that maps them — ten worker processes sharing the same 200MB library each show +200MB in RSS, even though it's one 200MB region in physical memory. Summing RSS across processes wildly overcounts. VSZ is worse: it's address space *reserved*, including memory never touched, so a Java process routinely shows tens of GB of VSZ on a 4GB box. Neither is the answer.

```bash
cat /proc/PID/smaps_rollup
```
```
Rss:              412300 kB
Pss:              184320 kB
Private_Clean:      8192 kB
Private_Dirty:    102400 kB
Swap:              20480 kB
```
`Pss` (Proportional Set Size) divides shared pages fairly across the processes mapping them — this is the number to trust when comparing processes or summing to "how much RAM is this service family actually using."

`Private_Dirty` is the sharper number for leak hunting: it's memory that belongs to *this* process alone and **cannot be reclaimed** without killing it or having it free the memory. Cache and shared library text can be evicted; `Private_Dirty` cannot. If `Private_Dirty` grows monotonically across hours, that's your leak.

Sum PSS across a process family in one line:
```bash
for p in $(pgrep -f myapp); do awk '/^Pss:/{s+=$2} END{print s}' /proc/$p/smaps_rollup; done \
  | awk '{s+=$1} END {printf "%.1f MB\n", s/1024}'
```

```bash
cat /proc/PID/status | grep -E 'VmRSS|VmSwap|VmData|Threads'   # cheaper single read
wc -l < /proc/PID/maps                                          # mapping count — a runaway count is itself an mmap leak
```

### Did it already happen? Three places to look

```bash
dmesg -T | grep -iE 'oom|killed process'
```
```
[Tue Aug 26 09:14:02 2026] node invoked oom-killer: gfp_mask=0x..., order=0, oom_score_adj=0
[Tue Aug 26 09:14:02 2026] Memory cgroup out of memory: Killed process 4821 (java) total-vm:6291456kB, anon-rss:5872300kB
```
The wording distinguishes the two kernel paths: **"Out of memory: Killed process"** is global exhaustion; **"Memory cgroup out of memory"** is a cgroup limit, and the host may have had gigabytes free. Read the line, don't just grep for "killed."

Always check this **first** when a process "just disappeared" — if the OOM killer already fired, live memory tools will show a perfectly healthy system, because the problem process is gone.

**The cgroup counter (cgroup v2)** — cleaner than parsing dmesg, and it survives a full ring buffer:
```bash
cat /sys/fs/cgroup/memory.events
```
```
low 0
high 12043
max 8821
oom 14
oom_kill 3
```
`max` is the number of times allocation hit the hard limit; `oom_kill` is how many processes actually died. **`high` climbing with `oom_kill` at zero is the interesting case:** the cgroup is being throttled into reclaim constantly without dying — a slow, invisible performance problem that no alarm on OOM kills will ever catch.

**`systemd-oomd`** — the third killer, and the one nobody expects. It's a userspace daemon that uses PSI to kill cgroups *before* the kernel OOM killer would, and it's enabled by default on Ubuntu 22.04+ and Fedora. It leaves nothing in `dmesg`:
```bash
systemctl status systemd-oomd
journalctl -u systemd-oomd
oomctl                             # current pressure and configured limits per cgroup
```
If a service keeps dying with no kernel OOM message, this is usually why. Debian-family also sometimes ships `earlyoom` (`apt install earlyoom`), which behaves similarly but logs to its own unit.

### Influencing who dies

```bash
cat /proc/PID/oom_score            # 0–1000, kernel's badness ranking
cat /proc/PID/oom_score_adj        # -1000 (never kill) .. +1000 (kill first)
echo -500 > /proc/PID/oom_score_adj
```
The systemd equivalent, which is what you'd actually put in a unit file:
```ini
[Service]
OOMScoreAdjust=-500
MemoryMax=2G
MemoryHigh=1800M          # throttle-and-reclaim threshold, softer than MemoryMax
```
Protecting `sshd` with a negative `OOMScoreAdjust` is a cheap, high-value habit on any box you might need to get back into during an incident.

### Kernel-side consumers

```bash
cat /proc/meminfo               # Slab, SReclaimable, SUnreclaim
slabtop -s c                    # sorted by cache size — dentries, inodes, network buffers
cat /proc/slabinfo              # the raw source slabtop reads
```
Large, growing `SUnreclaim` is a genuine kernel leak — rare, but real. The common non-bug version is `dentry`/`inode_cache` ballooning on a box that touches millions of files (a build server, a log processor); that's reclaimable and mostly harmless, though you can force it:
```bash
sync; echo 2 > /proc/sys/vm/drop_caches     # dentries+inodes. Diagnostic only, never a fix in production
```

### Tuning knobs worth knowing

```bash
cat /proc/sys/vm/swappiness              # 60 default; 10 for latency-sensitive, 1 to nearly disable
cat /proc/sys/vm/min_free_kbytes         # reclaim watermark — too low causes allocation stalls under burst
cat /proc/sys/vm/overcommit_memory       # 0 heuristic (default), 1 always, 2 strict against CommitLimit
cat /proc/sys/vm/dirty_ratio             # see disk.md — write stalls masquerade as memory pressure
```

### Install

| Tool | RHEL-family (`dnf`) | Debian-family (`apt`) |
|---|---|---|
| `free`, `vmstat`, `slabtop` | `procps-ng` | `procps` ← **name differs** |
| `sar -r`, `pidstat -r` | `sysstat` | `sysstat` |
| `numastat`, `numactl` | `numactl` | `numactl` |
| `valgrind` (dev/staging only) | `valgrind` | `valgrind` |
| eBPF `memleak`, `oomkill` | `bcc-tools` | `bpfcc-tools` |
| `smem` (PSS reporting, convenient) | `smem` (EPEL) | `smem` |

## AWS-specific gotchas

- **Swap is off by default on EC2 AMIs.** Amazon Linux 2, AL2023, and Ubuntu cloud images all ship with no swap. If `vmstat`'s `si`/`so` are non-zero, someone deliberately configured it — worth asking why, since on ephemeral, replaceable EC2 instances, swapping to an EBS volume is almost always worse than right-sizing memory or letting the OOM killer act. It converts a fast, loud failure into a slow, quiet one.
- **CloudWatch has no memory metric by default.** `MemoryUtilization` does not exist unless you've installed the CloudWatch agent — the hypervisor cannot see guest memory. "Memory looks fine in CloudWatch" from someone who never installed the agent means "we have no data."
- **Containers on ECS/EKS have their own ceiling, independent of the host.** A container can be OOM-killed by its cgroup limit while the host still shows plenty of free RAM. Check `docker inspect <container> | grep -i oomkilled` or `kubectl describe pod` (`Last State: Terminated, Reason: OOMKilled, Exit Code: 137`) — host-level tools won't show this at all.
- **No swap + a container memory limit turns a slow leak into a sudden, hard kill** with no gradual degradation to warn you — which is why OOM-focused container alarms matter more on AWS than "low free memory" alarms do.
- **JVM/runtime heap must be sized against the container limit, not the host's.** Modern JVMs (11+) are container-aware and respect the cgroup limit via `-XX:MaxRAMPercentage`, but only if you *use* it — a hardcoded `-Xmx` set from host RAM inside a memory-limited task is a classic self-inflicted OOM. The same trap exists for Node's `--max-old-space-size` and Go's `GOMEMLIMIT`.
- **Fargate has no host to inspect.** You get no `dmesg`, no node access, no `docker inspect`. Your only evidence is the task stop reason (`OutOfMemoryError: Container killed due to memory usage`) and whatever the application logged before it died — which makes instrumenting heap metrics *inside* the container non-optional there.

## Worked example: "Java service OOM-killed nightly, node memory looks fine"

**Symptom:** a pod in an EKS cluster (EC2 node group, not Fargate) is killed every night around 02:00. The node's CloudWatch memory metrics never look alarming, and there's no kernel OOM message in the node's `dmesg`.

**Step 1 — establish *which* killer fired.**
```bash
kubectl describe pod checkout-7d9f
```
```
Last State:   Terminated
Reason:       OOMKilled
Exit Code:    137
```
On the node itself:
```bash
dmesg -T | grep -i "out of memory"
```
```
Memory cgroup out of memory: Killed process 4821 (java)
```
The wording is **"Memory cgroup out of memory"**, not "Out of memory" — so this is the container's limit, not host exhaustion. That's consistent with the node looking healthy, and it immediately rules out "add more nodes" as a fix.

**Step 2 — confirm from the cgroup's own counters, and check whether it was already struggling before it died.**
```bash
POD_CG=/sys/fs/cgroup/kubepods.slice/.../memory.events
cat $POD_CG
```
```
high 84120
max 231
oom 14
oom_kill 3
```
`high` in the tens of thousands is the important number: the container had been hammering its reclaim threshold constantly, long before it ever hit the hard limit. This wasn't a sudden spike — it had been running on the edge for hours, and only crossed over when something added a little more.

**Step 3 — compare the limit to what the runtime thinks it has.**
```bash
cat /sys/fs/cgroup/kubepods.slice/.../memory.max     # 1073741824  (1 GiB)
kubectl exec checkout-7d9f -- jcmd 1 VM.flags | tr ' ' '\n' | grep -i maxheap
```
```
-XX:MaxHeapSize=939524096      (896 MiB)
```
That leaves ~128 MiB for everything *outside* the Java heap — thread stacks, metaspace, code cache, direct byte buffers, GC structures, and the JVM itself. That's tight for any service and gets tighter under load, because each additional thread costs a stack allocation outside the heap.

**Step 4 — confirm the mechanism with a rollup under load.**
```bash
kubectl exec checkout-7d9f -- cat /proc/1/smaps_rollup
```
Non-heap `Private_Dirty` grows steadily during the nightly batch window as the worker pool expands, pushing total RSS across the cgroup limit while heap usage itself stays well inside `-Xmx`. The JVM never sees a problem; the cgroup does.

**Root cause:** heap sized at 87% of the container limit with no margin for non-heap memory, plus a nightly batch job that transiently raises thread count and therefore non-heap usage just enough to cross the limit.

**Fix:** replace the hardcoded `-Xmx` with `-XX:MaxRAMPercentage=70`, so the heap tracks the container limit automatically and can't drift out of sync when someone resizes the pod. Cap the batch worker pool. Raise the memory limit if cost allows.
**Prevention:** alarm on **container-level** memory (Container Insights, `kubectl top pod`) rather than only node-level metrics — the two diverge exactly in cases like this. Better still, alarm on `memory.events`' `high` counter, which was rising for hours before the first kill: it's a leading indicator where `oom_kill` is a trailing one.

## Cheat sheet

```bash
# Read it correctly
free -h -w                      # available, NOT free. -w splits buff and cache
grep -E 'MemAvailable|Committed_AS|Dirty|Slab|SUnreclaim' /proc/meminfo
vmstat 1 5                      # si/so = swap RATE (swpd total alone means nothing)
cat /proc/pressure/memory       # PSI: 'full' rising = whole system stalled (not on AL2)
cat /sys/fs/cgroup/memory.pressure    # per-cgroup PSI — better signal in a container

# Who owns it, honestly
ps -eo pid,rss,vsz,comm --sort=-rss | head    # RSS overcounts, VSZ is meaningless
cat /proc/PID/smaps_rollup      # Pss = fair share; Private_Dirty = unreclaimable, the leak signal
cat /proc/PID/status            # VmRSS, VmSwap, Threads — cheap single read
wc -l < /proc/PID/maps          # mapping count — runaway = mmap leak
smem -tk -s pss                 # PSS across all processes, sorted

# Did it already die — check all three killers
dmesg -T | grep -iE 'oom|killed process'      # "Out of memory" = global; "Memory cgroup" = limit
cat /sys/fs/cgroup/memory.events              # oom_kill (fired) + high (throttling, leading indicator)
grep oom_kill /proc/vmstat                    # global cumulative counter
journalctl -u systemd-oomd ; oomctl           # Ubuntu 22.04+/Fedora — kills with NO dmesg line
kubectl describe pod X | grep -A3 'Last State'
docker inspect X | grep -i oomkilled

# Who dies next
cat /proc/PID/oom_score /proc/PID/oom_score_adj
echo -500 > /proc/PID/oom_score_adj           # or OOMScoreAdjust= in the unit file

# Kernel-side
slabtop -s c ; cat /proc/slabinfo
grep -E 'pgscan|pgsteal|pgmajfault' /proc/vmstat   # scan >> steal = reclaim failing

# Limits (cgroup v2)
cat /sys/fs/cgroup/memory.max /sys/fs/cgroup/memory.current /sys/fs/cgroup/memory.stat
# cgroup v1: /sys/fs/cgroup/memory/memory.limit_in_bytes, memory.usage_in_bytes

# Tuning
cat /proc/sys/vm/{swappiness,min_free_kbytes,overcommit_memory}

# Install
dnf install procps-ng sysstat numactl valgrind bcc-tools     # RHEL-family
apt  install procps    sysstat numactl valgrind bpfcc-tools  # Debian-family (procps, not procps-ng)

# AWS
# No swap by default. CloudWatch has NO memory metric without the agent installed.
# Container limit != host memory. Size -Xmx / MaxRAMPercentage against the CGROUP, not the node.
```