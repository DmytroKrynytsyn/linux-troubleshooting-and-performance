# Boot & Startup Troubleshooting

An EC2 instance that's slow to boot, fails its status checks, or comes up without networking is one of the more stressful incidents to debug — you often can't SSH in yet, so your first evidence is the console output. This guide covers diagnosing it once you *can* get in (or via `EC2 Serial Console` if you can't), plus the AWS-specific causes that don't show up on a laptop.

## 1-Minute Summary

- `systemd-analyze critical-chain`, not `blame` — `blame` lists slow units, `critical-chain` shows what was actually *waited on*. Trust the latter.
- `journalctl -xb` for this boot; `journalctl -b -1 -p err` for the *previous* boot if the instance crashed/rebooted unexpectedly.
- EC2 status checks pass but SSH times out → almost always guest-side networking (NIC renamed, firewall), not the hypervisor. Check `ip -br a` and `dmesg | grep -i eth` first.
- Can't get in over SSH yet? Use the **EC2 Serial Console** — it shows kernel boot output before networking is even up.
- `cloud-init` runs after systemd's core targets and fails silently to a normal boot check — verify it separately with `cloud-init status --long` and `/var/log/cloud-init-output.log`.

## Methodology

1. **Did it boot at all, and how long did it take?** `systemd-analyze` gives you the headline number.
2. **What actually delayed it?** `blame` shows the slowest *individual* units, but a slow unit isn't necessarily on the critical path — `critical-chain` shows what was actually blocking.
3. **Did anything fail outright?** `systemctl list-units --failed` and `journalctl -xb`.
4. **If it's an EC2-specific symptom** (instance passes system status check but fails reachability check, or SSH just won't connect): work down the stack — NIC naming → network config service → firewall/security group → sshd itself.

## Commands, explained

### `systemd-analyze` — how long did boot take, and where did the time go

```bash
systemd-analyze                    # total time: firmware + bootloader + kernel + userspace
systemd-analyze blame              # units sorted by their own startup time
systemd-analyze critical-chain     # the actual dependency chain that determined boot time
systemd-analyze plot > boot.svg    # visual timeline — good for a ticket/PR, or a LinkedIn post
```

**`blame` vs. `critical-chain` — this trips people up.** `blame` lists every unit by how long *it individually* took, but many of those units start in parallel and finish well before boot completes — they were never on the critical path. `critical-chain` walks the actual dependency graph and shows you what was *waited on*. If `blame` shows a 20s unit but `critical-chain` doesn't mention it, it ran in the background and didn't cost you anything. Always trust `critical-chain` for "what do I actually need to fix."

### `journalctl` — what happened, in the kernel's and systemd's own words

```bash
journalctl -xb                     # this boot, with explanations (-x) expanded
journalctl -b -1 -p err            # the *previous* boot, errors only — for "it crashed and rebooted"
journalctl -u <unit> -b            # one service's logs for this boot
```

`-b -1` is the one people forget: if the instance rebooted unexpectedly (OOM, kernel panic, `sudo reboot` from user-data), the reason is in the *previous* boot's log, not the current one.

### `systemctl` — what's actually running

```bash
systemctl list-units --failed      # anything that gave up
systemctl status <unit>            # detail + last few log lines for one unit
```

## AWS-specific gotchas

- **EC2 status checks are not systemd.** "System status check" tests the underlying hardware/hypervisor; "Instance reachability check" tests that your instance responds on the network. A boot that completes locally but fails the reachability check is almost always a networking problem inside the guest — see the NIC/firewall sections below, and the [network guide](network.md).
- **No console access before boot finishes?** Use the **EC2 Serial Console** (or "Get system log" in the console) — it gives you raw boot output including kernel messages, before `sshd` is even up. This is your only view into a boot that hangs before networking comes online.
- **cloud-init.** Most Amazon Linux/Ubuntu AMIs boot through `cloud-init`, which runs *after* systemd's core targets but is a common source of "boot completed, but the instance isn't actually ready" (user-data script hung, package install failed). Check it separately:
  ```bash
  cloud-init status --long
  journalctl -u cloud-init -u cloud-init-local -u cloud-config -u cloud-final
  cat /var/log/cloud-init-output.log     # stdout/stderr of your user-data script
  ```
- **A custom AMI or kernel bump renames your NIC.** Covered in detail below — it's common enough after building a custom AMI that it deserves its own section.

## Worked example: "New AMI passes system checks but SSH times out"

**Symptom:** You baked a new AMI from a snapshot after a kernel update, launched an instance from it, and the EC2 console shows both status checks passing — but SSH connections just hang and time out.

**Step 1 — get in via Serial Console, since SSH is what's broken.**

```bash
ip -br a
```
```
lo               UNKNOWN        127.0.0.1/8
```
No `eth0`. The interface isn't there under the name your network config expects.

**Step 2 — check dmesg for what the kernel actually named it.**

```bash
dmesg | grep -i eth
```
```
[    2.041]  ens5: renamed from eth0
```
Predictable network interface naming (`ens5`) has kicked in because the new kernel/AMI generates persistent names from PCI bus location, but your netplan/ifcfg config still refers to the old `eth0`. The interface is up, just under a name nothing is configured to use — which is why the instance still *passed* status checks (those just need the hypervisor-guest link healthy) but reachability from your side (SSH) never completes.

**Step 3 — confirm the fix, two options:**

```bash
# Option A: force legacy naming back (fastest, matches old AMI behavior)
# add to kernel cmdline (GRUB_CMDLINE_LINUX in /etc/default/grub, then grub2-mkconfig):
net.ifnames=0 biosdevname=0

# Option B (preferred going forward): update the network config to the new name
# Ubuntu: edit /etc/netplan/*.yaml to reference ens5 instead of eth0, then:
netplan apply
```

**Step 4 — verify the rest of the chain, since NIC naming is only the first place this class of bug shows up:**

```bash
systemctl status NetworkManager systemd-networkd   # RHEL/AL2023 vs. Ubuntu server — check whichever owns this AMI
nmcli device show                                  # RHEL/AL
netplan get                                         # Ubuntu
journalctl -u systemd-networkd -b

systemctl status sshd
ss -tulpn | grep :22

firewall-cmd --list-all      # RHEL
ufw status verbose           # Ubuntu
nft list ruleset
# then, if the guest firewall is clean: check the security group and NACL in the VPC console
```

**Root cause:** interface renamed by the kernel, network config not updated to match. **Fix:** update the config (don't just paper over it with `net.ifnames=0` unless you specifically want legacy naming across your fleet). **Prevention:** bake AMI validation into your pipeline — launch, wait for reachability check, SSH in, before promoting an AMI to "known good."

## Cheat sheet

```bash
# Boot timing
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain     # ← the real answer to "what delayed boot"
systemd-analyze plot > boot.svg

# What happened
systemctl list-units --failed
systemctl status <unit>
journalctl -xb                     # this boot
journalctl -b -1 -p err            # previous boot, errors only
journalctl -u <unit> -b

# cloud-init
cloud-init status --long
cat /var/log/cloud-init-output.log

# 1. NIC renamed after kernel/AMI change
ip -br a
dmesg | grep -i eth
# fix: net.ifnames=0 biosdevname=0 on cmdline, or update the network config

# 2. Network config layer
systemctl status NetworkManager systemd-networkd   # RHEL vs. Ubuntu server
nmcli device show                                  # RHEL/AL
netplan get ; netplan apply                        # Ubuntu
journalctl -u systemd-networkd -b

# 3. sshd
systemctl status sshd
ss -tulpn | grep :22
journalctl -u sshd -b

# 4. Firewall / cloud
firewall-cmd --list-all      # RHEL
ufw status verbose           # Ubuntu
nft list ruleset
# then: security group, NACL, route table
```
