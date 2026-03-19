# Understanding Linux Storage and Memory: Complete Guide

**A comprehensive guide to understanding `df -hT`, virtual filesystems, /proc, /sys, sector sizes, and kernel memory management**

---

## Table of Contents

1. [Understanding df -hT Output](#understanding-df--ht-output)
2. [The Three Key Columns Explained](#the-three-key-columns-explained)
3. [Virtual Filesystems Deep Dive](#virtual-filesystems-deep-dive)
4. [Understanding /proc and /sys](#understanding-proc-and-sys)
5. [Sector Size and Conversion](#sector-size-and-conversion)
6. [Kernel RAM vs User RAM](#kernel-ram-vs-user-ram)
7. [Practical Commands and Scripts](#practical-commands-and-scripts)
8. [Quick Reference](#quick-reference)

---

# Understanding df -hT Output

## The Command

```bash
df -hT
# df = Disk Free (show disk space usage)
# -h = Human-readable (show sizes in GB, MB instead of bytes)
# -T = Type (show filesystem type)
```

---

## Example Output (RHEL 9 System)

```bash
[root@rhel9exam01 ~]# df -hT
Filesystem                        Type      Size  Used Avail Use% Mounted on
devtmpfs                          devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs                             tmpfs     3.8G  168K  3.8G   1% /dev/shm
tmpfs                             tmpfs     1.6G  9.6M  1.5G   1% /run
efivarfs                          efivarfs  256K   27K  225K  11% /sys/firmware/efi/efivars
/dev/mapper/rhel_rhel9exam01-root xfs        37G  9.1G   28G  25% /
/dev/sda2                         xfs      1014M  381M  634M  38% /boot
/dev/mapper/rhel_rhel9exam01-home xfs        18G  3.5G   15G  20% /home
/dev/sda1                         vfat      599M  7.4M  592M   2% /boot/efi
tmpfs                             tmpfs     769M   52K  769M   1% /run/user/42
tmpfs                             tmpfs     769M  116K  768M   1% /run/user/1000
```

---

# The Three Key Columns Explained

## Column 1: Filesystem (What Device?)

**Definition:** The device or source that provides the storage space

**Think of it as:** "WHERE is the data physically stored?"

---

### Physical Devices

```bash
/dev/sda1                         vfat      599M  7.4M  592M   2% /boot/efi
 ↑
Physical partition on disk sda (partition 1)

/dev/sda2                         xfs      1014M  381M  634M  38% /boot
 ↑
Physical partition on disk sda (partition 2)
```

**Breakdown:**
- `/dev/` = Device directory (where Linux stores device files)
- `sda` = First SCSI/SATA disk
- `1` or `2` = Partition number

**Physical location:**
```
sda (Physical disk)
├─ sda1 (/boot/efi) ← Actual partition on hardware
└─ sda2 (/boot)     ← Actual partition on hardware
```

---

### LVM Devices (Virtual)

```bash
/dev/mapper/rhel_rhel9exam01-root xfs        37G  9.1G   28G  25% /
 ↑
LVM Logical Volume (virtual device)

/dev/mapper/rhel_rhel9exam01-home xfs        18G  3.5G   15G  20% /home
 ↑
Another LVM Logical Volume
```

**Breakdown:**
- `/dev/mapper/` = Device mapper directory (for LVM)
- `rhel_rhel9exam01` = Volume Group name
- `root` or `home` = Logical Volume name

**What's behind it:**
```
/dev/mapper/rhel_rhel9exam01-root
        ↓
    dm-0 (Device Mapper device #0)
        ↓
    LVM Logical Volume "root"
        ↓
    Physical storage on /dev/sda3
```

---

### Virtual Filesystems (RAM-based)

```bash
tmpfs                             tmpfs     3.8G  168K  3.8G   1% /dev/shm
 ↑
Virtual filesystem (exists only in RAM, not on disk!)

devtmpfs                          devtmpfs  4.0M     0  4.0M   0% /dev
 ↑
Special virtual filesystem for device files
```

**Breakdown:**
- `tmpfs` = Type of virtual filesystem
- Not a physical device - stored in RAM
- Data is lost when system reboots
- No `/dev/` prefix because it's not a device file

---

### Special Filesystems

```bash
efivarfs                          efivarfs  256K   27K  225K  11% /sys/firmware/efi/efivars
 ↑
Special filesystem for UEFI variables (stored in motherboard firmware)
```

**What is this:**
- Not a disk device
- Not RAM either
- Stored in UEFI/BIOS firmware (motherboard chip)
- Persists across reboots

---

## Column 2: Type (What Kind of Filesystem?)

**Definition:** The filesystem format/driver used to organize data

**Think of it as:** "HOW is the data organized and accessed?"

---

### xfs (Physical Filesystems)

```bash
/dev/mapper/rhel_rhel9exam01-root xfs        37G  9.1G   28G  25% /
/dev/sda2                         xfs      1014M  381M  634M  38% /boot
                                   ↑
                                  XFS filesystem type
```

**What is XFS:**
- **X**tended **F**ile **S**ystem
- Default filesystem for RHEL/CentOS
- High-performance, journaling filesystem
- Good for large files and large volumes

**Used for:**
- Root filesystem (/)
- /boot partition
- /home directory
- Most data partitions in RHEL

---

### vfat (FAT32)

```bash
/dev/sda1                         vfat      599M  7.4M  592M   2% /boot/efi
                                   ↑
                                 FAT32 filesystem
```

**What is vfat:**
- **V**irtual **FAT** (File Allocation Table)
- Also known as FAT32
- Very old filesystem (Microsoft, 1996)
- **Required** for UEFI boot partitions

**Why used for /boot/efi:**
- UEFI firmware can ONLY read FAT32
- Must be compatible with firmware (pre-OS)

---

### tmpfs (RAM-based)

```bash
tmpfs                             tmpfs     3.8G  168K  3.8G   1% /dev/shm
tmpfs                             tmpfs     1.6G  9.6M  1.5G   1% /run
                                   ↑
                            Temporary filesystem in RAM
```

**What is tmpfs:**
- **T**e**mp**orary **F**ile**s**ystem
- Stored entirely in RAM (not on disk)
- Very fast (RAM speed)
- Lost when system reboots
- Size is virtual (only uses RAM as needed)

**Used for:**
- `/dev/shm` = Shared memory
- `/run` = Runtime data
- `/run/user/*` = User session data

---

### devtmpfs (Device Filesystem)

```bash
devtmpfs                          devtmpfs  4.0M     0  4.0M   0% /dev
                                   ↑
                            Device filesystem
```

**What is devtmpfs:**
- **Dev**ice **tmpfs**
- Virtual filesystem for device files
- Automatically creates device files (/dev/sda, /dev/tty, etc.)
- Also in RAM (like tmpfs)

---

### efivarfs (UEFI Variables)

```bash
efivarfs                          efivarfs  256K   27K  225K  11% /sys/firmware/efi/efivars
                                   ↑
                            UEFI variable filesystem
```

**What is efivarfs:**
- **EFI** **var**iables **F**ile**s**ystem
- Interface to UEFI firmware variables
- Stored in motherboard firmware (NVRAM)
- Persists across reboots

---

## Column 3: Mounted on (Where is it Accessible?)

**Definition:** The directory path where the filesystem is accessible

**Think of it as:** "WHERE can I access this storage?"

---

### Root Filesystem

```bash
/dev/mapper/rhel_rhel9exam01-root xfs        37G  9.1G   28G  25% /
                                                                   ↑
                                                            Mounted at /
```

**What this means:**
- This filesystem is mounted at `/` (root directory)
- Everything starts from here
- All other mount points are subdirectories of `/`

---

### Boot Partition

```bash
/dev/sda2                         xfs      1014M  381M  634M  38% /boot
                                                                   ↑
                                                            Mounted at /boot
```

**What's stored here:**
```
/boot/
├── vmlinuz-5.14.0-...  (Linux kernel)
├── initramfs-5.14.0... (Initial RAM disk)
├── grub2/              (GRUB bootloader)
└── config-5.14.0-...   (Kernel config)
```

---

### Home Directory

```bash
/dev/mapper/rhel_rhel9exam01-home xfs        18G  3.5G   15G  20% /home
                                                                   ↑
                                                            Mounted at /home
```

**What's stored here:**
```
/home/
├── kanishka/
│   ├── Documents/
│   ├── Downloads/
│   └── mini-lsblk.sh
└── otheruser/
```

---

### The Mount Point Hierarchy

```
Complete Mount Point Tree:

/ (root - dm-0)                          ← Base of everything
├── boot/ (sda2 mounted here)            ← Separate partition
│   └── efi/ (sda1 mounted here)         ← Nested mount
├── home/ (dm-2 mounted here)            ← Separate LV
├── dev/ (devtmpfs mounted here)         ← Virtual
├── dev/shm/ (tmpfs mounted here)        ← Virtual
├── run/ (tmpfs mounted here)            ← Virtual
│   └── user/
│       ├── 42/ (tmpfs)                  ← Per-user
│       └── 1000/ (tmpfs)                ← Per-user
└── sys/
    └── firmware/
        └── efi/
            └── efivars/ (efivarfs)      ← UEFI vars
```

---

# Virtual Filesystems Deep Dive

## What Are Virtual Filesystems?

Virtual filesystems are **not stored on physical disks**. They exist in RAM or are generated by the kernel on-the-fly.

---

## Two Categories of Virtual Filesystems

### Category 1: Zero-Size Virtual Filesystems (Hidden by Default)

**These filesystems show size = 0 in df**

```bash
# Show hidden virtual filesystems
df -hT -a | grep -E "proc|sysfs"

Filesystem     Type   Size  Used Avail Use% Mounted on
proc           proc      0     0     0    - /proc
sysfs          sysfs     0     0     0    - /sys
```

**Characteristics:**
- ❌ No RAM allocated for file storage
- ❌ Cannot create files
- ✅ Data generated from kernel structures
- ✅ Always shows size = 0

**Examples:**
- `proc` - Process and kernel information
- `sysfs` - Device and driver information
- `cgroup2` - Control groups
- `devpts` - Pseudo-terminals
- `bpf` - BPF filesystem
- `configfs` - Kernel configuration
- `debugfs` - Kernel debugging
- `tracefs` - Kernel tracing
- `securityfs` - Security modules

---

### Category 2: Non-Zero Virtual Filesystems (Shown by Default)

**These filesystems show size > 0 in df**

```bash
df -hT | grep -E "tmpfs|devtmpfs|efivarfs"

Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs          tmpfs     3.8G  168K  3.8G   1% /dev/shm
tmpfs          tmpfs     1.6G  9.6M  1.5G   1% /run
efivarfs       efivarfs  256K   27K  225K  11% /sys/firmware/efi/efivars
```

**Characteristics:**
- ✅ Allocates real RAM (tmpfs) or firmware space (efivarfs)
- ✅ Can create files
- ✅ Shows actual size
- ⚠️ Lost on reboot (tmpfs only)

**Examples:**
- `tmpfs` - Temporary storage in RAM
- `devtmpfs` - Device file management
- `efivarfs` - UEFI variables in firmware

---

## Complete Virtual Filesystem List

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              VIRTUAL FILESYSTEMS: VISIBILITY IN df -hT                      │
└─────────────────────────────────────────────────────────────────────────────┘

SHOWN by default (Size > 0):
═══════════════════════════════
✓ tmpfs        (RAM-based, configurable size)
✓ devtmpfs     (Device files, has size)
✓ efivarfs     (UEFI variables, 256K firmware storage)

HIDDEN by default (Size = 0):
═══════════════════════════════
✗ proc         (Process info - virtual)
✗ sysfs        (Device info - virtual)
✗ cgroup2      (Control groups - virtual)
✗ devpts       (Pseudo-terminals - virtual)
✗ bpf          (BPF programs - virtual)
✗ configfs     (Kernel config - virtual)
✗ debugfs      (Debugging - virtual)
✗ tracefs      (Tracing - virtual)
✗ pstore       (Crash logs - virtual)
✗ securityfs   (Security - virtual)

To see hidden ones:
═══════════════════
df -hT -a      (show all)
```

---

# Understanding /proc and /sys

## Why Size = 0?

```bash
df -hT /proc
Filesystem     Type  Size  Used Avail Use% Mounted on
proc           proc     0     0     0    - /proc

df -hT /sys
Filesystem     Type   Size  Used Avail Use% Mounted on
sysfs          sysfs     0     0     0    - /sys
```

**These filesystems don't allocate storage!**

---

## How /proc Works

### What is /proc?

```
/proc = Process Filesystem (procfs)

Purpose:  Interface to kernel and process information
Storage:  Generated dynamically by kernel
Type:     proc (virtual)
Size:     0 (no storage consumed)
```

---

### Real Example: /proc/swaps

```bash
cat /proc/swaps
Filename                                Type            Size            Used            Priority
/dev/dm-1                               partition       4141052         1566208         -2
```

**What happened:**
1. You requested `/proc/swaps`
2. Kernel checked: "What swap devices are active?"
3. Kernel found: dm-1 is active swap
4. Kernel read current swap usage from memory structures
5. Kernel formatted data as text
6. Returned the output

**The file `/proc/swaps` doesn't exist as a stored file!**

---

### Proof: /proc Files Show Size = 0

```bash
ls -lh /proc/swaps
-r--r--r--. 1 root root 0 Mar 20 10:30 /proc/swaps
                         ↑
                    Size = 0 bytes!

# But when you read it:
cat /proc/swaps
# Returns data! How?

# Because the kernel generates it when you read!
```

---

### What /proc Contains

```
/proc/
├── cpuinfo        # CPU information
├── meminfo        # Memory statistics
├── swaps          # Swap devices
├── mounts         # Mounted filesystems
├── [PID]/         # Per-process info
│   ├── cmdline    # Command line
│   ├── environ    # Environment variables
│   └── status     # Process status
└── sys/           # Kernel parameters (sysctl)
```

**Used by:**
- `free` (reads /proc/meminfo)
- `top` (reads /proc/[PID]/*)
- `ps` (reads /proc/[PID]/*)
- `lsblk` (reads /proc/mounts)

---

## How /sys Works

### What is /sys?

```
/sys = Sysfs (System Filesystem)

Purpose:  Interface to kernel device and driver information
Storage:  Generated dynamically by kernel
Type:     sysfs (virtual)
Size:     0 (no storage consumed)
```

---

### Real Example: /sys/block/dm-0/size

```bash
cat /sys/block/dm-0/size
76742656

# What happened:
# 1. You requested /sys/block/dm-0/size
# 2. Kernel checked dm-0 device structure
# 3. Kernel read device size (76742656 sectors)
# 4. Kernel returned as text
# 5. No file storage involved!
```

---

### What /sys Contains

```
/sys/
├── block/              # Block devices
│   ├── sda/            # Disk information
│   │   ├── size        # Disk size in sectors
│   │   └── sda1/       # Partition info
│   └── dm-0/           # Device mapper
│       └── dm/
│           └── name    # LVM name
├── class/              # Device classes
│   ├── net/            # Network interfaces
│   └── block/          # Block devices
├── devices/            # Device tree
└── firmware/           # Firmware interfaces
```

**Used by:**
- `lsblk` (reads /sys/block/*)
- `ip addr` (reads /sys/class/net/*)
- mini-lsblk script (reads /sys/block/*)

---

## The Key Difference: tmpfs vs proc/sysfs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              tmpfs (/dev/shm) vs proc/sysfs                                 │
└─────────────────────────────────────────────────────────────────────────────┘

tmpfs (/dev/shm):
═════════════════
Storage:  Allocates real RAM
Size:     3.8G (shows in df)
Files:    You can create files
Usage:    Creating 500MB file uses 500MB RAM
Reboot:   All files lost

Example:
echo "test" > /dev/shm/file.txt  ✅ Works!
ls -lh /dev/shm/file.txt
-rw-r--r--. 1 root root 5 Mar 20 10:30 file.txt


proc/sysfs:
═══════════
Storage:  No RAM allocated
Size:     0 (shows in df -a)
Files:    Cannot create files
Usage:    Reading uses NO additional RAM
Reboot:   Regenerated on boot

Example:
echo "test" > /proc/testfile  ❌ Permission denied!
cat /proc/meminfo  ✅ Works - kernel generates data
```

---

# Sector Size and Conversion

## What Are Sectors?

A **sector** is the smallest unit of storage on a disk.

```
Physical Disk Structure:
┌────────────────────────────────────────────────────────────┐
│                        Disk Surface                        │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐       │
│  │Sector│Sector│Sector│Sector│Sector│Sector│Sector│  ...  │
│  │  0   │  1   │  2   │  3   │  4   │  5   │  6   │       │
│  │ 512  │ 512  │ 512  │ 512  │ 512  │ 512  │ 512  │  ...  │
│  │bytes │bytes │bytes │bytes │bytes │bytes │bytes │       │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘       │
└────────────────────────────────────────────────────────────┘

Each sector = 512 bytes (traditional)
```

---

## Why /sys/block Shows Sectors

```bash
cat /sys/block/sda/size
125829120

# This is NOT bytes - it's SECTORS!
```

**Why sectors?**
- Historical standard from early hard drives
- Hardware-level addressing uses sectors
- Sector is the atomic unit of disk I/O

---

## The 512-Byte Standard

```bash
# Check sector size for a device
cat /sys/block/sda/queue/hw_sector_size
512

# Most traditional disks use 512 bytes per sector
```

**Industry standard:**
- **512 bytes** = Traditional (since 1950s!)
- **4096 bytes** = Advanced Format (newer drives)

---

## Converting Sectors to GB

### The Conversion Formula

```bash
# Generic formula:
size_in_gb = (sectors × 512) ÷ 1024 ÷ 1024 ÷ 1024

# In bash:
size_gb=$((sectors * 512 / 1024 / 1024 / 1024))
```

---

### Real Examples from RHEL 9

#### Example 1: Physical Disk (sda)

```bash
cat /sys/block/sda/size
125829120

# Convert to GB
cat /sys/block/sda/size | awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
60.00 GB
```

**Calculation:**
```
Sectors:  125,829,120
    ↓ × 512 (bytes per sector)
Bytes:    64,424,509,440
    ↓ ÷ 1024³
GB:       60.00
```

---

#### Example 2: Root LVM Volume (dm-0)

```bash
cat /sys/block/dm-0/size
76742656

# Convert to GB
cat /sys/block/dm-0/size | awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
36.59 GB
```

**Calculation:**
```
Sectors:  76,742,656
    ↓ × 512
Bytes:    39,292,239,872
    ↓ ÷ 1024³
GB:       36.59
```

---

#### Example 3: Swap LVM Volume (dm-1)

```bash
cat /sys/block/dm-1/size
8282104

# Convert to GB
cat /sys/block/dm-1/size | awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
3.95 GB
```

---

#### Example 4: Home LVM Volume (dm-2)

```bash
cat /sys/block/dm-2/size
37487616

# Convert to GB
cat /sys/block/dm-2/size | awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
17.87 GB
```

---

## Conversion Methods

### Method 1: Bash Integer Math

```bash
size_sectors=$(cat /sys/block/dm-0/size)
size_gb=$((size_sectors * 512 / 1024 / 1024 / 1024))
echo "${size_gb}G"
# Output: 36G

# Pros: Fast, no dependencies
# Cons: No decimals
```

---

### Method 2: Using awk (Recommended)

```bash
cat /sys/block/dm-0/size | awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
# Output: 36.59 GB

# Pros: Decimal precision, no extra packages
# Cons: Slightly slower
```

---

### Method 3: Using bc

```bash
echo "scale=3; $(cat /sys/block/dm-0/size) * 512 / 1024 / 1024 / 1024" | bc
# Output: 36.593

# Requires: dnf install -y bc
# Pros: Arbitrary precision
# Cons: Extra dependency
```

---

### Method 4: Using numfmt

```bash
numfmt --to=iec-i --suffix=B $(($(cat /sys/block/dm-0/size) * 512))
# Output: 36GiB

# Pros: Automatic unit selection
# Cons: Less control
```

---

## Conversion Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECTOR TO GB CONVERSION FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

Starting Point: /sys/block/dm-0/size
       │
       ↓
┌──────────────┐
│ 76,742,656   │  Sectors (what kernel stores)
└──────┬───────┘
       │ × 512 (bytes per sector)
       ↓
┌──────────────────┐
│ 39,292,239,872   │  Bytes
└──────┬───────────┘
       │ ÷ 1024 (bytes to kilobytes)
       ↓
┌──────────────────┐
│ 38,371,328       │  Kilobytes (KB)
└──────┬───────────┘
       │ ÷ 1024 (KB to megabytes)
       ↓
┌──────────────────┐
│ 37,472           │  Megabytes (MB)
└──────┬───────────┘
       │ ÷ 1024 (MB to gigabytes)
       ↓
┌──────────────────┐
│ 36.59            │  Gigabytes (GB) ← Final!
└──────────────────┘
```

---

# Kernel RAM vs User RAM

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    YOUR PHYSICAL RAM (7.5 GB)                               │
└─────────────────────────────────────────────────────────────────────────────┘

This ONE physical RAM is divided by the kernel:

┌─────────────────────────────────────────────────────────────────────────────┐
│ ████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
│                                                                             │
│ Kernel Space              User Space                                        │
│ (Reserved)                (Available)                                       │
│ ↓                         ↓                                                 │
│ Kernel code               Applications                                      │
│ Kernel structures         Processes                                         │
│ Device drivers            Libraries                                         │
│ Buffer cache              tmpfs files                                       │
│ Slab cache                User data                                         │
│                                                                             │
│ Can't be swapped         Can be swapped                                     │
└─────────────────────────────────────────────────────────────────────────────┘

⚠️ IMPORTANT: Same physical RAM!
   Kernel just manages portions for different purposes.
```

---

## There is NO Separate Kernel RAM Module

```
Physical RAM Modules (DIMMs):
┌──────────────────┐
│   RAM Module 1   │  ← Physical stick (e.g., DDR4)
│     4 GB         │
└──────────────────┘
┌──────────────────┐
│   RAM Module 2   │  ← Another physical stick
│     4 GB         │
└──────────────────┘

Total: ~8 GB (shows as 7.5 GB usable)

The kernel uses THIS SAME RAM!
There is NO separate "kernel RAM module"!
```

---

## How to See Kernel RAM Usage

### Method 1: Using free

```bash
free -h
              total        used        free      shared  buff/cache   available
Mem:          7.5Gi       4.7Gi       721Mi       168Ki       2.1Gi       2.7Gi
Swap:         3.9Gi       1.5Gi       2.4Gi
```

**Column meanings:**
- `total`: 7.5Gi - Total physical RAM
- `used`: 4.7Gi - Used by processes + kernel
- `free`: 721Mi - Completely unused
- `buff/cache`: 2.1Gi - Kernel cache (reclaimable)
- `available`: 2.7Gi - Available for new processes

---

### Method 2: Using /proc/meminfo

```bash
cat /proc/meminfo
```

**Key kernel memory fields:**
```
MemTotal:        7864532 kB  ← Total RAM
Slab:             352968 kB  ← Kernel slab cache ← KERNEL RAM!
SReclaimable:     195136 kB  ← Reclaimable kernel objects
SUnreclaim:       157832 kB  ← Permanent kernel objects ← KERNEL RAM!
KernelStack:       16736 kB  ← Kernel stack ← KERNEL RAM!
PageTables:        73560 kB  ← Page tables ← KERNEL RAM!
```

---

### Method 3: Calculate Kernel RAM

```bash
# Permanent kernel memory (cannot be freed)
awk '/^SUnreclaim:|^KernelStack:|^PageTables:/ {sum+=$2} END {print "Permanent Kernel RAM: " sum/1024 " MB"}' /proc/meminfo
# Output: Permanent Kernel RAM: 242.62 MB

# Total kernel memory (including reclaimable)
awk '/^Slab:|^KernelStack:|^PageTables:/ {sum+=$2} END {print "Total Kernel RAM: " sum/1024 " MB"}' /proc/meminfo
# Output: Total Kernel RAM: 430.85 MB
```

---

### Method 4: Using slabtop (Real-Time)

```bash
# Show kernel memory object cache
sudo slabtop -o

# Output shows kernel memory usage:
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
463680 461337  99%    0.19K  22080       21     88320K dentry
172032 171456  99%    0.10K   4416       39     17664K buffer_head
```

**What this shows:**
- `dentry` - Directory entry cache (kernel filesystem metadata)
- `buffer_head` - Block buffer cache (kernel I/O)
- All kernel RAM usage in real-time

---

## RAM Breakdown Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    YOUR 7.5 GB RAM BREAKDOWN                                │
└─────────────────────────────────────────────────────────────────────────────┘

Total RAM: 7.5 GB
│
├─ Kernel Permanent: ~243 MB ←── KERNEL RAM (Cannot be freed)
│   ├─ Kernel code
│   ├─ Kernel data structures
│   ├─ Page tables (73 MB)
│   ├─ Kernel stacks (16 MB)
│   └─ SUnreclaim (157 MB)
│
├─ Kernel Reclaimable: ~2.1 GB ←── KERNEL RAM (Can be freed if needed)
│   ├─ Page cache (1.8 GB)
│   ├─ Buffer cache (89 MB)
│   └─ Slab cache (195 MB)
│
├─ User Space: ~3.5 GB
│   ├─ Applications
│   ├─ Processes
│   └─ tmpfs files
│
└─ Free: ~721 MB ←── Immediately available
```

---

## Kernel RAM Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        KERNEL RAM BREAKDOWN                                 │
└─────────────────────────────────────────────────────────────────────────────┘

1. Kernel Code (Permanent)
   ├─ Linux kernel executable
   ├─ Kernel modules (drivers)
   └─ ~10-30 MB

2. Kernel Data (Permanent)
   ├─ Process descriptors
   ├─ Memory management
   ├─ Scheduler data
   └─ ~157 MB (SUnreclaim)

3. Kernel Stack (Permanent)
   ├─ Kernel thread stacks
   └─ ~16 MB (KernelStack)

4. Page Tables (Permanent)
   ├─ Virtual to physical mapping
   └─ ~73 MB (PageTables)

5. Cache (Reclaimable)
   ├─ Page cache (~1.8 GB)
   ├─ Buffer cache (~89 MB)
   └─ Slab cache (~195 MB)

Total Kernel:
Permanent: ~243 MB (3.2%)
Reclaimable: ~2.1 GB (28%)
Total: ~2.3 GB (31%)
```

---

# Practical Commands and Scripts

## Show All Filesystems (Including Hidden)

```bash
# Show ALL filesystems
df -hT -a

# Show only virtual filesystems
df -hT -a | grep -vE "^/dev/sd|^/dev/mapper|xfs|vfat"

# Show only hidden (size=0) filesystems
df -hT -a | awk '$3 == 0 {print}'
```

---

## Convert Sector Sizes

```bash
# Function to convert /sys/block size to GB
sys_block_gb() {
    local device="$1"
    cat "/sys/block/$device/size" | \
        awk '{printf "%.2f GB\n", $1 * 512 / 1024 / 1024 / 1024}'
}

# Usage:
sys_block_gb sda
# 60.00 GB

sys_block_gb dm-0
# 36.59 GB
```

---

## Complete Memory Breakdown Script

```bash
#!/bin/bash
# memory-breakdown.sh

echo "========================================="
echo "     COMPLETE RAM BREAKDOWN"
echo "========================================="

# Total RAM
total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
total_gb=$(echo "scale=2; $total/1024/1024" | bc)
echo "Total RAM: ${total_gb} GB"

# Kernel permanent
sunreclaim=$(awk '/SUnreclaim/ {print $2}' /proc/meminfo)
kstack=$(awk '/KernelStack/ {print $2}' /proc/meminfo)
ptables=$(awk '/PageTables/ {print $2}' /proc/meminfo)
perm=$((sunreclaim + kstack + ptables))
perm_mb=$(echo "scale=2; $perm/1024" | bc)

echo ""
echo "Kernel Permanent: ${perm_mb} MB"
echo "  ├─ SUnreclaim:  $(echo "scale=2; $sunreclaim/1024" | bc) MB"
echo "  ├─ KernelStack: $(echo "scale=2; $kstack/1024" | bc) MB"
echo "  └─ PageTables:  $(echo "scale=2; $ptables/1024" | bc) MB"

# Kernel reclaimable
cached=$(awk '/^Cached:/ {print $2}' /proc/meminfo)
buffers=$(awk '/^Buffers:/ {print $2}' /proc/meminfo)
sreclaimable=$(awk '/SReclaimable/ {print $2}' /proc/meminfo)
reclaim=$((cached + buffers + sreclaimable))
reclaim_gb=$(echo "scale=2; $reclaim/1024/1024" | bc)

echo ""
echo "Kernel Reclaimable: ${reclaim_gb} GB"
echo "  ├─ Cached:       $(echo "scale=2; $cached/1024" | bc) MB"
echo "  ├─ Buffers:      $(echo "scale=2; $buffers/1024" | bc) MB"
echo "  └─ SReclaimable: $(echo "scale=2; $sreclaimable/1024" | bc) MB"

# Free
free=$(awk '/MemFree/ {print $2}' /proc/meminfo)
free_gb=$(echo "scale=2; $free/1024/1024" | bc)

echo ""
echo "Free RAM: ${free_gb} GB"

# User space
used=$((total - free - reclaim))
user=$((used - perm))
user_gb=$(echo "scale=2; $user/1024/1024" | bc)

echo "User Space: ${user_gb} GB"
echo "========================================="
```

---

## Show All Device Sizes

```bash
#!/bin/bash
# show-device-sizes.sh

echo "Device         Sectors          Bytes                GB"
echo "================================================================="

for device in /sys/block/{sd*,dm-*}; do
    if [ -f "$device/size" ]; then
        dev=$(basename "$device")
        sectors=$(cat "$device/size")
        bytes=$((sectors * 512))
        gb=$(awk -v s=$sectors 'BEGIN {printf "%.2f", s * 512 / 1024 / 1024 / 1024}')
        
        printf "%-12s %14d %18d %12.2f GB\n" "$dev" "$sectors" "$bytes" "$gb"
    fi
done
```

**Output:**
```
Device         Sectors          Bytes                GB
=================================================================
sda              125829120       64424509440        60.00 GB
dm-0              76742656       39292239872        36.59 GB
dm-1               8282104        4240437248         3.95 GB
dm-2              37487616       19193659392        17.87 GB
```

---

# Quick Reference

## Filesystem Types Quick Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FILESYSTEM TYPE REFERENCE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│ Type      │ Storage │ Speed    │ Persistent? │ Use Case                    │
├───────────┼─────────┼──────────┼─────────────┼─────────────────────────────┤
│ xfs       │ Disk    │ Fast     │ Yes         │ Main data (RHEL default)    │
│ vfat      │ Disk    │ Moderate │ Yes         │ UEFI boot only              │
│ tmpfs     │ RAM     │ Very Fast│ No          │ Temporary data              │
│ devtmpfs  │ RAM     │ Very Fast│ No          │ Device files                │
│ proc      │ Virtual │ Instant  │ Regenerated │ Process/kernel info         │
│ sysfs     │ Virtual │ Instant  │ Regenerated │ Device/driver info          │
│ efivarfs  │ Firmware│ Slow     │ Yes         │ UEFI settings               │
└───────────┴─────────┴──────────┴─────────────┴─────────────────────────────┘
```

---

## Sector Conversion Cheat Sheet

```bash
# To Bytes:
bytes=$((sectors * 512))

# To KB:
kb=$((sectors * 512 / 1024))

# To MB:
mb=$((sectors * 512 / 1024 / 1024))

# To GB (integer):
gb=$((sectors * 512 / 1024 / 1024 / 1024))

# To GB (decimal with awk):
gb=$(awk -v s=$sectors 'BEGIN {printf "%.2f", s * 512 / 1024 / 1024 / 1024}')
```

---

## Common df Commands

```bash
# Show all filesystems in human-readable format with types
df -hT

# Show ALL filesystems (including hidden virtual ones)
df -hT -a

# Show only physical filesystems
df -hT | grep -E "xfs|vfat|ext4"

# Show only virtual filesystems
df -hT | grep -E "tmpfs|devtmpfs"

# Show specific filesystem
df -hT /proc
df -hT /home

# Show inodes instead of blocks
df -i

# Exclude certain filesystem types
df -hT -x tmpfs -x devtmpfs
```

---

## Memory Commands

```bash
# Show RAM usage summary
free -h

# Show detailed memory info
cat /proc/meminfo

# Show kernel slab usage (real-time)
sudo slabtop -o

# Calculate kernel RAM
awk '/^SUnreclaim:|^KernelStack:|^PageTables:/ {sum+=$2} END {print sum/1024 " MB"}' /proc/meminfo

# Show swap usage
cat /proc/swaps
swapon --show
```

---

## Virtual Filesystem Commands

```bash
# List all proc entries
ls /proc

# List all sys entries
ls /sys

# Check if tmpfs file uses RAM
dd if=/dev/zero of=/dev/shm/test bs=1M count=100
free -h  # Check RAM usage
rm /dev/shm/test
free -h  # RAM freed

# Read kernel info
cat /proc/cpuinfo
cat /proc/meminfo
cat /sys/block/sda/size
```

---

## Summary

### Key Takeaways

```
1. df -hT shows three key columns:
   ├─ Filesystem: Device providing storage
   ├─ Type: Filesystem format
   └─ Mounted on: Where accessible

2. Virtual filesystems come in two types:
   ├─ Zero-size (proc, sysfs): No storage, data generated
   └─ Non-zero (tmpfs): Allocates RAM, lost on reboot

3. Sector conversion:
   └─ GB = sectors × 512 ÷ 1024³

4. Kernel RAM:
   ├─ Same physical RAM as user space
   ├─ Permanent: ~243 MB (3%)
   ├─ Reclaimable: ~2.1 GB (28%)
   └─ Total: ~31% of RAM

5. Important locations:
   ├─ /proc: Process and kernel info
   ├─ /sys: Device and driver info
   ├─ /dev/shm: Temporary RAM storage
   └─ /sys/block: Block device info
```

---

## Additional Resources

### Man Pages

```bash
man df
man proc       # /proc filesystem
man sysfs      # /sys filesystem
man free       # Memory display
```

### Related Commands

```bash
lsblk          # List block devices
mount          # Show mounted filesystems
findmnt        # Find mounted filesystems
blkid          # Block device attributes
```

---

## System Tested On

```
OS: Red Hat Enterprise Linux 9.0
Hostname: rhel9exam01
Disk: 60GB (VMware VMDK)
RAM: 7.5GB
Storage Layout:
├─ sda1: 600MB (vfat) - /boot/efi
├─ sda2: 1GB (xfs) - /boot
└─ sda3: 58GB (LVM)
   ├─ dm-0: 36GB (xfs) - /
   ├─ dm-1: 4GB (swap)
   └─ dm-2: 17GB (xfs) - /home
```

---

## License

This guide is provided under the MIT License.

---

**Last Updated:** March 2026  
**Version:** 1.0  
**Author:** Based on RHEL 9 learning sessions

---

**Happy Learning!** 🚀
