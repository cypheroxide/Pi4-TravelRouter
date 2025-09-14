# Hardware Requirements & Compatibility Guide

## Overview

This guide provides comprehensive hardware requirements, compatibility information, and purchasing recommendations for building your Pi4-TravelRouter.

## Required Components

### Raspberry Pi 4

| Specification | Requirement | Recommended | Notes |
|---------------|-------------|-------------|-------|
| **Model** | Raspberry Pi 4 | Raspberry Pi 4 Model B | Only Pi 4 has sufficient performance |
| **RAM** | 4GB minimum | 8GB | 4GB works but 8GB provides better performance |
| **Storage** | Any capacity | N/A | System runs from SD + USB storage |

**Purchase Links:**
- [Raspberry Pi Foundation](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)
- [Adafruit](https://www.adafruit.com/product/4564)
- [Amazon](https://amazon.com/dp/B0899VXM8F) (8GB model)

### MicroSD Card

| Specification | Requirement | Recommended | Notes |
|---------------|-------------|-------------|-------|
| **Capacity** | 32GB minimum | 64GB | Only holds OS, all data on USB |
| **Class** | Class 10 | A1/A2 Application Class | Faster boot times |
| **Brand** | Any reputable | SanDisk Extreme | Better reliability |

**Recommended Models:**
- SanDisk Extreme 64GB microSDXC (A2, V30, U3)
- Samsung EVO Select 64GB microSDXC (U3, V30)

### USB Wi-Fi Adapter (Access Point)

The USB Wi-Fi adapter creates your private access point. This is the most critical component for compatibility.

#### ✅ Confirmed Compatible Adapters

| Model | Chipset | Bands | Max Speed | AP Mode | Notes |
|-------|---------|-------|-----------|---------|-------|
| **Panda PAU09** | Ralink RT5372 | 2.4GHz | 300Mbps | ✅ Yes | **Recommended** - Excellent Linux support |
| **TP-Link AC600T2U Plus** | Realtek RTL8811AU | 2.4/5GHz | 600Mbps | ✅ Yes | Dual-band, good performance |
| **Alfa AWUS036ACS** | Realtek RTL8812AU | 2.4/5GHz | 867Mbps | ✅ Yes | High power, external antenna |
| **EDUP EP-AC1605** | Realtek RTL8811AU | 2.4/5GHz | 600Mbps | ✅ Yes | Budget-friendly option |

#### ⚠️ Adapter Requirements

**Must Support:**
- Linux AP mode (`hostapd` compatibility)
- Standard Linux drivers (no proprietary drivers)
- Sufficient power draw for Pi 4 USB ports

**Chipset Recommendations:**
- ✅ **Ralink/MediaTek**: Best Linux compatibility
- ✅ **Realtek RTL8811AU/RTL8812AU**: Good with proper drivers
- ⚠️ **Broadcom**: Mixed compatibility
- ❌ **Realtek RTL8188**: Poor AP mode support

**Power Considerations:**
- USB 2.0 devices preferred (lower power draw)
- High-power adapters may need powered USB hub

#### ❌ Known Incompatible Adapters

| Model | Chipset | Issue |
|-------|---------|-------|
| Generic RTL8188 dongles | RTL8188EUS/CUS | No reliable AP mode |
| Most USB-C adapters | Various | Driver issues with Pi OS |
| AC1200+ high-power adapters | Various | Power requirements too high |

### USB Storage Drive

All Docker containers, data, and swap files are stored on external USB storage.

| Specification | Minimum | Recommended | Optimal |
|---------------|---------|-------------|---------|
| **Capacity** | 64GB | 128GB | 256GB+ |
| **Interface** | USB 3.0 | USB 3.0/3.1 | USB 3.1 Gen 2 |
| **Type** | Flash Drive | SSD | Portable SSD |
| **Brand** | Any | SanDisk, Samsung | Samsung T7 |

**Storage Breakdown:**
- System swap: 8GB (recommended)
- Docker containers: 10-20GB
- Pi-hole data: 1-2GB
- Logs and backups: 5-10GB
- VPN configs: <100MB
- **Total minimum**: ~25GB (64GB drive recommended)

**Recommended Options:**

1. **Budget Option**: SanDisk Extreme Go 128GB USB 3.1
2. **Performance Option**: Samsung T7 Portable SSD 500GB USB 3.2
3. **High Capacity**: SanDisk Extreme Portable SSD 1TB USB 3.2

### Power Supply

The Pi 4 + USB devices require adequate power delivery.

| Specification | Minimum | Recommended | Notes |
|---------------|---------|-------------|-------|
| **Voltage** | 5V | 5V | Exactly 5V (not 5.1V or 4.8V) |
| **Current** | 3A | 3.5A+ | Higher current = more stable |
| **Connector** | USB-C | USB-C | Must be USB-C for Pi 4 |
| **Certification** | Any | Official Pi | Better voltage regulation |

**Power Budget:**
- Raspberry Pi 4: ~3-4W (idle) to 7-8W (load)
- USB Wi-Fi Adapter: 1-3W
- USB Storage: 1-2W
- **Total**: ~15W maximum

**Recommended Power Supplies:**
1. **Official Raspberry Pi 4 Power Supply** (15.3W/3.5A) - Best choice
2. **CanaKit 3.5A USB-C Power Supply** - Good alternative
3. **Anker PowerPort III 20W USB-C** - For portable use

### Optional/Recommended Components

#### Ethernet Cable
- **Cat5e or Cat6**
- **1-3 meters length**
- **Purpose**: Initial setup and troubleshooting
- **Cost**: $5-10

#### Protective Case
- **Must have**: Pi 4 compatibility, GPIO access
- **Should have**: Cooling (fan or heatsinks)
- **Nice to have**: DIN rail mounting for permanent installations

**Recommended Cases:**
1. **Argon ONE V2** - Active cooling, clean design
2. **Flirc Case** - Passive cooling, aluminum construction
3. **Official Pi 4 Case** - Budget option with GPIO access

#### USB Hub (For Multiple Devices)
- **Powered USB 3.0 Hub** (if using multiple USB devices)
- **4+ ports**
- **External power adapter**
- **Needed when**: Using both storage and Wi-Fi adapter + other devices

#### Cooling Solutions
- **Active Cooling**: 5V fan (3000-4000 RPM)
- **Passive Cooling**: Heatsinks for CPU and RAM
- **Required when**: Continuous operation in warm environments

#### Portable Power (Mobile Use)

| Capacity | Runtime | Weight | Use Case |
|----------|---------|---------|----------|
| 10,000mAh | 4-6 hours | 200g | Short trips |
| 20,000mAh | 8-12 hours | 400g | Day trips |
| 26,800mAh | 12-18 hours | 600g | Multi-day |

**USB-C PD Support Required**: Most modern power banks work

## Total Cost Breakdown

### Budget Build (~$150)
- Raspberry Pi 4 (4GB): $55
- MicroSD 64GB: $15
- Panda PAU09 Wi-Fi: $25
- USB Flash Drive 128GB: $25
- Power Supply: $15
- Case: $15

### Recommended Build (~$250)
- Raspberry Pi 4 (8GB): $75
- MicroSD 64GB A2: $20
- TP-Link AC600T2U Plus: $35
- Samsung T7 SSD 500GB: $80
- Official Pi Power Supply: $15
- Argon ONE V2 Case: $25

### High-End Build (~$400)
- Raspberry Pi 4 (8GB): $75
- MicroSD 128GB A2: $25
- Alfa AWUS036ACS: $50
- Samsung T7 SSD 1TB: $150
- Official Pi Power Supply: $15
- Premium Case + Cooling: $35
- Powered USB Hub: $25
- 26,800mAh Power Bank: $50

## Purchasing Checklist

### Before You Buy
- [ ] Verify Pi 4 model (not Pi 3 or earlier)
- [ ] Confirm Wi-Fi adapter AP mode compatibility
- [ ] Check power supply specifications (5V, 3A+)
- [ ] Ensure USB storage is USB 3.0+
- [ ] Consider your portability needs

### Retailer Recommendations
1. **Official Distributors**: Best for Pi and official accessories
2. **Amazon**: Good selection, fast shipping, reviews
3. **Adafruit**: Excellent for development boards and accessories
4. **Micro Center**: Good in-store support (US)
5. **RS Components**: Professional-grade components (EU)

### Avoid These Retailers/Products
- ❌ Unbranded "Arduino/Raspberry Pi kits" with generic components
- ❌ Extremely cheap USB Wi-Fi adapters (<$10) - likely incompatible
- ❌ Non-USB-C power supplies for Pi 4
- ❌ MicroSD cards below Class 10
- ❌ Any "Pi 4 compatible" claims without specific chipset information

## Compatibility Database

For the most up-to-date compatibility information:

- **Wi-Fi Adapters**: [Linux Wireless Wiki](https://wireless.wiki.kernel.org/en/users/devices)
- **USB Storage**: Generally all USB 3.0+ devices work
- **Power Supplies**: [Pi Foundation Requirements](https://www.raspberrypi.org/documentation/hardware/raspberrypi/power/README.md)

## Next Steps

Once you have your hardware:
1. Read [Initial Setup Guide](../setup/01-system-preparation.md)
2. Follow [Installation Phase 1](../setup/02-base-installation.md)
3. Configure [Network Setup](../setup/03-network-configuration.md)

For questions about hardware compatibility, check our [Troubleshooting Guide](../troubleshooting/hardware-issues.md) or open an issue on GitHub.