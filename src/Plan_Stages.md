Create a complete plan, with steps, to turn a Raspberry Pi 4 to function as such:
1. Wireless router/AP with USB Wi-Fi card (Panda PAU09 N600) to connect my devices to Wi-Fi when traveling without connecting directly to public Wi-Fi APs
2. Docker + Portainer for applications and services
3. PiHole Docker for RPI network-level adblocking
4. Tailscale VPN to connect to personal network
5. WireGuard for toggleable private VPN via ProtonVPN WG config
6. USB drive for data and docker app storage
FreedomBox and RaspAP have been used before, but configuration has led to failed configuration, freezing, or cannot integrate with requirements.

This approach will use standard Raspberry Pi OS as the base and configure each component individually for better reliability.

Raspberry Pi 4 Travel Router Complete Setup Plan: `rpi4-travel-router-plan.md`
Document 
This comprehensive plan provides a step-by-step approach to building your Raspberry Pi 4 travel router with all requested features. Here are the key highlights:

Architecture Overview:

Uses Raspberry Pi OS Lite as the stable base
USB Wi-Fi adapter (wlan1) creates the access point
Built-in Wi-Fi (wlan0) connects to public networks
All data and Docker containers stored on USB drive
NAT routing between interfaces for internet sharing
Key Features Implemented:

Wireless AP with the Panda PAU09 adapter using hostapd/dnsmasq
Docker + Portainer with USB storage backend
PiHole as Docker container with DNS integration
Tailscale for personal network access
WireGuard with toggle script for ProtonVPN
Management interface at http://192.168.4.1:8080
Important Notes:

The plan avoids problematic solutions like FreedomBox/RaspAP that caused your previous issues
Uses standard, well-tested components (hostapd, dnsmasq, Docker)
Includes comprehensive backup and troubleshooting procedures
All services are designed to start automatically on boot
Provides easy-to-use scripts for common operations
Before You Start:

Ensure your Panda PAU09 is compatible (most are with modern kernels)
Have your ProtonVPN WireGuard configuration ready
Plan to change all default passwords in the configuration files