# How Linux Commands Really Work - Under the Hood

**A Deep Dive into Linux Command Architecture, Virtual Filesystems, and System Information**

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Core Concept](#the-core-concept)
3. [Virtual Filesystems: /proc and /sys](#virtual-filesystems-proc-and-sys)
4. [How Commands Get Information](#how-commands-get-information)
5. [Command Types and Locations](#command-types-and-locations)
6. [Understanding /bin vs /usr/bin](#understanding-bin-vs-usrbin)
7. [The PATH Environment Variable](#the-path-environment-variable)
8. [Installing Custom Scripts](#installing-custom-scripts)
9. [Building Your Own Commands](#building-your-own-commands)
10. [System Calls vs File Reading](#system-calls-vs-file-reading)
11. [Hands-On Demonstrations](#hands-on-demonstrations)
12. [Best Practices](#best-practices)
13. [Command Reference](#command-reference)

---

# Introduction

Ever wondered how Linux commands like `lsblk`, `free`, or `df` actually work? Are they magic? Do they have special access to hidden system information?

**The answer might surprise you:** Most Linux commands are simply programs that read from special files and system interfaces where the kernel stores information. They're sophisticated, but not magical!

This guide explains:
- How commands access system information
- What `/proc` and `/sys` really are
- The difference between `/bin` and `/usr/bin`
- How to create and install your own commands
- The complete command execution flow

---

# The Core Concept

## Everything is a File

```
Unix/Linux Philosophy: "Everything is a file"

This means:
├─ Hardware devices → Files (/dev/sda)
├─ Process information → Files (/proc/PID/)
├─ System statistics → Files (/proc/meminfo)
├─ Device information → Files (/sys/block/)
└─ Even network sockets → Files

Benefits:
✅ Simple, uniform interface
✅ Standard tools work everywhere (cat, grep, etc.)
✅ Easy to script
✅ Extensible by kernel
```

---

## How Commands Work: The Simple Answer

```
Question: How does 'lsblk' show swap?

Answer: It reads from files where the kernel stores this information!

Flow:
1. You type: lsblk
2. Shell finds: /usr/bin/lsblk (binary program)
3. lsblk reads:
   ├─ /sys/block/*     (device info)
   ├─ /proc/swaps      (swap detection)
   └─ /proc/partitions (partition table)
4. lsblk formats data
5. Displays output

That's it! No magic.
```

---

# Virtual Filesystems: /proc and /sys

## What Are Virtual Filesystems?

```
Virtual filesystem = Filesystem that exists only in RAM

Characteristics:
├─ Not stored on disk
├─ Created by kernel at boot
├─ Updated in real-time
├─ Generated on-the-fly when read
└─ Zero disk usage

Examples:
/proc   → Process and kernel information
/sys    → Device and hardware information
```

---

## /proc - Process Filesystem

### Overview

```
/proc = procfs (Process Filesystem)

Purpose:
├─ Process information (per-process directories)
├─ Kernel statistics
├─ System configuration
└─ Hardware information

Location: Virtual (RAM only)
Created by: Kernel
Updated: Real-time
```

---

### Key /proc Files

```bash
# Memory information
/proc/meminfo
├─ MemTotal, MemFree, MemAvailable
├─ Buffers, Cached
├─ SwapTotal, SwapFree, SwapCached
└─ Used by: free, top, vmstat

# CPU information  
/proc/cpuinfo
├─ Processor model, speed, cores
├─ Cache sizes
└─ Used by: lscpu, top

# Mounted filesystems
/proc/mounts
├─ Currently mounted filesystems
├─ Mount points and options
└─ Used by: mount, df, findmnt

# Active swap devices
/proc/swaps
├─ Swap device, type, size, usage
└─ Used by: swapon, lsblk

# Partition table
/proc/partitions
├─ Major/minor numbers
├─ Block counts, device names
└─ Used by: lsblk, fdisk

# System load
/proc/loadavg
├─ 1, 5, 15 minute load averages
└─ Used by: uptime, top

# Uptime
/proc/uptime
├─ System uptime in seconds
└─ Used by: uptime

# Per-process information
/proc/[PID]/
├─ cmdline  → Command line
├─ environ  → Environment variables
├─ fd/      → Open file descriptors
├─ maps     → Memory mappings
├─ stat     → Process statistics
├─ status   → Human-readable status
└─ Used by: ps, top, pgrep
```

---

### Example: Reading /proc/meminfo

```bash
# What 'free' command reads:
cat /proc/meminfo

# Output:
MemTotal:        7864532 kB
MemFree:          738220 kB
MemAvailable:    2927760 kB
Buffers:            3444 kB
Cached:          2600280 kB
SwapCached:       123456 kB
SwapTotal:       4096000 kB
SwapFree:        2528256 kB
...

# This is exactly what 'free -h' reads and formats!
```

---

### Verify /proc is Virtual

```bash
# Check size
df -h /proc

# Output:
Filesystem      Size  Used Avail Use% Mounted on
proc              0     0     0    -  /proc
                  ↑
              Zero size! (virtual, not on disk)

# Check type
mount | grep proc
# proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
#                     ↑
#                 Type = proc (virtual)
```

---

## /sys - System Device Filesystem

### Overview

```
/sys = sysfs (System Filesystem)

Purpose:
├─ Hardware device information
├─ Device drivers
├─ Kernel modules
├─ Device hierarchy
└─ Hardware attributes

Location: Virtual (RAM only)
Created by: Kernel
Updated: Real-time
```

---

### Key /sys Directories

```bash
# Block devices
/sys/block/
├─ sda/      → First disk
├─ sdb/      → Second disk
├─ dm-0/     → Device mapper devices (LVM)
└─ Used by: lsblk, fdisk

# Device classes
/sys/class/
├─ net/      → Network interfaces
├─ block/    → Block devices
├─ input/    → Input devices
└─ Used by: ip, lsblk

# Devices by path
/sys/devices/
├─ Complete device hierarchy
├─ pci0000:00/  → PCI devices
└─ Used by: lspci, lsusb

# Firmware
/sys/firmware/
├─ efi/      → UEFI variables
└─ Used by: efibootmgr
```

---

### Example: Reading /sys/block

```bash
# List block devices
ls /sys/block/
# sda  sr0

# Check sda details
ls /sys/block/sda/
# device  holders  queue  size  stat  ...

# Get disk size (in 512-byte sectors)
cat /sys/block/sda/size
# 125829120

# Calculate GB
echo $(($(cat /sys/block/sda/size) * 512 / 1024 / 1024 / 1024))
# 60

# This is what lsblk does!

# List partitions
ls /sys/block/sda/
# sda1  sda2  sda3

# Get partition size
cat /sys/block/sda/sda1/size
# 1228800
```

---

### Verify /sys is Virtual

```bash
# Check size
df -h /sys

# Output:
Filesystem      Size  Used Avail Use% Mounted on
sysfs             0     0     0    -  /sys
                  ↑
              Zero size! (virtual)

# Check type
mount | grep sysfs
# sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
#                    ↑
#                Type = sysfs (virtual)
```

---

# How Commands Get Information

## Two Primary Methods

### Method 1: Reading Files (Most Common)

```
Commands read from /proc and /sys:

Examples:
├─ free   → reads /proc/meminfo
├─ lsblk  → reads /sys/block/* and /proc/swaps
├─ mount  → reads /proc/mounts
├─ ps     → reads /proc/[PID]/*
└─ uptime → reads /proc/uptime and /proc/loadavg

Advantages:
✅ Simple interface
✅ Standard file operations
✅ Easy to debug
✅ Scriptable
```

---

### Method 2: System Calls (Direct Kernel Interface)

```
Commands make system calls to kernel:

Examples:
├─ df       → statfs() syscall
├─ uname    → uname() syscall
├─ ip addr  → netlink sockets
└─ hostname → gethostname() syscall

Advantages:
✅ More efficient
✅ Atomic operations
✅ Type-safe
✅ Kernel-enforced security
```

---

## Command Information Sources

| Command | Information Source | Method |
|---------|-------------------|--------|
| `free` | `/proc/meminfo` | File read |
| `lsblk` | `/sys/block/*`, `/proc/swaps` | File read |
| `df` | `/proc/mounts` + `statfs()` | Both |
| `mount` | `/proc/mounts` | File read |
| `ps` | `/proc/[PID]/*` | File read |
| `top` | `/proc/[PID]/*`, `/proc/stat` | File read |
| `uptime` | `/proc/uptime`, `/proc/loadavg` | File read |
| `lscpu` | `/proc/cpuinfo`, `/sys/devices/system/cpu/` | File read |
| `uname` | `uname()` syscall | System call |
| `hostname` | `gethostname()` syscall | System call |
| `ip addr` | Netlink sockets | System call |

---

# Command Types and Locations

## Four Types of Commands

### 1. Binary Executables (Compiled Programs)

```bash
# Most system commands
type lsblk
# lsblk is /usr/bin/lsblk

file /usr/bin/lsblk
# /usr/bin/lsblk: ELF 64-bit LSB pie executable, x86-64...

# Compiled from C/C++ or other languages
# Fast execution
# Machine code
```

---

### 2. Shell Built-ins (Part of Shell)

```bash
# Commands built into bash/shell
type cd
# cd is a shell builtin

type echo
# echo is a shell builtin

type pwd
# pwd is a shell builtin

# Why built-in?
# - cd changes shell's working directory (can't be external)
# - echo, pwd faster as built-ins (no process creation)
```

---

### 3. Shell Scripts (Interpreted)

```bash
# Text files with commands
type startx
# startx is /usr/bin/startx

file /usr/bin/startx
# /usr/bin/startx: POSIX shell script, ASCII text executable

# Advantages:
# - Easy to modify
# - Portable
# - No compilation needed
```

---

### 4. Aliases and Functions

```bash
# Shortcuts defined by user
type ll
# ll is aliased to `ls -l --color=auto'

# Define alias
alias myls='ls -lah'

# Define function
myfunction() {
    echo "Hello from function"
}

type myfunction
# myfunction is a function
```

---

## Where Commands Live

### Standard Binary Locations

```
/usr/bin/              # Most user commands
├─ ls, cat, grep, vim
├─ python3, gcc, git
└─ ~1000+ commands

/usr/sbin/             # System administration commands
├─ fdisk, mkfs, useradd
├─ iptables, systemctl
└─ Requires root typically

/usr/local/bin/        # Locally installed software
├─ Custom scripts (YOUR SCRIPTS!)
├─ Compiled from source
└─ Higher priority in PATH

/bin → /usr/bin        # Symlink (compatibility)
/sbin → /usr/sbin      # Symlink (compatibility)

~/bin/                 # User-specific commands
└─ Personal scripts
```

---

### Finding Commands

```bash
# Find command location
which lsblk
# /usr/bin/lsblk

# Show all locations
type -a python3
# python3 is /usr/bin/python3
# python3 is /bin/python3

# Check command type
type lsblk
# lsblk is /usr/bin/lsblk

# Get file information
file /usr/bin/lsblk
# /usr/bin/lsblk: ELF 64-bit LSB pie executable...
```

---

# Understanding /bin vs /usr/bin

## Modern RHEL 9: They're the Same!

### The Merge

```bash
# Check what /bin really is
ls -ld /bin
# lrwxrwxrwx. 1 root root 7 ... /bin -> usr/bin
#  ↑
# It's a SYMLINK to /usr/bin!

# Verify
readlink /bin
# usr/bin

# They point to the same location
ls -i /bin/ls /usr/bin/ls
# 12345 /bin/ls
# 12345 /usr/bin/ls
#   ↑
# Same inode = same file!

# All merged
/bin    → /usr/bin      (symlink)
/sbin   → /usr/sbin     (symlink)
/lib    → /usr/lib      (symlink)
/lib64  → /usr/lib64    (symlink)
```

---

## Historical Context (Pre-2012)

### Why Two Directories Existed

```
Old UNIX/Linux (1970s-2000s):
================================

/bin - Essential Binaries
├─ Commands needed to boot system
├─ Available before /usr mounted
├─ Minimal set for system repair
├─ Examples: ls, cat, cp, mv, rm, bash, sh
└─ Small, on root filesystem

/usr/bin - User Binaries
├─ Normal user programs
├─ Could be on separate partition
├─ Could be network-mounted (NFS)
├─ Larger collection
├─ Examples: vim, gcc, python, wget
└─ Mounted later in boot

Reason for split:
├─ Disk space was expensive
├─ /usr could be on network
├─ Boot from minimal root partition
└─ Mount /usr later
```

---

## Why the Merge? (Fedora 17+, RHEL 7+)

### Problems with Old Split

```
Issues:
❌ Confusing (where does X go?)
❌ Dependencies got messy
❌ Modern systems: /usr available early
❌ systemd needs many tools early
❌ Maintaining separation was pointless

Solution:
✅ Merge everything into /usr/bin
✅ Make /bin → /usr/bin symlink
✅ Backward compatibility maintained
✅ Simpler to understand
✅ Easier to maintain
```

---

### Verify the Merge

```bash
# Check symlinks
ls -l / | grep "^l" | grep bin
# lrwxrwxrwx. bin -> usr/bin
# lrwxrwxrwx. sbin -> usr/sbin
# lrwxrwxrwx. lib -> usr/lib
# lrwxrwxrwx. lib64 -> usr/lib64

# Count files (should be identical)
ls /bin | wc -l
# 1234

ls /usr/bin | wc -l
# 1234

# Compare contents
diff <(ls /bin | sort) <(ls /usr/bin | sort)
# (No output = identical)

# Both paths work
/bin/ls --version
/usr/bin/ls --version
# Same program!
```

---

## Complete Directory Structure

```
Modern RHEL 9 Binary Locations:

/usr/bin/                    # ALL binaries (merged)
├─ System binaries (former /bin)
├─ User programs (former /usr/bin)
└─ Everything together

/bin → /usr/bin              # Symlink (compatibility)
/sbin → /usr/sbin            # Symlink (compatibility)

/usr/sbin/                   # System admin tools
├─ fdisk, mkfs, useradd
└─ Root privileges typically needed

/usr/local/bin/              # Local installations
├─ Custom scripts (YOURS!)
├─ Compiled from source
├─ Software not in repos
└─ Higher priority in PATH

~/bin/                       # User-specific
├─ Personal scripts
└─ Only for your user

/opt/[app]/bin/              # Third-party apps
├─ Self-contained applications
└─ Each app in own directory
```

---

# The PATH Environment Variable

## What is PATH?

```
PATH = List of directories to search for commands

Format:
directory1:directory2:directory3:...

Purpose:
├─ Shell searches these directories in order
├─ When you type a command name
├─ First match wins
└─ Determines which command executes
```

---

## Viewing Your PATH

```bash
# View PATH
echo $PATH
# /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/user/.local/bin:/home/user/bin

# Pretty print (one per line)
echo $PATH | tr ':' '\n'
# /usr/local/bin
# /usr/bin
# /usr/local/sbin
# /usr/sbin
# /home/user/.local/bin
# /home/user/bin

# Typical order (priority):
1. /usr/local/bin      ← Checked FIRST (highest priority)
2. /usr/bin            ← System binaries
3. /usr/local/sbin     ← Local admin tools
4. /usr/sbin           ← System admin tools
5. ~/.local/bin        ← User Python packages
6. ~/bin               ← Personal scripts
```

---

## How Shell Finds Commands

```
When you type: mini-lsblk

Shell searches in order:

1. Check if it's a built-in (cd, echo, etc.)
   → No

2. Check if it's an alias or function
   → No

3. Search PATH directories in order:
   a. /usr/local/bin/mini-lsblk  → Found! ✓
      Execute this one
   
   (If not found, continue...)
   b. /usr/bin/mini-lsblk        → Not found
   c. /usr/local/sbin/mini-lsblk → Not found
   d. /usr/sbin/mini-lsblk       → Not found
   e. ~/.local/bin/mini-lsblk    → Not found
   f. ~/bin/mini-lsblk           → Not found

4. If not found anywhere:
   bash: mini-lsblk: command not found
```

---

## Which Command Runs?

```bash
# See which command will execute
which ls
# /usr/bin/ls

# Show all locations
type -a python3
# python3 is /usr/bin/python3
# python3 is /bin/python3
#  ↑
# First one wins (actually same file via symlink)

# Example: Multiple pythons
type -a python
# python is /usr/local/bin/python  ← This runs (first in PATH)
# python is /usr/bin/python

# Find specific command
command -v mini-lsblk
# /usr/local/bin/mini-lsblk
```

---

## Modifying PATH

### Temporarily (Current Session)

```bash
# Add directory to beginning (highest priority)
export PATH="/opt/myapp/bin:$PATH"

# Add directory to end (lowest priority)
export PATH="$PATH:/opt/myapp/bin"

# Verify
echo $PATH | tr ':' '\n'

# This lasts only until you logout
```

---

### Permanently (All Sessions)

```bash
# For your user only (~/.bashrc)
echo 'export PATH="/opt/myapp/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# For all users (system-wide)
echo 'export PATH="/opt/myapp/bin:$PATH"' | sudo tee /etc/profile.d/myapp.sh

# Verify in new shell
bash -c 'echo $PATH'
```

---

# Installing Custom Scripts

## Installation Hierarchy

```
Where to install your scripts:

Personal use (only you):
└─ ~/bin/myscript
   ✅ No sudo needed
   ✅ Easy to manage
   ✅ Only your user
   ❌ Not available to others

System-wide (all users):
└─ /usr/local/bin/myscript
   ✅ Standard location
   ✅ Survives updates
   ✅ Won't conflict with packages
   ✅ All users can access

Package-managed:
└─ /usr/bin/program
   ⚠️ Managed by DNF/RPM
   ⚠️ Don't manually add here
   ⚠️ May be overwritten

Large applications:
└─ /opt/myapp/bin/myapp
   ✅ Self-contained
   ✅ Easy to uninstall
   ✅ Third-party software
```

---

## Method 1: Install to /usr/local/bin (Recommended)

```bash
# Create script
cat > ~/myscript.sh << 'EOF'
#!/bin/bash
echo "Hello from myscript!"
EOF

# Make executable
chmod +x ~/myscript.sh

# Install (with install command - better than cp)
sudo install -m 755 ~/myscript.sh /usr/local/bin/myscript
#      ↑         ↑                                  ↑
#   install    perms                      Remove .sh extension

# Verify
ls -l /usr/local/bin/myscript
# -rwxr-xr-x. 1 root root ... /usr/local/bin/myscript

# Test
which myscript
# /usr/local/bin/myscript

# Run from anywhere
cd /tmp
myscript
# Hello from myscript!
```

---

## Method 2: Install to ~/bin (User-Specific)

```bash
# Create ~/bin if it doesn't exist
mkdir -p ~/bin

# Copy script
cp ~/myscript.sh ~/bin/myscript
chmod 755 ~/bin/myscript

# Check if ~/bin is in PATH
echo $PATH | grep "$HOME/bin"

# If not, add to PATH
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Test
which myscript
# /home/user/bin/myscript

# Works (only for you)
myscript
```

---

## Method 3: Create Custom Directory

```bash
# Create custom scripts directory
sudo mkdir -p /opt/myscripts

# Install script
sudo install -m 755 ~/myscript.sh /opt/myscripts/myscript

# Add to system-wide PATH
echo 'export PATH="/opt/myscripts:$PATH"' | sudo tee /etc/profile.d/myscripts.sh

# Apply immediately
source /etc/profile.d/myscripts.sh

# Test
which myscript
# /opt/myscripts/myscript
```

---

## Best Practices

### File Permissions

```bash
# Correct permissions for scripts:

# Standard (executable by all, writable by root)
-rwxr-xr-x  root root  /usr/local/bin/myscript
 ↑  ↑  ↑
 │  │  └─ Others: read, execute
 │  └─ Group: read, execute
 └─ Owner: read, write, execute

# Set with:
sudo chmod 755 /usr/local/bin/myscript

# Restrictive (root only)
sudo chmod 700 /usr/local/bin/admin-script

# NEVER do this (security risk):
sudo chmod 777 /usr/local/bin/script  # ❌ Anyone can modify!
```

---

### Script Headers

```bash
# Always include shebang
#!/bin/bash

# Or for Python
#!/usr/bin/env python3

# For portability
#!/usr/bin/env bash

# Good practice:
#!/bin/bash
set -euo pipefail  # Exit on error, undefined vars, pipe failures

# Your script here
```

---

### Naming Conventions

```bash
# Good names:
backup-system       # Descriptive, hyphenated
check-disk-space    # Clear purpose
mini-lsblk          # Clear what it does

# Avoid:
script.sh           # Too generic
my_script           # Underscores less common
123test             # Starting with numbers
```

---

## Package Manager Awareness

### What Package Manager Controls

```bash
# Check which package owns a file
rpm -qf /usr/bin/lsblk
# util-linux-2.37.4-18.el9.x86_64

# Try with your custom script
rpm -qf /usr/local/bin/myscript
# file /usr/local/bin/myscript is not owned by any package
#  ↑
# Good! Not managed by RPM

# List all package-managed files in /usr/bin
rpm -qal | grep "^/usr/bin/" | wc -l
# 1234

# List files in /usr/local (should be empty)
rpm -qal | grep "^/usr/local/"
# (No output - DNF/RPM never touches /usr/local)
```

---

# Building Your Own Commands

## Example: Building mini-lsblk

### Complete Script

```bash
#!/bin/bash
# mini-lsblk - Simplified lsblk implementation
# Demonstrates reading from /sys and /proc

echo "NAME    SIZE    TYPE    FSTYPE    MOUNTPOINT"
echo "=================================================="

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
    
    echo "$dev_name    ${size_gb}G    disk"
    
    # List partitions
    for part in "$device/$dev_name"*; do
        if [ -d "$part" ]; then
            part_name=$(basename "$part")
            
            # Get partition size
            if [ -f "$part/size" ]; then
                part_size=$(cat "$part/size" 2>/dev/null || echo 0)
                part_gb=$((part_size * 512 / 1024 / 1024 / 1024))
            else
                part_gb=0
            fi
            
            # Check filesystem type
            fstype="unknown"
            if [ -f "$part/uevent" ]; then
                fstype=$(lsblk -n -o FSTYPE "/dev/$part_name" 2>/dev/null || echo "")
            fi
            
            # Check mount point
            mountpoint=$(lsblk -n -o MOUNTPOINT "/dev/$part_name" 2>/dev/null || echo "")
            
            echo "├─$part_name    ${part_gb}G    part    $fstype    $mountpoint"
        fi
    done
done

# Check for swap from /proc/swaps
echo ""
echo "SWAP DEVICES:"
if [ -f /proc/swaps ]; then
    cat /proc/swaps
else
    echo "No swap information available"
fi
```

---

### Install and Use

```bash
# Save script
cat > ~/mini-lsblk.sh << 'EOF'
[paste script above]
EOF

# Make executable
chmod +x ~/mini-lsblk.sh

# Test
~/mini-lsblk.sh

# Install system-wide
sudo install -m 755 ~/mini-lsblk.sh /usr/local/bin/mini-lsblk

# Use like any command
mini-lsblk
```

---

## How It Works

```
mini-lsblk demonstrates:

1. Reading /sys/block/* for device info
   ├─ Device names
   ├─ Sizes (in sectors)
   └─ Partition list

2. Reading /proc/swaps for swap detection
   └─ Active swap devices

3. Using lsblk for some info
   ├─ Filesystem types
   └─ Mount points

4. Formatting output
   └─ Presenting in readable format

This is essentially what the real lsblk does,
just more sophisticated!
```

---

# System Calls vs File Reading

## Using strace to See What Commands Do

### Example: Trace 'free' Command

```bash
# Trace system calls and file opens
strace -e trace=open,openat,read free -h 2>&1 | head -20

# Key lines you'll see:
openat(AT_FDCWD, "/proc/meminfo", O_RDONLY) = 3
read(3, "MemTotal:        7864532 kB\nMemFree...") = 1391
#  ↑                                            ↑
# File descriptor 3                        Bytes read

# free is reading /proc/meminfo!
```

---

### Example: Trace 'lsblk' Command

```bash
# See what lsblk accesses
strace -e trace=open,openat lsblk 2>&1 | grep -E "/sys/|/proc/" | head -10

# You'll see:
openat(AT_FDCWD, "/sys/block", O_RDONLY|O_DIRECTORY) = 3
openat(AT_FDCWD, "/sys/block/sda", O_RDONLY|O_DIRECTORY) = 4
openat(AT_FDCWD, "/sys/block/sda/size", O_RDONLY) = 5
openat(AT_FDCWD, "/proc/swaps", O_RDONLY) = 6
#                 ↑
# Reading /sys and /proc to get device info!
```

---

### Example: Trace 'df' Command

```bash
# df uses both files and system calls
strace df / 2>&1 | grep -E "statfs|/proc/mounts"

# You'll see:
openat(AT_FDCWD, "/proc/mounts", O_RDONLY) = 3
#                 ↑
# Reading mount points from /proc

statfs("/", {f_type=XFS_SUPER_MAGIC, f_bsize=4096, ...})
#       ↑
# Using statfs() system call to get filesystem stats
```

---

## Library Call Tracing (ltrace)

```bash
# Trace library function calls
ltrace free -h 2>&1 | grep -E "fopen|fread" | head -5

# Shows:
fopen("/proc/meminfo", "r") = 0x...
fread(...) = 1391
#  ↑
# Using fopen() to open /proc/meminfo
```

---

# Hands-On Demonstrations

## Explore /proc Yourself

### Process Information

```bash
# Your shell's process ID
echo $$
# 12345

# Explore your shell's info
ls /proc/$$/
# cmdline  cwd  environ  exe  fd  maps  stat  status  ...

# Command line
cat /proc/$$/cmdline
# /bin/bash

# Working directory
ls -l /proc/$$/cwd
# lrwxrwxrwx ... /proc/12345/cwd -> /home/user

# Memory usage
cat /proc/$$/status | grep -E "VmSize|VmRSS"
# VmSize:    12345 kB  (virtual memory)
# VmRSS:      6789 kB  (resident/physical memory)

# Open files
ls -l /proc/$$/fd/
# 0 -> /dev/pts/0  (stdin)
# 1 -> /dev/pts/0  (stdout)
# 2 -> /dev/pts/0  (stderr)

# This is what ps and top read!
```

---

### System Information

```bash
# CPU info
cat /proc/cpuinfo | head -20

# Memory info
cat /proc/meminfo | head -10

# Mount points
cat /proc/mounts

# Swap devices
cat /proc/swaps

# Load average
cat /proc/loadavg
# 0.15 0.18 0.12 1/234 12345
#  ↑    ↑    ↑   ↑ ↑   ↑
# 1min 5min 15m r/t  last PID

# Uptime
cat /proc/uptime
# 123456.78 987654.32
#    ↑         ↑
# uptime    idle time (in seconds)
```

---

## Explore /sys Yourself

### Block Devices

```bash
# List block devices
ls /sys/block/
# sda  sr0

# Explore sda
ls /sys/block/sda/
# device  holders  power  queue  size  slaves  stat  ...

# Device size (in 512-byte sectors)
cat /sys/block/sda/size
# 125829120

# Calculate actual size
echo $(($(cat /sys/block/sda/size) * 512 / 1024 / 1024 / 1024))
# 60  (60GB)

# Check if it's removable
cat /sys/block/sda/removable
# 0  (not removable)

# Read statistics
cat /sys/block/sda/stat
# read I/Os, read sectors, write I/Os, write sectors, etc.
```

---

### Network Devices

```bash
# List network interfaces
ls /sys/class/net/
# ens33  lo

# Interface details
ls /sys/class/net/ens33/
# address  carrier  duplex  mtu  operstate  speed  statistics/  ...

# MAC address
cat /sys/class/net/ens33/address
# 00:0c:29:12:34:56

# Link status
cat /sys/class/net/ens33/operstate
# up

# Speed
cat /sys/class/net/ens33/speed
# 1000  (1000 Mbps = 1 Gbps)
```

---

## Recreate Simple Commands

### Recreate 'uptime'

```bash
#!/bin/bash
# mini-uptime - Simple uptime implementation

# Read uptime
read uptime idle < /proc/uptime

# Read load average
read load1 load5 load15 rest < /proc/loadavg

# Convert uptime to readable format
days=$((${uptime%.*} / 86400))
hours=$(((${uptime%.*} % 86400) / 3600))
minutes=$(((${uptime%.*} % 3600) / 60))

# Display
echo "up $days days, $hours:$minutes, load average: $load1, $load5, $load15"

# Compare to real uptime
# uptime
```

---

### Recreate 'free'

```bash
#!/bin/bash
# mini-free - Simple free implementation

# Read meminfo
while IFS=: read key value; do
    case "$key" in
        MemTotal) mem_total=${value//[^0-9]/} ;;
        MemFree) mem_free=${value//[^0-9]/} ;;
        MemAvailable) mem_avail=${value//[^0-9]/} ;;
        Buffers) buffers=${value//[^0-9]/} ;;
        Cached) cached=${value//[^0-9]/} ;;
        SwapTotal) swap_total=${value//[^0-9]/} ;;
        SwapFree) swap_free=${value//[^0-9]/} ;;
    esac
done < /proc/meminfo

# Calculate used
mem_used=$((mem_total - mem_free))
swap_used=$((swap_total - swap_free))

# Display (in MB)
echo "             total       used       free     available"
echo "Mem:    $((mem_total/1024))    $((mem_used/1024))    $((mem_free/1024))    $((mem_avail/1024))"
echo "Swap:   $((swap_total/1024))    $((swap_used/1024))    $((swap_free/1024))"

# Compare to real free
# free -m
```

---

# Best Practices

## Script Development

```bash
# 1. Always include shebang
#!/bin/bash

# 2. Set strict mode (exit on errors)
set -euo pipefail

# 3. Add description
# Script: backup-system
# Purpose: Automated system backup
# Author: Your Name
# Date: 2026-03-17

# 4. Use functions for organization
main() {
    check_requirements
    perform_backup
    cleanup
}

check_requirements() {
    # Check dependencies
    command -v rsync >/dev/null || { echo "rsync required"; exit 1; }
}

# 5. Call main function
main "$@"
```

---

## Installation Best Practices

```
Recommended Locations:

System-wide scripts:
├─ /usr/local/bin/        ← BEST for custom scripts
├─ Pros: Standard, survives updates, all users
└─ Use: sudo install -m 755 script.sh /usr/local/bin/script

Personal scripts:
├─ ~/bin/                 ← BEST for personal use
├─ Pros: No sudo, easy to manage
└─ Use: cp script.sh ~/bin/script && chmod 755 ~/bin/script

Avoid:
├─ /usr/bin/              ← Managed by package manager
└─ May be overwritten during updates

Large applications:
├─ /opt/myapp/            ← Good for third-party apps
└─ Self-contained, easy to uninstall
```

---

## Naming and Permissions

```bash
# Good script names:
check-disk-space
backup-database
monitor-logs

# Good permissions:
-rwxr-xr-x  (755)  # Standard
-rwxr-x---  (750)  # Group only
-rwx------  (700)  # Owner only

# Set permissions:
chmod 755 /usr/local/bin/myscript

# Set ownership (if needed):
sudo chown root:root /usr/local/bin/myscript
```

---

## Documentation

```bash
# Include help text
if [[ "${1:-}" == "-h" ]] || [[ "${1:-}" == "--help" ]]; then
    cat << EOF
Usage: $(basename "$0") [OPTIONS]

Description:
    Brief description of what the script does

Options:
    -h, --help     Show this help message
    -v, --verbose  Verbose output

Examples:
    $(basename "$0") --verbose

EOF
    exit 0
fi
```

---

# Command Reference

## Essential Commands for Exploring

```bash
# Find command location
which command_name
type command_name
command -v command_name

# Show all locations
type -a command_name

# Check command type
type command_name
file /usr/bin/command_name

# Trace system calls
strace command_name
strace -e trace=open,read command_name

# Trace library calls
ltrace command_name

# View PATH
echo $PATH
echo $PATH | tr ':' '\n'

# Find files owned by packages
rpm -qf /path/to/file
rpm -qal | grep pattern

# Explore /proc
ls /proc/
cat /proc/meminfo
cat /proc/cpuinfo
cat /proc/mounts

# Explore /sys
ls /sys/block/
ls /sys/class/net/
cat /sys/block/sda/size
```

---

## Installation Commands

```bash
# Install to /usr/local/bin (recommended)
sudo install -m 755 script.sh /usr/local/bin/scriptname

# Install to ~/bin
cp script.sh ~/bin/scriptname
chmod 755 ~/bin/scriptname

# Create directory
mkdir -p ~/bin

# Add to PATH
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# System-wide PATH
echo 'export PATH="/opt/bin:$PATH"' | sudo tee /etc/profile.d/custom.sh
```

---

# Summary

## Key Takeaways

```
1. Commands are programs
   └─ Binary executables in /usr/bin, /usr/local/bin, etc.

2. They read from special files
   ├─ /proc/* for process/kernel information
   ├─ /sys/* for device/hardware information
   └─ Or make system calls to kernel

3. Virtual filesystems
   ├─ /proc and /sys exist only in RAM
   ├─ Generated by kernel on-the-fly
   └─ Always up-to-date, no disk storage

4. Modern /bin = /usr/bin
   ├─ Merged for simplicity
   ├─ /bin is symlink to /usr/bin
   └─ Backward compatibility maintained

5. PATH determines command search order
   ├─ Colon-separated directory list
   ├─ First match wins
   └─ /usr/local/bin has highest priority

6. Install custom scripts in:
   ├─ /usr/local/bin (system-wide)
   ├─ ~/bin (personal)
   └─ NOT /usr/bin (package manager)

7. Everything is a file
   ├─ Uniform interface
   ├─ Standard tools work
   └─ Easy to script and debug
```

---

## The Complete Flow

```
User types: lsblk
    ↓
Shell searches PATH:
    ├─ /usr/local/bin/lsblk → Not found
    ├─ /usr/bin/lsblk       → Found! ✓
    └─ Execute this
    ↓
lsblk program runs:
    ├─ Opens /sys/block/sda/
    ├─ Reads size, partitions
    ├─ Opens /proc/swaps
    ├─ Reads swap info
    ├─ Parses and formats
    └─ Writes to stdout
    ↓
Output displayed on terminal
    ↓
lsblk exits, returns to shell
```

---

## Quick Reference

```bash
# View PATH
echo $PATH | tr ':' '\n'

# Find command
which lsblk
type lsblk

# Install script (system-wide)
sudo install -m 755 script.sh /usr/local/bin/scriptname

# Install script (personal)
mkdir -p ~/bin
cp script.sh ~/bin/scriptname
chmod 755 ~/bin/scriptname

# Trace what command does
strace -e open,read command_name 2>&1 | less

# Explore /proc
cat /proc/meminfo
cat /proc/cpuinfo
ls /proc/$$

# Explore /sys
ls /sys/block/
cat /sys/block/sda/size

# Check if file is managed by package
rpm -qf /usr/bin/command
```

---

## Resources

```
Man Pages:
├─ man proc       # /proc filesystem
├─ man sysfs      # /sys filesystem
├─ man hier       # Filesystem hierarchy
└─ man bash       # Bash shell

Commands to Explore:
├─ strace         # System call tracer
├─ ltrace         # Library call tracer
├─ which          # Find command location
├─ type           # Command information
└─ ldd            # Library dependencies

Filesystems:
├─ /proc          # Process and kernel info
├─ /sys           # Device and hardware info
├─ /dev           # Device files
└─ /run           # Runtime data
```

---

**Now you understand how Linux commands really work!** 🚀

You can:
- ✅ Understand where commands get their information
- ✅ Read /proc and /sys yourself
- ✅ Create your own commands
- ✅ Install scripts properly
- ✅ Understand the PATH and directory structure
- ✅ Trace what any command does

---

*Document Version: 1.0*  
*Last Updated: March 2026*  
*System: RHEL 9 (rhel9exam01)*  
*Based on hands-on Linux learning and exploration*
