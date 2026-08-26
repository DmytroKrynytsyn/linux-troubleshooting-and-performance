# Boot & Startup Troubleshooting

An EC2 instance that's slow to boot, fails its status checks, or comes up without networking is one of the more stressful incidents to debug — you often can't SSH in yet, so your first evidence is the console output. This guide covers diagnosing it once you *can* get in (or via the EC2 Serial Console if you can't), plus the AWS-specific causes that don't show up on a laptop. If the instance won't boot at all, go to [recovery.md](recovery.md).

## 1-Minute Summary

- `systemd-analyze critical-chain`, not `blame` — `blame` lists slow units, `critical-chain` shows what was actually *waited on*.
- `journalctl -xb` for this boot. **`journalctl -b -1` only works if the journal is persistent — it usually isn't on Amazon Linux.** Check `journalctl --list-boots` before you rely on it.
- EC2 status checks pass but SSH times out → almost always guest-side networking (NIC renamed, firewall), not the hypervisor.
- Can't get in over SSH? **EC2 Serial Console** shows kernel output before networking is up.
- `cloud-init` runs after systemd's core targets and fails silently to a normal boot check — verify separately with `cloud-init status --long`.
- A bad `/etc/fstab` entry is the most common way to make an instance permanently unbootable. `nofail` prevents it.

## The boot chain, and where each stage fails

| Stage | Fails as | Where the evidence is |
|---|---|---|
| Firmware (BIOS/UEFI) | Instance never reaches GRUB | EC2 system log ("Get system log") |
| Bootloader (GRUB2) | "no such partition", GRUB rescue prompt | System log / Serial Console |
| Kernel | Panic, "unable to mount root fs" | System log / Serial Console |
| initramfs | Dracut emergency shell, root device missing | Serial Console — the kernel is up, so you can interact |
| systemd → `basic.target` | Emergency mode, failed mounts | Serial Console, then `journalctl -xb` |
| systemd → `multi-user.target` | Boots but a service is down | SSH works; `systemctl --failed` |
| cloud-init | Boots and looks fine, but isn't configured | SSH works; `cloud-init status --long` |

The dividing line that matters operationally: **before `sshd` starts you need the Serial Console; after it, you can use normal tools.** Knowing which side of that line your symptom sits on decides your entire approach.

## Methodology

1. **Did it boot at all, and how long did it take?** `systemd-analyze`.
2. **What actually delayed it?** `blame` shows the slowest individual units, but a slow unit isn't necessarily on the critical path — `critical-chain` shows what was blocking.
3. **Did anything fail outright?** `systemctl list-units --failed` and `journalctl -xb`.
4. **Is this boot's failure explained by the *previous* boot?** Only if the journal is persistent — verify first.
5. **If it's an EC2-specific symptom** (passes system status check, fails reachability, or SSH won't connect): work down the stack — NIC naming → network config service → guest firewall → sshd → security group.

## Commands, explained

### `systemd-analyze` — how long did boot take, and where did the time go

```bash
systemd-analyze                    # firmware + bootloader + kernel + initrd + userspace
systemd-analyze blame              # units sorted by their own startup time
systemd-analyze critical-chain     # the dependency chain that actually determined boot time
systemd-analyze critical-chain sshd.service     # ...for ONE unit specifically
systemd-analyze plot > boot.svg    # visual timeline
```
```
Startup finished in 1.9s (kernel) + 3.1s (initrd) + 42.8s (userspace) = 47.8s
```
The split tells you where to look before you run anything else. Long **kernel** time is hardware/driver probing. Long **initrd** is usually a storage or network-root problem. Long **userspace** — the common case — is a unit.

**`blame` vs. `critical-chain` — this trips people up.** `blame` lists every unit by how long *it individually* took, but many start in parallel and finish well before boot completes; they were never on the critical path. `critical-chain` walks the actual dependency graph and shows what was *waited on*. If `blame` shows a 20s unit but `critical-chain` doesn't mention it, it ran in the background and cost you nothing.

```bash
systemctl list-jobs                # what is STILL running or waiting, right now
systemd-analyze verify foo.service # syntax-check a unit before you deploy it
```
`systemctl list-jobs` is the right command during a hung boot: it shows the job that's blocking, which `blame` (a post-hoc tool) cannot tell you because boot never finished.

### `journalctl` — with the persistence caveat that matters most

```bash
journalctl -xb                     # this boot, with explanations expanded
journalctl -b -1 -p err            # the PREVIOUS boot, errors only
journalctl -u <unit> -b            # one service, this boot
journalctl -k                      # kernel messages only (the dmesg equivalent)
journalctl --since "10 min ago" -p warning
journalctl -f -u <unit>            # follow
```

> **`-b -1` silently doesn't work on most Amazon Linux instances.** The journal is only kept across reboots if `/var/log/journal/` exists. `systemd-journald`'s default `Storage=auto` means: persistent if that directory exists, volatile (RAM only, gone on reboot) if it doesn't.
>
> - **Ubuntu cloud images** create `/var/log/journal` → persistent by default.
> - **Amazon Linux 2 / 2023, RHEL** do not → **volatile**. Every reboot destroys the evidence of why you rebooted.
>
> Check, then fix:
> ```bash
> journalctl --list-boots          # one entry = volatile, evidence is gone
> ```
> ```bash
> mkdir -p /var/log/journal
> systemd-tmpfiles --create --prefix /var/log/journal
> systemctl restart systemd-journald
> journalctl --disk-usage
> ```
> Cap it so it can't fill the root volume — in `/etc/systemd/journald.conf`:
> ```ini
> [Journal]
> Storage=persistent
> SystemMaxUse=500M
> ```
> **Bake this into your AMI.** It's the difference between root-causing an unexplained reboot and shrugging at it, and it's a strong thing to raise unprompted in an interview: it shows you've been on the wrong side of it.

For an unexplained reboot with no persistent journal, your remaining evidence is the EC2 **system log** (which the hypervisor captures independently of the guest) and CloudWatch — assuming an agent was shipping logs off-box, which is the real lesson.

### `systemctl` — what's actually running

```bash
systemctl list-units --failed      # anything that gave up
systemctl status <unit>            # detail + last log lines
systemctl list-dependencies <unit> # why does this start after that
systemctl cat <unit>               # the effective unit file, including drop-ins
systemctl show <unit> -p <Prop>    # one resolved property, e.g. LimitNOFILE, Restart
systemctl is-enabled <unit>
```
`systemctl cat` matters on any real system: the file in `/usr/lib/systemd/system/` is frequently overridden by drop-ins in `/etc/systemd/system/<unit>.d/*.conf`. Reading the vendor file and drawing conclusions is a common wrong turn. `systemctl cat` shows the vendor file *and* every override, in application order.

### cloud-init — the stage that fails without failing

Most Amazon Linux and Ubuntu AMIs boot through cloud-init, which runs *after* systemd's core targets. It's the classic source of "boot completed, but the instance isn't actually ready."

```bash
cloud-init status --long           # done / running / error
cloud-init analyze blame           # per-module timing — cloud-init's own systemd-analyze
cloud-init analyze show            # full staged timeline
cat /var/log/cloud-init-output.log # stdout/stderr of YOUR user-data script ← usually the answer
cat /var/log/cloud-init.log        # cloud-init's own detailed log
journalctl -u cloud-init -u cloud-init-local -u cloud-config -u cloud-final
```
cloud-init runs in four ordered stages (`local` → `init` → `config` → `final`) and user-data scripts run in `final`. A failure in an earlier stage can silently skip later ones, so `cloud-init status --long` before hunting through logs.

`cloud-init analyze blame` is the underused one — a user-data script doing `dnf update` adds minutes to every instance launch, which matters enormously for autoscaling responsiveness and shows up nowhere in `systemd-analyze`.

To re-run during development:
```bash
cloud-init clean --logs --reboot   # clears state so user-data runs again
```

## Distro contrasts that matter at boot

### Bootloader configuration

**RHEL-family (Fedora, RHEL 8/9, Amazon Linux 2023) — BLS (BootLoaderSpec):** boot entries live as individual files in `/boot/loader/entries/*.conf`, not inside `grub.cfg`. Editing `grub.cfg` by hand does nothing.
```bash
grubby --info=ALL                                     # list entries
grubby --update-kernel=ALL --args="net.ifnames=0"     # add a cmdline arg to every kernel
grubby --update-kernel=ALL --remove-args="quiet"
grubby --set-default-index=0
```
`grubby` is the correct tool and it edits the BLS entries directly. Amazon Linux 2 is the exception in this family — it predates BLS and uses a traditional `grub.cfg`:
```bash
vi /etc/default/grub                                  # edit GRUB_CMDLINE_LINUX
grub2-mkconfig -o /boot/grub2/grub.cfg                # BIOS
grub2-mkconfig -o /boot/efi/EFI/amzn/grub.cfg         # UEFI — path differs!
```

**Debian-family (Ubuntu, Debian):**
```bash
vi /etc/default/grub                                  # GRUB_CMDLINE_LINUX_DEFAULT
update-grub                                           # wrapper around grub-mkconfig
```

Verify what actually took effect, on any family:
```bash
cat /proc/cmdline
```
This is the authoritative answer to "did my kernel argument apply." It's read-only and reflects what the kernel was actually handed, which is frequently not what you thought you configured.

### initramfs regeneration

**RHEL-family** uses **dracut**:
```bash
dracut -f                                    # rebuild for the running kernel
dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
lsinitrd | head -50                          # inspect contents
dracut --print-cmdline
```
**Debian-family** uses **initramfs-tools**:
```bash
update-initramfs -u                          # update for current kernel
update-initramfs -u -k all
lsinitramfs /boot/initrd.img-$(uname -r) | head -50
```
Forgetting to rebuild the initramfs after changing a storage driver, adding a module, or editing `/etc/crypttab` is a reliable way to produce a box that boots to a dracut emergency shell.

### Network configuration layer

| Family | Owner | Config location | Apply |
|---|---|---|---|
| Amazon Linux 2 | `network` (legacy scripts) | `/etc/sysconfig/network-scripts/ifcfg-eth0` | `systemctl restart network` |
| AL2023, RHEL 8/9, Fedora | NetworkManager | `/etc/NetworkManager/system-connections/` | `nmcli connection reload && nmcli con up eth0` |
| Ubuntu server | netplan → systemd-networkd | `/etc/netplan/*.yaml` | `netplan apply` |
| Debian (minimal) | ifupdown or systemd-networkd | `/etc/network/interfaces` or `/etc/systemd/network/` | `systemctl restart networking` |

```bash
# Which one is actually in charge?
systemctl status NetworkManager systemd-networkd network networking 2>/dev/null | grep -E 'Loaded|Active'
nmcli device show          # RHEL-family
netplan get                # Ubuntu
netplan try                # Ubuntu — applies with an automatic rollback timer. Use this over `apply` remotely
journalctl -u systemd-networkd -b
```
`netplan try` deserves a specific mention: it applies the config and reverts automatically after 120 seconds unless you confirm. On a remote instance where a bad network config means losing SSH, that's the difference between a mistake and an outage.

## AWS-specific gotchas

- **EC2 status checks are not systemd.** The **system status check** tests the underlying hardware/hypervisor. The **instance status check** (reachability) tests that your instance responds on the network. A boot that completes locally but fails the reachability check is almost always a guest networking problem — see [network.md](network.md).
- **The EC2 Serial Console** gives raw boot output including kernel messages before `sshd` is up. It's your only interactive view into a boot that hangs before networking. It requires being enabled at the account level and a password set for a local user — **do both before you need them**, because you cannot enable it on an instance you can't reach. "Get system log" in the console is the non-interactive fallback and works with no setup.
- **A custom AMI or kernel bump renames your NIC** — see the worked example below.
- **IMDSv2 is enforced on newer AMIs.** A user-data script or agent using the old unauthenticated `curl http://169.254.169.254/latest/meta-data/` will hang or 401 during boot, and often fails in a way that stalls cloud-init.
  ```bash
  TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
  ```
- **Time sync.** AWS provides the Time Sync Service at `169.254.169.123`; both Amazon Linux and Ubuntu preconfigure `chrony` for it. Clock skew after boot breaks TLS and SigV4 request signing, producing baffling `InvalidSignatureException` errors that look like credential problems.
  ```bash
  chronyc sources -v ; chronyc tracking ; timedatectl
  ```
- **fstab is the #1 way to make an instance permanently unbootable.** Any non-root mount that fails blocks `local-fs.target`, which blocks boot, which drops you into emergency mode with no network and no SSH. Two habits prevent it entirely:
  ```
  UUID=6d9e...  /data  xfs  defaults,nofail,x-systemd.device-timeout=5  0 2
  ```
  **`nofail`** — boot continues if the device is absent. **UUID, not `/dev/nvme1n1`** — NVMe enumeration order is not stable across reboots on Nitro. Always test with `mount -a` before rebooting; on a fresh instance it costs nothing and it's the one check that would prevent most of these incidents.

## Worked example: "New AMI passes system checks but SSH times out"

**Symptom:** you baked a new AMI from a snapshot after a kernel update, launched an instance, and the console shows both status checks passing — but SSH hangs and times out.

**Step 1 — get in via the Serial Console, since SSH is what's broken.**
```bash
ip -br a
```
```
lo               UNKNOWN        127.0.0.1/8
```
No `eth0`, no `ens5`. No interface is configured at all.

**Step 2 — check what the kernel actually named it.**
```bash
dmesg | grep -i -E 'eth|ens|renamed'
```
```
[    2.041]  ens5: renamed from eth0
```
Predictable network interface naming has kicked in: the new kernel generates persistent names from PCI bus location, but the netplan/ifcfg config still refers to `eth0`. The interface is up under a name nothing is configured for.

This also explains why the instance still *passed* both status checks: the system check only needs a healthy hypervisor-guest link, and the reachability check can pass on a link that carries no configured IP. Passing status checks is a much weaker statement than people assume.

**Step 3 — fix, two options.**
```bash
# Option A: force legacy naming (fastest, matches old AMI behaviour)
grubby --update-kernel=ALL --args="net.ifnames=0 biosdevname=0"     # AL2023/RHEL
# or Ubuntu:
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/&net.ifnames=0 biosdevname=0 /' /etc/default/grub && update-grub

# Option B (preferred going forward): update the network config to the new name
# Ubuntu:
sed -i 's/eth0/ens5/' /etc/netplan/*.yaml && netplan try
# AL2023 / RHEL:
nmcli connection modify "System eth0" connection.interface-name ens5
```
Option A is the right call mid-incident because it's reversible and restores the previous known-good behaviour fleet-wide. Option B is the right call in the follow-up PR. Saying that distinction out loud — stabilise now, fix properly after — is worth as much as the command itself.

Verify the cmdline actually took effect after reboot:
```bash
cat /proc/cmdline
```

**Step 4 — verify the rest of the chain, since NIC naming is only the first place this class of bug appears.**
```bash
systemctl status NetworkManager systemd-networkd network
journalctl -u systemd-networkd -b
systemctl status sshd
ss -tulpn | grep :22
journalctl -u sshd -b
firewall-cmd --list-all      # RHEL-family
ufw status verbose           # Debian-family
nft list ruleset
# then: security group, NACL, route table in the VPC console
```

**Root cause:** interface renamed by the kernel; network config not updated to match.
**Fix:** update the config rather than papering over it with `net.ifnames=0`, unless you specifically want legacy naming fleet-wide.
**Prevention:** bake AMI validation into the pipeline — launch from the candidate AMI, wait for the reachability check, SSH in and run a smoke test, *then* promote it to "known good." This entire incident is a missing pipeline stage, not a Linux problem, and that's the framing worth leading with.

## Cheat sheet

```bash
# Timing
systemd-analyze                          # kernel + initrd + userspace split tells you where to look
systemd-analyze blame                    # slowest units individually
systemd-analyze critical-chain           # ← what was actually WAITED ON
systemd-analyze critical-chain sshd.service
systemd-analyze plot > boot.svg
systemctl list-jobs                      # what's blocking RIGHT NOW during a hung boot

# What failed
systemctl list-units --failed
systemctl status <unit>
systemctl cat <unit>                     # vendor file + ALL drop-in overrides
systemctl list-dependencies <unit>
journalctl -xb ; journalctl -k ; journalctl -u <unit> -b

# ★ Journal persistence — check BEFORE relying on -b -1
journalctl --list-boots                  # one entry = volatile, previous boot is GONE
mkdir -p /var/log/journal && systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
# Ubuntu: persistent by default. Amazon Linux / RHEL: NOT. Bake it into the AMI.

# cloud-init
cloud-init status --long
cloud-init analyze blame                 # per-module timing — launch latency lives here
cat /var/log/cloud-init-output.log       # your user-data script's stdout/stderr
cloud-init clean --logs --reboot         # re-run user-data (dev only)

# Kernel cmdline
cat /proc/cmdline                        # authoritative: what the kernel ACTUALLY got
grubby --update-kernel=ALL --args="..."  # RHEL-family + AL2023 (BLS)
vi /etc/default/grub && grub2-mkconfig -o /boot/grub2/grub.cfg     # AL2 (BIOS)
vi /etc/default/grub && update-grub                                # Debian-family

# initramfs
dracut -f ; lsinitrd | head             # RHEL-family
update-initramfs -u ; lsinitramfs /boot/initrd.img-$(uname -r) | head   # Debian-family

# Network config layer — find the owner first
systemctl status NetworkManager systemd-networkd network networking
nmcli device show                        # RHEL-family
netplan get ; netplan try                # Ubuntu — `try` auto-reverts in 120s. Use it remotely.
journalctl -u systemd-networkd -b

# NIC renamed after a kernel/AMI change
ip -br a ; dmesg | grep -iE 'renamed|ens|eth'
# fix: net.ifnames=0 biosdevname=0 on cmdline, or update the config to the new name

# AWS
# Serial Console: enable it and set a local password BEFORE you need it.
# "Get system log" works with no setup.
# Status checks: system = hypervisor, instance = reachability. Both can pass on a broken guest.
# IMDSv2: PUT a token first, or user-data hangs.
# chronyc sources -v   → 169.254.169.123; skew breaks TLS and SigV4
# fstab: ALWAYS UUID + nofail + x-systemd.device-timeout. Test with `mount -a` before rebooting.
```