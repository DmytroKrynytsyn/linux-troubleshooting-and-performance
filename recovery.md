# Recovery & Rescue

[boot.md](boot.md) covers an instance that boots badly. This one covers an instance that doesn't boot at all — no SSH, no application, nothing but a console log. It's the highest-stakes situation in this repo, and the one where a clear sequence matters most, because every option costs downtime and some of them cost data.

## 1-Minute Summary

- **Read the system log before you touch anything.** "Get system log" in the EC2 console needs no setup and tells you which stage failed.
- **Serial Console is the fast path** — interactive, pre-network, works when SSH is dead. But it must be enabled at the account level and needs an OS user with a password *set in advance*. Do this before you need it.
- If you can reach GRUB: `systemd.unit=emergency.target` gets you a root shell with only `/` mounted. `init=/bin/bash` skips systemd entirely — the bigger hammer.
- If you can't reach the guest at all: **stop the instance, detach the root volume, attach it to a rescue instance, fix it, reattach.** This always works. It costs 15–20 minutes.
- **Never `mkfs`, never terminate, and snapshot before you modify.** A snapshot is the cheapest insurance in AWS.

## Decision tree

```
Instance won't boot
│
├─ Can you see the console log?  → "Get system log" (no setup needed)
│  └─ Identify the failing stage: GRUB / kernel panic / initramfs / systemd unit / fstab
│
├─ Is Serial Console enabled and do you have a password-set OS user?
│  ├─ YES → connect, interrupt GRUB, boot to emergency.target, fix in place    ← ~2 minutes
│  └─ NO  → continue below
│
├─ Is the failure inside the filesystem (fstab, sshd config, bad package, full /)?
│  └─ Detach root volume → attach to rescue instance → chroot → fix → reattach  ← ~20 minutes
│
└─ Is the root volume itself corrupt or the failure unidentifiable?
   └─ Launch fresh from a known-good AMI, attach the old volume as data, recover files
```

The instinct to "just terminate and relaunch" is often correct for a stateless fleet member — and saying so in an incident is good judgement, not laziness. Reserve the recovery work for instances that hold state or whose failure you need to understand so it doesn't recur across the fleet.

## Step 0 — read the console log

```
EC2 Console → Instance → Actions → Monitor and troubleshoot → Get system log
```

No agent, no configuration, no SSH. It's the raw serial output the instance produced during boot. Match what you see against the stage:

| What the log shows | Stage | Where to go |
|---|---|---|
| Nothing at all, or firmware lines only | Firmware/GRUB | Root volume unreadable, wrong device, or corrupt bootloader |
| `error: no such partition` / `grub rescue>` | GRUB | Bootloader config broken — chroot repair |
| `Kernel panic - not syncing: VFS: Unable to mount root fs` | Kernel/initramfs | Wrong root UUID, or a rebuilt initramfs missing the ENA/NVMe driver |
| `dracut-initqueue timeout` / `Warning: /dev/... does not exist` | initramfs | Root device not found — UUID mismatch after a volume swap |
| `Dependency failed for /data` → `Welcome to emergency mode` | systemd | **Almost always `/etc/fstab`** |
| `Reached target Multi-User System` but no SSH | Boot fine, network/sshd broken | Go to [boot.md](boot.md) |

That last distinction is the important one: if the log reaches `Multi-User System`, the box booted. You have a networking or sshd problem, not a boot problem, and this guide is the wrong tool.

## Step 1 — EC2 Serial Console

Interactive terminal to the instance's serial port. Works before networking, works when sshd is dead, works from emergency mode.

**Three prerequisites, all of which must be arranged in advance:**

1. Account-level access enabled (`EC2 → Settings → EC2 Serial Console`), plus IAM permission for `ec2:SendSerialConsoleSSHPublicKey`.
2. A **Nitro-based** instance type. Xen-based older types don't support it.
3. **An OS user with a password actually set.** This is the one everyone gets wrong. Cloud AMIs ship with password login disabled, so the Serial Console gives you a login prompt you cannot satisfy.

Set the password now, on every instance you care about, while things are healthy:
```bash
sudo passwd ec2-user        # Amazon Linux / RHEL
sudo passwd ubuntu          # Ubuntu
```
Better: bake it into user-data or the AMI so it's never missing when it matters. This is a genuinely good thing to raise in an operational review — it's a five-minute change that converts a 20-minute recovery into a 2-minute one.

## Step 2 — boot into a rescue shell from GRUB

Once you have a console, reboot and interrupt GRUB. On EC2 the timeout is short (often 0–1 second), so hold `Esc` or spam it during the reboot. If the menu never appears, raise `GRUB_TIMEOUT` in `/etc/default/grub` on your AMIs as a standing improvement.

At the menu, press `e` to edit the entry, find the line starting `linux` or `linux16`, and append one of these to the end. Then `Ctrl-X` to boot. **Edits are one-shot** — they don't persist, which is exactly what you want.

| Append to cmdline | Result | Use when |
|---|---|---|
| `systemd.unit=rescue.target` | Single-user, `/` mounted, most services stopped | General repair with tooling available |
| `systemd.unit=emergency.target` | Minimal shell, only `/` mounted **read-only** | fstab is broken and rescue.target also fails |
| `init=/bin/bash` | No systemd at all, `/` read-only, PID 1 is your shell | systemd itself is broken, or you need to reset a password |
| `rd.break` | Shell **inside initramfs**, before switch_root; real root is at `/sysroot` | Root device or initramfs problem, LUKS, SELinux relabel |
| `single` / `1` | Legacy alias for rescue | Older systems |
| `net.ifnames=0 biosdevname=0` | Legacy NIC naming | NIC renamed after a kernel bump ([boot.md](boot.md)) |
| `systemd.log_level=debug` | Verbose systemd output | Boot hangs with no message |
| `selinux=0` | Disable SELinux for this boot | Suspected mislabelling blocking sshd — RHEL-family |

**With `init=/bin/bash` or `emergency.target`, `/` is read-only.** Remount before editing anything:
```bash
mount -o remount,rw /
# ... make your fix ...
mount -o remount,ro /
sync
exec /sbin/init          # or: reboot -f
```
Forgetting the remount and getting "Read-only file system" from `vi` is the classic five-minute detour.

**With `rd.break`,** you're in the initramfs and the real root is mounted at `/sysroot`:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
# ... fix ...
exit
mount -o remount,ro /sysroot
exit
```
On RHEL-family with SELinux enforcing, any file you create in this state has no label. Trigger a relabel before you leave or sshd may refuse to start:
```bash
touch /.autorelabel
```

## Step 3 — the universal fallback: detach the root volume

This works when everything else has failed. It always works. It costs downtime but nothing else.

```bash
# 1. STOP the instance (do not terminate). Note its AZ and the root device name (/dev/xvda).
aws ec2 stop-instances --instance-ids i-BROKEN

# 2. SNAPSHOT the root volume before you touch it. Non-negotiable.
aws ec2 create-snapshot --volume-id vol-BROKEN --description "pre-rescue $(date -Is)"

# 3. Detach it.
aws ec2 detach-volume --volume-id vol-BROKEN

# 4. Attach to a rescue instance IN THE SAME AZ, as a secondary device.
aws ec2 attach-volume --volume-id vol-BROKEN --instance-id i-RESCUE --device /dev/sdf
```

> **The rescue instance must be in the same Availability Zone.** EBS volumes are AZ-scoped. If it isn't, snapshot the volume and restore the snapshot into the rescue AZ instead.

> **Use the same OS family and a similar version for the rescue instance.** You're going to `chroot` into the broken root and run its package manager; a Ubuntu rescue host chrooting into an Amazon Linux root will fail on architecture and glibc mismatches.

On the rescue instance:
```bash
lsblk -f
```
```
nvme0n1                                        ← rescue instance's own root, DON'T TOUCH
└─nvme0n1p1  xfs   /
nvme1n1                                        ← the broken volume
├─nvme1n1p1  xfs   (the root filesystem)
└─nvme1n1p128 vfat (the EFI partition, if UEFI)
```
Identify by UUID, not by device number — NVMe numbering is not stable. Confirm with `blkid` against the UUID you expect from the broken instance's fstab.

**Duplicate-UUID warning:** if the rescue instance was launched from the *same AMI* as the broken instance, both root filesystems have the same filesystem UUID, and XFS in particular will refuse to mount the second one. Either use a different AMI for the rescue host, or mount with `-o nouuid`:
```bash
mount -o nouuid /dev/nvme1n1p1 /mnt/rescue
```

```bash
mkdir -p /mnt/rescue
mount /dev/nvme1n1p1 /mnt/rescue
ls /mnt/rescue                     # sanity check: does this look like a root filesystem?
```

### Fix it — the four things it usually is

**1. Broken `/etc/fstab`** (the most common cause by a wide margin):
```bash
vi /mnt/rescue/etc/fstab
```
Comment out the offending line, or repair it properly:
```
UUID=6d9e1c...  /data  xfs  defaults,nofail,x-systemd.device-timeout=5  0 2
```
`nofail` means a missing device logs a warning instead of dropping the machine into emergency mode. `x-systemd.device-timeout=5` stops systemd waiting the default 90 seconds per entry. **Both belong on every non-root mount on every cloud instance.** Verify the UUID actually exists before you reattach:
```bash
blkid | grep 6d9e1c
```

**2. Full root filesystem** — nothing can be written, so services fail at boot:
```bash
df -h /mnt/rescue
du -xh --max-depth=1 /mnt/rescue | sort -h
journalctl --directory=/mnt/rescue/var/log/journal --vacuum-size=100M
rm -rf /mnt/rescue/var/cache/{dnf,apt}/*
```
See [disk.md](disk.md) for the deleted-but-open-file case, which won't apply here (the process is gone) but will explain how it got full.

**3. Broken sshd config or a lost key:**
```bash
sshd -t -f /mnt/rescue/etc/ssh/sshd_config          # syntax check without starting anything
vi /mnt/rescue/home/ec2-user/.ssh/authorized_keys
chown -R 1000:1000 /mnt/rescue/home/ec2-user/.ssh   # ownership by UID — names differ across chroots
chmod 700 /mnt/rescue/home/ec2-user/.ssh
chmod 600 /mnt/rescue/home/ec2-user/.ssh/authorized_keys
```
Wrong permissions on `.ssh` cause sshd to silently refuse the key. Check these before assuming the key itself is wrong.

**4. A bad package, kernel, or bootloader — needs a full chroot.**

### Full chroot for package and bootloader work

```bash
for d in dev proc sys run; do mount --bind /$d /mnt/rescue/$d; done
mount --bind /dev/pts /mnt/rescue/dev/pts
cp /etc/resolv.conf /mnt/rescue/etc/resolv.conf     # so the chroot has DNS
chroot /mnt/rescue /bin/bash
```

Inside the chroot:

**RHEL-family (Amazon Linux, RHEL, Fedora):**
```bash
dnf history                              # what changed most recently?
dnf history undo last                    # roll back the last transaction
rpm -qa --last | head                    # recently installed packages
dracut -f --kver $(ls /lib/modules | tail -1)    # rebuild initramfs for a SPECIFIC kernel
grub2-mkconfig -o /boot/grub2/grub.cfg           # BIOS
grub2-mkconfig -o /boot/efi/EFI/amzn/grub.cfg    # UEFI (path varies: amzn, redhat, fedora)
grubby --info=ALL                                # inspect BLS entries on AL2023/RHEL9
```

**Debian-family (Ubuntu, Debian):**
```bash
grep -E ' (install|upgrade) ' /var/log/dpkg.log | tail -20
apt install --reinstall linux-image-$(ls /lib/modules | tail -1)
update-initramfs -u -k $(ls /lib/modules | tail -1)
update-grub
grub-install /dev/nvme1n1                # only if the bootloader itself is damaged
```

> **`--kver` / `-k` matters.** Inside a chroot, `uname -r` reports the *rescue instance's* running kernel, not the broken volume's. Rebuilding an initramfs against the wrong kernel version produces a system that boots into an even worse state. Always name the kernel explicitly from `/lib/modules`.

> **The initramfs must contain the ENA and NVMe drivers.** A hand-built or minimal initramfs that omits them produces `Kernel panic: VFS: Unable to mount root fs` on Nitro — the kernel can't see its own root volume. Verify before you reattach:
> ```bash
> lsinitrd /boot/initramfs-$KVER.img | grep -E 'ena|nvme'          # RHEL-family
> lsinitramfs /boot/initrd.img-$KVER | grep -E 'ena|nvme'          # Debian-family
> ```

**Reset a root or user password from the chroot:**
```bash
passwd ec2-user
touch /.autorelabel        # RHEL-family with SELinux — forces a relabel on next boot
```

Exit cleanly, in reverse order:
```bash
exit
for d in dev/pts dev proc sys run; do umount /mnt/rescue/$d; done
umount /mnt/rescue
sync
```
An unclean unmount is how you turn a recoverable instance into a corrupt one. If `umount` reports the device is busy, find the holder with `lsof +D /mnt/rescue` or `fuser -vm /mnt/rescue` rather than forcing it.

### Reattach and start

```bash
aws ec2 detach-volume --volume-id vol-BROKEN
aws ec2 attach-volume --volume-id vol-BROKEN --instance-id i-BROKEN --device /dev/xvda
aws ec2 start-instances --instance-ids i-BROKEN
```
**The root device name must be exactly what it was** — `/dev/xvda` for most Linux AMIs, `/dev/sda1` for some. Attaching the root volume at the wrong device name produces an instance that will not boot, and you get to do all of this again.

## Filesystem repair

If the volume itself is damaged, repair it on the rescue instance while it is **unmounted**.

**XFS** (Amazon Linux 2/2023, RHEL, Fedora) — there is no online fsck:
```bash
umount /mnt/rescue
xfs_repair -n /dev/nvme1n1p1        # dry run FIRST — reports without changing anything
xfs_repair /dev/nvme1n1p1
xfs_repair -L /dev/nvme1n1p1        # LAST RESORT: zeroes the log. DESTROYS uncommitted data
```
`-L` is genuinely destructive. Snapshot first, always, and only use it when a normal repair says the log is corrupt and cannot be replayed.

**ext4** (Ubuntu, Debian):
```bash
umount /mnt/rescue
e2fsck -n /dev/nvme1n1p1            # dry run
e2fsck -f -y /dev/nvme1n1p1         # force, auto-answer yes
tune2fs -l /dev/nvme1n1p1 | grep -i 'state\|mount count'
```

## Prevention — the changes worth making before the incident

This section is the one to bring up in a design review. Every item here converts a bad hour into a non-event.

| Change | Prevents |
|---|---|
| `nofail` + `x-systemd.device-timeout=5` on every non-root fstab entry | Unbootable instance from a missing or renamed volume |
| Mount by `UUID=`, never `/dev/nvme1n1` | NVMe enumeration order changing across reboots |
| Set an OS password on every instance for Serial Console | A 2-minute fix becoming a 20-minute one |
| Persistent journal (`mkdir /var/log/journal`) | Losing all evidence of why the previous boot failed |
| `GRUB_TIMEOUT=5` in your AMIs | Being unable to interrupt GRUB at all |
| Test `mount -a` after **every** fstab edit, before rebooting | Discovering the typo at the worst possible time |
| Automated AMI validation: launch → wait for reachability → SSH → run smoke test | Promoting a broken AMI to the fleet |
| Regular snapshots / DLM lifecycle policy | Recovery being impossible rather than merely slow |
| Immutable, stateless instances behind an ASG | The entire contents of this guide |

That last row is the real answer at fleet scale, and it's worth saying explicitly: the best recovery procedure is not needing one. Recovery skills matter for the instances that genuinely hold state, and for understanding a failure well enough that you can stop it recurring across the other five hundred.

## Cheat sheet

```bash
# 0. Read the log first — no setup required
# EC2 Console → Actions → Monitor and troubleshoot → Get system log
# "Reached target Multi-User System" in the log = it BOOTED. Not a boot problem. → boot.md

# 1. Serial Console (Nitro only; needs account setting + an OS user WITH A PASSWORD SET)
sudo passwd ec2-user        # DO THIS IN ADVANCE, on every instance

# 2. GRUB cmdline (press 'e', edit the 'linux' line, Ctrl-X. One-shot, doesn't persist)
systemd.unit=rescue.target        # single-user, / mounted
systemd.unit=emergency.target     # minimal, / read-only
init=/bin/bash                    # no systemd at all
rd.break                          # shell in initramfs; real root at /sysroot
net.ifnames=0 biosdevname=0       # NIC renaming
systemd.log_level=debug           # boot hangs silently
selinux=0                         # RHEL-family, suspected relabel issue

mount -o remount,rw /             # ← ALWAYS, before editing anything
# ... fix ...
mount -o remount,ro / ; sync ; exec /sbin/init

# 3. Universal fallback: detach the root volume
aws ec2 stop-instances --instance-ids i-BROKEN
aws ec2 create-snapshot --volume-id vol-BROKEN          # ← non-negotiable
aws ec2 detach-volume --volume-id vol-BROKEN
aws ec2 attach-volume --volume-id vol-BROKEN --instance-id i-RESCUE --device /dev/sdf
#   SAME AZ. Same OS family. Different AMI (or mount -o nouuid) to avoid UUID collision.

lsblk -f ; blkid
mount /dev/nvme1n1p1 /mnt/rescue          # add -o nouuid if XFS refuses (duplicate UUID)

# The four usual causes
vi /mnt/rescue/etc/fstab                  # ← #1 by a mile. Add nofail. Verify UUID with blkid
df -h /mnt/rescue                         # full root
sshd -t -f /mnt/rescue/etc/ssh/sshd_config ; ls -la /mnt/rescue/home/*/.ssh
# bad package/kernel → full chroot below

# Full chroot
for d in dev proc sys run; do mount --bind /$d /mnt/rescue/$d; done
mount --bind /dev/pts /mnt/rescue/dev/pts
cp /etc/resolv.conf /mnt/rescue/etc/
chroot /mnt/rescue /bin/bash

KVER=$(ls /lib/modules | tail -1)         # ← NOT uname -r; that's the rescue host's kernel
dnf history undo last ; dracut -f --kver $KVER ; grub2-mkconfig -o /boot/grub2/grub.cfg
apt install --reinstall linux-image-$KVER ; update-initramfs -u -k $KVER ; update-grub
lsinitrd /boot/initramfs-$KVER.img | grep -E 'ena|nvme'   # MUST contain both on Nitro
passwd ec2-user ; touch /.autorelabel

exit
for d in dev/pts dev proc sys run; do umount /mnt/rescue/$d; done
umount /mnt/rescue ; sync                 # clean unmount or you corrupt it

# Repair (UNMOUNTED)
xfs_repair -n /dev/nvme1n1p1 ; xfs_repair /dev/nvme1n1p1     # -L zeroes the log, destroys data
e2fsck -n /dev/nvme1n1p1  ; e2fsck -f -y /dev/nvme1n1p1

# 4. Reattach at the EXACT original device name
aws ec2 attach-volume --volume-id vol-BROKEN --instance-id i-BROKEN --device /dev/xvda
aws ec2 start-instances --instance-ids i-BROKEN

# Never: mkfs, terminate before snapshotting, force-unmount, or skip the dry-run repair.
```