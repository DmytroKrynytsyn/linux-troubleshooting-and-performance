# Linux Troubleshooting & Performance on AWS

A field guide to diagnosing Linux systems under real production pressure — written from the perspective of a systems/DevOps engineer running workloads on EC2, EBS, and VPC networking.

This isn't a man-page dump. Each guide follows the same discipline:

1. **A methodology** — the order you actually check things in, and why that order matters.
2. **Commands, explained** — not just *what* to run, but what the output means and what a bad number looks like.
3. **A worked example** — a realistic incident, start to finish: symptom → investigation → root cause → fix → how to prevent it next time.
4. **AWS-specific gotchas** — the stuff that only bites you in EC2 (CPU credits, EBS burst balance, ENA driver quirks, security groups vs. NACLs) that generic Linux docs never mention.
5. **A condensed cheat sheet** — the same commands, stripped down, for when you already know the theory and just need the syntax under pressure.

## Why this exists

Most "Linux troubleshooting" content is either a wall of man-page flags with no context, or a war story with no reusable method. The goal here is to get both in one place: understand *why* `vmstat`'s `r` column matters before you're staring at it during an incident, so reading it under pressure is recognition, not discovery.

## Guides

| Guide | Covers | Start here if... |
|---|---|---|
| [Boot & startup](boot.md) | `systemd-analyze`, `journalctl`, unit failures, NIC renaming, cloud-init | An instance is slow to come up, or fails an EC2 status check |
| [CPU](cpu.md) | Load average, run queue, per-core saturation, PSI, `perf`, CPU credits | Load is high, latency is up, or a `t3` instance is mysteriously throttling |
| [Memory (RAM)](ram.md) | `free`, OOM killer, PSI, per-process RSS/PSS, swap | A process got OOM-killed, or memory usage keeps climbing |
| [Disk & storage](disk.md) | `iostat`, `df`/`du`, inode exhaustion, EBS burst balance, `fio` | I/O is slow, a volume is full, or `await` is spiking |
| [Network](network.md) | `ss`, `ip`, `tcpdump`, security groups vs. NACLs, ENA, DNS | Connections are timing out, dropping, or "it's not the network" needs proof |
| [Advanced diagnostics](performance.md) | `strace`, `ltrace`, `perf`, eBPF (`bcc`) tools, packet capture workflow | The above guides told you *what* is slow — this tells you *why*, at the syscall/kernel level |

## The 30-second version

Every incident starts the same way, regardless of which guide you end up in:

```bash
uptime                # load average vs. nproc — are we even under load?
top                    # who's using what, right now
dmesg -T | tail -50    # did the kernel already tell you what happened?
cat /proc/pressure/{cpu,memory,io}   # PSI — the fastest way to know *which* resource is the bottleneck
```

PSI (Pressure Stall Information, kernel 4.20+) is the single highest-leverage habit in this repo: instead of guessing from five different tools, it directly answers "is something waiting on CPU, memory, or I/O, and for how long." Every guide below uses it as step one.

## Who this is for

Systems/DevOps/SRE engineers operating Linux workloads on AWS who want a reference that assumes you know *what* the tools are, and focuses on *when* and *why* to reach for each one — plus the AWS-specific traps (CPU credit exhaustion, EBS burst balance, security group vs. NACL confusion) that don't show up in general Linux documentation.

## Contributing

Found a sharper way to explain something, or a scenario worth adding? PRs welcome — keep the same structure (methodology → explained commands → worked example → cheat sheet) so the guides stay consistent.
