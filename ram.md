# Memory (RAM) Troubleshooting

Memory problems on Linux have a uniquely misleading first symptom: `free -h` almost always shows "low free memory," even on a perfectly healthy box, because the kernel aggressively uses spare RAM for page cache. The actual skill here is telling *reclaimable* cache from *real* pressure, and finding which process actually owns the memory before you reach for `kill`.

## Methodology

1. **Is "low free" actually a problem, or just cache doing its job?** `free -h -w`, look at `available` not `free`.
2. **Is there real memory pressure right now?** `/proc/pressure/memory`.
3. **Is the system swapping?** `si`/`so` in `vmstat` — on EC2 this is almost always a configuration mistake, not intended behavior.
4. **Which process, honestly?** RSS overcounts shared memory — use PSS.
5. **Did the OOM killer already act?** Check `dmesg` before you spend time hunting live — it may already be over.

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

**Actual signal of trouble:** `available` shrinking over time, or approaching `used`.

### `vmstat 1 5` — is the system actually swapping

```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io----
 r  b   swpd   free  buff  cache   si   so    bi    bo
 0  1  102400  95200  1200 512000  340  180    50    20
```
`si`/`so` (swap in/out) non-zero means the kernel is actively moving pages to/from swap *right now* — this is the clearest "you're out of memory" signal there is, and it's expensive: every swapped page is a disk round-trip on the critical path of whatever process needed it.

### `/proc/pressure/memory` — PSI, the fast path

```bash
cat /proc/pressure/memory
```
```
some avg10=15.20 avg60=8.44 avg300=2.10
full avg10=3.10 avg60=1.02 avg300=0.20
```
`some` = at least one process stalled waiting on memory; `full` = *all* runnable processes stalled simultaneously (much worse — the whole system is blocked, not just one task). A rising `full` number is a much stronger "act now" signal than any `free -h` output.

### Finding the actual owner — RSS lies, PSS tells the truth

```bash
ps -eo pid,rss,vsz,comm --sort=-rss | head
```
RSS (Resident Set Size) counts shared library pages against *every* process that maps them — ten worker processes sharing the same 200MB library each show +200MB in RSS, even though it's one 200MB region in physical memory. Summing RSS across processes wildly overcounts. For an honest per-process number:

```bash
cat /proc/PID/smaps_rollup
```
```
Pss:              184320 kB
Private_Dirty:    102400 kB
```
`Pss` (Proportional Set Size) divides shared pages fairly across the processes mapping them — this is the number to trust when comparing processes or summing to "how much RAM is this service family actually using."

```bash
cat /proc/PID/status            # VmRSS, VmSwap per process, cheaper single read
grep -c . /proc/PID/smaps       # rough count of distinct mappings — a runaway count can itself indicate a leak (mmap leak)
```

### Did it already happen?

```bash
dmesg -T | grep -iE 'oom|killed process'
```
```
[Tue Aug 26 09:14:02 2026] Out of memory: Killed process 4821 (java) total-vm:6291456kB, anon-rss:5872300kB
```
Always check this **first** when a process "just disappeared" — if the OOM killer already fired, live memory tools will show a perfectly healthy system, because the problem process is gone.

### Kernel-side consumers

```bash
cat /proc/meminfo               # MemAvailable, Slab, Committed_AS, Dirty — the full picture free -h summarizes
slabtop                         # kernel slab cache consumers — catches kernel-side leaks (rare, but real: dentries, inodes, network buffers)
```

### Install

```bash
dnf install sysstat        # sar -r, pidstat -r
dnf install procps-ng      # free, vmstat, slabtop
dnf install numactl        # numastat, numactl --hardware — matters on larger/NUMA instance types
dnf install valgrind       # leak hunting, dev/staging only — heavy overhead
dnf install bcc-tools      # memleak, oomkill — eBPF, low overhead, safe for production
```

## AWS-specific gotchas

- **Swap is usually *off* by default on EC2 AMIs.** If `vmstat`'s `si`/`so` are non-zero, someone deliberately configured swap — worth asking why, since on ephemeral/replaceable EC2 instances, swapping to an EBS volume is almost always worse than just right-sizing memory or letting the OOM killer act.
- **Containers on ECS/EKS have their own memory ceiling, independent of the host.** A container can be OOM-killed by its **cgroup limit** (`memory.max` under cgroup v2) while the host EC2 instance still shows plenty of free RAM in `free -h`. Check `docker inspect <container> | grep -i oomkilled` or `kubectl describe pod` (`Last State: Terminated, Reason: OOMKilled`) — host-level tools won't show this at all.
- **No swap + container memory limit is a common combo that turns a slow leak into a sudden, hard kill** with no gradual degradation to warn you — which is why the OOM-focused CloudWatch/container-insights alarms matter more on AWS than "low free memory" alarms do.
- **JVM/runtime heap settings need to respect the container limit, not the host's.** A Java process with `-Xmx` set relative to host RAM inside a memory-limited ECS task is a classic self-inflicted OOM — the heap thinks it has more room than the cgroup will actually allow.

## Worked example: "Java service OOM-killed nightly, host memory looks fine"

**Symptom:** an ECS task on Fargate gets killed every night around the same time; the underlying instance's CloudWatch `MemoryUtilization` never looks alarming.

**Step 1 — check if it's actually the OOM killer, and at what level:**
```bash
kubectl describe pod checkout-7d9f  # (or docker inspect, if plain ECS/EC2)
```
```
Last State:   Terminated
Reason:       OOMKilled
Exit Code:    137
```
Confirmed — but the **host** shows healthy memory, which means this is the **cgroup/task memory limit**, not host exhaustion.

**Step 2 — compare the task's configured memory limit to the JVM's heap setting:**
```bash
cat /proc/PID/status | grep VmRSS
```
Task memory limit: 1024 MiB. JVM: `-Xmx896m`. That leaves only ~128MiB for everything *outside* the Java heap — thread stacks, metaspace, direct buffers, native libraries — which is tight for most services and gets tighter under load (more threads, more native buffer allocation for I/O).

**Step 3 — confirm with `smaps_rollup` under load** that non-heap RSS grows toward the ceiling during the traffic pattern that precedes the nightly kill (a batch job spinning up extra worker threads).

**Root cause:** heap sized against the container limit with no margin for non-heap memory, and a nightly batch job that transiently increases thread count (and therefore non-heap usage) just enough to cross the cgroup limit.

**Fix:** lower `-Xmx` to leave real headroom (rule of thumb: heap ≤ 65–75% of the container limit for typical JVM workloads), and/or raise the task memory limit if cost allows. **Prevention:** alarm on container-level memory (ECS Container Insights / `kubectl top pod`) rather than only host-level `MemoryUtilization` — the two diverge exactly in cases like this.

## Cheat sheet

```bash
free -h
free -h -w                      # -w splits buff and cache
vmstat 1 5                      # si/so = swap in/out
ps -eo pid,rss,vsz,comm --sort=-rss | head
top                             # then press M
cat /proc/meminfo               # MemAvailable, Slab, Committed_AS, Dirty
cat /proc/pressure/memory       # PSI — real stall time (kernel 4.20+)
cat /proc/PID/status            # VmRSS, VmSwap per process
cat /proc/PID/smaps_rollup      # Pss, Private_Dirty — the honest number
grep -c . /proc/PID/smaps       # mapping count
dmesg -T | grep -iE 'oom|killed process'
slabtop                         # kernel slab consumers

dnf install sysstat        # sar -r, pidstat -r
dnf install procps-ng      # free, vmstat, slabtop
dnf install numactl        # numastat, numactl --hardware
dnf install valgrind       # leak hunting
dnf install bcc-tools      # memleak, oomkill

# AWS: check container/cgroup memory limits (ECS/EKS) separately from host MemoryUtilization
```
