# Pi4-TravelRouter Troubleshooting Guide

## Overview

This guide provides comprehensive solutions for common issues encountered with the Pi4-TravelRouter. Issues are organized by category with diagnostic steps and solutions.

## Quick Diagnostic Commands

Before diving into specific issues, run these commands to get a system overview:

```bash
# System status overview
/usr/local/bin/router-status.sh  # If created in service installation

# Manual system check
echo "=== Quick System Diagnostics ==="
echo "System Load: $(uptime)"
echo "Memory Usage: $(free -h)"
echo "Disk Usage: $(df -h /mnt/storage)"
echo "Temperature: $(vcgencmd measure_temp)"

# Network interfaces
echo -e "\nNetwork Interfaces:"
ip addr show wlan0 wlan1

# Service status
echo -e "\nCore Services:"
systemctl status hostapd --no-pager -l
systemctl status dnsmasq --no-pager -l

# Docker containers
echo -e "\nDocker Containers:"
sudo docker ps
```

## Common Issues by Category

### 1. Access Point Issues

#### 🔴 Access Point Not Broadcasting

**Symptoms:**
- Clients cannot see the `RaspberryPi-Travel` network
- USB Wi-Fi adapter not recognized
- hostapd service failures

**Diagnostic Steps:**
```bash
# Check hostapd service status
sudo systemctl status hostapd

# Check USB adapter recognition
lsusb | grep -i wireless
iwconfig

# Check if wlan1 interface exists
ip link show wlan1

# View hostapd logs
sudo journalctl -u hostapd -f --no-pager

# Test hostapd configuration
sudo hostapd -d /etc/hostapd/hostapd.conf
```

**Common Solutions:**

1. **USB Adapter Issues:**
   ```bash
   # Reconnect USB adapter
   sudo rmmod <driver_name>
   sudo modprobe <driver_name>
   
   # Or physically unplug and reconnect
   
   # Check if adapter supports AP mode
   iw list | grep -A 10 "Supported interface modes"
   ```

2. **hostapd Configuration Issues:**
   ```bash
   # Verify hostapd configuration
   sudo hostapd -t /etc/hostapd/hostapd.conf
   
   # Check for conflicting services
   sudo systemctl status NetworkManager
   sudo systemctl stop NetworkManager  # If running
   
   # Restart hostapd
   sudo systemctl restart hostapd
   ```

3. **Interface Issues:**
   ```bash
   # Bring up wlan1 interface
   sudo ip link set wlan1 up
   
   # Check for RF blocks
   sudo rfkill list all
   sudo rfkill unblock wifi
   ```

#### 🔴 Clients Connect but No Internet Access

**Symptoms:**
- Devices can connect to Wi-Fi but cannot access internet
- DNS resolution fails
- No traffic routing

**Diagnostic Steps:**
```bash
# Check internet connectivity on Pi
ping -c 3 8.8.8.8

# Check NAT rules
sudo iptables -t nat -L -v

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward  # Should be 1

# Check wlan0 connection status
iwconfig wlan0
ip route show

# Check default route
ip route show default

# Test traffic flow
sudo tcpdump -i wlan1 -n  # Monitor client traffic
```

**Solutions:**

1. **Enable IP Forwarding:**
   ```bash
   # Temporarily enable
   echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
   
   # Permanently enable in /etc/sysctl.conf
   echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
   ```

2. **Fix NAT Rules:**
   ```bash
   # Apply correct NAT rules
   sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
   sudo iptables -A FORWARD -i wlan1 -o wlan0 -j ACCEPT
   sudo iptables -A FORWARD -i wlan0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
   
   # Save rules
   sudo iptables-save > /etc/iptables/rules.v4
   ```

3. **Fix Upstream Connection:**
   ```bash
   # Restart network services
   sudo systemctl restart dhcpcd
   sudo systemctl restart wpa_supplicant
   
   # Reconnect to public Wi-Fi
   sudo wpa_cli -i wlan0 reconnect
   ```

### 2. DNS Resolution Issues

#### 🔴 Pi-hole Not Working

**Symptoms:**
- Ads not being blocked
- DNS queries timing out
- Pi-hole web interface not accessible

**Diagnostic Steps:**
```bash
# Check Pi-hole container status
sudo docker ps | grep pihole
sudo docker logs pihole

# Test DNS resolution
nslookup google.com 192.168.4.1
dig @192.168.4.1 google.com

# Check Pi-hole statistics
sudo docker exec pihole pihole -c

# Check dnsmasq status (if using system dnsmasq)
sudo systemctl status dnsmasq

# Test blocked domains
nslookup doubleclick.net 192.168.4.1  # Should return 0.0.0.0
```

**Solutions:**

1. **Restart Pi-hole Container:**
   ```bash
   sudo docker restart pihole
   
   # If container won't start, check logs
   sudo docker logs pihole --tail 50
   
   # Recreate if necessary
   cd /mnt/storage/docker/compose
   sudo docker compose -f docker-compose-pihole.yml down
   sudo docker compose -f docker-compose-pihole.yml up -d
   ```

2. **Fix DNS Configuration:**
   ```bash
   # Update system DNS to use Pi-hole
   sudo nano /etc/systemd/resolved.conf
   # Add: DNS=192.168.4.1
   
   sudo systemctl restart systemd-resolved
   
   # Or update dnsmasq configuration
   echo "server=192.168.4.1" | sudo tee /etc/dnsmasq.conf
   sudo systemctl restart dnsmasq
   ```

3. **Fix Pi-hole Permissions:**
   ```bash
   # Fix container permissions
   sudo chown -R 999:999 /mnt/storage/docker/pihole
   sudo chmod -R 755 /mnt/storage/docker/pihole
   ```

#### 🔴 Slow DNS Resolution

**Symptoms:**
- Web pages load slowly
- Long DNS lookup times
- Frequent DNS timeouts

**Solutions:**

1. **Increase Pi-hole Cache:**
   ```bash
   # Add to Pi-hole configuration
   echo "cache-size=10000" | sudo tee -a /mnt/storage/docker/pihole/dnsmasq.d/01-cache.conf
   sudo docker restart pihole
   ```

2. **Optimize Upstream DNS:**
   ```bash
   # Use faster DNS servers in Pi-hole
   # Web interface → Settings → DNS
   # Add: 1.1.1.1, 8.8.8.8, 1.0.0.1, 8.8.4.4
   
   # Or via command line
   sudo docker exec pihole pihole -a setdns 1.1.1.1 8.8.8.8
   ```

### 3. VPN Connection Issues

#### 🔴 Tailscale Connection Problems

**Symptoms:**
- Cannot connect to Tailscale network
- Route advertisement not working
- Authentication failures

**Diagnostic Steps:**
```bash
# Check Tailscale status
sudo tailscale status

# View Tailscale logs
sudo journalctl -u tailscaled -f

# Check network routes
ip route show | grep tailscale
```

**Solutions:**

1. **Restart and Re-authenticate:**
   ```bash
   sudo tailscale down
   sudo tailscale up --advertise-routes=192.168.4.0/24
   
   # If authentication fails
   sudo tailscale login
   ```

2. **Check Route Advertisement:**
   - Log into Tailscale admin console
   - Navigate to the Pi4-TravelRouter device
   - Enable route advertisement for 192.168.4.0/24
   - Approve the route on other devices

#### 🔴 ProtonVPN Connection Issues

**Symptoms:**
- Cannot connect to ProtonVPN
- VPN connection drops frequently
- No traffic routing through VPN

**Diagnostic Steps:**
```bash
# Check ProtonVPN status
sudo protonvpn-cli status

# Test connection
sudo protonvpn-cli connect --fastest

# Check routing when VPN is active
ip route show
```

**Solutions:**

1. **Reconnect ProtonVPN:**
   ```bash
   sudo protonvpn-cli disconnect
   sudo protonvpn-cli connect --fastest
   
   # Or try specific server
   sudo protonvpn-cli connect US-FREE#1
   ```

2. **Check VPN Routing:**
   ```bash
   # Ensure traffic routes through VPN
   sudo iptables -t nat -A POSTROUTING -o tun+ -j MASQUERADE
   ```

### 4. Performance Issues

#### 🔴 Slow Network Performance

**Symptoms:**
- Low throughput
- High latency
- Frequent disconnections

**Diagnostic Steps:**
```bash
# Check system resources
htop
iostat -x 1
vcgencmd measure_temp

# Network interface statistics
ip -s link show wlan0 wlan1
cat /proc/net/dev

# Check for packet loss
ping -c 10 8.8.8.8
```

**Solutions:**

1. **Optimize System Performance:**
   ```bash
   # Check CPU frequency
   cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
   
   # Ensure adequate cooling
   vcgencmd measure_temp  # Should be < 70°C
   
   # Check memory usage
   free -h
   ```

2. **Optimize Network Settings:**
   ```bash
   # Disable power saving on Wi-Fi
   sudo iw dev wlan1 set power_save off
   
   # Optimize USB power management
   echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="[your_vendor_id]", ATTR{power/control}="on"' | sudo tee /etc/udev/rules.d/50-usb-power.rules
   ```

#### 🔴 High Memory Usage

**Symptoms:**
- System becomes unresponsive
- Out of memory errors
- Swap usage is high

**Solutions:**

1. **Optimize Docker Containers:**
   ```bash
   # Check container memory usage
   sudo docker stats
   
   # Restart memory-heavy containers
   sudo docker restart pihole
   
   # Limit container memory (in compose file)
   # mem_limit: 512m
   ```

2. **Increase Swap:**
   ```bash
   # Check current swap
   swapon -s
   
   # Increase swap file size
   sudo swapoff /mnt/storage/swapfile
   sudo dd if=/dev/zero of=/mnt/storage/swapfile bs=1M count=2048
   sudo mkswap /mnt/storage/swapfile
   sudo swapon /mnt/storage/swapfile
   ```

### 5. Storage Issues

#### 🔴 USB Drive Not Mounting

**Symptoms:**
- Docker containers fail to start
- Configuration not persistent
- Storage errors

**Diagnostic Steps:**
```bash
# Check USB drive detection
lsblk
fdisk -l

# Check mount points
mount | grep storage
df -h

# Check file system
sudo fsck /dev/sda1
```

**Solutions:**

1. **Remount USB Drive:**
   ```bash
   # Unmount and remount
   sudo umount /mnt/storage
   sudo mount /dev/sda1 /mnt/storage
   
   # Check /etc/fstab entry
   grep storage /etc/fstab
   ```

2. **Fix File System:**
   ```bash
   # Check and repair filesystem
   sudo umount /mnt/storage
   sudo fsck -f /dev/sda1
   sudo mount /dev/sda1 /mnt/storage
   ```

### 6. Container Issues

#### 🔴 Docker Containers Won't Start

**Symptoms:**
- Container exits immediately
- Port binding errors
- Volume mount failures

**Solutions:**

1. **Check Docker Service:**
   ```bash
   sudo systemctl status docker
   sudo systemctl restart docker
   
   # Check Docker daemon logs
   sudo journalctl -u docker -f
   ```

2. **Fix Port Conflicts:**
   ```bash
   # Check what's using ports
   sudo netstat -tulpn | grep -E ":(53|80|9000|51821)"
   
   # Stop conflicting services
   sudo systemctl stop systemd-resolved  # If using port 53
   ```

3. **Fix Volume Permissions:**
   ```bash
   # Fix common permission issues
   sudo chown -R 999:999 /mnt/storage/docker/pihole
   sudo chown -R $(id -u):$(id -g) /mnt/storage/docker/portainer
   ```

## Recovery Procedures

### Complete System Recovery

If your Pi4-TravelRouter becomes completely unresponsive:

1. **Emergency Access via Ethernet:**
   ```bash
   # Connect via Ethernet
   ssh pi@<ethernet_ip>
   
   # Check system status
   systemctl status
   ```

2. **Service Recovery:**
   ```bash
   # Stop all services
   sudo systemctl stop hostapd dnsmasq docker
   
   # Start services one by one
   sudo systemctl start docker
   sudo systemctl start dnsmasq
   sudo systemctl start hostapd
   ```

3. **Factory Reset:**
   ```bash
   # Backup current config
   tar -czf /home/pi/emergency-backup-$(date +%F).tar.gz /etc/hostapd /etc/dnsmasq.conf /mnt/storage/docker
   
   # Reset to known good state
   # Restore from backup or reinstall
   ```

### Backup and Restore

**Create Emergency Backup:**
```bash
#!/bin/bash
BACKUP_DIR="/mnt/storage/backups/emergency-$(date +%F-%H%M)"
mkdir -p "$BACKUP_DIR"

# System configurations
cp -r /etc/hostapd "$BACKUP_DIR/"
cp -r /etc/systemd/network "$BACKUP_DIR/"
cp /etc/dnsmasq.conf "$BACKUP_DIR/"

# Docker configurations
cp -r /mnt/storage/docker "$BACKUP_DIR/"

echo "Emergency backup created: $BACKUP_DIR"
```

**Restore from Backup:**
```bash
#!/bin/bash
BACKUP_DIR="$1"

if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <backup-directory>"
    exit 1
fi

# Stop services
sudo systemctl stop hostapd dnsmasq docker

# Restore configurations
sudo cp -r "$BACKUP_DIR/hostapd" /etc/
sudo cp -r "$BACKUP_DIR/systemd" /etc/
sudo cp "$BACKUP_DIR/dnsmasq.conf" /etc/
sudo cp -r "$BACKUP_DIR/docker" /mnt/storage/

# Restart services
sudo systemctl start docker dnsmasq hostapd

echo "Restore completed from: $BACKUP_DIR"
```

## Advanced Troubleshooting

### Network Traffic Analysis

```bash
# Monitor network traffic
sudo tcpdump -i wlan1 -n host 192.168.4.100  # Monitor specific client
sudo iftop -i wlan1  # Real-time bandwidth usage

# Analyze Pi-hole queries
tail -f /mnt/storage/docker/pihole/pihole.log | grep BLOCKED
```

### Performance Monitoring

```bash
# System performance monitoring
while true; do
    echo "$(date): Load: $(uptime | cut -d',' -f3-), Temp: $(vcgencmd measure_temp), Memory: $(free -m | grep Mem | awk '{print $3"/"$2}')"
    sleep 60
done
```

### Log Analysis

```bash
# Important logs to check
sudo journalctl -u hostapd --since "1 hour ago"
sudo journalctl -u dnsmasq --since "1 hour ago"
sudo docker logs pihole --since 1h
sudo dmesg | grep -i usb  # USB device issues
```

## Getting Help

### Information to Collect

When seeking help, collect this information:

```bash
# System information
uname -a
cat /etc/os-release
vcgencmd version

# Hardware information
lscpu
free -h
lsblk
lsusb

# Service status
systemctl status hostapd dnsmasq docker --no-pager
sudo docker ps -a

# Network configuration
ip addr show
ip route show
sudo iptables -t nat -L -v
```

### Support Resources

- **GitHub Issues**: [Pi4-TravelRouter Issues](https://github.com/cypheroxide/Pi4-TravelRouter/issues)
- **Community Discussions**: [GitHub Discussions](https://github.com/cypheroxide/Pi4-TravelRouter/discussions)
- **Configuration Reference**: [Configuration Guides](../configuration/)
- **Security Hardening**: [Security Guide](../security/)

---

## Prevention Tips

1. **Regular Monitoring**: Check system health weekly
2. **Keep Backups**: Automated daily backups to USB storage
3. **Update Software**: Monthly system and container updates  
4. **Monitor Resources**: Watch CPU temperature, memory, and storage
5. **Test Changes**: Always test configuration changes in non-critical times
6. **Document Changes**: Keep notes of any custom modifications

Remember: Most issues can be resolved by methodically checking each component in the network chain: USB adapter → hostapd → routing → DNS → internet connectivity.