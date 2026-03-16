# Complete Linux Learning Guide with Hands-On Exercises

**Comprehensive System Administration Guide**  
**System:** RHEL 9 (rhel9exam01) on VMware ESXi 6.7  
**Learning Path:** Boot Process → Filesystems → Storage → Memory Management

---

## Table of Contents

1. [Linux Filesystem Structure](#1-linux-filesystem-structure)
2. [Boot Process & Architecture](#2-boot-process--architecture)
3. [Disk & Storage Management](#3-disk--storage-management)
4. [Memory & Swap Management](#4-memory--swap-management)
5. [Package Management (RPM/DNF)](#5-package-management-rpmdnf)
6. [Symbolic Links & Mount Points](#6-symbolic-links--mount-points)
7. [System Monitoring & Performance](#7-system-monitoring--performance)
8. [Troubleshooting & Recovery](#8-troubleshooting--recovery)
9. [Hands-On Lab Exercises](#9-hands-on-lab-exercises)
10. [Quick Reference Commands](#10-quick-reference-commands)

---

# 1. Linux Filesystem Structure

## 1.1 Filesystem Hierarchy Standard (FHS)

### Complete Directory Structure

```
/ (root)
├── /boot           Boot loader files (kernels, initramfs)
│   └── /boot/efi   UEFI boot partition (FAT32)
├── /dev            Device files (hardware interfaces)
├── /etc            System configuration files
├── /home           User home directories
├── /root           Root user's home directory
├── /tmp            Temporary files
├── /var            Variable data (logs, caches, spools)
│   ├── /var/log    Log files
│   ├── /var/cache  Cached data
│   └── /var/lib    State information
├── /usr            User programs and data
│   ├── /usr/bin    User commands
│   ├── /usr/sbin   System binaries
│   ├── /usr/lib    Libraries
│   └── /usr/share  Shared data
├── /opt            Optional third-party software
├── /mnt            Temporary mount points
├── /media          Removable media mount points
├── /proc           Process and kernel information (virtual)
├── /sys            Device and driver information (virtual)
└── /run            Runtime data (tmpfs)
```

### Symbolic Link Redirects (Modern RHEL 9)

```
/bin    → usr/bin
/sbin   → usr/sbin
/lib    → usr/lib
/lib64  → usr/lib64
```

**Purpose:** Backward compatibility while consolidating to `/usr`

---

## 1.2 Your RHEL9 System Layout

### Physical Disk Structure

```
/dev/sda (60GB SSD)
│
├─ /dev/sda1 (600MB)  → /boot/efi  [vfat/FAT32]
├─ /dev/sda2 (1GB)    → /boot      [xfs]
└─ /dev/sda3 (58.4GB) → LVM Physical Volume
   └─ rhel_rhel9exam01 (Volume Group)
      ├─ root (37GB)  → /       [xfs]
      ├─ swap (3.9GB) → [SWAP]  [swap]
      └─ home (18GB)  → /home   [xfs]
```

### Directory Locations After Boot

```
/ (from root LV on sda3/LVM)
├── /bin       → usr/bin (symlink)
├── /etc/      (actual files on root LV)
├── /usr/      (actual directories on root LV)
├── /var/      (actual directories on root LV)
├── /boot/     (mount point - sda2 mounted here)
│   └── /boot/efi/ (mount point - sda1 mounted here)
└── /home/     (mount point - home LV mounted here)
```

---

## 1.3 Key Directories Deep Dive

### /boot - Boot Files

**Purpose:** Contains bootloader and kernel files  
**Filesystem:** XFS  
**Size:** ~1GB  
**Why separate:** Bootloaders can't read LVM/encryption

**Contents:**
```bash
/boot/
├── vmlinuz-*              # Kernel images
├── initramfs-*            # Initial RAM filesystem
├── System.map-*           # Kernel symbol tables
├── config-*               # Kernel build configuration
├── grub2/                 # GRUB2 configuration
│   ├── grub.cfg          # Main GRUB config (auto-generated)
│   └── grubenv           # GRUB environment variables
└── efi/                   # EFI partition mount point
    └── EFI/
        └── redhat/
            ├── shimx64.efi   # Secure Boot shim
            └── grubx64.efi   # GRUB2 for UEFI
```

---

### /dev - Device Files

**Purpose:** Hardware device interfaces  
**Type:** Virtual filesystem (devtmpfs)  
**Created by:** Kernel and udev

**Key Device Files:**
```
Block Devices (storage):
/dev/sda, /dev/sdb        # SATA/SCSI disks
/dev/sda1, /dev/sda2      # Partitions
/dev/nvme0n1              # NVMe SSD
/dev/sr0                  # CD/DVD drive
/dev/dm-0                 # Device mapper (LVM)
/dev/mapper/...           # LVM logical volumes

Character Devices:
/dev/null                 # Discards all data
/dev/zero                 # Produces zeros
/dev/random               # True random generator
/dev/urandom              # Pseudo-random generator
/dev/tty                  # Current terminal
/dev/ttyS0                # Serial port
```

**Device Naming:**
```
sda = s(SCSI) d(disk) a(first)
sdb = second disk
nvme0n1 = NVMe device 0, namespace 1
dm-0 = Device mapper device 0
```

---

### /etc - Configuration Files

**Purpose:** System-wide configuration  
**Type:** Regular directory on root filesystem  
**Format:** Plain text (human-readable)

**Critical Configuration Files:**

```bash
User/Group Management:
/etc/passwd              # User accounts (world-readable)
/etc/shadow              # Encrypted passwords (root only)
/etc/group               # Group definitions
/etc/gshadow             # Group passwords

System Identity:
/etc/hostname            # System hostname
/etc/hosts               # Static hostname to IP mappings
/etc/resolv.conf         # DNS resolver configuration
/etc/os-release          # OS version information

Filesystem:
/etc/fstab               # Filesystem mount table (permanent mounts)

Boot & Init:
/etc/default/grub        # GRUB2 user settings
/etc/grub.d/             # GRUB2 config generator scripts

Network:
/etc/sysconfig/network-scripts/  # Network interface configs (RHEL)
/etc/NetworkManager/     # NetworkManager configuration

Services:
/etc/systemd/            # Systemd configuration
/etc/ssh/                # SSH server/client configuration

Package Management:
/etc/yum.repos.d/        # DNF/YUM repository definitions
/etc/dnf/dnf.conf        # DNF configuration

System Settings:
/etc/sysctl.conf         # Kernel parameters
/etc/security/           # Security policies
/etc/sudoers             # Sudo permissions (edit with visudo!)
```

---

### /var - Variable Data

**Purpose:** Files that change during system operation  
**Growth:** Can grow significantly over time

```bash
/var/log/                # Log files
├── messages             # General system logs
├── secure               # Authentication logs
├── boot.log             # Boot messages
└── httpd/               # Web server logs

/var/cache/              # Application caches
/var/lib/                # State information
├── rpm/                 # RPM database
├── mysql/               # MySQL databases
└── docker/              # Docker data

/var/spool/              # Queues
├── mail/                # Mail queue
├── cron/                # Cron job spools
└── cups/                # Print queue

/var/tmp/                # Temporary files (survives reboot)
```

---

### /proc - Process Information (Virtual)

**Purpose:** Kernel and process information  
**Type:** Virtual filesystem (procfs)  
**Contents:** Generated by kernel in real-time

```bash
/proc/cpuinfo            # CPU information
/proc/meminfo            # Memory statistics
/proc/swaps              # Active swap devices
/proc/mounts             # Currently mounted filesystems
/proc/[PID]/             # Per-process information
/proc/sys/               # Kernel parameters (tunable)
├── vm/swappiness        # Swap behavior control
└── net/                 # Network parameters
```

---

### /sys - System Device Information (Virtual)

**Purpose:** Hardware and driver information  
**Type:** Virtual filesystem (sysfs)

```bash
/sys/block/              # Block devices
/sys/class/              # Device classes
/sys/firmware/efi/       # UEFI firmware variables
/sys/devices/            # Device tree
```

---

# 2. Boot Process & Architecture

## 2.1 Complete Boot Sequence

### Stage 1: UEFI Firmware

```
Power On
    ↓
UEFI Firmware (motherboard chip)
    ↓
POST (Power-On Self-Test)
    ├─ Test RAM
    ├─ Test CPU
    ├─ Detect storage devices
    └─ Initialize hardware
    ↓
Read boot configuration from NVRAM
    ↓
Scan for EFI System Partition (ESP)
    ├─ Must be FAT32 (UEFI requirement!)
    └─ Located at /boot/efi (sda1)
    ↓
Load bootloader: /boot/efi/EFI/redhat/shimx64.efi
    ↓
Secure Boot verification (if enabled)
    ↓
Transfer control to GRUB2
```

**Key Points:**
- UEFI can ONLY read FAT32
- Cannot read: ext4, xfs, LVM, encryption
- Stored on motherboard firmware chip
- Replacement for legacy BIOS

---

### Stage 2: GRUB2 Bootloader

```
GRUB2 starts (grubx64.efi)
    ↓
Read basic config: /boot/efi/EFI/redhat/grub.cfg
    ↓
Mount /boot partition (sda2, XFS)
    ↓
Read full config: /boot/grub2/grub.cfg
    ↓
Display boot menu (5 second timeout)
    ┌───────────────────────────────────────┐
    │ > Red Hat Enterprise Linux 9          │
    │   (kernel 5.14.0-611.24.1)            │
    │   Red Hat Enterprise Linux 9          │
    │   (kernel 5.14.0-70.22.1)             │
    │   UEFI Firmware Settings              │
    └───────────────────────────────────────┘
    ↓
User selects (or auto-select after timeout)
    ↓
Load kernel: /boot/vmlinuz-5.14.0-611.24.1.el9_7.x86_64
    ↓
Load initramfs: /boot/initramfs-5.14.0-611.24.1.el9_7.x86_64.img
    ↓
Pass kernel parameters (from grub.cfg)
    ↓
Transfer control to Linux kernel
```

**Key Points:**
- GRUB2 can read: ext4, xfs, btrfs (simple filesystems)
- GRUB2 cannot read: LVM directly (needs modules)
- Configuration auto-generated (don't edit grub.cfg!)
- User settings in /etc/default/grub

---

### Stage 3: Linux Kernel & initramfs

```
Kernel executes in memory
    ↓
Decompress and mount initramfs (temporary root filesystem)
    ↓
initramfs contains:
    ├─ LVM drivers
    ├─ Device drivers
    ├─ Basic utilities (mount, lvm commands)
    └─ init script
    ↓
Run init script from initramfs:
    ├─ Scan for storage devices
    ├─ Activate LVM: vgchange -ay
    ├─ LVM volumes now available! (/dev/mapper/...)
    ├─ Mount real root: /dev/mapper/rhel_rhel9exam01-root on /
    └─ Switch from initramfs to real root filesystem
    ↓
Execute /sbin/init (systemd)
    ↓
systemd initialization
    ├─ Mount /boot (from /etc/fstab)
    ├─ Mount /boot/efi (from /etc/fstab)
    ├─ Mount /home (from /etc/fstab)
    ├─ Activate swap
    ├─ Start services
    └─ Reach default target (multi-user.target or graphical.target)
    ↓
System fully booted!
Login prompt appears
```

**Key Concepts:**
- Chicken-and-egg: LVM needs kernel, kernel in /boot, /boot might be on LVM
- Solution: initramfs contains LVM drivers
- initramfs is temporary, replaced by real root
- /etc/fstab controls permanent mounts

---

## 2.2 UEFI (Unified Extensible Firmware Interface)

### UEFI vs BIOS Comparison

| Feature | BIOS (Legacy) | UEFI (Modern) |
|---------|---------------|---------------|
| **Released** | 1970s | 2005+ |
| **Code** | 16-bit | 32/64-bit |
| **Interface** | Text only | Text or GUI |
| **Boot time** | Slower | Faster |
| **Partition table** | MBR | GPT |
| **Max partitions** | 4 primary | 128+ |
| **Max disk size** | 2TB | 9.4 ZB |
| **Secure Boot** | No | Yes |
| **Network boot** | Limited | Built-in |
| **Boot partition** | Any | Must be FAT32 |

### UEFI Requirements

```
UEFI System Partition (ESP):
- Must be FAT32 filesystem
- Typically 200-600MB
- Contains .efi bootloader files
- Located at /boot/efi

Why FAT32?
- UEFI spec requires it
- Built into firmware (hardware limitation)
- No ext4, xfs, NTFS, LVM support in UEFI
```

### UEFI Boot Entries

```bash
# View boot entries
efibootmgr

# Example output:
BootCurrent: 0000
Timeout: 5 seconds
BootOrder: 0000,0001,0002
Boot0000* Red Hat Enterprise Linux
Boot0001* Windows Boot Manager
Boot0002* UEFI: USB Drive

# Add new entry
sudo efibootmgr --create \
  --disk /dev/sda \
  --part 1 \
  --label "My Boot" \
  --loader '\EFI\custom\bootx64.efi'

# Delete entry
sudo efibootmgr --bootnum 0003 --delete-bootnum

# Change boot order
sudo efibootmgr --bootorder 0000,0002,0001
```

---

## 2.3 GRUB2 (Grand Unified Bootloader version 2)

### GRUB2 Configuration Files

```
User Settings (YOU EDIT THIS):
/etc/default/grub
├─ GRUB_TIMEOUT=5          # Menu timeout (seconds)
├─ GRUB_DEFAULT=saved      # Default kernel (0, 1, or "saved")
├─ GRUB_CMDLINE_LINUX="..." # Kernel parameters
└─ GRUB_DISABLE_RECOVERY   # Show/hide recovery entries

Config Generator Scripts:
/etc/grub.d/
├─ 00_header               # Basic settings
├─ 10_linux                # Linux kernel detection
├─ 30_os-prober            # Detect other OS (Windows)
├─ 40_custom               # Custom entries (YOU CAN EDIT)
└─ 41_custom               # More custom entries

Generated Config (DON'T EDIT!):
/boot/grub2/grub.cfg       # Auto-generated from above

UEFI Bootloader:
/boot/efi/EFI/redhat/
├─ grubx64.efi             # GRUB2 for UEFI
├─ shimx64.efi             # Secure Boot shim
└─ grub.cfg                # Minimal config (points to /boot/grub2/grub.cfg)
```

### GRUB2 Workflow

```
Edit settings:
  /etc/default/grub
      ↓
  OR add custom entries:
  /etc/grub.d/40_custom
      ↓
Regenerate config:
  sudo grub2-mkconfig -o /boot/grub2/grub.cfg
      ↓
Reboot to test:
  sudo reboot
```

### Important GRUB2 Parameters

```bash
# In /etc/default/grub:

GRUB_CMDLINE_LINUX="..."
Common parameters:
├─ crashkernel=auto        # Memory for crash dumps
├─ resume=/dev/mapper/...  # Hibernation resume device
├─ rd.lvm.lv=...           # LVM volumes to activate
├─ rhgb                    # Red Hat graphical boot (splash)
├─ quiet                   # Less verbose boot messages
├─ debug                   # Verbose debug output
├─ single                  # Single-user mode
└─ systemd.unit=rescue.target  # Boot to rescue mode
```

---

## 2.4 initramfs (Initial RAM Filesystem)

### Purpose & Contents

```
initramfs is a temporary root filesystem that:
1. Contains drivers needed to mount real root
2. Includes LVM, RAID, encryption tools
3. Allows boot before real root is accessible
4. Self-destructs after switching to real root

Contents:
├─ Kernel modules (drivers)
│  ├─ LVM drivers
│  ├─ Filesystem drivers (xfs, ext4)
│  ├─ SCSI/SATA drivers
│  └─ RAID drivers
├─ Essential binaries
│  ├─ /bin/sh (shell)
│  ├─ /sbin/lvm (LVM tools)
│  └─ /sbin/mount (mount command)
├─ init script (startup logic)
└─ Configuration files
```

### Working with initramfs

```bash
# List contents
lsinitrd /boot/initramfs-$(uname -r).img

# Extract specific file
lsinitrd /boot/initramfs-$(uname -r).img | grep lvm

# Rebuild initramfs (after hardware changes)
sudo dracut --force

# Rebuild for specific kernel
sudo dracut --force /boot/initramfs-5.14.0-611.24.1.el9_7.x86_64.img 5.14.0-611.24.1.el9_7.x86_64

# Add module to initramfs
echo 'add_drivers+=" mymodule "' | sudo tee /etc/dracut.conf.d/mymodule.conf
sudo dracut --force
```

---

# 3. Disk & Storage Management

## 3.1 Partition Types & Naming

### Device Naming Convention

```
SATA/SCSI/SAS Disks:
/dev/sda, /dev/sdb, /dev/sdc
├─ sd = SCSI Disk (or Storage Device)
├─ a, b, c = First, second, third disk
└─ 1, 2, 3 = Partition numbers

Examples:
/dev/sda1 = First partition on first disk
/dev/sdb2 = Second partition on second disk

NVMe SSDs:
/dev/nvme0n1, /dev/nvme1n1
├─ nvme = NVMe device
├─ 0, 1 = Controller number
├─ n1, n2 = Namespace number
└─ p1, p2 = Partition numbers

Examples:
/dev/nvme0n1p1 = First partition on first NVMe
/dev/nvme1n1p2 = Second partition on second NVMe

Device Mapper (LVM, encryption):
/dev/dm-0, /dev/dm-1
/dev/mapper/vg_name-lv_name

Examples:
/dev/mapper/rhel_rhel9exam01-root
/dev/mapper/rhel_rhel9exam01-swap
```

### Partition Table Types

```
MBR (Master Boot Record) - Legacy:
├─ Max 4 primary partitions
├─ Or 3 primary + 1 extended (with logical partitions)
├─ Max disk size: 2TB
└─ Used with BIOS boot

GPT (GUID Partition Table) - Modern:
├─ 128+ partitions (standard)
├─ Max disk size: 9.4 ZB (zettabytes)
├─ Redundant partition table (backup at end)
├─ Partition UUIDs
└─ Used with UEFI boot

Check partition table type:
sudo gdisk -l /dev/sda
sudo parted /dev/sda print
```

---

## 3.2 Filesystem Types

### Common Filesystems on RHEL 9

```
XFS (Default on RHEL 9):
├─ Excellent performance with large files
├─ Online resizing (grow only, cannot shrink!)
├─ Good for databases, video files
├─ Maximum filesystem: 8 EB
└─ Default for /, /home, /boot on RHEL 9

ext4 (Previous default):
├─ Mature, stable filesystem
├─ Can shrink and grow
├─ Good all-around performance
└─ Maximum filesystem: 1 EB

vfat/FAT32:
├─ Windows-compatible
├─ Required for /boot/efi (UEFI)
├─ No permissions, no symlinks
└─ Max file size: 4GB

swap:
├─ Not a filesystem (raw swap space)
├─ Virtual memory overflow
└─ Cannot mount or browse

btrfs (Optional):
├─ Advanced features (snapshots, compression)
├─ Copy-on-write
└─ Not default on RHEL

tmpfs:
├─ Temporary filesystem in RAM
├─ Lost on reboot
└─ Used for /tmp, /run, /dev/shm
```

### Filesystem Commands

```bash
# Check filesystem type
df -T /
lsblk -f
file -s /dev/sda1

# Create filesystems
sudo mkfs.xfs /dev/sdb1          # XFS
sudo mkfs.ext4 /dev/sdb1         # ext4
sudo mkfs.vfat /dev/sdb1         # FAT32
sudo mkswap /dev/sdb1            # Swap

# Filesystem check and repair
sudo xfs_repair /dev/sdb1        # XFS (must be unmounted!)
sudo fsck.ext4 /dev/sdb1         # ext4 (must be unmounted!)

# Filesystem info
sudo xfs_info /dev/sdb1          # XFS details
sudo tune2fs -l /dev/sdb1        # ext4 details

# Resize filesystems
sudo xfs_growfs /mount_point     # Grow XFS (online)
sudo resize2fs /dev/sdb1         # Resize ext4
```

---

## 3.3 Mount Points & /etc/fstab

### Understanding Mount Points

```
Mount point = Directory where filesystem is attached

Before mounting:
/
├─ boot/  (empty directory on root filesystem)
└─ home/  (empty directory on root filesystem)

After mounting:
/
├─ boot/  (sda2 contents "cover" this directory)
│  └─ vmlinuz-*, initramfs-*  (from sda2)
└─ home/  (home LV contents "cover" this directory)
   └─ kanishka/  (from home LV)

The original empty directories are "hidden" by mounts!
```

### /etc/fstab Format

```
<device>  <mount point>  <type>  <options>  <dump>  <pass>

Example from your system:
UUID=ff3ea389-5a1b-4cd6-9b65-1944fcf6b766  /      xfs   defaults  0  0
UUID=d9f4cbe5-2da2-4810-bf84-d27da0deb3ea  /home  xfs   defaults  0  0
UUID=86d53e6a-5d74-474a-baca-cac6cdd87ec8  /boot  xfs   defaults  0  0
UUID=BFA7-875B                              /boot/efi  vfat  umask=0077  0  2
UUID=7ead638e-e8cc-44f7-b309-2c9145f1322c  none   swap  defaults  0  0

Columns explained:
1. Device: UUID, LABEL, or /dev/path (UUID preferred!)
2. Mount point: Where to attach (/ for root, /home, etc.)
3. Filesystem type: xfs, ext4, vfat, swap, etc.
4. Options: defaults, ro, noexec, etc.
5. Dump: 0=don't backup, 1=backup (mostly unused)
6. Pass: fsck order (0=skip, 1=root, 2=others)
```

### Common Mount Options

```bash
defaults        # rw, suid, dev, exec, auto, nouser, async
ro              # Read-only
rw              # Read-write
noexec          # Don't allow executing binaries
nosuid          # Ignore SUID bits
nodev           # Don't interpret device files
noauto          # Don't mount at boot (manual only)
user            # Allow normal users to mount
_netdev         # Network device (wait for network)
nofail          # Don't fail boot if device missing

Examples:
# Secure /tmp
tmpfs  /tmp  tmpfs  defaults,noexec,nosuid,nodev  0  0

# Read-only backup
UUID=...  /backup  ext4  ro,noauto  0  0

# User-mountable USB
/dev/sdb1  /mnt/usb  vfat  noauto,user,rw  0  0
```

### Mount Commands

```bash
# Mount manually
sudo mount /dev/sdb1 /mnt/usb

# Mount with options
sudo mount -o ro,noexec /dev/sdb1 /mnt/usb

# Mount by UUID
sudo mount UUID=abc-123 /mnt/usb

# Mount all from /etc/fstab
sudo mount -a

# Unmount
sudo umount /mnt/usb
sudo umount /dev/sdb1

# Force unmount (if busy)
sudo umount -f /mnt/usb
sudo fuser -km /mnt/usb  # Kill processes using mount point
sudo umount /mnt/usb

# Remount with new options
sudo mount -o remount,ro /

# View all mounts
mount
mount | grep "^/dev"
findmnt  # Tree view (prettier!)

# Check what's using a mount point
lsof +D /mnt/usb
fuser -v /mnt/usb
```

### Testing /etc/fstab Changes

```bash
# CRITICAL: Always test before rebooting!

# 1. Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# 2. Edit fstab
sudo vim /etc/fstab

# 3. Verify syntax
sudo findmnt --verify

# 4. Test mount all
sudo mount -a

# 5. Check if successful
df -h
mount | grep "^/dev"

# 6. If OK, reboot
sudo reboot

# If errors:
# - Fix /etc/fstab
# - Retest with mount -a
# - Don't reboot until mount -a succeeds!
```

---

## 3.4 LVM (Logical Volume Manager)

### LVM Architecture

```
Physical Disk (e.g., /dev/sda3)
    ↓
Physical Volume (PV)
    ├─ Lowest layer
    └─ Raw partition marked for LVM use
    ↓
Volume Group (VG)
    ├─ Pool of storage from one or more PVs
    └─ Example: rhel_rhel9exam01
    ↓
Logical Volumes (LVs)
    ├─ Virtual partitions from VG
    ├─ Example: root, swap, home
    └─ Can be resized online!
```

### LVM Components

```
Your System:

Physical Volume (PV):
/dev/sda3  (58.4GB)

Volume Group (VG):
rhel_rhel9exam01  (58.4GB from sda3)

Logical Volumes (LVs):
├─ rhel_rhel9exam01-root  (37GB)  → /
├─ rhel_rhel9exam01-swap  (3.9GB) → [SWAP]
└─ rhel_rhel9exam01-home  (18GB)  → /home

Mapped Devices:
/dev/dm-0  = /dev/mapper/rhel_rhel9exam01-root
/dev/dm-1  = /dev/mapper/rhel_rhel9exam01-swap
/dev/dm-2  = /dev/mapper/rhel_rhel9exam01-home
```

### LVM Commands

```bash
# Physical Volumes
sudo pvs                   # Quick view
sudo pvdisplay             # Detailed view
sudo pvscan                # Scan for PVs
sudo pvcreate /dev/sdb1    # Create PV

# Volume Groups
sudo vgs                   # Quick view
sudo vgdisplay             # Detailed view
sudo vgscan                # Scan for VGs
sudo vgcreate vg_name /dev/sdb1     # Create VG
sudo vgextend vg_name /dev/sdc1     # Add PV to VG

# Logical Volumes
sudo lvs                   # Quick view
sudo lvdisplay             # Detailed view
sudo lvscan                # Scan for LVs
sudo lvcreate -L 10G -n lv_name vg_name     # Create 10GB LV
sudo lvcreate -l 100%FREE -n lv_name vg_name # Use all free space

# Extend Logical Volume
sudo lvextend -L +5G /dev/vg_name/lv_name       # Add 5GB
sudo lvextend -l +100%FREE /dev/vg_name/lv_name # Use all free
sudo lvextend -L 20G /dev/vg_name/lv_name       # Resize to 20GB

# Resize filesystem after extending LV
sudo xfs_growfs /mount_point        # For XFS (online)
sudo resize2fs /dev/vg_name/lv_name # For ext4 (online)

# Remove LV (DANGEROUS!)
sudo lvremove /dev/vg_name/lv_name

# Activate/Deactivate
sudo vgchange -ay          # Activate all VGs
sudo vgchange -an          # Deactivate all VGs
```

### LVM Snapshots (Backup)

```bash
# Create snapshot
sudo lvcreate -L 5G -s -n root_snapshot /dev/rhel_rhel9exam01/root

# Mount snapshot
sudo mkdir /mnt/snapshot
sudo mount /dev/rhel_rhel9exam01/root_snapshot /mnt/snapshot

# Backup from snapshot
sudo tar czf /backup/root_backup.tar.gz -C /mnt/snapshot .

# Remove snapshot
sudo umount /mnt/snapshot
sudo lvremove /dev/rhel_rhel9exam01/root_snapshot
```

---

## 3.5 Disk Usage Analysis

### Finding Large Files/Directories

```bash
# Top-level directory sizes
du -sh /* 2>/dev/null | sort -rh | head -10

# Current directory breakdown
du -sh * | sort -rh

# Specific depth
du -h --max-depth=1 /var | sort -rh

# Find large files
find / -type f -size +100M 2>/dev/null
find /var/log -type f -size +100M
find /home -type f -size +1G -exec ls -lh {} \;

# Top 20 largest files
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Files modified in last 7 days
find /var/log -type f -mtime -7 -exec ls -lh {} \;

# Old files (older than 30 days)
find /tmp -type f -mtime +30

# Disk usage by user
sudo du -sh /home/* 2>/dev/null
```

### Inode Usage

```bash
# Check inode usage (can run out even with free space!)
df -i

# Find directories with most files
find /path -xdev -type f | cut -d "/" -f 2 | sort | uniq -c | sort -rn

# Count files in directory
find /var/log -type f | wc -l
```

---

# 4. Memory & Swap Management

## 4.1 Understanding Linux Memory

### Memory Hierarchy

```
CPU Registers
    ↓ (nanoseconds)
L1/L2/L3 Cache
    ↓ (nanoseconds)
RAM (Physical Memory)
    ↓ (nanoseconds - 100ns)
Swap (Disk)
    ↓ (milliseconds - 10ms on SSD, 50ms on HDD)
    100x - 1000x slower than RAM!
```

### Memory Types

```
Physical RAM (7.5GB on your system):
├─ Fast (nanoseconds)
├─ Limited size
├─ Volatile (lost on power off)
└─ Expensive

Swap Space (3.9GB on your system):
├─ Slow (milliseconds)
├─ Larger, flexible size
├─ Persistent (survives reboot)
├─ Cheap (uses disk space)
└─ NOT extra RAM! (overflow storage)
```

---

## 4.2 Memory Statistics

### Using free Command

```bash
free -h

#               total        used        free      shared  buff/cache   available
# Mem:           7.5Gi       4.7Gi       672Mi        21Mi       2.5Gi       2.8Gi
# Swap:          3.9Gi       1.5Gi       2.5Gi

Columns explained:
├─ total: Total physical RAM
├─ used: RAM used by processes and kernel
├─ free: Completely unused RAM
├─ shared: RAM used by tmpfs (shared memory)
├─ buff/cache: RAM used for disk caching (can be freed!)
└─ available: RAM available for new apps (free + reclaimable cache)

Important: Look at "available", not "free"!
Linux uses "free" RAM for caching to improve performance.
It will automatically free cache when applications need RAM.

Swap row:
├─ total: Total swap space
├─ used: Currently swapped out
└─ free: Available swap

Options:
-h  Human-readable (GB, MB)
-w  Wide format (separates buff and cache)
-s  Continuous updates (free -s 2 = update every 2 seconds)
```

### Understanding Memory Fields

```
Real Example:
Mem:  7.5G total, 4.7G used, 672M free, 2.5G buff/cache, 2.8G available

Analysis:
✅ Available: 2.8G - What actually matters!
   (672M free + ~2.1G reclaimable cache)

⚠️ Free: 672M - Doesn't tell full story
   (Linux uses "free" RAM for caching)

✅ Buff/Cache: 2.5G - Can be freed if needed
   (Disk cache for performance)

Scenarios:
1. You start Chrome (needs 1GB):
   - Available: 2.8G (enough!)
   - Kernel frees 400M from cache
   - Chrome starts successfully ✓

2. If Available < 500M:
   - System under memory pressure
   - May start swapping
   - Performance degradation
```

---

## 4.3 Swap Space

### What is Swap?

```
Swap = Disk space used as overflow when RAM is full

NOT Extra RAM:
❌ Swap doesn't increase RAM size
❌ Swap doesn't make RAM faster
✅ Swap is emergency overflow storage
✅ Swap allows system to survive memory pressure

Located:
/dev/mapper/rhel_rhel9exam01-swap (3.9GB partition on disk)

Purpose:
1. Handle RAM overflow (prevent OOM killer)
2. Hibernate/suspend to disk (needs swap >= RAM)
3. Better memory management (inactive pages → swap)
4. Emergency safety net (time to react to memory leaks)
```

### Swap Types

```
Swap Partition (Your System):
✅ Dedicated partition
✅ Slightly faster (contiguous)
✅ Fixed size
✅ Better for servers
✅ Required for reliable hibernate
❌ Can't easily resize

Swap File (Alternative):
✅ Regular file on filesystem
✅ Easy to create/resize/delete
✅ Flexible
✅ Good for desktops
⚠️ Slightly slower
⚠️ Not ideal for hibernate
```

### Swap Commands

```bash
# View swap status
swapon --show
# NAME      TYPE      SIZE USED PRIO
# /dev/dm-1 partition 3.9G 1.5G   -2

free -h
# Swap:          3.9Gi       1.5Gi       2.5Gi

cat /proc/swaps
# Filename                Type        Size    Used    Priority
# /dev/dm-1               partition   4096000 1567744 -2

grep -i swap /proc/meminfo
# SwapTotal:       4096000 kB
# SwapFree:        2528256 kB
# SwapCached:       123456 kB

# Enable/Disable swap
sudo swapon -a              # Enable all (from /etc/fstab)
sudo swapon /dev/dm-1       # Enable specific
sudo swapoff -a             # Disable all
sudo swapoff /dev/dm-1      # Disable specific

# Clear swap (forces swapped data back to RAM)
# WARNING: Only if you have enough free RAM!
sudo swapoff -a
sudo swapon -a

# Create swap file (alternative to partition)
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048  # 2GB
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab

# Remove swap file
sudo swapoff /swapfile
sudo rm /swapfile
# Remove line from /etc/fstab
```

### Swap Size Recommendations

```
Modern Guidelines:

RAM Size    Without Hibernate    With Hibernate
========    =================    ==============
< 2GB       2x RAM               3x RAM
2-8GB       = RAM                2x RAM
8-64GB      4-8GB                1.5x RAM
> 64GB      4GB minimum          Not recommended

Your System:
8GB RAM → 3.9GB swap
✅ Good for general use without hibernate
✅ Enough for emergency overflow
❌ Too small for hibernate (would need 8GB+)
```

---

## 4.4 Swappiness

### What is Swappiness?

```
Swappiness = Controls how aggressively Linux uses swap

Value: 0-100

0   ╠═══════════════════════════════════════════════╣ 100
    ↑                       ↑                        ↑
  Avoid swap           Balanced             Aggressive swap
  (use only           (default              (swap even with
   when RAM             60)                  free RAM)
   critical)

Scale:
0   = Only swap when RAM critically low (>95% used)
10  = Minimal swapping (good for databases)
30  = Conservative (good for servers)
60  = Balanced (default, general use)
80  = Aggressive swapping
100 = Swap very aggressively
```

### Swappiness Commands

```bash
# Check current swappiness
cat /proc/sys/vm/swappiness
# 60  (default on most systems)
# 30  (default on some RHEL configurations)

# Change temporarily (until reboot)
sudo sysctl vm.swappiness=10

# Verify change
cat /proc/sys/vm/swappiness
# 10

# Change permanently
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Apply immediately
sudo sysctl -p

# Or edit directly
sudo vim /etc/sysctl.conf
# Add line: vm.swappiness=10
sudo sysctl -p
```

### Swappiness Recommendations

```
Use Case                          Recommended Swappiness
==========================================
Desktop/Laptop                    60 (default)
Database server (MySQL, PostgreSQL)  10-20
Web server (Apache, Nginx)        30
File server                       30-40
Memory-constrained system         60-80
Performance-critical app          0-10
General server                    30

Your RHEL9 System:
Current: 60 (balanced)
Recommended: 10-30 (prefer RAM for learning/development)
```

---

## 4.5 Memory Monitoring

### Virtual Memory Statistics (vmstat)

```bash
vmstat 2 5
# Update every 2 seconds, 5 samples

# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  0  0 1567744 688868   3444 2600240    0    0     0     0  256  423  0  0 100  0  0

Columns explained:

Procs:
  r = Processes waiting for CPU (run queue)
      Good: 0-2
      Warning: > CPU count
  b = Processes blocked on I/O
      Good: 0
      Warning: > 5

Memory (in KB):
  swpd = Amount in swap
  free = Free RAM
  buff = Buffer cache
  cache = Page cache

Swap (KB/sec):
  si = Swap In (reading from swap) - SHOULD BE 0!
  so = Swap Out (writing to swap) - SHOULD BE 0!

I/O (blocks/sec):
  bi = Blocks in (disk reads)
  bo = Blocks out (disk writes)

System:
  in = Interrupts per second
  cs = Context switches per second

CPU (%):
  us = User time
  sy = System time
  id = Idle (should be high when not busy)
  wa = I/O wait (should be < 10%)
  st = Stolen (VM only - hypervisor overhead)

Health Indicators:
✅ si=0, so=0 → No active swapping (GOOD!)
⚠️ si>0, so>0 → System is swapping (SLOW!)
⚠️ wa>10% → Disk bottleneck
⚠️ r > CPU count → CPU overloaded
```

### Process Memory Usage

```bash
# Top memory consumers
ps aux --sort=-%mem | head -10

# With headers
ps aux --sort=-%mem | head -11

# Specific user
ps aux --sort=-%mem | grep username | head -10

# Which processes are using swap
for file in /proc/*/status; do 
    awk '/VmSwap|Name/{printf $2 " " $3}END{print ""}' "$file"
done | sort -k 2 -n -r | head -10

# Memory map of a process
pmap -x PID

# Detailed process memory
cat /proc/PID/status | grep -i vm
```

### Interactive Monitoring

```bash
# Top (interactive)
top
# Press 'M' to sort by memory
# Press 'P' to sort by CPU
# Press 'k' to kill process
# Press 'q' to quit

# Htop (if installed - better than top)
htop

# Watch memory continuously
watch -n 2 free -h

# Watch specific process
watch -n 1 'ps aux | grep firefox'
```

---

## 4.6 Memory Pressure & OOM

### Out of Memory (OOM) Killer

```
When RAM + Swap are exhausted:

1. System enters OOM condition
2. Kernel invokes OOM killer
3. Selects process to kill (based on score)
4. Kills process to free memory
5. System logs event to /var/log/messages

Check OOM events:
grep -i "out of memory" /var/log/messages
grep -i "killed process" /var/log/messages
dmesg | grep -i "out of memory"

OOM Score (which process to kill):
cat /proc/PID/oom_score
# Higher score = more likely to be killed

Adjust OOM score:
echo -1000 > /proc/PID/oom_score_adj  # Never kill
echo 1000 > /proc/PID/oom_score_adj   # Kill first
```

### Memory Pressure

```bash
# Check memory pressure (RHEL 8+)
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

Interpretation:
avg10  = Average pressure over 10 seconds
avg60  = Average pressure over 60 seconds
avg300 = Average pressure over 300 seconds

0.00 = No pressure ✅
> 1.00 = Moderate pressure ⚠️
> 10.00 = High pressure ❌

# Memory available
cat /proc/meminfo | grep -i available
MemAvailable:     2867200 kB

# Cache that can be freed
cat /proc/meminfo | grep -E "^Cached|^Buffers"
```

### Clearing Caches (Emergency)

```bash
# Drop caches (safe - only drops clean cache)
# Always sync first!
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches

Cache types:
echo 1 = Free page cache
echo 2 = Free dentries and inodes
echo 3 = Free page cache + dentries + inodes

WARNING: Only use if system is having memory issues!
Normal operation: Let Linux manage cache automatically.
```

---

# 5. Package Management (RPM/DNF)

## 5.1 RPM (Red Hat Package Manager)

### What is RPM?

```
RPM = Package file format + low-level package manager

Package file:
nginx-1.20.1-14.el9.x86_64.rpm
│     │       │    │    └─ Architecture
│     │       │    └──── Distribution (el9 = Enterprise Linux 9)
│     │       └───────── Release number
│     └───────────────── Version
└─────────────────────── Package name

RPM Features:
✅ Install/remove packages
✅ Query package information
✅ Verify package integrity
❌ Does NOT resolve dependencies
❌ Does NOT download packages
```

### RPM Query Commands

```bash
# List all installed packages
rpm -qa
rpm -qa | wc -l              # Count packages
rpm -qa | sort               # Sorted list

# Search for package
rpm -qa | grep kernel
rpm -qa | grep -i python

# Package information
rpm -qi package_name         # Detailed info
rpm -qi bash                 # Info about bash package

# List files in package
rpm -ql package_name         # All files
rpm -ql coreutils            # Files in coreutils
rpm -ql coreutils | wc -l    # Count files

# Which package owns a file?
rpm -qf /usr/bin/ls          # Finds: coreutils
rpm -qf /etc/passwd          # Finds: setup
rpm -qf $(which python3)     # Which package provides python3

# Package dependencies
rpm -qR package_name         # What does this package require?
rpm -q --requires bash

# What provides a capability?
rpm -q --whatprovides /bin/sh

# Configuration files
rpm -qc package_name         # List config files
rpm -qc openssh-server

# Documentation
rpm -qd package_name         # List documentation
rpm -qd bash

# Scripts (pre/post install)
rpm -q --scripts package_name

# Verify package integrity
rpm -V package_name          # Check if files modified
rpm -Va                      # Verify all packages
```

### RPM Install/Remove (Low-Level)

```bash
# Install package (manual, no dependencies!)
sudo rpm -ivh package.rpm
# -i = install
# -v = verbose
# -h = hash marks (progress)

# Upgrade package
sudo rpm -Uvh package.rpm
# -U = upgrade (or install if not present)

# Freshen (only upgrade if already installed)
sudo rpm -Fvh package.rpm

# Remove package
sudo rpm -e package_name
# -e = erase

# Force install (dangerous!)
sudo rpm -ivh --force package.rpm
sudo rpm -ivh --nodeps package.rpm  # Ignore dependencies

# Query package file (not installed)
rpm -qip package.rpm         # Info
rpm -qlp package.rpm         # List files
```

### RPM Database

```bash
# RPM database location
ls /var/lib/rpm/

# Rebuild database (if corrupted)
sudo rpm --rebuilddb

# Import GPG key
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# Verify signature
rpm -K package.rpm
rpm --checksig package.rpm
```

---

## 5.2 DNF (Dandified YUM)

### DNF vs YUM vs RPM

```
RPM (Low-level):
├─ Installs local .rpm files
├─ No dependency resolution
├─ No automatic downloads
└─ Use for: queries, verification

YUM (Deprecated):
├─ High-level package manager
├─ Automatic dependency resolution
├─ Downloads from repositories
└─ Replaced by DNF in RHEL 8+

DNF (Modern):
├─ YUM replacement (faster, better)
├─ Automatic dependency resolution
├─ Downloads from repositories
├─ Better performance
└─ Use for: normal package management

Workflow:
User wants nginx
    ↓
DNF: Search repos → Find nginx + dependencies → Download all
    ↓
DNF: Install using RPM backend
    ↓
RPM: Actually install packages to system
```

### DNF Configuration

```bash
# Main config
/etc/dnf/dnf.conf

Common settings:
[main]
gpgcheck=1                    # Verify package signatures
installonly_limit=3           # Keep 3 kernel versions
clean_requirements_on_remove=True
best=True                     # Install best version available
skip_if_unavailable=True

# Repository definitions
/etc/yum.repos.d/

# Example repo file:
[baseos]
name=Red Hat Enterprise Linux 9 - BaseOS
baseurl=https://cdn.redhat.com/...
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

### DNF Repository Management

```bash
# List repositories
dnf repolist                 # Enabled repos
dnf repolist all             # All repos (enabled + disabled)
dnf repolist --enabled
dnf repolist --disabled

# Repository info
dnf repoinfo
dnf repoinfo baseos

# Enable/disable repository
sudo dnf config-manager --enable repo-name
sudo dnf config-manager --disable repo-name

# Add repository
sudo dnf config-manager --add-repo https://example.com/repo

# Clean cache
sudo dnf clean all           # Remove all cached data
sudo dnf clean packages      # Remove cached packages
sudo dnf clean metadata      # Remove cached metadata

# Make cache
sudo dnf makecache           # Download fresh metadata
sudo dnf makecache --refresh # Force refresh
```

### DNF Package Operations

```bash
# Search for packages
dnf search keyword           # Search package names/summaries
dnf search web server
dnf search all keyword       # Search in descriptions too

# Package information
dnf info package_name        # From repository
dnf info nginx

# List packages
dnf list                     # All packages
dnf list installed           # Installed packages
dnf list available           # Available to install
dnf list updates             # Available updates
dnf list kernel              # Specific package

# Install packages
sudo dnf install package_name
sudo dnf install nginx mysql vim
sudo dnf install -y package  # Assume yes (no prompts)

# Install local RPM (with dependency resolution!)
sudo dnf install ./package.rpm
sudo dnf install /path/to/package.rpm

# Install from URL
sudo dnf install https://example.com/package.rpm

# Reinstall package
sudo dnf reinstall package_name

# Downgrade package
sudo dnf downgrade package_name

# Remove packages
sudo dnf remove package_name
sudo dnf remove nginx mysql

# Remove with dependencies
sudo dnf autoremove package_name

# Remove orphaned packages
sudo dnf autoremove

# Update packages
sudo dnf check-update        # Check for updates (don't install)
sudo dnf update              # Update all packages
sudo dnf update package_name # Update specific package
sudo dnf upgrade             # Same as update (in DNF)

# Update security patches only
sudo dnf update --security
sudo dnf update-minimal --security

# Group operations
dnf group list               # List package groups
dnf group info "Group Name"  # Group details
sudo dnf group install "Development Tools"
sudo dnf group remove "Development Tools"
```

### DNF History & Rollback

```bash
# View transaction history
dnf history                  # All transactions
dnf history list
dnf history list kernel      # Transactions involving kernel

# Transaction details
dnf history info 5           # Details of transaction 5
dnf history info last        # Last transaction

# Undo transaction
sudo dnf history undo last   # Undo last transaction
sudo dnf history undo 5      # Undo transaction 5

# Redo transaction
sudo dnf history redo 5

# Rollback to transaction
sudo dnf history rollback 5  # Revert to state at transaction 5

# Clear history
sudo dnf history clear
```

### DNF Advanced

```bash
# Dependencies
dnf deplist package_name     # Show dependencies
dnf repoquery --requires nginx

# What provides?
dnf provides /usr/bin/python3
dnf provides */semanage

# Download only (don't install)
sudo dnf download package_name
sudo dnf download --resolve package_name  # With dependencies
sudo dnf download --resolve --destdir=/tmp package_name

# Show what would be installed (dry-run)
dnf install --assumeno package_name

# List duplicate packages
dnf list --duplicates

# List extras (not in any repo)
dnf list --extras

# Check for problems
sudo dnf check

# Verbose output
sudo dnf install -v package_name

# Debug level
sudo dnf install -d 10 package_name
```

### Kernel Management

```bash
# List installed kernels
rpm -qa | grep kernel
dnf list installed kernel

# List kernels in /boot
ls /boot/vmlinuz-*

# Install new kernel
sudo dnf install kernel

# Remove old kernels (keep 3 most recent)
sudo dnf remove --oldinstallonly --setopt installonly_limit=3 kernel

# Check kernel version
uname -r                     # Running kernel
grubby --default-kernel      # Default boot kernel

# Set default kernel
sudo grubby --set-default /boot/vmlinuz-5.14.0-611.24.1.el9_7.x86_64

# View all kernels in GRUB
sudo grubby --info=ALL
```

---

# 6. Symbolic Links & Mount Points

## 6.1 Symbolic Links (Symlinks)

### What are Symbolic Links?

```
Symbolic Link = Pointer to another file or directory

Similar to:
├─ Windows: Shortcut
└─ macOS: Alias

Purpose:
├─ Create shortcuts
├─ Maintain backward compatibility
├─ Simplify complex paths
└─ Manage multiple versions
```

### Creating Symbolic Links

```bash
# Create symlink to file
ln -s /path/to/original /path/to/link

# Example:
ln -s /usr/bin/python3.9 /usr/bin/python3

# Create symlink to directory
ln -s /opt/app-v2.0 /opt/app

# Force overwrite existing link
ln -sf /new/target /existing/link

# Relative symlink (relative to link location)
ln -s ../dir/file link_name

# Absolute symlink (full path)
ln -s /absolute/path/to/file link_name
```

### Working with Symbolic Links

```bash
# Identify symlink
ls -l /bin
# lrwxrwxrwx. 1 root root 7 ... /bin -> usr/bin
#  ↑ = symlink

file /bin
# /bin: symbolic link to usr/bin

# Show target without following
readlink /bin
# usr/bin

# Show full resolved path
readlink -f /bin/ls
# /usr/bin/ls

# Check if path is symlink
[ -L /bin ] && echo "Symlink" || echo "Not symlink"

# Find all symlinks in directory
find /usr/bin -type l
ls -l /usr/bin | grep "^l"

# Find broken symlinks
find /usr/bin -type l ! -exec test -e {} \; -print

# Update symlink target
ln -sf /new/target /existing/link

# Remove symlink
rm /path/to/link              # Remove link only
unlink /path/to/link          # Alternative
# Note: rm link/ (with slash) removes target!
#       rm link (no slash) removes symlink ✓
```

### Common System Symlinks (RHEL 9)

```bash
/bin → usr/bin
/sbin → usr/sbin
/lib → usr/lib
/lib64 → usr/lib64

# Application version management
/usr/bin/python3 → python3.9
/usr/bin/python → python3
/usr/bin/java → /opt/java-17/bin/java

# Check all root-level symlinks
ls -la / | grep "^l"
```

---

## 6.2 Hard Links

### Hard Links vs Symbolic Links

```
Symbolic Link (Soft Link):
├─ Points to filename
├─ Can cross filesystems
├─ Can link to directories
├─ Breaks if target deleted
├─ Small size (just stores path)
└─ Shows with ls -l as: link → target

Hard Link:
├─ Points to inode (same as original)
├─ Cannot cross filesystems
├─ Cannot link directories (usually)
├─ Survives if "original" deleted
├─ Same size as original
└─ Both files are "equal" (no original vs copy)
```

### Hard Link Commands

```bash
# Create hard link
ln /path/to/file /path/to/hardlink

# Example:
ln /home/user/file.txt /tmp/file_link

# Check if files are hard-linked (same inode)
ls -i file1 file2
# 12345678 file1
# 12345678 file2  ← Same inode = hard linked

stat file1
stat file2
# Links: 2  (both will show 2)

# Find all hard links to a file
find / -inum 12345678 2>/dev/null

# Count hard links
ls -l file
# -rw-r--r--. 2 user user ...
#             ↑ = 2 hard links
```

---

## 6.3 Mount Points vs Symlinks

### Key Differences

```
Mount Point:
├─ Attaches filesystem to directory tree
├─ Different storage device
├─ Managed by kernel
├─ Listed in: df, mount, /etc/fstab
├─ Example: /boot → sda2 filesystem

Symbolic Link:
├─ Pointer within same or different filesystem
├─ No device attachment
├─ Just a special file
├─ Listed in: ls -l (shows →)
├─ Example: /bin → usr/bin
```

### Examples from Your System

```bash
# Mount Points (different filesystems)
df -h
# /dev/sda2                  /boot       ← Mount point
# /dev/mapper/...-root       /           ← Mount point
# /dev/mapper/...-home       /home       ← Mount point

# Symbolic Links (pointers)
ls -l / | grep "^l"
# /bin → usr/bin                          ← Symlink
# /lib → usr/lib                          ← Symlink
# /sbin → usr/sbin                        ← Symlink
```

---

# 7. System Monitoring & Performance

## 7.1 System Load & Uptime

```bash
# System uptime and load averages
uptime
# 14:23:45 up 5 days, 3:42, 2 users, load average: 0.15, 0.18, 0.12
#                                                  1min  5min  15min

Load Average Interpretation (for 4-core system):
< 4.0  = Good (not overloaded)
4.0-8.0 = Moderate (getting busy)
> 8.0  = High (overloaded)

Rule: Load < CPU count = healthy
```

## 7.2 Process Monitoring

```bash
# Top (real-time)
top
# Interactive commands:
#   M = Sort by memory
#   P = Sort by CPU  
#   k = Kill process (enter PID)
#   1 = Show individual CPUs
#   q = Quit

# Process snapshot
ps aux                       # All processes
ps aux | grep nginx          # Specific process
ps -ef                       # Alternative format

# Process tree
pstree
pstree -p                    # With PIDs
pstree -p | grep -i gunicorn

# Kill processes
kill PID                     # Graceful
kill -9 PID                  # Force kill
killall process_name         # Kill all by name
pkill process_name           # Kill by pattern

# Process priority (nice)
nice -n 10 command           # Start with lower priority
renice -n 5 -p PID          # Change priority of running process
```

## 7.3 I/O Monitoring

```bash
# I/O statistics
iostat -x 1                  # Extended stats, 1 sec updates
# %util column: < 80% = good, > 80% = bottleneck

# Per-process I/O
iotop                        # Real-time I/O by process
iotop -o                     # Only show active I/O

# Disk activity
pidstat -d 1                 # Per-process disk stats
```

---

# 8. Troubleshooting & Recovery

## 8.1 Boot Issues

### GRUB Rescue

```bash
# If dropped to grub> prompt:

grub> ls                           # List partitions
grub> ls (hd0,gpt2)/              # View partition contents
grub> set root=(hd0,gpt2)         # Set boot partition
grub> linux /vmlinuz-... root=/dev/mapper/... ro
grub> initrd /initramfs-...
grub> boot

# After successful boot:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Emergency Mode Boot

```
At GRUB menu:
1. Press 'e' to edit boot entry
2. Find line starting with 'linux'
3. Add to end: systemd.unit=rescue.target
4. Press Ctrl+X to boot

You'll boot to single-user mode with root access
```

### Reset Root Password

```
At GRUB menu:
1. Press 'e' to edit
2. Find 'linux' line
3. Add: rd.break
4. Press Ctrl+X

At switch_root prompt:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

---

## 8.2 Disk Full Issues

```bash
# Find what's using space
df -h                        # Which partition full?
du -sh /* 2>/dev/null | sort -rh | head -10

# Clean up options
sudo journalctl --vacuum-size=100M
sudo dnf clean all
sudo dnf remove --oldinstallonly --setopt installonly_limit=3 kernel
find /var/log -name "*.log" -mtime +30 -delete
find /tmp -mtime +7 -delete
```

## 8.3 High Memory Usage

```bash
# Find memory hogs
ps aux --sort=-%mem | head -10
top  # Press 'M' to sort by memory

# Clear swap (if safe)
free -h                      # Check available RAM first
sudo swapoff -a && sudo swapon -a

# Drop caches (emergency)
sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

## 8.4 System Logs

```bash
# System logs (journald)
journalctl                   # All logs
journalctl -b                # Current boot
journalctl -b -1             # Previous boot
journalctl -f                # Follow (like tail -f)
journalctl -p err            # Errors only
journalctl -p err -n 50      # Last 50 errors
journalctl -u sshd           # Specific service
journalctl --since "1 hour ago"
journalctl --since "2024-01-01" --until "2024-01-31"

# Legacy logs
tail -f /var/log/messages    # General system
tail -f /var/log/secure      # Authentication
grep -i error /var/log/messages

# Kernel logs
dmesg                        # Kernel ring buffer
dmesg | grep -i error
dmesg -T                     # Human-readable timestamps
```

---

# 9. Hands-On Lab Exercises

## Exercise 1: Explore Boot Process

**Objective:** Understand your system's boot configuration

```bash
# 1. View your system's boot layout
lsblk -f

# 2. Check EFI boot partition contents
ls -la /boot/efi/EFI/

# 3. View boot partition
ls -lh /boot/

# 4. Count installed kernels
ls /boot/vmlinuz-* | wc -l

# 5. Check current kernel
uname -r

# 6. View GRUB configuration
sudo cat /etc/default/grub

# 7. List GRUB menu entries
sudo awk -F\' '/menuentry / {print $2}' /boot/grub2/grub.cfg

# 8. Check default kernel
sudo grub2-editenv list

# Questions to answer:
# - How many kernels are installed?
# - Which kernel is default?
# - What's the GRUB timeout?
# - Is your system UEFI or BIOS?
```

---

## Exercise 2: Disk & Storage Analysis

**Objective:** Analyze disk usage and layout

```bash
# 1. Complete disk overview
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID

# 2. Check disk space
df -hT

# 3. Find top 10 space consumers
sudo du -sh /* 2>/dev/null | sort -rh | head -10

# 4. Check inode usage
df -i

# 5. View mount points
mount | grep "^/dev"
findmnt

# 6. Check /etc/fstab
cat /etc/fstab

# 7. Verify fstab syntax
sudo findmnt --verify

# 8. LVM information
sudo pvs
sudo vgs  
sudo lvs

# Questions:
# - What's your total disk size?
# - Which partition uses most space?
# - How much swap do you have?
# - What filesystem types are in use?
# - Document your complete partition layout
```

---

## Exercise 3: Memory & Swap Investigation

**Objective:** Monitor memory and swap usage

```bash
# 1. Memory overview
free -h

# 2. Swap status
swapon --show
cat /proc/swaps

# 3. Check swappiness
cat /proc/sys/vm/swappiness

# 4. Monitor swap activity (run for 30 seconds)
vmstat 2 15

# 5. Find top memory users
ps aux --sort=-%mem | head -10

# 6. Check if anything is in swap
for f in /proc/*/status; do 
    awk '/VmSwap|Name/{printf $2" "$3}END{print""}' "$f"
done | sort -k2 -rn | head -10

# 7. Memory details
cat /proc/meminfo | grep -i mem
cat /proc/meminfo | grep -i swap

# Questions:
# - How much RAM does your system have?
# - How much swap is currently used?
# - Are si/so values in vmstat zero? (they should be!)
# - What's using the most memory?
# - Is your swappiness value appropriate for your use case?
```

---

## Exercise 4: Symbolic Links Practice

**Objective:** Create and manage symbolic links

```bash
# 1. Create test directory
mkdir -p ~/symlink_test
cd ~/symlink_test

# 2. Create test file
echo "Original content" > original.txt

# 3. Create symbolic link
ln -s original.txt link.txt

# 4. Verify symlink
ls -l
file link.txt
readlink link.txt

# 5. Read through symlink
cat link.txt

# 6. Write through symlink
echo "Added via symlink" >> link.txt
cat original.txt  # Should see both lines

# 7. Delete original
rm original.txt

# 8. Try to read symlink (should fail)
cat link.txt
ls -l link.txt  # Broken link

# 9. Check system symlinks
ls -l / | grep "^l"
readlink /bin

# 10. Create directory symlink
mkdir data
ln -s data shortcut
cd shortcut
pwd       # Shows symlink path
pwd -P    # Shows real path

# Cleanup
cd ~
rm -rf ~/symlink_test
```

---

## Exercise 5: Package Management

**Objective:** Practice RPM and DNF commands

```bash
# 1. Count installed packages
rpm -qa | wc -l

# 2. Find package that provides 'ls' command
rpm -qf $(which ls)

# 3. List files in coreutils package
rpm -ql coreutils | head -20

# 4. Search for python packages
rpm -qa | grep python

# 5. Get info about bash package
rpm -qi bash

# 6. Check for updates
dnf check-update | head -20

# 7. Search for web server
dnf search web server

# 8. Get info about package (without installing)
dnf info nginx

# 9. View DNF history
dnf history | head -10

# 10. List installed kernels
rpm -qa | grep kernel
dnf list installed kernel

# Questions:
# - How many packages are installed?
# - Which package provides /bin/bash?
# - Are there any updates available?
# - How many kernels are installed?
```

---

## Exercise 6: System Monitoring

**Objective:** Monitor system performance

```bash
# 1. Check system load
uptime

# 2. View processes (run for 30 seconds, then q)
top
# Press 'M' for memory sort
# Press 'P' for CPU sort
# Press 'q' to quit

# 3. All processes snapshot
ps aux | head -20

# 4. Find nginx processes (if running)
ps aux | grep nginx

# 5. Virtual memory statistics
vmstat 2 10

# 6. I/O statistics
iostat -x 2 5

# 7. Memory pressure
cat /proc/pressure/memory

# 8. Check last boot time
uptime -s
who -b

# Questions:
# - What's your system load average?
# - How many processes are running?
# - Is CPU mostly idle or busy?
# - Is there any I/O wait?
```

---

## Exercise 7: LVM Management

**Objective:** Understand LVM structure on your system

```bash
# 1. Physical Volumes
sudo pvdisplay
sudo pvs

# 2. Volume Groups
sudo vgdisplay
sudo vgs

# 3. Logical Volumes
sudo lvdisplay
sudo lvs

# 4. Check free space in VG
sudo vgs
# Look at VFree column

# 5. Detailed LV info
sudo lvdisplay /dev/rhel_rhel9exam01/root

# 6. See device mapper devices
ls -l /dev/mapper/
ls -l /dev/rhel_rhel9exam01/

# 7. Check LV usage
df -h /
df -h /home

# Questions to document:
# - What PV(s) do you have?
# - What's your VG name?
# - How many LVs?
# - How much free space in VG?
# - Can you extend /home if needed?
```

---

## Exercise 8: Filesystem Deep Dive

**Objective:** Explore /etc configuration directory

```bash
# 1. Count files in /etc
find /etc -type f | wc -l

# 2. Find largest config files
du -sh /etc/* 2>/dev/null | sort -rh | head -10

# 3. View critical configs
cat /etc/hostname
cat /etc/os-release
cat /etc/fstab

# 4. User/group info
wc -l /etc/passwd
wc -l /etc/group
grep $(whoami) /etc/passwd

# 5. Find all .conf files
find /etc -name "*.conf" -type f 2>/dev/null | wc -l

# 6. Recently modified configs
find /etc -type f -mtime -7 2>/dev/null | head -20

# 7. Network configuration
cat /etc/hosts
cat /etc/resolv.conf

# Questions:
# - How many files in /etc?
# - What's your system hostname?
# - How many users in /etc/passwd?
# - What are your DNS servers?
```

---

## Exercise 9: Complete System Health Check

**Objective:** Create comprehensive health report

```bash
# Create script
cat > ~/system_health.sh << 'EOF'
#!/bin/bash
echo "========================================="
echo "System Health Report"
echo "Date: $(date)"
echo "Hostname: $(hostname)"
echo "========================================="
echo ""

echo "=== SYSTEM INFO ==="
echo "OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d '"' -f2)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"
echo ""

echo "=== CPU ==="
lscpu | grep -E "Model name|Socket|Core|Thread"
echo "Load Average: $(uptime | awk -F'load average:' '{print $2}')"
echo ""

echo "=== MEMORY ==="
free -h
echo ""

echo "=== SWAP ==="
swapon --show
echo ""

echo "=== DISK SPACE ==="
df -hT --total | grep -v tmpfs
echo ""

echo "=== DISK LAYOUT ==="
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
echo ""

echo "=== LVM ==="
echo "VGs:"
sudo vgs
echo ""
echo "LVs:"
sudo lvs
echo ""

echo "=== TOP MEMORY USERS ==="
ps aux --sort=-%mem | head -6
echo ""

echo "=== TOP CPU USERS ==="
ps aux --sort=-%cpu | head -6
echo ""

echo "=== SWAP ACTIVITY ==="
vmstat 1 3
echo ""

echo "=== RECENT ERRORS ==="
journalctl -p err -n 10 --no-pager
echo ""

echo "========================================="
echo "End of Report"
echo "========================================="
EOF

# Make executable
chmod +x ~/system_health.sh

# Run it
~/system_health.sh

# Save to file
~/system_health.sh > ~/health_report_$(date +%Y%m%d).txt

# Review the report and answer:
# - Is your system healthy?
# - Any concerning errors?
# - Is swap being used actively?
# - Any resources under pressure?
```

---

## Exercise 10: Troubleshooting Simulation

**Objective:** Practice troubleshooting skills

```bash
# Scenario 1: Find what's using disk space
df -h  # Which partition is filling up?
sudo du -sh /* 2>/dev/null | sort -rh | head -10
sudo du -sh /var/* 2>/dev/null | sort -rh | head -10

# Scenario 2: Find memory leak
# Monitor memory usage over time
while true; do 
    date
    free -h | grep Mem
    sleep 5
done
# (Press Ctrl+C to stop)

# Scenario 3: Check for failed services
systemctl --failed
journalctl -p err -n 20

# Scenario 4: Verify filesystem integrity
# View mount options
mount | grep "^/dev"
# Check if any filesystems are read-only

# Scenario 5: Check network connectivity
ping -c 4 8.8.8.8
cat /etc/resolv.conf

# Document your findings and solutions
```

---

# 10. Quick Reference Commands

## 10.1 Most Essential Commands

### Daily Must-Run
```bash
free -h              # Memory status
df -hT               # Disk usage
lsblk -f             # Disk layout
uptime               # System load
```

### Disk & Storage
```bash
df -hT               # Disk free space with types
lsblk                # Block device tree
lsblk -f             # With filesystems and UUIDs
du -sh /*            # Directory sizes
mount                # Show mounts
findmnt              # Pretty mount tree
sudo blkid           # UUIDs of all partitions
```

### Memory & Swap
```bash
free -h              # Memory overview
swapon --show        # Swap devices
vmstat 2 5           # Virtual memory stats (5 samples)
ps aux --sort=-%mem | head -10  # Top memory users
```

### Process & Monitoring
```bash
top                  # Interactive monitor (M=mem, P=cpu, q=quit)
ps aux               # All processes
ps aux | grep name   # Find process
uptime               # Load averages
vmstat 2             # System stats
```

### Package Management
```bash
rpm -qa              # List all packages
rpm -qf /path/file   # Which package owns file
rpm -ql package      # Files in package
dnf search keyword   # Search packages
sudo dnf install pkg # Install package
sudo dnf update      # Update all
```

### Boot & Kernel
```bash
uname -r             # Current kernel
ls /boot/vmlinuz-*   # Installed kernels
sudo grub2-editenv list  # Default kernel
sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # Rebuild GRUB
```

### LVM
```bash
sudo pvs             # Physical volumes
sudo vgs             # Volume groups
sudo lvs             # Logical volumes
```

### Logs
```bash
journalctl -f        # Follow logs
journalctl -p err    # Errors only
journalctl -b        # Current boot
dmesg                # Kernel messages
```

---

## 10.2 Command Patterns

### Pattern Recognition
```
-h = Human-readable (KB, MB, GB)
-a = All (show everything)
-f = Follow / Force / File (context-dependent)
-r = Recursive / Reverse
-v = Verbose
-i = Interactive / Case-insensitive
-l = Long format / List
```

### Common Combinations
```bash
ls -lah              # List all, long format, human-readable
df -hT               # Disk free, human-readable with types
free -h              # Memory, human-readable
du -sh *             # Disk usage, summary, human-readable
ps aux               # All processes, user-oriented
```

---

## 10.3 Useful Aliases

Add to ~/.bashrc:
```bash
alias ll='ls -lah'
alias df='df -hT'
alias free='free -h'
alias du='du -h'
alias grep='grep --color=auto'
alias mem='ps aux --sort=-%mem | head -10'
alias cpu='ps aux --sort=-%cpu | head -10'
alias ports='sudo ss -tulnp'
```

---

## Summary

This guide covers:
✅ Complete Linux filesystem structure  
✅ Boot process (UEFI → GRUB2 → Kernel)  
✅ Disk management & partitioning  
✅ LVM architecture  
✅ Memory & swap management  
✅ Package management (RPM/DNF)  
✅ Symbolic links & mount points  
✅ System monitoring & performance  
✅ Troubleshooting procedures  
✅ 10 hands-on lab exercises  
✅ Quick reference commands  

**Study Approach:**
1. Read each section thoroughly
2. Run the commands on your RHEL9 system
3. Complete all 10 exercises
4. Create your own notes and diagrams
5. Practice daily with the quick reference commands

**Remember:** The best way to learn is by doing! 🚀

---

*Based on: RHEL 9 (rhel9exam01) on VMware ESXi 6.7*  
*Last Updated: March 2026*
