# Advanced Diagnostics: strace, perf, and eBPF

The other guides in this repo answer "*what* is slow" — CPU, memory, disk, or network. This one answers "*why*," at the syscall or kernel level, once you've narrowed the problem to a specific process. These are the tools for the moment `top` and `iostat` have told you *which* process to look at, but not *what it's actually doing*.

## 1-Minute Summary

- Confirm there's an actual queue first (`vmstat`/`mpstat`) — don't profile a process that isn't contended.
- `strace -c -p PID` before a raw trace — the syscall summary alone usually tells you the shape of the problem (I/O-bound, lock contention, or failing config lookups).
- `/proc/PID/stack` and `/proc/PID/wchan` cost nothing to read and work even on a wedged process that `strace -p` might hang trying to attach to.
- `perf stat -p PID`'s IPC number separates "genuinely CPU-bound, needs algorithmic work" from "stalling on cache misses/branch mispredictions" — two problems that look identical in `top`.
- Prefer eBPF (`bcc-tools`) over `strace`/`ltrace` for anything continuous or production-facing — `ptrace`-based tracing can slow a process 2–10x.
- **In containers:** `strace` often needs the `SYS_PTRACE` capability added explicitly (ECS/EKS tasks don't have it by default) or it just hangs/fails.

## Methodology

1. **Locate the queue first.** `vmstat`/`mpstat` tell you *if* there's a backlog before you spend time explaining *why* — no point profiling a process that isn't actually contended.
2. **Syscall-level: is it stuck on I/O, blocked on a lock, or genuinely computing?** `strace -c` gives you the summary before you drown in a raw trace.
3. **If it's a kernel-level stall** (a process that won't even respond to signals): `/proc/PID/stack` and `/proc/PID/wchan` show exactly where in the kernel it's parked.
4. **If it's CPU-bound and you need to know which function:** `perf`.
5. **If you need this system-wide, continuously, with near-zero overhead:** eBPF (`bcc-tools`) instead of `strace`, which is invasive enough to change timing on a live system.
6. **If the problem is network-shaped and you need proof:** capture the packets, don't infer from application logs alone.

## Commands, explained

### Detecting queueing before you profile anything

```bash
uptime + nproc                  # is load actually high relative to core count?
vmstat 1                        # r column — queue length, right now
mpstat -P ALL 1                 # per-core — confirms whether it's one hot core or system-wide
```
This is the same first step as the [CPU guide](cpu.md) — worth repeating here because it's the gate that decides whether the rest of this guide is even the right direction. Profiling a process that isn't actually contended just tells you it's idle most of the time, which you already knew.

### `strace` — what syscalls is this process making, and how long do they take

```bash
strace -c -p 1234                       # SUMMARY: call count + time per syscall ← always start here, not with a raw trace
```
```
% time     seconds  usecs/call     calls    syscall
------ ----------- ----------- --------- ----------------
 78.40    2.104000        4200       501 read
 12.10    0.325000         650       500 futex
```
This one command usually tells you the shape of the problem before you read a single raw line: dominated by `read`/`write` → I/O-bound, go to the [disk](disk.md) or [network](network.md) guide; dominated by `futex` → lock contention between threads; dominated by `open`/`stat` calls that mostly fail → a config or path problem, not a performance problem at all.

```bash
strace -f -o /tmp/out.txt ./app        # -f follows forks/threads — almost always want it for anything multi-process
strace -p 1234 -f                       # attach to a process already running, with children
strace -T -p 1234                       # per-call wall time — which specific calls are slow, not just frequent
strace -tt -p 1234                      # wall-clock timestamp per line — correlate against logs/metrics by time
strace -e trace=file ./app              # only file syscalls: opens, stats, unlinks
strace -e trace=network ./app           # only network syscalls: socket, connect, sendto
strace -e trace=openat -e status=failed ./app   # only *failing* opens ← gold for "why won't this config load"
strace -s 4096 -p 1234                  # don't truncate printed strings (default is a stingy 32 chars)
strace -y -p 1234                       # decode raw fds to real paths/socket addresses
strace -k -e trace=openat ./app         # stack trace per syscall — which code path is making this call
```
**Cost warning:** `strace` intercepts every syscall via `ptrace`, which can slow the traced process by 2–10x. Fine for a short diagnostic attach; do not leave it running against a production process under real load longer than you need, and prefer the eBPF tools below for anything continuous.

### `ltrace` — the userspace-library equivalent

```bash
ltrace -f ./app
ltrace -c ./app                 # summary by library call
ltrace -e 'malloc+free' ./app   # narrow to specific calls — e.g. hunting a leak pattern
```
Same idea as `strace`, one layer up: library calls (`malloc`, `memcpy`, etc.) instead of syscalls. Useful when `strace -c` shows the process barely making syscalls at all — the time is going somewhere in userspace code instead.

### `/proc/PID/*` — a stuck process, without attaching anything

```bash
cat /proc/PID/stack        # kernel stack — exactly where a D-state process is stuck
cat /proc/PID/wchan        # the single kernel function it's sleeping in, in one word
cat /proc/PID/syscall      # current syscall number + raw arguments, right now
ls -l /proc/PID/fd         # every open file/socket, live — fd exhaustion shows up here as a wall of entries
cat /proc/PID/status       # thread count, voluntary vs non-voluntary context switches
cat /proc/PID/limits       # hit an fd or process limit? confirms instantly
```
These cost nothing to check (no attach, no overhead) and are the right first move on a process that's unresponsive — `strace -p` on a truly wedged process can itself hang waiting to attach, while reading `/proc` never blocks.

### `perf` — CPU profiling, from live view to flame graphs

```bash
perf top -p PID                          # live, sampled, hottest functions right now — the profiling equivalent of top
perf record -F 99 -g -p PID -- sleep 30
perf report --stdio                      # analyze the 30s capture after the fact — this is what flame graphs are built from
perf stat -p PID -- sleep 5              # IPC, cache misses, branch misses — is the CPU doing useful work per cycle, or stalling internally?
perf trace -p PID                        # strace-like syscall view, but built on the same low-overhead sampling as perf — roughly 10x cheaper than strace
```
`perf stat`'s IPC (instructions per cycle) is the number that separates "this code is CPU-bound and needs algorithmic work" from "this code is stalling on cache misses/branch mispredictions" — two problems that look identical in `top` but have completely different fixes.

### eBPF (`bcc-tools`) — system-wide, continuous, low overhead

```bash
execsnoop            # every new process, live — "what's spawning?" answered directly, no polling needed
opensnoop -p PID     # every file open, live, low overhead — the continuous version of strace -e trace=openat
tcpconnect / tcpaccept / tcplife / tcpretrans   # connection lifecycle and retransmits, system-wide
biosnoop / biolatency                            # per-I/O trace and latency histogram — see the disk guide
runqlat              # scheduler wait-time histogram — how long processes actually wait for a CPU, not just whether the queue is nonzero
profile -F 99 -p PID # CPU stack sampling for flame graphs, lower overhead than perf record for long-running captures
memleak -p PID        # growth in outstanding allocations over time — a leak detector that doesn't require a restart under Valgrind
funccount 'vfs_*'     # frequency count of any kernel function matching a pattern — good for "is this subsystem even being hit"
```
The advantage over `strace`/`ltrace` isn't just speed — these attach via kernel probes and can run safely against live production traffic for extended periods, which `ptrace`-based tools genuinely cannot.

### Packet capture and analysis workflow

```bash
# 1. Capture to file, bounded, on the server — always bound with -c or a time limit in production
tcpdump -i any -nn -s 0 -c 5000 -w /tmp/cap.pcap 'host 10.0.1.5 and not port 22'

# 2. Quick look on the box, no GUI needed
tcpdump -nn -r /tmp/cap.pcap | head -50

# 3. Deeper analysis
tshark -r /tmp/cap.pcap -q -z conv,tcp        # conversation summary — who talked to whom, how much
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.retransmission'   # isolate just the retransmits
tshark -r /tmp/cap.pcap -q -z io,stat,1        # throughput over time, 1s buckets

# 4. Pull the .pcap to a workstation and open in Wireshark for anything visual/interactive
```
`-s 0` captures full packets (no truncation) — needed if you'll inspect payloads, not just headers. `-c 5000` (or a time-boxed run) keeps a capture on a live production host from filling disk or adding meaningful overhead; `not port 22` in the filter above is a small habit worth keeping — it keeps your own SSH session out of the capture you're trying to read.

## AWS-specific gotchas

- **`perf` needs kernel symbols and appropriate permissions** — on a locked-down AMI or a container without `SYS_ADMIN`/`perf_event_paranoid` access, `perf record` and the eBPF tools may simply refuse to attach. Check `cat /proc/sys/kernel/perf_event_paranoid` and container capabilities before assuming the tool is broken.
- **eBPF tools need a kernel that supports it** — solidly available on modern Amazon Linux 2023/Ubuntu 20.04+ kernels, but older AL2 kernels or minimal custom kernels may lack the needed BPF features. Confirm `bcc-tools` actually runs (`execsnoop` for a few seconds) before building an incident response plan around it.
- **`strace`/`ptrace` may be restricted by the container runtime.** ECS/EKS tasks often run without `SYS_PTRACE`; you'll need to add that capability (or use `kubectl debug`/an ephemeral container with it) before `strace -p` works at all inside a container — this is a common "why does this basic command just hang/fail" surprise the first time someone debugs a containerized workload.
- **Packet captures inside a VPC only see what reaches the guest** — traffic dropped by a security group or NACL never arrives at the instance, so `tcpdump` showing nothing is itself informative (rules out the guest, points at the AWS network layer — pair with VPC Flow Logs, see the [network guide](network.md)).

## Worked example: "API endpoint intermittently takes 8 seconds instead of 80ms"

**Symptom:** one specific endpoint occasionally spikes to 8s response time. CPU and memory on the host are unremarkable throughout.

**Step 1 — since it's not obviously CPU or memory, get the syscall summary during a slow window:**
```bash
strace -c -p 4821
```
```
% time     seconds  usecs/call     calls    syscall
------ ----------- ----------- --------- ----------------
 91.20    7.902000      158040        50 connect
  4.10    0.355000         710       500 read
```
`connect` dominating, and at 158ms average with only 50 calls — this isn't a busy-loop problem, it's a small number of connection attempts each taking unreasonably long.

**Step 2 — narrow to just those calls, with timing and resolved addresses:**
```bash
strace -T -y -e trace=network -p 4821
```
```
connect(52, {sa_family=AF_INET, sin_port=htons(5432), sin_addr="10.0.4.22"}, ...) = -1 ETIMEDOUT <7.980112>
```
A `connect()` to the database on port 5432 that's timing out after ~8 seconds — the connect timeout itself, not application logic, is the 8 seconds users see.

**Step 3 — since this points at the network path to the DB, capture during the next slow window:**
```bash
tcpdump -ni any host 10.0.4.22 and port 5432 -c 100
```
Shows SYN packets leaving the instance with no SYN-ACK returning during the bad window, then succeeding moments later — consistent with either intermittent packet loss on the path or the DB-side connection limit being briefly exhausted (new SYNs queued/dropped server-side under load), not a guest-side problem.

**Root cause (confirmed via RDS CloudWatch):** `DatabaseConnections` on the RDS instance was briefly hitting `max_connections` during the same windows — the app's connection pool wasn't capping outbound connections aggressively enough under a traffic burst, so a subset of connection attempts arrived when the DB had no room to accept them and sat until the OS-level connect timeout fired.

**Fix:** cap the application's connection pool below the DB's `max_connections` with margin for other clients, and lower the connect timeout so a rejected/stalled attempt fails fast (and retries or surfaces an error) instead of blocking a request thread for 8 seconds. **Prevention:** alarm on RDS `DatabaseConnections` approaching `max_connections`, and load-test the connection pool's behavior at the traffic burst level that triggers this, not just steady-state.

## Cheat sheet

```bash
uptime; nproc; vmstat 1; mpstat -P ALL 1        # confirm there's actually a queue before profiling

strace -c -p 1234                       # SUMMARY: count + time per syscall ← start here
strace -f -o /tmp/out.txt ./app         # -f follows forks/threads
strace -p 1234 -f
strace -T -p 1234                       # time per call
strace -tt -p 1234                      # wall-clock timestamps
strace -e trace=file ./app
strace -e trace=network ./app
strace -e trace=openat -e status=failed ./app   # only failures ← gold for config bugs
strace -s 4096 -p 1234
strace -y -p 1234                       # decode fds to real paths/sockets
strace -k -e trace=openat ./app         # stack trace per syscall

ltrace -f ./app
ltrace -c ./app                 # summary by library call
ltrace -e 'malloc+free' ./app

cat /proc/PID/stack        # kernel stack — where a D-state process is stuck
cat /proc/PID/wchan        # single kernel function it's sleeping in
cat /proc/PID/syscall      # current syscall number + args
ls -l /proc/PID/fd         # open files/sockets, right now
cat /proc/PID/status       # threads, ctx switches
cat /proc/PID/limits       # hit an fd limit?

perf top -p PID
perf record -F 99 -g -p PID -- sleep 30; perf report --stdio
perf stat -p PID -- sleep 5              # IPC, cache misses, branch misses
perf trace -p PID                        # strace-like but ~10x cheaper

execsnoop            # every new process
opensnoop -p PID     # file opens, live, low overhead
tcpconnect / tcpaccept / tcplife / tcpretrans
biosnoop / biolatency
runqlat              # scheduler wait times
profile -F 99 -p PID # CPU stacks for flame graphs
memleak -p PID
funccount 'vfs_*'

# Packet capture workflow
tcpdump -i any -nn -s 0 -c 5000 -w /tmp/cap.pcap 'host 10.0.1.5 and not port 22'
tcpdump -nn -r /tmp/cap.pcap | head -50
tshark -r /tmp/cap.pcap -q -z conv,tcp
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.retransmission'
tshark -r /tmp/cap.pcap -q -z io,stat,1
```
