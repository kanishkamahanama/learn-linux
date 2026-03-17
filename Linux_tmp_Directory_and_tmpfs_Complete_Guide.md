# Linux /tmp Directory and tmpfs - Complete Guide

**Understanding, Testing, and Configuring Temporary File Storage in Linux**

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding /tmp](#understanding-tmp)
3. [tmpfs Explained](#tmpfs-explained)
4. [Disk-based /tmp vs tmpfs Comparison](#disk-based-tmp-vs-tmpfs-comparison)
5. [Why RHEL 9 Uses Disk-based /tmp](#why-rhel-9-uses-disk-based-tmp)
6. [Where tmpfs IS Used on Your System](#where-tmpfs-is-used-on-your-system)
7. [Checking Your Current Configuration](#checking-your-current-configuration)
8. [Safe Testing Guide](#safe-testing-guide)
9. [Making tmpfs Permanent](#making-tmpfs-permanent)
10. [Sizing Guidelines](#sizing-guidelines)
11. [Monitoring and Maintenance](#monitoring-and-maintenance)
12. [Troubleshooting](#troubleshooting)
13. [Best Practices](#best-practices)
14. [Command Reference](#command-reference)

---

# Introduction

The `/tmp` directory in Linux is used for temporary file storage. It can be configured in two ways:

1. **Disk-based** (traditional) - A regular directory on the root filesystem
2. **tmpfs** (modern alternative) - A RAM-based temporary filesystem

This guide covers both approaches, their differences, and how to safely test and implement tmpfs.

---

# Understanding /tmp

## What is /tmp?

```
/tmp = Temporary Directory

Purpose:
├─ Store temporary files created by applications
├─ Inter-process communication
├─ Session data
├─ Build artifacts
└─ Scratch space for scripts

Characteristics:
├─ World-writable (chmod 1777)
├─ Sticky bit set (only owner can delete their files)
├─ Should be regularly cleaned
└─ Not for long-term storage
```

## /tmp vs /var/tmp

```
┌─────────────────────────────────────────────────────┐
│ /tmp - Truly Temporary                              │
├─────────────────────────────────────────────────────┤
│ Purpose: Short-term scratch space                   │
│ Cleanup: Aggressive (often on every reboot)         │
│ Duration: Hours to days                             │
│ Examples: Build files, session data, IPC            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ /var/tmp - Temporary but Persistent                 │
├─────────────────────────────────────────────────────┤
│ Purpose: Files that survive reboots                 │
│ Cleanup: Less aggressive (30+ days)                 │
│ Duration: Days to weeks                             │
│ Examples: Package manager cache, large temp files   │
│ Location: Always on disk (never tmpfs)              │
└─────────────────────────────────────────────────────┘
```

---

# tmpfs Explained

## What is tmpfs?

```
tmpfs = Temporary File System (in RAM)

Type: Virtual filesystem
Storage: RAM (with swap overflow if needed)
Speed: Very fast (RAM speed, not disk speed)
Persistence: Lost on reboot (volatile)
Size: Configurable, limited by available RAM

How it works:
┌─────────────────────────────────────────────────────┐
│ Physical RAM                                        │
│ ████████████████░░░░░░░░░░░░░░░░░░░░ 60% used      │
│                                                     │
│ tmpfs allocation (e.g., 2GB)                        │
│ ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 20% used        │
│                                                     │
│ Files stored directly in RAM                        │
│ - Very fast read/write                              │
│ - Cleared on reboot                                 │
└─────────────────────────────────────────────────────┘
```

## tmpfs Characteristics

```
Performance:
✅ Extremely fast (RAM speed: ~5-10 GB/s)
✅ No disk I/O overhead
✅ Low latency

Resource Usage:
⚠️ Uses RAM (less available for applications)
✅ Dynamically sized (only uses what's needed)
⚠️ Can use swap if RAM is full

Persistence:
❌ Cleared on every reboot (by design)
✅ Automatic cleanup (no manual cleaning needed)

Security:
✅ Sensitive data doesn't persist
✅ Fresh start on every boot
✅ Reduced disk forensics
```

---

# Disk-based /tmp vs tmpfs Comparison

## Side-by-Side Comparison

| Feature | Disk-based /tmp | tmpfs /tmp |
|---------|-----------------|------------|
| **Storage** | Hard disk/SSD | RAM |
| **Speed** | Disk speed (~500 MB/s SSD) | RAM speed (~5000 MB/s) |
| **Persistence** | Survives reboot | Lost on reboot |
| **Size limit** | Disk space available | Configured limit (based on RAM) |
| **Resource** | Uses disk space | Uses RAM |
| **Cleanup** | Manual or scheduled | Automatic (on reboot) |
| **Performance impact** | Disk I/O | RAM consumption |
| **Large files** | ✅ Supported | ⚠️ Limited by RAM |
| **Default on** | RHEL 9, CentOS 9 | Arch Linux, Fedora |
| **Security** | Files persist | Auto-wiped on boot |
| **Reliability** | Survives crashes | Lost on crashes |

---

## Detailed Comparison

### Disk-based /tmp (RHEL 9 Default)

**Advantages:**
```
✅ Persistent across reboots
   - Applications can rely on files surviving restarts
   - Useful for multi-stage processes

✅ Large file support
   - Can use all available disk space
   - No RAM limitation

✅ RAM conservation
   - Doesn't consume RAM
   - More memory available for applications

✅ Predictable behavior
   - Standard across most enterprise systems
   - No surprises with size limits

✅ Compatibility
   - Works with all applications
   - No special considerations needed
```

**Disadvantages:**
```
❌ Slower performance
   - Limited by disk I/O speed
   - Disk seek times

❌ Manual cleanup required
   - Files accumulate over time
   - Needs periodic cleaning

❌ Security concerns
   - Sensitive data may persist
   - Forensic recovery possible

❌ Disk wear
   - Constant writes to disk (SSD wear)

❌ Can fill root filesystem
   - May impact system if not monitored
```

---

### tmpfs /tmp

**Advantages:**
```
✅ Extreme performance
   - RAM speed (10-20x faster than SSD)
   - Ideal for build systems, compile jobs

✅ Automatic cleanup
   - Cleared on every reboot
   - No manual maintenance needed

✅ Enhanced security
   - Sensitive temp data doesn't survive reboot
   - Reduced forensic footprint

✅ Reduced disk wear
   - No disk writes for temporary files
   - Extends SSD lifespan

✅ Isolated from root filesystem
   - Cannot fill up / partition
   - Better stability
```

**Disadvantages:**
```
❌ RAM consumption
   - Reduces available RAM
   - May impact application performance

❌ Size limited
   - Cannot exceed allocated size
   - Large files may fail

❌ Lost on reboot
   - Applications expecting persistence will fail
   - Data loss if system crashes

❌ Swap usage possible
   - If RAM full, tmpfs can use swap
   - Defeats purpose (slow)

❌ Incompatibility risks
   - Some applications assume /tmp persists
   - May break certain workflows
```

---

## Performance Comparison

### Speed Test Example

```bash
# Disk-based /tmp (XFS on SSD)
$ time dd if=/dev/zero of=/tmp/test bs=1M count=1000
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB) copied, 2.1s, 500 MB/s
                                            ↑
                                    SSD speed (~500 MB/s)

# tmpfs /tmp (RAM)
$ time dd if=/dev/zero of=/tmp/test bs=1M count=1000
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB) copied, 0.15s, 7.0 GB/s
                                             ↑
                                    RAM speed (~7 GB/s)

Result: tmpfs is ~14x faster!
```

---

# Why RHEL 9 Uses Disk-based /tmp

## Red Hat's Design Decisions

### Historical Context

```
Enterprise Linux Philosophy:
├─ Stability over innovation
├─ Predictability over performance
├─ Compatibility over convenience
└─ Safety over speed

RHEL 9 keeps /tmp on disk because:
1. Backward compatibility
2. Enterprise application expectations
3. Large file support requirements
4. RAM conservation for applications
5. Predictable behavior across installations
```

### Enterprise Considerations

```
Why enterprises prefer disk-based /tmp:
✅ Applications assume /tmp persists across reboots
✅ Build processes may span multiple restarts
✅ Large compilation jobs need space > RAM
✅ Critical services expect /tmp availability
✅ Easier capacity planning (disk vs RAM)
```

### Distribution Comparison

```
Distribution          Default /tmp
==========================================
RHEL 9                Disk (XFS)
CentOS Stream 9       Disk (XFS)
Ubuntu 20.04/22.04    Disk (ext4)
Debian 11/12          Disk (ext4)
Fedora 36+            tmpfs
Arch Linux            tmpfs
openSUSE              Disk (btrfs/ext4)

Trend: Servers → Disk, Desktops → tmpfs
```

---

# Where tmpfs IS Used on Your System

Even though `/tmp` is on disk, RHEL 9 uses tmpfs extensively elsewhere!

## /dev/shm - Shared Memory

```bash
$ df -hT /dev/shm
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  3.8G  168K  3.8G   1% /dev/shm

Purpose:
├─ Shared memory segments
├─ POSIX shared memory
├─ Inter-process communication (IPC)
└─ Fast temporary storage for applications

Size: 50% of RAM by default

Example usage:
├─ Database shared buffers
├─ Multiprocess applications
├─ High-performance computing
└─ Real-time data exchange
```

---

## /run - Runtime Data

```bash
$ df -hT /run
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  1.6G  9.8M  1.6G   1% /run

Purpose:
├─ Runtime system information
├─ PID files (process IDs)
├─ Socket files
├─ Lock files
└─ Early boot data

Contents examples:
/run/
├─ systemd/          # systemd runtime state
├─ udev/             # Device manager state
├─ lock/             # Lock files
├─ user/             # User runtime directories
└─ *.pid             # Process ID files

Size: ~20% of RAM (configurable)
```

---

## /run/user/[UID] - User Runtime Directory

```bash
$ df -hT /run/user/1000
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  768M  120K  768M   1% /run/user/1000

Purpose:
├─ User-specific runtime files
├─ User session data
├─ XDG_RUNTIME_DIR location
└─ Per-user temporary files

Environment variable:
$ echo $XDG_RUNTIME_DIR
/run/user/1000

Size: 10% of RAM per user (default)

Used by:
├─ Desktop environments (GNOME, KDE)
├─ Wayland compositors
├─ PulseAudio/PipeWire
└─ User services (systemd --user)
```

---

## tmpfs Summary on RHEL 9

```
Location          Size        Purpose
================================================
/dev/shm          50% RAM     Shared memory IPC
/run              20% RAM     System runtime data
/run/user/UID     10% RAM     User runtime data
/tmp              N/A         NOT tmpfs (on disk!)

Total tmpfs allocation: ~80% of RAM (max)
But only uses space as needed (dynamic)
```

---

# Checking Your Current Configuration

## Verify /tmp Location

### Method 1: Check Filesystem Type

```bash
# Check /tmp filesystem type
df -hT /tmp

# Disk-based /tmp (RHEL 9 default):
Filesystem                        Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel_rhel9exam01-root xfs    37G  9.1G   28G  25% /
#                                  ↑                              ↑
#                             XFS type                      Mounted at /

# tmpfs /tmp (if configured):
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  2.0G  4.0K  2.0G   1% /tmp
#               ↑
#          tmpfs type
```

---

### Method 2: Check Mount Points

```bash
# List all mounts
mount | grep /tmp

# No output = /tmp is not separately mounted (on root filesystem)

# If tmpfs:
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime,size=2097152k)

# Alternative: Use findmnt
findmnt /tmp

# Disk-based:
TARGET SOURCE                            FSTYPE OPTIONS
/      /dev/mapper/rhel_rhel9exam01-root xfs    rw,relatime...

# tmpfs:
TARGET SOURCE FSTYPE OPTIONS
/tmp   tmpfs  tmpfs  rw,nosuid,nodev,relatime,size=2097152k
```

---

### Method 3: Check /etc/fstab

```bash
# Check if /tmp is in fstab
grep /tmp /etc/fstab

# No output = /tmp not configured to mount separately

# If configured:
# tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0
```

---

### Method 4: Check systemd tmp.mount

```bash
# Check if tmp.mount service exists
systemctl status tmp.mount

# If disabled/inactive (default RHEL 9):
● tmp.mount - Temporary Directory (/tmp)
   Loaded: loaded (/usr/lib/systemd/system/tmp.mount; disabled; ...)
   Active: inactive (dead)
#           ↑
#    Not using tmpfs

# If enabled:
● tmp.mount - Temporary Directory (/tmp)
   Loaded: loaded (/usr/lib/systemd/system/tmp.mount; enabled; ...)
   Active: active (mounted) since ...
#           ↑
#    Using tmpfs
```

---

### Method 5: Compare Devices

```bash
# Check if /tmp is on same device as /
df / /tmp

# Same device = /tmp is on root filesystem
Filesystem                        1K-blocks    Used Available Use% Mounted on
/dev/mapper/rhel_rhel9exam01-root  38352592 9458964  28893628  25% /
/dev/mapper/rhel_rhel9exam01-root  38352592 9458964  28893628  25% /tmp
#                                    ↑
#                          Same device, same filesystem

# Different device = /tmp is separately mounted
Filesystem                        1K-blocks    Used Available Use% Mounted on
/dev/mapper/rhel_rhel9exam01-root  38352592 9458964  28893628  25% /
tmpfs                               2097152       4   2097148   1% /tmp
#                                    ↑
#                          Different (tmpfs)
```

---

## Check Current /tmp Usage

```bash
# Size of /tmp
du -sh /tmp/
# 120K    /tmp/

# Detailed breakdown
du -sh /tmp/* 2>/dev/null | sort -rh

# List files
ls -la /tmp/

# Count files
find /tmp -type f | wc -l

# Find processes using /tmp
lsof +D /tmp/

# Largest files in /tmp
find /tmp -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
```

---

# Safe Testing Guide

## Pre-Testing Checklist

```
Before testing tmpfs:

☑ Document current configuration
☑ Check available RAM (need at least 2GB free)
☑ Backup important data (shouldn't be in /tmp anyway!)
☑ Understand reboot reverts changes
☑ Have console access (in case of issues)
☑ Plan for 24-48 hour testing period
```

---

## Stage 1: Document Current State

### Create Baseline Documentation

```bash
# Create documentation file
echo "=== /tmp Migration Documentation ===" > ~/tmp_migration_log.txt
echo "Date: $(date)" >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Current system info
echo "=== System Information ===" >> ~/tmp_migration_log.txt
uname -a >> ~/tmp_migration_log.txt
cat /etc/os-release | grep PRETTY_NAME >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Current RAM
echo "=== Memory Status ===" >> ~/tmp_migration_log.txt
free -h >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Current /tmp status
echo "=== Current /tmp Configuration ===" >> ~/tmp_migration_log.txt
df -hT /tmp >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt
mount | grep /tmp >> ~/tmp_migration_log.txt || echo "No separate mount" >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# /tmp contents
echo "=== /tmp Contents ===" >> ~/tmp_migration_log.txt
ls -la /tmp/ >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt
du -sh /tmp/ >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Processes using /tmp
echo "=== Processes Using /tmp ===" >> ~/tmp_migration_log.txt
lsof +D /tmp/ 2>/dev/null >> ~/tmp_migration_log.txt || echo "None found" >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# View the log
cat ~/tmp_migration_log.txt
```

---

## Stage 2: Temporary tmpfs Mount (Reversible)

### Mount tmpfs Over /tmp

```bash
# This is SAFE - completely reversible!
# Changes revert on reboot

# 1. Check available RAM
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.5Gi       4.7Gi       721Mi        21Mi       2.5Gi       2.8Gi
#                                                                            ↑
#                                                               Need 2GB+ available

# 2. Mount tmpfs on /tmp (temporary)
sudo mount -t tmpfs -o size=2G,mode=1777,nosuid,nodev tmpfs /tmp

# Options explained:
# -t tmpfs           : Filesystem type
# -o                 : Options
#    size=2G         : Maximum size 2GB
#    mode=1777       : Permissions (sticky bit: rwxrwxrwt)
#    nosuid          : Ignore set-user-ID bits (security)
#    nodev           : No device files (security)
# tmpfs              : Device name (can be anything, convention is "tmpfs")
# /tmp               : Mount point

# 3. Verify the mount
df -hT /tmp

# Expected output:
# Filesystem     Type   Size  Used Avail Use% Mounted on
# tmpfs          tmpfs  2.0G  4.0K  2.0G   1% /tmp
#                 ↑      ↑
#            tmpfs!   2GB max

# 4. Check mount details
mount | grep /tmp
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime,size=2097152k,mode=1777)

# 5. Verify RAM usage increased
free -h
# Mem "used" should increase slightly (only what's actually used in /tmp)
```

---

### What Just Happened?

```
Before mount:
/tmp was a directory on / (root filesystem)
Files: Stored on disk (/dev/mapper/rhel_rhel9exam01-root)

After mount:
/tmp is now tmpfs (separate filesystem in RAM)
Files: Stored in RAM
Old /tmp directory: Hidden (not deleted!)

Reverting:
Option 1: Reboot (automatic)
Option 2: umount /tmp (manual)

Note: Files created on tmpfs will be LOST when unmounting!
```

---

## Stage 3: Test the tmpfs /tmp

### Basic Functionality Test

```bash
# 1. Create test file
echo "Testing tmpfs /tmp" > /tmp/test.txt
cat /tmp/test.txt
# Testing tmpfs /tmp

# 2. Check permissions
ls -ld /tmp
# drwxrwxrwt. ... /tmp
#  ↑
# Correct (sticky bit set)

# 3. Test from different user (if available)
sudo -u nobody touch /tmp/nobody_test.txt
ls -l /tmp/nobody_test.txt
# Verify it works

# 4. Clean up
rm /tmp/test.txt /tmp/nobody_test.txt
```

---

### Performance Test

```bash
# Write speed test
echo "=== tmpfs Write Speed Test ===" | tee -a ~/tmp_migration_log.txt
time sh -c "dd if=/dev/zero of=/tmp/write_test bs=1M count=200 && sync" 2>&1 | tee -a ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Read speed test
echo "=== tmpfs Read Speed Test ===" | tee -a ~/tmp_migration_log.txt
time dd if=/tmp/write_test of=/dev/null bs=1M 2>&1 | tee -a ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Compare with disk (for reference)
echo "=== Disk Write Speed Test ===" | tee -a ~/tmp_migration_log.txt
time sh -c "dd if=/dev/zero of=/home/disk_test bs=1M count=200 && sync" 2>&1 | tee -a ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

echo "=== Disk Read Speed Test ===" | tee -a ~/tmp_migration_log.txt
time dd if=/home/disk_test of=/dev/null bs=1M 2>&1 | tee -a ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Clean up
rm /tmp/write_test /home/disk_test

# View results
tail -40 ~/tmp_migration_log.txt
```

---

### RAM Impact Test

```bash
# Before creating files
echo "=== RAM Before ===" >> ~/tmp_migration_log.txt
free -h >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Create several files
for i in {1..10}; do
    dd if=/dev/zero of=/tmp/file$i bs=1M count=50 2>/dev/null
done

# After creating files (500MB total)
echo "=== RAM After (500MB in /tmp) ===" >> ~/tmp_migration_log.txt
free -h >> ~/tmp_migration_log.txt
df -h /tmp >> ~/tmp_migration_log.txt
echo "" >> ~/tmp_migration_log.txt

# Clean up
rm /tmp/file*

# View impact
tail -20 ~/tmp_migration_log.txt
```

---

### Application Compatibility Test

```bash
# Test common operations that use /tmp

# 1. Compile test (if gcc installed)
echo 'int main() { return 0; }' > /tmp/test.c
gcc -o /tmp/test /tmp/test.c 2>&1
if [ $? -eq 0 ]; then
    echo "Compilation: OK"
else
    echo "Compilation: FAILED"
fi
rm /tmp/test.c /tmp/test 2>/dev/null

# 2. Archive operations
tar czf /tmp/test.tar.gz /etc/hostname
if [ $? -eq 0 ]; then
    echo "Archive creation: OK"
else
    echo "Archive creation: FAILED"
fi
rm /tmp/test.tar.gz

# 3. Script execution
echo '#!/bin/bash' > /tmp/test.sh
echo 'echo "Script test"' >> /tmp/test.sh
chmod +x /tmp/test.sh
/tmp/test.sh
rm /tmp/test.sh

# 4. mktemp test (common tool)
TMPFILE=$(mktemp)
if [ -f "$TMPFILE" ]; then
    echo "mktemp: OK ($TMPFILE)"
    rm "$TMPFILE"
else
    echo "mktemp: FAILED"
fi
```

---

## Stage 4: Normal Usage Test (24-48 Hours)

### Monitoring During Test Period

```bash
# Create monitoring script
cat > ~/monitor_tmp_test.sh << 'EOF'
#!/bin/bash
echo "=== /tmp Monitor: $(date) ==="
echo ""
echo "Mount status:"
df -hT /tmp
echo ""
echo "Usage:"
du -sh /tmp
echo ""
echo "RAM impact:"
free -h | grep Mem
echo ""
echo "Largest files:"
find /tmp -type f -exec du -h {} + 2>/dev/null | sort -rh | head -5
echo ""
echo "================================"
echo ""
EOF

chmod +x ~/monitor_tmp_test.sh

# Run manually to check
~/monitor_tmp_test.sh

# Or schedule to run every hour during test
# (Optional - adds to cron)
(crontab -l 2>/dev/null; echo "0 * * * * ~/monitor_tmp_test.sh >> ~/tmp_monitor.log") | crontab -

# View monitoring log
tail -30 ~/tmp_monitor.log
```

---

### What to Watch For

```
During 24-48 hour test period:

✓ Check regularly:
  - Is /tmp filling up? (df -h /tmp)
  - Is RAM being consumed? (free -h)
  - Any application errors? (journalctl -p err)
  - System performance OK?

✓ Test your workflows:
  - Development/compilation
  - Package installations
  - System updates
  - Regular tasks

✗ Problems to watch for:
  - "No space left" errors
  - Application failures
  - System slowdown
  - High swap usage
```

---

## Stage 5: Revert to Original (If Needed)

### Option 1: Reboot (Automatic Revert)

```bash
# Simplest method - just reboot
sudo reboot

# After reboot:
df -hT /tmp
# Back to original:
# /dev/mapper/rhel_rhel9exam01-root xfs    37G  9.1G   28G  25% /

# All tmpfs changes are gone!
# System is exactly as before
```

---

### Option 2: Manual Unmount

```bash
# Unmount tmpfs immediately (without reboot)

# 1. Check for processes using /tmp
lsof +D /tmp/

# 2. If any processes found, stop them or wait

# 3. Unmount
sudo umount /tmp

# 4. Verify
df -hT /tmp
# Should show root filesystem again

# 5. Check original /tmp directory
ls -la /tmp/
# Original contents (if any) should be visible again

# Note: Files created on tmpfs are LOST!
```

---

### Verify Revert Was Successful

```bash
# Check filesystem type
df -hT /tmp | grep xfs
# Should show xfs (or your root filesystem type)

# Check it's on root device
df / /tmp | grep -c "rhel_rhel9exam01-root"
# Should output: 2 (both on same device)

# Check mount
mount | grep /tmp
# Should have no output (not separately mounted)

# Document revert
echo "=== Reverted to Original $(date) ===" >> ~/tmp_migration_log.txt
df -hT /tmp >> ~/tmp_migration_log.txt
```

---

# Making tmpfs Permanent

**Only proceed if testing was successful and you want tmpfs permanently!**

---

## Method 1: Using systemd (Recommended)

### Enable tmp.mount Unit

```bash
# 1. Check if tmp.mount exists
systemctl cat tmp.mount

# If exists, you'll see:
# [Unit]
# Description=Temporary Directory (/tmp)
# Documentation=man:hier(7)
# ...
# [Mount]
# What=tmpfs
# Where=/tmp
# Type=tmpfs
# Options=mode=1777,strictatime,nosuid,nodev

# 2. Check current status
systemctl status tmp.mount
# ● tmp.mount - Temporary Directory (/tmp)
#    Loaded: loaded (/usr/lib/systemd/system/tmp.mount; disabled; ...)
#    Active: inactive (dead)

# 3. Enable for next boot
sudo systemctl enable tmp.mount
# Created symlink /etc/systemd/system/local-fs.target.wants/tmp.mount → /usr/lib/systemd/system/tmp.mount

# 4. Start now (without reboot)
sudo systemctl start tmp.mount

# 5. Verify
systemctl status tmp.mount
# ● tmp.mount - Temporary Directory (/tmp)
#    Loaded: loaded (/usr/lib/systemd/system/tmp.mount; enabled; ...)
#    Active: active (mounted) since ...

df -hT /tmp
# Filesystem     Type   Size  Used Avail Use% Mounted on
# tmpfs          tmpfs  3.8G  4.0K  3.8G   1% /tmp

# 6. Test persistence by rebooting
sudo reboot

# After reboot:
df -hT /tmp
# Should still show tmpfs
```

---

### Customize tmp.mount Size (Optional)

```bash
# 1. Create override directory
sudo mkdir -p /etc/systemd/system/tmp.mount.d

# 2. Create custom configuration
sudo tee /etc/systemd/system/tmp.mount.d/override.conf << 'EOF'
[Mount]
Options=mode=1777,strictatime,nosuid,nodev,size=2G
EOF

# Explanation:
# size=2G  : Limit tmpfs to 2GB (adjust as needed)
# Default is 50% of RAM if not specified

# 3. Reload systemd
sudo systemctl daemon-reload

# 4. Restart tmp.mount
sudo systemctl restart tmp.mount

# 5. Verify new size
df -h /tmp
# Filesystem     Type   Size  Used Avail Use% Mounted on
# tmpfs          tmpfs  2.0G  4.0K  2.0G   1% /tmp
#                        ↑
#                    Custom 2GB size
```

---

## Method 2: Using /etc/fstab

### Add tmpfs Entry to fstab

```bash
# 1. BACKUP fstab first! (CRITICAL!)
sudo cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)

# 2. Verify backup
ls -l /etc/fstab*
# -rw-r--r--. 1 root root ... /etc/fstab
# -rw-r--r--. 1 root root ... /etc/fstab.backup.20260317

# 3. Edit fstab
sudo vim /etc/fstab

# 4. Add this line at the end:
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev   0   0

# Line breakdown:
# tmpfs              : Device (virtual)
# /tmp               : Mount point
# tmpfs              : Filesystem type
# defaults           : Default mount options
# size=2G            : Maximum size (REQUIRED - prevents using all RAM!)
# mode=1777          : Permissions with sticky bit
# nosuid             : Ignore SUID bits (security)
# nodev              : Don't interpret device files (security)
# 0                  : Don't dump (backup)
# 0                  : Don't fsck (filesystem check)

# Optional: Add noexec for additional security
# tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev,noexec   0   0

# 5. Verify syntax (CRITICAL before rebooting!)
sudo findmnt --verify

# Expected output:
# Success, no errors found
# OR
# Lists any syntax errors (FIX BEFORE CONTINUING!)

# 6. Test mount WITHOUT rebooting
sudo mount -a

# If errors:
# - Check /etc/fstab syntax
# - Fix errors
# - Try mount -a again
# - DO NOT REBOOT until mount -a succeeds!

# 7. Verify mount
df -hT /tmp
# Filesystem     Type   Size  Used Avail Use% Mounted on
# tmpfs          tmpfs  2.0G  4.0K  2.0G   1% /tmp

# 8. If successful, reboot to test persistence
sudo reboot

# 9. After reboot, verify
df -hT /tmp
mount | grep /tmp

# 10. Document success
echo "=== Made Permanent via fstab $(date) ===" >> ~/tmp_migration_log.txt
cat /etc/fstab | grep tmpfs >> ~/tmp_migration_log.txt
df -hT /tmp >> ~/tmp_migration_log.txt
```

---

### fstab Options Reference

```bash
# Basic configuration (recommended)
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0

# Security-focused configuration
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev,noexec   0   0

# Performance-focused configuration
tmpfs   /tmp   tmpfs   defaults,size=4G,mode=1777,noatime   0   0

# Size options:
# size=2G        : Fixed 2GB limit
# size=50%       : 50% of RAM
# size=1048576   : Size in KB (1GB)
# (no size)      : Default 50% of RAM (NOT RECOMMENDED!)

# Security options:
# nosuid         : Ignore set-UID/set-GID bits
# nodev          : Don't allow device files
# noexec         : Don't allow executing binaries (may break some apps!)

# Performance options:
# noatime        : Don't update access times (faster)
# nodiratime     : Don't update directory access times
```

---

## Verification After Making Permanent

```bash
# Complete verification checklist

# 1. Check mount
mount | grep /tmp
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime,size=2097152k,mode=1777)

# 2. Check fstab
grep /tmp /etc/fstab
# tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev   0   0

# OR check systemd
systemctl status tmp.mount
# Active: active (mounted)

# 3. Verify size limit
df -h /tmp
# Size column should match your configuration

# 4. Test persistence
sudo reboot
# After reboot, check again

# 5. Document final configuration
cat > ~/tmp_final_config.txt << EOF
Final tmpfs Configuration
========================
Date: $(date)
Method: $([ -f /etc/fstab ] && grep -q tmpfs /etc/fstab && echo "fstab" || echo "systemd")
Size: $(df -h /tmp | tail -1 | awk '{print $2}')
Mount: $(mount | grep /tmp)
Status: Permanent (survives reboot)
EOF

cat ~/tmp_final_config.txt
```

---

# Sizing Guidelines

## Calculating tmpfs Size

### RAM-Based Recommendations

```
System RAM    Recommended tmpfs Size    Reasoning
============  ========================   =================
< 4GB         1GB or 25% RAM            Conservative
4-8GB         2GB or 25-30% RAM         Balanced
8-16GB        2-4GB or 25-30% RAM       Moderate
16-32GB       4-8GB or 25-30% RAM       Generous
> 32GB        8GB+ or 20-25% RAM        Server workload

General Formula:
tmpfs_size = RAM × 0.25 (25% of RAM)

For your system (7.5GB RAM):
Recommended: 2GB (conservative)
Maximum safe: 3-4GB (moderate)
```

---

### Usage-Based Sizing

```
Check your current /tmp usage:
$ du -sh /tmp/
120K    /tmp/     ← Very small!

Recommendation:
├─ Current usage: 120KB
├─ Buffer (100x): 12MB
├─ Headroom (10x): 120MB
└─ Recommended: 1-2GB (plenty of space!)

Your /tmp is barely used!
2GB is more than enough.
```

---

### Application-Specific Considerations

```
Workload Type              Recommended Size
============================================
Desktop/laptop             1-2GB
General server             2-4GB
Development machine        3-4GB
Build server              4-8GB
Database server           1-2GB (minimal)
Web server                2-4GB
Container host            4-8GB

Special cases:
├─ Large compilations: 4GB+
├─ Video editing: 8GB+
├─ Virtual machines: 2-4GB
└─ Minimal system: 512MB-1GB
```

---

### Dynamic vs Fixed Sizing

```
Dynamic (no size= option):
├─ Default: 50% of RAM
├─ Grows as needed
├─ Can use all RAM (dangerous!)
└─ NOT RECOMMENDED!

Fixed (size=2G):
├─ Hard limit: 2GB
├─ Safe (can't exhaust RAM)
├─ Predictable
└─ RECOMMENDED ✓

Example:
# WRONG (can use all RAM):
tmpfs   /tmp   tmpfs   defaults,mode=1777   0   0

# CORRECT (limited to 2GB):
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0
```

---

## Monitoring Size Usage

```bash
# Current usage
df -h /tmp
# Filesystem     Type   Size  Used Avail Use% Mounted on
# tmpfs          tmpfs  2.0G   45M  2.0G   3% /tmp
#                        ↑      ↑     ↑    ↑
#                     Total  Used  Avail  Percentage

# Watch in real-time
watch -n 2 'df -h /tmp'

# Alert if > 80% full
df -h /tmp | awk 'NR==2 {if (int($5) > 80) print "WARNING: /tmp is " $5 " full"}'

# Detailed usage
du -sh /tmp/*

# Largest items
du -sh /tmp/* 2>/dev/null | sort -rh | head -10
```

---

# Monitoring and Maintenance

## Daily Monitoring

### Quick Health Check

```bash
# Create daily check script
cat > ~/check_tmp_health.sh << 'EOF'
#!/bin/bash

echo "=== /tmp Health Check: $(date) ==="
echo ""

# 1. Mount status
echo "Mount Status:"
df -hT /tmp
echo ""

# 2. Usage percentage
USAGE=$(df /tmp | tail -1 | awk '{print int($5)}')
echo "Usage: ${USAGE}%"
if [ $USAGE -gt 80 ]; then
    echo "⚠️  WARNING: /tmp is ${USAGE}% full!"
elif [ $USAGE -gt 90 ]; then
    echo "❌ CRITICAL: /tmp is ${USAGE}% full!"
else
    echo "✓ Usage is normal"
fi
echo ""

# 3. RAM impact
echo "RAM Status:"
free -h | grep Mem
echo ""

# 4. Top 5 largest items
echo "Largest items in /tmp:"
du -sh /tmp/* 2>/dev/null | sort -rh | head -5
echo ""

# 5. File count
FILE_COUNT=$(find /tmp -type f 2>/dev/null | wc -l)
echo "Total files: $FILE_COUNT"
echo ""

# 6. Oldest files
echo "Oldest files:"
find /tmp -type f -printf '%T+ %p\n' 2>/dev/null | sort | head -5
echo ""

echo "================================"
EOF

chmod +x ~/check_tmp_health.sh

# Run manually
~/check_tmp_health.sh

# Schedule daily (optional)
# (crontab -l 2>/dev/null; echo "0 9 * * * ~/check_tmp_health.sh >> ~/tmp_health.log") | crontab -
```

---

### Automated Monitoring

```bash
# Create monitoring service (systemd timer)

# 1. Create monitor script
sudo tee /usr/local/bin/monitor-tmp.sh << 'EOF'
#!/bin/bash
USAGE=$(df /tmp | tail -1 | awk '{print int($5)}')
if [ $USAGE -gt 90 ]; then
    logger -t tmpfs-monitor -p user.warning "/tmp is ${USAGE}% full!"
    # Optional: Send email
    # echo "/tmp is ${USAGE}% full on $(hostname)" | mail -s "tmpfs Alert" admin@example.com
fi
EOF

sudo chmod +x /usr/local/bin/monitor-tmp.sh

# 2. Create systemd service
sudo tee /etc/systemd/system/monitor-tmp.service << 'EOF'
[Unit]
Description=Monitor tmpfs usage

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-tmp.sh
EOF

# 3. Create systemd timer
sudo tee /etc/systemd/system/monitor-tmp.timer << 'EOF'
[Unit]
Description=Monitor tmpfs usage every hour

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
EOF

# 4. Enable and start timer
sudo systemctl daemon-reload
sudo systemctl enable monitor-tmp.timer
sudo systemctl start monitor-tmp.timer

# 5. Check timer status
systemctl status monitor-tmp.timer
systemctl list-timers monitor-tmp.timer
```

---

## Cleanup Procedures

### Manual Cleanup

```bash
# WARNING: Be careful when cleaning /tmp!

# 1. Check what's being deleted first
find /tmp -type f -mtime +7
# Shows files older than 7 days

# 2. Delete old files (older than 7 days)
find /tmp -type f -mtime +7 -delete

# 3. Or delete everything (DANGEROUS!)
# Only if you're sure:
sudo rm -rf /tmp/*

# 4. Recreate proper permissions
sudo chmod 1777 /tmp
```

---

### Automated Cleanup (systemd-tmpfiles)

```bash
# RHEL 9 uses systemd-tmpfiles for cleanup

# View cleanup configuration
cat /usr/lib/tmpfiles.d/tmp.conf

# Example configuration:
# q /tmp 1777 root root 10d
# Explanation:
# q     : Create if needed, set permissions
# /tmp  : Directory
# 1777  : Permissions (sticky bit)
# root  : Owner
# root  : Group
# 10d   : Clean files older than 10 days

# Manual trigger of cleanup
sudo systemd-tmpfiles --clean

# Test cleanup (dry-run)
sudo systemd-tmpfiles --clean --dry-run

# Create custom cleanup rules
sudo tee /etc/tmpfiles.d/custom-tmp.conf << 'EOF'
# Clean /tmp files older than 3 days
d /tmp 1777 root root 3d
EOF

# Apply new rules
sudo systemd-tmpfiles --create
```

---

### Emergency Cleanup (tmpfs Full)

```bash
# If /tmp is 100% full and system is having issues:

# 1. Check largest files
du -sh /tmp/* 2>/dev/null | sort -rh | head -10

# 2. Identify safe-to-delete files
# - Build artifacts (*.o, *.a)
# - Old session files
# - Temp downloads

# 3. Delete specific items
rm -rf /tmp/specific_large_item

# 4. Or nuclear option (CAUTION!)
sudo rm -rf /tmp/*
sudo chmod 1777 /tmp

# 5. Verify space freed
df -h /tmp

# 6. If tmpfs limit too small, increase it:
sudo mount -o remount,size=4G /tmp
# Then make permanent change to fstab or tmp.mount
```

---

# Troubleshooting

## Common Issues and Solutions

### Issue 1: "No space left on device" on /tmp

**Symptoms:**
```
$ cp largefile /tmp/
cp: cannot create regular file '/tmp/largefile': No space left on device

df -h /tmp
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  2.0G  2.0G     0 100% /tmp
                                      ↑
                                   Full!
```

**Solutions:**

```bash
# Option 1: Clean up /tmp
sudo rm -rf /tmp/*

# Option 2: Increase tmpfs size temporarily
sudo mount -o remount,size=4G /tmp
df -h /tmp
# Now 4GB

# Option 3: Make permanent size increase
# Edit /etc/fstab or tmp.mount override
# Change: size=2G → size=4G

# Option 4: Revert to disk-based /tmp
sudo umount /tmp
# Or disable tmp.mount
sudo systemctl disable tmp.mount
sudo systemctl stop tmp.mount
```

---

### Issue 2: High RAM Usage

**Symptoms:**
```
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.5Gi       6.8Gi       100Mi        21Mi       600Mi       400Mi
#                             ↑           ↑                                  ↑
#                       Very high    Very low                           Very low

df -h /tmp
# tmpfs          2.0G  1.8G  200M  90% /tmp
#                       ↑
#                  Using lots of RAM
```

**Solutions:**

```bash
# 1. Check what's using /tmp
du -sh /tmp/* 2>/dev/null | sort -rh | head -10

# 2. Clean up large files
rm -rf /tmp/large_files

# 3. Reduce tmpfs size
sudo mount -o remount,size=1G /tmp

# 4. Or revert to disk-based /tmp
sudo umount /tmp
```

---

### Issue 3: Application Errors After tmpfs

**Symptoms:**
```
Application crashes or errors like:
- "Permission denied on /tmp/..."
- "Cannot execute /tmp/..."
- "Operation not permitted"
```

**Possible Causes & Solutions:**

```bash
# Cause 1: noexec option
mount | grep /tmp
# tmpfs on /tmp type tmpfs (...,noexec,...)
#                                 ↑
#                          Can't execute binaries!

# Solution: Remove noexec
sudo mount -o remount,nosuid,nodev /tmp  # (removed noexec)
# Or edit /etc/fstab to remove noexec

# Cause 2: Wrong permissions
ls -ld /tmp
# drwxr-xr-x  # WRONG! Should be drwxrwxrwt

# Solution: Fix permissions
sudo chmod 1777 /tmp

# Cause 3: File too large for tmpfs
# Solution: Increase tmpfs size or use /var/tmp
```

---

### Issue 4: Lost Data After Reboot

**Symptoms:**
```
After reboot, files in /tmp are gone!
```

**This is EXPECTED behavior with tmpfs!**

**Solutions:**

```bash
# 1. Don't store important data in /tmp
# /tmp is for TEMPORARY files only

# 2. Use /var/tmp for persistent temporary storage
cp /tmp/important /var/tmp/important

# 3. Or use a different directory entirely
mkdir ~/persistent_temp
cp /tmp/important ~/persistent_temp/

# 4. If you need /tmp to persist, revert to disk-based
sudo systemctl disable tmp.mount
sudo systemctl stop tmp.mount
```

---

### Issue 5: System Won't Boot (Bad fstab)

**Symptoms:**
```
System drops to emergency shell:
"Failed to mount /tmp"
"Enter root password for maintenance"
```

**Recovery:**

```bash
# 1. Enter root password at emergency prompt

# 2. Remount / as read-write
mount -o remount,rw /

# 3. Edit /etc/fstab
vim /etc/fstab

# 4. Comment out or fix the tmpfs line
# tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0
# ↓
## tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0

# 5. Save and exit

# 6. Reboot
reboot

# System should boot normally now
# Fix the fstab entry properly, then re-enable
```

---

### Issue 6: Swap Usage Increasing

**Symptoms:**
```
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.5Gi       7.2Gi       100Mi       2Gi       200Mi       100Mi
#                                                     ↑
#                                              tmpfs using RAM!
# Swap:          3.9Gi       1.8Gi       2.1Gi
#                             ↑
#                        Swap increasing!
```

**This defeats the purpose of tmpfs (using RAM)!**

**Solutions:**

```bash
# 1. tmpfs is too large for available RAM
# Reduce tmpfs size:
sudo mount -o remount,size=1G /tmp

# 2. Or revert to disk-based /tmp
sudo umount /tmp

# 3. Or add more RAM to system
# 4. Or close other applications
```

---

### Issue 7: systemd tmp.mount Won't Start

**Symptoms:**
```
$ sudo systemctl start tmp.mount
Failed to start tmp.mount: Unit tmp.mount is masked

OR

$ sudo systemctl enable tmp.mount
Failed to enable unit: Unit file tmp.mount does not exist
```

**Solutions:**

```bash
# If masked:
sudo systemctl unmask tmp.mount
sudo systemctl enable tmp.mount
sudo systemctl start tmp.mount

# If doesn't exist:
# Use /etc/fstab method instead
echo "tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777   0   0" | sudo tee -a /etc/fstab
sudo mount -a
```

---

## Diagnostic Commands

```bash
# Complete diagnostic script
cat > ~/diagnose_tmp.sh << 'EOF'
#!/bin/bash
echo "=== /tmp Diagnostic Report ==="
echo "Date: $(date)"
echo ""

echo "1. Mount Status:"
mount | grep /tmp || echo "  /tmp not separately mounted"
echo ""

echo "2. Filesystem Type:"
df -hT /tmp
echo ""

echo "3. fstab Entry:"
grep /tmp /etc/fstab || echo "  No /tmp entry in fstab"
echo ""

echo "4. systemd tmp.mount:"
systemctl status tmp.mount 2>&1 | head -5
echo ""

echo "5. Usage:"
du -sh /tmp
echo ""

echo "6. Permissions:"
ls -ld /tmp
echo ""

echo "7. Processes Using /tmp:"
lsof +D /tmp 2>/dev/null | wc -l
echo "  Number of processes: $(lsof +D /tmp 2>/dev/null | wc -l)"
echo ""

echo "8. RAM Status:"
free -h
echo ""

echo "9. Recent Errors:"
journalctl -p err --since "1 hour ago" 2>&1 | grep -i tmp || echo "  No recent errors"
echo ""

echo "==================================="
EOF

chmod +x ~/diagnose_tmp.sh
~/diagnose_tmp.sh
```

---

# Best Practices

## General Guidelines

```
✓ DO:
  ├─ Set size limit on tmpfs (never unlimited!)
  ├─ Monitor tmpfs usage regularly
  ├─ Test changes in non-production first
  ├─ Document your configuration
  ├─ Keep /tmp clean (let systemd-tmpfiles handle it)
  └─ Use /var/tmp for persistent temporary files

✗ DON'T:
  ├─ Store important data in /tmp
  ├─ Use tmpfs without size limit
  ├─ Make /tmp too large (> 30% RAM)
  ├─ Disable systemd-tmpfiles cleanup
  └─ Assume /tmp persists across reboots (with tmpfs)
```

---

## Security Best Practices

```bash
# Recommended mount options:
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev   0   0
#                                                     ↑      ↑
#                                           Ignore SUID  No devices

# High security (may break some apps):
tmpfs   /tmp   tmpfs   defaults,size=2G,mode=1777,nosuid,nodev,noexec   0   0
#                                                                  ↑
#                                                    No execution (strict!)

# Verify security options:
mount | grep /tmp
# Should show: nosuid,nodev (minimum)
```

---

## Performance Best Practices

```
For best performance:

1. Size appropriately
   ├─ Too small: Frequent "no space" errors
   └─ Too large: RAM pressure, swap usage

2. Monitor regularly
   ├─ Watch RAM usage
   └─ Check for swap activity

3. Clean proactively
   ├─ Use systemd-tmpfiles
   └─ Don't let it fill up

4. Use for right workload
   ├─ Good: Build artifacts, small temp files
   └─ Bad: Large downloads, video files
```

---

## Backup and Recovery

```bash
# 1. Always backup fstab before editing
sudo cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)

# 2. Test changes before rebooting
sudo findmnt --verify
sudo mount -a

# 3. Document your configuration
cat > ~/tmp_config_backup.txt << EOF
tmpfs Configuration Backup
==========================
Date: $(date)
Hostname: $(hostname)

fstab entry:
$(grep /tmp /etc/fstab)

systemd status:
$(systemctl status tmp.mount 2>&1 | head -5)

Current mount:
$(mount | grep /tmp)
EOF

# 4. Keep multiple backups
cp ~/tmp_config_backup.txt ~/tmp_config_backup.$(date +%Y%m%d).txt
```

---

## Capacity Planning

```
Estimating tmpfs size:

1. Monitor current /tmp usage for 1-2 weeks
   du -sh /tmp >> ~/tmp_usage.log

2. Find peak usage
   sort -rh ~/tmp_usage.log | head -1

3. Add buffer (3x minimum)
   If peak = 500MB, allocate 1.5GB

4. Consider workload changes
   ├─ Development builds: +2GB
   ├─ Package updates: +1GB
   └─ User growth: +500MB per user

5. Never exceed 30% of RAM
   8GB RAM → Max 2.4GB tmpfs
```

---

# Command Reference

## Quick Reference Card

```bash
# === Checking Current Status ===
df -hT /tmp                    # Filesystem type and usage
mount | grep /tmp              # Mount details
du -sh /tmp                    # Size of /tmp
free -h                        # RAM status

# === Temporary tmpfs Mount ===
sudo mount -t tmpfs -o size=2G,mode=1777,nosuid,nodev tmpfs /tmp
sudo umount /tmp               # Revert

# === Permanent Configuration ===
# Method 1: systemd
sudo systemctl enable tmp.mount
sudo systemctl start tmp.mount

# Method 2: fstab
echo "tmpfs /tmp tmpfs defaults,size=2G,mode=1777 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# === Monitoring ===
df -h /tmp                     # Current usage
watch -n 2 'df -h /tmp'       # Real-time monitoring
du -sh /tmp/*                  # What's using space

# === Cleanup ===
sudo systemd-tmpfiles --clean  # Auto cleanup
sudo rm -rf /tmp/*             # Manual cleanup (careful!)

# === Troubleshooting ===
sudo findmnt --verify          # Verify fstab syntax
systemctl status tmp.mount     # Check systemd service
journalctl -xe | grep tmp      # Check for errors
lsof +D /tmp                   # What's using /tmp
```

---

## Complete Command List

### Status and Information

```bash
# Filesystem type
df -T /tmp
df -hT /tmp

# Mount point details
mount | grep /tmp
findmnt /tmp

# Usage statistics
du -sh /tmp
du -sh /tmp/*

# Detailed breakdown
du -h --max-depth=1 /tmp | sort -rh

# File count
find /tmp -type f | wc -l

# Permissions
ls -ld /tmp

# Processes using /tmp
lsof +D /tmp
fuser -v /tmp
```

---

### Mounting and Unmounting

```bash
# Temporary mount (reverts on reboot)
sudo mount -t tmpfs -o size=2G,mode=1777 tmpfs /tmp

# With security options
sudo mount -t tmpfs -o size=2G,mode=1777,nosuid,nodev tmpfs /tmp

# Remount with new options
sudo mount -o remount,size=4G /tmp

# Unmount
sudo umount /tmp

# Force unmount (if busy)
sudo fuser -km /tmp
sudo umount /tmp
```

---

### Systemd Management

```bash
# Status
systemctl status tmp.mount

# Enable
sudo systemctl enable tmp.mount

# Start
sudo systemctl start tmp.mount

# Stop
sudo systemctl stop tmp.mount

# Disable
sudo systemctl disable tmp.mount

# Reload configuration
sudo systemctl daemon-reload

# View unit file
systemctl cat tmp.mount

# Edit (create override)
sudo systemctl edit tmp.mount
```

---

### Cleanup

```bash
# Systemd automated cleanup
sudo systemd-tmpfiles --clean
sudo systemd-tmpfiles --clean --dry-run

# Manual cleanup (files older than 7 days)
find /tmp -type f -mtime +7 -delete

# Delete everything (DANGEROUS!)
sudo rm -rf /tmp/*

# Restore permissions
sudo chmod 1777 /tmp
```

---

### Monitoring

```bash
# Current usage
df -h /tmp

# Watch usage (updates every 2 seconds)
watch -n 2 'df -h /tmp'

# Usage percentage
df /tmp | tail -1 | awk '{print $5}'

# Check if over threshold
USAGE=$(df /tmp | tail -1 | awk '{print int($5)}')
if [ $USAGE -gt 80 ]; then echo "WARNING: /tmp is ${USAGE}% full"; fi

# RAM impact
free -h

# Largest files
find /tmp -type f -exec du -h {} + | sort -rh | head -10
```

---

### Testing and Verification

```bash
# Verify fstab syntax
sudo findmnt --verify

# Test mount all
sudo mount -a

# Check for errors
echo $?

# Verify mount succeeded
df -hT /tmp

# Performance test
time dd if=/dev/zero of=/tmp/test bs=1M count=100

# Cleanup test
rm /tmp/test
```

---

# Summary

## Key Takeaways

```
1. /tmp can be disk-based OR tmpfs
   ├─ RHEL 9 default: Disk-based (XFS)
   ├─ Alternative: tmpfs (RAM-based)
   └─ Choice depends on use case

2. tmpfs is NOT always better
   ├─ Faster: Yes
   ├─ Uses RAM: Yes (tradeoff)
   ├─ Lost on reboot: Yes (by design)
   └─ Best for: Desktops, development, security

3. Disk-based /tmp is NOT wrong
   ├─ RHEL 9 default for good reasons
   ├─ Enterprise-friendly
   ├─ Large file support
   └─ RAM conservation

4. Testing before permanent is CRITICAL
   ├─ Use temporary mount first
   ├─ Test 24-48 hours
   ├─ Monitor RAM and performance
   └─ Verify applications work

5. Size limit is REQUIRED for tmpfs
   ├─ Never unlimited
   ├─ Recommended: 25-30% of RAM
   └─ Your system: 2GB recommended

6. Cleanup is important
   ├─ tmpfs: Automatic on reboot
   ├─ Disk: systemd-tmpfiles
   └─ Monitor regularly
```

---

## Decision Matrix

```
Use Disk-based /tmp if:
✓ Enterprise/server environment
✓ Large temporary files needed
✓ Limited RAM (< 8GB)
✓ Applications need persistence
✓ Default RHEL behavior acceptable

Use tmpfs /tmp if:
✓ Desktop/laptop system
✓ Development/build workstation
✓ Security focus (auto-wipe)
✓ Plenty of RAM (16GB+)
✓ Performance critical
✓ Small temporary files only
```

---

## Quick Start Guide

```bash
# 1. Check current status
df -hT /tmp

# 2. Test tmpfs (reversible - reverts on reboot)
sudo mount -t tmpfs -o size=2G,mode=1777,nosuid,nodev tmpfs /tmp

# 3. Use normally for 24-48 hours

# 4. If happy, make permanent:
sudo systemctl enable tmp.mount
sudo systemctl start tmp.mount

# 5. Monitor regularly:
df -h /tmp
free -h

# That's it!
```

---

## Additional Resources

```
Man Pages:
├─ man tmpfs
├─ man fstab
├─ man systemd.mount
└─ man systemd-tmpfiles

Documentation:
├─ /usr/share/doc/systemd/
├─ https://www.freedesktop.org/software/systemd/man/
└─ https://access.redhat.com/documentation/

Your Documentation:
├─ ~/tmp_migration_log.txt
├─ ~/tmp_config_backup.txt
└─ This guide!
```

---

**Good luck with your /tmp configuration!** 🚀

---

*Document Version: 1.0*  
*Last Updated: March 2026*  
*System: RHEL 9 (rhel9exam01)*  
*Author: Based on hands-on RHEL 9 learning sessions*
