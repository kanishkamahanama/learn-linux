# Complete /etc/ Directory Exploration Guide

**A comprehensive guide to understanding Linux system configuration through the /etc/ directory**

---

## Table of Contents

1. [Introduction to /etc/](#introduction-to-etc)
2. [Critical Files You Must Know](#critical-files-you-must-know)
3. [Network Configuration Files](#network-configuration-files)
4. [User and Security Files](#user-and-security-files)
5. [System Services and Startup](#system-services-and-startup)
6. [Package Management](#package-management)
7. [Storage and Filesystem](#storage-and-filesystem)
8. [Application Configuration](#application-configuration)
9. [Hands-On Exploration Labs](#hands-on-exploration-labs)
10. [Quick Reference](#quick-reference)

---

# Introduction to /etc/

## What is /etc/?

```
/etc/ = "Editable Text Configuration" or "Et Cetera"

Purpose:  System-wide configuration files
Location: /etc/
Type:     Text files (mostly)
Owner:    Root
Format:   Human-readable text (can edit with vim/nano)
```

**Key principle:** Almost ALL system configuration lives here!

---

## Why /etc/ Matters

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY /etc/ IS CRITICAL                                    │
└─────────────────────────────────────────────────────────────────────────────┘

As a System Administrator / DevOps Engineer:
═══════════════════════════════════════════════

✅ Configure network interfaces       → /etc/sysconfig/network-scripts/
✅ Manage users and groups            → /etc/passwd, /etc/group
✅ Control system services            → /etc/systemd/
✅ Set up DNS                         → /etc/resolv.conf
✅ Configure SSH access               → /etc/ssh/sshd_config
✅ Manage package repositories        → /etc/yum.repos.d/
✅ Mount filesystems at boot          → /etc/fstab
✅ Configure firewall                 → /etc/firewalld/
✅ Set hostname                       → /etc/hostname
✅ Manage sudo permissions            → /etc/sudoers

Everything you need to configure a Linux system is in /etc/!
```

---

## Initial Exploration

```bash
# Navigate to /etc/
cd /etc/

# List all files and directories
ls -la

# Count how many items
ls | wc -l
# Output: ~200-300 items (depends on installed packages)

# See directory structure
tree -L 1 /etc/ | head -30
# (if tree not installed: sudo dnf install -y tree)

# Find all .conf files
find /etc/ -name "*.conf" -type f | wc -l
# Output: ~100-200 .conf files
```

---

# Critical Files You Must Know

## 1. /etc/fstab (Filesystem Table) ⭐⭐⭐

**What it is:** Defines which filesystems mount at boot time

**Why critical:** Without this, your system won't mount partitions correctly!

---

### View Your Current fstab

```bash
cat /etc/fstab
```

**Example output (RHEL 9):**
```
#
# /etc/fstab
# Created by anaconda on [date]
#
/dev/mapper/rhel_rhel9exam01-root /                       xfs     defaults        0 0
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890 /boot           xfs     defaults        0 0
UUID=1234-5678                  /boot/efi               vfat    umask=0077,shortname=winnt 0 2
/dev/mapper/rhel_rhel9exam01-home /home                   xfs     defaults        0 0
/dev/mapper/rhel_rhel9exam01-swap none                    swap    defaults        0 0
```

---

### Understanding fstab Format

```
Column Layout:
Device          Mount Point    FS Type    Options         Dump  Pass
───────────────────────────────────────────────────────────────────────
Field 1         Field 2        Field 3    Field 4         F5    F6
```

**Line-by-line breakdown:**

```bash
/dev/mapper/rhel_rhel9exam01-root  /  xfs  defaults  0 0
 ↓                                 ↓   ↓      ↓      ↓ ↓
Device to mount              Mount   FS   Mount    Dump Pass
                             point   type options  flag order
```

---

#### Field 1: Device

```
What can be here:
═════════════════

1. Device path:
   /dev/sda1
   /dev/mapper/rhel_rhel9exam01-root

2. UUID (Preferred - survives device name changes):
   UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890

3. Label:
   LABEL=BOOT

4. Network share:
   //server/share  (for CIFS/SMB)
   server:/export  (for NFS)
```

**Find UUIDs:**
```bash
# Show all UUIDs
blkid

# Show specific device UUID
blkid /dev/sda1
# /dev/sda1: UUID="1234-5678" TYPE="vfat"

# Alternative method
ls -l /dev/disk/by-uuid/
# Shows symlinks: UUID → device
```

---

#### Field 2: Mount Point

```
Where the filesystem will be accessible:

/               ← Root filesystem
/boot           ← Boot partition
/boot/efi       ← EFI boot partition
/home           ← User home directories
none            ← For swap (not mounted as directory)
```

---

#### Field 3: Filesystem Type

```
Common types:

xfs             ← RHEL 9 default
vfat            ← FAT32 (UEFI boot)
ext4            ← Fourth extended filesystem
swap            ← Swap space
nfs             ← Network File System
cifs            ← Windows shares (SMB)
tmpfs           ← RAM-based temporary filesystem
```

---

#### Field 4: Mount Options

```
Common options:
═══════════════

defaults        ← Standard options (rw,suid,dev,exec,auto,nouser,async)
ro              ← Read-only
rw              ← Read-write
noauto          ← Don't mount automatically at boot
user            ← Allow non-root users to mount
noexec          ← Don't allow execution of binaries
nosuid          ← Ignore SUID bits
nodev           ← Don't interpret device files
relatime        ← Update access time efficiently
nofail          ← Don't fail boot if device missing

Multiple options separated by commas:
rw,relatime,noexec,nosuid
```

---

#### Field 5: Dump Flag

```
Dump backup flag:
════════════════

0 = Don't backup with dump utility
1 = Backup with dump utility

Note: dump is rarely used anymore
Most people set this to 0
```

---

#### Field 6: Pass (fsck order)

```
Filesystem check order at boot:
═══════════════════════════════

0 = Don't check
1 = Check first (root filesystem only)
2 = Check after root

Order:
  Root (/) gets 1
  Other filesystems get 2
  Swap gets 0
  Network shares get 0
```

---

### Practical fstab Examples

#### Example 1: Add tmpfs for /tmp

```bash
# Edit fstab
sudo vim /etc/fstab

# Add line:
tmpfs  /tmp  tmpfs  defaults,nodev,nosuid,noexec,size=2G  0 0
#  ↓     ↓     ↓              ↓                            ↓ ↓
# Device Mount FS    Options (security + 2GB limit)      Dump/Pass
#        point type

# Test before reboot (mount without reboot)
sudo mount -a
# Mounts all from fstab

# Verify
df -hT /tmp
# Should show tmpfs, 2G size
```

---

#### Example 2: Add NFS Share

```bash
# Add NFS share to fstab
sudo vim /etc/fstab

# Add line:
192.168.1.100:/shared  /mnt/nfs  nfs  defaults,_netdev  0 0
#      ↓                  ↓       ↓         ↓
#  NFS server:/export  Local    NFS    _netdev = wait for network
#                      mount           before mounting
#                      point

# Create mount point
sudo mkdir -p /mnt/nfs

# Mount
sudo mount -a

# Verify
df -hT /mnt/nfs
```

---

#### Example 3: Add External USB Drive (by UUID)

```bash
# Find USB drive UUID
sudo blkid /dev/sdb1
# /dev/sdb1: UUID="a1b2c3d4-5678-90ab-cdef-1234567890ab" TYPE="ext4"

# Add to fstab
sudo vim /etc/fstab

# Add line:
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /media/usb  ext4  defaults,nofail  0 2
#                                             ↓           ↓        ↓
#                                         Mount point  FS type  nofail = don't
#                                                               fail boot if
#                                                               USB not present

# Create mount point
sudo mkdir -p /media/usb

# Test
sudo mount -a
```

---

### Testing fstab Changes Safely

```bash
# IMPORTANT: Always test before rebooting!

# Method 1: Mount all from fstab
sudo mount -a
# If error, fix before reboot!

# Method 2: Test specific mount
sudo mount /mnt/test
# Reads from fstab for /mnt/test

# Method 3: Unmount and remount
sudo umount /mnt/test
sudo mount /mnt/test

# Check for errors
echo $?
# 0 = success
# non-zero = error
```

---

### Common fstab Mistakes

```bash
# MISTAKE 1: Typo in device name
/dev/sda99  /boot  xfs  defaults  0 0
#       ↑
# Doesn't exist! Boot will fail!

# FIX: Use UUID instead
UUID=correct-uuid  /boot  xfs  defaults  0 0


# MISTAKE 2: Wrong filesystem type
/dev/sda1  /boot  ext4  defaults  0 0
#                   ↑
# Wrong! It's actually xfs
# Mount will fail!

# FIX: Check with blkid
blkid /dev/sda1
# Shows correct type


# MISTAKE 3: Mount point doesn't exist
/dev/sdb1  /mnt/newdisk  ext4  defaults  0 0
#               ↑
# Directory doesn't exist!

# FIX: Create it first
sudo mkdir -p /mnt/newdisk


# MISTAKE 4: Missing nofail for optional drives
UUID=usb-drive-uuid  /media/usb  ext4  defaults  0 2
#                                           ↑
# If USB not present, boot hangs!

# FIX: Add nofail
UUID=usb-drive-uuid  /media/usb  ext4  defaults,nofail  0 2
```

---

### fstab Recovery (Emergency)

```bash
# If fstab error prevents boot:
# ═════════════════════════════

# 1. System drops to emergency mode
# 2. Root filesystem mounted read-only
# 3. Fix fstab:

# Remount root as read-write
mount -o remount,rw /

# Edit fstab
vi /etc/fstab
# Fix the error

# Test
mount -a
# Should work now

# Reboot
reboot
```

---

## 2. /etc/hostname (System Name) ⭐⭐⭐

**What it is:** Contains the system's hostname

---

### View Hostname

```bash
# Method 1: Read file directly
cat /etc/hostname
# rhel9exam01

# Method 2: Use hostname command
hostname
# rhel9exam01

# Method 3: Use hostnamectl
hostnamectl
# Static hostname: rhel9exam01
# ...
```

---

### Change Hostname

```bash
# Method 1: Edit file directly
sudo vim /etc/hostname
# Change to: myserver.example.com
# Reboot required

# Method 2: Use hostnamectl (preferred - no reboot)
sudo hostnamectl set-hostname myserver.example.com

# Verify
hostname
# myserver.example.com

# See details
hostnamectl status
```

---

### Hostname Best Practices

```
Good hostnames:
═══════════════
webserver01.company.com
db-primary.prod.local
rhel9-test01

Bad hostnames:
══════════════
server                    ← Too generic
my_server                 ← Underscore not recommended
SERVER-01.EXAMPLE.COM     ← All caps not standard
server with spaces        ← Spaces not allowed
```

---

## 3. /etc/hosts (Static Host Resolution) ⭐⭐⭐

**What it is:** Maps hostnames to IP addresses (local DNS)

---

### View /etc/hosts

```bash
cat /etc/hosts
```

**Default RHEL 9 content:**
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

---

### Understanding /etc/hosts Format

```
Format:
IP_ADDRESS    HOSTNAME    [ALIASES...]

Examples:
127.0.0.1     localhost   localhost.localdomain
192.168.1.10  webserver   web.example.com web
192.168.1.20  database    db.example.com db
```

---

### Practical Examples

```bash
# Edit hosts file
sudo vim /etc/hosts

# Add custom entries:
192.168.1.100    storage.local    storage
192.168.1.101    backup.local     backup
10.0.0.50        gitlab.company.com gitlab
172.16.0.10      jenkins.company.com jenkins

# Save and test
ping storage.local
# Should resolve to 192.168.1.100

ping gitlab
# Should resolve to 10.0.0.50
```

---

### Common Use Cases

```bash
# Use Case 1: Testing before DNS is set up
# ════════════════════════════════════════
# You're setting up a new web server
192.168.1.200    newweb.company.com

# Can now test: curl http://newweb.company.com


# Use Case 2: Override DNS for testing
# ═════════════════════════════════════
# Block a website
0.0.0.0    ads.example.com

# Redirect to local dev server
127.0.0.1    api.company.com


# Use Case 3: Lab environment shortcuts
# ══════════════════════════════════════
192.168.1.10    node1
192.168.1.11    node2
192.168.1.12    node3

# Now: ssh node1 (instead of ssh 192.168.1.10)


# Use Case 4: Docker/Kubernetes services
# ═══════════════════════════════════════
127.0.0.1    mysql.local
127.0.0.1    redis.local
```

---

### Host Resolution Order

```
When you type: ping webserver

Linux resolves in this order:
═════════════════════════════

1. /etc/hosts              ← Check here FIRST
   ↓ (if not found)
2. /etc/resolv.conf        ← DNS servers
   ↓ (queries DNS)
3. DNS server response
   ↓
4. Result

You can change this order in: /etc/nsswitch.conf
(We'll cover that later)
```

---

## 4. /etc/resolv.conf (DNS Configuration) ⭐⭐⭐

**What it is:** Configures DNS servers for name resolution

---

### View /etc/resolv.conf

```bash
cat /etc/resolv.conf
```

**Example output:**
```
# Generated by NetworkManager
search example.com
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 192.168.1.1
```

---

### Understanding resolv.conf Format

```
Directive: search
═════════════════
search example.com company.local
↓
When you type: ping webserver
Linux tries:
  1. webserver.example.com
  2. webserver.company.local


Directive: nameserver
══════════════════════
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 192.168.1.1
↓
DNS servers to query (in order)
Up to 3 nameservers supported


Directive: domain (alternative to search)
═════════════════════════════════════════
domain example.com
↓
Default domain for unqualified names
```

---

### IMPORTANT: NetworkManager Managed

```bash
⚠️ WARNING on RHEL 9:
/etc/resolv.conf is typically managed by NetworkManager!

# Check if managed:
ls -l /etc/resolv.conf
# lrwxrwxrwx. 1 root root 39 Mar 15 10:30 /etc/resolv.conf -> ../run/NetworkManager/resolv.conf
#  ↑
# Symlink! Managed by NetworkManager

# If you edit /etc/resolv.conf directly:
# Changes will be OVERWRITTEN by NetworkManager!
```

---

### Configure DNS Properly (RHEL 9)

```bash
# Method 1: Using nmcli (NetworkManager CLI)
# ═══════════════════════════════════════════

# Show current connection
nmcli connection show

# Set DNS servers
sudo nmcli connection modify ens160 ipv4.dns "8.8.8.8 8.8.4.4"

# Set search domain
sudo nmcli connection modify ens160 ipv4.dns-search "example.com"

# Restart connection
sudo nmcli connection up ens160

# Verify
cat /etc/resolv.conf
# Should show your DNS servers


# Method 2: Using nmtui (Text UI)
# ═══════════════════════════════
sudo nmtui
# Navigate to: Edit a connection
# Select your interface
# Set DNS servers
# Save and exit


# Method 3: Edit interface config file
# ═════════════════════════════════════
sudo vim /etc/sysconfig/network-scripts/ifcfg-ens160

# Add lines:
DNS1=8.8.8.8
DNS2=8.8.4.4
DOMAIN=example.com

# Restart NetworkManager
sudo systemctl restart NetworkManager
```

---

### Test DNS Resolution

```bash
# Method 1: ping
ping -c 1 google.com
# Tests if DNS works

# Method 2: nslookup
nslookup google.com
# Shows which DNS server answered

# Method 3: dig (detailed)
dig google.com

# Method 4: host
host google.com

# Test specific DNS server
nslookup google.com 8.8.8.8
dig @8.8.8.8 google.com
```

---

## 5. /etc/passwd (User Accounts) ⭐⭐⭐

**What it is:** Contains user account information

**You already explored the format!** Let's review and expand.

---

### View /etc/passwd

```bash
cat /etc/passwd | grep -E "root|kanishka"
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
kanishka:x:1000:1000:kanishka:/home/kanishka:/bin/bash
```

---

### Format Review

```
username:password:UID:GID:comment:home:shell

root:x:0:0:root:/root:/bin/bash
 │   │ │ │  │     │      │
 │   │ │ │  │     │      └─ Login shell
 │   │ │ │  │     └─ Home directory
 │   │ │ │  └─ Comment/full name
 │   │ │ └─ Primary GID
 │   │ └─ UID
 │   └─ Password (x = in /etc/shadow)
 └─ Username
```

---

### Create User Manually

```bash
# Add user to /etc/passwd
sudo vim /etc/passwd

# Add line:
testuser:x:2001:2001:Test User:/home/testuser:/bin/bash

# Create home directory
sudo mkdir /home/testuser
sudo chown testuser:testuser /home/testuser

# Set password
sudo passwd testuser

# This updates /etc/shadow automatically
```

**Better method:** Use `useradd` command (does all this automatically)

```bash
sudo useradd testuser
sudo passwd testuser
```

---

### Understanding Special Users

```bash
# System users (UID < 1000):
cat /etc/passwd | awk -F: '$3 < 1000 {print $1, $3}' | head -10

# Output:
root 0             ← Superuser
bin 1              ← Binary files owner
daemon 2           ← System daemons
adm 3              ← Administrative files
lp 4               ← Printer system
sync 5             ← Sync command
shutdown 6         ← Shutdown command
halt 7             ← Halt command
mail 8             ← Mail system
operator 9         ← Operator user

# These are system accounts, not real users!
```

---

## 6. /etc/group (Group Information) ⭐⭐

**What it is:** Contains group definitions

---

### View /etc/group

```bash
cat /etc/group | grep -E "wheel|kanishka"
```

**Output:**
```
wheel:x:10:kanishka
kanishka:x:1000:
```

---

### Format

```
groupname:password:GID:member1,member2,member3

wheel:x:10:kanishka
  │   │ │    │
  │   │ │    └─ Group members (comma-separated)
  │   │ └─ GID (Group ID)
  │   └─ Password (x = not used)
  └─ Group name
```

---

### Practical Examples

```bash
# See all groups
cat /etc/group

# See your groups
groups
# kanishka wheel

# See another user's groups
groups root
# root

# Add user to group
sudo usermod -aG wheel newuser
# -a = append
# -G = supplementary groups

# Create new group
sudo groupadd developers

# Add users to developers group
sudo usermod -aG developers user1
sudo usermod -aG developers user2

# Verify
grep developers /etc/group
# developers:x:2002:user1,user2
```

---

## 7. /etc/shadow (Password Hashes) ⭐⭐⭐

**What it is:** Stores encrypted passwords (root-only readable)

---

### View /etc/shadow (requires root)

```bash
sudo cat /etc/shadow | grep kanishka
```

**Output:**
```
kanishka:$6$randomsalt$hashedhashehashed...:19800:0:99999:7:::
```

---

### Format

```
username:encrypted_password:lastchange:min:max:warn:inactive:expire:reserved

kanishka:$6$salt$hash...:19800:0:99999:7:::
   │          │          │    │   │    │ │││
   │          │          │    │   │    │ ││└─ Reserved
   │          │          │    │   │    │ │└─ Account expiration
   │          │          │    │   │    │ └─ Inactive period
   │          │          │    │   │    └─ Warning period (days)
   │          │          │    │   └─ Maximum days between changes
   │          │          │    └─ Minimum days between changes
   │          │          └─ Last password change (days since 1970-01-01)
   │          └─ Encrypted password ($6$ = SHA-512)
   └─ Username
```

---

### Password Aging Policy

```bash
# Set password to expire in 90 days
sudo chage -M 90 kanishka

# Set minimum days between password changes
sudo chage -m 7 kanishka

# Set warning 14 days before expiration
sudo chage -W 14 kanishka

# See password aging info
sudo chage -l kanishka

# Output:
# Last password change                    : Mar 20, 2026
# Password expires                        : Jun 18, 2026
# Password inactive                       : never
# Account expires                         : never
# Minimum number of days between password change     : 7
# Maximum number of days between password change     : 90
# Number of days of warning before password expires  : 14
```

---

### Lock/Unlock User Account

```bash
# Lock account (adds ! to password hash)
sudo passwd -l testuser
sudo cat /etc/shadow | grep testuser
# testuser:!$6$hash...:19800:0:99999:7:::
#          ↑
#      Exclamation mark = locked

# Unlock account
sudo passwd -u testuser

# Check lock status
sudo passwd -S testuser
# testuser PS 2026-03-20 7 90 14 -1 (Password set, SHA512 crypt.)
#          ↑
#       PS = Password Set
#       LK = Locked
```

---

## 8. /etc/sudoers (Sudo Permissions) ⭐⭐⭐

**What it is:** Controls who can use sudo

**⚠️ NEVER edit directly! Always use `visudo`**

---

### View sudoers

```bash
# View (read-only)
sudo cat /etc/sudoers

# Edit (ALWAYS use visudo - validates syntax)
sudo visudo
```

---

### Understanding sudoers Format

```bash
# Basic format:
user    host=(runas)    commands

# Examples from default file:
root    ALL=(ALL)       ALL
#  ↓     ↓    ↓          ↓
# User  From Run as   Commands
#       hosts users

# Translation:
# root can run ANY command on ANY host as ANY user
```

---

### Common sudoers Configurations

```bash
# Allow user full sudo access
kanishka    ALL=(ALL)       ALL

# Allow user sudo without password (DANGEROUS!)
kanishka    ALL=(ALL)       NOPASSWD: ALL

# Allow group wheel full sudo access
%wheel      ALL=(ALL)       ALL
# ↑
# % = group

# Allow user specific command only
backup_user ALL=(ALL)       NOPASSWD: /usr/bin/rsync

# Allow user multiple specific commands
webadmin    ALL=(ALL)       /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd

# Allow user to run commands as specific user
developer   ALL=(www-data)  /usr/bin/vim /var/www/html/*
```

---

### Add User to sudo

```bash
# Method 1: Add to wheel group (RHEL/CentOS way)
sudo usermod -aG wheel kanishka

# Method 2: Add to /etc/sudoers.d/ (preferred)
sudo visudo -f /etc/sudoers.d/kanishka

# Add line:
kanishka ALL=(ALL) ALL

# Save (Ctrl+X in nano, :wq in vim)

# Test
sudo -l
# Shows allowed commands
```

---

### sudoers Best Practices

```bash
# ✅ GOOD:
# Use /etc/sudoers.d/ for custom configs
sudo visudo -f /etc/sudoers.d/myapp

# ✅ GOOD:
# Limit commands to specific paths
user ALL=(ALL) /usr/bin/systemctl restart myapp

# ❌ BAD:
# Editing /etc/sudoers directly
vim /etc/sudoers  # WRONG! Use visudo

# ❌ BAD:
# Allowing ALL commands
user ALL=(ALL) NOPASSWD: ALL  # Security risk!

# ❌ BAD:
# Using wildcards carelessly
user ALL=(ALL) /bin/*  # Too broad!
```

---

## Quick Navigation Commands

```bash
# Jump to /etc/
cd /etc/

# Go back to previous directory
cd -

# Go to home
cd ~

# List all .conf files
ls /etc/*.conf

# Find configuration files
find /etc/ -name "*.conf" -type f

# Search for text in /etc/
grep -r "search_term" /etc/ 2>/dev/null

# Show recently modified files
ls -lt /etc/ | head -20
```

---

# Network Configuration Files

## RHEL 9 uses NetworkManager

On RHEL 9, network configuration is managed by NetworkManager. Let's explore the important files.

---

## 1. /etc/sysconfig/network-scripts/ ⭐⭐⭐

**What it is:** Network interface configuration files (legacy but still used)

---

### List Interface Configuration Files

```bash
ls -la /etc/sysconfig/network-scripts/

# Output:
ifcfg-ens160    ← Your primary network interface
ifcfg-lo        ← Loopback interface
```

---

### View Interface Configuration

```bash
cat /etc/sysconfig/network-scripts/ifcfg-ens160
```

**Example content:**
```
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=ens160
UUID=12345678-90ab-cdef-1234-567890abcdef
DEVICE=ens160
ONBOOT=yes
```

---

### Understanding Interface Configuration

```bash
# Key settings:
# ════════════

DEVICE=ens160           ← Interface name
TYPE=Ethernet           ← Interface type
ONBOOT=yes              ← Start at boot
BOOTPROTO=dhcp          ← Get IP from DHCP
                          (or 'none' for static)

# For static IP, you'd have:
BOOTPROTO=none
IPADDR=192.168.1.100
NETMASK=255.255.255.0
# or
PREFIX=24
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```

---

### Configure Static IP

```bash
# Method 1: Edit interface file
# ══════════════════════════════
sudo vim /etc/sysconfig/network-scripts/ifcfg-ens160

# Change from DHCP to static:
BOOTPROTO=none
IPADDR=192.168.1.100
PREFIX=24
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=8.8.4.4

# Restart NetworkManager
sudo systemctl restart NetworkManager

# OR restart specific interface
sudo nmcli connection down ens160
sudo nmcli connection up ens160


# Method 2: Use nmcli (preferred)
# ═══════════════════════════════
sudo nmcli connection modify ens160 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4"

sudo nmcli connection up ens160


# Method 3: Use nmtui (easiest)
# ═════════════════════════════
sudo nmtui
# Select: Edit a connection
# Choose: ens160
# Change IPv4 CONFIGURATION to Manual
# Set IP, Gateway, DNS
# Save and exit
```

---

### Verify Network Configuration

```bash
# Check IP address
ip addr show ens160

# Check routing
ip route show

# Check DNS
cat /etc/resolv.conf

# Test connectivity
ping -c 4 8.8.8.8        # Test internet
ping -c 4 google.com     # Test DNS

# Show NetworkManager connections
nmcli connection show

# Show device status
nmcli device status
```

---

## 2. /etc/NetworkManager/ ⭐⭐

**What it is:** NetworkManager configuration directory

```bash
ls -la /etc/NetworkManager/

# Key files:
NetworkManager.conf     ← Main config
system-connections/     ← Connection profiles
```

---

### View NetworkManager Configuration

```bash
cat /etc/NetworkManager/NetworkManager.conf
```

**Default content:**
```
[main]
plugins=ifcfg-rh

[logging]
```

---

## 3. /etc/firewalld/ ⭐⭐⭐

**What it is:** Firewall configuration (firewalld)

---

### Explore Firewall Configuration

```bash
ls -la /etc/firewalld/

# Key items:
firewalld.conf          ← Main config
zones/                  ← Firewall zones
services/               ← Service definitions
```

---

### View Firewall Status

```bash
# Check if firewalld is running
sudo systemctl status firewalld

# Show active zones
sudo firewall-cmd --get-active-zones

# Show default zone
sudo firewall-cmd --get-default-zone
# public

# List all rules in public zone
sudo firewall-cmd --zone=public --list-all
```

---

### Common Firewall Operations

```bash
# Allow HTTP
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

# Allow specific port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Allow port range
sudo firewall-cmd --permanent --add-port=5000-5100/tcp
sudo firewall-cmd --reload

# Remove port
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload

# List all allowed services
sudo firewall-cmd --list-services

# List all allowed ports
sudo firewall-cmd --list-ports
```

---

# System Services and Startup

## 1. /etc/systemd/ ⭐⭐⭐

**What it is:** Systemd configuration directory

---

### Explore systemd Directory

```bash
ls -la /etc/systemd/

# Key directories:
system/           ← Custom service units
user/             ← User service units
```

---

### View Custom Services

```bash
ls -la /etc/systemd/system/

# Shows:
multi-user.target.wants/    ← Services enabled for multi-user
default.target              ← Default target (usually symlink)
*.service                    ← Custom service files
```

---

### Create Custom Service

```bash
# Example: Create a simple service
sudo vim /etc/systemd/system/myapp.service
```

**Content:**
```
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/start.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl status myapp.service
```

---

## 2. /etc/default/ ⭐⭐

**What it is:** Default settings for various services

---

### Explore /etc/default/

```bash
ls -la /etc/default/

# Common files:
grub              ← GRUB bootloader defaults
useradd           ← Default useradd settings
```

---

### View GRUB Defaults

```bash
cat /etc/default/grub
```

**Example:**
```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/rhel_rhel9exam01-swap rd.lvm.lv=rhel_rhel9exam01/root rd.lvm.lv=rhel_rhel9exam01/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
```

**After editing, regenerate GRUB config:**
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

# Package Management

## 1. /etc/yum.repos.d/ ⭐⭐⭐

**What it is:** Repository definitions for DNF/YUM

---

### List Repository Files

```bash
ls -la /etc/yum.repos.d/

# Example output:
redhat.repo
epel.repo     # If EPEL is installed
```

---

### View Repository Configuration

```bash
cat /etc/yum.repos.d/redhat.repo | head -20
```

**Example format:**
```
[rhel-9-baseos-rpms]
name = Red Hat Enterprise Linux 9 - BaseOS
baseurl = https://cdn.redhat.com/content/dist/rhel9/$releasever/$basearch/baseos/os
enabled = 1
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

---

### Add Custom Repository

```bash
# Create new repo file
sudo vim /etc/yum.repos.d/myrepo.repo
```

**Content:**
```
[myrepo]
name=My Custom Repository
baseurl=http://repo.example.com/rhel9/
enabled=1
gpgcheck=0
```

**Update repo cache:**
```bash
sudo dnf clean all
sudo dnf makecache
```

---

### List Enabled Repositories

```bash
# List all repos
dnf repolist all

# List enabled only
dnf repolist

# Enable/disable repo temporarily
dnf --enablerepo=epel install package
dnf --disablerepo=* --enablerepo=rhel-9-baseos-rpms install package
```

---

## 2. /etc/dnf/ ⭐⭐

**What it is:** DNF package manager configuration

---

### View DNF Configuration

```bash
cat /etc/dnf/dnf.conf
```

**Example:**
```
[main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
```

**Useful settings:**
```
# Keep downloaded packages
keepcache=1

# Exclude packages from updates
exclude=kernel*

# Set install limit for kernels
installonly_limit=3
```

---

# Storage and Filesystem

## 1. /etc/lvm/ ⭐⭐

**What it is:** LVM configuration directory

---

### Explore LVM Configuration

```bash
ls -la /etc/lvm/

# Key files:
lvm.conf          ← Main LVM config
backup/           ← LVM metadata backups
archive/          ← LVM configuration archive
```

---

### View LVM Configuration

```bash
sudo cat /etc/lvm/lvm.conf | grep -v "^#" | grep -v "^$" | head -20
```

---

### LVM Metadata Backups

```bash
# List backups
sudo ls -lh /etc/lvm/backup/

# View backup
sudo cat /etc/lvm/backup/rhel_rhel9exam01

# These are automatic backups of your VG configuration!
```

---

## 2. /etc/crypttab ⭐

**What it is:** Encrypted filesystem table (like fstab for encryption)

---

### View crypttab (if you have encrypted partitions)

```bash
cat /etc/crypttab

# Example (if you had LUKS encryption):
# luks-<uuid>  UUID=<device-uuid>  none
```

---

# Application Configuration

## Common Application Config Directories

```bash
/etc/ssh/              ← SSH configuration
/etc/httpd/            ← Apache web server
/etc/nginx/            ← Nginx web server
/etc/mysql/            ← MySQL database
/etc/postgresql/       ← PostgreSQL database
/etc/docker/           ← Docker
/etc/kubernetes/       ← Kubernetes
/etc/ansible/          ← Ansible
```

---

## 1. /etc/ssh/ ⭐⭐⭐

**What it is:** SSH server and client configuration

---

### View SSH Server Configuration

```bash
cat /etc/ssh/sshd_config | grep -v "^#" | grep -v "^$"
```

**Important settings:**
```
Port 22                    ← SSH port
PermitRootLogin no         ← Disable root login (recommended)
PasswordAuthentication yes ← Allow password login
PubkeyAuthentication yes   ← Allow key-based login
```

---

### Common SSH Hardening

```bash
sudo vim /etc/ssh/sshd_config

# Recommended changes:
Port 2222                           # Change default port
PermitRootLogin no                  # Disable root login
PasswordAuthentication no           # Force key-based auth only
MaxAuthTries 3                      # Limit auth attempts
ClientAliveInterval 300             # Timeout idle sessions
ClientAliveCountMax 2

# Restart SSH
sudo systemctl restart sshd

# Test (don't close current session!)
ssh -p 2222 user@localhost
```

---

# Hands-On Exploration Labs

## Lab 1: Explore Your /etc/ Directory

```bash
# Step 1: Count files
ls /etc/ | wc -l

# Step 2: Find largest files
du -sh /etc/* | sort -hr | head -10

# Step 3: Find recently modified
ls -lt /etc/ | head -20

# Step 4: Find all .conf files
find /etc/ -name "*.conf" -type f

# Step 5: Search for your username
grep -r "kanishka" /etc/ 2>/dev/null

# Step 6: Check which files you can edit
find /etc/ -writable -type f 2>/dev/null
```

---

## Lab 2: Create Test fstab Entry

```bash
# Step 1: Create mount point
sudo mkdir /mnt/test-tmpfs

# Step 2: Add to fstab
echo "tmpfs  /mnt/test-tmpfs  tmpfs  defaults,size=100M  0 0" | sudo tee -a /etc/fstab

# Step 3: Mount
sudo mount -a

# Step 4: Verify
df -hT /mnt/test-tmpfs

# Step 5: Test write
echo "test" | sudo tee /mnt/test-tmpfs/testfile

# Step 6: Cleanup (remove from fstab when done)
sudo vim /etc/fstab  # Remove the line
sudo umount /mnt/test-tmpfs
```

---

## Lab 3: Network Configuration Practice

```bash
# Step 1: View current config
nmcli connection show ens160

# Step 2: Add secondary IP
sudo nmcli connection modify ens160 +ipv4.addresses 192.168.1.200/24

# Step 3: Apply
sudo nmcli connection up ens160

# Step 4: Verify
ip addr show ens160

# Step 5: Remove (cleanup)
sudo nmcli connection modify ens160 -ipv4.addresses 192.168.1.200/24
sudo nmcli connection up ens160
```

---

## Lab 4: Create Custom Systemd Service

```bash
# Step 1: Create script
cat << 'EOF' | sudo tee /opt/hello-service.sh
#!/bin/bash
while true; do
    echo "Hello from my service at $(date)" >> /tmp/hello.log
    sleep 60
done
EOF

sudo chmod +x /opt/hello-service.sh

# Step 2: Create service
cat << 'EOF' | sudo tee /etc/systemd/system/hello.service
[Unit]
Description=Hello Service
After=network.target

[Service]
Type=simple
ExecStart=/opt/hello-service.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Step 3: Start service
sudo systemctl daemon-reload
sudo systemctl start hello.service
sudo systemctl status hello.service

# Step 4: Check log
tail -f /tmp/hello.log

# Step 5: Cleanup
sudo systemctl stop hello.service
sudo systemctl disable hello.service
sudo rm /etc/systemd/system/hello.service
sudo systemctl daemon-reload
```

---

# Quick Reference

## Essential /etc/ Files Cheat Sheet

```
Configuration File                  Purpose
══════════════════════════════════  ═══════════════════════════════════════
/etc/fstab                          Filesystem mount table
/etc/hostname                       System hostname
/etc/hosts                          Static host resolution
/etc/resolv.conf                    DNS servers (NetworkManager managed)
/etc/passwd                         User accounts
/etc/group                          Group definitions
/etc/shadow                         Password hashes
/etc/sudoers                        Sudo permissions
/etc/ssh/sshd_config                SSH server config
/etc/sysconfig/network-scripts/     Network interface configs
/etc/systemd/system/                Custom systemd services
/etc/yum.repos.d/                   Package repositories
/etc/lvm/                           LVM configuration
/etc/default/grub                   GRUB bootloader defaults
/etc/firewalld/                     Firewall configuration
```

---

## Common Commands for /etc/ Exploration

```bash
# View file
cat /etc/filename

# Edit file (always backup first!)
sudo cp /etc/filename /etc/filename.bak
sudo vim /etc/filename

# Search for text
grep -r "search_term" /etc/

# Find files modified today
find /etc/ -mtime 0 -type f

# Find files by name
find /etc/ -name "*.conf"

# Compare two files
diff /etc/file1 /etc/file2

# Show file permissions
ls -l /etc/filename

# Validate syntax (for specific files)
sudo visudo -c              # For sudoers
sudo sshd -t                # For sshd_config
sudo mount -a               # For fstab (dry run)
```

---

## Backup Strategy for /etc/

```bash
# Method 1: Simple backup
sudo tar -czf /root/etc-backup-$(date +%F).tar.gz /etc/

# Method 2: With etckeeper (version control for /etc/)
sudo dnf install -y etckeeper
sudo etckeeper init
sudo etckeeper commit "Initial commit"

# After making changes:
sudo etckeeper commit "Changed SSH port"

# View history
cd /etc/
sudo git log

# Restore previous version
sudo git checkout HEAD~1 ssh/sshd_config
```

---

## Next Steps

After mastering /etc/, explore:

1. **/var/log/** - System logs
2. **/sys/class/net/** - Network interfaces
3. **/proc/sys/** - Kernel parameters
4. **/usr/lib/systemd/** - System service units

---

## Summary

### What You've Learned

```
✅ /etc/ contains ALL system configuration
✅ /etc/fstab - Filesystem mounts
✅ /etc/hostname - System name
✅ /etc/hosts - Static DNS
✅ /etc/resolv.conf - DNS servers
✅ /etc/passwd, /etc/group, /etc/shadow - Users
✅ /etc/sudoers - Sudo permissions
✅ /etc/sysconfig/network-scripts/ - Network config
✅ /etc/systemd/ - System services
✅ /etc/yum.repos.d/ - Package repos
✅ /etc/ssh/ - SSH configuration
✅ /etc/firewalld/ - Firewall rules

Key Principle:
Before editing any file in /etc/:
1. Make a backup
2. Understand the format
3. Test changes before reboot
4. Document what you changed
```

---

## License

This guide is provided under the MIT License.

---

**Last Updated:** March 2026  
**Version:** 1.0  
**System:** RHEL 9 (rhel9exam01)

---

**Happy Exploring!** 🚀
