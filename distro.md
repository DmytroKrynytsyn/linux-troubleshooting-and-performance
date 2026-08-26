# Distro Matrix

You know the tool. You need the right name on *this* box. This is the lookup table.

Two families cover essentially all EC2 Linux workloads:

- **RHEL-family** — Amazon Linux 2, Amazon Linux 2023, RHEL, Rocky, CentOS Stream, Fedora
- **Debian-family** — Ubuntu (overwhelmingly the most common non-Amazon AMI), Debian

The differences that actually cost you time in an incident are not the package manager — everyone knows `dnf` vs `apt`. They're the ones below: package *names* that don't match the binary, the network configuration layer, and where logs land.

## Identify what you're on, in one command

```bash
cat /etc/os-release
```
```
NAME="Amazon Linux"
VERSION="2023"
ID="amzn"
ID_LIKE="fedora"
PLATFORM_ID="platform:al2023"
```
`ID_LIKE` is the field that matters for scripting — it tells you which family's conventions apply without enumerating every distro.

```bash
# Portable family detection for a runbook script
. /etc/os-release
case "$ID_LIKE $ID" in
  *rhel*|*fedora*|*amzn*) PKG=dnf ;;
  *debian*|*ubuntu*)      PKG=apt ;;
esac
```

Other quick identifiers:
```bash
uname -r                        # kernel. Ends in .amzn2 / .amzn2023 / -aws / -generic
hostnamectl                     # OS, kernel, virtualization, architecture
stat -fc %T /sys/fs/cgroup/     # cgroup2fs = v2, tmpfs = v1
rpm -q filesystem 2>/dev/null || dpkg -l | head -1     # which package DB exists
```

## Package names where the binary ≠ the package

This is the table worth memorising. Everything else you can guess.

| Binary / capability | RHEL-family | Debian-family |
|---|---|---|
| `ip`, `ss` | `iproute` | **`iproute2`** |
| `dig`, `nslookup` | `bind-utils` | **`dnsutils`** / `bind9-dnsutils` |
| `nc` | `nmap-ncat` | **`netcat-openbsd`** |
| `free`, `vmstat`, `slabtop`, `ps` | `procps-ng` | **`procps`** |
| `growpart` | `cloud-utils-growpart` | **`cloud-guest-utils`** |
| bcc/eBPF tools | `bcc-tools` | **`bpfcc-tools`** (binaries only `-bpfcc` suffixed) |
| `tshark` | `wireshark-cli` | **`tshark`** |
| `conntrack` | `conntrack-tools` | **`conntrack`** |
| `perf` | `perf` | **`linux-tools-$(uname -r)` + `linux-tools-aws`** |
| Debug symbols | `debuginfo-install <pkg>` | `<pkg>-dbgsym` (via the ddebs repo) |
| `mtr` | `mtr` | `mtr-tiny` or `mtr` |
| `ifconfig`, `netstat` | `net-tools` | `net-tools` |
| `sysstat`, `iotop`, `lsof`, `strace`, `tcpdump`, `ethtool`, `fio`, `xfsprogs`, `nvme-cli`, `bpftrace`, `chrony` | *same name* | *same name* |

**The one-liner that covers 90% of an incident toolkit:**
```bash
# RHEL-family
dnf install -y sysstat iproute bind-utils nmap-ncat net-tools tcpdump lsof strace \
  iotop ethtool procps-ng nvme-cli mtr traceroute bcc-tools

# Debian-family
apt install -y sysstat iproute2 dnsutils netcat-openbsd net-tools tcpdump lsof strace \
  iotop ethtool procps nvme-cli mtr traceroute bpfcc-tools
```

## Package management

| Task | RHEL-family | Debian-family |
|---|---|---|
| Install | `dnf install X` (`yum` on AL2) | `apt install X` |
| Search | `dnf search X` / `dnf provides */binary` | `apt search X` / `apt-file search bin/X` |
| What owns this file | `rpm -qf /usr/bin/ss` | `dpkg -S /usr/bin/ss` |
| List files in a package | `rpm -ql iproute` | `dpkg -L iproute2` |
| Recently installed | `rpm -qa --last \| head` | `grep ' install ' /var/log/dpkg.log \| tail` |
| **Transaction history** | `dnf history` | `less /var/log/apt/history.log` |
| **Roll back** | `dnf history undo last` | *(no equivalent — pin and downgrade manually)* |
| Verify integrity | `rpm -Va` | `debsums -c` (install `debsums`) |
| Downgrade | `dnf downgrade X` | `apt install X=version` |
| Hold a version | `dnf versionlock add X` | `apt-mark hold X` |
| Clean cache | `dnf clean all` | `apt clean` |

**`dnf history undo last` has no clean Debian equivalent**, and that's a genuinely useful thing to know when a bad package update broke a box at 3am. It's also a good argument in an AMI-strategy discussion.

**Amazon Linux 2 specifics:** uses `yum` (though `dnf` is aliased in later versions), and extra software comes from `amazon-linux-extras install <topic>` rather than a normal repo. AL2023 dropped that entirely in favour of standard `dnf` plus versioned repositories pinned by `releasever`.

## Network configuration — the biggest divergence

Getting this wrong means editing a file that nothing reads. **Always find the owner first:**
```bash
systemctl status NetworkManager systemd-networkd network networking 2>/dev/null | grep -E 'â—|Active'
```

| | Amazon Linux 2 | Amazon Linux 2023 / RHEL 9 | Ubuntu |
|---|---|---|---|
| Manager | `network` (legacy scripts) | **NetworkManager** | **netplan** → `systemd-networkd` |
| Config path | `/etc/sysconfig/network-scripts/ifcfg-eth0` | `/etc/NetworkManager/system-connections/` | `/etc/netplan/*.yaml` |
| Inspect | `cat ifcfg-eth0` | `nmcli device show` | `netplan get` |
| Apply | `systemctl restart network` | `nmcli connection up eth0` | `netplan apply` |
| **Safe apply** | — | — | **`netplan try`** (auto-reverts in 120s) |
| DNS config | `/etc/resolv.conf` (real file) | `/etc/resolv.conf`, NM-managed | **`/etc/resolv.conf` is a stub → `resolvectl status`** |

Two things to internalise:

**`netplan try` is the best remote-safe command in this table.** It applies the config, waits 120 seconds for you to confirm, and rolls back if you don't. Locking yourself out of a remote box with a network config change is a rite of passage; `netplan try` is how you skip it.

**On Ubuntu, `/etc/resolv.conf` is a symlink to a `systemd-resolved` stub showing `nameserver 127.0.0.53`.** That is not your real resolver. `resolvectl status` shows the actual per-link upstreams. Reading the stub and concluding "DNS points at localhost, that's the bug" is one of the most common wrong turns in cross-distro troubleshooting.

## Firewall

| | RHEL-family | Debian-family |
|---|---|---|
| Front-end | `firewalld` | `ufw` |
| Status | `firewall-cmd --list-all` | `ufw status verbose` |
| Backend | nftables | nftables |
| Raw | `nft list ruleset` / `iptables-save` | *same* |
| **Default on EC2** | **inactive** | **inactive** |

Both Amazon Linux and Ubuntu cloud AMIs ship with **no host firewall rules**, because AWS expects security groups to be the boundary. If you find a populated ruleset on an EC2 instance, something put it there — usually Docker or kube-proxy — and you should understand it before flushing it.

## Boot, bootloader, initramfs

| | RHEL-family | Debian-family |
|---|---|---|
| GRUB config source | `/etc/default/grub` | `/etc/default/grub` |
| Regenerate (BIOS) | `grub2-mkconfig -o /boot/grub2/grub.cfg` | `update-grub` |
| Regenerate (UEFI) | `grub2-mkconfig -o /boot/efi/EFI/<distro>/grub.cfg` | `update-grub` |
| Boot entries | **BLS** in `/boot/loader/entries/` (AL2023, RHEL 9) | entries inside `grub.cfg` |
| Manage entries | `grubby --info=ALL` / `grubby --update-kernel=ALL --args=...` | edit `/etc/default/grub` |
| Initramfs tool | **`dracut`** | **`initramfs-tools`** |
| Rebuild | `dracut -f --kver $KVER` | `update-initramfs -u -k $KVER` |
| Inspect contents | `lsinitrd /boot/initramfs-$KVER.img` | `lsinitramfs /boot/initrd.img-$KVER` |
| Config | `/etc/dracut.conf.d/` | `/etc/initramfs-tools/` |

**BLS is the thing that surprises people on AL2023 and RHEL 9.** Editing `GRUB_CMDLINE_LINUX` in `/etc/default/grub` and regenerating no longer changes existing boot entries, because each entry in `/boot/loader/entries/*.conf` carries its own `options` line. Use `grubby` instead:
```bash
grubby --update-kernel=ALL --args="net.ifnames=0"
grubby --update-kernel=ALL --remove-args="quiet"
grubby --default-kernel
```

## Logging

| | RHEL-family | Debian-family |
|---|---|---|
| Journal | `journalctl` | `journalctl` |
| **Persistent by default?** | **No** (AL2/AL2023 — volatile) | **Yes** (Ubuntu creates `/var/log/journal`) |
| Make persistent | `mkdir -p /var/log/journal && systemd-tmpfiles --create --prefix /var/log/journal && systemctl restart systemd-journald` | *(already)* |
| Traditional syslog | `/var/log/messages` | `/var/log/syslog` |
| Auth log | `/var/log/secure` | `/var/log/auth.log` |
| Package log | `dnf history` | `/var/log/apt/history.log` |
| `sar` history | `/var/log/sa/` | `/var/log/sysstat/` |
| Cloud-init output | `/var/log/cloud-init-output.log` | *same* |

**The persistent-journal difference bites hard.** On Amazon Linux, `journalctl -b -1` returns nothing after a reboot — the evidence for why the previous boot failed is simply gone. Check before relying on it:
```bash
journalctl --list-boots
```
Making the journal persistent on every AMI is a one-line change with a very high payoff during incidents.

## Filesystem and storage defaults

| | Amazon Linux 2/2023, RHEL, Fedora | Ubuntu, Debian |
|---|---|---|
| Root filesystem | **XFS** | **ext4** |
| Grow | `xfs_growfs -d /` (takes a **mountpoint**) | `resize2fs /dev/nvme0n1p1` (takes a **device**) |
| Shrink | **Impossible** | `resize2fs` (offline) |
| Repair (unmounted) | `xfs_repair` | `e2fsck` |
| Inodes | Dynamic, effectively unlimited | **Fixed at mkfs — can run out** |
| Inspect | `xfs_info /` | `tune2fs -l /dev/...` |

The consequence for troubleshooting: `df -i` is a routine check on Ubuntu and almost never the answer on Amazon Linux. And "we'll shrink it later" is not an option on XFS, ever.

## SELinux vs. AppArmor

| | RHEL-family | Debian-family |
|---|---|---|
| MAC system | **SELinux** | **AppArmor** |
| Status | `getenforce` / `sestatus` | `aa-status` |
| Temporarily permissive | `setenforce 0` | `aa-complain /path/to/profile` |
| Denials | `ausearch -m AVC -ts recent` | `journalctl \| grep -i apparmor` |
| Fix context | `restorecon -Rv /path` | edit the profile in `/etc/apparmor.d/` |
| Force full relabel | `touch /.autorelabel` + reboot | — |

Amazon Linux 2 ships SELinux **permissive** or disabled; AL2023 ships it **enforcing**. That's a real migration gotcha: a service that worked on AL2 can fail on AL2023 purely on file contexts, particularly anything serving files from a non-standard path. `ausearch -m AVC -ts recent` is the first thing to run when a service starts fine but can't read its own data.

## Users and defaults on EC2

| AMI | Default user | sudo |
|---|---|---|
| Amazon Linux 2 / 2023 | `ec2-user` | passwordless |
| RHEL | `ec2-user` | passwordless |
| Ubuntu | `ubuntu` | passwordless |
| Debian | `admin` or `debian` | passwordless |

Relevant for Serial Console: none of these have a password set by default, so the Serial Console gives you a login prompt you can't satisfy. See [recovery.md](recovery.md).

## Time synchronisation

Both families use `chrony` on modern EC2 AMIs, pointed at the Amazon Time Sync Service:
```bash
chronyc sources -v
chronyc tracking
grep -r 169.254.169.123 /etc/chrony*
```
Clock skew breaks SigV4 request signing, TLS certificate validation, and log correlation across a fleet. It presents as "random `SignatureDoesNotMatch` errors" and is diagnosed nowhere else.

## Service management — the parts that are actually the same

systemd is universal on both families now, so this is mostly common ground. Worth having anyway:

```bash
systemctl status X ; systemctl list-units --failed ; systemctl list-jobs
systemctl cat X                 # the unit file PLUS all drop-ins, merged ← use this, not `find`
systemctl show X                # every resolved property
systemctl edit X                # create a drop-in (never edit the vendor unit directly)
systemctl daemon-reload         # after any unit change
systemd-analyze verify X.service
journalctl -u X -b --no-pager
systemctl list-dependencies X --reverse
```
`systemctl cat` is the underrated one: drop-ins in `/etc/systemd/system/X.service.d/` override the vendor unit, and `cat` shows you the merged result rather than making you go find them.