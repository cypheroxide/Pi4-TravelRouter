# Pi-hole DNS Configuration Reference

## Overview

This guide covers Pi-hole DNS server configuration for the Pi4-TravelRouter, including blocklist management, custom DNS settings, DHCP configuration, and advanced features.

## Basic Pi-hole Setup

### Initial Configuration

Access Pi-hole admin interface at: `http://192.168.4.1/admin`

**Default Login:**
- Username: `admin`
- Password: Set during installation or via environment variable

### Core DNS Settings

**Location**: Settings → DNS

#### Upstream DNS Servers

**Recommended Configuration:**
```
Primary DNS Servers:
- Cloudflare: 1.1.1.1, 1.0.0.1
- Google: 8.8.8.8, 8.8.4.4

Alternative Options:
- Quad9: 9.9.9.9, 149.112.112.112
- OpenDNS: 208.67.222.222, 208.67.220.220
```

**Advanced DNS Settings:**
- ✅ Enable DNSSEC
- ✅ Never forward non-FQDNs
- ✅ Never forward reverse lookups for private IP ranges
- ✅ Use DNSSEC for validation

### Interface Configuration

**Location**: Settings → DNS → Interface settings

```
Listen on: All interfaces
Permit all origins
```

**Security Note**: Only use "All interfaces" if Pi is on isolated network

## Blocklist Configuration

### Default Blocklists

Pi-hole comes with these default lists:
```
- StevenBlack's Unified List
- MalwareDomainList.com Hosts List
```

### Recommended Additional Blocklists

**Location**: Group Management → Adlists

#### Essential Lists
```bash
# Malware & Phishing
https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20of%20existing%20lists/AntiMalwareList.txt
https://phishing.army/download/phishing_army_blocklist_extended.txt

# Privacy & Tracking
https://www.github.developerdan.com/hosts/lists/amp-hosts-extended.txt
https://www.github.developerdan.com/hosts/lists/ads-and-tracking-extended.txt

# Social Media & Distractions
https://www.github.developerdan.com/hosts/lists/facebook-extended.txt
https://raw.githubusercontent.com/jmdugan/blocklists/master/corporations/pinterest/all

# Smart TV & IoT
https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV.txt
https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/AmazonFireTV.txt
```

#### Regional Lists (Optional)
```bash
# EasyList regions - choose your region
https://easylist-downloads.adblockplus.org/easylistdutch.txt  # Netherlands
https://easylist-downloads.adblockplus.org/easylistgermany.txt # Germany
https://easylist-downloads.adblockplus.org/liste_fr.txt       # France
```

### Custom Blocklist Management

#### Automatic Blocklist Updater Script

**File**: `/mnt/storage/scripts/update-blocklists.sh`

```bash
#!/bin/bash

# Pi-hole Blocklist Updater
# Updates custom blocklists and refreshes Pi-hole

BLOCKLIST_DIR="/mnt/storage/configs/Lists"
LOG_FILE="/var/log/blocklist-update.log"
PIHOLE_CONTAINER="pihole"

# Create directories
mkdir -p "$BLOCKLIST_DIR"

# Function to download and process blocklist
download_blocklist() {
    local url="$1"
    local filename="$2"
    local temp_file="/tmp/${filename}.tmp"
    
    echo "$(date): Downloading $filename from $url" >> "$LOG_FILE"
    
    if curl -s -L "$url" -o "$temp_file"; then
        # Remove comments and blank lines
        grep -v '^#' "$temp_file" | grep -v '^$' > "$BLOCKLIST_DIR/$filename"
        rm "$temp_file"
        echo "$(date): Successfully updated $filename" >> "$LOG_FILE"
    else
        echo "$(date): Failed to download $filename" >> "$LOG_FILE"
    fi
}

# Update custom lists
download_blocklist "https://someonewhocares.org/hosts/zero/hosts" "someonewhocares.txt"
download_blocklist "https://raw.githubusercontent.com/AdguardTeam/AdguardFilters/master/BaseFilter/sections/adservers.txt" "adguard-base.txt"
download_blocklist "https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts&showintro=0&mimetype=plaintext" "yoyo.txt"

# Restart pihole to reload lists
echo "$(date): Restarting Pi-hole to reload blocklists" >> "$LOG_FILE"
docker restart "$PIHOLE_CONTAINER"

echo "$(date): Blocklist update completed" >> "$LOG_FILE"
```

**Make executable and create cron job:**
```bash
chmod +x /mnt/storage/scripts/update-blocklists.sh

# Add to crontab (runs daily at 3 AM)
echo "0 3 * * * /mnt/storage/scripts/update-blocklists.sh" | sudo crontab -
```

## Custom DNS Configuration

### Local Domain Resolution

**File**: `/mnt/storage/docker/pihole/etc/custom.list`

```bash
# Pi4-TravelRouter Local DNS
192.168.4.1    pi4router.local
192.168.4.1    pihole.local
192.168.4.1    router.local
192.168.4.1    wg.local
192.168.4.1    admin.local

# Add your local devices
192.168.4.10   laptop.local
192.168.4.20   phone.local
192.168.4.30   tablet.local
```

### DNS Records via dnsmasq

**File**: `/mnt/storage/docker/pihole/dnsmasq.d/02-custom.conf`

```bash
# Custom DNS configuration for Pi4-TravelRouter

# Local domain
local=/local/
domain=local
expand-hosts

# DHCP Options
dhcp-option=option:domain-name,local
dhcp-option=option:domain-search,local

# Custom CNAMEs
cname=admin.local,pi4router.local
cname=router.local,pi4router.local
cname=pihole.local,pi4router.local

# DNS over HTTPS (DoH) blocking
address=/dns.google.com/
address=/cloudflare-dns.com/
address=/dns.quad9.net/

# Block Windows 10 telemetry
address=/vortex.data.microsoft.com/
address=/vortex-win.data.microsoft.com/
address=/telecommand.telemetry.microsoft.com/
address=/oca.telemetry.microsoft.com/
address=/sqm.telemetry.microsoft.com/
address=/watson.telemetry.microsoft.com/
```

## DHCP Configuration

### Enable Pi-hole DHCP

**Location**: Settings → DHCP

**Configuration:**
```
DHCP Enabled: Yes
Range: 192.168.4.100 - 192.168.4.200
Router (Gateway): 192.168.4.1
DHCP Lease Time: 24 hours
Domain Name: local
```

**Advanced DHCP Options:**
- ✅ Enable IPv6 support (SLAAC + RA)
- ✅ Rapid Commit (fast DHCP)
- ✅ DHCP Authoritative

### Static DHCP Leases

**Location**: Group Management → DHCP Leases

**Example Static Assignments:**
```
MAC Address         | IP Address    | Hostname
AA:BB:CC:DD:EE:01  | 192.168.4.10  | laptop
AA:BB:CC:DD:EE:02  | 192.168.4.20  | phone
AA:BB:CC:DD:EE:03  | 192.168.4.30  | tablet
```

## Advanced Configuration

### Query Logging

**Location**: Settings → Privacy

**Recommended Settings:**
```
Privacy Level: Show everything
Query Log: Enable
Log Queries: Enable (for troubleshooting)
Anonymize Logs: Consider enabling for privacy
```

**Log Retention:**
```bash
# Rotate Pi-hole logs weekly
# Add to /etc/logrotate.d/pihole
/var/log/pihole.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
}
```

### Performance Optimization

#### Memory Usage Settings

**File**: `/mnt/storage/docker/pihole/etc/pihole-FTL.conf`

```bash
# Performance tuning for Raspberry Pi 4
MAXDBDAYS=7
RESOLVE_IPV6=yes
RESOLVE_IPV4=yes
PIHOLE_PTR=PI4ROUTER
CACHE_SIZE=10000
MAXLOGAGE=24.0
```

#### DNS Cache Settings

```bash
# Increase DNS cache size for better performance
cache-size=10000

# Enable query logging for debugging
log-queries

# Set maximum TTL for cached records
max-cache-ttl=3600
max-ttl=3600

# Enable DNSSEC validation
dnssec
trust-anchor=.,20326,8,2,E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D
```

### Backup and Restore

#### Backup Pi-hole Configuration

```bash
#!/bin/bash
# Backup Pi-hole settings

BACKUP_DIR="/mnt/storage/backups/pihole"
DATE=$(date +%F)

mkdir -p "$BACKUP_DIR"

# Backup Pi-hole configuration
docker exec pihole pihole -a -t "$BACKUP_DIR/pihole-backup-$DATE.tar.gz"

# Backup custom configurations
tar -czf "$BACKUP_DIR/pihole-custom-$DATE.tar.gz" \
  /mnt/storage/docker/pihole/etc/custom.list \
  /mnt/storage/docker/pihole/dnsmasq.d/

echo "Pi-hole backup completed: $BACKUP_DIR"
```

#### Restore Pi-hole Configuration

```bash
#!/bin/bash
# Restore Pi-hole from backup

BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz>"
    exit 1
fi

# Stop Pi-hole container
docker stop pihole

# Restore backup
docker run --rm -it \
  -v pihole_etc:/etc/pihole \
  -v "$PWD/$BACKUP_FILE:/backup.tar.gz" \
  pihole/pihole:latest \
  bash -c "cd / && tar -xzf backup.tar.gz"

# Start Pi-hole container
docker start pihole

echo "Pi-hole restore completed"
```

## Monitoring and Maintenance

### Health Checks

```bash
#!/bin/bash
# Pi-hole health check script

echo "=== Pi-hole Health Check ==="

# Check if Pi-hole is running
if docker ps | grep -q pihole; then
    echo "✓ Pi-hole container is running"
else
    echo "✗ Pi-hole container is not running"
fi

# Test DNS resolution
if dig @192.168.4.1 google.com +short > /dev/null; then
    echo "✓ DNS resolution working"
else
    echo "✗ DNS resolution failed"
fi

# Check blocklist count
BLOCKLIST_COUNT=$(docker exec pihole pihole -c -j | jq '.domains_being_blocked')
echo "📊 Blocking $BLOCKLIST_COUNT domains"

# Check query count (last 24h)
QUERY_COUNT=$(docker exec pihole pihole -c -j | jq '.dns_queries_today')
echo "📊 $QUERY_COUNT DNS queries today"

# Check blocked percentage
BLOCKED_PCT=$(docker exec pihole pihole -c -j | jq '.ads_percentage_today')
echo "📊 ${BLOCKED_PCT}% queries blocked today"
```

### Log Analysis

```bash
# View recent blocked queries
docker exec pihole tail -f /var/log/pihole.log | grep "blocked"

# Check top blocked domains
docker exec pihole pihole -t 10 blocked

# Check top queried domains
docker exec pihole pihole -t 10 queries

# View client activity
docker exec pihole pihole -c
```

## Troubleshooting

### Common Issues

#### DNS Not Working

```bash
# Check if Pi-hole is listening on port 53
sudo netstat -tulpn | grep :53

# Test DNS resolution
dig @192.168.4.1 google.com
nslookup google.com 192.168.4.1

# Check Pi-hole logs
docker logs pihole | tail -20
```

#### Web Interface Not Accessible

```bash
# Check if web server is running
curl -I http://192.168.4.1/admin

# Restart Pi-hole container
docker restart pihole

# Check container logs
docker logs pihole
```

#### Slow DNS Resolution

```bash
# Increase cache size in FTL config
echo "CACHE_SIZE=10000" >> /mnt/storage/docker/pihole/etc/pihole-FTL.conf

# Restart Pi-hole
docker restart pihole

# Test resolution speed
time dig @192.168.4.1 google.com
```

### Performance Monitoring

```bash
# Monitor Pi-hole performance
watch -n 5 'docker exec pihole pihole -c -j | jq ".dns_queries_today, .ads_percentage_today, .domains_being_blocked"'

# Check memory usage
docker stats pihole

# Monitor query response times
tail -f /var/log/pihole.log | grep "reply"
```

## Security Best Practices

1. **Change Default Password**: Always set a strong admin password
2. **Regular Updates**: Keep Pi-hole container updated
3. **Backup Configuration**: Regular automated backups
4. **Monitor Logs**: Check for unusual query patterns
5. **Limit Access**: Restrict web interface access to local network
6. **DNSSEC**: Enable DNSSEC validation for security

## Integration with Other Services

### Tailscale Integration

Allow Pi-hole access via Tailscale network:

```bash
# Add Tailscale subnet to allowed networks
echo "interface=tailscale0" >> /mnt/storage/docker/pihole/dnsmasq.d/03-tailscale.conf
```

### WireGuard Integration

Configure clients to use Pi-hole DNS:

```bash
# In WG-Easy client configuration
WG_DEFAULT_DNS=192.168.4.1
```

## Next Steps

- **Network Configuration**: See [networking.md](networking.md) for system networking
- **Docker Compose**: See [docker-compose.md](docker-compose.md) for container orchestration  
- **Security Hardening**: See [Security Guide](../security/) for additional protection