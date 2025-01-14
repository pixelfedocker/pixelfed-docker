`scripts/backup.sh`:

```bash
#!/bin/bash
set -e

# Configuration
BACKUP_DIR="/opt/pixelfed/backups"
DATE=$(date +%Y%m%d_%H%M%S)
INSTANCE_NAME="pixelfed"
RETENTION_DAYS=7
LOG_FILE="${BACKUP_DIR}/backup.log"

# Ensure backup directory exists
mkdir -p "${BACKUP_DIR}/${DATE}"

# Logging function
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "${LOG_FILE}"
}

# Error handling
handle_error() {
    log_message "ERROR: Backup failed at line $1"
    exit 1
}

trap 'handle_error $LINENO' ERR

# Start backup process
log_message "Starting backup process..."

# Verify docker-compose is running
if ! docker-compose ps | grep -q "Up"; then
    log_message "ERROR: Docker containers are not running!"
    exit 1
fi

# Create database backup
log_message "Creating database backup..."
docker-compose exec -T db mysqldump \
    --single-transaction \
    --quick \
    --lock-tables=false \
    -u "${DB_USERNAME}" \
    -p"${DB_PASSWORD}" \
    "${DB_DATABASE}" > "${BACKUP_DIR}/${DATE}/database.sql"

# Compress database backup
gzip "${BACKUP_DIR}/${DATE}/database.sql"

# Backup storage directory
log_message "Backing up storage directory..."
tar -czf "${BACKUP_DIR}/${DATE}/storage.tar.gz" \
    -C /opt/pixelfed storage/

# Backup configurations
log_message "Backing up configuration files..."
tar -czf "${BACKUP_DIR}/${DATE}/config.tar.gz" \
    -C /opt/pixelfed \
    docker-compose.yml \
    .env \
    nginx/

# Create checksums
log_message "Creating checksums..."
cd "${BACKUP_DIR}/${DATE}"
sha256sum * > checksums.txt

# Cleanup old backups
log_message "Cleaning up old backups..."
find "${BACKUP_DIR}" -type d -mtime "+${RETENTION_DAYS}" -exec rm -rf {} \;

# Create backup symlink
ln -sf "${BACKUP_DIR}/${DATE}" "${BACKUP_DIR}/latest"

log_message "Backup completed successfully!"

# Print backup size
BACKUP_SIZE=$(du -sh "${BACKUP_DIR}/${DATE}" | cut -f1)
log_message "Backup size: ${BACKUP_SIZE}"
```
----------------------------------------------------------------------------------
`scripts/email-setup.sh`:

```bash
#!/bin/bash
set -e

# Configuration
DOMAIN="$1"
EMAIL="$2"
MAIL_SERVER_DIR="/opt/pixelfed/mailserver"

# Check arguments
if [ -z "$DOMAIN" ] || [ -z "$EMAIL" ]; then
    echo "Usage: $0 <domain> <email>"
    echo "Example: $0 example.com admin@example.com"
    exit 1
fi

# Create required directories
mkdir -p "${MAIL_SERVER_DIR}"/{mail-data,mail-state,mail-logs,config}

# Download docker-mailserver setup script
curl -o "${MAIL_SERVER_DIR}/setup.sh" \
    https://raw.githubusercontent.com/docker-mailserver/docker-mailserver/master/setup.sh
chmod +x "${MAIL_SERVER_DIR}/setup.sh"

# Generate DKIM keys
cd "${MAIL_SERVER_DIR}"
./setup.sh config dkim

# Create default email account
./setup.sh email add "${EMAIL}" 

# Configure anti-spam
./setup.sh config spamassassin
./setup.sh config fail2ban

# Configure SSL with Let's Encrypt
if [ ! -d "/etc/letsencrypt/live/${DOMAIN}" ]; then
    certbot certonly --standalone \
        -d "mail.${DOMAIN}" \
        --agree-tos \
        --email "${EMAIL}" \
        --rsa-key-size 4096
fi

# Configure postfix
cat > "${MAIL_SERVER_DIR}/config/postfix-main.cf" << EOF
smtpd_tls_cert_file=/etc/letsencrypt/live/${DOMAIN}/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/${DOMAIN}/privkey.pem
smtp_tls_security_level=may
smtpd_tls_security_level=may
smtp_tls_note_starttls_offer=yes
smtpd_tls_loglevel=1
EOF

# Show DKIM key for DNS configuration
echo "Please add the following DKIM record to your DNS configuration:"
cat "${MAIL_SERVER_DIR}/config/opendkim/keys/${DOMAIN}/mail.txt"

# Create test script
cat > "${MAIL_SERVER_DIR}/test-email.sh" << EOF
#!/bin/bash
docker-compose exec -T mailserver swaks --to ${EMAIL} \
    --from "test@${DOMAIN}" \
    --header "Subject: Test Email" \
    --body "This is a test email from your mail server."
EOF
chmod +x "${MAIL_SERVER_DIR}/test-email.sh"

echo "Email server setup completed!"
echo "Please configure your DNS records as shown above"
echo "Run ./test-email.sh to test your email setup"
```

------------------------------------------------------------------------------------

`scripts/maintenance.sh`:

```bash
#!/bin/bash
set -e

# Configuration
PIXELFED_DIR="/opt/pixelfed"
LOG_FILE="${PIXELFED_DIR}/maintenance.log"

# Logging function
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "${LOG_FILE}"
}

# Check if running as root
if [ "$EUID" -ne 0 ]; then
    log_message "Please run as root"
    exit 1
fi

# Start maintenance
log_message "Starting maintenance tasks..."

# Update system packages
log_message "Updating system packages..."
apt update && apt upgrade -y

# Docker maintenance
log_message "Performing Docker maintenance..."
docker system prune -f --volumes
docker-compose pull

# Pixelfed maintenance
log_message "Performing Pixelfed maintenance..."
cd "${PIXELFED_DIR}"

# Stop services
docker-compose down

# Backup before maintenance
./scripts/backup.sh

# Start services
docker-compose up -d

# Run Pixelfed maintenance commands
docker-compose exec -T app php artisan cache:clear
docker-compose exec -T app php artisan config:cache
docker-compose exec -T app php artisan view:clear
docker-compose exec -T app php artisan route:cache
docker-compose exec -T app php artisan storage:link

# Clean old media files (90 days)
log_message "Cleaning old media files..."
find "${PIXELFED_DIR}/storage/app/public/media" -type f -mtime +90 -delete

# Check SSL certificates
log_message "Checking SSL certificates..."
certbot renew --dry-run

# Check disk space
log_message "Checking disk space..."
df -h | grep '/dev/sda1'

# Check Docker container status
log_message "Checking container status..."
docker-compose ps

# Check logs for errors
log_message "Checking for errors in logs..."
docker-compose logs --tail=100 app | grep -i error || true

# Verify services are running
log_message "Verifying services..."
docker-compose exec -T app php artisan queue:status
docker-compose exec -T app php artisan horizon:status || true

# Security checks
log_message "Running security checks..."
if command -v rkhunter > /dev/null; then
    rkhunter --check --skip-keypress
fi

# Create maintenance report
REPORT="${PIXELFED_DIR}/maintenance-report-$(date +%Y%m%d).txt"
{
    echo "Maintenance Report $(date)"
    echo "========================"
    echo
    echo "Disk Usage:"
    df -h /
    echo
    echo "Docker Status:"
    docker-compose ps
    echo
    echo "Recent Errors:"
    tail -n 50 "${LOG_FILE}"
} > "${REPORT}"

log_message "Maintenance completed! Report saved to ${REPORT}"
```

These scripts should be placed in the `scripts` directory of your Pixelfed installation. Don't forget to make them executable:

```bash
chmod +x scripts/*.sh
```

Each script includes:
- Error handling
- Logging
- Progress feedback
- Configuration options
- Security considerations
- Backup procedures before dangerous operations

You can call these scripts manually or set them up in cron jobs:

```bash
# Example cron entries
# Daily backup at 2 AM
0 2 * * * /opt/pixelfed/scripts/backup.sh

# Weekly maintenance at 3 AM on Sundays
0 3 * * 0 /opt/pixelfed/scripts/maintenance.sh

# Check email setup daily at 4 AM
0 4 * * * /opt/pixelfed/scripts/email-setup.sh
```
