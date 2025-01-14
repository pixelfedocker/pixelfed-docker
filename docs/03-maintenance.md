
# Maintenance Guide

This document covers routine maintenance, backup procedures, and monitoring for your Pixelfed instance.

## Table of Contents
- [Routine Maintenance](#routine-maintenance)
- [Backup Procedures](#backup-procedures)
- [System Monitoring](#system-monitoring)
- [Update Procedures](#update-procedures)
- [Troubleshooting](#troubleshooting)

## Routine Maintenance

### Daily Tasks
```bash
#!/bin/bash
# daily-maintenance.sh

# Check system status
docker-compose ps

# Clear cache
docker-compose exec app php artisan cache:clear

# Run scheduled tasks
docker-compose exec app php artisan schedule:run

# Check storage space
df -h /var/lib/docker

# Check logs for errors
docker-compose logs --tail=100 app | grep -i error
```

### Weekly Tasks
```bash
#!/bin/bash
# weekly-maintenance.sh

# Update system packages
sudo apt update && sudo apt upgrade -y

# Prune Docker system
docker system prune -f

# Check SSL certificates
certbot certificates

# Verify backups
./scripts/verify-backups.sh

# Clean old media files (adjust retention period as needed)
find ./storage/app/public/media -type f -mtime +90 -delete
```

## Backup Procedures

### 1. Create Backup Script
```bash
#!/bin/bash
# backup.sh

# Set variables
BACKUP_DIR="/path/to/backups"
DATE=$(date +%Y%m%d_%H%M%S)
INSTANCE_NAME="pixelfed"

# Create backup directory
mkdir -p "$BACKUP_DIR/$DATE"

# Stop services
docker-compose down

# Backup database
docker-compose exec -T db mysqldump \
    --single-transaction \
    --quick \
    --lock-tables=false \
    -u root \
    -p"${DB_ROOT_PASSWORD}" \
    pixelfed > "$BACKUP_DIR/$DATE/database.sql"

# Backup storage
tar -czf "$BACKUP_DIR/$DATE/storage.tar.gz" ./storage

# Backup configurations
tar -czf "$BACKUP_DIR/$DATE/configs.tar.gz" \
    docker-compose.yml \
    .env \
    nginx/

# Start services
docker-compose up -d

# Create checksums
cd "$BACKUP_DIR/$DATE"
sha256sum * > checksums.txt

# Clean old backups (keep last 7 days)
find "$BACKUP_DIR" -type d -mtime +7 -exec rm -rf {} \;

# Log backup completion
echo "Backup completed at $(date)" >> "$BACKUP_DIR/backup.log"
```

### 2. Backup Verification
```bash
#!/bin/bash
# verify-backups.sh

BACKUP_DIR="/path/to/backups"
LATEST_BACKUP=$(ls -td $BACKUP_DIR/*/ | head -1)

# Check backup existence
if [ ! -d "$LATEST_BACKUP" ]; then
    echo "No backup found!"
    exit 1
fi

# Verify checksums
cd "$LATEST_BACKUP"
sha256sum -c checksums.txt

# Test database backup
gunzip -c database.sql.gz | mysql --verbose --force test_restore

# Log verification
echo "Backup verified at $(date)" >> "$BACKUP_DIR/verify.log"
```

## System Monitoring

### 1. Resource Monitoring
```bash
#!/bin/bash
# monitor-resources.sh

# Check CPU usage
top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1"%"}'

# Check memory usage
free -m | awk 'NR==2{printf "Memory Usage: %s/%sMB (%.2f%%)\n", $3,$2,$3*100/$2 }'

# Check disk usage
df -h | grep '/dev/sda1'

# Check Docker container stats
docker stats --no-stream
```

### 2. Application Monitoring
```bash
#!/bin/bash
# monitor-app.sh

# Check application status
docker-compose ps app

# Check application logs
docker-compose logs --tail=100 app

# Check queue status
docker-compose exec app php artisan queue:status

# Check cache status
docker-compose exec app php artisan cache:status
```

### 3. Setup Monitoring Alerts
```bash
#!/bin/bash
# setup-alerts.sh

# Install monitoring tools
sudo apt install -y prometheus node-exporter alertmanager

# Configure Prometheus
cat > /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'docker'
    static_configs:
      - targets: ['localhost:9323']
EOF

# Configure AlertManager
cat > /etc/alertmanager/alertmanager.yml << EOF
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@yourdomain.com'
  smtp_require_tls: false

route:
  group_by: ['alertname']
  receiver: 'admin-mail'

receivers:
- name: 'admin-mail'
  email_configs:
  - to: 'admin@yourdomain.com'
EOF
```

## Update Procedures

### 1. Application Updates
```bash
#!/bin/bash
# update-pixelfed.sh

# Pull latest changes
git pull origin main

# Pull latest Docker images
docker-compose pull

# Stop services
docker-compose down

# Start updated services
docker-compose up -d

# Run migrations
docker-compose exec app php artisan migrate --force

# Clear cache
docker-compose exec app php artisan cache:clear
docker-compose exec app php artisan config:clear
docker-compose exec app php artisan view:clear

# Verify update
docker-compose ps
```

### 2. System Updates
```bash
#!/bin/bash
# update-system.sh

# Update package list
sudo apt update

# Upgrade packages
sudo apt upgrade -y

# Remove unused packages
sudo apt autoremove -y

# Update Docker images
docker-compose pull

# Restart services
docker-compose down
docker-compose up -d

# Clean up
docker system prune -f
```

## Troubleshooting

### Common Issues

1. **Database Connection Issues**
```bash
# Check database connection
docker-compose exec app php artisan db:monitor

# Reset database connection
docker-compose restart db
docker-compose exec app php artisan config:clear
```

2. **Cache Issues**
```bash
# Clear all caches
docker-compose exec app php artisan cache:clear
docker-compose exec app php artisan config:clear
docker-compose exec app php artisan view:clear
docker-compose exec app php artisan route:clear
```

3. **Storage Issues**
```bash
# Fix permissions
docker-compose exec app chown -R www-data:www-data storage
docker-compose exec app chmod -R 755 storage

# Create storage link
docker-compose exec app php artisan storage:link
```

### Maintenance Checklist
- [ ] Daily backup verification
- [ ] Weekly system updates
- [ ] Monthly security audit
- [ ] Quarterly backup restoration test
- [ ] SSL certificate monitoring
- [ ] Disk space monitoring
- [ ] Log rotation check
- [ ] Database optimization
- [ ] Cache performance check
- [ ] Security patch review

### Emergency Procedures
```bash
# Quick instance recovery
./scripts/emergency-recovery.sh

# Service status check
./scripts/check-services.sh

# Incident logging
./scripts/log-incident.sh
```

## Additional Resources
- [Official Pixelfed Documentation](https://docs.pixelfed.org/)
- [Docker Documentation](https://docs.docker.com/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
```
