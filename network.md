# Network Troubleshooting

Network incidents on AWS have an extra wrinkle that pure-Linux guides skip: the guest OS is only *one* of several layers that can drop or block a connection. Security groups, NACLs, route tables, and the ENA driver all sit between "the app called `connect()`" and "the packet actually left the instance" — and each one fails silently from the guest's point of view. This guide works outward from the guest OS, then names where to look once the guest itself is clean.

## Methodology

1. **Can the box route to where it needs to go, at all?** `ip r`, `ip a`.
2. **Is anything actually listening, on the port you expect?** `ss -tulpn` — don't assume, verify.
3. **Guest firewall clean?** Check before blaming AWS-layer rules.
4. **If the guest looks clean and it's still not working: it's above the guest.** Security group → NACL → route table, in that order (that's the order AWS evaluates a lot of failure modes practically, and it's the cheapest-to-check order too).
5. **Confirm at the packet level if anything above is ambiguous.** `tcpdump` is ground truth — everything else is inference.
6. **DNS is a separate failure mode from routing** — a connection that resolves but doesn't connect, and a connection that never resolves, need different fixes.

## Commands, explained

### `ip` — interfaces, routes, neighbors

```bash
ip -br a                        # brief interface + address summary
ip r                            # routing table — is there actually a route to the destination?
ip neigh                        # ARP table — resolved MACs for recently-contacted hosts on the local subnet
ip -s link                      # RX/TX errors, dropped, overruns — physical/driver-level health
```
```
2: eth0    UP    10.0.1.15/24
```
```
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.15
```
No route to a destination subnet here means either the VPC route table is genuinely missing a route (check the console) or you're testing from the wrong subnet/AZ context — this is a two-minute check that rules out an entire class of "network is broken" reports before you touch anything else.

### `ss` — sockets, without guessing

```bash
ss -tulpn                       # listening sockets + owning PID — replaces netstat
```
```
tcp   LISTEN  0  128   0.0.0.0:8080   0.0.0.0:*   users:(("java",pid=4821,fd=42))
```
This answers "is my app actually listening on the port I think it is" directly — a shockingly common root cause is the app bound to `127.0.0.1:8080` instead of `0.0.0.0:8080`, which looks identical from inside the box (`curl localhost:8080` works fine) but refuses every connection from anywhere else, including a load balancer health check.

```bash
ss -tan state syn-recv          # connections stuck in SYN-RECV — backlog exhaustion or a SYN flood
ss -s                           # one-screen socket summary — fast sanity check on total connection counts
ss -tin                         # per-socket detail: rtt, cwnd, retransmits — TCP's own view of path quality
```
A growing count of `syn-recv` sockets usually means the accept backlog (`somaxconn`, and the app's own listen backlog) is too small for the connection rate — the OS is holding half-open connections it hasn't handed to the application yet.

### Reachability and path

```bash
ping <host>                      # ICMP — note: many AWS security groups block ICMP by default, a failed ping alone proves nothing
traceroute / mtr -rwbc 100 <host>   # mtr is traceroute + ping, continuously — shows *where* loss/latency is introduced, not just that it exists
```
**Don't trust a failed `ping` as proof of an outage on AWS** — ICMP is routinely blocked by security groups while TCP traffic on the actual application port flows fine. Test the real port (`nc -zv host port`, or just the actual app request) before concluding anything from ping alone.

### Packet capture — ground truth

```bash
tcpdump -ni any port 443 -c 100
```
When `ss` and application logs disagree, or the symptom is intermittent, a capture settles it — you either see the SYN leave and no SYN-ACK come back (path/firewall problem beyond the guest), or you see a full handshake and then the application-layer response is slow/wrong (not a network problem at all, look at the app).

### DNS

```bash
nslookup example.com
dig example.com
```
On EC2, also check `/etc/resolv.conf` points at the VPC's `.2` resolver (or your configured resolver) — a common failure after custom AMI/network config is a stale or hardcoded resolver that doesn't route correctly from a private subnet.

### Interface health at the driver level

```bash
ethtool eth0
```
Link speed/duplex, and (on ENA-driver instances) driver-level statistics that `ip -s link` doesn't surface — see the ENA note below.

### /proc alternatives — same data, scriptable

```bash
cat /proc/net/dev               # what ip -s link reads
cat /proc/net/tcp               # raw socket table (hex ports/addresses)
cat /proc/net/snmp               # TcpRetransSegs, InErrs — retransmit rate over time
cat /proc/net/netstat            # ListenDrops, ListenOverflows, TCPSynRetrans — backlog problems, in counter form
cat /proc/net/softnet_stat       # col2 = dropped, col3 = time_squeeze — NIC ring buffer / softirq backlog
cat /proc/interrupts ; cat /proc/softirqs   # interrupt distribution across cores — one core handling all NIC IRQs is a scaling ceiling
cat /proc/sys/net/ipv4/ip_local_port_range
cat /proc/sys/net/core/somaxconn
```
`ListenDrops`/`ListenOverflows` climbing in `/proc/net/netstat` is the counter-based confirmation of the `syn-recv` symptom above — connections arriving faster than the app calls `accept()`.

### Firewall — the guest layer only

```bash
iptables -L -n -v            # legacy, still common
nft list ruleset             # nftables, the modern replacement
firewall-cmd --list-all      # RHEL/firewalld front-end
ufw status verbose           # Ubuntu front-end
```

### Install

```bash
dnf install iproute            # ip, ss (usually preinstalled)
dnf install bind-utils         # dig, nslookup
dnf install tcpdump mtr traceroute nmap-ncat
dnf install ethtool
dnf install iperf3             # throughput testing between two hosts you control
dnf install nftables           # or iptables-services
dnf install bcc-tools          # tcplife, tcpretrans, tcpconnect
```

## AWS-specific gotchas

- **Security groups vs. NACLs — they are not the same layer, and confusing them wastes time.** Security groups are **stateful** (a reply to allowed outbound traffic is automatically allowed back in, no matching inbound rule needed) and attach to the **ENI**. NACLs are **stateless** (you must explicitly allow both directions) and apply at the **subnet** boundary. A connection that gets through the security group but stalls can still be an outbound NACL rule silently dropping the return traffic — check both, and remember NACLs need explicit ephemeral-port return rules that security groups handle automatically.
- **ENA driver and multi-queue.** On larger/newer instance types, the Elastic Network Adapter driver spreads RX/TX across multiple queues/cores; `ethtool -S eth0` exposes per-queue drop counters that `ip -s link`'s aggregate view hides — a single overloaded queue can drop packets while the interface's headline stats look fine.
- **Cross-AZ traffic has real latency and cost you won't see in a same-AZ test.** If a symptom only reproduces under production traffic, check whether the failing path crosses an AZ boundary — it changes both the acceptable latency baseline and (separately) the bill.
- **VPC Flow Logs are your evidence when the guest OS shows nothing wrong.** If `ss`, `tcpdump`, and the guest firewall are all clean but connections still fail, flow logs will show `REJECT` records at the security-group/NACL layer that never reach the guest at all — this is the single most useful "I've exhausted the guest OS" next step.
- **Path MTU mismatches show up as connections that establish fine but hang on larger payloads** — common after VPN/Direct Connect/peering changes. `ip link show` reports the configured MTU; a mismatch across a path shows as a handshake succeeding (small packets) and then a stall once payload exceeds the smaller path's MTU.

## Worked example: "Health checks failing intermittently, app logs show nothing"

**Symptom:** an ALB marks targets unhealthy every few minutes, then healthy again. The application's own logs show no errors, no restarts, nothing during the unhealthy windows.

**Step 1 — confirm the app is actually listening correctly, from inside the box:**
```bash
ss -tulpn | grep 8080
```
```
tcp   LISTEN  0  128   0.0.0.0:8080   0.0.0.0:*   users:(("node",pid=2210))
```
Bound to `0.0.0.0`, not `127.0.0.1` — ruled out.

**Step 2 — check for backlog exhaustion, since intermittent + no app-side errors suggests connections not reaching the app at all:**
```bash
ss -tan state syn-recv | wc -l
cat /proc/net/netstat | grep -i listen
```
```
ListenOverflows: 340
ListenDrops: 340
```
Non-zero and climbing — the kernel is dropping incoming SYNs because the listen backlog filled up faster than the application called `accept()`. Matches the intermittent pattern: fine most of the time, fails during brief bursts (a GC pause, a slow downstream call blocking the accept loop) when the app can't drain the backlog fast enough.

**Step 3 — confirm with a capture during a failure window:**
```bash
tcpdump -ni any port 8080 -c 200
```
Shows SYNs arriving from the ALB's health-check IPs with no SYN-ACK response during the bad window — consistent with a full backlog rather than a security-group/NACL block (which would show the SYN never arriving at all, not arriving-and-being-ignored).

**Root cause:** the application's listen backlog (and/or `somaxconn`) is too small for a brief burst of connections that coincides with occasional GC pauses, causing the OS to silently drop new SYNs during that window — which the ALB correctly interprets as a failed health check.

**Fix:** raise `net.core.somaxconn` and the application's own listen-backlog setting to match expected burst concurrency; separately, address the GC pause if it's frequent enough to matter beyond this symptom. **Prevention:** alarm on `ListenDrops`/`ListenOverflows` directly — they caught this before a single application log line did.

## Cheat sheet

```bash
ip -s link                      # RX/TX errors, dropped, overruns
ip -br a ; ip r ; ip neigh
ss -tulpn                       # listening sockets + PID
ss -tan state syn-recv          # SYN flood / backlog issues
ss -s                           # socket summary
ss -tin                         # per-socket: rtt, cwnd, retrans
ping / traceroute / mtr -rwbc 100 <host>
tcpdump -ni any port 443 -c 100

# /proc alternatives
cat /proc/net/dev               # what ip -s link reads
cat /proc/net/tcp               # raw socket table (hex)
cat /proc/net/snmp              # TcpRetransSegs, InErrs
cat /proc/net/netstat           # ListenDrops, ListenOverflows, TCPSynRetrans
cat /proc/net/softnet_stat      # col2 = dropped, col3 = time_squeeze (backlog)
cat /proc/interrupts ; cat /proc/softirqs
cat /proc/sys/net/ipv4/ip_local_port_range
cat /proc/sys/net/core/somaxconn

dnf install iproute            # ip, ss (usually preinstalled)
dnf install bind-utils         # dig, nslookup
dnf install tcpdump mtr traceroute nmap-ncat
dnf install ethtool
dnf install iperf3
dnf install nftables           # or iptables-services
dnf install bcc-tools          # tcplife, tcpretrans, tcpconnect

# AWS: security groups (stateful, ENI) vs NACLs (stateless, subnet) — check both, in that order
# then: VPC Flow Logs if the guest OS is clean but connections still fail
```
