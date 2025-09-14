# Phase 3: Network Configuration & Access Point Setup

## Overview

This phase configures the dual Wi-Fi setup, creates the access point, and establishes NAT routing between interfaces.

**Estimated Time**: 45-60 minutes  
**Difficulty**: Intermediate  
**Prerequisites**: [Phase 2 Complete](02-docker-setup.md)

## Pre-Phase Checklist

Ensure Phase 2 is complete:

- [ ] Docker installed and running
- [ ] Portainer accessible
- [ ] USB Wi-Fi adapter connected
- [ ] Container management working

```bash
# Quick verification
docker ps | grep portainer && echo "✅ Ready for Phase 3"
```

## Step 1: Identify Network Interfaces

### Check Available Interfaces

```bash
# List all network interfaces
ip link show

# Check Wi-Fi interfaces specifically
iwconfig 2>/dev/null | grep -E "wlan|IEEE"

# Check USB devices
lsusb | grep -i wireless
```

**Expected output:**
- `wlan0`: Built-in Wi-Fi (disabled in boot config)
- `wlan1`: USB Wi-Fi adapter (for Access Point)
- `eth0`: Ethernet interface

### Verify USB Wi-Fi Adapter

```bash
# Check if USB adapter supports AP mode
iw list | grep -A 10 "Supported interface modes" | grep -i "ap"

# If no output, your adapter may not support AP mode
# Check our compatibility guide for alternatives
```

## Step 2: Install Network Packages

### Install Required Software

```bash
# Install hostapd (Access Point daemon) and dnsmasq (DHCP/DNS server)
sudo apt update
sudo apt install -y hostapd dnsmasq iptables-persistent

# Stop services initially for configuration
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq

# Prevent auto-start until configured
sudo systemctl disable hostapd
sudo systemctl disable dnsmasq
```

## Step 3: Configure Access Point (hostapd)

### Create hostapd Configuration

```bash
# Create hostapd configuration file
sudo tee /etc/hostapd/hostapd.conf > /dev/null << 'EOF'
# Pi4-TravelRouter Access Point Configuration

# Interface and driver
interface=wlan1
driver=nl80211

# Network settings
ssid=Pi4-TravelRouter
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0

# Security settings
wpa=2
wpa_passphrase=ChangeThisPassword123!
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

# Country code (change to your country)
country_code=US

# Additional settings
ieee80211n=1
ieee80211d=1
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
EOF

# Set proper permissions
sudo chmod 600 /etc/hostapd/hostapd.conf
```

### Configure hostapd Daemon

```bash
# Tell hostapd where to find config file
sudo tee /etc/default/hostapd > /dev/null << 'EOF'
# Location of hostapd configuration file
DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Additional daemon options
DAEMON_OPTS=""
EOF
```

## Step 4: Configure DHCP Server (dnsmasq)

### Backup and Configure dnsmasq

```bash
# Backup original configuration
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.backup

# Create new dnsmasq configuration
sudo tee /etc/dnsmasq.conf > /dev/null << 'EOF'
# Pi4-TravelRouter DHCP Configuration

# Interface to bind to
interface=wlan1

# DHCP pool configuration
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h

# Default gateway
dhcp-option=3,192.168.4.1

# DNS servers (will be updated when Pi-hole is configured)
dhcp-option=6,192.168.4.1

# Domain name
domain=pi4router.local

# Bogus private reverse lookups
bogus-priv

# Don't read /etc/resolv.conf
no-resolv

# Upstream DNS servers (temporary, will be replaced by Pi-hole)
server=1.1.1.1
server=8.8.8.8

# Local domain handling
local=/pi4router.local/
domain=pi4router.local

# Cache size
cache-size=1000

# Log queries (useful for debugging)
log-queries
log-facility=/var/log/dnsmasq.log
EOF
```

## Step 5: Configure Static IP for Access Point

### Configure dhcpcd

```bash
# Configure static IP for wlan1 interface
sudo tee -a /etc/dhcpcd.conf > /dev/null << 'EOF'

# Pi4-TravelRouter Access Point Configuration
interface wlan1
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
EOF
```

### Configure Network Interfaces (Alternative Method)

If dhcpcd doesn't work properly, use systemd-networkd:

```bash
# Create networkd configuration for wlan1
sudo tee /etc/systemd/network/10-wlan1.network > /dev/null << 'EOF'
[Match]
Name=wlan1

[Network]
Address=192.168.4.1/24
DHCPServer=yes
IPMasquerade=yes
IPForward=yes

[DHCPServer]
PoolOffset=10
PoolSize=20
EmitDNS=yes
DNS=192.168.4.1
EOF

# Enable systemd-networkd (if needed)
# sudo systemctl enable systemd-networkd
```

## Step 6: Configure NAT and Routing

### Create iptables Rules Script

```bash
# Create iptables configuration script
sudo tee /usr/local/bin/setup-routing.sh > /dev/null << 'EOF'
#!/bin/bash

# Pi4-TravelRouter Routing Configuration

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log() {
    echo -e "${GREEN}[$(date '+%H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
}

# Determine active internet interface
INTERNET_IFACE=""
for iface in wlan0 eth0; do
    if ip route | grep -q "default.*$iface"; then
        INTERNET_IFACE="$iface"
        break
    fi
done

if [ -z "$INTERNET_IFACE" ]; then
    error "No active internet interface found"
    exit 1
fi

log "Using internet interface: $INTERNET_IFACE"

# Clear existing rules
log "Clearing existing iptables rules"
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X

# Set default policies
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Basic INPUT rules
log "Setting up INPUT rules"
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -i wlan1 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# SSH access from AP network
iptables -A INPUT -i wlan1 -p tcp --dport 22 -j ACCEPT

# Web interfaces from AP network
iptables -A INPUT -i wlan1 -p tcp --dport 80 -j ACCEPT   # Pi-hole
iptables -A INPUT -i wlan1 -p tcp --dport 9000 -j ACCEPT # Portainer
iptables -A INPUT -i wlan1 -p tcp --dport 51821 -j ACCEPT # WG-Easy (future)

# DNS from AP network
iptables -A INPUT -i wlan1 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i wlan1 -p tcp --dport 53 -j ACCEPT

# DHCP from AP network
iptables -A INPUT -i wlan1 -p udp --dport 67 -j ACCEPT

# NAT rules for internet sharing
log "Setting up NAT rules"
iptables -t nat -A POSTROUTING -o "$INTERNET_IFACE" -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.4.0/24 -o "$INTERNET_IFACE" -j MASQUERADE

# Forward rules
log "Setting up FORWARD rules"
iptables -A FORWARD -o "$INTERNET_IFACE" -i wlan1 -s 192.168.4.0/24 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow forwarding between interfaces
iptables -A FORWARD -i wlan1 -o "$INTERNET_IFACE" -j ACCEPT
iptables -A FORWARD -i "$INTERNET_IFACE" -o wlan1 -j ACCEPT

# Save rules
log "Saving iptables rules"
iptables-save > /etc/iptables/rules.v4

log "Routing configuration completed successfully"
log "Internet interface: $INTERNET_IFACE"
log "AP network: 192.168.4.0/24"
log "AP gateway: 192.168.4.1"
EOF

# Make script executable
sudo chmod +x /usr/local/bin/setup-routing.sh
```

### Enable IP Forwarding

```bash
# Enable IP forwarding (should already be enabled from Phase 1)
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf

# Apply immediately
sudo sysctl -p

# Verify it's enabled
cat /proc/sys/net/ipv4/ip_forward  # Should output: 1
```

## Step 7: Create Wi-Fi Management Scripts

### Wi-Fi Connection Script

```bash
# Create Wi-Fi connection management script
sudo tee /usr/local/bin/wifi-connect.sh > /dev/null << 'EOF'
#!/bin/bash

# Pi4-TravelRouter Wi-Fi Connection Manager

SSID="$1"
PASSWORD="$2"
INTERFACE="wlan0"

show_usage() {
    echo "Usage: $0 <SSID> [PASSWORD]"
    echo ""
    echo "Examples:"
    echo "  $0 \"Hotel-WiFi\" \"password123\""
    echo "  $0 \"Open-Network\""
    echo ""
    echo "Current status:"
    iwconfig $INTERFACE 2>/dev/null | grep -E "ESSID|Access Point|Bit Rate"
}

if [ -z "$SSID" ]; then
    show_usage
    exit 1
fi

echo "Connecting to Wi-Fi network: $SSID"

# Stop AP temporarily to avoid conflicts
echo "Temporarily stopping access point..."
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq

# Create wpa_supplicant configuration
if [ -n "$PASSWORD" ]; then
    # Network with password
    sudo tee /etc/wpa_supplicant/wpa_supplicant.conf > /dev/null << EOF
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="$SSID"
    psk="$PASSWORD"
    priority=1
}
EOF
else
    # Open network
    sudo tee /etc/wpa_supplicant/wpa_supplicant.conf > /dev/null << EOF
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="$SSID"
    key_mgmt=NONE
    priority=1
}
EOF
fi

# Restart networking
echo "Restarting network interface..."
sudo wpa_cli -i $INTERFACE reconfigure
sleep 3

# Check connection
echo "Testing connection..."
for i in {1..10}; do
    if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
        echo "✅ Successfully connected to $SSID"
        
        # Update routing
        echo "Updating routing configuration..."
        sudo /usr/local/bin/setup-routing.sh
        
        # Restart services
        echo "Restarting access point services..."
        sudo systemctl start dnsmasq
        sudo systemctl start hostapd
        
        echo "Connection complete! Access point restored."
        exit 0
    fi
    echo "Attempt $i/10: Waiting for connection..."
    sleep 2
done

echo "❌ Failed to connect to $SSID"
echo "Restarting access point services..."
sudo systemctl start dnsmasq
sudo systemctl start hostapd

exit 1
EOF

# Make script executable
sudo chmod +x /usr/local/bin/wifi-connect.sh
```

### Network Status Script

```bash
# Create network status script
sudo tee /usr/local/bin/network-status.sh > /dev/null << 'EOF'
#!/bin/bash

# Pi4-TravelRouter Network Status

echo "=== Pi4-TravelRouter Network Status ==="
echo "$(date)"
echo ""

# Interface status
echo "📡 Network Interfaces:"
ip addr show | grep -E "^[0-9]+:|inet " | sed 's/^/  /'

echo ""
echo "🌐 Wi-Fi Status:"
for iface in wlan0 wlan1; do
    if ip link show "$iface" >/dev/null 2>&1; then
        echo "  $iface:"
        iwconfig "$iface" 2>/dev/null | grep -E "ESSID|Access Point|Bit Rate|Signal level" | sed 's/^/    /'
    fi
done

echo ""
echo "🔄 Routing Table:"
ip route | sed 's/^/  /'

echo ""
echo "📊 Active Connections:"
ss -tuln | grep -E ":(22|53|80|9000|51821)" | sed 's/^/  /'

echo ""
echo "🏠 DHCP Leases (Active Clients):"
if [ -f /var/lib/dhcp/dhcpd.leases ]; then
    awk '/lease/ { ip = $2 } /binding state active/ { print "  " ip }' /var/lib/dhcp/dhcpd.leases
elif [ -f /var/lib/dhcpcd5/dhcpcd.leases ]; then
    grep -E "^[0-9]" /var/lib/dhcpcd5/dhcpcd.leases | sed 's/^/  /'
else
    # Use ARP table as fallback
    arp -a | grep "192.168.4" | sed 's/^/  /'
fi

echo ""
echo "🔥 Services Status:"
for service in hostapd dnsmasq docker; do
    status=$(sudo systemctl is-active $service)
    if [ "$status" = "active" ]; then
        echo "  ✅ $service: $status"
    else
        echo "  ❌ $service: $status"
    fi
done

echo ""
echo "🌍 Internet Connectivity:"
if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
    echo "  ✅ Internet: Connected"
else
    echo "  ❌ Internet: Disconnected"
fi

if ping -c 1 google.com >/dev/null 2>&1; then
    echo "  ✅ DNS: Working"
else
    echo "  ❌ DNS: Not working"
fi
EOF

# Make script executable
sudo chmod +x /usr/local/bin/network-status.sh
```

## Step 8: Create Startup Service

### Create Travel Router Service

```bash
# Create systemd service for travel router
sudo tee /etc/systemd/system/pi4-travel-router.service > /dev/null << 'EOF'
[Unit]
Description=Pi4 Travel Router Network Setup
After=network-online.target
Wants=network-online.target
Before=hostapd.service dnsmasq.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-routing.sh
ExecStartPost=/bin/sleep 2
RemainAfterExit=yes
User=root

[Install]
WantedBy=multi-user.target
EOF

# Enable the service
sudo systemctl enable pi4-travel-router.service
```

## Step 9: Test Network Configuration

### Initial Interface Test

```bash
# Set static IP manually for testing
sudo ip addr add 192.168.4.1/24 dev wlan1
sudo ip link set wlan1 up

# Check if interface has IP
ip addr show wlan1 | grep "inet "
```

### Start Services in Test Mode

```bash
# Test hostapd configuration
sudo hostapd -d /etc/hostapd/hostapd.conf &
HOSTAPD_PID=$!

# Wait a few seconds
sleep 5

# Check if AP is broadcasting
iwlist scan | grep -A 2 -B 2 "Pi4-TravelRouter"

# Kill test hostapd
sudo kill $HOSTAPD_PID
```

### Full Service Test

```bash
# Start services
sudo systemctl start pi4-travel-router
sudo systemctl start dnsmasq
sudo systemctl start hostapd

# Check service status
sudo systemctl status pi4-travel-router
sudo systemctl status dnsmasq
sudo systemctl status hostapd

# Run network status check
network-status.sh
```

## Step 10: Validation & Testing

### Comprehensive Network Test

```bash
# Create network validation script
sudo tee /usr/local/bin/validate-network.sh > /dev/null << 'EOF'
#!/bin/bash

# Network Validation Script

echo "=== Pi4-TravelRouter Network Validation ==="

# Test 1: Interface Configuration
echo "🔧 Test 1: Interface Configuration"
if ip addr show wlan1 | grep -q "192.168.4.1"; then
    echo "  ✅ wlan1 has static IP 192.168.4.1"
else
    echo "  ❌ wlan1 missing static IP"
fi

# Test 2: Services Running
echo "🔧 Test 2: Service Status"
for service in hostapd dnsmasq pi4-travel-router; do
    if sudo systemctl is-active --quiet $service; then
        echo "  ✅ $service is running"
    else
        echo "  ❌ $service is not running"
    fi
done

# Test 3: AP Broadcasting
echo "🔧 Test 3: Access Point Broadcasting"
if iwlist wlan1 scan 2>/dev/null | grep -q "Pi4-TravelRouter"; then
    echo "  ✅ Access Point is broadcasting"
else
    echo "  ❌ Access Point not detected"
fi

# Test 4: DHCP Range
echo "🔧 Test 4: DHCP Configuration"
if grep -q "192.168.4.2,192.168.4.20" /etc/dnsmasq.conf; then
    echo "  ✅ DHCP range configured"
else
    echo "  ❌ DHCP range not configured"
fi

# Test 5: IP Forwarding
echo "🔧 Test 5: IP Forwarding"
if [ "$(cat /proc/sys/net/ipv4/ip_forward)" = "1" ]; then
    echo "  ✅ IP forwarding enabled"
else
    echo "  ❌ IP forwarding disabled"
fi

# Test 6: NAT Rules
echo "🔧 Test 6: NAT Rules"
if sudo iptables -t nat -L | grep -q "MASQUERADE"; then
    echo "  ✅ NAT rules present"
else
    echo "  ❌ NAT rules missing"
fi

# Test 7: Internet Connectivity
echo "🔧 Test 7: Internet Connectivity"
if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
    echo "  ✅ Internet connectivity working"
else
    echo "  ❌ No internet connectivity"
fi

echo ""
echo "Use 'network-status.sh' for detailed status information"
EOF

# Make script executable
sudo chmod +x /usr/local/bin/validate-network.sh

# Run validation
validate-network.sh
```

### Client Connection Test

After setting up a client device to connect to "Pi4-TravelRouter":

1. **Connect a device** (phone, laptop) to the "Pi4-TravelRouter" network
2. **Password**: Use the password from hostapd.conf (default: `ChangeThisPassword123!`)
3. **Verify DHCP**: Device should get IP in range 192.168.4.2-192.168.4.20
4. **Test internet**: Try browsing a website
5. **Test SSH**: `ssh pi@192.168.4.1`

## Troubleshooting Common Issues

### Access Point Not Broadcasting
```bash
# Check hostapd status and logs
sudo systemctl status hostapd
sudo journalctl -u hostapd -f

# Test hostapd configuration
sudo hostapd -d /etc/hostapd/hostapd.conf

# Check USB adapter
lsusb | grep -i wireless
iwconfig wlan1
```

### No Internet Access for Clients
```bash
# Check routing
ip route
iptables -t nat -L

# Reapply routing
sudo /usr/local/bin/setup-routing.sh

# Check internet connection on Pi
ping -c 3 8.8.8.8
```

### DHCP Not Working
```bash
# Check dnsmasq status
sudo systemctl status dnsmasq
sudo journalctl -u dnsmasq -f

# Check configuration
sudo dnsmasq --test
```

### Interface Configuration Issues
```bash
# Check interface status
ip link show wlan1
ip addr show wlan1

# Manually configure interface
sudo ip addr add 192.168.4.1/24 dev wlan1
sudo ip link set wlan1 up
```

## What's Next?

After completing this phase:

✅ **Completed Tasks:**
- USB Wi-Fi adapter configured as access point
- DHCP server providing IP addresses
- NAT routing between interfaces
- Internet sharing functional
- Management scripts created

➡️ **Next Phase:** [Service Installation](04-service-installation.md)

### Final Network Test

```bash
# Run comprehensive validation
validate-network.sh

# Check full network status
network-status.sh

# Test client connectivity (connect a device and verify internet access)
```

Your Pi4-TravelRouter should now be broadcasting a Wi-Fi network that provides internet access to connected devices. Before proceeding to install additional services, ensure all network tests pass.

If you encounter issues, check the [Network Troubleshooting Guide](../troubleshooting/network-issues.md).