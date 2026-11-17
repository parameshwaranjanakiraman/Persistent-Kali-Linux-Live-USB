# Persistent Kali Linux Live USB

A portable Kali Linux environment on a ~32 GB pendrive. It boots on any machine I own or am authorized to use, and keeps my files, tools, and settings across reboots through a dedicated ext4 persistence partition.

> **Status:** Persistence built and verified. Drive integrity testing (f3) and LUKS encryption are planned next (see [Future Work](#future-work)).

---

## Table of Contents
- [Goals](#goals)
- [Lab Environment](#lab-environment)
- [How Persistence Works](#how-persistence-works)
- [Build Steps](#build-steps)
- [Verification](#verification)
- [Boot Menu Reference](#boot-menu-reference)
- [Troubleshooting Log](#troubleshooting-log)
- [Security Notes](#security-notes)
- [Lessons Learned](#lessons-learned)
- [Future Work](#future-work)

---

## Goals
- Build a portable pentest environment that does not depend on the host OS.
- Understand how Linux live boot, hybrid ISOs, and persistence partitions work.
- Document the process and the real errors I hit, not just the happy path.

## Lab Environment

| Item | Details |
|---|---|
| Host OS (used for flashing) | Windows 11 |
| Target laptop | Acer TravelMate (UEFI, Secure Boot) |
| Pendrive | ~32 GB (29.3 GiB usable) |
| Image | Kali Linux 2026.3 **Live** (amd64) |
| Flashing tool | Rufus 4.15 (portable) |

## How Persistence Works

The Kali live image is a **read-only** hybrid ISO. Persistence adds a second layer:

1. At boot, Kali looks for a partition labelled **`persistence`**.
2. It reads `persistence.conf` from that partition.
3. A line of `/ union` tells it to save changes made anywhere in the filesystem as an overlay on top of the read-only base.

Result: the Kali OS stays untouched, and everything I change is stored on the persistence partition.

**Final disk layout:**

| Partition | Size | Purpose |
|---|---|---|
| `sda1` | ~5.5 GB | Kali live image (iso9660) |
| `sda2` | ~4.3 MB | EFI boot partition (vfat) |
| `sda3` | ~23.8 GB | `persistence` partition (ext4) |

---

## Build Steps

### 1. Download and verify the ISO
Download the **Live** image (the Installer image cannot do persistence) from kali.org, then verify the SHA256 checksum.

```powershell
Get-FileHash "D:\kali live\kali-linux-2026.3-live-amd64.iso" -Algorithm SHA256
```
Compare against the hash published on the Kali download page. They must match.

### 2. Flash with Rufus in DD mode
- Select the pendrive and the ISO.
- Click **START** and choose **Write in DD Image mode**.
- Do not format the drive if Windows offers to afterwards.

DD mode copies the hybrid image byte-for-byte, preserving its own partition table and bootloaders. ISO mode rebuilds the layout and can break persistence.

### 3. Boot the USB
- Press **F12** at startup and pick the USB entry (it appeared as *USB CDROM* because the image is a hybrid ISO).
- If boot is blocked, see [Secure Boot error](#1-verification-failed-0x1a-security-violation).
- Choose **Live system (amd64)** for the first boot (no persistence yet).

### 4. Identify the pendrive (read-only checks)
```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,TRAN
sudo parted /dev/sda print free
```
The pendrive was the disk with `TRAN = usb` (`sda`). The internal NVMe SSD was never touched. `print free` showed ~25.5 GB of unallocated space after `sda2`.

### 5. Create the persistence partition
```bash
sudo parted /dev/sda mkpart primary ext4 11599872s 100%
```
The start sector is the first multiple of 2048 after the free space begins, so the partition is properly aligned (see [Troubleshooting #3](#3-partition-alignment-warning)).

### 6. Format and configure
```bash
sudo mkfs.ext4 -O ^has_journal -L persistence /dev/sda3
sudo mkdir -p /mnt/usb
sudo mount /dev/sda3 /mnt/usb
echo "/ union" | sudo tee /mnt/usb/persistence.conf
sudo umount /dev/sda3
```
- `-L persistence` sets the label Kali searches for.
- `-O ^has_journal` disables the ext4 journal (see [Troubleshooting #4](#4-ext4-journal-error-on-mount)).

### 7. Boot with persistence
Reboot, press F12, pick the USB, and choose **Live system with USB persistence**.

---

## Verification

```bash
echo "persistence works" > ~/test.txt
sudo reboot
# boot again with the persistence entry, then:
cat ~/test.txt
```
Output: `persistence works`. The file survived a reboot, so persistence is working.

---

## Boot Menu Reference

| Entry | Use |
|---|---|
| Live system (amd64) | Normal live boot, nothing saved |
| Fail-safe mode | Live boot with features disabled, for hangs |
| Forensic mode | Live boot that does not touch internal drives |
| **Live system with USB persistence** | **The one to use daily** |
| Live system with USB Encrypted persistence | Needs a LUKS partition (not set up yet) |
| Start installer | Installs Kali to a disk. Avoid on a laptop with data |

---

## Troubleshooting Log

### 1. `Verification failed: (0x1A) Security Violation`
- **Cause:** Secure Boot rejected the loader on the pendrive.
- **Fix:** In BIOS, set a supervisor password (Acer hides the option until one exists), then set **Secure Boot: Disabled**. Checked BitLocker status first so a BIOS change would not trigger a recovery-key prompt.

### 2. PowerShell `Get-FileHash` positional parameter error
- **Cause:** The folder name contained a space (`kali live`), so PowerShell split the path into two arguments.
- **Fix:** Wrap the path in quotes.

### 3. Partition alignment warning
- **Message:** `The resulting partition is not properly aligned for best performance: 11598248s % 2048s != 0s`
- **Cause:** I started the partition at `5938MB`, which converted to a sector not divisible by 2048 (1 MiB).
- **Fix:** Cancelled, then restarted the partition at `11599872s` (5664 x 2048), the next aligned sector. Flash media performs best when partitions align to its erase-block size.

### 4. ext4 journal error on mount
- **Symptom:** `mount: wrong fs type, bad option, bad superblock on /dev/sda3`.
- **Diagnosis:** `blkid` showed a valid ext4 filesystem with the right label, but `dmesg` showed `JBD2: no valid journal superblock found`.
- **Fix:** Re-formatted with the journal disabled (`-O ^has_journal`). The mount then worked. A journal only protects against corruption after sudden power loss, and skipping it also reduces writes to flash.
- **Follow-up:** Run an f3 test to rule out a faulty drive.

### 5. Mounted ISO showing as a DVD drive in Windows
- **Cause:** Double-clicking an ISO in Windows mounts it as a virtual drive.
- **Fix:** Right-click, **Eject** (or `Dismount-DiskImage`). The ISO file itself was unaffected.

### 6. Slow performance in persistence mode
- Expected on USB flash: small random reads and writes are slow, and the first persistent boot builds caches.
- Checks: `lsusb -t` (USB 2.0 vs 3.0 link) and `sudo hdparm -t /dev/sda` for read speed.

---

## Security Notes
- Use this only on systems I own or have **written authorization** to test.
- The persistence partition is currently **unencrypted**. Anyone who finds the drive can read it, so no credentials, client data, or private keys should be stored on it until encryption is added.
- Secure Boot was disabled on my laptop to boot Kali. It can be turned back on after use.
- Default live credentials (`kali` / `kali`) should be changed before using the drive on any network.

## Lessons Learned
- Hybrid ISOs must be written in DD mode to keep their boot layout.
- Partition alignment matters on flash media, and parted warns you for a reason.
- A generic `mount` error hides the real cause. `dmesg` and `blkid` found it.
- Secure Boot and BIOS password quirks vary by vendor.
- Documenting failures is as valuable as documenting the working result.

## Future Work
- [ ] Run **f3write / f3read** to verify real capacity and data integrity
- [ ] Convert to **LUKS-encrypted persistence**
- [ ] Install a custom tool set and save package lists for quick rebuilds
- [ ] Add a second-stage backup of the persistence partition (`dd` or `rsync` image)

---
