# Pi4-TravelRouter Security Guide

## Overview

This guide provides comprehensive security hardening recommendations for the Pi4-TravelRouter. Security is particularly important for travel routers as they often operate on untrusted networks and handle sensitive traffic.

**Security Principles:**
- **Defense in Depth**: Multiple layers of protection
- **Least Privilege**: Minimal access rights and services
- **Regular Updates**: Keep all software current
- **Monitoring**: Track and audit system activity
- **Physical Security**: Protect physical access

## Pre-Deployment Security Checklist

### ✅ Essential Security Steps (Before First Use)

**1. Change All Default Passwords**
```bash
# System user password
passwd

# Pi-hole admin password
sudo docker exec pihole pihole -a -p "YourVerySecurePasswordHere123!"

# WiFi access point password (edit /etc/hostapd/hostapd.conf)
sudo nano /etc/hostapd/hostapd.conf
# Change: wpa_passphrase=YourVeryStrongWiFiPassword123!

# SSH key-based authentication (recommended)
ssh-copy-id pi@192.168.4.1
```

**2. Update All Software**
```bash
# System packages
sudo apt update && sudo apt full-upgrade -y

# Docker containers
cd /mnt/storage/docker/compose
sudo docker compose pull
sudo docker compose up -d

# Tailscale (if installed)
sudo tailscale update

# ProtonVPN (if installed)
sudo protonvpn-cli update
```

**3. Secure SSH Configuration**
```bash
# Edit SSH configuration
sudo nano /etc/ssh/sshd_config

# Recommended settings:
Port 2222                           # Change from default 22
PasswordAuthentication no           # Use keys only
PermitRootLogin no                 # Disable root login
Protocol 2                         # Use SSH protocol 2
MaxAuthTries 3                     # Limit login attempts
ClientAliveInterval 300            # Timeout idle sessions
ClientAliveCountMax 2
AllowUsers pi                      # Restrict users
```

**4. Configure Firewall**
```bash
# Install and configure UFW
sudo apt install -y ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Essential services
sudo ufw allow 2222/tcp comment 'SSH (custom port)'
sudo ufw allow 53/udp comment 'DNS'
sudo ufw allow 67/udp comment 'DHCP'

# Web interfaces (local network only)
sudo ufw allow from 192.168.4.0/24 to any port 80 comment 'Pi-hole admin'
sudo ufw allow from 192.168.4.0/24 to any port 9000 comment 'Portainer'
sudo ufw allow from 192.168.4.0/24 to any port 51821 comment 'WG-Easy'

# VPN services
sudo ufw allow 51820/udp comment 'WireGuard'

# Enable firewall
sudo ufw enable
```

## Network Security

### WiFi Access Point Security

**1. Strong WPA Configuration**

Edit `/etc/hostapd/hostapd.conf`:
```bash
# Use WPA3 if supported by all devices
wpa=3
wpa_key_mgmt=SAE
sae_password=YourVeryStrongPassphrase123!@#
sae_require_mfp=1
ieee80211w=2

# Or mixed WPA2/WPA3 for compatibility
wpa=2
wpa_key_mgmt=WPA-PSK SAE
wpa_passphrase=YourVeryStrongPassphrase123!@#
sae_password=YourVeryStrongPassphrase123!@#
ieee80211w=1

# Security enhancements
wps_state=0                    # Disable WPS
ignore_broadcast_ssid=0        # Keep at 0 (hiding SSID is weak security)
macaddr_acl=0                 # Change to 1 for MAC filtering (if needed)

# Limit concurrent clients
max_num_sta=10

# Strong encryption
wpa_pairwise=CCMP             # AES encryption
rsn_pairwise=CCMP
```

**2. MAC Address Filtering (Optional High-Security)**

For environments requiring strict access control:
```bash
# Enable MAC filtering in hostapd.conf
macaddr_acl=1
accept_mac_file=/etc/hostapd/hostapd.accept

# Create MAC address whitelist
sudo nano /etc/hostapd/hostapd.accept
# Add allowed MAC addresses (one per line):
aa:bb:cc:dd:ee:01  # Trusted Device 1
aa:bb:cc:dd:ee:02  # Trusted Device 2
```

**3. Client Isolation (Optional)**

Prevent clients from communicating with each other:
```bash
# Add to hostapd.conf
ap_isolate=1
```

### Network Segmentation

**1. Isolated Management Network**

Create a separate management network for admin access:
```bash
# Create additional interface configuration
cat > /etc/systemd/network/35-management.network << 'EOF'
[Match]
Name=wlan1

[Network]
Address=192.168.5.1/24
DHCPServer=yes
IPForward=yes

[DHCPServer]
PoolOffset=10
PoolSize=10
EmitDNS=yes
DNS=192.168.5.1
EmitRouter=yes
EOF
```

**2. VLAN Segmentation (Advanced)**

For advanced users with VLAN-capable hardware:
```bash
# Create VLAN interfaces
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 192.168.100.1/24 dev eth0.100

# Configure separate routing tables
echo "200 management" >> /etc/iproute2/rt_tables
ip route add 192.168.100.0/24 dev eth0.100 table management
```

## DNS Security

### Pi-hole Security Configuration

**1. Secure Pi-hole Settings**

Access Pi-hole admin interface and configure:

**Privacy Settings:**
- Privacy level: Show everything and permit all origins (for troubleshooting)
- Or: Anonymous mode (for privacy)

**DNS Settings:**
- Use secure upstream DNS:
  - Cloudflare DNS over HTTPS: 1.1.1.1#cloudflare-dns.com
  - Quad9 Secure: 9.9.9.9#dns.quad9.net
- Enable DNSSEC validation

**Query Logging:**
- Enable for security monitoring
- Set appropriate retention period (7 days recommended)

**2. DNS Filtering Enhancement**

Add security-focused blocklists:
```bash
# Access Pi-hole admin → Group Management → Adlists
# Add these security lists:

# Malware and phishing
https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20of%20existing%20lists/AntiMalwareList.txt
https://phishing.army/download/phishing_army_blocklist_extended.txt
https://gitlab.com/quidsup/notrack-blocklists/raw/master/notrack-malware.txt

# Cryptomining protection
https://raw.githubusercontent.com/hoshsadiq/adblock-nocoin-list/master/hosts.txt

# Command and Control (C&C) servers
https://rules.emergingthreats.net/fwrules/emerging-Block-IPs.txt
```

**3. DNS Monitoring Script**

Create monitoring for suspicious DNS activity:
```bash
#!/bin/bash
# File: /usr/local/bin/dns-monitor.sh

ALERT_EMAIL="admin@example.com"  # Configure your email
LOG_FILE="/var/log/dns-security.log"
THRESHOLD=100  # Queries per minute threshold

# Monitor for DNS tunneling attempts
tail -f /mnt/storage/docker/pihole/pihole.log | while read line; do
    # Look for suspicious patterns
    if echo "$line" | grep -E "(\.tk|\.ml|\.ga|\.cf)" | grep -q "query"; then
        echo "$(date): Suspicious TLD detected: $line" >> "$LOG_FILE"
    fi
    
    # Monitor for excessive queries from single client
    client_ip=$(echo "$line" | awk '{print $5}')
    query_count=$(grep -c "$client_ip" /tmp/recent_queries_$(date +%M) 2>/dev/null || echo 0)
    
    if [ "$query_count" -gt "$THRESHOLD" ]; then
        echo "$(date): Excessive queries from $client_ip ($query_count/min)" >> "$LOG_FILE"
    fi
done
```

## VPN Security

### Tailscale Security

**1. Tailscale Hardening**
```bash
# Enable MFA on your Tailscale account (via web console)

# Use ACLs to restrict access
# In Tailscale admin console, configure ACLs:
{
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:admin"],
      "dst": ["tag:pi4router:*"]
    },
    {
      "action": "accept", 
      "src": ["tag:trusted"],
      "dst": ["tag:pi4router:22,80,9000"]
    }
  ],
  "tags": {
    "tag:pi4router": ["admin@example.com"],
    "tag:trusted": ["admin@example.com"]
  }
}

# Enable key expiry
sudo tailscale up --advertise-routes=192.168.4.0/24 --ssh --key-expiry=720h

# Disable unused services
sudo tailscale set --shields-up=true
```

**2. ProtonVPN Security**

Secure ProtonVPN configuration:
```bash
# Use secure configuration
protonvpn-cli configure
# Choose:
# - OpenVPN protocol: UDP (faster) or TCP (more reliable through firewalls)
# - Kill switch: Enable
# - DNS management: Enable
# - Split tunneling: Disable for maximum privacy

# Verify no DNS leaks
curl -s https://1.1.1.1/cdn-cgi/trace | grep ip=
# Should show ProtonVPN server IP, not your real IP

# Test for WebRTC leaks
curl -s https://browserleaks.com/webrtc
```

### WireGuard Security

**1. Key Management**
```bash
# Generate strong keys
wg genkey | tee private.key | wg pubkey > public.key

# Secure key storage
chmod 600 private.key
chown root:root private.key

# Regular key rotation (monthly recommended)
# Update keys in WG-Easy interface or configuration files
```

**2. WireGuard Firewall Rules**
```bash
# Restrict WireGuard access
sudo ufw allow from any to any port 51820 proto udp

# Limit concurrent connections in WG-Easy
# Set reasonable client limits in docker-compose configuration
```

## Container Security

### Docker Security Hardening

**1. Docker Daemon Security**

Update `/etc/docker/daemon.json`:
```json
{
  "data-root": "/mnt/storage/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "seccomp-profile": "/etc/docker/seccomp.json",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  }
}
```

**2. Container Security Best Practices**

Update Docker Compose configurations with security settings:
```yaml
version: '3.8'

services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    cap_drop:
      - ALL
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    sysctls:
      - net.ipv4.ping_group_range=0 2147483647
    ulimits:
      nproc: 65535
      nofile:
        soft: 262144
        hard: 262144
```

**3. Container Vulnerability Scanning**
```bash
# Install and run security scanners
sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image pihole/pihole:latest

# Scan for container misconfigurations
sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc/docker:/etc/docker:ro \
  aquasec/docker-bench-security
```

## System Security Hardening

### File System Security

**1. Secure Mount Options**

Update `/etc/fstab`:
```bash
# Secure USB storage mount
/dev/sda1 /mnt/storage ext4 defaults,nodev,nosuid,noexec 0 2

# Secure temp directories
tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0
tmpfs /var/tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0
```

**2. File Permissions**
```bash
# Secure system files
chmod 600 /etc/ssh/ssh_host_*_key
chmod 644 /etc/ssh/ssh_host_*_key.pub
chmod 700 /root
chmod 755 /home/pi

# Secure Docker files
chown -R root:root /mnt/storage/docker
chmod -R 755 /mnt/storage/docker
chmod -R 600 /mnt/storage/docker/*/config

# Secure configuration files
chmod 600 /etc/hostapd/hostapd.conf
chmod 644 /etc/dnsmasq.conf
chown root:root /etc/hostapd/hostapd.conf
```

### Kernel Security

**1. Kernel Parameters**

Add to `/etc/sysctl.d/99-security.conf`:
```bash
# Network security
net.ipv4.ip_forward=1                    # Required for routing
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0
net.ipv4.conf.all.log_martians=1
net.ipv4.conf.default.log_martians=1
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
net.ipv4.tcp_syncookies=1

# IPv6 security (if not using IPv6)
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1

# Kernel security
kernel.dmesg_restrict=1
kernel.kptr_restrict=2
kernel.yama.ptrace_scope=1
```

**2. Disable Unused Services**
```bash
# List and disable unnecessary services
systemctl list-unit-files --state=enabled

# Disable unused services (examples)
sudo systemctl disable bluetooth
sudo systemctl disable avahi-daemon
sudo systemctl disable cups
sudo systemctl disable ModemManager

# Keep only essential services enabled:
# - ssh, docker, hostapd, dnsmasq, tailscaled (if using)
```

## Monitoring and Logging

### Security Monitoring

**1. Log Monitoring Script**
```bash
#!/bin/bash
# File: /usr/local/bin/security-monitor.sh

LOG_FILE="/var/log/security-monitor.log"
ALERT_THRESHOLD=5

# Monitor failed SSH attempts
FAILED_SSH=$(grep "Failed password" /var/log/auth.log | grep "$(date '+%b %d')" | wc -l)
if [ "$FAILED_SSH" -gt "$ALERT_THRESHOLD" ]; then
    echo "$(date): WARNING: $FAILED_SSH failed SSH attempts today" >> "$LOG_FILE"
fi

# Monitor suspicious network activity
DROPPED_PACKETS=$(dmesg | grep -c "DPT=")
if [ "$DROPPED_PACKETS" -gt 100 ]; then
    echo "$(date): INFO: $DROPPED_PACKETS packets dropped by firewall" >> "$LOG_FILE"
fi

# Check for USB device changes
USB_DEVICES=$(lsusb | wc -l)
echo "$(date): USB devices connected: $USB_DEVICES" >> "$LOG_FILE"

# Monitor system resources
CPU_TEMP=$(vcgencmd measure_temp | cut -d'=' -f2 | cut -d"'" -f1)
if [ "${CPU_TEMP%.*}" -gt 75 ]; then
    echo "$(date): WARNING: High CPU temperature: $CPU_TEMP°C" >> "$LOG_FILE"
fi

# Check Docker container status
CONTAINERS_RUNNING=$(docker ps -q | wc -l)
if [ "$CONTAINERS_RUNNING" -lt 2 ]; then
    echo "$(date): WARNING: Only $CONTAINERS_RUNNING containers running" >> "$LOG_FILE"
fi
```

**2. Automated Security Updates**
```bash
# Install unattended-upgrades
sudo apt install -y unattended-upgrades

# Configure automatic security updates
echo 'APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades

# Configure Docker container updates (weekly)
cat > /usr/local/bin/update-containers.sh << 'EOF'
#!/bin/bash
cd /mnt/storage/docker/compose
docker compose pull
docker compose up -d --remove-orphans
docker image prune -f
EOF

chmod +x /usr/local/bin/update-containers.sh

# Add to crontab (run weekly)
echo "0 3 * * 0 /usr/local/bin/update-containers.sh >> /var/log/container-updates.log 2>&1" | sudo crontab -
```

### Security Auditing

**1. Regular Security Checks**
```bash
#!/bin/bash
# File: /usr/local/bin/security-audit.sh

echo "=== Pi4-TravelRouter Security Audit ==="
echo "Date: $(date)"

# Check for weak passwords
echo "Checking for default passwords..."
if grep -q "CHANGE_THIS_PASSWORD" /etc/hostapd/hostapd.conf; then
    echo "❌ Default WiFi password detected!"
fi

# Check SSH configuration
echo "Checking SSH security..."
if grep -q "^PasswordAuthentication yes" /etc/ssh/sshd_config; then
    echo "⚠️ SSH password authentication enabled"
fi

# Check firewall status
echo "Checking firewall status..."
ufw status | grep "Status: active" > /dev/null && echo "✅ UFW firewall active" || echo "❌ UFW firewall inactive"

# Check for security updates
echo "Checking for security updates..."
UPDATES=$(apt list --upgradable 2>/dev/null | grep -c security)
if [ "$UPDATES" -gt 0 ]; then
    echo "⚠️ $UPDATES security updates available"
else
    echo "✅ System up to date"
fi

# Check container security
echo "Checking container security..."
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check VPN status
echo "Checking VPN security..."
tailscale status > /dev/null 2>&1 && echo "✅ Tailscale active" || echo "ℹ️ Tailscale not active"
protonvpn-cli status | grep -q "Connected" && echo "✅ ProtonVPN active" || echo "ℹ️ ProtonVPN not active"

echo "=== Audit Complete ==="
```

## Incident Response

### Security Incident Procedures

**1. Suspected Compromise**
```bash
# Immediate actions
# 1. Disconnect from internet
sudo ip link set wlan0 down

# 2. Check for suspicious processes
ps aux | grep -v "\[.*\]"
netstat -tulpn | grep LISTEN

# 3. Check system logs
journalctl --since "24 hours ago" | grep -i "error\|fail\|attack"
tail -100 /var/log/auth.log

# 4. Check network connections
ss -tuln
lsof -i

# 5. Backup current state for analysis
tar -czf /mnt/storage/incident-backup-$(date +%F-%H%M).tar.gz \
    /var/log /etc /mnt/storage/docker
```

**2. Recovery Procedures**
```bash
# After analysis, recovery steps:

# 1. Change all passwords
passwd
# Update all service passwords

# 2. Regenerate SSH keys
sudo rm /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart ssh

# 3. Update all software
sudo apt update && sudo apt full-upgrade -y
docker compose pull && docker compose up -d

# 4. Review and update firewall rules
sudo ufw reset
# Reconfigure firewall rules

# 5. Restore from clean backup if necessary
```

### Forensic Data Collection

**1. Evidence Collection Script**
```bash
#!/bin/bash
# File: /usr/local/bin/collect-evidence.sh

EVIDENCE_DIR="/mnt/storage/evidence-$(date +%F-%H%M)"
mkdir -p "$EVIDENCE_DIR"

# System information
uname -a > "$EVIDENCE_DIR/system-info.txt"
date >> "$EVIDENCE_DIR/system-info.txt"
uptime >> "$EVIDENCE_DIR/system-info.txt"

# Process information
ps auxwww > "$EVIDENCE_DIR/processes.txt"
netstat -tulpna > "$EVIDENCE_DIR/network-connections.txt"
lsof > "$EVIDENCE_DIR/open-files.txt"

# Log files
cp /var/log/syslog "$EVIDENCE_DIR/"
cp /var/log/auth.log "$EVIDENCE_DIR/"
docker logs pihole > "$EVIDENCE_DIR/pihole.log" 2>&1

# Network configuration
ip addr show > "$EVIDENCE_DIR/network-config.txt"
ip route show >> "$EVIDENCE_DIR/network-config.txt"
iptables -L -n -v > "$EVIDENCE_DIR/firewall-rules.txt"

# Hash all evidence
find "$EVIDENCE_DIR" -type f -exec sha256sum {} \; > "$EVIDENCE_DIR/hashes.sha256"

echo "Evidence collected in: $EVIDENCE_DIR"
```

## Regular Maintenance

### Security Maintenance Schedule

**Daily:**
- Monitor system logs for suspicious activity
- Check system temperature and resource usage
- Verify all services are running correctly

**Weekly:**
- Review Pi-hole query logs for unusual patterns
- Update Docker containers
- Check firewall logs
- Run security audit script

**Monthly:**
- Update system packages
- Review and update SSH keys
- Change service passwords
- Review network configurations
- Test backup and restore procedures

**Quarterly:**
- Full security audit
- Penetration testing (if skilled)
- Review and update security configurations
- Update emergency response procedures

### Security Checklist Template

```bash
# Monthly Security Review Checklist

□ All default passwords changed
□ SSH key authentication enabled
□ Firewall configured and active
□ All software updated to latest versions
□ Container images updated
□ VPN services functioning correctly
□ DNS filtering working effectively
□ System logs reviewed for anomalies
□ Backup system tested and working
□ Physical security measures in place
□ Network segmentation properly configured
□ Security monitoring scripts running
□ Incident response procedures tested
□ Documentation up to date
```

---

## Security Resources

- **CVE Database**: https://cve.mitre.org/
- **Docker Security Best Practices**: https://docs.docker.com/engine/security/
- **Pi Security Guide**: https://www.raspberrypi.org/documentation/configuration/security.md
- **NIST Cybersecurity Framework**: https://www.nist.gov/cyberframework

Remember: Security is an ongoing process, not a one-time setup. Regular monitoring, updates, and assessment are essential for maintaining a secure Pi4-TravelRouter deployment.