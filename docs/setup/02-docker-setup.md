# Phase 2: Docker Installation & Container Management

## Overview

This phase installs Docker and Portainer, configures them to use USB storage, and sets up the foundation for containerized services.

**Estimated Time**: 20-30 minutes  
**Difficulty**: Beginner  
**Prerequisites**: [Phase 1 Complete](01-system-preparation.md)

## Pre-Phase Checklist

Ensure Phase 1 is complete:

- [ ] SSH access working
- [ ] USB storage mounted at `/mnt/storage`
- [ ] System updated and optimized
- [ ] 8GB swap file active

```bash
# Quick verification
df -h /mnt/storage && swapon --show && echo "✅ Ready for Phase 2"
```

## Step 1: Install Docker

### Download and Install Docker

```bash
# Download Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh

# Review the script (optional but recommended)
less get-docker.sh

# Install Docker
sudo sh get-docker.sh

# Add current user to docker group
sudo usermod -aG docker $USER

# Clean up installation script
rm get-docker.sh
```

### Configure Docker to Use USB Storage

Docker's default location (`/var/lib/docker`) is on the SD card. We need to move it to USB storage.

```bash
# Stop Docker service
sudo systemctl stop docker

# Create Docker configuration
sudo mkdir -p /etc/docker

# Configure Docker daemon to use USB storage with optimized settings
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/mnt/storage/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
EOF

# Create Docker directory on USB storage
sudo mkdir -p /mnt/storage/docker

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker
```

### Apply User Group Changes

```bash
# Logout and login to apply group changes
echo "Logging out to apply Docker group membership..."
echo "Please SSH back in after this command:"
exit

# After reconnecting via SSH, verify Docker works without sudo:
docker --version
docker ps
```

## Step 2: Install Portainer

Portainer provides a web-based Docker management interface.

### Create Portainer Directories

```bash
# Create Portainer data directory
sudo mkdir -p /mnt/storage/docker/portainer

# Set proper ownership
sudo chown -R $USER:$USER /mnt/storage/docker/portainer
```

### Deploy Portainer Container

```bash
# Pull Portainer image
docker pull portainer/portainer-ce:latest

# Run Portainer with bind mount
docker run -d \
  --name portainer \
  --restart always \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /mnt/storage/docker/portainer:/data \
  portainer/portainer-ce:latest

# Verify Portainer is running
docker ps | grep portainer
```

### Initial Portainer Setup

1. **Access Portainer**: Open `http://YOUR_PI_IP:9000` in your browser
2. **Create Admin Account**: 
   - Username: `admin` (or your preference)
   - Password: Choose a strong password (minimum 12 characters)
3. **Select Environment**: Choose "Docker" (local environment)
4. **Connect**: Portainer should automatically detect your local Docker instance

## Step 3: Create Docker Compose Structure

### Install Docker Compose

```bash
# Docker Compose should be included with Docker installation
# Verify it's available:
docker compose version

# If not available, install it manually:
# sudo apt update && sudo apt install -y docker-compose-plugin
```

### Create Compose Directory Structure

```bash
# Create compose directories
mkdir -p /mnt/storage/docker/compose/{pihole,wg-easy,monitoring}

# Create shared Docker network for containers
docker network create pi-router-net --driver bridge --subnet=172.20.0.0/16

# Verify network was created
docker network ls | grep pi-router-net
```

## Step 4: Create Management Scripts

### Docker Management Script

```bash
# Create Docker management script
sudo tee /usr/local/bin/docker-manager.sh > /dev/null << 'EOF'
#!/bin/bash

# Pi4-TravelRouter Docker Management Script

show_usage() {
    echo "Usage: $0 {status|logs|restart|cleanup|backup}"
    echo ""
    echo "Commands:"
    echo "  status   - Show all container status"
    echo "  logs     - Show logs for all containers"
    echo "  restart  - Restart all containers"
    echo "  cleanup  - Clean unused images and containers"
    echo "  backup   - Backup container data"
}

show_status() {
    echo "=== Docker System Status ==="
    echo "Docker version: $(docker --version)"
    echo "System resources:"
    docker system df
    echo ""
    echo "=== Running Containers ==="
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    echo ""
    echo "=== Docker Networks ==="
    docker network ls
}

show_logs() {
    echo "=== Recent Container Logs ==="
    for container in $(docker ps --format "{{.Names}}"); do
        echo "--- $container ---"
        docker logs --tail 10 "$container"
        echo ""
    done
}

restart_containers() {
    echo "Restarting all containers..."
    docker restart $(docker ps -q)
    echo "All containers restarted"
}

cleanup_docker() {
    echo "Cleaning up Docker system..."
    docker system prune -f
    docker image prune -f
    echo "Docker cleanup completed"
}

backup_containers() {
    TIMESTAMP=$(date +%F_%H%M)
    BACKUP_DIR="/mnt/storage/backups/docker_$TIMESTAMP"
    
    echo "Creating Docker backup: $BACKUP_DIR"
    mkdir -p "$BACKUP_DIR"
    
    # Backup container data
    cp -r /mnt/storage/docker/portainer "$BACKUP_DIR/"
    
    # Backup compose files
    cp -r /mnt/storage/docker/compose "$BACKUP_DIR/"
    
    # Create container list
    docker ps -a > "$BACKUP_DIR/container_list.txt"
    docker images > "$BACKUP_DIR/image_list.txt"
    
    # Compress backup
    tar -czf "/mnt/storage/backups/docker_backup_$TIMESTAMP.tar.gz" -C "$BACKUP_DIR" .
    rm -rf "$BACKUP_DIR"
    
    echo "Docker backup completed: docker_backup_$TIMESTAMP.tar.gz"
}

case "$1" in
    status)   show_status ;;
    logs)     show_logs ;;
    restart)  restart_containers ;;
    cleanup)  cleanup_docker ;;
    backup)   backup_containers ;;
    *)        show_usage; exit 1 ;;
esac
EOF

# Make script executable
sudo chmod +x /usr/local/bin/docker-manager.sh

# Create convenient alias
echo "alias docker-status='docker-manager.sh status'" >> ~/.bashrc
source ~/.bashrc
```

### Container Health Check Script

```bash
# Create health check script
sudo tee /usr/local/bin/container-health.sh > /dev/null << 'EOF'
#!/bin/bash

# Container Health Check Script

check_container_health() {
    local container_name="$1"
    local expected_ports="$2"
    
    if docker ps --format "{{.Names}}" | grep -q "^${container_name}$"; then
        status="🟢 Running"
        health=$(docker inspect "$container_name" --format='{{.State.Health.Status}}' 2>/dev/null || echo "none")
        if [ "$health" != "none" ] && [ "$health" != "healthy" ]; then
            status="🟡 Unhealthy"
        fi
    else
        status="🔴 Stopped"
    fi
    
    printf "%-15s %s\n" "$container_name" "$status"
}

echo "=== Pi4-TravelRouter Container Health ==="
echo "$(date)"
echo ""

# Check core containers
check_container_health "portainer" "9000,9443"

# Check for other containers that may exist
for container in $(docker ps -a --format "{{.Names}}" | grep -v portainer); do
    check_container_health "$container"
done

echo ""
echo "=== System Resources ==="
echo "Memory: $(free -h | grep Mem | awk '{print $3 "/" $2}')"
echo "Disk:   $(df -h /mnt/storage | tail -1 | awk '{print $3 "/" $2 " (" $5 " used)"}')"
echo "Temp:   $(vcgencmd measure_temp | cut -d'=' -f2)"

echo ""
echo "=== Quick Access URLs ==="
echo "Portainer: http://$(hostname -I | awk '{print $1}'):9000"
EOF

# Make script executable
sudo chmod +x /usr/local/bin/container-health.sh
```

## Step 5: Configure Docker Service

### Create Docker Service Override

```bash
# Create systemd override directory
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create override configuration
sudo tee /etc/systemd/system/docker.service.d/override.conf > /dev/null << 'EOF'
[Unit]
# Wait for USB storage to be available
After=mnt-storage.mount
Requires=mnt-storage.mount

[Service]
# Restart policy
Restart=always
RestartSec=10
# Increase timeout for slower USB storage
TimeoutStartSec=300
TimeoutStopSec=120
EOF

# Reload systemd and restart Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Enable Docker Logging

```bash
# Configure rsyslog for Docker (optional but helpful)
echo "# Docker logging" | sudo tee -a /etc/rsyslog.conf
echo "*.info;mail.none;authpriv.none;cron.none /var/log/docker.log" | sudo tee -a /etc/rsyslog.conf

# Restart rsyslog
sudo systemctl restart rsyslog

# Create logrotate config for Docker logs
sudo tee /etc/logrotate.d/docker > /dev/null << 'EOF'
/var/log/docker.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    copytruncate
}
EOF
```

## Step 6: Validation & Testing

### Verify Docker Installation

```bash
# Check Docker service status
sudo systemctl status docker

# Verify Docker is using USB storage
docker system info | grep -E "Docker Root Dir|Storage Driver"

# Check available storage
docker system df
```

### Test Portainer Access

```bash
# Get Pi's IP address
PI_IP=$(hostname -I | awk '{print $1}')
echo "Access Portainer at: http://$PI_IP:9000"

# Test if Portainer is responding
curl -f http://localhost:9000 > /dev/null && echo "✅ Portainer is accessible" || echo "❌ Portainer not responding"
```

### Run Container Tests

```bash
# Test Docker functionality with a simple container
docker run --rm hello-world

# Test network connectivity from container
docker run --rm --network pi-router-net alpine ping -c 3 google.com
```

### Resource Usage Check

```bash
# Check system resources
echo "=== System Resources After Docker Setup ==="
free -h
df -h /mnt/storage
docker system df

# Run health check
container-health.sh
```

## Step 7: Create Automated Maintenance

### Daily Maintenance Cron Job

```bash
# Create maintenance script
sudo tee /usr/local/bin/docker-maintenance.sh > /dev/null << 'EOF'
#!/bin/bash

# Daily Docker Maintenance Script

LOG_FILE="/var/log/docker-maintenance.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log "Starting daily Docker maintenance"

# Clean up unused containers, networks, images
log "Cleaning unused Docker resources"
docker system prune -f >> "$LOG_FILE" 2>&1

# Check container health
log "Checking container health"
/usr/local/bin/container-health.sh >> "$LOG_FILE" 2>&1

# Backup container data (weekly on Sunday)
if [ $(date +%w) -eq 0 ]; then
    log "Running weekly Docker backup"
    /usr/local/bin/docker-manager.sh backup >> "$LOG_FILE" 2>&1
fi

# Rotate logs if they get too large
if [ -f "$LOG_FILE" ] && [ $(stat -f%z "$LOG_FILE" 2>/dev/null || stat -c%s "$LOG_FILE") -gt 10485760 ]; then
    mv "$LOG_FILE" "${LOG_FILE}.old"
    touch "$LOG_FILE"
fi

log "Docker maintenance completed"
EOF

# Make script executable
sudo chmod +x /usr/local/bin/docker-maintenance.sh

# Add to cron (daily at 2:00 AM)
echo "0 2 * * * /usr/local/bin/docker-maintenance.sh" | sudo crontab -

# Verify cron job
sudo crontab -l
```

## Troubleshooting Common Issues

### Docker Won't Start
```bash
# Check logs
sudo journalctl -u docker -f

# Check disk space
df -h /mnt/storage

# Restart Docker
sudo systemctl restart docker
```

### Portainer Not Accessible
```bash
# Check container status
docker ps | grep portainer

# Check logs
docker logs portainer

# Restart container
docker restart portainer
```

### Permission Issues
```bash
# Fix Docker group membership
sudo usermod -aG docker $USER
# Then logout and login again

# Fix directory permissions
sudo chown -R $USER:$USER /mnt/storage/docker/portainer
```

### USB Storage Issues
```bash
# Check mount status
mount | grep /mnt/storage

# Check disk usage
df -h /mnt/storage

# Remount if needed
sudo umount /mnt/storage
sudo mount /mnt/storage
```

## What's Next?

After completing this phase:

✅ **Completed Tasks:**
- Docker installed and configured for USB storage
- Portainer web interface deployed
- Container management scripts created
- Automated maintenance scheduled
- Health monitoring in place

➡️ **Next Phase:** [Network Configuration](03-network-configuration.md)

### Quick Status Check

```bash
# Run this command to verify everything is working:
container-health.sh && echo "✅ Phase 2 Complete - Ready for Network Configuration"
```

Before proceeding to network configuration, ensure:
- Portainer is accessible via web browser
- Docker containers can access the internet
- USB storage is properly utilized by Docker

If you encounter issues, check the [Docker Troubleshooting Guide](../troubleshooting/docker-issues.md).