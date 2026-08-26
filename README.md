# Linux Troubleshooting & Performance on AWS

A field guide to diagnosing Linux systems under real production pressure — written from the perspective of a systems/DevOps engineer running workloads on EC2, EBS, and VPC networking.

This isn't a man-page dump. Each guide follows the same discipline:

1. **A methodology** — the order you actually check things in, and why that order matters.
2. **Commands, explained** — not just *what* to run, but what the output means and what a bad number looks like.
3. **A worked example** — a realistic incident, start to finish: symptom → investigation → root cause → fix → how to prevent it next time.
4. **AWS-specific gotchas** — the stuff that only bites you in EC2 (CPU credits, EBS burst balance, ENA allowances, security groups vs. NACLs) that generic Linux docs never mention.
5. **A condensed cheat sheet** — the same commands, stripped down, for when you already know the theory and just need the syntax under pressure.

## Distro convention used throughout

Every command block is tiered. Reach for tier 1 first — it works everywhere and you'll never be wrong-footed by a distro you didn't expect.

| Tier | Scope | Marker |
|---|---|---|
| 1 | Portable across all Linux families — `top`, `ss`, `/proc` | *(unmarked, the default)* |
| 2 | Fedora / RHEL / CentOS Stream / **Amazon Linux** | **RHEL-family** |
| 3 | Debian / **Ubuntu** | **Debian-family** |

Where a tool needs installing, both the `dnf` and `apt` package names are given — they differ more often than people expect (`bind-utils` vs. `dnsutils`, `procps-ng` vs. `procps`, `iproute` vs. `iproute2`).

Where the kernel exposes the same data through a file, it's called out as a **`/proc` alternative**. These matter more than they look: they work on a minimal or hardened image with no diagnostic packages installed, they're trivially scriptable, and reading them never blocks — which is the difference between getting an answer and hanging your shell on a wedged process.

## Guides

| Guide | Covers | Start here if... |
|---|---|---|
| [Boot & startup](boot.md) | `systemd-analyze`, `journalctl`, unit failures, NIC renaming, cloud-init, journal persistence | An instance is slow to come up, or fails an EC2 status check |
| [Recovery & rescue](recovery.md) | Emergency/rescue targets, GRUB cmdline edits, root-password reset, detaching a root volume, chroot repair, initramfs rebuild | The instance won't boot at all and you can't SSH in |
| [CPU](cpu.md) | Load average, run queue, per-core saturation, PSI, `perf`, CPU credits, cgroup throttling | Load is high, latency is up, or a `t3` instance is mysteriously throttling |
| [Memory (RAM)](ram.md) | `free`, OOM killer, PSI, RSS vs. PSS, swap, cgroup v2 memory limits, `systemd-oomd` | A process got OOM-killed, or memory usage keeps climbing |
| [Disk & storage](disk.md) | `iostat`, `df`/`du`, inode exhaustion, deleted-but-open files, growing a volume, EBS burst balance, NVMe naming | I/O is slow, a volume is full, or `await` is spiking |
| [Network](network.md) | `ss`, `ip`, `tcpdump`, security groups vs. NACLs, ENA allowance counters, conntrack, ephemeral ports, MTU, DNS limits | Connections are timing out, dropping, or "it's not the network" needs proof |
| [Advanced diagnostics](performance.md) | `strace`, `ltrace`, `perf`, eBPF (`bcc`/`bpftrace`), fd and process limits, packet capture workflow | The above guides told you *what* is slow — this tells you *why*, at the syscall/kernel level |
| [Distro matrix](distro-matrix.md) | Package names, service managers, network config layers, firewall front-ends, bootloader tooling, side by side | You know the tool, you just need the right name on *this* box |
| [Interview drills](interview-drills.md) | 30 scenario questions with model answers, structured the way you'd actually talk through them | You're preparing for a systems engineering interview |

## The 30-second version

Every incident starts the same way, regardless of which guide you end up in:

```bash
uptime                              # load average vs. nproc — are we even under load?
top                                 # who's using what, right now
dmesg -T | tail -50                 # did the kernel already tell you what happened?
cat /proc/pressure/{cpu,memory,io}  # PSI — the fastest way to know *which* resource is the bottleneck
```

PSI (Pressure Stall Information) is the single highest-leverage habit in this repo: instead of guessing from five different tools, it directly answers "is something waiting on CPU, memory, or I/O, and for how long."

**But check it's actually there before you build a runbook on it.** PSI needs kernel 4.20+ *and* `CONFIG_PSI=y`:

| Platform | Default kernel | PSI? |
|---|---|---|
| Amazon Linux 2 | 4.14 | **No** — unless you've moved to the 5.10/5.15 kernel |
| Amazon Linux 2023 | 6.1+ | Yes |
| Ubuntu 20.04 / 22.04 / 24.04 | 5.4 / 5.15 / 6.8 | Yes |
| RHEL 8 / 9 | 4.18 / 5.14 | RHEL 8 no, RHEL 9 yes |

```bash
ls /proc/pressure/ 2>/dev/null || echo "no PSI on this kernel"
```

If the kernel is new enough but the directory is missing, it was built with `CONFIG_PSI_DEFAULT_DISABLED=y` — add `psi=1` to the kernel cmdline and reboot.

**Fallback when PSI isn't available** (memorise this — mixed fleets are the norm):

| Resource | PSI | Fallback |
|---|---|---|
| CPU | `/proc/pressure/cpu` | `vmstat 1` → `r` column vs. `nproc` |
| Memory | `/proc/pressure/memory` | `vmstat 1` → `si`/`so` non-zero; `free -h -w` → `available` shrinking |
| I/O | `/proc/pressure/io` | `iostat -xz 1` → `await` + `aqu-sz` climbing together; `D`-state process count |

## Who this is for

Systems/DevOps/SRE engineers operating Linux workloads on AWS who want a reference that assumes you know *what* the tools are, and focuses on *when* and *why* to reach for each one — plus the AWS-specific traps (CPU credit exhaustion, EBS burst balance, ENA allowance shaping, security group vs. NACL confusion) that don't show up in general Linux documentation.

## Contributing

Found a sharper way to explain something, or a scenario worth adding? PRs welcome — keep the same structure (methodology → explained commands → worked example → cheat sheet), keep the distro tiering, and give both `dnf` and `apt` package names.