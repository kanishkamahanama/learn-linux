# Linux Daily Commands - Essential Cheat Sheet

**Quick Reference for System Administration**  
Based on: Boot process, filesystems, disk management, memory, and swap

---

## 📊 SYSTEM OVERVIEW (Start Here Every Day!)

### Quick Health Check (One-Liner)
```bash
# Complete system status at a glance
free -h && df -hT && uptime && who

# Or separately:
free -h              # Memory and swap usage
df -hT               # Disk space and filesystem types
uptime               # System uptime and load
who                  # Who's logged in
```

---

## 💾 DISK & STORAGE

### Check Disk Space
```bash
# Simple view
df -h                          # Human-readable disk usage

# With filesystem types
df -hT                         # Shows xfs, vfat, etc.

# Specific directory
df -h /home                    # Check /home partition

# All filesystems (including virtual)
df -a                          # Shows tmpfs, devtmpfs, etc.
```

### Disk Layout & Partitions
```bash
# Tree view of disks (BEST!)
lsblk                          # Shows partition hierarchy

# With filesystems and mount points
lsblk -f                       # Most detailed view

# With sizes and types
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

# Disk UUIDs (for /etc/fstab)
sudo blkid                     # Show all UUIDs
sudo blkid /dev/sda1           # Specific partition
```

### Find What's Using Disk Space
```bash
# Top 10 largest directories
du -sh /* 2>/dev/null | sort -rh | head -10

# Current directory usage
du -sh *                       # Size of each item

# Specific directory
du -sh /var/log                # Check log size
du -h --max-depth=1 /home      # One level deep
```

### Check Mount Points
```bash
# Show all mounts
mount | grep "^/dev"           # Real devices only
mount | column -t              # Formatted view

# What's mounted where
findmnt                        # Tree view (pretty!)
findmnt -t xfs,ext4            # Specific filesystem types
```

---

## 🧠 MEMORY & SWAP

### Memory Status
```bash
# Quick memory check (BEST!)
free -h                        # Human-readable

# Detailed breakdown
free -h -w                     # Wide format (separates cache)

# Watch memory in real-time
watch -n 2 free -h             # Updates every 2 seconds
```

### Swap Status
```bash
# Swap usage
swapon --show                  # Shows swap devices

# Swap activity (IMPORTANT!)
vmstat 2 5                     # 5 samples, 2 sec apart
# Look at 'si' and 'so' columns:
#   si = swap in  (should be 0)
#   so = swap out (should be 0)

# Swap details
cat /proc/swaps                # Kernel's view
grep -i swap /proc/meminfo     # Memory info
```

### Check Swappiness
```bash
# Current swappiness value
cat /proc/sys/vm/swappiness    # Usually 30 or 60

# Change temporarily
sudo sysctl vm.swappiness=10

# Change permanently
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Top Memory Users
```bash
# Processes by memory usage
ps aux --sort=-%mem | head -10

# Or use top
top                            # Press 'M' to sort by memory
# Press 'q' to quit

# Which processes are in swap
for f in /proc/*/status; do 
    awk '/VmSwap|Name/{printf $2" "$3}END{print""}' $f
done | sort -k2 -rn | head -10
```

---

## 📁 FILESYSTEM & DIRECTORIES

### Navigate & List
```bash
# List with details
ls -lah                        # All files, human-readable
ls -lh /boot                   # Specific directory

# Find symbolic links
ls -la / | grep "^l"           # Links in root
ls -la /usr/bin | grep "^l"    # Links in /usr/bin

# Follow symlinks
readlink /bin                  # Show where it points
readlink -f /bin/ls            # Full resolved path
```

### File Information
```bash
# File type and details
file /bin/ls                   # What kind of file
stat /etc/passwd               # Detailed file stats

# Check if symlink
[ -L /bin ] && echo "Symlink" || echo "Not a symlink"
```

### Disk & Inode Usage
```bash
# Inode usage (can run out even with free space!)
df -i                          # Show inode usage

# Find files by size
find /var/log -type f -size +100M    # Files > 100MB
find /home -type f -size +1G         # Files > 1GB
```

---

## 🔍 PROCESS & SYSTEM MONITORING

### System Load
```bash
# Quick load check
uptime                         # Load averages

# Detailed process view
top                            # Interactive (press 'q' to quit)
# Useful keys in top:
#   M = sort by memory
#   P = sort by CPU
#   k = kill process
#   q = quit

# One-time snapshot
ps aux                         # All processes
ps aux --sort=-%cpu | head -10 # Top CPU users
ps aux --sort=-%mem | head -10 # Top memory users
```

### I/O and Disk Activity
```bash
# Disk I/O statistics
iostat -x 1                    # Every 1 second
# Look at %util column (should be < 80%)

# Virtual memory stats
vmstat 2 5                     # 5 samples, 2 seconds apart
# r = run queue (should be < CPU count)
# b = blocked processes (should be low)
# si/so = swap in/out (should be 0)
# wa = I/O wait (should be < 10%)
```

---

## 🥾 BOOT & KERNEL

### Kernel Information
```bash
# Current kernel
uname -r                       # Kernel version
uname -a                       # All system info

# Installed kernels
ls /boot/vmlinuz-*             # List kernels
rpm -qa | grep kernel          # Kernel packages

# Boot partition usage
df -h /boot                    # Check space
du -sh /boot/*                 # What's using space
```

### GRUB & Boot
```bash
# List boot entries
sudo awk -F\' '/menuentry / {print $2}' /boot/grub2/grub.cfg

# Check default kernel
sudo grub2-editenv list        # Shows saved_entry

# Set default kernel
sudo grub2-set-default 0       # First entry

# Regenerate GRUB config (after changes)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### System Boot Info
```bash
# Last boot time
uptime -s                      # When system booted

# Boot messages
dmesg | less                   # Kernel ring buffer
dmesg | grep -i error          # Boot errors

# System logs
journalctl -b                  # Current boot logs
journalctl -b -1               # Previous boot
journalctl -f                  # Follow logs (like tail -f)
```

---

## 📦 PACKAGE MANAGEMENT

### Query Packages
```bash
# List all installed
rpm -qa                        # All packages
rpm -qa | grep kernel          # Specific packages

# Package info
rpm -qi package_name           # Detailed info
rpm -ql package_name           # Files in package

# Which package owns a file
rpm -qf /usr/bin/ls            # Finds: coreutils

# Search available packages
dnf search keyword             # Search repos
dnf info package_name          # Package details
```

### Install/Update/Remove
```bash
# Install
sudo dnf install package_name

# Update specific package
sudo dnf update package_name

# Update all
sudo dnf update

# Remove package
sudo dnf remove package_name

# Clean up old kernels (keeps latest 3)
sudo dnf remove --oldinstallonly --setopt installonly_limit=3 kernel
```

### Repository Management
```bash
# List repos
dnf repolist                   # Enabled repos
dnf repolist all               # All repos

# Clear cache
sudo dnf clean all
```

---

## 🔐 USER & PERMISSIONS

### User Information
```bash
# Current user
whoami                         # Your username
id                             # Your UID/GID/groups

# User details
grep username /etc/passwd      # User account info
groups username                # User's groups

# Who's logged in
who                            # Current users
w                              # Detailed (with activity)
last                           # Login history
```

### File Permissions
```bash
# Change permissions
chmod 755 file                 # rwxr-xr-x
chmod +x script.sh             # Add execute

# Change ownership
sudo chown user:group file
sudo chown -R user:group /path # Recursive
```

---

## 🌐 NETWORK (BONUS)

### Quick Network Check
```bash
# IP address
ip addr show                   # All interfaces
ip addr show eth0              # Specific interface

# Test connectivity
ping -c 4 8.8.8.8              # Ping Google DNS
ping -c 4 google.com           # Ping by hostname

# DNS lookup
nslookup google.com
host google.com

# Check ports
sudo ss -tulnp                 # Listening ports
sudo netstat -tulnp            # Alternative (if installed)
```

---

## 🛠️ SYSTEM MAINTENANCE

### Cleanup Commands
```bash
# Clear old kernels
sudo dnf remove --oldinstallonly --setopt installonly_limit=3 kernel

# Clear package cache
sudo dnf clean all

# Find and remove old logs (careful!)
sudo journalctl --vacuum-time=30d    # Keep 30 days
sudo journalctl --vacuum-size=500M   # Keep 500MB

# Clear user cache (your home)
rm -rf ~/.cache/*
```

### System Updates
```bash
# Check for updates
dnf check-update

# Update all packages
sudo dnf update -y

# Reboot if needed (after kernel update)
sudo reboot
```

---

## 🔄 LVM MANAGEMENT

### LVM Status
```bash
# Quick overview
sudo pvs                       # Physical volumes
sudo vgs                       # Volume groups
sudo lvs                       # Logical volumes

# Detailed info
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay

# Specific LV
sudo lvdisplay /dev/rhel_rhel9exam01/root
```

### Extend Logical Volume (Advanced)
```bash
# Extend root by 5GB
sudo lvextend -L +5G /dev/rhel_rhel9exam01/root
sudo xfs_growfs /              # For xfs filesystem
# or
sudo resize2fs /dev/rhel_rhel9exam01/root  # For ext4

# Extend to use all free space
sudo lvextend -l +100%FREE /dev/rhel_rhel9exam01/home
sudo xfs_growfs /home
```

---

## 📋 CONFIGURATION FILES

### Important Files to Know
```bash
# System configuration
cat /etc/os-release            # OS version info
cat /etc/hostname              # System hostname
cat /etc/hosts                 # Static DNS entries
cat /etc/resolv.conf           # DNS servers

# Boot configuration
cat /etc/default/grub          # GRUB settings
cat /etc/fstab                 # Mount points

# User/Group
cat /etc/passwd                # User accounts
cat /etc/group                 # Groups
sudo cat /etc/shadow           # Passwords (root only)
```

### Check/Edit Configuration
```bash
# Always backup before editing!
sudo cp /etc/fstab /etc/fstab.backup

# Edit safely
sudo vim /etc/fstab

# Test fstab before rebooting
sudo findmnt --verify          # Check syntax
sudo mount -a                  # Try mounting all
```

---

## 🚨 EMERGENCY & TROUBLESHOOTING

### System Not Responding
```bash
# Check load
uptime                         # Load averages

# Kill unresponsive process
ps aux | grep process_name     # Find PID
kill PID                       # Gentle kill
kill -9 PID                    # Force kill
```

### Disk Full
```bash
# Find what's using space
df -h                          # Which partition is full?
du -sh /* 2>/dev/null | sort -rh | head -10

# Clear space
sudo journalctl --vacuum-size=100M
sudo dnf clean all
```

### High Memory Usage
```bash
# Find memory hog
ps aux --sort=-%mem | head -10

# Clear swap (if needed and safe)
sudo swapoff -a && sudo swapon -a

# Drop caches (safe)
sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

### System Logs
```bash
# Recent errors
journalctl -p err -n 50        # Last 50 errors

# Follow live logs
journalctl -f                  # Like tail -f

# Specific service
journalctl -u sshd             # SSH logs
journalctl -u nginx            # Nginx logs
```

---

## 💡 QUICK TIPS & TRICKS

### Useful Aliases (Add to ~/.bashrc)
```bash
# Add these to your ~/.bashrc
alias ll='ls -lah'
alias df='df -hT'
alias free='free -h'
alias du='du -h'
alias psg='ps aux | grep -v grep | grep -i -e VSZ -e'
alias mem='ps aux --sort=-%mem | head -10'
alias cpu='ps aux --sort=-%cpu | head -10'

# Then reload:
source ~/.bashrc
```

### One-Line Health Checks
```bash
# Complete system health
echo "=== MEMORY ===" && free -h && \
echo "=== DISK ===" && df -hT && \
echo "=== LOAD ===" && uptime && \
echo "=== SWAP ===" && swapon --show

# Save to script
cat > ~/health_check.sh << 'EOF'
#!/bin/bash
echo "=== System Health Check ==="
echo "Date: $(date)"
echo ""
echo "=== MEMORY ==="
free -h
echo ""
echo "=== DISK SPACE ==="
df -hT
echo ""
echo "=== SWAP ==="
swapon --show
echo ""
echo "=== LOAD ==="
uptime
echo ""
echo "=== TOP MEMORY USERS ==="
ps aux --sort=-%mem | head -6
EOF

chmod +x ~/health_check.sh
```

---

## 📖 COMMAND CATEGORIES SUMMARY

### Daily Must-Run (Every Login)
```bash
free -h              # Memory status
df -h                # Disk space
lsblk                # Disk layout
uptime               # System load
```

### Weekly Checks
```bash
sudo dnf update      # System updates
df -i                # Inode usage
journalctl -p err    # Error logs
```

### Monthly Maintenance
```bash
sudo dnf remove --oldinstallonly kernel    # Clean old kernels
sudo journalctl --vacuum-time=30d          # Clean old logs
du -sh /var/log/*                          # Check log sizes
```

### When Things Go Wrong
```bash
top                  # What's using CPU/RAM
iostat -x 1          # Disk I/O issues
vmstat 2 5           # Memory/swap issues
journalctl -f        # Watch live logs
```

---

## 🎯 MEMORIZATION TIPS

### The "Big Five" (Learn These First!)
1. `free -h` - Memory status
2. `df -h` - Disk usage
3. `lsblk` - Disk layout
4. `top` - Process monitor
5. `journalctl -f` - Live logs

### Pattern Recognition
- `-h` = Human-readable (sizes in GB/MB)
- `-a` = All (show everything)
- `-f` = Follow (real-time updates)
- `-r` = Recursive (include subdirectories)
- `-v` = Verbose (show details)

### Command Structure
```
command [options] [target]
   ↑        ↑        ↑
 What?    How?    Where?

Examples:
ls -lah /home     # List all in /home, human-readable
df -hT            # Show disk, human-readable with types
free -h           # Show memory, human-readable
```

---

## 🔖 QUICK REFERENCE CARD

**Print this section and keep it handy!**

```
┌─────────────────────────────────────────────────────┐
│ DAILY LINUX COMMAND QUICK REFERENCE                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│ DISK:     df -hT        lsblk -f                    │
│ MEMORY:   free -h       vmstat 2                    │
│ SWAP:     swapon --show                             │
│ PROCESS:  top           ps aux                      │
│ PACKAGE:  rpm -qa       dnf update                  │
│ LOGS:     journalctl -f                             │
│ BOOT:     uname -r      ls /boot/vmlinuz-*          │
│ FIND:     find / -name file                         │
│ SPACE:    du -sh /*     sort -rh                    │
│ LINKS:    ls -la | grep "^l"                        │
│                                                     │
│ HELP:     man command   command --help              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

**Study Method:**
1. Run these commands daily until they become muscle memory
2. Add your most-used commands to this list
3. Create aliases for frequently-used combinations
4. Practice on your RHEL9 VM regularly

**Remember:** The best way to learn is by doing! 🚀

---

*Last Updated: Based on RHEL 9 learning sessions*  
*System: rhel9exam01 on VMware ESXi 6.7*
