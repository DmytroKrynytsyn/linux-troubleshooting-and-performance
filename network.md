# Network Troubleshooting

Network incidents on AWS have an extra wrinkle that pure-Linux guides skip: the guest OS is only *one* of several layers that can drop or block a connection. Security groups, NACLs, route tables, the ENA driver's shaping, and per-instance connection-tracking limits all sit between "the app called `connect()`" and "the packet actually left the instance" — and most of them fail **silently and invisibly** from the guest's point of view. This guide works outward from the guest OS, then names exactly where to look once the guest itself is clean.

## 1-Minute Summary

- `ip r` for a route, `ss -tulpn` to confirm the app is listening on `0.0.0.0`, not `127.0.0.1` — a shockingly common root cause that works fine from `localhost` and fails for everyone else.
- A failed `ping` proves nothing on AWS — ICMP is routinely blocked by security groups while the real application port works fine.
- **Security groups (stateful, per-ENI) vs. NACLs (stateless, per-subnet)** are different layers with different failure signatures — check both.
- **`ethtool -S eth0 | grep exceeded` is the single most AWS-specific command in this repo.** It exposes per-instance bandwidth, PPS, and connection-tracking shaping that is invisible to every generic Linux tool.
- Connections failing while `ss` looks healthy? Two different conntrack ceilings: the guest kernel's, and the instance's.
- `EADDRNOTAVAIL` on outbound connections = ephemeral port exhaustion, not a network fault.
- `tcpdump` is ground truth. Guest clean but connections still fail → **VPC Flow Logs**.

## Methodology

1. **Can the box route to where it needs to go, at all?** `ip r`, `ip a`.
2. **Is anything actually listening, on the port you expect, on the right address?** `ss -tulpn` — don't assume, verify.
3. **Guest firewall clean?** Check before blaming AWS-layer rules.
4. **Is the guest hitting a resource ceiling rather than a rule?** Ephemeral ports, conntrack, socket backlog, fd limit. These look like "the network is broken" and aren't.
5. **Is the *instance* being shaped?** `ethtool -S` allowance counters. Invisible everywhere else.
6. **If the guest is genuinely clean, it's above the guest.** Security group → NACL → route table.
7. **Confirm at the packet level if anything above is ambiguous.** `tcpdump` is ground truth; everything else is inference.
8. **DNS is a separate failure mode from routing** — resolves-but-won't-connect and never-resolves need different fixes.

Steps 4 and 5 are the ones missing from most runbooks, and they're where the genuinely hard AWS incidents live.

## Commands, explained

### `ip` — interfaces, routes, neighbours

```bash
ip -br a                        # brief interface + address summary
ip r                            # routing table
ip r get 10.0.4.22              # which route would ACTUALLY be used for this destination
ip neigh                        # ARP table
ip -s link                      # RX/TX errors, dropped, overruns
```
```
2: eth0    UP    10.0.1.15/24
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.15
```
`ip r get <dest>` is underused and better than reading the table by eye — with multiple ENIs, policy routing, or VPN routes present, it tells you which route and which source address the kernel will actually pick, including for asymmetric-routing bugs on multi-ENI instances.

No route to a destination subnet means either the VPC route table is genuinely missing one, or you're testing from the wrong subnet/AZ context. Two-minute check that rules out an entire class of "network is broken" reports.

**`/proc` alternatives:**
```bash
cat /proc/net/route             # IPv4 routes, hex, little-endian (annoying but scriptable)
cat /proc/net/arp               # ARP table
cat /proc/net/dev               # what `ip -s link` reads
```

**Legacy tools you should know but not use** — `ifconfig`, `netstat`, `route`, `arp` are all in `net-tools` (`dnf install net-tools` / `apt install net-tools`), deprecated since ~2011 and often absent from modern AMIs. `ip` and `ss` are the iproute2 replacements and expose things the old tools can't (multiple addresses per interface, policy routing, socket-level TCP internals). Knowing both, and using the modern one by default, is the right posture.

### `ss` — sockets, without guessing

```bash
ss -tulpn                       # listening TCP+UDP sockets, with owning PID
```
```
tcp   LISTEN  0  128   0.0.0.0:8080   0.0.0.0:*   users:(("java",pid=4821,fd=42))
```
This directly answers "is my app listening where I think." A classic root cause is the app bound to `127.0.0.1:8080` instead of `0.0.0.0:8080` — identical from inside the box (`curl localhost:8080` works fine), refuses every connection from anywhere else including a load balancer health check.

The two numbers before the address are `Recv-Q` and `Send-Q`. **On a LISTEN socket they mean something different from an established one:** `Recv-Q` is the current accept-queue depth and `Send-Q` is the configured backlog. A LISTEN socket showing `128 128` is a full accept queue — the app isn't calling `accept()` fast enough, and the kernel is about to start dropping SYNs.

```bash
ss -tan state syn-recv          # half-open connections — backlog exhaustion or SYN flood
ss -tan state time-wait | wc -l # TIME_WAIT count — relevant to port exhaustion, below
ss -s                           # one-screen socket summary
ss -tin                         # per-socket TCP internals: rtt, cwnd, retrans, bytes_acked
ss -tp dst 10.0.4.22            # every connection to one host, with process
ss -tm                          # socket memory usage — for "why is this box out of RAM"
```
`ss -tin` is the one worth practising. It exposes the kernel's own view of path quality per connection:
```
cubic wscale:7,7 rto:236 rtt:35.5/1.75 mss:1448 cwnd:10 bytes_acked:84213 retrans:0/4 
```
`rtt` far above the AZ baseline (sub-millisecond same-AZ, ~1-2ms cross-AZ in-region) points at the path. `retrans` climbing points at loss. `cwnd` stuck at 10 (the initial window) on a long-lived connection means the connection keeps getting reset back to slow start — usually by loss, sometimes by idle timeouts.

### Reachability and path

```bash
ping <host>                          # ICMP — often blocked by SG; a failed ping proves NOTHING on AWS
nc -zv <host> <port>                 # test the ACTUAL port — this is the real test
curl -sv --connect-timeout 3 telnet://host:port
traceroute -T -p 443 <host>          # TCP traceroute — ICMP traceroute is usually filtered in VPC
mtr -rwbc 100 <host>                 # traceroute + ping, continuously — WHERE loss/latency appears
```
**Don't trust a failed `ping` as proof of an outage on AWS.** ICMP is blocked by default in most security group configurations while TCP on the application port flows fine. Test the real port before concluding anything.

`traceroute -T` matters inside a VPC: AWS's network doesn't decrement TTL in a way that produces useful hop-by-hop output for most paths, and ICMP responses are typically filtered. You will often see nothing but stars, and that's not evidence of a problem.

### Packet capture — ground truth

```bash
tcpdump -ni any port 443 -c 100
tcpdump -ni eth0 'tcp[tcpflags] & (tcp-syn) != 0 and not port 22' -c 50   # SYNs only
tcpdump -ni any host 10.0.4.22 -w /tmp/cap.pcap -c 5000                   # bounded capture to file
```
When `ss` and application logs disagree, or the symptom is intermittent, a capture settles it:

| What you see | What it means |
|---|---|
| SYN leaves, no SYN-ACK returns | Blocked or lost beyond the guest — SG, NACL, route, or the far end is down |
| SYN arrives, no SYN-ACK sent | **Your** box is dropping it — full accept queue, local firewall, or nothing listening |
| Full handshake, then slow response | Not a network problem. Go look at the application |
| RST immediately after SYN | Something actively refused — nothing listening, or a NACL/host firewall configured to reject rather than drop |
| Handshake fine, stall on first large payload | **Path MTU mismatch** — see below |

Always bound production captures with `-c` or a time limit, and always exclude your own SSH session (`not port 22`) or you'll be reading your own typing.

### DNS

```bash
dig example.com                      # +short for just the answer, +trace to follow delegation
dig @169.254.169.253 example.com     # bypass local resolver, ask the VPC resolver directly
nslookup example.com
getent hosts example.com             # what the app actually gets — respects nsswitch, /etc/hosts, NSS modules
```
`getent hosts` is the important one and the one people skip: `dig` talks to DNS directly, while your application uses `getaddrinfo()`, which goes through `/etc/nsswitch.conf`, `/etc/hosts`, and any NSS modules. A `dig` that works while the app fails almost always means the difference is in that path.

**Distro contrast — this is where the families genuinely diverge:**

**RHEL-family (AL2, AL2023, RHEL):** `/etc/resolv.conf` is usually the real file, managed by NetworkManager or cloud-init.
```bash
cat /etc/resolv.conf
nmcli device show eth0 | grep DNS
```

**Debian-family (Ubuntu):** `/etc/resolv.conf` is typically a symlink to a `systemd-resolved` stub, and points at `127.0.0.53`. Reading it tells you almost nothing about the real upstream:
```bash
ls -l /etc/resolv.conf                  # → ../run/systemd/resolve/stub-resolv.conf
resolvectl status                       # the ACTUAL upstream servers, per-link
resolvectl query example.com            # test through the same path the app uses
resolvectl statistics                   # cache hits/misses
resolvectl flush-caches
```
Seeing `nameserver 127.0.0.53` and concluding "DNS is misconfigured" is a very common wrong turn on Ubuntu.

**Install:** `dnf install bind-utils` / `apt install dnsutils` (or `bind9-dnsutils` on newer Debian) ← **name differs**

### Interface health at the driver level

```bash
ethtool eth0                    # link speed, duplex
ethtool -i eth0                 # DRIVER and version — is this even ENA?
ethtool -S eth0                 # full statistics, including AWS allowance counters (see below)
ethtool -g eth0                 # ring buffer sizes, current vs max
ethtool -l eth0                 # queue counts
ethtool -c eth0                 # interrupt coalescing
```
`ethtool -i` first: if the driver isn't `ena`, you're on an older instance family (`vif`/`ixgbevf`) and the allowance counters below won't exist.

**Install:** `ethtool` on both families.

### /proc — the same data, scriptable

```bash
cat /proc/net/dev               # per-interface counters
cat /proc/net/snmp              # TcpRetransSegs, TcpInErrs, TcpOutRsts — protocol-level health
cat /proc/net/netstat           # ListenDrops, ListenOverflows, TCPSynRetrans, TCPTimeouts
cat /proc/net/sockstat          # sockets in use, TIME_WAIT count, socket memory
cat /proc/net/softnet_stat      # per-CPU: col1 processed, col2 DROPPED, col3 time_squeeze
cat /proc/interrupts            # NIC IRQ distribution — one core doing all of it is a ceiling
cat /proc/softirqs              # NET_RX per core
cat /proc/sys/net/ipv4/ip_local_port_range
cat /proc/sys/net/core/somaxconn
```
`ListenDrops`/`ListenOverflows` climbing in `/proc/net/netstat` is the counter-based confirmation of the `syn-recv` symptom — connections arriving faster than the app calls `accept()`.

`/proc/net/softnet_stat` deserves attention on high-PPS workloads: **column 2 is packets dropped** because the per-CPU backlog filled, and **column 3 (`time_squeeze`)** counts times the softirq handler ran out of budget with work remaining. Non-zero in column 2 on a specific CPU, alongside a lopsided `/proc/interrupts`, means your NIC interrupts aren't spread across cores.

### Firewall — the guest layer only

```bash
nft list ruleset                # nftables — the modern backend, and what you'll actually find
iptables -L -n -v               # legacy syntax; on modern distros this is iptables-nft, a shim
iptables-save                   # more readable than -L for anything non-trivial
conntrack -L | wc -l            # current tracked connections (needs conntrack-tools/conntrack)
```

**RHEL-family:**
```bash
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --list-all-zones
```
Note: **Amazon Linux 2 and AL2023 ship with no firewall enabled by default** — the security group is the firewall. Finding `firewalld` inactive on an AL box is normal, not a finding.

**Debian-family:**
```bash
ufw status verbose
```
Also inactive by default on Ubuntu cloud images.

Docker/containerd inject their own iptables chains (`DOCKER`, `DOCKER-USER`, `KUBE-*`). If a box runs containers, `iptables -L -n -v` will be hundreds of lines, and rules you add to `INPUT` may be bypassed entirely because `DOCKER-USER` is evaluated in `FORWARD` first.

**Install:** `dnf install nftables iptables-services conntrack-tools` / `apt install nftables iptables conntrack`

### Install summary

| Tool | RHEL-family (`dnf`) | Debian-family (`apt`) |
|---|---|---|
| `ip`, `ss` | `iproute` | `iproute2` ← **name differs** |
| `dig`, `nslookup` | `bind-utils` | `dnsutils` ← **name differs** |
| `nc` | `nmap-ncat` | `netcat-openbsd` ← **name differs** |
| `tcpdump` | `tcpdump` | `tcpdump` |
| `mtr` | `mtr` | `mtr-tiny` or `mtr` |
| `traceroute` | `traceroute` | `traceroute` |
| `ethtool` | `ethtool` | `ethtool` |
| `iperf3` | `iperf3` | `iperf3` |
| `conntrack` | `conntrack-tools` | `conntrack` ← **name differs** |
| `tshark` | `wireshark-cli` | `tshark` ← **name differs** |
| legacy `ifconfig`/`netstat` | `net-tools` | `net-tools` |
| eBPF `tcplife`, `tcpretrans` | `bcc-tools` | `bpfcc-tools` |

## Guest-side resource ceilings that look like network faults

### Ephemeral port exhaustion

**Symptom:** outbound connections start failing with `EADDRNOTAVAIL` ("Cannot assign requested address") under load, while the network is completely healthy.

A TCP connection is identified by the 4-tuple (src IP, src port, dst IP, dst port). For a fixed destination, one source IP gives you only as many source ports as the ephemeral range — about 28,000 by default. Add `TIME_WAIT`, which holds a tuple for 60 seconds after close, and a service making short-lived connections to a single backend can exhaust the range in seconds.

```bash
cat /proc/sys/net/ipv4/ip_local_port_range      # 32768 60999 → ~28k ports
ss -tan state time-wait | wc -l
cat /proc/net/sockstat                          # TCP: inuse ... tw 24310 ...
ss -tan | awk '{print $1}' | sort | uniq -c     # state distribution at a glance
```

**Fixes, in the order you should actually try them:**

1. **Fix the application.** Connection pooling or HTTP keep-alive removes the problem entirely. Everything below is a workaround for an app that opens a new connection per request.
2. `net.ipv4.tcp_tw_reuse=1` — lets the kernel reuse a `TIME_WAIT` socket for a *new outbound* connection when timestamps prove it's safe. Safe for clients. This is the correct knob.
3. Widen the range: `net.ipv4.ip_local_port_range = 10240 65535`.
4. Add source IPs (secondary private IPs on the ENI) if the destination tuple is genuinely fixed — each new source IP is a fresh 28k ports.

**Do not reach for `tcp_tw_recycle`.** Every 2014-era blog post recommends it. It breaks badly for clients behind NAT (it rejects connections from hosts whose TCP timestamps appear to go backwards, which is normal across a NAT) and it was **removed from the kernel in 4.12**. Knowing this is a good signal that you've operated real systems rather than copied sysctl snippets.

### Connection tracking — two separate ceilings

**Ceiling 1 — the guest kernel's conntrack table.** Only applies if netfilter connection tracking is loaded, which it is if you have iptables/nftables stateful rules or run Docker/Kubernetes.

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
dmesg | grep -i "table full"
```
```
nf_conntrack: table full, dropping packet
```
That kernel message is unambiguous. Fix:
```bash
sysctl -w net.netfilter.nf_conntrack_max=262144
sysctl -w net.netfilter.nf_conntrack_buckets=65536   # keep max ≈ 4× buckets
```
Also worth checking is `nf_conntrack_tcp_timeout_established`, which defaults to **5 days** — a box handling many short connections accumulates entries for long-dead flows.

**Ceiling 2 — the instance's connection tracking, enforced by AWS.** This is the one that isn't in any Linux book. Because security groups are **stateful**, AWS itself tracks every connection per ENI, with a per-instance-type limit entirely independent of your kernel. When you exceed it, new connections are simply dropped — and the guest sees nothing at all.

```bash
ethtool -S eth0 | grep conntrack
```
```
conntrack_allowance_available: 136812
conntrack_allowance_exceeded: 0
```
`conntrack_allowance_exceeded` counts packets dropped because connection tracking hit the instance maximum and new connections could not be established. `conntrack_allowance_available` shows headroom remaining — it needs ENA driver 2.8.1 or newer, so its absence doesn't mean you're unaffected, only that you can't see the headroom.

Mitigation is architectural: fewer, longer-lived connections; a larger instance type; or security group rules that allow all traffic on a port in both directions from `0.0.0.0/0`, which AWS treats as untracked and exempts from the limit.

### Socket accept-queue exhaustion

```bash
ss -tulpn | grep 8080           # Recv-Q vs Send-Q on the LISTEN socket
ss -tan state syn-recv | wc -l
grep -E 'ListenDrops|ListenOverflows' /proc/net/netstat
cat /proc/sys/net/core/somaxconn
```
Two limits apply and **the smaller wins**: `net.core.somaxconn` (kernel-wide cap) and the `backlog` argument the application passes to `listen()`. Raising `somaxconn` alone does nothing if the app hardcodes `listen(fd, 128)`.

### File descriptor exhaustion

Every socket is an fd. An app that hits its fd limit stops accepting connections and logs "too many open files" — which reads like a filesystem problem and is actually a network capacity problem.

```bash
cat /proc/PID/limits | grep 'open files'
ls /proc/PID/fd | wc -l
cat /proc/sys/fs/file-nr        # allocated, free, max — system-wide
```
The systemd angle matters: a service started by systemd inherits `LimitNOFILE` from its unit, **not** from `/etc/security/limits.conf` (which only applies to PAM login sessions). This catches people constantly.
```bash
systemctl show myapp -p LimitNOFILE
```

## AWS-specific gotchas

### ENA allowance counters — the most important command in this guide

Every EC2 instance type has caps on aggregate bandwidth, packets per second, connection tracking, and traffic to link-local services. When you exceed one, AWS **queues or drops** the packets. From inside the guest this is indistinguishable from mysterious packet loss: `ip -s link` shows no errors, the firewall is clean, `tcpdump` just shows retransmits.

The ENA driver exposes the counters. There is no other way to see this:

```bash
ethtool -S eth0 | grep -E 'exceeded|allowance'
```
```
bw_in_allowance_exceeded: 0
bw_out_allowance_exceeded: 0
pps_allowance_exceeded: 0
conntrack_allowance_exceeded: 0
linklocal_allowance_exceeded: 0
conntrack_allowance_available: 136812
```

| Counter | Non-zero means |
|---|---|
| `bw_in_allowance_exceeded` | Inbound aggregate bandwidth exceeded the instance maximum; packets queued or dropped |
| `bw_out_allowance_exceeded` | Same, outbound |
| `pps_allowance_exceeded` | Bidirectional packets-per-second exceeded the maximum. **Small packets hit this long before bandwidth limits** — a high-QPS, small-payload service saturates PPS at a fraction of its nominal Gbps |
| `conntrack_allowance_exceeded` | Instance connection-tracking limit hit; new connections dropped |
| `linklocal_allowance_exceeded` | PPS to **DNS (the VPC resolver), IMDS, and the Amazon Time Sync Service** exceeded the per-ENI limit |

**`linklocal_allowance_exceeded` is the hidden cause of intermittent DNS failures on busy nodes**, and it's a well-known EKS problem: every pod's DNS query on a node counts toward one ENI's link-local budget. Symptoms are sporadic resolution timeouts with a perfectly healthy CoreDNS. Mitigations are node-local DNS caching, `ndots` tuning, and spreading load across more nodes.

These are cumulative counters since boot, so sample them twice and diff — a non-zero value from three weeks ago is not an active incident:
```bash
watch -n5 'ethtool -S eth0 | grep exceeded'
```

Ship them to CloudWatch via the agent's `ethtool` plugin, where they appear prefixed (`ethtool_bw_in_allowance_exceeded`).

The limits are not exclusive to Nitro — older Xen instances have them too, often lower — but only ENA exposes the counters, so on a non-Nitro instance you're inferring rather than measuring.

### Security groups vs. NACLs

| | Security Group | NACL |
|---|---|---|
| Attached to | **ENI** | **Subnet** |
| State | **Stateful** — return traffic auto-allowed | **Stateless** — you must allow both directions explicitly |
| Rules | Allow only | Allow **and deny**, evaluated in numbered order |
| Default | Deny all inbound, allow all outbound | Default NACL allows everything |

The practical consequence: a NACL needs an explicit **outbound rule for ephemeral ports (1024–65535)** to let return traffic out. A security group handles that automatically. A connection that establishes and then stalls, with the guest completely clean, is very often a missing NACL ephemeral-port rule.

Note that for *inbound* traffic AWS evaluates the NACL first and the security group second; it's the reverse for outbound. Check both regardless — the "which comes first" question is a good interview follow-up and the honest answer is that ordering rarely changes your diagnostic sequence.

### Path MTU

Within a VPC, instances can use jumbo frames at **9001 bytes MTU**. Traffic leaving through an internet gateway is capped at **1500**. Transit Gateway and Direct Connect sit in between (around 8500). The result: a connection whose handshake succeeds (small packets) but hangs the moment a large payload is sent.

```bash
ip link show eth0 | grep mtu
ping -M do -s 8972 <dest>       # 8972 payload + 28 header = 9000; -M do sets DF
tracepath <dest>                # reports the discovered path MTU
```
PMTU discovery relies on ICMP "fragmentation needed" messages. **If a security group or NACL blocks ICMP, PMTUD breaks silently** and you get a black hole instead of a clean fallback — which is the strongest practical argument for allowing ICMP type 3 code 4 in your security groups.

### DNS in a VPC

- The VPC resolver lives at the **VPC CIDR base + 2** (a `10.0.0.0/16` VPC has it at `10.0.0.2`) and also at the link-local `169.254.169.253`.
- There's a hard limit of **1024 packets per second per network interface** to the Route 53 Resolver. Exceeding it drops queries, which surfaces as random, unattributable resolution failures — and increments `linklocal_allowance_exceeded`.
- A stale or hardcoded `/etc/resolv.conf` in a custom AMI is a common failure after re-baking: the old VPC's `.2` address doesn't route from the new VPC.

### Other traps

- **Cross-AZ traffic has real latency and real cost.** If a symptom only reproduces under production traffic, check whether the failing path crosses an AZ boundary. It changes both the acceptable latency baseline and the bill.
- **VPC Flow Logs are your evidence when the guest shows nothing wrong.** If `ss`, `tcpdump`, and the guest firewall are all clean, flow logs show `REJECT` records at the SG/NACL layer that never reach the guest at all. This is the definitive "I've exhausted the guest OS" next step.
- **Flow logs won't show intra-instance or link-local traffic**, so they can't help with the DNS/IMDS throttling case above — that's what `ethtool -S` is for.
- **Multi-ENI instances need policy routing.** Attaching a second ENI without source-based routing rules produces asymmetric routing: packets arrive on `eth1` and leave via `eth0`'s default route, and the stateful SG on `eth1` drops the reply. `ip rule` and per-table default routes are the fix.

## Worked example: "Health checks failing intermittently, app logs show nothing"

**Symptom:** an ALB marks targets unhealthy every few minutes, then healthy again. The application's own logs show no errors, no restarts, nothing during the unhealthy windows.

**Step 1 — confirm the app is listening correctly, from inside the box.**
```bash
ss -tulpn | grep 8080
```
```
State   Recv-Q  Send-Q  Local Address:Port
LISTEN  128     128     0.0.0.0:8080     users:(("node",pid=2210))
```
Bound to `0.0.0.0`, so that classic is ruled out — but look at `Recv-Q 128` against `Send-Q 128`. On a LISTEN socket that's a **completely full accept queue**, right now, at the moment we happened to look. That's the finding, and it took one command.

**Step 2 — confirm with counters, since a snapshot could be a coincidence.**
```bash
ss -tan state syn-recv | wc -l
grep -E 'ListenOverflows|ListenDrops' /proc/net/netstat
```
```
ListenOverflows: 340
ListenDrops: 340
```
Non-zero and climbing. The kernel is dropping incoming SYNs because the accept queue filled faster than the application called `accept()`. This matches the intermittent pattern exactly: fine most of the time, failing during brief bursts.

**Step 3 — rule out the AWS layer before touching the app, since "intermittent" is also the signature of instance-level shaping.**
```bash
ethtool -S eth0 | grep exceeded
```
```
bw_in_allowance_exceeded: 0
pps_allowance_exceeded: 0
conntrack_allowance_exceeded: 0
```
All zero. Not shaping — the guest really is the bottleneck.

**Step 4 — confirm at packet level during a failure window.**
```bash
tcpdump -ni any 'port 8080 and tcp[tcpflags] & tcp-syn != 0' -c 200
```
SYNs arrive from the ALB's health-check IPs with no SYN-ACK during the bad window. That distinguishes the two candidate causes cleanly: a security group or NACL block would mean the SYN **never arrives**; here it arrives and is ignored, which is what a full accept queue looks like from the wire.

**Step 5 — find the real limit.**
```bash
cat /proc/sys/net/core/somaxconn        # 4096
systemctl show myapp -p LimitNOFILE     # fine
```
`somaxconn` is 4096, but `ss` showed a backlog of 128 — so the **application** is passing `listen(fd, 128)` and the kernel is silently capping to the smaller value. Raising the sysctl alone would have changed nothing, which is exactly the trap.

**Root cause:** the application's listen backlog is 128, too small for the burst of connections that coincides with periodic GC pauses. During a pause the accept loop stops draining, the queue fills in milliseconds, and the kernel drops new SYNs — which the ALB correctly reports as a failed health check.

**Fix:** raise the application's `listen()` backlog to match burst concurrency (and keep `somaxconn` at or above it). Separately, address the GC pause if it's frequent enough to matter beyond this symptom.
**Prevention:** alarm on `ListenDrops`/`ListenOverflows` directly. They identified this before a single application log line did, and they're two lines from `/proc` with no agent required.

## Cheat sheet

```bash
# Routes and interfaces
ip -br a ; ip r ; ip neigh ; ip -s link
ip r get 10.0.4.22              # which route WOULD be used (multi-ENI, policy routing)

# Sockets
ss -tulpn                       # listening + PID. On LISTEN: Recv-Q = queue depth, Send-Q = backlog
ss -tan state syn-recv          # backlog exhaustion / SYN flood
ss -tan state time-wait | wc -l # port exhaustion input
ss -tin                         # per-socket rtt, cwnd, retrans ← practise reading this
ss -s ; ss -tp dst 10.0.4.22

# Reachability
nc -zv host port                # test the ACTUAL port — a failed ping proves NOTHING on AWS
traceroute -T -p 443 host       # TCP; ICMP traceroute is filtered in VPC
mtr -rwbc 100 host

# Capture
tcpdump -ni any port 443 -c 100
tcpdump -ni any 'tcp[tcpflags] & tcp-syn != 0 and not port 22' -c 50
#   SYN out, no SYN-ACK back  → blocked above the guest (SG/NACL/route)
#   SYN in,  no SYN-ACK sent  → YOUR box: full accept queue / firewall / nothing listening
#   handshake ok, stall on big payload → path MTU

# DNS
dig +short example.com ; dig @169.254.169.253 example.com
getent hosts example.com        # what the APP gets — nsswitch, /etc/hosts, NSS
resolvectl status               # Ubuntu: resolv.conf is a 127.0.0.53 stub, this shows real upstreams
nmcli device show eth0 | grep DNS   # RHEL-family

# /proc
cat /proc/net/snmp              # TcpRetransSegs, TcpInErrs, TcpOutRsts
cat /proc/net/netstat           # ListenDrops, ListenOverflows, TCPSynRetrans
cat /proc/net/sockstat          # sockets in use, tw count
cat /proc/net/softnet_stat      # col2 = DROPPED, col3 = time_squeeze
cat /proc/interrupts ; cat /proc/softirqs
cat /proc/sys/net/ipv4/ip_local_port_range
cat /proc/sys/net/core/somaxconn

# Guest resource ceilings that look like network faults
ss -tan state time-wait | wc -l                     # EADDRNOTAVAIL → port exhaustion
#   fix order: app keep-alive/pooling > tcp_tw_reuse=1 > widen range > more source IPs
#   NEVER tcp_tw_recycle — broken behind NAT, REMOVED in kernel 4.12
cat /proc/sys/net/netfilter/nf_conntrack_{count,max}
dmesg | grep "table full"
cat /proc/PID/limits | grep 'open files' ; ls /proc/PID/fd | wc -l
systemctl show myapp -p LimitNOFILE                 # NOT /etc/security/limits.conf for systemd services

# ★ AWS instance-level shaping — invisible to every other tool
ethtool -i eth0                                     # confirm driver is 'ena' first
ethtool -S eth0 | grep -E 'exceeded|allowance'
#   bw_in/bw_out_allowance_exceeded   → instance bandwidth cap
#   pps_allowance_exceeded            → packet rate cap (small packets hit this first)
#   conntrack_allowance_exceeded      → instance connection-tracking cap
#   linklocal_allowance_exceeded      → DNS / IMDS / Time Sync throttled ← intermittent DNS failures
watch -n5 'ethtool -S eth0 | grep exceeded'         # cumulative — DIFF them, don't read absolutes

# Firewall (guest only)
nft list ruleset ; iptables-save
firewall-cmd --list-all     # RHEL-family (inactive by default on Amazon Linux)
ufw status verbose          # Debian-family (inactive by default on Ubuntu cloud images)

# Install
dnf install iproute bind-utils nmap-ncat tcpdump mtr traceroute ethtool iperf3 conntrack-tools nftables bcc-tools
apt  install iproute2 dnsutils netcat-openbsd tcpdump mtr traceroute ethtool iperf3 conntrack nftables bpfcc-tools

# AWS layers, in check order once the guest is clean
# Security group (STATEFUL, per-ENI) → NACL (STATELESS, per-subnet, needs explicit ephemeral
# return rule) → route table → VPC Flow Logs for REJECT records
# MTU: 9001 in-VPC, 1500 over an IGW. Blocking ICMP breaks PMTUD silently.
# DNS: VPC resolver at CIDR base+2 and 169.254.169.253, hard cap 1024 pps per ENI.
```