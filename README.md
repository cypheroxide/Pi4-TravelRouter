# Pi4-TravelRouter

> 🚀 **Portable Raspberry Pi 4 Travel Router** - A secure, dockerized travel router with built-in ad-blocking, VPN, and wireless AP capabilities

![GitHub last commit](https://img.shields.io/github/last-commit/cypheroxide/Pi4-TravelRouter?style=flat-square)
![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=flat-square&logo=docker&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/-Raspberry_Pi-C51A4A?style=flat-square&logo=Raspberry-Pi)
![WireGuard](https://img.shields.io/badge/wireguard-%2388171A.svg?style=flat-square&logo=wireguard&logoColor=white)

## Project Overview

Pi4-TravelRouter transforms a Raspberry Pi 4 into a feature-rich travel router that provides secure internet access when traveling. Instead of connecting your devices directly to untrusted public Wi-Fi networks, this solution creates your own private wireless access point while routing traffic through the public network via a secure, managed connection.

**Key Benefits:**
- 🔒 **Security**: Your devices never connect directly to public Wi-Fi
- 🛡️ **Privacy**: Built-in Pi-hole blocks ads and trackers at the network level
- 🌐 **VPN Integration**: Seamless Tailscale and ProtonVPN WireGuard support
- 📱 **Easy Management**: Web-based interfaces for all components
- 💾 **Persistent Storage**: All data and containers stored on USB drive
- ⚡ **Optimized Performance**: 8GB SSD swap and automated backups

This project was developed as a reliable alternative to solutions like FreedomBox and RaspAP, using standard Raspberry Pi OS and well-tested components for maximum stability.

## Key Features

### 📡 **Wireless Router/Access Point**
- **Panda PAU09 N600 USB Wi-Fi Adapter**: Creates a secure wireless AP for your devices
- **Dual Wi-Fi Setup**: Built-in Wi-Fi connects to public networks, USB adapter broadcasts your private network
- **WPA2 Security**: Protected access point with customizable SSID and password
- **DHCP Server**: Automatic IP assignment for connected devices (192.168.4.0/24 subnet)

### 🐳 **Docker + Portainer Container Management**
- **Portainer Web UI**: Easy container management via web interface
- **Persistent Storage**: All containers and data stored on USB drive for portability
- **Docker Compose**: Orchestrated multi-container deployments
- **Log Management**: Automated log rotation to prevent disk space issues

### 🛡️ **Pi-hole Network-Level Ad Blocking**
- **DNS Filtering**: Blocks ads, trackers, and malicious domains at the network level
- **Custom Blocklists**: Pre-configured with BlockListProject comprehensive lists
- **Web Dashboard**: Monitor blocked queries and manage whitelist/blacklist
- **Network-Wide Protection**: All connected devices benefit from ad blocking

### 🔐 **Tailscale VPN Integration**
- **Personal Network Access**: Secure connection to your home/office network
- **Route Advertisement**: Share the travel router subnet with your Tailscale network
- **Zero-Config**: Simple setup with automatic mesh networking
- **Cross-Platform**: Access from any device with Tailscale installed

### 🌐 **WireGuard with ProtonVPN**
- **Toggle VPN Support**: Easy switching between direct connection and ProtonVPN
- **Multiple Endpoints**: Pre-configured Swiss (CH) and Swedish (SE) servers
- **Kill Switch**: Automatic failsafe to prevent traffic leaks
- **NetShield Integration**: ProtonVPN's ad-blocking and malware protection

### 📊 **Advanced DNS Blocklists**
- **BlockListProject Integration**: Comprehensive blocklists for various categories
- **Categorized Blocking**: Ads, malware, phishing, tracking, social media, and more
- **Regular Updates**: Automated blocklist updates for current threat protection
- **Customizable**: Easy addition/removal of specific blocklist categories

### 💾 **Optimized Storage & Performance**
- **USB Drive Storage**: All persistent data stored on external USB drive
- **8GB SSD Swap**: High-performance swap file on SSD for heavy workloads
- **Memory Optimization**: Tuned kernel parameters for better RAM utilization
- **Automated Backups**: Daily system backups with 7-day retention

### 🔧 **System Management & Monitoring**
- **Automated Setup Scripts**: One-command deployment and configuration
- **Service Management**: Systemd integration for reliable startup/shutdown
- **Health Monitoring**: Built-in status checks and logging
- **Recovery Tools**: Backup and restore capabilities for disaster recovery

## Architecture Overview

The Pi4-TravelRouter uses a dual Wi-Fi interface design to provide secure internet access:

```
┌─────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────┐
│                     │    │                         │    │                     │
│   Public Wi-Fi      │◄───┤  Raspberry Pi 4         │───►│  Your Devices       │
│   (Hotel/Cafe/etc)  │    │  Travel Router          │    │  (Phone/Laptop/etc) │
│                     │    │                         │    │                     │
└─────────────────────┘    └─────────────────────────┘    └─────────────────────┘
           ▲                            │                             ▲
           │                            │                             │
       wlan0 (STA)                  Pi4 Router                   wlan1 (AP)
    Built-in Wi-Fi                     Core                   USB Wi-Fi Adapter
  Connects to public                                         Creates private AP
      networks                                              (192.168.4.0/24)
                                       │
                           ┌───────────┼───────────┐
                           │                       │
                       Pi-hole                WireGuard
                     DNS Filter              VPN Tunnel
                   (Docker Container)      (Optional)
```

### Network Flow
1. **wlan0** (Built-in Wi-Fi): Connects to public Wi-Fi networks as a client
2. **wlan1** (USB Adapter): Creates a secure access point for your devices
3. **NAT Routing**: Traffic from wlan1 → wlan0 with iptables rules
4. **DNS Filtering**: All DNS queries filtered through Pi-hole
5. **VPN Routing**: Optional routing through Tailscale or ProtonVPN WireGuard

### Interface Configuration
- **wlan0**: DHCP client (receives IP from public network)
- **wlan1**: Static IP `192.168.4.1/24` (serves as gateway)
- **Access Point**: SSID: `RaspberryPi-Travel`, WPA2-PSK security
- **DHCP Range**: `192.168.4.2` - `192.168.4.20`

## Hardware Requirements

### Required Components

| Component | Specification | Purpose |
|-----------|--------------|----------|
| **Raspberry Pi 4** | 4GB RAM minimum, 8GB recommended | Main processing unit |
| **MicroSD Card** | 32GB+ Class 10 or better | Boot drive and OS |
| **USB Wi-Fi Adapter** | Panda PAU09 N600 or compatible | Creates access point (wlan1) |
| **USB Storage Drive** | 64GB+ USB 3.0 recommended | Docker containers and data |
| **Power Supply** | Official Pi 4 power adapter (5V/3A) | Reliable power for Pi + USB devices |

### Recommended Components

| Component | Specification | Benefit |
|-----------|--------------|----------|
| **External SSD** | 64GB+ USB 3.0 SSD | Faster swap and better performance |
| **USB 3.0 Hub** | Powered hub with 4+ ports | More USB ports for storage + adapters |
| **Case with Cooling** | Pi 4 case with fan/heatsinks | Better thermal management |
| **Ethernet Cable** | Cat5e/6 cable | Initial setup and troubleshooting |

### Tested Wi-Fi Adapters

✅ **Confirmed Compatible:**
- Panda PAU09 N600 (Recommended)
- TP-Link AC600T2U Plus
- Alfa AWUS036ACS

⚠️ **Requirements for USB Wi-Fi Adapter:**
- **AP Mode Support**: Must support `hostapd` access point mode
- **Driver Support**: Should work with standard Linux kernels (no proprietary drivers)
- **Sufficient Power**: USB 2.0 devices recommended (lower power consumption)
- **Dual-Band**: 2.4GHz minimum, 5GHz support beneficial

### Power Considerations
- **Total Power Draw**: ~15W (Pi 4 + USB adapter + storage)
- **Power Bank**: 20,000mAh+ for 8-12 hours portable operation
- **USB-C PD**: Supports USB-C Power Delivery for newer power banks

## Directory Structure

```
Pi4-TravelRouter/
├── README.md                           # This documentation
├── src/                               # Source files and configurations
│   ├── Lists/                         # DNS Blocklists from BlockListProject
│   │   ├── README.md                  # BlockListProject documentation
│   │   ├── ads.txt                    # Ad-blocking domains
│   │   ├── malware.txt                # Malware/phishing domains
│   │   ├── tracking.txt               # Tracking domains
│   │   ├── everything.txt             # Combined blocklist
│   │   ├── facebook.txt               # Facebook/Meta domains
│   │   ├── tiktok.txt                 # TikTok domains
│   │   ├── gambling.txt               # Gambling sites
│   │   ├── piracy.txt                 # Piracy/torrent sites
│   │   └── [additional blocklists]    # Various category-specific lists
│   │
│   ├── scripts/                       # Setup and management scripts
│   │   ├── docker.sh                  # Docker installation script
│   │   └── dockermanager.deb          # Docker management package
│   │
│   ├── wg-easy/                       # WireGuard Easy management interface
│   │   ├── docker-compose.yml         # WG-Easy container configuration
│   │   ├── README.md                  # WG-Easy documentation
│   │   └── [wg-easy files]            # WireGuard web UI components
│   │
│   ├── vpn_config/                    # VPN configuration files
│   │   ├── ch-us-03.protonvpn.udp.ovpn # ProtonVPN OpenVPN config (Swiss)
│   │   ├── pi-router-swiss-CH-US-3.conf # ProtonVPN WireGuard config (Swiss)
│   │   └── pi-router-SE-US-1.conf      # ProtonVPN WireGuard config (Swedish)
│   │
│   ├── docker-compose-pihole.yml      # Pi-hole container configuration
│   ├── rpi4-travel-router-plan.md     # Complete setup instructions
│   ├── Plan_Stages.md                 # Project planning notes
│   ├── OPTIMIZATION_COMPLETE.md       # System optimization log
│   └── [additional configs]           # Various configuration files
└── [git files]                        # .gitignore, etc.
```

### Directory Purposes

| Directory | Purpose | Key Files |
|-----------|---------|----------|
| **`src/Lists/`** | DNS blocklists for Pi-hole | `ads.txt`, `malware.txt`, `everything.txt` |
| **`src/scripts/`** | Setup and automation scripts | `docker.sh` (Docker installer) |
| **`src/wg-easy/`** | WireGuard web management UI | `docker-compose.yml`, web interface |
| **`src/vpn_config/`** | VPN configuration files | ProtonVPN configs for WireGuard/OpenVPN |
| **Root** | Documentation and compose files | `README.md`, main docker-compose files |

## Installation Guide

### Quick Start

For users who want to get started immediately with default settings:

```bash
# Clone the repository
git clone https://github.com/cypheroxide/Pi4-TravelRouter.git
cd Pi4-TravelRouter

# Make setup script executable (when available)
sudo chmod +x src/scripts/setup.sh

# Run automated setup (future enhancement)
# sudo ./src/scripts/setup.sh
```

### Detailed Installation

For complete setup instructions with customization options:

**📖 Follow the comprehensive guide:** [`src/rpi4-travel-router-plan.md`](src/rpi4-travel-router-plan.md)

The detailed installation guide covers:
- **Phase 1**: Base Raspberry Pi OS setup and system configuration
- **Phase 2**: Docker installation and USB storage configuration
- **Phase 3**: Wi-Fi AP setup with hostapd/dnsmasq
- **Phase 4**: Pi-hole installation and DNS configuration
- **Phase 5**: Tailscale and WireGuard VPN setup
- **Phase 6**: Service management and automated startup

### Prerequisites

✅ **Before You Begin:**

1. **Raspberry Pi 4** with fresh Raspberry Pi OS Lite installation
2. **SSH Access** enabled and working
3. **Internet Connection** via Ethernet (for initial setup)
4. **USB Wi-Fi Adapter** (Panda PAU09 or compatible) connected
5. **USB Storage Drive** (64GB+ recommended) connected
6. **ProtonVPN Account** (for WireGuard VPN functionality)
7. **Tailscale Account** (for personal network access)

### Installation Steps Overview

1. **System Setup**: Install base packages and configure USB storage
2. **Docker Setup**: Install Docker and Portainer with USB backend
3. **Network Setup**: Configure dual Wi-Fi interfaces and NAT routing
4. **Pi-hole Setup**: Deploy Pi-hole container with custom blocklists
5. **VPN Setup**: Configure Tailscale and ProtonVPN WireGuard
6. **Service Setup**: Configure systemd services for auto-start
7. **Optimization**: Apply performance tuning and backup configuration

### Estimated Setup Time

- **Quick Setup** (experienced users): 1-2 hours
- **Detailed Setup** (following full guide): 3-4 hours
- **First-time Setup** (new to Pi/Linux): 4-6 hours

> **💡 Tip**: Start with a wired Ethernet connection for initial setup, then configure Wi-Fi functionality once the base system is working.

## Configuration Files

The project includes several pre-configured files for easy deployment:

### Docker Compose Files

**📄 `src/docker-compose-pihole.yml`**
```yaml
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
      WEBPASSWORD: 'CHANGE_THIS_PASSWORD'
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
    volumes:
      - '/mnt/storage/pihole/etc/:/etc/pihole'
      - '/mnt/storage/pihole/dnsmasq.d:/etc/dnsmasq.d'
```

**📄 `src/wg-easy/docker-compose.yml`**
```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
```

### VPN Configuration Files

**🇨🇭 Swiss ProtonVPN WireGuard** (`src/vpn_config/pi-router-swiss-CH-US-3.conf`)
```ini
[Interface]
# NetShield = 2 (blocks ads and malware)
PrivateKey = [REDACTED_PRIVATE_KEY]
Address = 10.2.0.2/32
DNS = 10.2.0.1

[Peer]
# CH-US#3 Server
PublicKey = [REDACTED_PUBLIC_KEY]
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [REDACTED_ENDPOINT]:51820
```

**🇸🇪 Swedish ProtonVPN WireGuard** (`src/vpn_config/pi-router-SE-US-1.conf`)
```ini
[Interface]
# NetShield = 2 (blocks ads and malware)
PrivateKey = [REDACTED_PRIVATE_KEY]
Address = 10.2.0.2/32
DNS = 10.2.0.1

[Peer]
# SE-US#1 Server
PublicKey = [REDACTED_PUBLIC_KEY]
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [REDACTED_ENDPOINT]:51820
```

### DNS Blocklist Files

**📁 `src/Lists/` Directory** - Contains comprehensive blocklists from [BlockListProject](https://github.com/blocklistproject/Lists):

| File | Domains | Description |
|------|---------|-------------|
| `everything.txt` | ~3M+ domains | Combined list of all categories |
| `ads.txt` | ~500K+ domains | Advertisement servers |
| `malware.txt` | ~100K+ domains | Malware and phishing sites |
| `tracking.txt` | ~200K+ domains | Analytics and tracking |
| `facebook.txt` | ~1K+ domains | Facebook/Meta services |
| `tiktok.txt` | ~500+ domains | TikTok/ByteDance services |
| `gambling.txt` | ~10K+ domains | Gambling and betting sites |
| `piracy.txt` | ~50K+ domains | Torrent and piracy sites |

### System Optimization Settings

**⚡ Performance Optimizations Applied:**
- **8GB SSD Swap**: `/mnt/storage/swapfile` (replaces 512MB SD card swap)
- **Memory Tuning**: `vm.swappiness=1`, `vm.min_free_kbytes=8192`
- **Docker Log Rotation**: 10MB max file size, 3 file retention
- **Automated Backups**: Daily at 3:00 AM with 7-day retention

## Usage Instructions

### Connecting to Public Wi-Fi

Use the included script to connect the Pi to public Wi-Fi networks:

```bash
# Connect to an open network
sudo /usr/local/bin/connect-wifi.sh "Hotel-WiFi"

# Connect to a password-protected network  
sudo /usr/local/bin/connect-wifi.sh "Cafe-WiFi" "password123"
```

The script will:
1. Temporarily stop AP services
2. Configure wpa_supplicant with network credentials
3. Restart networking services
4. Test internet connectivity
5. Restart AP services if successful

### Managing VPN Connections

#### Tailscale VPN

```bash
# Start Tailscale (personal network access)
sudo tailscale up --advertise-routes=192.168.4.0/24

# Check Tailscale status
sudo tailscale status

# Stop Tailscale
sudo tailscale down
```

#### ProtonVPN WireGuard

```bash
# Start ProtonVPN (Swiss server)
sudo /usr/local/bin/toggle-vpn.sh up

# Stop ProtonVPN
sudo /usr/local/bin/toggle-vpn.sh down

# Check WireGuard status
sudo wg show
```

### Web Interface Access

Once connected to the travel router's Wi-Fi (`RaspberryPi-Travel`):

| Service | URL | Purpose | Default Credentials |
|---------|-----|---------|---------------------|
| **Pi-hole** | `http://192.168.4.1/admin` | DNS filtering management | Password: `CHANGE_THIS_PASSWORD` |
| **Portainer** | `http://192.168.4.1:9000` | Docker container management | Set during first login |
| **WG-Easy** | `http://192.168.4.1:51821` | WireGuard client management | Password: Set in compose file |
| **Router SSH** | `ssh pi@192.168.4.1` | Command line access | SSH key or password |

### Common Management Tasks

**📊 Monitor System Status:**
```bash
# Check all services
sudo systemctl status travel-router
sudo systemctl status hostapd
sudo systemctl status dnsmasq

# Check Docker containers
sudo docker ps

# View Pi-hole logs
sudo docker logs pihole
```

**🔄 Restart Services:**
```bash
# Restart access point
sudo systemctl restart hostapd dnsmasq

# Restart Pi-hole
sudo docker restart pihole

# Apply iptables rules
sudo /etc/iptables-rules.sh
```

**📱 Connected Devices:**
```bash
# View connected devices
arp -a | grep 192.168.4

# Check DHCP leases
sudo cat /var/lib/dhcp/dhcpd.leases
```

## Network Details

### Interface Configuration

| Interface | Type | IP Address | Purpose | Status |
|-----------|------|------------|---------|--------|
| **eth0** | Ethernet | DHCP | Initial setup, backup connection | Optional |
| **wlan0** | Built-in Wi-Fi | DHCP from public network | Internet gateway (STA mode) | Active |
| **wlan1** | USB Wi-Fi Adapter | `192.168.4.1/24` | Access Point for client devices | Active |
| **tailscale0** | VPN Interface | Tailscale-assigned | Personal network tunnel | Optional |
| **protonvpn** | WireGuard VPN | `10.2.0.2/32` | ProtonVPN tunnel | Optional |

### IP Address Ranges

**🏠 Access Point Subnet: `192.168.4.0/24`**
- **Gateway**: `192.168.4.1` (Raspberry Pi)
- **DHCP Range**: `192.168.4.2` - `192.168.4.20` (19 addresses)
- **Reserved**: `192.168.4.21` - `192.168.4.254` (static assignments)
- **Broadcast**: `192.168.4.255`

**🌐 VPN Networks:**
- **ProtonVPN**: `10.2.0.2/32` (point-to-point)
- **Tailscale**: `100.x.x.x/32` (assigned by Tailscale)
- **WG-Easy**: `10.42.42.0/24` (WireGuard clients)

### NAT and Routing Rules

The system uses iptables for NAT and routing between interfaces:

```bash
# Main NAT rule (routes wlan1 traffic to wlan0)
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

# Forward traffic between interfaces
iptables -A FORWARD -i wlan0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan1 -o wlan0 -j ACCEPT

# Allow local traffic to Pi services
iptables -A INPUT -i wlan1 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
```

**VPN Routing (when ProtonVPN is active):**
```bash
# Replace default NAT with ProtonVPN tunnel
iptables -t nat -D POSTROUTING -o wlan0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o protonvpn -j MASQUERADE
```

### DNS Configuration

**📡 DNS Resolution Flow:**
1. **Client devices** → `192.168.4.1:53` (Pi-hole)
2. **Pi-hole filtering** → Blocked domains return NXDOMAIN
3. **Allowed domains** → Forwarded to upstream DNS
4. **Upstream DNS**: `1.1.1.1`, `8.8.8.8` (or VPN DNS when active)

**DNS Servers by Connection Mode:**
- **Direct**: Cloudflare (`1.1.1.1`) + Google (`8.8.8.8`)
- **ProtonVPN**: ProtonVPN DNS (`10.2.0.1`)
- **Tailscale**: Split DNS (local domains to Tailscale)

### Port Configuration

| Port | Protocol | Service | Access |
|------|----------|---------|--------|
| `22` | TCP | SSH | Local network only |
| `53` | TCP/UDP | Pi-hole DNS | Internal clients only |
| `80` | TCP | Pi-hole Web UI | Local network only |
| `9000` | TCP | Portainer | Local network only |
| `51820` | UDP | WireGuard | Internet (for remote clients) |
| `51821` | TCP | WG-Easy Web UI | Local network only |

## Backup & Maintenance

### Automated Backup System

**💾 Daily Backups** configured via `/usr/local/bin/backup-router.sh`:

```bash
# Backup script runs daily at 3:00 AM
0 3 * * * /usr/local/bin/backup-router.sh >> /var/log/backup.log 2>&1
```

**What's Backed Up:**
- System configuration files (`/etc/hostapd/`, `/etc/dnsmasq.conf`, etc.)
- Docker compose files and container volumes
- Pi-hole configuration and blocklists
- VPN configuration files
- Network configuration (`/etc/dhcpcd.conf`, iptables rules)
- Custom scripts and services

**Backup Location:** `/mnt/storage/backups/`
**Retention:** 7 days (older backups automatically deleted)
**Log File:** `/var/log/backup.log`

### Manual Backup Commands

```bash
# Run backup immediately
sudo /usr/local/bin/backup-router.sh

# View backup log
sudo tail -f /var/log/backup.log

# List available backups
ls -la /mnt/storage/backups/
```

### System Updates

**🔄 Monthly Maintenance Routine:**

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Update Docker containers
sudo docker-compose pull
sudo docker-compose up -d

# 3. Clean up old images
sudo docker system prune -f

# 4. Update Tailscale
sudo tailscale update

# 5. Restart services
sudo systemctl restart travel-router
```

**⚙️ System Health Checks:**

```bash
# Check disk usage
df -h /mnt/storage

# Check memory usage
free -h

# Check swap usage
swapon -s

# Check system temperature
vcgencmd measure_temp
```

### Disaster Recovery

**🆘 Complete System Restore:**

1. **Fresh Pi Installation**: Install Raspberry Pi OS on new SD card
2. **Restore from Backup**: 
   ```bash
   # Mount USB drive
   sudo mount /dev/sda1 /mnt/storage
   
   # Extract latest backup
   cd /
   sudo tar -xzf /mnt/storage/backups/backup-YYYYMMDD.tar.gz
   
   # Restart services
   sudo systemctl restart hostapd dnsmasq docker
   ```
3. **Verify Services**: Check all containers and services are running
4. **Update Network Config**: Ensure interfaces are correctly configured

## Troubleshooting

### Common Issues & Solutions

#### ⚠️ **Access Point Not Broadcasting**

**Symptoms**: Clients can't see `RaspberryPi-Travel` network

**Diagnosis:**
```bash
# Check hostapd status
sudo systemctl status hostapd

# Check USB adapter recognition
lsusb | grep -i wireless
iwconfig

# View hostapd logs
sudo journalctl -u hostapd -f
```

**Solutions:**
```bash
# Restart hostapd service
sudo systemctl restart hostapd

# Check hostapd configuration
sudo hostapd -d /etc/hostapd/hostapd.conf

# Ensure USB adapter is in AP mode
sudo iw dev wlan1 info
```

#### 🌐 **No Internet Access for Clients**

**Symptoms**: Clients connect to AP but can't reach internet

**Diagnosis:**
```bash
# Check internet connectivity on Pi
ping -c 3 8.8.8.8

# Check NAT rules
sudo iptables -t nat -L -v

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward

# Check wlan0 connection
iwconfig wlan0
```

**Solutions:**
```bash
# Restart networking
sudo systemctl restart dhcpcd

# Reapply iptables rules
sudo /etc/iptables-rules.sh

# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

#### 📋 **DNS Resolution Errors**

**Symptoms**: Websites won't load, DNS timeouts

**Diagnosis:**
```bash
# Check Pi-hole status
sudo docker logs pihole
sudo docker exec pihole pihole status

# Test DNS resolution
nslookup google.com 192.168.4.1
dig @192.168.4.1 google.com

# Check dnsmasq
sudo systemctl status dnsmasq
```

**Solutions:**
```bash
# Restart Pi-hole
sudo docker restart pihole

# Restart dnsmasq
sudo systemctl restart dnsmasq

# Flush Pi-hole cache
sudo docker exec pihole pihole flush
```

#### 🔒 **VPN Connection Failures**

**ProtonVPN WireGuard Issues:**
```bash
# Check WireGuard status
sudo wg show

# Test VPN connection
sudo wg-quick up protonvpn
ping -c 3 1.1.1.1

# Check routing
sudo ip route show
```

**Tailscale Issues:**
```bash
# Check Tailscale status
sudo tailscale status

# Restart Tailscale
sudo tailscale down && sudo tailscale up

# Check logs
sudo journalctl -u tailscaled -f
```

### Performance Issues

#### 🐌 **Slow Performance**

**Diagnosis:**
```bash
# Check system load
htop

# Check memory usage
free -h

# Check disk I/O
iostat -x 1

# Check temperature
vcgencmd measure_temp
```

**Solutions:**
- Ensure adequate cooling (add heatsink/fan)
- Use faster USB 3.0 storage
- Optimize Docker container resources
- Check for runaway processes

#### 📎 **High Memory Usage**

```bash
# Check swap usage
swapon -s

# Identify memory-hungry processes
sudo ps aux --sort=-%mem | head -10

# Restart high-usage containers
sudo docker restart pihole
```

### Log Files

**Important log locations:**
- **System logs**: `/var/log/syslog`
- **Backup logs**: `/var/log/backup.log`
- **hostapd logs**: `sudo journalctl -u hostapd`
- **dnsmasq logs**: `sudo journalctl -u dnsmasq`
- **Pi-hole logs**: `sudo docker logs pihole`
- **Docker logs**: `sudo journalctl -u docker`

## Security Considerations

### 🔒 Essential Security Steps

**⚠️ Before First Use:**

1. **Change Default Passwords**
   ```bash
   # Change Pi user password
   passwd
   
   # Update Pi-hole web password
   sudo docker exec pihole pihole -a -p NEW_PASSWORD
   
   # Update hostapd WiFi password in /etc/hostapd/hostapd.conf
   sudo nano /etc/hostapd/hostapd.conf
   # Change wpa_passphrase=ChangeThisPassword123
   ```

2. **Configure SSH Key Authentication**
   ```bash
   # Copy your public key to the Pi
   ssh-copy-id pi@192.168.4.1
   
   # Disable password authentication (optional)
   sudo nano /etc/ssh/sshd_config
   # Set: PasswordAuthentication no
   sudo systemctl restart ssh
   ```

3. **Update All Software**
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo docker-compose pull
   sudo tailscale update
   ```

### 🚪 Access Control

**Firewall Configuration:**
```bash
# Allow only necessary ports
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 53/udp comment 'DNS'
sudo ufw allow 80/tcp comment 'Pi-hole Web'
sudo ufw allow 51821/tcp comment 'WG-Easy'
```

**Pi-hole Security:**
- Enable query logging for monitoring
- Regularly update blocklists
- Monitor for unusual DNS queries
- Use strong admin password

### 🔑 VPN Security

**ProtonVPN:**
- Keep WireGuard private keys secure
- Rotate keys periodically
- Use NetShield (included in configs)
- Monitor for VPN disconnections

**Tailscale:**
- Enable MFA on Tailscale account
- Use ACLs to restrict access
- Regularly audit connected devices
- Enable key expiry

### 📶 Wi-Fi Security

**Access Point Security:**
- Use WPA2-PSK with strong passphrase (20+ characters)
- Disable WPS
- Hide SSID if desired (security through obscurity)
- Regularly change Wi-Fi password
- Monitor connected devices

**Recommended AP Configuration:**
```bash
# In /etc/hostapd/hostapd.conf
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
wpa_passphrase=YourVeryStrongPasswordHere123!
```

### 📊 Monitoring & Alerting

**Security Monitoring:**
```bash
# Monitor failed SSH attempts
sudo grep "Failed password" /var/log/auth.log

# Check for unusual network activity
sudo netstat -tuln

# Monitor Pi-hole query logs
sudo docker exec pihole tail -f /var/log/pihole.log
```

**Recommended Regular Tasks:**
- Weekly review of Pi-hole query logs
- Monthly security updates
- Quarterly password changes
- Monitor system resource usage
- Check backup integrity

### 🚑 Incident Response

**If Compromise is Suspected:**
1. Immediately disconnect from internet
2. Check system logs for suspicious activity
3. Change all passwords
4. Update all software
5. Restore from clean backup if necessary
6. Analyze logs to determine breach vector

## Credits & License

### 🚀 Open Source Components

This project builds upon excellent open-source software:

- **[Pi-hole](https://pi-hole.net/)** - Network-level advertisement and internet tracker blocking
- **[WG-Easy](https://github.com/wg-easy/wg-easy)** - Simple WireGuard VPN management interface  
- **[BlockListProject](https://github.com/blocklistproject/Lists)** - Comprehensive DNS blocklists
- **[Tailscale](https://tailscale.com/)** - Zero-config mesh VPN networking
- **[ProtonVPN](https://protonvpn.com/)** - Privacy-focused VPN service
- **[Docker](https://www.docker.com/)** & **[Portainer](https://portainer.io/)** - Container management
- **[Raspberry Pi Foundation](https://www.raspberrypi.org/)** - Amazing single-board computers

### 🙏 Acknowledgments

- **BlockListProject Team** - For maintaining comprehensive, categorized blocklists
- **Pi-hole Developers** - For creating the best network-level ad blocker
- **WG-Easy Contributors** - For simplifying WireGuard management
- **Raspberry Pi Community** - For countless tutorials and troubleshooting guides
- **r/raspberry_pi** & **r/pihole** - For community support and ideas

### 📝 License

**MIT License**

```
Copyright (c) 2025 CypherOxide

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

### ❤️ Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes, please open an issue first to discuss what you would like to change.

**Areas where contributions are especially welcome:**
- Additional tested Wi-Fi adapter compatibility
- Performance optimizations
- Additional VPN provider configurations
- Docker container updates
- Documentation improvements
- Automated setup scripts

### 💬 Support

- **Issues**: [GitHub Issues](https://github.com/cypheroxide/Pi4-TravelRouter/issues)
- **Discussions**: [GitHub Discussions](https://github.com/cypheroxide/Pi4-TravelRouter/discussions)
- **Documentation**: Check `src/rpi4-travel-router-plan.md` for detailed setup

---

**🇨🇭 Made with Swiss precision and ☕ caffeine by [CypherOxide](https://github.com/cypheroxide)**
