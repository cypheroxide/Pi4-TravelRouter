# Pi4-TravelRouter

> 🚀 **Portable Raspberry Pi 4 Travel Router** - A secure, dockerized travel router with built-in ad-blocking, VPN, and wireless AP capabilities

![GitHub last commit](https://img.shields.io/github/last-commit/cypheroxide/Pi4-TravelRouter?style=flat-square)
![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=flat-square&logo=docker&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/-Raspberry_Pi-C51A4A?style=flat-square&logo=Raspberry-Pi)
![WireGuard](https://img.shields.io/badge/wireguard-%2388171A.svg?style=flat-square&logo=wireguard&logoColor=white)

## Overview

Transform a Raspberry Pi 4 into a secure travel router that creates your own private wireless network, protecting your devices from untrusted public Wi-Fi. Features include network-level ad-blocking, VPN integration, and web-based management interfaces.

**Key Benefits:**
- 🔒 **Security**: Private wireless AP instead of direct public Wi-Fi connection
- 🛡️ **Privacy**: Pi-hole blocks ads and trackers for all connected devices
- 🌐 **VPN Integration**: Support for Tailscale, ProtonVPN, and WireGuard
- 📱 **Easy Management**: Web interfaces for all services
- 💾 **Portable**: All data stored on USB drive for complete portability

## Key Features

- **📡 Wireless Access Point**: Dual Wi-Fi setup with secure WPA2 AP for your devices
- **🛡️ Network-Level Ad Blocking**: Pi-hole DNS filtering with comprehensive blocklists
- **🔐 VPN Integration**: Support for Tailscale, ProtonVPN, and custom WireGuard
- **🐳 Container Management**: Docker services with Portainer web interface
- **💾 Portable Storage**: All data stored on USB drive for complete portability
- **🔧 Automated Management**: Health monitoring, backups, and web interfaces

## How It Works

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

**Network Flow:**
1. **wlan0** (Built-in): Connects to public Wi-Fi as client
2. **wlan1** (USB Adapter): Creates secure access point for your devices
3. **Pi-hole**: Filters all DNS requests to block ads and malware
4. **VPN** (Optional): Routes traffic through Tailscale or ProtonVPN

## Hardware Requirements

**Essential Components:**
- Raspberry Pi 4 (4GB+ RAM)
- Compatible USB Wi-Fi adapter (Panda PAU09 recommended)
- USB 3.0 storage drive (64GB+)
- MicroSD card (32GB+ Class 10)
- Official Pi 4 power supply

📊 **[Complete Hardware Guide](docs/hardware/requirements.md)** - Detailed requirements, compatibility lists, and purchasing recommendations


## Getting Started

### 📋 Prerequisites

**Quick Checklist:**
- [ ] Raspberry Pi 4 (4GB+ RAM)
- [ ] Compatible USB Wi-Fi adapter
- [ ] USB 3.0 storage drive (64GB+)
- [ ] MicroSD card (32GB+ Class 10)
- [ ] Official Pi 4 power supply

📊 **[Complete Hardware Guide](docs/hardware/requirements.md)**

### 🚀 Installation Options

#### Guided Setup (Recommended for Beginners)
Follow our step-by-step phase-based installation guides:

1. **[System Preparation](docs/setup/01-system-preparation.md)** - Pi OS setup and storage
2. **[Docker Setup](docs/setup/02-docker-setup.md)** - Container management
3. **[Network Configuration](docs/setup/03-network-configuration.md)** - Access point and routing
4. **[Service Installation](docs/setup/04-service-installation.md)** - Pi-hole and VPN setup

#### Quick Reference (For Experienced Users)
**[Configuration Reference](docs/configuration/)** - All config files and templates

**Estimated Time:** 2-6 hours depending on experience level

## Documentation

The Pi4-TravelRouter documentation is organized into focused, modular guides:

### 📚 Setup & Installation
- **[Hardware Guide](docs/hardware/requirements.md)** - Hardware requirements and compatibility
- **[Setup Guides](docs/setup/)** - Step-by-step installation instructions
- **[Configuration Reference](docs/configuration/)** - Config files and templates

### 🔧 Management & Maintenance
- **[Troubleshooting Guide](docs/troubleshooting/)** - Common issues and solutions
- **[Security Guide](docs/security/)** - Best practices and hardening
- **[Maintenance Guide](docs/maintenance/)** - Backup, updates, and monitoring

### 📋 Service-Specific Guides
- **[Pi-hole DNS](docs/configuration/pihole.md)** - DNS filtering and ad blocking
- **[Docker Compose](docs/configuration/docker-compose.md)** - Container orchestration
- **[Network Config](docs/configuration/networking.md)** - Network setup and routing
- **[ProtonVPN](docs/configuration/protonvpn.md)** - VPN client configuration
- **Tailscale & WG-Easy guides** - Coming soon

## Quick Usage Guide

### Web Interface Access
Once connected to the router's Wi-Fi network (`RaspberryPi-Travel`):

| Service | URL | Purpose |
|---------|-----|----------|
| **Pi-hole** | `http://192.168.4.1/admin` | DNS filtering management |
| **Portainer** | `http://192.168.4.1:9000` | Docker container management |
| **WG-Easy** | `http://192.168.4.1:51821` | WireGuard VPN management |
| **SSH Access** | `ssh pi@192.168.4.1` | Command line access |

### Common Commands
```bash
# Connect to public Wi-Fi
sudo connect-wifi.sh "Network-Name" "password"

# Check system status
sudo systemctl status hostapd
sudo docker ps

# VPN management
sudo tailscale status
sudo wg show
```

📊 **[Complete Usage Guide](docs/setup/04-service-installation.md)**

## Additional Information

🌐 **[Network Details](docs/configuration/networking.md)** - Complete network configuration reference

💾 **[Backup & Maintenance](docs/maintenance/)** - Automated backups and system updates

## Troubleshooting

For comprehensive troubleshooting guides and solutions:

🔍 **[Troubleshooting Guide](docs/troubleshooting/)** - Complete issue resolution guide

### Quick Diagnostics
```bash
# Check service status
sudo systemctl status hostapd
sudo docker ps

# Check network connectivity
ping 8.8.8.8
sudo iptables -t nat -L

# View logs
sudo journalctl -u hostapd -f
sudo docker logs pihole
```

## Security

### Essential Security Steps

⚠️ **Before First Use:**

1. Change all default passwords (system, Wi-Fi, Pi-hole)
2. Configure SSH key authentication
3. Update all software packages
4. Enable firewall with appropriate rules

🔒 **[Complete Security Guide](docs/security/)** - Comprehensive security hardening and best practices

## Credits & License

### Open Source Components
Built with excellent open-source software:
- [Pi-hole](https://pi-hole.net/) - Network-level ad blocking
- [WG-Easy](https://github.com/wg-easy/wg-easy) - WireGuard management UI
- [Docker](https://www.docker.com/) & [Portainer](https://portainer.io/) - Container management
- [Tailscale](https://tailscale.com/) & [ProtonVPN](https://protonvpn.com/) - VPN services
- [BlockListProject](https://github.com/blocklistproject/Lists) - DNS blocklists

### License
**MIT License** - See [LICENSE](LICENSE) file for details

### Contributing
Contributions welcome! Please open an issue for major changes.
- Hardware compatibility testing
- Documentation improvements
- Performance optimizations

### Support
- **Issues**: [GitHub Issues](https://github.com/cypheroxide/Pi4-TravelRouter/issues)
- **Discussions**: [GitHub Discussions](https://github.com/cypheroxide/Pi4-TravelRouter/discussions)

---

🇨🇭 **Made with Swiss precision by [CypherOxide](https://github.com/cypheroxide)**
**🇨🇭 Made with Swiss precision and ☕ caffeine by [CypherOxide](https://github.com/cypheroxide)**
