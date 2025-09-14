# Configuration Reference Guide

## Overview

This section provides comprehensive configuration templates, examples, and reference information for all components of the Pi4-TravelRouter.

## Quick Navigation

| Component | Configuration File | Documentation | Status |
|-----------|-------------------|---------------|---------|
| **Docker Compose** | [docker-compose.md](docker-compose.md) | Complete configs for all services | ✅ Ready |
| **Pi-hole** | [pihole.md](pihole.md) | DNS filtering and blocklist setup | ✅ Ready |
| **Network Config** | [networking.md](networking.md) | hostapd, dnsmasq, and routing | ✅ Ready |
| **ProtonVPN** | [protonvpn.md](protonvpn.md) | WireGuard VPN configuration | ✅ Ready |
| **Tailscale** | [tailscale.md](tailscale.md) | Personal network VPN setup | 🚧 Coming Soon |
| **WG-Easy** | [wg-easy.md](wg-easy.md) | WireGuard web management | 🚧 Coming Soon |
| **System Config** | [system.md](system.md) | OS and performance settings | ✅ Ready |

## Configuration Templates

All configuration files use standardized paths and include:
- **Template files** with placeholder values
- **Production examples** with real-world settings
- **Customization options** with explanations
- **Security considerations** for each component

## File Locations

### Standard Directory Structure

```bash
# Main storage location
/mnt/storage/
├── configs/                    # User-customizable configs
├── docker/                     # Docker data and containers
│   ├── compose/               # Docker Compose files
│   ├── portainer/             # Portainer data
│   └── [service-name]/        # Individual service data
├── backups/                    # System and config backups
└── data/                       # User data and logs
```

### System Configuration Locations

```bash
# System network configuration
/etc/hostapd/hostapd.conf           # Access Point settings
/etc/dnsmasq.conf                   # DHCP/DNS server settings
/etc/dhcpcd.conf                    # Interface IP configuration
/etc/wpa_supplicant/                # Wi-Fi client settings

# Docker configuration
/etc/docker/daemon.json             # Docker daemon settings
/mnt/storage/docker/compose/        # Docker Compose files

# System optimization
/etc/sysctl.conf                    # Kernel parameters
/etc/fstab                          # Filesystem mounts
```

## Quick Start Templates

### 1. Copy Base Templates
```bash
# Copy configuration templates to your Pi
git clone https://github.com/cypheroxide/Pi4-TravelRouter.git
cd Pi4-TravelRouter
cp -r src/ /mnt/storage/configs/
```

### 2. Customize Settings
Edit the following key files for your setup:

| File | What to Change |
|------|----------------|
| `hostapd.conf` | Wi-Fi SSID and password |
| `docker-compose-pihole.yml` | Pi-hole admin password, timezone |
| `protonvpn.conf` | Your ProtonVPN credentials |
| System scripts | Timezone, locale settings |

### 3. Security Setup
⚠️ **Critical**: Change these default values:
- Access Point password in `hostapd.conf`
- Pi-hole web admin password
- SSH keys and user passwords
- VPN credentials

## Configuration Validation

Each configuration guide includes validation commands to test your setup:

```bash
# Network configuration test
validate-network.sh

# Docker services test
container-health.sh

# System health check
/usr/local/bin/backup-system.sh
```

## Environment Variables

Key environment variables used throughout the system:

```bash
# Standard paths
STORAGE_PATH="/mnt/storage"
DOCKER_ROOT="/mnt/storage/docker"
CONFIG_PATH="/mnt/storage/configs"

# Network settings
AP_SUBNET="192.168.4.0/24"
AP_GATEWAY="192.168.4.1"
AP_DHCP_RANGE="192.168.4.2,192.168.4.20"

# Service ports
PORTAINER_PORT="9000"
PIHOLE_WEB_PORT="80"
WGEASY_PORT="51821"
```

## Getting Help

- **Setup Issues**: See [Troubleshooting Guide](../troubleshooting/)
- **Security Questions**: See [Security Guide](../security/)
- **Customization**: Each config file includes detailed comments
- **Validation**: Each guide includes test procedures

## Next Steps

1. **Start with System Config**: Review [system.md](system.md) for OS-level settings
2. **Network Setup**: Configure networking with [networking.md](networking.md)
3. **Docker Services**: Set up containers with [docker-compose.md](docker-compose.md)
4. **Security Hardening**: Follow [Security Guide](../security/) recommendations

---

*All configuration examples use sanitized, production-ready templates with security best practices.*