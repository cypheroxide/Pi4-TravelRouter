# Raspberry Pi 4 Travel Router Complete Setup Plan

## System Optimization Updates (Latest)

**Key Performance Improvements Made:**
- **8GB Swap File on SSD**: Moved from 512MB swap on SD card to 8GB on external SSD for better performance and longevity
- **Optimized Memory Management**: Set vm.swappiness=1 and vm.min_free_kbytes=8192 for better RAM utilization
- **Docker Log Rotation**: Configured 10MB max log files with 3-file rotation to prevent disk space issues
- **Enhanced Backup System**: Comprehensive daily backups with automatic cleanup (7-day retention)
- **Automated Scheduling**: Daily backups at 3:00 AM with logging to /var/log/backup.log

These optimizations prepare the system for running multiple Docker containers efficiently.

---

## Prerequisites & Hardware
- Raspberry Pi 4 (4GB+ RAM recommended)
- MicroSD card (32GB+ Class 10)
- Panda PAU09 N600 USB Wi-Fi adapter
- USB drive for storage (64GB+ recommended)
- Power supply and cables

## Phase 1: Base System Setup

### Step 1: Install Raspberry Pi OS
1. Download Raspberry Pi OS Lite (64-bit) from official website
2. Use Raspberry Pi Imager to flash to SD card
3. Enable SSH in imager advanced options
4. Set username/password and enable SSH
5. Boot Pi and connect via Ethernet initially

### Step 2: Initial System Configuration
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y git curl wget vim htop iptables-persistent

# Configure timezone and locale
sudo raspi-config
# Navigate to: Localisation Options → Timezone
# Navigate to: Localisation Options → Locale

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Step 3: USB Drive Setup
```bash
# Identify USB drive
lsblk

# Format USB drive (assuming /dev/sda1)
sudo mkfs.ext4 /dev/sda1

# Create mount point and mount
sudo mkdir -p /mnt/storage
sudo mount /dev/sda1 /mnt/storage

# Get UUID for persistent mounting
sudo blkid /dev/sda1

# Add to fstab (replace UUID with actual UUID)
echo 'UUID=your-uuid-here /mnt/storage ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

# Create directories for Docker and data
sudo mkdir -p /mnt/storage/docker
sudo mkdir -p /mnt/storage/data
sudo chown -R $USER:$USER /mnt/storage
```

## Phase 2: Docker Installation & Configuration

### Step 4: Install Docker
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Configure Docker to use USB storage
sudo systemctl stop docker
sudo mkdir -p /mnt/storage/docker
echo '{"data-root": "/mnt/storage/docker"}' | sudo tee /etc/docker/daemon.json
sudo systemctl start docker
sudo systemctl enable docker

# Logout and login to apply group changes
exit
# SSH back in
```

### Step 5: Install Portainer
```bash
# Create Portainer directory (using correct storage path)
sudo mkdir -p /mnt/storage/docker/portainer

# Run Portainer with bind mount (more reliable than named volumes)
sudo docker run -d \
  --name portainer \
  --restart always \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /mnt/storage/docker/portainer:/data \
  portainer/portainer-ce:latest
```

## Phase 3: Network Configuration & Access Point

### Step 6: Configure USB Wi-Fi Adapter
  #### NOTE: Read all of these instructions before implementing. I experienced `dnsmasq` errors and had to change my setup (found in Issue #4)
```bash
# Install hostapd and dnsmasq
sudo apt install -y hostapd dnsmasq

# Stop services initially
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq

# Check if USB adapter is recognized
lsusb
iwconfig

# Configure hostapd
sudo tee /etc/hostapd/hostapd.conf > /dev/null <<EOF
interface=wlan1
driver=nl80211
ssid=RaspberryPi-Travel
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=ChangeThisPassword123
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
EOF

# Configure dnsmasq 
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
sudo tee /etc/dnsmasq.conf > /dev/null <<EOF
interface=wlan1
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
domain=local
address=/pi.hole/192.168.4.1
EOF

# Configure static IP for AP interface
sudo tee -a /etc/dhcpcd.conf > /dev/null <<EOF
interface wlan1
static ip_address=192.168.4.1/24
nohook wpa_supplicant
EOF

# Enable hostapd
echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' | sudo tee -a /etc/default/hostapd
```

### Step 7: Configure NAT and Routing
```bash
# Create iptables rules script
sudo tee /etc/iptables-rules.sh > /dev/null <<'EOF'
#!/bin/bash

# Clear existing rules
iptables -F
iptables -t nat -F

# Set default policies
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# NAT rules for internet sharing
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan1 -o wlan0 -j ACCEPT

# Allow local traffic
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -i wlan1 -j ACCEPT

# Save rules
iptables-save > /etc/iptables/rules.v4
EOF

sudo chmod +x /etc/iptables-rules.sh
sudo /etc/iptables-rules.sh
```

### Step 8: Create Connection Management Script
```bash
# Create Wi-Fi connection script
sudo tee /usr/local/bin/connect-wifi.sh > /dev/null <<'EOF'
#!/bin/bash

SSID="$1"
PASSWORD="$2"

if [ -z "$SSID" ]; then
    echo "Usage: $0 <SSID> [PASSWORD]"
    exit 1
fi

# Stop AP mode
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq

# Configure wpa_supplicant
sudo tee /etc/wpa_supplicant/wpa_supplicant.conf > /dev/null <<EOL
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="$SSID"
    psk="$PASSWORD"
    priority=1
}
EOL

# Restart networking
sudo systemctl restart dhcpcd
sleep 5

# Check connection
if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
    echo "Connected to $SSID successfully!"
    # Update iptables for new interface
    sudo /etc/iptables-rules.sh
    # Start AP services
    sudo systemctl start dnsmasq
    sudo systemctl start hostapd
else
    echo "Failed to connect to $SSID"
fi
EOF

sudo chmod +x /usr/local/bin/connect-wifi.sh
```

## Phase 4: PiHole Installation

### Step 9: Install PiHole via Docker
```bash
# Create PiHole directories (using correct storage path)
sudo mkdir -p /mnt/storage/docker/pihole/etc
sudo mkdir -p /mnt/storage/docker/pihole/dnsmasq.d
sudo mkdir -p /mnt/storage/docker/compose

# Set proper permissions for PiHole
sudo chown -R 999:999 /mnt/storage/docker/pihole/

# Create docker-compose file for PiHole
tee /mnt/storage/docker/compose/docker-compose-pihole.yml > /dev/null <<EOF
version: '3.8'
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
    environment:
      TZ: 'America/Chicago'
      WEBPASSWORD: 'changethispassword'
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
    volumes:
      - '/mnt/storage/docker/pihole/etc:/etc/pihole'
      - '/mnt/storage/docker/pihole/dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
EOF

# Deploy PiHole
cd /mnt/storage/docker/compose && sudo docker-compose -f docker-compose-pihole.yml up -d
```

### Step 10: Configure PiHole Integration
```bash
# Update dnsmasq to use PiHole
sudo tee /etc/dnsmasq.conf > /dev/null <<EOF
interface=wlan1
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
domain=local
address=/pi.hole/192.168.4.1
server=127.0.0.1#53
EOF

# Restart dnsmasq
sudo systemctl restart dnsmasq
```

## Phase 5: VPN Configuration

### Step 11: Install Tailscale
```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale
sudo tailscale up --advertise-routes=192.168.4.0/24

# Note: Complete authentication via the provided URL
```

### Step 12: Configure WireGuard
```bash
# Install WireGuard
sudo apt install -y wireguard

# Create WireGuard directory
sudo mkdir -p /etc/wireguard

# Create ProtonVPN WireGuard config (you'll need to get this from ProtonVPN)
sudo tee /etc/wireguard/protonvpn.conf > /dev/null <<'EOF'
# Replace this with your actual ProtonVPN WireGuard config
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = YOUR_ASSIGNED_IP/32
DNS = 10.2.0.1

[Peer]
PublicKey = PROTONVPN_PUBLIC_KEY
Endpoint = YOUR_ENDPOINT:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Create toggle script
sudo tee /usr/local/bin/toggle-vpn.sh > /dev/null <<'EOF'
#!/bin/bash

if [ "$1" = "up" ]; then
    echo "Starting ProtonVPN..."
    sudo wg-quick up protonvpn
    # Update iptables for VPN
    sudo iptables -t nat -D POSTROUTING -o wlan0 -j MASQUERADE 2>/dev/null
    sudo iptables -t nat -A POSTROUTING -o protonvpn -j MASQUERADE
elif [ "$1" = "down" ]; then
    echo "Stopping ProtonVPN..."
    sudo wg-quick down protonvpn
    # Restore original routing
    sudo iptables -t nat -D POSTROUTING -o protonvpn -j MASQUERADE 2>/dev/null
    sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
else
    echo "Usage: $0 [up|down]"
fi
EOF

sudo chmod +x /usr/local/bin/toggle-vpn.sh
```

## Phase 6: Service Management & Startup

### Step 13: Create Startup Service
```bash
# Create startup service
sudo tee /etc/systemd/system/travel-router.service > /dev/null <<'EOF'
[Unit]
Description=Travel Router Setup
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/travel-router-start.sh
RemainAfterExit=yes
User=root

[Install]
WantedBy=multi-user.target
EOF

# Create startup script
sudo tee /usr/local/bin/travel-router-start.sh > /dev/null <<'EOF'
#!/bin/bash

# Wait for system to be ready
sleep 30

# Start Docker containers
cd /mnt/storage/docker/compose && docker-compose -f docker-compose-pihole.yml up -d

# Configure iptables
/etc/iptables-rules.sh

# Start AP services
systemctl start hostapd
systemctl start dnsmasq

# Start Tailscale if not running
if ! tailscale status >/dev/null 2>&1; then
    tailscale up --advertise-routes=192.168.4.0/24
fi

echo "Travel router started successfully!"
EOF

sudo chmod +x /usr/local/bin/travel-router-start.sh
sudo systemctl enable travel-router.service
```

### Step 14: Create Management Interface
```bash
# Create simple web interface for management
mkdir -p /mnt/storage/docker/web-interface
tee /mnt/storage/docker/web-interface/docker-compose.yml > /dev/null <<EOF
version: '3.8'
services:
  travel-router-ui:
    container_name: travel-router-ui
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - './html:/usr/share/nginx/html'
EOF

# Create simple HTML interface
mkdir -p /mnt/storage/docker/web-interface/html
tee /mnt/storage/docker/web-interface/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Raspberry Pi Travel Router</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .container { max-width: 800px; margin: 0 auto; }
        .section { margin: 20px 0; padding: 20px; border: 1px solid #ccc; border-radius: 5px; }
        .button { background: #4CAF50; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; display: inline-block; margin: 5px; }
        .button:hover { background: #45a049; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🛜 Raspberry Pi Travel Router</h1>
        
        <div class="section">
            <h3>📊 Services</h3>
            <a href="http://192.168.4.1:9000" class="button" target="_blank">Portainer</a>
            <a href="http://192.168.4.1/admin" class="button" target="_blank">PiHole Admin</a>
        </div>
        
        <div class="section">
            <h3>📡 Network Status</h3>
            <p><strong>Access Point:</strong> RaspberryPi-Travel</p>
            <p><strong>Router IP:</strong> 192.168.4.1</p>
            <p><strong>DHCP Range:</strong> 192.168.4.2 - 192.168.4.20</p>
        </div>
        
        <div class="section">
            <h3>🔧 Quick Commands</h3>
            <p><strong>Connect to Wi-Fi:</strong> <code>sudo /usr/local/bin/connect-wifi.sh "SSID" "PASSWORD"</code></p>
            <p><strong>Toggle VPN:</strong> <code>sudo /usr/local/bin/toggle-vpn.sh [up|down]</code></p>
            <p><strong>Restart Services:</strong> <code>sudo systemctl restart travel-router</code></p>
        </div>
    </div>
</body>
</html>
EOF

# Start the web interface
cd /mnt/storage/docker/web-interface && docker-compose up -d
```

## Phase 7: Final Configuration & Testing

### Step 15: System Optimization
```bash
# IMPORTANT: Move swap from SD card to external SSD for better performance
# First, disable existing swap (managed by dphys-swapfile)
sudo swapoff /var/swap
sudo rm -f /var/swap
sudo systemctl disable dphys-swapfile

# Create 8GB swap file on external SSD (adjust size based on planned Docker workload)
sudo fallocate -l 8G /mnt/storage/swapfile
sudo chmod 600 /mnt/storage/swapfile
sudo mkswap /mnt/storage/swapfile
sudo swapon /mnt/storage/swapfile

# Make swap persistent across reboots
echo "/mnt/storage/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# Apply system performance optimizations
sudo sysctl vm.swappiness=1          # Minimize swap usage (prefer RAM)
sudo sysctl vm.min_free_kbytes=8192  # Reserve 8MB free memory

# Make sysctl settings persistent
echo "vm.swappiness=1" | sudo tee -a /etc/sysctl.conf
echo "vm.min_free_kbytes=8192" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configure Docker log rotation to prevent disk space issues
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "data-root": "/mnt/storage/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

### Step 16: Create Backup Script
```bash
# Create backup directory and set permissions
sudo mkdir -p /mnt/storage/backups
sudo chown $(whoami) /mnt/storage/backups

# Create comprehensive backup script
sudo tee /usr/local/bin/backup-router.sh > /dev/null <<'EOF'
#!/bin/bash

# Pi 4 Travel Router Backup Script
TIMESTAMP=$(date +%F_%H%M)
DEST=/mnt/storage/backups/backup_$TIMESTAMP.tar.gz

echo "Starting backup at $(date)"

# Create backup of important system configurations
tar czf "$DEST" \
  /etc/fstab \
  /etc/sysctl.conf \
  /etc/docker/daemon.json \
  /etc/hosts \
  /etc/network/interfaces.d \
  /etc/rc.local \
  /etc/dhcpcd.conf \
  /etc/dnsmasq.conf \
  /etc/hostapd \
  /etc/wpa_supplicant \
  /mnt/storage/docker-compose-pihole.yml \
  /mnt/storage/docker \
  2>/dev/null

if [ $? -eq 0 ]; then
    echo "Backup completed successfully: $DEST"
    echo "Backup size: $(du -h "$DEST" | cut -f1)"
else
    echo "Backup completed with some warnings: $DEST"
    echo "Some files may not exist yet - this is normal for partial setups"
fi

# Clean up old backups (keep last 7 days)
find /mnt/storage/backups -name "backup_*.tar.gz" -mtime +7 -delete 2>/dev/null

echo "Backup finished at $(date)"
EOF

# Make script executable and test
sudo chmod +x /usr/local/bin/backup-router.sh
/usr/local/bin/backup-router.sh

# Schedule automated daily backups at 3:00 AM
echo "0 3 * * * /usr/local/bin/backup-router.sh >> /var/log/backup.log 2>&1" | sudo crontab -

# Verify cron job was scheduled
echo "Scheduled backup job:"
sudo crontab -l
```

## Testing & Verification Steps

### Step 17: Complete Testing Checklist
1. **Reboot and verify services start automatically**
   ```bash
   sudo reboot
   # Wait for boot, then check:
   sudo systemctl status travel-router
   docker ps
   ```

2. **Test Access Point**
   - Connect device to "RaspberryPi-Travel" network
   - Verify you get IP in 192.168.4.x range
   - Test internet connectivity

3. **Test PiHole**
   - Visit http://192.168.4.1/admin
   - Verify ad blocking is working
   - Check query logs

4. **Test Portainer**
   - Visit http://192.168.4.1:9000
   - Verify Docker containers are visible
   - Test container management

5. **Test VPN Connections**
   ```bash
   # Test Tailscale
   sudo tailscale status
   
   # Test WireGuard
   sudo /usr/local/bin/toggle-vpn.sh up
   curl ifconfig.me  # Should show VPN IP
   sudo /usr/local/bin/toggle-vpn.sh down
   ```

6. **Test Wi-Fi Connection**
   ```bash
   sudo /usr/local/bin/connect-wifi.sh "TestNetwork" "password123"
   ```

## PiHole Integration Issues & Fixes

### Issue 1: Portainer Volume Mount Failure
**Problem:** When running Portainer with named volumes, Docker attempted to mount from non-existent path `/mnt/usb-storage/portainer` causing startup failure.

**Error Message:**
```
docker: Error response from daemon: error while mounting volume '/mnt/storage/docker/volumes/portainer_data/_data': failed to mount local volume: mount /mnt/usb-storage/portainer:/mnt/storage/docker/volumes/portainer_data/_data, flags: 0x1000: no such file or directory
```

**Root Cause:** Mismatch between expected storage paths and actual directory structure. System has `/mnt/storage/` instead of `/mnt/usb-storage/`.

**Solution Steps:**
1. **Remove failed container:**
   ```bash
   sudo docker rm portainer
   ```

2. **Create correct directory structure:**
   ```bash
   sudo mkdir -p /mnt/storage/docker/portainer
   ```

3. **Use bind mount instead of named volume:**
   ```bash
   sudo docker run -d --name portainer --restart always \
     -p 9000:9000 -p 9443:9443 \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v /mnt/storage/docker/portainer:/data \
     portainer/portainer-ce:latest
   ```

### Issue 2: PiHole Configuration Path Updates
**Problem:** Original configuration assumed `/mnt/usb-storage/` but system uses `/mnt/storage/`.

**Updated PiHole Docker Compose Configuration:**
```yaml
version: '3.8'
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
    environment:
      TZ: 'America/Chicago'
      WEBPASSWORD: 'changethispassword'
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
    volumes:
      - '/mnt/storage/docker/pihole/etc:/etc/pihole'
      - '/mnt/storage/docker/pihole/dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
```

**Pre-deployment Setup:**
```bash
# Create PiHole directories with correct paths
mkdir -p /mnt/storage/docker/pihole/etc
mkdir -p /mnt/storage/docker/pihole/dnsmasq.d

# Set proper permissions
sudo chown -R 999:999 /mnt/storage/docker/pihole/
```

### Issue 3: Storage Path Standardization
**Problem:** Mixed references to `/mnt/usb-storage/` and `/mnt/storage/` throughout configuration.

**Standardized Path Structure:**
- **Base Storage:** `/mnt/storage/`
- **Docker Root:** `/mnt/storage/docker/`
- **Container Data:** `/mnt/storage/docker/<service_name>/`
- **Compose Files:** `/mnt/storage/docker/compose/`

**Updated Directory Creation Commands:**
```bash
# Create standardized directory structure
sudo mkdir -p /mnt/storage/{data,docker/{compose,portainer,pihole/{etc,dnsmasq.d}}}

# Set ownership
sudo chown -R $USER:$USER /mnt/storage/data
sudo chown -R $USER:$USER /mnt/storage/docker/compose
```

### Verification Commands
```bash
# Verify Portainer is running
sudo docker ps | grep portainer

# Check Portainer logs
sudo docker logs portainer

# Verify PiHole directories exist
ls -la /mnt/storage/docker/pihole/

# Test PiHole deployment
cd /mnt/storage/docker/compose
sudo docker-compose -f docker-compose-pihole.yml up -d

# Check all containers
sudo docker ps -a
```

## Troubleshooting Commands

```bash
# Check service status
sudo systemctl status hostapd dnsmasq docker travel-router

# Check network interfaces
ip addr show
iwconfig

# Check Docker containers
docker ps -a
docker logs pihole
docker logs portainer

# Check volume mounts
docker inspect portainer | grep -A 10 "Mounts"
docker inspect pihole | grep -A 10 "Mounts"

# Check storage directories
ls -la /mnt/storage/docker/
df -h /mnt/storage/

# Check iptables rules
sudo iptables -t nat -L
sudo iptables -L

# Monitor logs
sudo journalctl -f -u travel-router
tail -f /var/log/syslog

# Docker system information
sudo docker system df
sudo docker system info | grep "Docker Root Dir"
```

## Usage Notes

- **Web Interface:** http://192.168.4.1:8080
- **PiHole Admin:** http://192.168.4.1/admin
- **Portainer:** http://192.168.4.1:9000
- **Default AP:** SSID "RaspberryPi-Travel", Password "ChangeThisPassword123"

Remember to change all default passwords and customize the configuration for your specific needs!
### Step 12: Configure WireGuard with wg-easy and ProtonVPN Integration

**UPDATED IMPLEMENTATION - Safe & Easy VPN Management**

```bash
# Install WireGuard and dependencies (if not already installed)
sudo apt update && sudo apt install -y wireguard wireguard-tools iptables-persistent resolvconf

# Verify WireGuard module loads
sudo modprobe wireguard && echo "WireGuard module loaded successfully"

# Create Docker directory for wg-easy
sudo mkdir -p /mnt/storage/docker/wg-easy/data
sudo chown -R cypheroxide:cypheroxide /mnt/storage/docker/wg-easy

# Create wg-easy configuration
tee /mnt/storage/docker/wg-easy/.env > /dev/null << 'WGENV'
WG_HOST=100.89.68.50
WG_PORT=51821
WG_DEFAULT_ADDRESS=10.8.0.1
WG_DEFAULT_DNS=172.18.0.2
WG_PERSISTENT_KEEPALIVE=25
WG_MTU=1420
WG_ALLOWED_IPS=10.8.0.0/24,192.168.4.0/24,192.168.12.0/24
PASSWORD=changethispassword
WGENV

# Create docker-compose.yml for wg-easy
sudo tee /mnt/storage/docker/wg-easy/docker-compose.yml > /dev/null << 'WGCOMPOSE'
version: '3.8'
services:
  wg-easy:
    image: weejewel/wg-easy
    container_name: wg-easy
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "51821:51820/udp"
      - "51822:51821/tcp"
    volumes:
      - ./data:/etc/wireguard
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    devices:
      - /dev/net/tun:/dev/net/tun
    networks:
      - wgnet

networks:
  wgnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.8.0.0/24
WGCOMPOSE

# Deploy wg-easy container
sudo docker-compose -f /mnt/storage/docker/wg-easy/docker-compose.yml up -d

# Configure firewall rules for WireGuard
sudo iptables -A INPUT -p udp --dport 51821 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 51822 -j ACCEPT

# Add NAT masquerading for wg-easy clients
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE
sudo iptables -A FORWARD -s 10.8.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -d 10.8.0.0/24 -j ACCEPT

# Save firewall rules
sudo netfilter-persistent save
```

### Step 12b: Create Safe ProtonVPN Integration Scripts

```bash
# Create safe ProtonVPN toggle script
sudo tee /usr/local/bin/safe-protonvpn.sh > /dev/null << 'SAFEVPN'
#!/bin/bash

CONFIG_DIR="/mnt/storage/projects/vpn_config"
WG_CONFIG_DIR="/etc/wireguard"
SAFE_CONFIG="$WG_CONFIG_DIR/protonvpn-safe.conf"

create_safe_config() {
    local source_config="$1"
    local server_name="$2"
    
    echo "Creating safe ProtonVPN config for $server_name..."
    
    # Copy original config but modify AllowedIPs to exclude local networks
    sudo cp "$source_config" "$SAFE_CONFIG"
    
    # Replace AllowedIPs with split-tunnel configuration
    sudo sed -i 's/AllowedIPs = 0.0.0.0\/0, ::/0/AllowedIPs = 0.0.0.0\/1, 128.0.0.0\/1/' "$SAFE_CONFIG"
    
    # Add routes to exclude local networks
    sudo tee -a "$SAFE_CONFIG" > /dev/null << 'CONFIG_EOF'

# Safe routing - exclude local networks
PostUp = ip route add 192.168.0.0/16 via %i table main
PostUp = ip route add 172.16.0.0/12 via %i table main  
PostUp = ip route add 10.0.0.0/8 via %i table main
PostUp = ip route add 169.254.0.0/16 via %i table main
PostDown = ip route del 192.168.0.0/16 via %i table main 2>/dev/null || true
PostDown = ip route del 172.16.0.0/12 via %i table main 2>/dev/null || true
PostDown = ip route del 10.0.0.0/8 via %i table main 2>/dev/null || true  
PostDown = ip route del 169.254.0.0/16 via %i table main 2>/dev/null || true
CONFIG_EOF
    
    sudo chmod 600 "$SAFE_CONFIG"
    echo "Safe config created for $server_name"
}

case "$1" in
    "up-se")
        create_safe_config "$CONFIG_DIR/pi-router-SE-US-1.conf" "Swedish"
        echo "Starting ProtonVPN Swedish server (safe mode)..."
        sudo wg-quick up protonvpn-safe
        echo "ProtonVPN (Swedish) started in safe mode"
        ;;
    "up-ch") 
        create_safe_config "$CONFIG_DIR/pi-router-swiss-CH-US-3.conf" "Swiss"
        echo "Starting ProtonVPN Swiss server (safe mode)..."
        sudo wg-quick up protonvpn-safe
        echo "ProtonVPN (Swiss) started in safe mode"
        ;;
    "down")
        echo "Stopping ProtonVPN..."
        sudo wg-quick down protonvpn-safe 2>/dev/null || echo "ProtonVPN was not running"
        echo "ProtonVPN stopped"
        ;;
    "status")
        echo "ProtonVPN Status:"
        sudo wg show protonvpn-safe 2>/dev/null || echo "ProtonVPN is not running"
        ;;
    "test")
        echo "Testing connectivity..."
        echo "External IP: $(curl -s --max-time 5 ifconfig.me || echo 'Failed to get external IP')"
        echo "Pi4-router ping: $(ping -c 1 192.168.12.155 >/dev/null 2>&1 && echo 'OK' || echo 'Failed')"
        echo "Tailscale ping: $(ping -c 1 100.89.68.50 >/dev/null 2>&1 && echo 'OK' || echo 'Failed')"
        ;;
    *)
        echo "Usage: $0 {up-se|up-ch|down|status|test}"
        echo ""
        echo "  up-se   - Start ProtonVPN with Swedish server (safe mode)"
        echo "  up-ch   - Start ProtonVPN with Swiss server (safe mode)"
        echo "  down    - Stop ProtonVPN"
        echo "  status  - Show ProtonVPN status"
        echo "  test    - Test connectivity"
        exit 1
        ;;
esac
SAFEVPN

# Make script executable
sudo chmod +x /usr/local/bin/safe-protonvpn.sh
```

### Step 12c: Create Management Summary Script

```bash
# Create summary script for easy status checking
sudo tee /usr/local/bin/wg-summary.sh > /dev/null << 'SUMMARY'
#!/bin/bash

echo "=============================================="
echo "     Pi4 Travel Router WireGuard Status      "
echo "=============================================="
echo ""
echo "📡 Services Status:"
echo "  - wg-easy container: $(sudo docker ps --format 'table {{.Status}}' --filter 'name=wg-easy' | tail -1)"
echo "  - PiHole container:  $(sudo docker ps --format 'table {{.Status}}' --filter 'name=pihole' | tail -1)"
echo "  - Portainer:         $(sudo docker ps --format 'table {{.Status}}' --filter 'name=portainer' | tail -1)"
echo ""
echo "🔒 VPN Status:"
sudo /usr/local/bin/safe-protonvpn.sh status
echo ""
echo "🌐 Access Points:"
echo "  - wg-easy Web UI:    http://100.89.68.50:51822 (password: changethispassword)"
echo "  - PiHole Admin:      http://192.168.4.1/admin"
echo "  - Portainer:         http://192.168.4.1:9000"
echo ""
echo "⚡ Quick Commands:"
echo "  - Start ProtonVPN SE:  sudo safe-protonvpn.sh up-se"
echo "  - Start ProtonVPN CH:  sudo safe-protonvpn.sh up-ch"  
echo "  - Stop ProtonVPN:      sudo safe-protonvpn.sh down"
echo "  - Test connectivity:   sudo safe-protonvpn.sh test"
echo ""
echo "📋 Network Details:"
echo "  - Tailscale IP:        100.89.68.50"
echo "  - Local Network:       192.168.12.155"
echo "  - AP Network:          192.168.4.1"
echo "  - WireGuard Network:   10.8.0.1"
echo ""
SUMMARY

# Make summary script executable
sudo chmod +x /usr/local/bin/wg-summary.sh
```

## WireGuard Usage Guide

### 🌐 **Access Points:**
- **wg-easy Web UI**: `http://100.89.68.50:51822` (password: `changethispassword`)
- **PiHole Admin**: `http://192.168.4.1/admin`
- **Portainer**: `http://192.168.4.1:9000`

### ⚡ **Quick Commands:**
```bash
# Check overall status
sudo wg-summary.sh

# ProtonVPN Management
sudo safe-protonvpn.sh up-se      # Start Swedish server
sudo safe-protonvpn.sh up-ch      # Start Swiss server
sudo safe-protonvpn.sh down       # Stop ProtonVPN
sudo safe-protonvpn.sh status     # Check VPN status
sudo safe-protonvpn.sh test       # Test connectivity

# Docker container management
sudo docker ps                    # See running containers
sudo docker logs wg-easy          # Check wg-easy logs
```

### 🎯 **How to Use wg-easy:**

1. **Access wg-easy**: Navigate to `http://100.89.68.50:51822` in your browser
2. **Login**: Use password `changethispassword` (**CHANGE THIS!**)
3. **Create Client**: Click "Add Client" to generate WireGuard configs for your devices
4. **Download Config**: Download the `.conf` file or scan the QR code with your phone
5. **Connect**: Import the config into your WireGuard client app

### 🔒 **ProtonVPN Integration:**
- **Without ProtonVPN**: wg-easy clients get direct internet access with PiHole DNS filtering
- **With ProtonVPN**: Start ProtonVPN first, then all wg-easy clients route through the VPN
- **Safe Routing**: Local network access is preserved even when ProtonVPN is active

### 🛟 **Safety Features:**
- **Network-Safe**: ProtonVPN scripts won't break local network connectivity
- **Split Tunneling**: Local traffic stays local, internet traffic can go through VPN
- **Manual Control**: ProtonVPN only runs when explicitly started
- **Easy Recovery**: If something breaks, just run `sudo safe-protonvpn.sh down`

### 📋 **Network Architecture:**
```
Internet
    ↕
ProtonVPN (optional) ← Manual toggle
    ↕
Pi4 Router (192.168.12.155, 100.89.68.50)
    ↕
├─ Access Point (192.168.4.0/24)
├─ wg-easy Server (10.8.0.0/24) ← Web UI management
├─ PiHole (172.18.0.2) ← DNS filtering
└─ Tailscale (100.89.68.50) ← Remote access
```

### 🔧 **Troubleshooting:**
```bash
# If wg-easy stops working
sudo docker restart wg-easy

# If ProtonVPN breaks connectivity
sudo safe-protonvpn.sh down

# Check logs
sudo docker logs wg-easy
sudo docker logs pihole

# Restart all services
sudo systemctl restart travel-router
```

## Updated Testing Checklist

### Step 17: Complete Testing Checklist (Updated)

1. **Test wg-easy Web Interface**
   - Visit http://100.89.68.50:51822
   - Login with password and **change it immediately**
   - Create a test client configuration
   - Download the config file

2. **Test WireGuard Client Connection**
   - Import config into WireGuard client on phone/laptop
   - Connect and verify you get 10.8.0.x IP address
   - Test internet connectivity
   - Verify PiHole is filtering ads

3. **Test ProtonVPN Integration**
   ```bash
   # Test without ProtonVPN
   curl ifconfig.me  # Should show your normal IP
   
   # Start ProtonVPN
   sudo safe-protonvpn.sh up-se
   curl ifconfig.me  # Should show ProtonVPN IP
   
   # Test that wg-easy clients now route through ProtonVPN
   # (Connect WireGuard client and check external IP)
   
   # Stop ProtonVPN
   sudo safe-protonvpn.sh down
   ```

4. **Test Network Resilience**
   ```bash
   # This should NOT break anything now
   sudo safe-protonvpn.sh up-se
   ping -c 3 192.168.12.155  # Should work
   ping -c 3 100.89.68.50    # Should work
   ssh pi4-router            # Should work
   ```

5. **Verify Service Persistence**
   ```bash
   sudo reboot
   # Wait for boot, then check:
   sudo wg-summary.sh        # All services should be up
   ```

## WireGuard Configuration Issues & Solutions

### Issue 1: ProtonVPN Breaking Network Connectivity
**Problem:** Original ProtonVPN configs route ALL traffic (0.0.0.0/0) through VPN, breaking local network access.

**Solution:** Created safe-protonvpn.sh script that:
- Modifies AllowedIPs to use split routing (0.0.0.0/1, 128.0.0.0/1)
- Preserves local network routes (192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8)
- Allows manual control instead of auto-start

### Issue 2: wg-easy Port Conflicts  
**Problem:** wg-easy default ports could conflict with existing services.

**Solution:** 
- WireGuard UDP: Port 51821 (instead of 51820)
- Web UI TCP: Port 51822 (instead of 51821)
- Updated firewall rules accordingly

### Issue 3: DNS Configuration for wg-easy Clients
**Problem:** Clients need proper DNS configuration to use PiHole.

**Solution:** Set WG_DEFAULT_DNS=172.18.0.2 (PiHole container IP) in wg-easy config.

### Issue 4: Container Permissions and Mounting
**Problem:** wg-easy needs proper permissions and device access for WireGuard operations.

**Solution:** 
- Added /dev/net/tun device mapping
- Set proper sysctls for IP forwarding
- Used correct volume mounts for WireGuard configs

## Security Recommendations

1. **Change Default Passwords**
   - wg-easy admin password (in .env file)
   - PiHole admin password
   - Portainer admin password

2. **Firewall Configuration**
   - Only necessary ports are opened
   - Local network access preserved
   - NAT rules properly configured

3. **VPN Client Management**
   - Regularly review connected clients in wg-easy
   - Remove unused client configurations
   - Monitor bandwidth usage

4. **Regular Updates**
   ```bash
   # Update container images
   sudo docker-compose pull
   sudo docker-compose up -d
   
   # Update system packages
   sudo apt update && sudo apt upgrade -y
   ```

This implementation provides the **easiest possible** WireGuard setup with full web management, safe ProtonVPN integration, and robust network connectivity!
