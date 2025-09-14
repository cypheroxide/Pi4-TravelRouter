# Phase 4: Service Installation

## Overview

This phase covers the installation and configuration of core services including Pi-hole DNS filtering, VPN services (Tailscale and ProtonVPN), and container management with Portainer.

**Prerequisites:**
- Completed Phase 1-3 (System, Docker, and Network setup)
- Working access point with internet connectivity
- Docker and Docker Compose installed

**Estimated Time:** 45-75 minutes

## Step 1: Install Pi-hole DNS Filtering

### 1.1 Deploy Pi-hole Container

Navigate to your Docker storage directory:

```bash
cd /mnt/storage/docker/compose
```

Create the Pi-hole configuration:

```bash
# Create Pi-hole directories
sudo mkdir -p /mnt/storage/docker/pihole/{etc,dnsmasq.d}

# Set proper permissions
sudo chown -R 999:999 /mnt/storage/docker/pihole
```

Deploy Pi-hole using our pre-configured compose file:

```bash
# Deploy Pi-hole
sudo docker compose -f docker-compose-pihole.yml up -d

# Check deployment status
sudo docker ps
sudo docker logs pihole
```

### 1.2 Configure Pi-hole Settings

Access the Pi-hole web interface at `http://192.168.4.1/admin`

**Initial Configuration:**
1. Set admin password (change from default)
2. Configure upstream DNS servers (1.1.1.1, 8.8.8.8)
3. Enable DHCP if not using dnsmasq
4. Import custom blocklists

```bash
# Set Pi-hole admin password
sudo docker exec pihole pihole -a -p "YourSecurePasswordHere"
```

### 1.3 Add Custom Blocklists

Configure comprehensive blocklists for enhanced protection:

```bash
# Access Pi-hole admin interface
# Go to Group Management → Adlists
# Add these recommended lists:

# Essential blocklists (add via web interface)
# - StevenBlack's Unified List (default)
# - BlockListProject - Ads: https://blocklistproject.github.io/Lists/ads.txt
# - BlockListProject - Malware: https://blocklistproject.github.io/Lists/malware.txt
# - BlockListProject - Tracking: https://blocklistproject.github.io/Lists/tracking.txt
```

**🔗 For detailed Pi-hole configuration, see [Pi-hole Configuration Guide](../configuration/pihole.md)**

## Step 2: Install Container Management

### 2.1 Deploy Portainer

Create Portainer configuration:

```bash
# Create Portainer directory
sudo mkdir -p /mnt/storage/docker/portainer

# Deploy Portainer
cat > /tmp/portainer-compose.yml << 'EOF'
version: '3.8'

services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports:
      - '9000:9000'
      - '9443:9443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/storage/docker/portainer:/data
    networks:
      - pi-router-net

networks:
  pi-router-net:
    external: true
EOF

# Deploy Portainer
sudo docker compose -f /tmp/portainer-compose.yml up -d
```

### 2.2 Access Portainer

1. Open browser to `http://192.168.4.1:9000`
2. Create admin user and password
3. Select "Docker" as the environment to manage
4. Connect to local Docker socket

## Step 3: Install VPN Services

### 3.1 Install Tailscale (Personal Network VPN)

Install Tailscale for secure access to your home/office network:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale and authenticate
sudo tailscale up --advertise-routes=192.168.4.0/24

# Check status
sudo tailscale status
```

**Configuration:**
- Log into your Tailscale admin console
- Approve the route advertisement for 192.168.4.0/24
- Enable key expiry and other security settings

### 3.2 Install ProtonVPN (Privacy VPN)

Install ProtonVPN CLI for routing all traffic through VPN:

```bash
# Install ProtonVPN repository
curl -L https://protonvpn.com/download/proton-vpn-gnulinux-stable-release_1.0.0-1_all.deb \
  -o /tmp/protonvpn-release.deb

sudo dpkg -i /tmp/protonvpn-release.deb
sudo apt update

# Install ProtonVPN CLI
sudo apt install -y protonvpn-cli

# Initialize ProtonVPN
sudo protonvpn-cli login
```

**🔗 For detailed ProtonVPN configuration, see [ProtonVPN Configuration Guide](../configuration/protonvpn.md)**

### 3.3 Install WG-Easy (Optional)

Deploy WG-Easy for managing additional WireGuard clients:

```bash
# Deploy WG-Easy
cd /mnt/storage/docker/compose
sudo docker compose -f docker-compose-wg-easy.yml up -d

# Check deployment
sudo docker logs wg-easy
```

Access WG-Easy at `http://192.168.4.1:51821`

## Step 4: Service Integration and Testing

### 4.1 Configure DNS Resolution

Update system DNS to use Pi-hole:

```bash
# Update /etc/systemd/resolved.conf
sudo nano /etc/systemd/resolved.conf

# Add these lines:
DNS=192.168.4.1
FallbackDNS=1.1.1.1 8.8.8.8
Domains=local

# Restart systemd-resolved
sudo systemctl restart systemd-resolved
```

### 4.2 Test DNS Filtering

Verify Pi-hole is blocking ads:

```bash
# Test blocked domain
nslookup doubleclick.net 192.168.4.1

# Test allowed domain
nslookup google.com 192.168.4.1

# Check Pi-hole statistics
sudo docker exec pihole pihole -c
```

### 4.3 Test VPN Connectivity

**Test Tailscale:**
```bash
# Check connection to Tailscale network
sudo tailscale status
ping 100.x.x.x  # Replace with your Tailscale IP
```

**Test ProtonVPN:**
```bash
# Connect to fastest server
sudo protonvpn-cli connect --fastest

# Check public IP
curl ifconfig.me

# Disconnect
sudo protonvpn-cli disconnect
```

### 4.4 Validate Service Integration

Run comprehensive service tests:

```bash
#!/bin/bash
# Service validation script

echo "=== Pi4-TravelRouter Service Check ==="

# Check Docker containers
echo "Docker Containers:"
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check network services
echo -e "\nNetwork Services:"
sudo systemctl status hostapd --no-pager -l
sudo systemctl status dnsmasq --no-pager -l

# Test DNS resolution
echo -e "\nDNS Test:"
dig @192.168.4.1 google.com +short

# Check VPN services
echo -e "\nVPN Services:"
sudo tailscale status 2>/dev/null || echo "Tailscale not active"
sudo protonvpn-cli status 2>/dev/null || echo "ProtonVPN not active"

# Check web interfaces
echo -e "\nWeb Interface Status:"
curl -s -o /dev/null -w "Pi-hole: %{http_code}\n" http://192.168.4.1/admin
curl -s -o /dev/null -w "Portainer: %{http_code}\n" http://192.168.4.1:9000

echo -e "\n=== Service Check Complete ==="
```

## Step 5: Final Configuration and Optimization

### 5.1 Enable Service Auto-Start

Ensure all services start automatically on boot:

```bash
# Enable Docker service
sudo systemctl enable docker

# Create Tailscale auto-start service
sudo systemctl enable tailscaled
```

### 5.2 Configure Log Rotation

Set up log rotation to prevent disk space issues:

```bash
# Configure Docker log rotation (already done in docker daemon.json)
# Configure system log rotation
sudo nano /etc/logrotate.d/pi4-router

# Add content:
/var/log/pi4-router/*.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
}
```

### 5.3 Create Management Scripts

Create convenient management scripts:

```bash
# Create service management script
cat > /usr/local/bin/router-status.sh << 'EOF'
#!/bin/bash
echo "=== Pi4-TravelRouter Status ==="
echo "System Load: $(uptime | cut -d',' -f3-)"
echo "Memory: $(free -h | grep Mem | awk '{print $3"/"$2}')"
echo "Storage: $(df -h /mnt/storage | tail -1 | awk '{print $3"/"$2" ("$5")"}')"
echo "Temperature: $(vcgencmd measure_temp)"
echo
echo "Network Interfaces:"
ip -br addr show wlan0 wlan1
echo
echo "Connected Devices:"
arp -a | grep 192.168.4 | wc -l
echo
echo "Docker Containers:"
docker ps --format "table {{.Names}}\t{{.Status}}"
EOF

sudo chmod +x /usr/local/bin/router-status.sh
```

## Step 6: Usage Instructions

### 6.1 Web Interface Access

| Service | URL | Purpose | Default Credentials |
|---------|-----|---------|---------------------|
| **Pi-hole** | `http://192.168.4.1/admin` | DNS filtering management | Password set in Step 1.2 |
| **Portainer** | `http://192.168.4.1:9000` | Docker container management | Created during setup |
| **WG-Easy** | `http://192.168.4.1:51821` | WireGuard VPN management | Set in compose file |

### 6.2 Common Management Commands

```bash
# Connect to public Wi-Fi
sudo /usr/local/bin/connect-wifi.sh "Network-Name" "password"

# Check system status
/usr/local/bin/router-status.sh

# Restart services
sudo systemctl restart hostapd dnsmasq
sudo docker restart pihole

# VPN management
sudo tailscale up/down
sudo protonvpn-cli connect/disconnect

# View logs
sudo journalctl -u hostapd -f
sudo docker logs pihole -f
```

### 6.3 Backup Important Configuration

Create backup of service configurations:

```bash
# Backup script (run regularly)
sudo /usr/local/bin/backup-router.sh

# Manual backup
sudo tar -czf /mnt/storage/backups/services-$(date +%F).tar.gz \
  /mnt/storage/docker \
  /etc/tailscale \
  /etc/systemd/network
```

## Next Steps

### Security Hardening
🔒 **[Security Hardening Guide](../security/)** - Secure your installation

### Monitoring and Maintenance  
📊 **[Maintenance Guide](../maintenance/)** - Keep your system healthy

### Troubleshooting
🔍 **[Troubleshooting Guide](../troubleshooting/)** - Resolve common issues

### Advanced Configuration
⚙️ **[Configuration Reference](../configuration/)** - Detailed service configuration

---

## Installation Complete!

Your Pi4-TravelRouter is now fully configured with:

- ✅ **Pi-hole DNS filtering** blocking ads and malware
- ✅ **Container management** via Portainer web interface  
- ✅ **VPN integration** with Tailscale and ProtonVPN
- ✅ **Automated services** starting on boot
- ✅ **Web interfaces** for easy management
- ✅ **Backup system** for configuration protection

Connect your devices to the `RaspberryPi-Travel` Wi-Fi network and enjoy secure, filtered internet access anywhere!