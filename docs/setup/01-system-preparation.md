# Phase 1: System Preparation & Initial Setup

## Overview

This phase covers the initial Raspberry Pi OS installation, basic system configuration, and USB storage setup. This is the foundation that all other components build upon.

**Estimated Time**: 30-45 minutes  
**Difficulty**: Beginner  
**Prerequisites**: [Hardware Requirements](../hardware/requirements.md) fulfilled

## Pre-Installation Checklist

Before starting, ensure you have:

- [ ] Raspberry Pi 4 (4GB+ recommended)
- [ ] MicroSD card (32GB+ Class 10)
- [ ] USB storage drive (64GB+ USB 3.0)
- [ ] Ethernet cable for initial connection
- [ ] Computer for flashing SD card
- [ ] Stable internet connection

## Step 1: Flash Raspberry Pi OS

### Download Raspberry Pi Imager

1. Download [Raspberry Pi Imager](https://www.raspberrypi.org/software/) for your operating system
2. Install and launch the application

### Configure and Flash

1. **Select OS**: Choose "Raspberry Pi OS Lite (64-bit)"
   - Why Lite? Smaller footprint, no GUI overhead
   - Why 64-bit? Better performance on Pi 4

2. **Configure Advanced Options** (⚙️ gear icon):
   ```
   ✅ Enable SSH
   ✅ Set username: pi (or your preferred username)
   ✅ Set password: [Choose strong password]
   ✅ Configure WiFi (optional for backup)
   ✅ Set locale settings (timezone, keyboard)
   ```

3. **Select Storage**: Choose your microSD card

4. **Write**: Click "Write" and wait for completion (~10 minutes)

## Step 2: Initial Boot & Connection

### First Boot

1. Insert the microSD card into your Pi 4
2. Connect Ethernet cable (for stable initial setup)
3. Connect power supply
4. Wait 2-3 minutes for boot completion

### Connect via SSH

```bash
# Find Pi's IP address (check your router or use network scanner)
nmap -sn 192.168.1.0/24

# Connect via SSH
ssh pi@192.168.1.XXX  # Replace with actual IP
```

**If connection fails**:
- Check Ethernet cable connection
- Verify SSH was enabled in imager
- Try connecting monitor/keyboard directly

## Step 3: System Updates & Essential Packages

### Update System

```bash
# Update package lists and system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    htop \
    iptables-persistent \
    net-tools \
    rsync \
    unzip
```

**Note**: This may take 10-15 minutes depending on internet speed.

### Configure System Settings

```bash
# Set timezone (replace with your timezone)
sudo timedatectl set-timezone America/Chicago

# Enable IP forwarding (required for router functionality)
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Verify IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward  # Should output: 1
```

### Configure Basic Security

```bash
# Change default hostname (optional but recommended)
sudo hostnamectl set-hostname pi4-router

# Update /etc/hosts with new hostname
sudo sed -i 's/raspberrypi/pi4-router/g' /etc/hosts

# Disable automatic login (if enabled)
sudo systemctl set-default multi-user.target
```

## Step 4: USB Storage Configuration

### Identify Storage Device

```bash
# List all block devices
lsblk

# You should see something like:
# sda           8:0    1  119.2G  0 disk 
# └─sda1        8:1    1  119.2G  0 part 
```

**Note the device name** (usually `/dev/sda` for first USB device)

### Partition and Format USB Drive

⚠️ **Warning**: This will erase all data on the USB drive!

```bash
# Unmount if currently mounted
sudo umount /dev/sda* 2>/dev/null || true

# Create new partition table and partition
sudo fdisk /dev/sda << 'EOF'
o
n
p
1


w
EOF

# Format the partition as ext4
sudo mkfs.ext4 -F /dev/sda1

# Set filesystem label
sudo e2label /dev/sda1 "pi4-storage"
```

### Create Mount Point and Mount

```bash
# Create mount point
sudo mkdir -p /mnt/storage

# Get UUID of the partition for persistent mounting
UUID=$(sudo blkid -s UUID -o value /dev/sda1)
echo "USB Drive UUID: $UUID"

# Add to fstab for persistent mounting
echo "UUID=$UUID /mnt/storage ext4 defaults,nofail,noatime 0 2" | sudo tee -a /etc/fstab

# Mount the drive
sudo mount /mnt/storage

# Verify mount was successful
df -h /mnt/storage
```

### Set Up Directory Structure

```bash
# Create standard directories
sudo mkdir -p /mnt/storage/{docker,data,backups,configs}

# Create Docker subdirectories
sudo mkdir -p /mnt/storage/docker/{containers,volumes,compose}

# Set ownership to current user for data directories
sudo chown -R $USER:$USER /mnt/storage/data
sudo chown -R $USER:$USER /mnt/storage/configs
sudo chown -R $USER:$USER /mnt/storage/backups

# Docker directories should remain root-owned for now
```

## Step 5: System Optimization

### Configure Swap File on USB Storage

The default Pi OS swap is too small and on SD card (bad for performance/longevity).

```bash
# Disable existing swap
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile

# Create 8GB swap file on USB storage
sudo fallocate -l 8G /mnt/storage/swapfile
sudo chmod 600 /mnt/storage/swapfile
sudo mkswap /mnt/storage/swapfile
sudo swapon /mnt/storage/swapfile

# Add to fstab for persistence
echo "/mnt/storage/swapfile none swap sw,pri=10 0 0" | sudo tee -a /etc/fstab

# Verify swap is active
swapon --show
free -h
```

### Apply Memory Optimizations

```bash
# Optimize memory management
echo "vm.swappiness=1" | sudo tee -a /etc/sysctl.conf
echo "vm.min_free_kbytes=8192" | sudo tee -a /etc/sysctl.conf
echo "vm.vfs_cache_pressure=50" | sudo tee -a /etc/sysctl.conf

# Apply immediately
sudo sysctl -p
```

### Configure Boot Optimizations

```bash
# Edit boot config for optimal performance
echo "
# Pi4-TravelRouter optimizations
gpu_mem=16
dtoverlay=disable-wifi
dtoverlay=disable-bt" | sudo tee -a /boot/config.txt
```

**Note**: `disable-wifi` disables built-in Wi-Fi to avoid conflicts. We'll use USB adapters.

## Step 6: Create Backup Script

Create a basic backup script for system configuration:

```bash
# Create backup script
sudo tee /usr/local/bin/backup-system.sh > /dev/null << 'EOF'
#!/bin/bash

TIMESTAMP=$(date +%F_%H%M)
BACKUP_DIR="/mnt/storage/backups/system_$TIMESTAMP"

echo "Creating system backup: $BACKUP_DIR"

mkdir -p "$BACKUP_DIR"

# Backup important system files
cp -r /etc/fstab "$BACKUP_DIR/"
cp -r /etc/sysctl.conf "$BACKUP_DIR/"
cp -r /etc/hostname "$BACKUP_DIR/"
cp -r /etc/hosts "$BACKUP_DIR/"
cp -r /boot/config.txt "$BACKUP_DIR/"

# Create system info
cat > "$BACKUP_DIR/system_info.txt" << 'INFO'
Hostname: $(hostname)
Kernel: $(uname -r)
OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)
Uptime: $(uptime)
Disk Usage: $(df -h /mnt/storage)
Memory: $(free -h)
Swap: $(swapon --show)
INFO

tar -czf "/mnt/storage/backups/system_backup_$TIMESTAMP.tar.gz" -C "$BACKUP_DIR" .
rm -rf "$BACKUP_DIR"

echo "Backup completed: system_backup_$TIMESTAMP.tar.gz"

# Cleanup old backups (keep last 7 days)
find /mnt/storage/backups -name "system_backup_*.tar.gz" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/backup-system.sh

# Create initial backup
/usr/local/bin/backup-system.sh
```

## Step 7: Validation & Testing

### System Health Check

```bash
# Check system status
echo "=== System Health Check ==="
echo "Hostname: $(hostname)"
echo "Uptime: $(uptime)"
echo "Memory: $(free -h | grep Mem)"
echo "Swap: $(free -h | grep Swap)"
echo "Storage: $(df -h /mnt/storage | tail -1)"
echo "Temperature: $(vcgencmd measure_temp)"
echo "IP Forwarding: $(cat /proc/sys/net/ipv4/ip_forward)"
```

### Validate USB Storage

```bash
# Test write/read performance
echo "Testing USB storage performance..."
dd if=/dev/zero of=/mnt/storage/testfile bs=1M count=100 2>&1 | grep -E 'copied|MB/s'
rm /mnt/storage/testfile
echo "Storage test completed"
```

### Network Connectivity Test

```bash
# Test internet connectivity
ping -c 3 8.8.8.8 && echo "✅ Internet connectivity: OK" || echo "❌ Internet connectivity: FAILED"
ping -c 3 google.com && echo "✅ DNS resolution: OK" || echo "❌ DNS resolution: FAILED"
```

### Reboot Test

```bash
echo "System preparation complete. Testing reboot..."
sudo reboot
```

**After reboot, reconnect via SSH and verify:**

```bash
# Quick validation after reboot
mount | grep /mnt/storage  # Should show USB drive mounted
swapon --show             # Should show swap on USB
free -h                   # Should show 8GB swap
df -h /mnt/storage        # Should show USB storage
```

## Troubleshooting Common Issues

### USB Drive Not Mounting
```bash
# Check if drive is detected
lsblk

# Check filesystem
sudo fsck -f /dev/sda1

# Check fstab syntax
sudo mount -a
```

### Swap Not Working
```bash
# Check swap status
sudo swapon --show
sudo swapon -a

# Check fstab entry
grep swap /etc/fstab
```

### SSH Connection Issues
```bash
# Check SSH service
sudo systemctl status ssh

# Check network connectivity
ip addr show eth0
ping -c 1 192.168.1.1  # Your router IP
```

## What's Next?

After completing this phase:

✅ **Completed Tasks:**
- Raspberry Pi OS installed and updated
- SSH access configured
- USB storage mounted and configured
- System optimized with proper swap
- Basic backup system in place

➡️ **Next Phase:** [Docker Installation](02-docker-setup.md)

Before proceeding, ensure all validation tests pass. If you encounter issues, check the [Troubleshooting Guide](../troubleshooting/system-issues.md).