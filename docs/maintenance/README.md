# Pi4-TravelRouter Maintenance Guide

## Overview

This guide covers ongoing maintenance tasks to keep your Pi4-TravelRouter running optimally, including automated backups, system updates, performance monitoring, and preventive maintenance procedures.

**Maintenance Philosophy:**
- **Proactive**: Prevent problems before they occur
- **Automated**: Minimize manual intervention
- **Monitored**: Track system health continuously
- **Documented**: Keep records of all maintenance activities

## Automated Backup System

### Backup Strategy

The Pi4-TravelRouter implements a comprehensive backup strategy with multiple layers:

1. **Daily Automated Backups**: Critical configurations and data
2. **Weekly Full System Backups**: Complete system state
3. **Manual Backups**: Before major changes
4. **Off-site Backups**: Cloud or external storage (optional)

### Daily Backup Configuration

**File**: `/usr/local/bin/backup-router.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter Daily Backup Script
# Run via cron daily at 3:00 AM

# Configuration
BACKUP_BASE_DIR="/mnt/storage/backups"
DATE=$(date +%F)
TIME=$(date +%H%M)
BACKUP_DIR="$BACKUP_BASE_DIR/daily/$DATE"
LOG_FILE="/var/log/backup.log"
RETENTION_DAYS=7
MAX_BACKUP_SIZE="500M"

# Logging function
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Check available space
check_space() {
    local available=$(df /mnt/storage | awk 'NR==2 {print $4}')
    local required=524288  # 512MB in KB
    
    if [ "$available" -lt "$required" ]; then
        log_message "ERROR: Insufficient disk space for backup"
        exit 1
    fi
}

# Create backup directory
create_backup_dir() {
    mkdir -p "$BACKUP_DIR"
    if [ $? -ne 0 ]; then
        log_message "ERROR: Failed to create backup directory: $BACKUP_DIR"
        exit 1
    fi
}

# Backup system configurations
backup_system_configs() {
    log_message "Backing up system configurations..."
    
    # Network configuration
    mkdir -p "$BACKUP_DIR/system/network"
    cp -r /etc/hostapd "$BACKUP_DIR/system/network/" 2>/dev/null
    cp /etc/dnsmasq.conf "$BACKUP_DIR/system/network/" 2>/dev/null
    cp -r /etc/systemd/network "$BACKUP_DIR/system/network/" 2>/dev/null
    cp /etc/dhcpcd.conf "$BACKUP_DIR/system/network/" 2>/dev/null
    
    # System configuration
    mkdir -p "$BACKUP_DIR/system/config"
    cp -r /etc/ssh "$BACKUP_DIR/system/config/" 2>/dev/null
    cp /etc/fstab "$BACKUP_DIR/system/config/" 2>/dev/null
    cp -r /etc/systemd/system "$BACKUP_DIR/system/config/" 2>/dev/null
    cp -r /etc/ufw "$BACKUP_DIR/system/config/" 2>/dev/null
    
    # iptables rules
    if [ -f /etc/iptables/rules.v4 ]; then
        cp /etc/iptables/rules.v4 "$BACKUP_DIR/system/config/"
    fi
    
    log_message "System configurations backed up"
}

# Backup Docker configurations and data
backup_docker() {
    log_message "Backing up Docker configurations..."
    
    # Docker compose files
    mkdir -p "$BACKUP_DIR/docker/compose"
    if [ -d /mnt/storage/docker/compose ]; then
        cp -r /mnt/storage/docker/compose/* "$BACKUP_DIR/docker/compose/" 2>/dev/null
    fi
    
    # Pi-hole configuration (lightweight backup)
    if [ -d /mnt/storage/docker/pihole ]; then
        mkdir -p "$BACKUP_DIR/docker/pihole"
        
        # Essential Pi-hole files only
        cp -r /mnt/storage/docker/pihole/etc "$BACKUP_DIR/docker/pihole/" 2>/dev/null
        cp -r /mnt/storage/docker/pihole/dnsmasq.d "$BACKUP_DIR/docker/pihole/" 2>/dev/null
        
        # Skip large log files to save space
        find "$BACKUP_DIR/docker/pihole" -name "*.log" -delete 2>/dev/null
    fi
    
    # Other container configs (excluding large data)
    for container_dir in portainer wg-easy; do
        if [ -d "/mnt/storage/docker/$container_dir" ]; then
            mkdir -p "$BACKUP_DIR/docker/$container_dir"
            # Only backup configuration files, not large data
            find "/mnt/storage/docker/$container_dir" -type f \( -name "*.conf" -o -name "*.json" -o -name "*.yml" -o -name "*.yaml" \) \
                -exec cp {} "$BACKUP_DIR/docker/$container_dir/" \; 2>/dev/null
        fi
    done
    
    log_message "Docker configurations backed up"
}

# Backup scripts and custom configurations
backup_scripts() {
    log_message "Backing up custom scripts..."
    
    mkdir -p "$BACKUP_DIR/scripts"
    
    # Custom scripts
    if [ -d /usr/local/bin ]; then
        find /usr/local/bin -name "*router*" -o -name "*wifi*" -o -name "*backup*" \
            | xargs -I {} cp {} "$BACKUP_DIR/scripts/" 2>/dev/null
    fi
    
    # Cron jobs
    crontab -l > "$BACKUP_DIR/scripts/crontab.txt" 2>/dev/null
    
    log_message "Scripts backed up"
}

# Create system information snapshot
create_system_snapshot() {
    log_message "Creating system information snapshot..."
    
    mkdir -p "$BACKUP_DIR/system/info"
    
    # System information
    {
        echo "=== System Information ==="
        date
        uname -a
        cat /etc/os-release
        
        echo -e "\n=== Hardware Information ==="
        lscpu
        free -h
        df -h
        lsblk
        lsusb
        vcgencmd measure_temp
        
        echo -e "\n=== Network Configuration ==="
        ip addr show
        ip route show
        iwconfig 2>/dev/null || echo "iwconfig not available"
        
        echo -e "\n=== Service Status ==="
        systemctl status hostapd dnsmasq docker --no-pager -l
        
        echo -e "\n=== Docker Status ==="
        docker ps -a
        docker images
        
        echo -e "\n=== VPN Status ==="
        tailscale status 2>/dev/null || echo "Tailscale not active"
        protonvpn-cli status 2>/dev/null || echo "ProtonVPN not active"
        
    } > "$BACKUP_DIR/system/info/system-snapshot.txt"
    
    log_message "System snapshot created"
}

# Compress backup
compress_backup() {
    log_message "Compressing backup..."
    
    cd "$BACKUP_BASE_DIR/daily"
    tar -czf "${DATE}.tar.gz" "$DATE"
    
    if [ $? -eq 0 ]; then
        rm -rf "$DATE"
        log_message "Backup compressed successfully: ${DATE}.tar.gz"
    else
        log_message "ERROR: Failed to compress backup"
        return 1
    fi
}

# Clean old backups
cleanup_old_backups() {
    log_message "Cleaning up old backups..."
    
    # Remove daily backups older than retention period
    find "$BACKUP_BASE_DIR/daily" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
    
    # Remove any leftover directories
    find "$BACKUP_BASE_DIR/daily" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +
    
    log_message "Old backups cleaned up"
}

# Main backup function
main() {
    log_message "Starting daily backup process..."
    
    check_space
    create_backup_dir
    backup_system_configs
    backup_docker
    backup_scripts
    create_system_snapshot
    compress_backup
    cleanup_old_backups
    
    log_message "Daily backup completed successfully"
    
    # Log backup size
    if [ -f "$BACKUP_BASE_DIR/daily/${DATE}.tar.gz" ]; then
        SIZE=$(du -h "$BACKUP_BASE_DIR/daily/${DATE}.tar.gz" | cut -f1)
        log_message "Backup size: $SIZE"
    fi
}

# Run main function
main
```

### Weekly Full System Backup

**File**: `/usr/local/bin/weekly-backup.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter Weekly Full System Backup
# Run via cron weekly on Sunday at 2:00 AM

BACKUP_DIR="/mnt/storage/backups/weekly"
DATE=$(date +%F)
LOG_FILE="/var/log/backup.log"
RETENTION_WEEKS=4

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

main() {
    log_message "Starting weekly full system backup..."
    
    mkdir -p "$BACKUP_DIR"
    
    # Create full system backup (excluding /proc, /sys, /dev, /tmp, /run)
    tar --create \
        --gzip \
        --file="$BACKUP_DIR/system-full-$DATE.tar.gz" \
        --exclude='/proc/*' \
        --exclude='/sys/*' \
        --exclude='/dev/*' \
        --exclude='/tmp/*' \
        --exclude='/run/*' \
        --exclude='/mnt/storage/backups/*' \
        --exclude='/var/log/*' \
        --exclude='/var/cache/*' \
        --one-file-system \
        / 2>/dev/null
    
    if [ $? -eq 0 ]; then
        log_message "Weekly full system backup completed"
        SIZE=$(du -h "$BACKUP_DIR/system-full-$DATE.tar.gz" | cut -f1)
        log_message "Full backup size: $SIZE"
    else
        log_message "ERROR: Weekly full system backup failed"
    fi
    
    # Clean old weekly backups
    find "$BACKUP_DIR" -name "system-full-*.tar.gz" -mtime +$((RETENTION_WEEKS * 7)) -delete
    log_message "Old weekly backups cleaned up"
}

main
```

### Backup Scheduling

Configure cron jobs for automated backups:

```bash
# Edit crontab
sudo crontab -e

# Add these entries:
# Daily backup at 3:00 AM
0 3 * * * /usr/local/bin/backup-router.sh

# Weekly full backup at 2:00 AM on Sunday
0 2 * * 0 /usr/local/bin/weekly-backup.sh

# Monthly backup cleanup
0 1 1 * * /usr/local/bin/cleanup-backups.sh
```

### Manual Backup Commands

**Quick Configuration Backup:**
```bash
# Before making changes
sudo /usr/local/bin/backup-router.sh

# Manual backup with custom name
sudo tar -czf "/mnt/storage/backups/manual/pre-change-$(date +%F-%H%M).tar.gz" \
    /etc/hostapd /etc/dnsmasq.conf /mnt/storage/docker/compose
```

**Emergency Backup:**
```bash
# Emergency backup to external drive
sudo tar -czf "/media/usb-backup/emergency-$(date +%F).tar.gz" \
    /etc /home/pi /mnt/storage/docker /usr/local/bin
```

## System Updates and Maintenance

### Automated Update System

**File**: `/usr/local/bin/system-update.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter System Update Script

LOG_FILE="/var/log/system-updates.log"
REBOOT_REQUIRED_FILE="/var/run/reboot-required"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Update system packages
update_system_packages() {
    log_message "Updating system packages..."
    
    apt update
    UPDATES_AVAILABLE=$(apt list --upgradable 2>/dev/null | wc -l)
    
    if [ "$UPDATES_AVAILABLE" -gt 1 ]; then
        log_message "$((UPDATES_AVAILABLE - 1)) updates available"
        
        # Create pre-update backup
        /usr/local/bin/backup-router.sh
        
        # Apply updates
        DEBIAN_FRONTEND=noninteractive apt upgrade -y
        DEBIAN_FRONTEND=noninteractive apt autoremove -y
        apt autoclean
        
        log_message "System packages updated"
        
        # Check if reboot is required
        if [ -f "$REBOOT_REQUIRED_FILE" ]; then
            log_message "System reboot required after updates"
        fi
    else
        log_message "System packages are up to date"
    fi
}

# Update Docker containers
update_docker_containers() {
    log_message "Updating Docker containers..."
    
    cd /mnt/storage/docker/compose
    
    # Pull latest images
    docker compose pull
    
    # Update containers with new images
    docker compose up -d --remove-orphans
    
    # Clean up old images
    docker image prune -f
    
    log_message "Docker containers updated"
}

# Update Tailscale
update_tailscale() {
    if command -v tailscale &> /dev/null; then
        log_message "Updating Tailscale..."
        tailscale update
        log_message "Tailscale updated"
    fi
}

# Update ProtonVPN
update_protonvpn() {
    if command -v protonvpn-cli &> /dev/null; then
        log_message "Checking ProtonVPN updates..."
        apt update
        apt install --only-upgrade protonvpn-cli -y
        log_message "ProtonVPN checked for updates"
    fi
}

# System health check after updates
post_update_health_check() {
    log_message "Performing post-update health check..."
    
    # Check critical services
    services=("hostapd" "dnsmasq" "docker")
    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service"; then
            log_message "✓ $service is running"
        else
            log_message "✗ $service is not running"
            systemctl restart "$service"
        fi
    done
    
    # Check Docker containers
    containers=$(docker ps --format "{{.Names}}")
    for container in $containers; do
        if docker ps | grep -q "$container"; then
            log_message "✓ Container $container is running"
        else
            log_message "✗ Container $container is not running"
        fi
    done
    
    # Check network connectivity
    if ping -c 1 8.8.8.8 > /dev/null 2>&1; then
        log_message "✓ Internet connectivity working"
    else
        log_message "✗ Internet connectivity failed"
    fi
    
    log_message "Post-update health check completed"
}

# Main update function
main() {
    log_message "Starting system update process..."
    
    update_system_packages
    update_docker_containers
    update_tailscale
    update_protonvpn
    post_update_health_check
    
    log_message "System update process completed"
}

main
```

### Update Scheduling

```bash
# Weekly system updates on Sunday at 4:00 AM
0 4 * * 0 /usr/local/bin/system-update.sh

# Daily security updates (unattended-upgrades handles this)
# Configured via /etc/apt/apt.conf.d/20auto-upgrades
```

## Performance Monitoring

### System Health Monitoring

**File**: `/usr/local/bin/health-monitor.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter Health Monitoring Script

LOG_FILE="/var/log/health-monitor.log"
ALERT_LOG="/var/log/health-alerts.log"

# Thresholds
CPU_TEMP_THRESHOLD=75        # Celsius
MEMORY_THRESHOLD=85          # Percentage
DISK_THRESHOLD=90           # Percentage
LOAD_THRESHOLD=3.0          # Load average

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

log_alert() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - ALERT: $1" | tee -a "$ALERT_LOG"
}

# Check CPU temperature
check_temperature() {
    local temp=$(vcgencmd measure_temp | cut -d'=' -f2 | cut -d"'" -f1)
    local temp_int=${temp%.*}
    
    log_message "CPU Temperature: ${temp}°C"
    
    if [ "$temp_int" -gt "$CPU_TEMP_THRESHOLD" ]; then
        log_alert "High CPU temperature: ${temp}°C (threshold: ${CPU_TEMP_THRESHOLD}°C)"
        return 1
    fi
    
    return 0
}

# Check memory usage
check_memory() {
    local memory_usage=$(free | grep Mem | awk '{printf("%.0f", $3/$2 * 100.0)}')
    
    log_message "Memory Usage: ${memory_usage}%"
    
    if [ "$memory_usage" -gt "$MEMORY_THRESHOLD" ]; then
        log_alert "High memory usage: ${memory_usage}% (threshold: ${MEMORY_THRESHOLD}%)"
        
        # Log top memory consumers
        log_message "Top memory consumers:"
        ps aux --sort=-%mem | head -10 >> "$LOG_FILE"
        
        return 1
    fi
    
    return 0
}

# Check disk usage
check_disk_usage() {
    local disk_usage=$(df /mnt/storage | awk 'NR==2 {print $5}' | sed 's/%//')
    
    log_message "Storage Usage: ${disk_usage}%"
    
    if [ "$disk_usage" -gt "$DISK_THRESHOLD" ]; then
        log_alert "High disk usage: ${disk_usage}% (threshold: ${DISK_THRESHOLD}%)"
        
        # Log largest directories
        log_message "Largest directories in /mnt/storage:"
        du -h /mnt/storage | sort -hr | head -10 >> "$LOG_FILE"
        
        return 1
    fi
    
    return 0
}

# Check system load
check_system_load() {
    local load_avg=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
    
    log_message "Load Average: $load_avg"
    
    # Compare load (bash doesn't handle float comparison well)
    local load_int=$(echo "$load_avg" | cut -d'.' -f1)
    local threshold_int=$(echo "$LOAD_THRESHOLD" | cut -d'.' -f1)
    
    if [ "$load_int" -gt "$threshold_int" ]; then
        log_alert "High system load: $load_avg (threshold: $LOAD_THRESHOLD)"
        
        # Log top CPU consumers
        log_message "Top CPU consumers:"
        ps aux --sort=-%cpu | head -10 >> "$LOG_FILE"
        
        return 1
    fi
    
    return 0
}

# Check network interfaces
check_network_interfaces() {
    local interfaces=("wlan0" "wlan1")
    local issues=0
    
    for interface in "${interfaces[@]}"; do
        if ip link show "$interface" | grep -q "state UP"; then
            log_message "Interface $interface is UP"
        else
            log_alert "Interface $interface is DOWN"
            issues=$((issues + 1))
        fi
    done
    
    return $issues
}

# Check essential services
check_services() {
    local services=("hostapd" "dnsmasq" "docker")
    local failed_services=0
    
    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service"; then
            log_message "Service $service is running"
        else
            log_alert "Service $service is not running"
            failed_services=$((failed_services + 1))
            
            # Attempt to restart failed service
            log_message "Attempting to restart $service..."
            systemctl restart "$service"
            
            sleep 5
            
            if systemctl is-active --quiet "$service"; then
                log_message "Service $service restarted successfully"
            else
                log_alert "Failed to restart $service"
            fi
        fi
    done
    
    return $failed_services
}

# Check Docker containers
check_docker_containers() {
    local expected_containers=("pihole")
    local failed_containers=0
    
    for container in "${expected_containers[@]}"; do
        if docker ps | grep -q "$container"; then
            log_message "Container $container is running"
        else
            log_alert "Container $container is not running"
            failed_containers=$((failed_containers + 1))
            
            # Attempt to restart failed container
            log_message "Attempting to restart $container..."
            docker restart "$container"
        fi
    done
    
    return $failed_containers
}

# Check internet connectivity
check_connectivity() {
    local hosts=("8.8.8.8" "1.1.1.1")
    local failed_hosts=0
    
    for host in "${hosts[@]}"; do
        if ping -c 1 -W 5 "$host" > /dev/null 2>&1; then
            log_message "Connectivity to $host: OK"
        else
            log_alert "Connectivity to $host: FAILED"
            failed_hosts=$((failed_hosts + 1))
        fi
    done
    
    return $failed_hosts
}

# Main monitoring function
main() {
    log_message "=== Health Check Started ==="
    
    local total_issues=0
    
    check_temperature
    total_issues=$((total_issues + $?))
    
    check_memory
    total_issues=$((total_issues + $?))
    
    check_disk_usage
    total_issues=$((total_issues + $?))
    
    check_system_load
    total_issues=$((total_issues + $?))
    
    check_network_interfaces
    total_issues=$((total_issues + $?))
    
    check_services
    total_issues=$((total_issues + $?))
    
    check_docker_containers
    total_issues=$((total_issues + $?))
    
    check_connectivity
    total_issues=$((total_issues + $?))
    
    if [ "$total_issues" -eq 0 ]; then
        log_message "=== Health Check Completed - All systems healthy ==="
    else
        log_alert "Health Check Completed - $total_issues issues detected"
    fi
    
    log_message "=== Health Check Ended ==="
}

main
```

### Performance Monitoring Dashboard

**File**: `/usr/local/bin/router-dashboard.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter Performance Dashboard

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Function to print colored output
print_status() {
    local status=$1
    local message=$2
    
    case $status in
        "OK") echo -e "${GREEN}✓${NC} $message" ;;
        "WARNING") echo -e "${YELLOW}⚠${NC} $message" ;;
        "ERROR") echo -e "${RED}✗${NC} $message" ;;
        "INFO") echo -e "${BLUE}ℹ${NC} $message" ;;
    esac
}

# System information
show_system_info() {
    echo "=== Pi4-TravelRouter Dashboard ==="
    echo "Date: $(date)"
    echo "Uptime: $(uptime -p)"
    
    # CPU Temperature
    local temp=$(vcgencmd measure_temp | cut -d'=' -f2 | cut -d"'" -f1)
    local temp_int=${temp%.*}
    
    if [ "$temp_int" -lt 60 ]; then
        print_status "OK" "CPU Temperature: ${temp}°C"
    elif [ "$temp_int" -lt 75 ]; then
        print_status "WARNING" "CPU Temperature: ${temp}°C"
    else
        print_status "ERROR" "CPU Temperature: ${temp}°C (HIGH)"
    fi
    
    # Memory Usage
    local memory_total=$(free -m | awk 'NR==2{print $2}')
    local memory_used=$(free -m | awk 'NR==2{print $3}')
    local memory_percent=$((memory_used * 100 / memory_total))
    
    if [ "$memory_percent" -lt 70 ]; then
        print_status "OK" "Memory: ${memory_used}MB/${memory_total}MB (${memory_percent}%)"
    elif [ "$memory_percent" -lt 85 ]; then
        print_status "WARNING" "Memory: ${memory_used}MB/${memory_total}MB (${memory_percent}%)"
    else
        print_status "ERROR" "Memory: ${memory_used}MB/${memory_total}MB (${memory_percent}%) HIGH"
    fi
    
    # Storage Usage
    local storage_info=$(df -h /mnt/storage | awk 'NR==2 {print $3"/"$2" ("$5")"}')
    local storage_percent=$(df /mnt/storage | awk 'NR==2 {print $5}' | sed 's/%//')
    
    if [ "$storage_percent" -lt 80 ]; then
        print_status "OK" "Storage: $storage_info"
    elif [ "$storage_percent" -lt 90 ]; then
        print_status "WARNING" "Storage: $storage_info"
    else
        print_status "ERROR" "Storage: $storage_info HIGH"
    fi
}

# Network status
show_network_status() {
    echo -e "\n=== Network Status ==="
    
    # Interface status
    if ip link show wlan0 | grep -q "state UP"; then
        print_status "OK" "wlan0 (Internet): UP"
    else
        print_status "ERROR" "wlan0 (Internet): DOWN"
    fi
    
    if ip link show wlan1 | grep -q "state UP"; then
        print_status "OK" "wlan1 (Access Point): UP"
        
        # Connected clients
        local clients=$(arp -a | grep "192.168.4" | wc -l)
        print_status "INFO" "Connected clients: $clients"
    else
        print_status "ERROR" "wlan1 (Access Point): DOWN"
    fi
    
    # Internet connectivity
    if ping -c 1 -W 5 8.8.8.8 > /dev/null 2>&1; then
        print_status "OK" "Internet connectivity"
    else
        print_status "ERROR" "Internet connectivity failed"
    fi
}

# Service status
show_service_status() {
    echo -e "\n=== Service Status ==="
    
    local services=("hostapd" "dnsmasq" "docker")
    
    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service"; then
            print_status "OK" "$service service"
        else
            print_status "ERROR" "$service service (stopped)"
        fi
    done
}

# Container status
show_container_status() {
    echo -e "\n=== Container Status ==="
    
    if command -v docker &> /dev/null; then
        local containers=$(docker ps --format "{{.Names}}")
        
        if [ -n "$containers" ]; then
            for container in $containers; do
                local status=$(docker inspect --format='{{.State.Status}}' "$container")
                if [ "$status" = "running" ]; then
                    print_status "OK" "Container: $container"
                else
                    print_status "ERROR" "Container: $container ($status)"
                fi
            done
        else
            print_status "WARNING" "No containers running"
        fi
    else
        print_status "ERROR" "Docker not available"
    fi
}

# VPN status
show_vpn_status() {
    echo -e "\n=== VPN Status ==="
    
    # Tailscale
    if command -v tailscale &> /dev/null; then
        if tailscale status | grep -q "logged in"; then
            print_status "OK" "Tailscale connected"
        else
            print_status "WARNING" "Tailscale not connected"
        fi
    else
        print_status "INFO" "Tailscale not installed"
    fi
    
    # ProtonVPN
    if command -v protonvpn-cli &> /dev/null; then
        if protonvpn-cli status | grep -q "Connected"; then
            print_status "OK" "ProtonVPN connected"
        else
            print_status "WARNING" "ProtonVPN not connected"
        fi
    else
        print_status "INFO" "ProtonVPN not installed"
    fi
}

# Performance metrics
show_performance_metrics() {
    echo -e "\n=== Performance Metrics ==="
    
    # Load average
    local load=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
    print_status "INFO" "Load average: $load"
    
    # Network traffic (if available)
    if command -v vnstat &> /dev/null; then
        print_status "INFO" "Network usage available via 'vnstat'"
    fi
    
    # Docker stats
    if [ "$(docker ps -q | wc -l)" -gt 0 ]; then
        echo -e "\n=== Container Resource Usage ==="
        docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
    fi
}

# Recent alerts
show_recent_alerts() {
    if [ -f "/var/log/health-alerts.log" ]; then
        echo -e "\n=== Recent Alerts (Last 24 Hours) ==="
        grep "$(date '+%Y-%m-%d')" /var/log/health-alerts.log | tail -5
    fi
}

# Main dashboard function
main() {
    clear
    show_system_info
    show_network_status
    show_service_status
    show_container_status
    show_vpn_status
    show_performance_metrics
    show_recent_alerts
    echo
}

# Check if running interactively
if [ -t 1 ]; then
    # Interactive mode
    while true; do
        main
        echo "Press Ctrl+C to exit, or wait 30 seconds for refresh..."
        sleep 30
    done
else
    # Non-interactive mode
    main
fi
```

## Log Management

### Log Rotation Configuration

**File**: `/etc/logrotate.d/pi4-router`

```bash
# Pi4-TravelRouter log rotation configuration

/var/log/backup.log
/var/log/health-monitor.log
/var/log/health-alerts.log
/var/log/system-updates.log
/var/log/dns-security.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
}

# Pi-hole logs (handled by container)
/mnt/storage/docker/pihole/pihole.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

### Log Monitoring Script

**File**: `/usr/local/bin/log-analyzer.sh`

```bash
#!/bin/bash
# Pi4-TravelRouter Log Analysis Script

REPORT_FILE="/var/log/daily-log-report.txt"
DATE=$(date +%F)

# Analyze system logs for issues
analyze_system_logs() {
    echo "=== System Log Analysis for $DATE ===" > "$REPORT_FILE"
    
    # Check for errors
    ERROR_COUNT=$(grep -c "error\|Error\|ERROR" /var/log/syslog)
    echo "System Errors: $ERROR_COUNT" >> "$REPORT_FILE"
    
    if [ "$ERROR_COUNT" -gt 0 ]; then
        echo "Recent errors:" >> "$REPORT_FILE"
        grep "error\|Error\|ERROR" /var/log/syslog | tail -10 >> "$REPORT_FILE"
    fi
    
    # Check for failed services
    FAILED_SERVICES=$(grep "Failed to start\|failed" /var/log/syslog | wc -l)
    echo "Failed service starts: $FAILED_SERVICES" >> "$REPORT_FILE"
}

# Analyze network logs
analyze_network_logs() {
    echo -e "\n=== Network Analysis ===" >> "$REPORT_FILE"
    
    # Check hostapd logs
    if journalctl -u hostapd --since "1 day ago" | grep -q "error\|failed"; then
        echo "hostapd issues detected:" >> "$REPORT_FILE"
        journalctl -u hostapd --since "1 day ago" | grep "error\|failed" | tail -5 >> "$REPORT_FILE"
    fi
    
    # Check for frequent disconnections
    DISCONNECTIONS=$(journalctl -u hostapd --since "1 day ago" | grep -c "disconnected")
    echo "WiFi disconnections today: $DISCONNECTIONS" >> "$REPORT_FILE"
}

# Generate summary
generate_summary() {
    echo -e "\n=== Daily Summary ===" >> "$REPORT_FILE"
    echo "Report generated: $(date)" >> "$REPORT_FILE"
    echo "System uptime: $(uptime -p)" >> "$REPORT_FILE"
    echo "CPU temperature: $(vcgencmd measure_temp)" >> "$REPORT_FILE"
    
    # Log file to system log
    logger -t "Pi4-Router-Report" "Daily log analysis completed. Check $REPORT_FILE for details."
}

main() {
    analyze_system_logs
    analyze_network_logs
    generate_summary
}

main
```

## Maintenance Scheduling

### Complete Maintenance Schedule

```bash
# Edit root crontab
sudo crontab -e

# Add these maintenance tasks:

# === DAILY TASKS ===
# Daily backup at 3:00 AM
0 3 * * * /usr/local/bin/backup-router.sh

# Health monitoring every 30 minutes
*/30 * * * * /usr/local/bin/health-monitor.sh

# Daily log analysis at 6:00 AM
0 6 * * * /usr/local/bin/log-analyzer.sh

# === WEEKLY TASKS ===
# Weekly full system backup on Sunday at 2:00 AM
0 2 * * 0 /usr/local/bin/weekly-backup.sh

# Weekly system updates on Sunday at 4:00 AM
0 4 * * 0 /usr/local/bin/system-update.sh

# Weekly log cleanup on Monday at 1:00 AM
0 1 * * 1 /usr/sbin/logrotate /etc/logrotate.d/pi4-router

# === MONTHLY TASKS ===
# Monthly backup cleanup on first day at 1:00 AM
0 1 1 * * /usr/local/bin/cleanup-backups.sh

# Monthly system maintenance on first Sunday at 5:00 AM
0 5 1-7 * 0 /usr/local/bin/monthly-maintenance.sh
```

## Maintenance Procedures

### Monthly Maintenance Checklist

**File**: `/usr/local/bin/monthly-maintenance.sh`

```bash
#!/bin/bash
# Monthly maintenance tasks

LOG_FILE="/var/log/monthly-maintenance.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

main() {
    log_message "=== Starting Monthly Maintenance ==="
    
    # System health check
    log_message "Running system health check..."
    /usr/local/bin/health-monitor.sh
    
    # Update system packages
    log_message "Updating system packages..."
    /usr/local/bin/system-update.sh
    
    # Docker maintenance
    log_message "Docker maintenance..."
    docker system prune -f
    
    # Check disk usage and clean up
    log_message "Checking disk usage..."
    df -h >> "$LOG_FILE"
    
    # Clean old logs
    log_message "Cleaning old logs..."
    find /var/log -name "*.log.*" -mtime +30 -delete
    
    # Generate monthly report
    log_message "Generating monthly report..."
    {
        echo "=== Monthly System Report ==="
        echo "Date: $(date)"
        echo "Uptime: $(uptime)"
        echo "Temperature: $(vcgencmd measure_temp)"
        echo "Memory: $(free -h)"
        echo "Disk: $(df -h)"
        echo "Services: $(systemctl list-units --failed)"
        echo "Docker: $(docker ps)"
    } > "/var/log/monthly-report-$(date +%Y-%m).txt"
    
    log_message "=== Monthly Maintenance Completed ==="
}

main
```

### Emergency Maintenance Procedures

**File**: `/usr/local/bin/emergency-maintenance.sh`

```bash
#!/bin/bash
# Emergency maintenance procedures

emergency_stop() {
    echo "EMERGENCY STOP: Shutting down non-essential services..."
    
    # Stop non-critical containers
    docker stop $(docker ps -q --filter "name=portainer")
    docker stop $(docker ps -q --filter "name=wg-easy")
    
    # Keep Pi-hole and core networking running
    echo "Core services (Pi-hole, hostapd, dnsmasq) remain active"
}

emergency_restart() {
    echo "EMERGENCY RESTART: Restarting all services..."
    
    # Restart networking
    systemctl restart hostapd dnsmasq
    
    # Restart Docker containers
    cd /mnt/storage/docker/compose
    docker compose restart
    
    echo "Emergency restart completed"
}

case "$1" in
    stop)
        emergency_stop
        ;;
    restart)
        emergency_restart
        ;;
    *)
        echo "Usage: $0 {stop|restart}"
        echo "  stop    - Emergency stop of non-essential services"
        echo "  restart - Emergency restart of all services"
        exit 1
        ;;
esac
```

## Making Scripts Executable

```bash
# Make all maintenance scripts executable
sudo chmod +x /usr/local/bin/backup-router.sh
sudo chmod +x /usr/local/bin/weekly-backup.sh
sudo chmod +x /usr/local/bin/system-update.sh
sudo chmod +x /usr/local/bin/health-monitor.sh
sudo chmod +x /usr/local/bin/router-dashboard.sh
sudo chmod +x /usr/local/bin/log-analyzer.sh
sudo chmod +x /usr/local/bin/monthly-maintenance.sh
sudo chmod +x /usr/local/bin/emergency-maintenance.sh
```

---

## Quick Reference Commands

### Daily Operations
```bash
# Check system status
/usr/local/bin/router-dashboard.sh

# Manual backup
sudo /usr/local/bin/backup-router.sh

# Health check
sudo /usr/local/bin/health-monitor.sh
```

### Emergency Operations
```bash
# Emergency service restart
sudo /usr/local/bin/emergency-maintenance.sh restart

# View system logs
sudo journalctl -f

# Check failed services
sudo systemctl --failed
```

### Monitoring Commands
```bash
# Real-time system monitoring
htop

# Network traffic monitoring
sudo iftop -i wlan1

# Docker container monitoring
docker stats
```

This comprehensive maintenance system ensures your Pi4-TravelRouter remains healthy, secure, and performant through automated monitoring, regular backups, and systematic updates.