# Docker Compose Configuration Reference

## Overview

This guide covers all Docker Compose configurations for the Pi4-TravelRouter, including Pi-hole, WG-Easy, and supporting services.

## Standard Docker Configuration

### Docker Daemon Configuration

**File**: `/etc/docker/daemon.json`

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
  "default-address-pools": [
    {
      "base": "172.20.0.0/16",
      "size": 24
    }
  ]
}
```

**Customization Options:**
- `max-size`: Adjust log file size (default: 10m)
- `max-file`: Number of log files to keep (default: 3)
- `data-root`: Change if using different storage location

## Pi-hole Configuration

### Basic Pi-hole Setup

**File**: `/mnt/storage/docker/compose/docker-compose-pihole.yml`

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
      # REQUIRED: Change these values
      TZ: 'America/Chicago'
      WEBPASSWORD: 'CHANGE_THIS_PASSWORD'
      
      # Network configuration
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
      PIHOLE_DNS_: '1.1.1.1;8.8.8.8'
      
      # Optional: Additional settings
      DNSMASQ_LISTENING: 'all'
      PIHOLE_DOMAIN: 'pi4router.local'
      VIRTUAL_HOST: '192.168.4.1'
      
      # Performance settings
      DNS_BOGUS_PRIV: 'true'
      DNS_FQDN_REQUIRED: 'true'
      REV_SERVER: 'false'
    volumes:
      - '/mnt/storage/docker/pihole/etc:/etc/pihole'
      - '/mnt/storage/docker/pihole/dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
    networks:
      - pi-router-net

networks:
  pi-router-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Pi-hole with Custom Blocklists

**File**: `/mnt/storage/docker/compose/docker-compose-pihole-advanced.yml`

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
      WEBPASSWORD: 'CHANGE_THIS_PASSWORD'
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
      
      # Enhanced DNS settings
      PIHOLE_DNS_: '1.1.1.1;8.8.8.8;1.0.0.1;8.8.4.4'
      DNS_BOGUS_PRIV: 'true'
      DNS_FQDN_REQUIRED: 'true'
      
      # Blocklist configuration
      WEBTHEME: 'default-dark'
      TEMPERATUREUNIT: 'c'
      
    volumes:
      - '/mnt/storage/docker/pihole/etc:/etc/pihole'
      - '/mnt/storage/docker/pihole/dnsmasq.d:/etc/dnsmasq.d'
      - '/mnt/storage/configs/Lists:/opt/blocklists:ro'
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
    networks:
      pi-router-net:
        ipv4_address: 172.20.0.10

  # Blocklist updater service
  blocklist-updater:
    container_name: blocklist-updater
    image: alpine:latest
    restart: unless-stopped
    volumes:
      - '/mnt/storage/configs/Lists:/blocklists'
      - './scripts/update-blocklists.sh:/usr/local/bin/update-blocklists.sh'
    command: |
      sh -c "
      apk add --no-cache curl
      chmod +x /usr/local/bin/update-blocklists.sh
      while true; do
        /usr/local/bin/update-blocklists.sh
        sleep 86400  # Update daily
      done"
    networks:
      - pi-router-net

networks:
  pi-router-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

## WG-Easy Configuration

### Basic WG-Easy Setup

**File**: `/mnt/storage/docker/compose/docker-compose-wg-easy.yml`

```yaml
version: '3.8'

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    restart: unless-stopped
    environment:
      # REQUIRED: Change these values
      PASSWORD: 'CHANGE_THIS_PASSWORD'
      WG_HOST: '192.168.4.1'  # Or your external IP/domain
      
      # Network configuration
      WG_PORT: '51820'
      WG_DEFAULT_ADDRESS: '10.8.0.x'
      WG_DEFAULT_DNS: '192.168.4.1'
      WG_ALLOWED_IPS: '192.168.4.0/24,10.8.0.0/24'
      
      # Optional settings
      WG_MTU: '1420'
      WG_PERSISTENT_KEEPALIVE: '25'
      WG_PRE_UP: ''
      WG_POST_UP: 'iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o wlan+ -j MASQUERADE'
      WG_PRE_DOWN: ''
      WG_POST_DOWN: 'iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o wlan+ -j MASQUERADE'
      
    volumes:
      - '/mnt/storage/docker/wg-easy:/etc/wireguard'
    ports:
      - '51820:51820/udp'
      - '51821:51821/tcp'
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    networks:
      pi-router-net:
        ipv4_address: 172.20.0.20

networks:
  pi-router-net:
    external: true
```

## Complete Multi-Service Stack

### Full Pi4-TravelRouter Stack

**File**: `/mnt/storage/docker/compose/docker-compose-full.yml`

```yaml
version: '3.8'

services:
  # Pi-hole DNS and ad blocking
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
    environment:
      TZ: '${TIMEZONE:-America/Chicago}'
      WEBPASSWORD: '${PIHOLE_PASSWORD:-CHANGE_THIS_PASSWORD}'
      FTLCONF_LOCAL_IPV4: '192.168.4.1'
      PIHOLE_DNS_: '1.1.1.1;8.8.8.8'
    volumes:
      - pihole-etc:/etc/pihole
      - pihole-dnsmasq:/etc/dnsmasq.d
    networks:
      pi-router-net:
        ipv4_address: 172.20.0.10
    healthcheck:
      test: ["CMD", "dig", "@127.0.0.1", "google.com"]
      interval: 30s
      timeout: 10s
      retries: 3

  # WireGuard Easy management
  wg-easy:
    container_name: wg-easy
    image: ghcr.io/wg-easy/wg-easy:15
    restart: unless-stopped
    environment:
      PASSWORD: '${WGEASY_PASSWORD:-CHANGE_THIS_PASSWORD}'
      WG_HOST: '${WG_HOST:-192.168.4.1}'
      WG_DEFAULT_DNS: '192.168.4.1'
      WG_MTU: '1420'
    volumes:
      - wg-easy-data:/etc/wireguard
    ports:
      - '51820:51820/udp'
      - '51821:51821/tcp'
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    networks:
      pi-router-net:
        ipv4_address: 172.20.0.20

  # Portainer container management
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports:
      - '9000:9000'
      - '9443:9443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    networks:
      - pi-router-net

  # System monitoring (optional)
  netdata:
    container_name: netdata
    image: netdata/netdata:latest
    restart: unless-stopped
    ports:
      - '19999:19999'
    volumes:
      - netdata-config:/etc/netdata
      - netdata-lib:/var/lib/netdata
      - netdata-cache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
    cap_add:
      - SYS_PTRACE
    security_opt:
      - apparmor:unconfined
    networks:
      - pi-router-net
    environment:
      - NETDATA_CLAIM_TOKEN=${NETDATA_CLAIM_TOKEN:-}
      - NETDATA_CLAIM_URL=https://app.netdata.cloud

volumes:
  pihole-etc:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/storage/docker/pihole/etc
  pihole-dnsmasq:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/storage/docker/pihole/dnsmasq.d
  wg-easy-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/storage/docker/wg-easy
  portainer-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/storage/docker/portainer
  netdata-config:
  netdata-lib:
  netdata-cache:

networks:
  pi-router-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Environment Variables File

**File**: `/mnt/storage/docker/compose/.env`

```bash
# Pi4-TravelRouter Environment Variables
# Copy this file and customize for your setup

# Timezone (use your local timezone)
TIMEZONE=America/Chicago

# Service Passwords (CHANGE THESE!)
PIHOLE_PASSWORD=your-secure-pihole-password-here
WGEASY_PASSWORD=your-secure-wgeasy-password-here

# WireGuard Configuration
WG_HOST=192.168.4.1
WG_PORT=51820

# Optional: Netdata monitoring
NETDATA_CLAIM_TOKEN=

# Network Configuration
AP_SUBNET=192.168.4.0/24
DOCKER_SUBNET=172.20.0.0/16
```

## Management Commands

### Deployment Commands

```bash
# Deploy Pi-hole only
cd /mnt/storage/docker/compose
docker compose -f docker-compose-pihole.yml up -d

# Deploy WG-Easy only
docker compose -f docker-compose-wg-easy.yml up -d

# Deploy complete stack
docker compose -f docker-compose-full.yml up -d

# Update all services
docker compose -f docker-compose-full.yml pull
docker compose -f docker-compose-full.yml up -d --force-recreate

# Stop all services
docker compose -f docker-compose-full.yml down
```

### Backup and Restore

```bash
# Backup all container data
tar -czf /mnt/storage/backups/docker-volumes-$(date +%F).tar.gz \
  -C /mnt/storage/docker .

# Restore container data (stop containers first)
docker compose -f docker-compose-full.yml down
tar -xzf /mnt/storage/backups/docker-volumes-YYYY-MM-DD.tar.gz \
  -C /mnt/storage/docker
docker compose -f docker-compose-full.yml up -d
```

## Troubleshooting

### Common Issues

1. **Permission Denied Errors**
   ```bash
   sudo chown -R 999:999 /mnt/storage/docker/pihole
   sudo chown -R $(whoami):$(whoami) /mnt/storage/docker/wg-easy
   ```

2. **Port Conflicts**
   ```bash
   # Check what's using ports
   sudo netstat -tulpn | grep -E ':(53|80|9000|51820|51821)'
   
   # Stop conflicting services
   sudo systemctl stop systemd-resolved  # If using port 53
   ```

3. **Network Issues**
   ```bash
   # Recreate Docker network
   docker network rm pi-router-net
   docker network create pi-router-net --driver bridge --subnet=172.20.0.0/16
   ```

### Health Checks

```bash
# Check all services
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check logs
docker logs pihole
docker logs wg-easy
docker logs portainer

# Test DNS resolution
dig @192.168.4.1 google.com
```

## Security Considerations

1. **Change Default Passwords**: Always modify default passwords in environment files
2. **Network Isolation**: Keep services on isolated Docker network
3. **Volume Permissions**: Set appropriate permissions for volume mounts
4. **Regular Updates**: Keep container images updated
5. **Backup Strategy**: Implement regular backups of container data

## Next Steps

- **Pi-hole Configuration**: See [pihole.md](pihole.md) for detailed DNS setup
- **Network Configuration**: See [networking.md](networking.md) for system-level networking
- **Security Hardening**: See [Security Guide](../security/) for additional protection