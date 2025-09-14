# ProtonVPN Configuration Reference

## Overview

This guide covers ProtonVPN client configuration for the Pi4-TravelRouter, including OpenVPN setup, connection management, and routing configurations to route all traffic through ProtonVPN for privacy.

## Installation and Setup

### Install ProtonVPN Client

```bash
# Install required packages
sudo apt update
sudo apt install -y curl gnupg

# Add ProtonVPN repository
curl -L https://protonvpn.com/download/proton-vpn-gnulinux-stable-release_1.0.0-1_all.deb \
  -o /tmp/proton-vpn-gnulinux-stable-release_1.0.0-1_all.deb

sudo dpkg -i /tmp/proton-vpn-gnulinux-stable-release_1.0.0-1_all.deb
sudo apt update

# Install ProtonVPN CLI
sudo apt install -y protonvpn-cli
```

### Alternative: OpenVPN Configuration

If you prefer manual OpenVPN configuration:

```bash
# Install OpenVPN
sudo apt install -y openvpn openvpn-systemd-resolved

# Create ProtonVPN directory
sudo mkdir -p /etc/openvpn/protonvpn
cd /etc/openvpn/protonvpn
```

## ProtonVPN CLI Setup

### Initial Configuration

```bash
# Initialize ProtonVPN
sudo protonvpn-cli login

# Check status
protonvpn-cli status

# List available servers
protonvpn-cli servers

# Connect to fastest server
protonvpn-cli connect --fastest

# Connect to specific country
protonvpn-cli connect --country US

# Connect to specific server
protonvpn-cli connect US-FREE#1
```

### Connection Management

```bash
# Connect with specific protocol
protonvpn-cli connect --protocol tcp
protonvpn-cli connect --protocol udp

# Check connection status
protonvpn-cli status

# Disconnect
protonvpn-cli disconnect

# Reconnect
protonvpn-cli reconnect
```

## Manual OpenVPN Configuration

### Download Configuration Files

1. Log into your ProtonVPN account at https://account.protonvpn.com/
2. Navigate to Downloads → OpenVPN configuration files
3. Download the configuration files for your preferred servers

### OpenVPN Configuration

**File**: `/etc/openvpn/protonvpn/proton-server.conf`

```bash
# ProtonVPN OpenVPN Configuration
# Downloaded from ProtonVPN dashboard

client
dev tun
proto udp
remote SERVER_ADDRESS PORT
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert protonvpn.crt
key protonvpn.key
cipher AES-256-GCM
auth SHA512
comp-lzo
verb 3

# DNS settings
dhcp-option DNS 10.2.0.1
dhcp-option DNS 1.1.1.1

# Routing and security
redirect-gateway def1
remote-cert-tls server
auth-user-pass /etc/openvpn/protonvpn/credentials

# Scripts for network management
up /etc/openvpn/protonvpn/up.sh
down /etc/openvpn/protonvpn/down.sh
```

### Credentials File

**File**: `/etc/openvpn/protonvpn/credentials`

```
your-protonvpn-username
your-protonvpn-password
```

```bash
# Secure the credentials file
sudo chmod 600 /etc/openvpn/protonvpn/credentials
sudo chown root:root /etc/openvpn/protonvpn/credentials
```

### Network Scripts

#### Up Script

**File**: `/etc/openvpn/protonvpn/up.sh`

```bash
#!/bin/bash

# ProtonVPN connection up script

LOG_FILE="/var/log/protonvpn.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Log connection
log_message "ProtonVPN connected - Interface: $dev, Gateway: $route_vpn_gateway"

# Update DNS (using systemd-resolved)
if command -v resolvectl &> /dev/null; then
    resolvectl dns "$dev" 10.2.0.1 1.1.1.1
    resolvectl domain "$dev" "~."
fi

# Add custom routing rules if needed
# Route specific traffic through VPN
ip route add 192.168.4.0/24 via $route_vpn_gateway dev $dev

# Update iptables for VPN
iptables -t nat -A POSTROUTING -o $dev -j MASQUERADE
iptables -A FORWARD -i wlan1 -o $dev -j ACCEPT
iptables -A FORWARD -i $dev -o wlan1 -j ACCEPT

log_message "ProtonVPN routing configured"
```

#### Down Script

**File**: `/etc/openvpn/protonvpn/down.sh`

```bash
#!/bin/bash

# ProtonVPN connection down script

LOG_FILE="/var/log/protonvpn.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Log disconnection
log_message "ProtonVPN disconnected"

# Restore DNS
if command -v resolvectl &> /dev/null; then
    resolvectl revert "$dev"
fi

# Clean up iptables rules
iptables -t nat -D POSTROUTING -o $dev -j MASQUERADE 2>/dev/null
iptables -D FORWARD -i wlan1 -o $dev -j ACCEPT 2>/dev/null  
iptables -D FORWARD -i $dev -o wlan1 -j ACCEPT 2>/dev/null

log_message "ProtonVPN cleanup completed"
```

```bash
# Make scripts executable
sudo chmod +x /etc/openvpn/protonvpn/up.sh
sudo chmod +x /etc/openvpn/protonvpn/down.sh
```

## Router Integration

### Routing All Traffic Through ProtonVPN

#### Method 1: Route All Client Traffic

**File**: `/etc/iptables/rules-protonvpn.v4`

```bash
# ProtonVPN routing rules
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# Route all client traffic through ProtonVPN
-A POSTROUTING -s 192.168.4.0/24 -o tun+ -j MASQUERADE

# Fallback to WAN if VPN is down (optional)
# -A POSTROUTING -s 192.168.4.0/24 -o eth0 -j MASQUERADE
# -A POSTROUTING -s 192.168.4.0/24 -o wlan0 -j MASQUERADE

COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Allow VPN traffic
-A INPUT -i tun+ -j ACCEPT
-A FORWARD -i wlan1 -o tun+ -j ACCEPT
-A FORWARD -i tun+ -o wlan1 -j ACCEPT

# Block non-VPN traffic (kill switch)
-A FORWARD -i wlan1 -o eth0 -j REJECT
-A FORWARD -i wlan1 -o wlan0 -j REJECT

COMMIT
```

#### Method 2: Policy-Based Routing

Create routing tables for VPN and non-VPN traffic:

**File**: `/etc/iproute2/rt_tables`

```bash
# Add custom routing tables
100 vpn
101 novpn
```

**Policy routing script**:

**File**: `/usr/local/bin/setup-vpn-routing.sh`

```bash
#!/bin/bash

# Policy-based routing for ProtonVPN

# VPN interface (adjust as needed)
VPN_IF="tun0"
VPN_GW="10.2.0.1"  # ProtonVPN gateway

# WAN interfaces
WAN_IF="eth0"
WAN_GW=$(ip route | grep "default via" | grep "$WAN_IF" | awk '{print $3}')

# Clear existing rules
ip rule del table vpn 2>/dev/null
ip rule del table novpn 2>/dev/null
ip route flush table vpn
ip route flush table novpn

# Setup VPN table
ip route add default via "$VPN_GW" dev "$VPN_IF" table vpn
ip route add 192.168.4.0/24 dev wlan1 table vpn

# Setup non-VPN table
ip route add default via "$WAN_GW" dev "$WAN_IF" table novpn
ip route add 192.168.4.0/24 dev wlan1 table novpn

# Policy rules
# Route all client traffic through VPN by default
ip rule add from 192.168.4.0/24 table vpn priority 100

# Route specific clients through regular WAN (example)
# ip rule add from 192.168.4.10 table novpn priority 99

echo "VPN routing configured"
```

### Kill Switch Implementation

Prevent traffic leaks when VPN is disconnected:

**File**: `/usr/local/bin/vpn-killswitch.sh`

```bash
#!/bin/bash

# VPN Kill Switch for ProtonVPN

enable_killswitch() {
    # Block all internet traffic except through VPN
    iptables -A FORWARD -i wlan1 -o eth0 -j REJECT
    iptables -A FORWARD -i wlan1 -o wlan0 -j REJECT
    
    # Allow only VPN traffic
    iptables -I FORWARD -i wlan1 -o tun+ -j ACCEPT
    iptables -I FORWARD -i tun+ -o wlan1 -j ACCEPT
    
    echo "Kill switch enabled"
}

disable_killswitch() {
    # Remove kill switch rules
    iptables -D FORWARD -i wlan1 -o eth0 -j REJECT 2>/dev/null
    iptables -D FORWARD -i wlan1 -o wlan0 -j REJECT 2>/dev/null
    iptables -D FORWARD -i wlan1 -o tun+ -j ACCEPT 2>/dev/null
    iptables -D FORWARD -i tun+ -o wlan1 -j ACCEPT 2>/dev/null
    
    echo "Kill switch disabled"
}

case "$1" in
    enable)
        enable_killswitch
        ;;
    disable)
        disable_killswitch
        ;;
    *)
        echo "Usage: $0 {enable|disable}"
        exit 1
        ;;
esac
```

## Systemd Service Management

### ProtonVPN Auto-Connect Service

**File**: `/etc/systemd/system/protonvpn-autoconnect.service`

```ini
[Unit]
Description=ProtonVPN Auto Connect
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=0

[Service]
Type=forking
User=root
ExecStart=/usr/bin/protonvpn-cli connect --fastest
ExecStop=/usr/bin/protonvpn-cli disconnect
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

### OpenVPN Service (if using manual config)

```bash
# Enable and start OpenVPN service
sudo systemctl enable openvpn@proton-server.service
sudo systemctl start openvpn@proton-server.service

# Check status
sudo systemctl status openvpn@proton-server.service
```

### Connection Monitoring Service

**File**: `/etc/systemd/system/protonvpn-monitor.service`

```ini
[Unit]
Description=ProtonVPN Connection Monitor
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/protonvpn-monitor.sh
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

**File**: `/usr/local/bin/protonvpn-monitor.sh`

```bash
#!/bin/bash

# ProtonVPN connection monitor

LOG_FILE="/var/log/protonvpn-monitor.log"
CHECK_INTERVAL=60  # seconds

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

check_vpn_connection() {
    if protonvpn-cli status | grep -q "Connected"; then
        return 0
    else
        return 1
    fi
}

check_internet_via_vpn() {
    # Test connectivity through VPN
    if curl -s --max-time 10 --interface tun0 https://ifconfig.me > /dev/null; then
        return 0
    else
        return 1
    fi
}

main() {
    log_message "ProtonVPN monitor started"
    
    while true; do
        if check_vpn_connection; then
            if check_internet_via_vpn; then
                log_message "VPN connection healthy"
            else
                log_message "VPN connected but no internet access, reconnecting..."
                protonvpn-cli disconnect
                sleep 10
                protonvpn-cli connect --fastest
            fi
        else
            log_message "VPN disconnected, attempting to reconnect..."
            protonvpn-cli connect --fastest
        fi
        
        sleep $CHECK_INTERVAL
    done
}

# Start monitoring
main
```

## DNS Configuration with ProtonVPN

### Using ProtonVPN DNS

Configure Pi-hole to use ProtonVPN DNS when connected:

**File**: `/mnt/storage/docker/pihole/dnsmasq.d/04-protonvpn.conf`

```bash
# ProtonVPN DNS configuration

# Use ProtonVPN DNS when VPN is active
server=10.2.0.1
server=1.1.1.1

# Conditional forwarding for VPN
server=/protonvpn.com/10.2.0.1
```

### DNS Leak Protection

**File**: `/usr/local/bin/dns-leak-protection.sh`

```bash
#!/bin/bash

# DNS leak protection for ProtonVPN

enable_dns_protection() {
    # Block all DNS requests except through VPN interface
    iptables -I OUTPUT -p udp --dport 53 -o eth0 -j REJECT
    iptables -I OUTPUT -p udp --dport 53 -o wlan0 -j REJECT
    iptables -I OUTPUT -p tcp --dport 53 -o eth0 -j REJECT
    iptables -I OUTPUT -p tcp --dport 53 -o wlan0 -j REJECT
    
    # Allow DNS through VPN
    iptables -I OUTPUT -p udp --dport 53 -o tun+ -j ACCEPT
    iptables -I OUTPUT -p tcp --dport 53 -o tun+ -j ACCEPT
    
    echo "DNS leak protection enabled"
}

disable_dns_protection() {
    # Remove DNS blocking rules
    iptables -D OUTPUT -p udp --dport 53 -o eth0 -j REJECT 2>/dev/null
    iptables -D OUTPUT -p udp --dport 53 -o wlan0 -j REJECT 2>/dev/null
    iptables -D OUTPUT -p tcp --dport 53 -o eth0 -j REJECT 2>/dev/null
    iptables -D OUTPUT -p tcp --dport 53 -o wlan0 -j REJECT 2>/dev/null
    iptables -D OUTPUT -p udp --dport 53 -o tun+ -j ACCEPT 2>/dev/null
    iptables -D OUTPUT -p tcp --dport 53 -o tun+ -j ACCEPT 2>/dev/null
    
    echo "DNS leak protection disabled"
}

case "$1" in
    enable)
        enable_dns_protection
        ;;
    disable)
        disable_dns_protection
        ;;
    *)
        echo "Usage: $0 {enable|disable}"
        exit 1
        ;;
esac
```

## Performance Optimization

### ProtonVPN Settings

```bash
# Configure ProtonVPN for better performance
protonvpn-cli configure

# Choose options:
# - Protocol: UDP (faster) or TCP (more reliable)
# - Default server: Fastest server
# - Kill switch: Enable
# - DNS management: Enable
# - Split tunneling: Disable (for full VPN routing)
```

### OpenVPN Performance Tuning

**File**: `/etc/openvpn/protonvpn/performance.conf`

```bash
# Performance optimizations
sndbuf 393216
rcvbuf 393216
push "sndbuf 393216"
push "rcvbuf 393216"

# Enable fast-io
fast-io

# Disable compression if not needed (can improve performance)
# compress

# Use hardware crypto if available
engine auto

# Reduce verbosity
verb 1
```

## Monitoring and Logging

### Connection Logs

```bash
# View ProtonVPN CLI logs
journalctl -u protonvpn-autoconnect -f

# View OpenVPN logs
journalctl -u openvpn@proton-server -f

# Check connection status
protonvpn-cli status
```

### IP Address Verification

```bash
# Check current public IP
curl -s https://ifconfig.me

# Check IP through VPN interface
curl -s --interface tun0 https://ifconfig.me

# Verify ProtonVPN connection
curl -s https://www.whatismyipaddress.com/
```

### Network Testing

```bash
# Test DNS leaks
dig +short myip.opendns.com @resolver1.opendns.com
dig +short o-o.myaddr.l.google.com @ns1.google.com

# Test WebRTC leaks (using curl with specific sites)
curl -s https://browserleaks.com/webrtc

# Speed test through VPN
curl -s https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py | python3
```

## Troubleshooting

### Common Issues

#### 1. VPN Won't Connect

```bash
# Check ProtonVPN service
systemctl status protonvpn-autoconnect

# Try manual connection
protonvpn-cli disconnect
protonvpn-cli connect --fastest

# Check logs
journalctl -u protonvpn-autoconnect --no-pager -l

# Verify credentials
protonvpn-cli login
```

#### 2. No Internet After VPN Connection

```bash
# Check routing
ip route show
ip route show table vpn

# Check DNS
nslookup google.com
resolvectl status

# Test connectivity
ping -c 3 8.8.8.8
curl -I https://google.com
```

#### 3. DNS Leaks

```bash
# Check current DNS servers
resolvectl status

# Verify DNS routing
iptables -L OUTPUT -n -v | grep :53

# Test DNS resolution
dig google.com
dig @10.2.0.1 google.com
```

#### 4. Performance Issues

```bash
# Check VPN server load
protonvpn-cli servers

# Try different servers
protonvpn-cli disconnect
protonvpn-cli connect US-FREE#2

# Check for packet loss
ping -c 10 google.com
mtr google.com
```

### Diagnostic Commands

```bash
# ProtonVPN status and debugging
protonvpn-cli status
protonvpn-cli servers
protonvpn-cli settings

# Network interface information
ip addr show tun0
ip route show dev tun0

# Firewall rules
iptables -L -n -v
iptables -t nat -L -n -v

# Process monitoring
ps aux | grep protonvpn
ps aux | grep openvpn
```

## Security Considerations

### VPN Security Best Practices

1. **Enable Kill Switch**: Always use kill switch to prevent leaks
2. **DNS Protection**: Block DNS leaks through firewall rules
3. **Regular Updates**: Keep ProtonVPN client updated
4. **Server Selection**: Choose servers based on security needs
5. **Monitor Connections**: Regularly check for connection drops
6. **Backup Connectivity**: Plan for VPN failures

### Advanced Security Features

#### Port Forwarding (ProtonVPN Plus)

```bash
# Enable port forwarding (Plus plan required)
protonvpn-cli connect --protocol tcp --port-forwarding
```

#### Secure Core Servers

```bash
# Connect to Secure Core servers (higher security)
protonvpn-cli connect --secure-core
```

## Integration with Other Services

### Docker Service Integration

Ensure Docker containers can access the internet through VPN:

```yaml
# Add to docker-compose.yml
networks:
  default:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-vpn
```

### WireGuard Compatibility

If running both ProtonVPN and WireGuard:

```bash
# Adjust routing to avoid conflicts
ip route add 10.8.0.0/24 via 192.168.4.1 dev wlan1
```

## Next Steps

- **Network Configuration**: See [networking.md](networking.md) for system networking
- **Security Hardening**: See [Security Guide](../security/) for additional protection
- **Monitoring Setup**: See [Troubleshooting Guide](../troubleshooting/) for monitoring tools