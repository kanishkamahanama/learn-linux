# Building mini-lsblk: Understanding Linux Block Devices

**A Complete Guide to Creating Your Own lsblk Command with Visual Storage Diagrams**

---

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [The Enhanced mini-lsblk Script](#the-enhanced-mini-lsblk-script)
4. [Installation Guide](#installation-guide)
5. [Usage Examples](#usage-examples)
6. [Understanding the Output](#understanding-the-output)
7. [Complete Visual Disk Layout](#complete-visual-disk-layout)
8. [How the Script Works](#how-the-script-works)
9. [Customization Options](#customization-options)
10. [Troubleshooting](#troubleshooting)

---

# Introduction

This guide demonstrates how Linux commands like `lsblk` actually work by building a simplified version from scratch. You'll learn how to read directly from `/proc` and `/sys` filesystems to gather block device information, and understand the complete storage architecture of a RHEL 9 system.

## Prerequisites

- RHEL 9 / CentOS Stream 9 / Rocky Linux 9 (or similar)
- Basic bash scripting knowledge
- Root or sudo access for installation

## System Tested On

```
OS: Red Hat Enterprise Linux 9.0
Hostname: rhel9exam01
Disk: 60GB VMDK (VMware)
RAM: 7.5GB
Filesystem: XFS on LVM
```

---

# What You'll Learn

By following this guide, you will understand:

✅ How to read block device information from `/sys/block/`  
✅ How to detect filesystems from `/proc/mounts`  
✅ How to identify swap devices from `/proc/swaps`  
✅ How Device Mapper and LVM work together  
✅ The complete storage stack from hardware to mount points  
✅ How to install custom commands system-wide  

**Most importantly:** You'll understand that Linux commands are just programs reading from special files!

---

# The Enhanced mini-lsblk Script

## Complete Script Code

```bash
#!/bin/bash
# mini-lsblk-enhanced.sh - Enhanced lsblk implementation
# Shows device, size, type, filesystem, and mount point
# Reads only from /sys and /proc (no external lsblk calls)
#
# Author: Based on RHEL 9 learning sessions
# Date: March 2026
# License: MIT

# Function to get filesystem type from /proc/mounts
get_fstype() {
    local device="$1"
    # Try both /dev/device and /dev/mapper/device formats
    awk -v dev="/dev/$device" '$1 == dev {print $3; exit}' /proc/mounts
}

# Function to get mount point from /proc/mounts
get_mountpoint() {
    local device="$1"
    awk -v dev="/dev/$device" '$1 == dev {print $2; exit}' /proc/mounts
}

# Function to check if device is swap from /proc/swaps
is_swap() {
    local device="$1"
    if [ -f /proc/swaps ]; then
        grep -q "/dev/$device" /proc/swaps && echo "swap"
    fi
}

echo "NAME              SIZE    TYPE    FSTYPE       MOUNTPOINT"
echo "==========================================================="

# List block devices from /sys
for device in /sys/block/*; do
    dev_name=$(basename "$device")
    
    # Skip loop and ram devices
    [[ $dev_name =~ ^(loop|ram) ]] && continue
    
    # Get size from /sys
    if [ -f "$device/size" ]; then
        size_sectors=$(cat "$device/size" 2>/dev/null || echo 0)
        size_gb=$((size_sectors * 512 / 1024 / 1024 / 1024))
    else
        size_gb=0
    fi
    
    # Print device
    printf "%-15s %4dG    disk\n" "$dev_name" "$size_gb"
    
    # List partitions
    for part in "$device/$dev_name"*; do
        if [ -d "$part" ] && [[ ! "$part" =~ holders$ ]]; then
            part_name=$(basename "$part")
            
            # Skip if it's the device itself
            [ "$part_name" = "$dev_name" ] && continue
            
            # Get partition size
            if [ -f "$part/size" ]; then
                part_size=$(cat "$part/size" 2>/dev/null || echo 0)
                part_gb=$((part_size * 512 / 1024 / 1024 / 1024))
            else
                part_gb=0
            fi
            
            # Get filesystem type
            fstype=$(get_fstype "$part_name")
            [ -z "$fstype" ] && fstype=$(is_swap "$part_name")
            [ -z "$fstype" ] && fstype=""
            
            # Get mount point
            mountpoint=$(get_mountpoint "$part_name")
            [ -z "$mountpoint" ] && mountpoint=""
            
            # Special handling for swap
            if [ "$fstype" = "swap" ]; then
                mountpoint="[SWAP]"
            fi
            
            # Print partition with tree characters
            printf "├─%-12s %4dG    part    %-12s %s\n" "$part_name" "$part_gb" "$fstype" "$mountpoint"
        fi
    done
done

# Check device mapper devices (LVM volumes)
if [ -d /sys/block/dm-0 ]; then
    echo ""
    echo "Device Mapper (LVM) Volumes:"
    echo "==========================================================="
    
    for dm_device in /sys/block/dm-*; do
        dm_name=$(basename "$dm_device")
        
        # Get size
        if [ -f "$dm_device/size" ]; then
            size_sectors=$(cat "$dm_device/size" 2>/dev/null || echo 0)
            size_gb=$((size_sectors * 512 / 1024 / 1024 / 1024))
        else
            size_gb=0
        fi
        
        # Get DM name (friendly name)
        if [ -f "$dm_device/dm/name" ]; then
            dm_friendly_name=$(cat "$dm_device/dm/name" 2>/dev/null)
        else
            dm_friendly_name="$dm_name"
        fi
        
        # Get filesystem type
        fstype=$(get_fstype "$dm_name")
        [ -z "$fstype" ] && fstype=$(get_fstype "mapper/$dm_friendly_name")
        [ -z "$fstype" ] && fstype=$(is_swap "$dm_name")
        [ -z "$fstype" ] && fstype=""
        
        # Get mount point
        mountpoint=$(get_mountpoint "$dm_name")
        [ -z "$mountpoint" ] && mountpoint=$(get_mountpoint "mapper/$dm_friendly_name")
        [ -z "$mountpoint" ] && mountpoint=""
        
        # Special handling for swap
        if [ "$fstype" = "swap" ]; then
            mountpoint="[SWAP]"
        fi
        
        printf "%-15s %4dG    lvm     %-12s %s\n" "$dm_friendly_name" "$size_gb" "$fstype" "$mountpoint"
    done
fi

echo ""
echo "SWAP DEVICES:"
echo "==========================================================="
if [ -f /proc/swaps ]; then
    cat /proc/swaps
else
    echo "No swap information available"
fi

echo ""
echo "MOUNTED FILESYSTEMS:"
echo "==========================================================="
# Filter out virtual/special filesystems for cleaner output
cat /proc/mounts | grep -v "^proc\|^sysfs\|^devpts\|^tmpfs\|^cgroup\|^mqueue\|^hugetlbfs\|^debugfs\|^tracefs\|^configfs\|^fusectl\|^selinuxfs\|^bpf\|^pstore" | column -t
```

---

# Installation Guide

## Quick Install (Recommended)

### Step 1: Download the Script

```bash
# Navigate to your home directory
cd ~

# Create the script file
cat > mini-lsblk.sh << 'EOF'
[Paste the complete script from above]
EOF

# Make it executable
chmod +x mini-lsblk.sh
```

---

### Step 2: Test the Script

```bash
# Run from current directory
./mini-lsblk.sh

# You should see output showing your block devices
```

---

### Step 3: Install System-Wide

```bash
# Install to /usr/local/bin (recommended for custom scripts)
sudo install -m 755 mini-lsblk.sh /usr/local/bin/mini-lsblk

# Verify installation
which mini-lsblk
# Should output: /usr/local/bin/mini-lsblk

# Check it's in your PATH
type mini-lsblk
# Should output: mini-lsblk is /usr/local/bin/mini-lsblk
```

---

### Step 4: Run from Anywhere

```bash
# Now you can run it from any directory
cd /tmp
mini-lsblk

# Works just like any other command!
```

---

## Alternative Installation Methods

### Personal Installation (No sudo required)

```bash
# Create personal bin directory
mkdir -p ~/bin

# Copy script
cp mini-lsblk.sh ~/bin/mini-lsblk
chmod 755 ~/bin/mini-lsblk

# Add to PATH if not already there
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Test
mini-lsblk
```

---

### Custom Directory Installation

```bash
# Create custom scripts directory
sudo mkdir -p /opt/myscripts

# Install script
sudo install -m 755 mini-lsblk.sh /opt/myscripts/mini-lsblk

# Add to system PATH
echo 'export PATH="/opt/myscripts:$PATH"' | sudo tee /etc/profile.d/myscripts.sh

# Apply immediately
source /etc/profile.d/myscripts.sh

# Test
mini-lsblk
```

---

# Usage Examples

## Basic Usage

```bash
# Run the command
mini-lsblk

# Example output on RHEL 9:
NAME              SIZE    TYPE    FSTYPE       MOUNTPOINT
===========================================================
dm-0              36G    disk
dm-1               3G    disk
dm-2              17G    disk
sda               60G    disk
├─sda1            0G    part    vfat         /boot/efi
├─sda2            1G    part    xfs          /boot
├─sda3           58G    part
sr0                7G    disk

Device Mapper (LVM) Volumes:
===========================================================
rhel_rhel9exam01-root   36G    lvm     xfs          /
rhel_rhel9exam01-swap    3G    lvm     swap         [SWAP]
rhel_rhel9exam01-home   17G    lvm     xfs          /home

SWAP DEVICES:
===========================================================
Filename                Type      Size      Used    Priority
/dev/dm-1               partition 4141052   1567232 -2

MOUNTED FILESYSTEMS:
===========================================================
[List of mounted filesystems]
```

---

## Compare with Real lsblk

```bash
# Run both side by side
echo "=== mini-lsblk ===" && mini-lsblk && echo "" && echo "=== Real lsblk ===" && lsblk -f

# See how close your implementation is!
```

---

## Redirect Output to File

```bash
# Save output for documentation
mini-lsblk > system-storage-layout.txt

# View later
cat system-storage-layout.txt
```

---

## Use in Scripts

```bash
#!/bin/bash
# Example: Check if swap is active

if mini-lsblk | grep -q "SWAP"; then
    echo "Swap is active"
else
    echo "No swap detected"
fi
```

---

# Understanding the Output

## Output Breakdown: Line by Line

Let's analyze actual output from a RHEL 9 system:

### Block Devices Section

```
NAME              SIZE    TYPE    FSTYPE       MOUNTPOINT
===========================================================
sda               60G    disk
```

**Explanation:**
- **sda** = First SCSI/SATA disk (physical disk)
- **60G** = Total disk size (read from `/sys/block/sda/size`)
- **disk** = Physical disk device type
- **Source:** `/sys/block/sda/`

---

```
├─sda1            0G    part    vfat         /boot/efi
```

**Explanation:**
- **sda1** = First partition on sda
- **0G** = Less than 1GB (actually 600MB, rounded down)
- **part** = Partition type
- **vfat** = FAT32 filesystem (required for UEFI)
- **/boot/efi** = Mount point (UEFI boot partition)
- **Source:** `/sys/block/sda/sda1/` + `/proc/mounts`

---

```
├─sda2            1G    part    xfs          /boot
```

**Explanation:**
- **sda2** = Second partition
- **1G** = 1GB size
- **xfs** = XFS filesystem
- **/boot** = Mount point (contains Linux kernels)
- **Purpose:** Boot partition (kernel, initramfs)

---

```
├─sda3           58G    part
```

**Explanation:**
- **sda3** = Third partition (58GB)
- **No FSTYPE** = Not directly formatted
- **No MOUNTPOINT** = Not directly mounted
- **Why?** This is an LVM Physical Volume
- **Contains:** All LVM Logical Volumes (root, swap, home)

---

### Device Mapper (LVM) Section

```
Device Mapper (LVM) Volumes:
===========================================================
rhel_rhel9exam01-root   36G    lvm     xfs          /
```

**Explanation:**
- **rhel_rhel9exam01-root** = LVM Logical Volume name
  - Format: `{VolumeGroup}-{LogicalVolume}`
- **36G** = Size of logical volume
- **lvm** = LVM volume type
- **xfs** = Filesystem on this volume
- **/** = Root filesystem mount point
- **Maps to:** dm-0 device
- **Source:** `/sys/block/dm-0/dm/name` + `/proc/mounts`

---

```
rhel_rhel9exam01-swap    3G    lvm     swap         [SWAP]
```

**Explanation:**
- **swap** LV = Swap space (virtual memory)
- **3G** = Actually 4GB (rounding in script)
- **swap** = Swap filesystem type
- **[SWAP]** = Active swap indicator
- **Maps to:** dm-1 device
- **Source:** `/proc/swaps`

---

```
rhel_rhel9exam01-home   17G    lvm     xfs          /home
```

**Explanation:**
- **home** LV = User home directories
- **17G** = Size
- **xfs** = Filesystem type
- **/home** = Mount point
- **Maps to:** dm-2 device

---

### Swap Devices Section

```
SWAP DEVICES:
===========================================================
Filename                Type      Size      Used    Priority
/dev/dm-1               partition 4141052   1567232 -2
```

**Explanation:**
- **Filename:** `/dev/dm-1` = Device file
- **Type:** `partition` = Partition-based swap (vs file-based)
- **Size:** `4141052` KB = ~4GB total swap
- **Used:** `1567232` KB = ~1.5GB currently in use (38%)
- **Priority:** `-2` = Swap priority (negative = kernel-assigned)
- **Source:** Direct output from `/proc/swaps`

---

# Complete Visual Disk Layout

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        YOUR RHEL 9 SYSTEM STORAGE                           │
└─────────────────────────────────────────────────────────────────────────────┘

Physical Hardware:
┌────────────────┐     ┌────────────────┐
│   sda (60GB)   │     │  sr0 (7GB ISO) │
│  SCSI/SATA SSD │     │  Optical Drive │
└────────┬───────┘     └────────────────┘
         │
         ├─ sda1 (600MB) ──────────────────────┐
         │                                      │
         ├─ sda2 (1GB) ────────────────────────┤
         │                                      │
         └─ sda3 (58GB) ──┐                    │
                          │                    │
                          ↓                    ↓
                    LVM Layer           Direct Filesystems
                          │                    │
         ┌────────────────┴────────────┐       │
         │                             │       │
         ↓                             ↓       ↓
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────┐  ┌──────┐
    │ dm-0    │  │ dm-1    │  │ dm-2    │  │ sda1 │  │ sda2 │
    │ 36GB    │  │ 4GB     │  │ 17GB    │  │600MB │  │ 1GB  │
    │ root LV │  │ swap LV │  │ home LV │  │      │  │      │
    └────┬────┘  └────┬────┘  └────┬────┘  └──┬───┘  └──┬───┘
         │            │            │           │         │
         │            │            │           │         │
    ┌────▼────┐  ┌───▼────┐  ┌───▼────┐  ┌───▼───┐ ┌──▼───┐
    │ xfs     │  │ swap   │  │ xfs    │  │ vfat  │ │ xfs  │
    └────┬────┘  └───┬────┘  └───┬────┘  └───┬───┘ └──┬───┘
         │            │            │           │        │
         ↓            ↓            ↓           ↓        ↓
        /          [SWAP]       /home     /boot/efi  /boot
```

---

## Physical Disk Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PHYSICAL DISK: sda (60GB)                           │
│                         Device File: /dev/sda                               │
│                         Location: /sys/block/sda/                           │
└─────────────────────────────────────────────────────────────────────────────┘

Byte Position:
0GB                                                                      60GB
├──────┬──────────┬──────────────────────────────────────────────────────┤
│      │          │                                                      │
│ sda1 │   sda2   │                      sda3                            │
│600MB │   1GB    │                      58GB                            │
│      │          │                                                      │
└──┬───┴────┬─────┴────────────────────┬─────────────────────────────────┘
   │        │                          │
   │        │                          │
   ↓        ↓                          ↓
┌──────┐ ┌──────┐               ┌──────────────┐
│ EFI  │ │/boot │               │ LVM Physical │
│System│ │      │               │   Volume     │
│Part  │ │      │               │   (PV)       │
└──────┘ └──────┘               └──────────────┘
 vfat     xfs                         │
   │       │                          │
   │       │                          └─────────────────────┐
   │       │                                                │
   ↓       ↓                                                ↓
/boot/efi /boot                                    LVM Volume Group
                                                   rhel_rhel9exam01
                                                          │
                                      ┌───────────────────┼───────────────────┐
                                      │                   │                   │
                                      ↓                   ↓                   ↓
                                  ┌────────┐         ┌────────┐         ┌────────┐
                                  │ root   │         │ swap   │         │ home   │
                                  │ 36GB   │         │ 4GB    │         │ 17GB   │
                                  │ LV     │         │ LV     │         │ LV     │
                                  └────────┘         └────────┘         └────────┘
```

---

## LVM Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LVM ARCHITECTURE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Layer 1: Physical Volume (PV)
═════════════════════════════
┌─────────────────────────────────────────────────────────────────────────────┐
│                         /dev/sda3 (58GB)                                    │
│                      LVM Physical Volume                                    │
│                                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │   PE    │ │   PE    │ │   PE    │ │   PE    │ │   PE    │ │   ...   │ │
│  │ (4MB)   │ │ (4MB)   │ │ (4MB)   │ │ (4MB)   │ │ (4MB)   │ │         │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
│                                                                             │
│  PE = Physical Extent (default 4MB chunks)                                 │
│  Total PEs: ~14,848 extents (58GB / 4MB)                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
                                      │
Layer 2: Volume Group (VG)            │
═══════════════════════════           │
┌─────────────────────────────────────┴───────────────────────────────────────┐
│                    Volume Group: rhel_rhel9exam01                           │
│                    Total Size: 58GB (14,848 PEs)                            │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌─────────┐  │
│  │  Allocated to  │  │  Allocated to  │  │  Allocated to  │  │  Free   │  │
│  │  root LV       │  │  swap LV       │  │  home LV       │  │  PEs    │  │
│  │  9,216 PEs     │  │  1,024 PEs     │  │  4,352 PEs     │  │  ~256   │  │
│  │  (36GB)        │  │  (4GB)         │  │  (17GB)        │  │  (~1GB) │  │
│  └────────────────┘  └────────────────┘  └────────────────┘  └─────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
                          ┌───────────┼───────────┐
                          │           │           │
Layer 3: Logical Volumes (LV)         │           │
═════════════════════════             │           │
                                      │           │
    ┌─────────────────────────────────┘           │
    │                                             │
    ↓                           ↓                 ↓
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Logical Volume  │    │ Logical Volume  │    │ Logical Volume  │
│                 │    │                 │    │                 │
│ Name: root      │    │ Name: swap      │    │ Name: home      │
│ Size: 36GB      │    │ Size: 4GB       │    │ Size: 17GB      │
│ PEs:  9,216     │    │ PEs:  1,024     │    │ PEs:  4,352     │
│                 │    │                 │    │                 │
│ Device:         │    │ Device:         │    │ Device:         │
│ /dev/dm-0       │    │ /dev/dm-1       │    │ /dev/dm-2       │
│                 │    │                 │    │                 │
│ /dev/mapper/    │    │ /dev/mapper/    │    │ /dev/mapper/    │
│ rhel_...root    │    │ rhel_...swap    │    │ rhel_...home    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        ↓                      ↓                      ↓
    ┌───────┐              ┌───────┐            ┌───────┐
    │  xfs  │              │ swap  │            │  xfs  │
    └───────┘              └───────┘            └───────┘
        │                      │                      │
        ↓                      ↓                      ↓
       /                    [SWAP]                 /home
```

---

## Device Name Mappings

### How All the Names Connect

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE DEVICE NAME MAPPING                             │
└─────────────────────────────────────────────────────────────────────────────┘

Physical Partitions:
════════════════════

┌──────────┬─────────────────┬──────────────────────┬─────────────────────────┐
│ Kernel   │ /sys Location   │ Device File          │ Description             │
│ Name     │                 │                      │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ sda      │ /sys/block/sda  │ /dev/sda             │ Entire physical disk    │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ sda1     │ /sys/block/     │ /dev/sda1            │ First partition (EFI)   │
│          │ sda/sda1        │                      │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ sda2     │ /sys/block/     │ /dev/sda2            │ Second partition (/boot)│
│          │ sda/sda2        │                      │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ sda3     │ /sys/block/     │ /dev/sda3            │ Third partition (LVM PV)│
│          │ sda/sda3        │                      │                         │
└──────────┴─────────────────┴──────────────────────┴─────────────────────────┘


LVM Logical Volumes:
════════════════════

┌──────────┬─────────────────┬──────────────────────┬─────────────────────────┐
│ Kernel   │ /sys Location   │ Device Files         │ Description             │
│ Name     │                 │ (Multiple Names!)    │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ dm-0     │ /sys/block/dm-0 │ /dev/dm-0            │ Root LV                 │
│          │                 │ /dev/mapper/         │ (All 3 names point to   │
│          │                 │ rhel_rhel9exam01-    │ the same device!)       │
│          │                 │ root                 │                         │
│          │                 │ /dev/rhel_rhel9-     │                         │
│          │                 │ exam01/root          │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ dm-1     │ /sys/block/dm-1 │ /dev/dm-1            │ Swap LV                 │
│          │                 │ /dev/mapper/         │ (All 3 names point to   │
│          │                 │ rhel_rhel9exam01-    │ the same device!)       │
│          │                 │ swap                 │                         │
│          │                 │ /dev/rhel_rhel9-     │                         │
│          │                 │ exam01/swap          │                         │
├──────────┼─────────────────┼──────────────────────┼─────────────────────────┤
│ dm-2     │ /sys/block/dm-2 │ /dev/dm-2            │ Home LV                 │
│          │                 │ /dev/mapper/         │ (All 3 names point to   │
│          │                 │ rhel_rhel9exam01-    │ the same device!)       │
│          │                 │ home                 │                         │
│          │                 │ /dev/rhel_rhel9-     │                         │
│          │                 │ exam01/home          │                         │
└──────────┴─────────────────┴──────────────────────┴─────────────────────────┘
```

---

## Complete Filesystem Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FILESYSTEM MOUNT TREE                                │
└─────────────────────────────────────────────────────────────────────────────┘

/ (root)                                  ← dm-0 (xfs, 36GB)
├── bin → usr/bin                         ← Symlink
├── boot/                                 ← sda2 (xfs, 1GB)
│   ├── vmlinuz-5.14.0-...               ← Kernel
│   ├── initramfs-5.14.0-...             ← Initial RAM filesystem
│   └── grub2/                            ← GRUB bootloader config
│       └── grub.cfg
├── boot/efi/                             ← sda1 (vfat, 600MB)
│   └── EFI/
│       ├── BOOT/
│       │   └── BOOTX64.EFI              ← UEFI bootloader
│       └── redhat/
│           ├── shimx64.efi              ← Secure Boot shim
│           └── grubx64.efi              ← GRUB for UEFI
├── dev/                                  ← devtmpfs (virtual)
│   ├── sda                               ← Physical disk
│   ├── sda1, sda2, sda3                 ← Partitions
│   ├── dm-0, dm-1, dm-2                 ← LVM volumes
│   └── mapper/
│       ├── rhel_rhel9exam01-root        ← dm-0 symlink
│       ├── rhel_rhel9exam01-swap        ← dm-1 symlink
│       └── rhel_rhel9exam01-home        ← dm-2 symlink
├── home/                                 ← dm-2 (xfs, 17GB)
│   ├── kanishka/                         ← User home directory
│   │   ├── Documents/
│   │   ├── Downloads/
│   │   └── mini-lsblk.sh                ← Your script!
│   └── lost+found/
├── proc/                                 ← procfs (virtual)
│   ├── meminfo                           ← Memory info
│   ├── mounts                            ← Mounted filesystems
│   └── swaps                             ← Swap devices
├── sys/                                  ← sysfs (virtual)
│   └── block/                            ← Block devices
│       ├── sda/
│       │   ├── size                      ← Disk size
│       │   └── sda1/, sda2/, sda3/      ← Partitions
│       └── dm-0/, dm-1/, dm-2/          ← LVM volumes
├── usr/                                  ← User programs
│   ├── bin/                              ← System binaries
│   │   └── lsblk                        ← Real lsblk command
│   └── local/
│       └── bin/                          ← Your custom scripts!
│           └── mini-lsblk               ← Installed here!
└── var/                                  ← Variable data

[SWAP]                                    ← dm-1 (swap, 4GB)
(Not mounted as filesystem, but active as swap space)
```

---

# How the Script Works

## Data Sources

The script reads from three main sources:

### 1. /sys/block/ - Block Device Information

```bash
# List all block devices
ls /sys/block/
# dm-0  dm-1  dm-2  sda  sr0

# For each device, read size
cat /sys/block/sda/size
# 125829120  (in 512-byte sectors)

# Convert to GB
size_gb=$((125829120 * 512 / 1024 / 1024 / 1024))
# 60

# List partitions
ls /sys/block/sda/sda*
# sda1  sda2  sda3

# Get LVM friendly name
cat /sys/block/dm-0/dm/name
# rhel_rhel9exam01-root
```

---

### 2. /proc/mounts - Filesystem and Mount Points

```bash
# Check filesystem type and mount point
grep sda1 /proc/mounts
# /dev/sda1 /boot/efi vfat rw,relatime,fmask=0077...
#    ↑         ↑       ↑
# Device   Mount pt   FS type

# Example script usage
get_fstype() {
    local device="$1"
    awk -v dev="/dev/$device" '$1 == dev {print $3; exit}' /proc/mounts
}

fstype=$(get_fstype "sda1")
# Returns: vfat
```

---

### 3. /proc/swaps - Swap Device Detection

```bash
# Check if device is swap
cat /proc/swaps
# Filename                Type      Size    Used    Priority
# /dev/dm-1               partition 4141052 1567232 -2

# Example script usage
is_swap() {
    local device="$1"
    grep -q "/dev/$device" /proc/swaps && echo "swap"
}

swap_type=$(is_swap "dm-1")
# Returns: swap
```

---

## Script Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SCRIPT EXECUTION FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

1. List all block devices
   ls /sys/block/
   ↓

2. For each device (e.g., sda):
   a) Read size from /sys/block/sda/size
   b) Convert sectors to GB
   c) Print device line
   ↓

3. For each partition (e.g., sda1):
   a) Read size from /sys/block/sda/sda1/size
   b) Get filesystem type from /proc/mounts
   c) Get mount point from /proc/mounts
   d) Check if it's swap from /proc/swaps
   e) Print partition line with tree character (├─)
   ↓

4. For Device Mapper devices (dm-*):
   a) Read friendly name from /sys/block/dm-0/dm/name
   b) Read size
   c) Get filesystem type
   d) Get mount point
   e) Print LVM volume line
   ↓

5. Display swap devices table
   cat /proc/swaps
   ↓

6. Display mounted filesystems
   cat /proc/mounts | grep ... | column -t
```

---

## Key Functions Explained

### get_fstype()

```bash
get_fstype() {
    local device="$1"
    # Search /proc/mounts for device
    # Extract 3rd field (filesystem type)
    awk -v dev="/dev/$device" '$1 == dev {print $3; exit}' /proc/mounts
}

# Usage example:
fstype=$(get_fstype "sda2")
# Returns: xfs
```

**What it does:**
- Takes device name as input (e.g., "sda2")
- Searches /proc/mounts for line starting with "/dev/sda2"
- Returns the 3rd column (filesystem type)

---

### get_mountpoint()

```bash
get_mountpoint() {
    local device="$1"
    # Search /proc/mounts for device
    # Extract 2nd field (mount point)
    awk -v dev="/dev/$device" '$1 == dev {print $2; exit}' /proc/mounts
}

# Usage example:
mountpoint=$(get_mountpoint "dm-0")
# Returns: /
```

**What it does:**
- Takes device name as input
- Searches /proc/mounts
- Returns the 2nd column (mount point)

---

### is_swap()

```bash
is_swap() {
    local device="$1"
    if [ -f /proc/swaps ]; then
        # Check if device appears in /proc/swaps
        grep -q "/dev/$device" /proc/swaps && echo "swap"
    fi
}

# Usage example:
swap_check=$(is_swap "dm-1")
# Returns: swap (if dm-1 is swap device)
```

**What it does:**
- Checks if device is listed in /proc/swaps
- Returns "swap" if found, empty otherwise

---

# Customization Options

## Modify Output Format

### Change Column Widths

```bash
# In the script, find printf lines like:
printf "%-15s %4dG    disk\n" "$dev_name" "$size_gb"
#       ↑       ↑
#    15 chars  4 digits

# Modify to your preference:
printf "%-20s %6dG    disk\n" "$dev_name" "$size_gb"
#       ↑       ↑
#    20 chars  6 digits (for larger disks)
```

---

### Add Color Output

```bash
# Add color definitions at the top
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Use in output:
printf "${GREEN}%-15s${NC} %4dG    disk\n" "$dev_name" "$size_gb"

# Color swap warnings:
if [ "$fstype" = "swap" ]; then
    printf "${YELLOW}%-15s${NC} %4dG    lvm     ${RED}%-12s${NC} %s\n" \
        "$dm_friendly_name" "$size_gb" "$fstype" "$mountpoint"
fi
```

---

### Filter Specific Devices

```bash
# Skip optical drives
[[ $dev_name =~ ^(loop|ram|sr) ]] && continue

# Only show LVM volumes
if [[ ! $dev_name =~ ^dm- ]]; then
    continue
fi

# Only show mounted filesystems
if [ -z "$mountpoint" ]; then
    continue
fi
```

---

### Add Size in Different Units

```bash
# Add MB column
size_mb=$((size_sectors * 512 / 1024 / 1024))
printf "%-15s %6dMB %4dGB    disk\n" "$dev_name" "$size_mb" "$size_gb"

# Add human-readable with numfmt
size_bytes=$((size_sectors * 512))
size_human=$(numfmt --to=iec-i --suffix=B $size_bytes)
printf "%-15s %-8s    disk\n" "$dev_name" "$size_human"
```

---

## Add Features

### Show Disk Usage Percentage

```bash
# For mounted filesystems, add usage
if [ -n "$mountpoint" ]; then
    usage=$(df "$mountpoint" 2>/dev/null | tail -1 | awk '{print $5}')
    printf "%-15s %4dG    lvm     %-12s %-15s %s\n" \
        "$dm_friendly_name" "$size_gb" "$fstype" "$mountpoint" "$usage"
fi
```

---

### Add Device UUID

```bash
# Read UUID from /dev/disk/by-uuid/
uuid=$(ls -l /dev/disk/by-uuid/ | grep "$dev_name" | awk '{print $9}')
printf "%-15s %4dG    disk    UUID: %s\n" "$dev_name" "$size_gb" "$uuid"
```

---

### Add Command-Line Options

```bash
# Add at the beginning of script
while getopts "hvs" opt; do
    case $opt in
        h) show_help; exit 0 ;;
        v) VERBOSE=1 ;;
        s) SHOW_STATS=1 ;;
        *) exit 1 ;;
    esac
done

# Use in script
if [ "$VERBOSE" = "1" ]; then
    echo "Reading from /sys/block/..."
fi
```

---

# Troubleshooting

## Common Issues and Solutions

### Issue 1: Permission Denied

```bash
# Error:
./mini-lsblk.sh: Permission denied

# Solution:
chmod +x mini-lsblk.sh

# Verify:
ls -l mini-lsblk.sh
# -rwxr-xr-x. 1 user user ... mini-lsblk.sh
```

---

### Issue 2: Command Not Found After Installation

```bash
# Error:
mini-lsblk: command not found

# Check if installed:
ls -l /usr/local/bin/mini-lsblk

# Check PATH:
echo $PATH | grep -o /usr/local/bin
# Should output: /usr/local/bin

# If not in PATH, add it:
export PATH="/usr/local/bin:$PATH"

# Make permanent:
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

### Issue 3: Empty Output

```bash
# Check if /sys/block exists:
ls /sys/block/
# Should show: dm-0  dm-1  dm-2  sda  sr0

# Check permissions:
ls -ld /sys/block
# dr-xr-xr-x. ... /sys/block

# If empty, kernel may not support sysfs:
mount | grep sysfs
# sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
```

---

### Issue 4: Wrong Sizes Displayed

```bash
# Verify sector size assumption (script assumes 512 bytes)
cat /sys/block/sda/queue/hw_sector_size
# 512

# If different (e.g., 4096 for advanced format drives):
# Modify script calculation:
sector_size=$(cat /sys/block/$dev_name/queue/hw_sector_size)
size_gb=$((size_sectors * sector_size / 1024 / 1024 / 1024))
```

---

### Issue 5: Missing LVM Information

```bash
# Check if /sys/block/dm-0 exists:
ls /sys/block/dm-*

# Check if dm/name file exists:
cat /sys/block/dm-0/dm/name
# rhel_rhel9exam01-root

# If missing, LVM may not be active:
sudo vgscan
sudo vgchange -ay

# Verify LVM is running:
sudo lvs
```

---

## Debugging Tips

### Enable Verbose Output

```bash
# Add debug statements in script:
set -x  # Enable debug mode
# Your script here
set +x  # Disable debug mode

# Or run with bash -x:
bash -x mini-lsblk.sh
```

---

### Check Individual Components

```bash
# Test /sys/block reading:
for dev in /sys/block/*; do
    echo "Device: $(basename $dev)"
    cat $dev/size 2>/dev/null || echo "No size file"
done

# Test /proc/mounts parsing:
grep sda1 /proc/mounts

# Test /proc/swaps:
cat /proc/swaps
```

---

### Compare with Real lsblk

```bash
# Run both and compare:
mini-lsblk > /tmp/mini.txt
lsblk -f > /tmp/real.txt
diff /tmp/mini.txt /tmp/real.txt
```

---

## Getting Help

### Check System Logs

```bash
# Look for errors:
sudo journalctl -xe | grep -i block

# Check dmesg for device issues:
sudo dmesg | grep -i sda
```

---

### Verify Kernel Support

```bash
# Check if necessary filesystems are mounted:
mount | grep -E "proc|sys"
# proc on /proc type proc ...
# sysfs on /sys type sysfs ...

# Check kernel version:
uname -r
# 5.14.0-70.el9.x86_64
```

---

## Next Steps

After mastering this script, consider:

1. **Add More Features:**
   - Disk I/O statistics from `/sys/block/*/stat`
   - SMART data reading
   - Partition alignment checking
   - RAID information

2. **Learn Related Tools:**
   - `fdisk` - Partition manipulation
   - `parted` - Advanced partitioning
   - `lvdisplay` - LVM details
   - `blkid` - Block device attributes

3. **Create More Scripts:**
   - `mini-df` - Disk free space
   - `mini-free` - Memory statistics
   - `mini-top` - Process monitoring

4. **Contribute:**
   - Share on GitHub
   - Add documentation
   - Create issues/PRs
   - Help others learn

---

## Additional Resources

### Official Documentation

- **RHEL 9 Documentation:** https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9
- **LVM Administration Guide:** https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/
- **Linux Kernel Documentation:** https://www.kernel.org/doc/html/latest/

### Man Pages

```bash
man lsblk
man proc       # /proc filesystem
man sysfs      # /sys filesystem
man lvm        # LVM overview
man mount      # Mount command and /etc/fstab
```

### Online Resources

- **Linux Documentation Project:** https://tldp.org/
- **Arch Wiki (excellent for all distros):** https://wiki.archlinux.org/
- **Stack Overflow:** https://stackoverflow.com/questions/tagged/linux

---

## License

This script and documentation are provided under the MIT License.

```
MIT License

Copyright (c) 2026 [Your Name]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## Conclusion

You've now created a working `lsblk` clone that demonstrates:

✅ How Linux commands read from `/proc` and `/sys`  
✅ Complete understanding of block device architecture  
✅ LVM structure from physical volumes to filesystems  
✅ Device naming and mapping in Linux  
✅ Professional script development and installation  

**Key Takeaway:** Linux commands aren't magic—they're just programs reading from special files that the kernel provides!

---

## Acknowledgments

- Based on hands-on RHEL 9 learning sessions
- Inspired by the real `lsblk` from util-linux package
- Thanks to the Linux kernel developers for excellent /proc and /sys documentation
- Community feedback and contributions

---

**Happy Learning!** 🚀

*Last Updated: March 2026*  
*Version: 1.0*  
*System: RHEL 9 (rhel9exam01)*
