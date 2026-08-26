# Advanced Diagnostics: strace, perf, eBPF, and Limits

The other guides answer "*what* is slow" — CPU, memory, disk, or network. This one answers "*why*," at the syscall or kernel level, once you've narrowed the problem to a specific process. These are the tools for the moment `top` and `iostat` have told you *which* process to look at, but not *what it's actually doing*.

It also covers the resource limits that produce failures which look like nothing else in this repo: fd exhaustion, thread limits, and PID limits. Those deserve to be here because they present as "the application is broken" and are diagnosed in exactly one place.

## 1-Minute Summary

- Confirm there's an actual queue first (`vmstat`/`mpstat`) — don't profile a process that isn't contended.
- `strace -c -p PID` before a raw trace — the syscall summary alone usually tells you the shape of the problem.
- `/proc/PID/stack`, `/proc/PID/wchan`, `/proc/PID/limits` cost nothing to read and work on a wedged process that `strace -p` would hang trying to attach to.
- `perf stat -p PID`'s **IPC** separates "genuinely CPU-bound" from "stalling on memory" — identical in `top`, opposite fixes.
- Prefer eBPF (`bcc-tools`, `bpftrace`) over `strace`/`ltrace` for anything continuous — `ptrace` tracing can slow a process 2–10x.
- **"Too many open files" is `/proc/PID/limits` vs. `ls /proc/PID/fd | wc -l`.** Nothing else. The systemd unit's `LimitNOFILE` overrides `/etc/security/limits.conf` — that's why editing the latter "doesn't work."
- **In containers:** `strace` needs `SYS_PTRACE`, which ECS/EKS tasks don't have by default. It just hangs.

## Methodology

1. **Locate the queue first.** No point explaining *why* a process is slow if it isn't actually contended.
2. **Is it a limit rather than a performance problem?** fds, threads, PIDs. Cheapest check, most often skipped.
3. **Syscall-level: stuck on I/O, blocked on a lock, or genuinely computing?** `strace -c` gives the shape before you drown in a raw trace.
4. **If it's a kernel-level stall** (won't respond to signals): `/proc/PID/stack` and `/proc/PID/wchan`.
5. **If it's CPU-bound and you need to know which function:** `perf`.
6. **If you need this system-wide, continuously, at near-zero overhead:** eBPF.
7. **If the problem is network-shaped and you need proof:** capture packets, don't infer from logs.

## Resource limits — the failures that look like bugs

### "Too many open files" (`EMFILE`)

Two numbers, and the gap between them is the entire diagnosis:

```bash
cat /proc/PID/limits | grep -E 'open files|processes'
```
```
Max open files            1024                 1024                 files
Max processes             31182                31182                processes
```
```bash
ls /proc/PID/fd | wc -l
```
```
1019
```
That's your answer. No further tooling required.

**Where the limit actually comes from — in order of who wins:**

| Source | Applies to | Note |
|---|---|---|
| `LimitNOFILE=` in the systemd unit | Services started by systemd | **Wins.** This is why `/etc/security/limits.conf` "doesn't work" |
| `/etc/security/limits.conf` | Login shells via PAM | Does **not** apply to systemd services |
| `DefaultLimitNOFILE=` in `/etc/systemd/system.conf` | systemd-wide default | |
| `fs.file-max` (`/proc/sys/fs/file-max`) | System-wide ceiling, all processes | Rarely the binding constraint |
| Docker `--ulimit` / Kubernetes | Containers | Inherited from the container runtime, not the host |

The correct fix for a service:
```bash
systemctl edit myapp.service
```
```ini
[Service]
LimitNOFILE=65536
```
```bash
systemctl daemon-reload && systemctl restart myapp
cat /proc/$(pidof myapp)/limits | grep 'open files'     # verify — don't assume
```

**What's holding them?** Sockets in `CLOSE_WAIT` are the classic leak: the peer closed, the application never called `close()`.
```bash
ls -l /proc/PID/fd | awk '{print $NF}' | sed 's/:.*//' | sort | uniq -c | sort -rn | head
ls -l /proc/PID/fd | grep -c socket
ss -tan state close-wait | wc -l                # ← a growing count is an application bug
```
System-wide:
```bash
cat /proc/sys/fs/file-nr        # allocated, unused, max
```

### Thread and PID exhaustion

```bash
cat /proc/PID/status | grep Threads
cat /proc/sys/kernel/threads-max
cat /proc/sys/kernel/pid_max
ps -eLf | wc -l                                  # total threads system-wide
```
`fork: Resource temporarily unavailable` (`EAGAIN`) is either `Max processes` in `/proc/PID/limits` (a per-user limit, and remember it counts *threads*, not processes) or `pid_max`. In a container, also check the cgroup:
```bash
cat /sys/fs/cgroup/pids.max /sys/fs/cgroup/pids.current       # cgroup v2
```
Kubernetes sets a default pod PID limit; a JVM or Go service spawning unbounded goroutine-backed threads will hit it while the node looks idle.

### Detecting queueing before you profile anything

```bash
uptime ; nproc
vmstat 1                        # r column — queue length, right now
mpstat -P ALL 1                 # per-core — one hot core or system-wide?
runqlat-bpfcc 10 1              # how LONG tasks wait for a CPU, as a histogram
```
`runqlat` is the sharpest of these: `r > nproc` tells you a queue exists, but `runqlat` tells you whether tasks wait 50 microseconds (irrelevant) or 40 milliseconds (your p99). Profiling a process that isn't actually contended just tells you it's idle most of the time, which you already knew.

## Commands, explained

### `strace` — what syscalls, and how long

```bash
strace -c -p 1234                       # SUMMARY ← always start here, never with a raw trace
```
```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 78.40    2.104000        4200       501           read
 12.10    0.325000         650       500           futex
  0.30    0.008000          16       500       498 openat
```
This one command usually tells you the shape of the problem before you read a single raw line:

| Dominant syscall | Means | Go to |
|---|---|---|
| `read`/`write` | I/O-bound | [disk.md](disk.md) or [network.md](network.md) |
| `futex` | Lock contention between threads | Application concurrency design |
| `epoll_wait`/`poll` with long times | **Idle, waiting for work.** Not a problem | Look elsewhere entirely |
| `openat`/`stat` with a high **error** count | A config or path problem, not a performance problem | `-e status=failed` below |
| `brk`/`mmap` in volume | Allocation churn | [ram.md](ram.md) |
| `connect` slow, few calls | Network path or downstream saturation | [network.md](network.md) |

The `errors` column is the one people skip. 498 failed `openat` calls out of 500 means the process is searching a path list for a file that doesn't exist — often thousands of times per second, and often the actual bug.

```bash
strace -f -o /tmp/out.txt ./app        # -f follows forks/threads — almost always want it
strace -p 1234 -f                       # attach to a running process, with children
strace -T -p 1234                       # per-call wall time — which calls are slow, not just frequent
strace -tt -p 1234                      # wall-clock timestamp per line — correlate against logs
strace -e trace=file ./app              # opens, stats, unlinks
strace -e trace=network ./app           # socket, connect, sendto
strace -e trace=openat -e status=failed ./app   # only FAILING opens ← gold for "why won't config load"
strace -s 4096 -p 1234                  # don't truncate strings (default is a stingy 32 chars)
strace -y -p 1234                       # decode raw fds to real paths/socket addresses
strace -k -e trace=openat ./app         # stack trace per syscall — which code path calls this
```
**Cost warning:** `strace` intercepts every syscall via `ptrace`, slowing the target 2–10x. Fine for a short diagnostic attach; never leave it running against a production process under load. Use `perf trace` or eBPF for anything continuous.

### `ltrace` — the userspace-library equivalent

```bash
ltrace -c ./app                 # summary by library call
ltrace -f ./app
ltrace -e 'malloc+free' ./app   # narrow to specific calls — e.g. hunting a leak pattern
```
Same idea one layer up. Useful when `strace -c` shows the process barely making syscalls at all — the time is going somewhere in userspace instead.

### `/proc/PID/*` — a stuck process, without attaching anything

```bash
cat /proc/PID/stack        # kernel stack — exactly where a D-state process is stuck
cat /proc/PID/wchan        # the single kernel function it's sleeping in
cat /proc/PID/syscall      # current syscall number + raw arguments, right now
ls -l /proc/PID/fd         # every open file/socket, live
cat /proc/PID/status       # threads, voluntary vs NONvoluntary context switches
cat /proc/PID/limits       # hit an fd or process limit? confirms instantly
cat /proc/PID/cgroup       # WHICH cgroup — then read its limits
cat /proc/PID/environ | tr '\0' '\n'    # the env it actually started with, not what you think
cat /proc/PID/cmdline | tr '\0' ' '     # the real argv
```
These cost nothing (no attach, no overhead) and are the right first move on an unresponsive process — `strace -p` on a truly wedged process can itself hang waiting to attach, while reading `/proc` never blocks.

`nonvoluntary_ctxt_switches` in `/proc/PID/status` is worth knowing: high and rising means the scheduler is preempting the process because CPU is contended (it wanted to keep running). High `voluntary_ctxt_switches` means it's blocking on something itself. That distinction points you at [cpu.md](cpu.md) or elsewhere without running a single tool.

`/proc/PID/environ` settles more arguments than it should have to. "The service has the wrong config" is very often "the service was started with a different environment than the one you're reading from a file."

### `perf` — CPU profiling, live view to flame graphs

```bash
perf top -p PID                          # live, sampled, hottest functions
perf record -F 99 -g -p PID -- sleep 30
perf report --stdio                      # analyse after — flame graph source
perf stat -p PID -- sleep 5              # IPC, cache misses, branch misses
perf trace -p PID                        # strace-like, ~10x cheaper (sampling, not ptrace)
perf sched latency                        # scheduler wait times per task
```
`perf stat`'s **IPC** is the number that separates "CPU-bound, needs algorithmic work" (IPC > 2) from "stalling on memory access" (IPC < 1, with high cache-miss rate). Identical in `top`, completely different fixes.

**Install note (this catches people):**
```bash
dnf install perf                                              # RHEL-family, simple
apt install linux-tools-common linux-tools-$(uname -r) linux-tools-aws    # Ubuntu on EC2
```
On Debian-family, `perf` is versioned against the running kernel and on EC2 you need the `-aws` variant. Without the exact match it prints a mismatch warning and refuses to run — which people misread as "perf is broken on this box."

### eBPF — system-wide, continuous, low overhead

```bash
execsnoop            # every new process, live — "what's spawning?"
opensnoop -p PID     # every file open — the continuous version of strace -e trace=openat
tcpconnect / tcpaccept / tcplife / tcpretrans   # connection lifecycle and retransmits
biosnoop / biolatency                            # per-I/O trace and latency histogram
runqlat              # scheduler wait-time HISTOGRAM
profile -F 99 -p PID # CPU stack sampling, lower overhead than perf record for long captures
memleak -p PID       # growth in outstanding allocations — leak detection without a restart
funccount 'vfs_*'    # frequency count of any kernel function matching a pattern
offcputime -p PID    # where a process spends time BLOCKED, which perf can't show you
```
Binary naming differs: `bcc-tools` on RHEL-family installs both plain names and `-bpfcc` suffixed; `bpfcc-tools` on Debian-family installs **only** the `-bpfcc` suffixed names (`biolatency-bpfcc`). Check `ls /usr/share/bcc/tools/` if a command isn't found.

`offcputime` deserves a mention: `perf` samples on-CPU time, so a process that spends 95% of a request blocked on a lock or an I/O is nearly invisible to it. `offcputime` profiles the inverse and is often where the actual latency lives.

**`bpftrace`** for ad-hoc one-liners, when no canned tool fits:
```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
bpftrace -e 'kprobe:vfs_read { @bytes = hist(arg2); }'
bpftrace -l 'tracepoint:syscalls:*'      # list available probes
```
Install: `dnf install bpftrace` / `apt install bpftrace`.

The advantage over `strace`/`ltrace` isn't only speed — these attach via kernel probes and can run safely against live production traffic for extended periods, which `ptrace`-based tools genuinely cannot.

### Packet capture and analysis workflow

```bash
# 1. Capture to file, BOUNDED, on the server
tcpdump -i any -nn -s 0 -c 5000 -w /tmp/cap.pcap 'host 10.0.1.5 and not port 22'

# 2. Quick look on the box, no GUI
tcpdump -nn -r /tmp/cap.pcap | head -50

# 3. Deeper analysis
tshark -r /tmp/cap.pcap -q -z conv,tcp                      # who talked to whom, how much
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.retransmission'    # isolate retransmits
tshark -r /tmp/cap.pcap -q -z io,stat,1                      # throughput over time, 1s buckets
tshark -r /tmp/cap.pcap -q -z expert                         # protocol-level warnings

# 4. Pull the .pcap to a workstation for Wireshark
```
`-s 0` captures full packets (needed for payload inspection). `-c 5000` or a time limit keeps a production capture from filling the disk. `not port 22` keeps your own SSH session out of the capture you're trying to read — a small habit worth keeping.

### Install

| Tool | RHEL-family (`dnf`) | Debian-family (`apt`) |
|---|---|---|
| `strace` | `strace` | `strace` |
| `ltrace` | `ltrace` | `ltrace` |
| `perf` | `perf` | `linux-tools-common linux-tools-$(uname -r) linux-tools-aws` |
| bcc tools | `bcc-tools` | `bpfcc-tools` ← **name differs** |
| `bpftrace` | `bpftrace` | `bpftrace` |
| `tshark` | `wireshark-cli` | `tshark` ← **name differs** |
| `tcpdump` | `tcpdump` | `tcpdump` |
| `lsof` | `lsof` | `lsof` |
| `gdb` | `gdb` | `gdb` |
| Debug symbols | `debuginfo-install <pkg>` | `<pkg>-dbgsym` (needs the ddebs repo) |

## AWS-specific gotchas

- **`perf` needs kernel symbols and permission.** Check `cat /proc/sys/kernel/perf_event_paranoid` — `2` (a common default) blocks kernel-level profiling; you need `1` or lower for `-g` call graphs, `-1` for full access. In a container without `SYS_ADMIN`, `perf record` simply refuses to attach, which reads as "the tool is broken."
- **`strace` needs `SYS_PTRACE`, which ECS/EKS tasks don't have by default.** It hangs or fails with `EPERM`. Add the capability, or use `kubectl debug -it pod --image=nicolaka/netshoot --target=app` to get an ephemeral debug container sharing the process namespace.
- **eBPF needs a modern kernel.** Solid on Amazon Linux 2023 and Ubuntu 20.04+. **Amazon Linux 2's 4.14 kernel lacks many BPF features** — some `bcc` tools fail outright there. Verify `execsnoop` runs for a few seconds before you build an incident-response plan around eBPF on AL2.
- **Packet captures inside a VPC only see what reaches the guest.** Traffic dropped by a security group, a NACL, or ENA allowance shaping never arrives, so `tcpdump` showing nothing is itself informative — it rules out the guest and points above it. Pair with `ethtool -S eth0 | grep exceeded` and VPC Flow Logs ([network.md](network.md)).
- **Core dumps go nowhere by default.** On AL2023 and Ubuntu, `systemd-coredump` captures them into the journal, which is volatile unless you've made it persistent. Check `coredumpctl list`; if the tool isn't there, `cat /proc/sys/kernel/core_pattern` tells you where (if anywhere) they land. A crash you can't post-mortem is a crash you get to have twice.

## Worked example: "API endpoint intermittently takes 8 seconds instead of 80ms"

**Symptom:** one specific endpoint occasionally spikes to 8s. CPU and memory on the host are unremarkable throughout.

**Step 1 — rule out a resource limit first, since it's free and it's often the answer.**
```bash
cat /proc/4821/limits | grep 'open files'      # 65536 — not near it
ls /proc/4821/fd | wc -l                        # 2140
```
Not fd exhaustion.

**Step 2 — get the syscall summary during a slow window.**
```bash
strace -c -p 4821
```
```
% time     seconds  usecs/call     calls    syscall
------ ----------- ----------- --------- ----------------
 91.20    7.902000      158040        50 connect
  4.10    0.355000         710       500 read
```
`connect` dominating at 158ms average with only 50 calls — not a busy loop, a small number of connection attempts each taking unreasonably long. That immediately reframes this from "the application is slow" to "the application is waiting on something downstream."

**Step 3 — narrow to those calls, with timing and resolved addresses.**
```bash
strace -T -y -e trace=network -p 4821
```
```
connect(52, {sa_family=AF_INET, sin_port=htons(5432), sin_addr="10.0.4.22"}, ...) = -1 ETIMEDOUT <7.980112>
```
A `connect()` to the database timing out after ~8 seconds. The connect timeout *is* the 8 seconds users see — no application logic involved.

**Step 4 — rule out instance-level shaping before blaming the database, because it produces exactly this symptom.**
```bash
ethtool -S eth0 | grep -E 'conntrack|pps' > /tmp/a; sleep 30
ethtool -S eth0 | grep -E 'conntrack|pps' > /tmp/b; diff /tmp/a /tmp/b
```
No change across the window, and `conntrack_allowance_available` is healthy. Not the ENA allowance.

**Step 5 — capture during the next slow window.**
```bash
tcpdump -ni any host 10.0.4.22 and port 5432 -c 100
```
SYNs leaving the instance with no SYN-ACK returning during the bad window, then succeeding moments later. Per the decision table in [network.md](network.md), "SYN leaves, no SYN-ACK returns" means the problem is beyond the guest.

**Root cause (confirmed via RDS CloudWatch):** `DatabaseConnections` was briefly hitting `max_connections` during the same windows. The application's connection pool wasn't capping outbound connections aggressively enough under a traffic burst, so a subset of attempts arrived when the DB had no room to accept them and sat until the OS-level connect timeout fired.

**Fix:** cap the application's pool below the DB's `max_connections` with margin for other clients, and lower the connect timeout so a stalled attempt fails fast — and retries or surfaces an error — instead of blocking a request thread for 8 seconds. An 8-second connect timeout on a same-VPC database is itself a bug; nothing healthy takes that long.
**Prevention:** alarm on RDS `DatabaseConnections` approaching `max_connections`, and load-test the pool's behaviour at the burst level that triggers this, not just steady-state.

## Cheat sheet

```bash
# 0. Is there actually a queue?
uptime; nproc; vmstat 1; mpstat -P ALL 1
runqlat-bpfcc 10 1              # how LONG tasks wait, not just whether they do

# 1. Is it a LIMIT, not a performance problem? (cheapest check, most often skipped)
cat /proc/PID/limits            # Max open files / Max processes
ls /proc/PID/fd | wc -l         # vs. the limit above ← the whole diagnosis
ls -l /proc/PID/fd | grep -c socket
ss -tan state close-wait | wc -l          # growing = app never calls close()
cat /proc/sys/fs/file-nr        # allocated, unused, max
cat /proc/PID/status | grep Threads
cat /sys/fs/cgroup/pids.{max,current}     # container PID limit
# systemd LimitNOFILE= OVERRIDES /etc/security/limits.conf:
systemctl edit myapp.service    # [Service] LimitNOFILE=65536

# 2. Syscall shape
strace -c -p 1234               # ← ALWAYS start here. Watch the ERRORS column
strace -f -o /tmp/out.txt ./app ; strace -p 1234 -f
strace -T -p 1234               # time per call
strace -tt -p 1234              # wall-clock timestamps, to correlate with logs
strace -e trace=file ./app ; strace -e trace=network ./app
strace -e trace=openat -e status=failed ./app     # ← gold for config bugs
strace -s 4096 -p 1234 ; strace -y -p 1234 ; strace -k -e trace=openat ./app
ltrace -c ./app                 # when strace shows almost no syscalls

# 3. Wedged process — /proc only, never blocks, no attach
cat /proc/PID/stack /proc/PID/wchan /proc/PID/syscall
cat /proc/PID/status            # voluntary vs NONvoluntary ctxt switches
cat /proc/PID/cgroup            # then read that cgroup's limits
cat /proc/PID/environ | tr '\0' '\n'      # what it ACTUALLY started with
cat /proc/PID/cmdline | tr '\0' ' '

# 4. CPU profiling
perf top -p PID
perf record -F 99 -g -p PID -- sleep 30 ; perf report --stdio
perf stat -p PID -- sleep 5      # IPC <1 = memory-stalled | >2 = compute-bound
perf trace -p PID                # ~10x cheaper than strace
cat /proc/sys/kernel/perf_event_paranoid   # 2 blocks kernel profiling

# 5. eBPF — safe on live production
execsnoop ; opensnoop -p PID
tcpconnect / tcpaccept / tcplife / tcpretrans
biosnoop / biolatency ; runqlat
profile -F 99 -p PID ; memleak -p PID
offcputime -p PID                # where it's BLOCKED — perf can't see this
funccount 'vfs_*'
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
# Debian-family binaries are ONLY suffixed: biolatency-bpfcc. ls /usr/share/bcc/tools/

# 6. Packet capture
tcpdump -i any -nn -s 0 -c 5000 -w /tmp/cap.pcap 'host 10.0.1.5 and not port 22'
tshark -r /tmp/cap.pcap -q -z conv,tcp
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.retransmission'
tshark -r /tmp/cap.pcap -q -z io,stat,1

# Install
dnf install strace ltrace perf bcc-tools bpftrace wireshark-cli lsof gdb
apt  install strace ltrace linux-tools-$(uname -r) linux-tools-aws bpfcc-tools bpftrace tshark lsof gdb

# AWS
# strace in ECS/EKS needs SYS_PTRACE — otherwise it just hangs. Or: kubectl debug --target=app
# eBPF is unreliable on Amazon Linux 2 (4.14). Verify execsnoop runs before depending on it.
# tcpdump seeing NOTHING is evidence: the drop is above the guest.
#   → ethtool -S eth0 | grep exceeded, then VPC Flow Logs
# coredumpctl list — core dumps go to a VOLATILE journal unless you made it persistent
```