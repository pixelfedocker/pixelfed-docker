# Email Server Setup Guide

This guide covers the setup and configuration of email services for your Pixelfed instance using docker-mailserver.

## Table of Contents
- [Prerequisites](#prerequisites)
- [DNS Configuration](#dns-configuration)
- [Docker Configuration](#docker-configuration)
- [Email Server Setup](#email-server-setup)
- [Pixelfed Integration](#pixelfed-integration)
- [Security & Spam Prevention](#security--spam-prevention)
- [Monitoring & Maintenance](#monitoring--maintenance)

## Prerequisites

### Required DNS Records
Before starting, add these records to your domain:
```text
# MX Record
Type: MX
Host: @
Value: mail.yourdomain.com
Priority: 10

# SPF Record
Type: TXT
Host: @
Value: v=spf1 mx a:mail.yourdomain.com -all

# DKIM Record (will be generated later)
Type: TXT
Host: mail._domainkey
Value: [Generated during setup]

# DMARC Record
Type: TXT
Host: _dmarc
Value: v=DMARC1; p=quarantine; rua=mailto:postmaster@yourdomain.com
```

## Docker Configuration

### 1. Update Docker Compose
Add to your existing `docker-compose.yml`:
```yaml
services:
  # ... existing services ...

  mailserver:
    image: docker.io/mailserver/docker-mailserver:latest
    container_name: mailserver
    hostname: mail.yourdomain.com
    domainname: yourdomain.com
    ports:
      - "25:25"    # SMTP  (required)
      - "465:465"  # SMTPS (optional)
      - "587:587"  # Submission (required)
      - "993:993"  # IMAPS (optional)
    volumes:
      - ./mailserver/mail-data:/var/mail
      - ./mailserver/mail-state:/var/mail-state
      - ./mailserver/mail-logs:/var/log/mail
      - ./mailserver/config:/tmp/docker-mailserver
      - /etc/letsencrypt:/etc/letsencrypt:ro
    environment:
      - ENABLE_SPAMASSASSIN=1
      - ENABLE_CLAMAV=1
      - ENABLE_FAIL2BAN=1
      - ENABLE_POSTGREY=1
      - SSL_TYPE=letsencrypt
      - ONE_DIR=1
      - PERMIT_DOCKER=network
      - POSTMASTER_ADDRESS=postmaster@yourdomain.com
      - POSTSCREEN_ACTION=enforce
      - SPAMASSASSIN_SPAM_TO_INBOX=0
      - ENABLE_MANAGESIEVE=1
      - ENABLE_QUOTAS=1
      - QUOTA_POLICY=20M
    cap_add:
      - NET_ADMIN
      - SYS_PTRACE
    restart: always
    stop_grace_period: 1m
    networks:
      - pixelfed_network
```

### 2. Create Directory Structure
```bash
#!/bin/bash
# setup-mail-dirs.sh

# Create mail server directories
mkdir -p mailserver/{mail-data,mail-state,mail-logs,config}

# Set permissions
chmod -R 755 mailserver
```

## Email Server Setup

### 1. Initial Setup Script
```bash
#!/bin/bash
# setup-mailserver.sh

# Download setup helpers
curl -o setup.sh https://raw.githubusercontent.com/docker-mailserver/docker-mailserver/master/setup.sh
chmod +x setup.sh

# Generate DKIM keys
./setup.sh config dkim

# Create first email account
./setup.sh email add admin@yourdomain.com yourpassword

# Configure anti-spam
./setup.sh config spamassassin
./setup.sh config fail2ban

# Show DKIM key (add this to DNS)
cat config/opendkim/keys/yourdomain.com/mail.txt
```

### 2. Configure SSL
```bash
#!/bin/bash
# setup-mail-ssl.sh

# Ensure SSL certificates exist
if [ ! -d "/etc/letsencrypt/live/yourdomain.com" ]; then
    certbot certonly --standalone \
        -d mail.yourdomain.com \
        --agree-tos \
        --email admin@yourdomain.com
fi

# Configure postfix to use SSL
cat > mailserver/config/postfix-main.cf << EOF
smtpd_tls_cert_file=/etc/letsencrypt/live/yourdomain.com/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/yourdomain.com/privkey.pem
smtp_tls_security_level=may
smtpd_tls_security_level=may
smtp_tls_note_starttls_offer=yes
smtpd_tls_loglevel=1
EOF
```

## Pixelfed Integration

### 1. Update Pixelfed Environment
Add to your `.env` file:
```env
MAIL_MAILER=smtp
MAIL_HOST=mailserver
MAIL_PORT=587
MAIL_USERNAME=pixelfed@yourdomain.com
MAIL_PASSWORD=your-secure-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=pixelfed@yourdomain.com
MAIL_FROM_NAME="${APP_NAME}"
```

### 2. Create Mail Account for Pixelfed
```bash
./setup.sh email add pixelfed@yourdomain.com your-secure-password
```

### 3. Test Email Configuration
```bash
#!/bin/bash
# test-mail.sh

# Test Pixelfed mail
docker-compose exec app php artisan mail:test

# Check mail logs
docker-compose exec mailserver tail -f /var/log/mail/mail.log
```

## Security & Spam Prevention

### 1. Configure SpamAssassin
Create `mailserver/config/spamassassin-rules.cf`:
```text
# SpamAssassin rules
required_score 5.0
report_safe 0
use_bayes 1
bayes_auto_learn 1
skip_rbl_checks 0
use_razor2 1
use_pyzor 1
```

### 2. Configure Fail2ban
Create `mailserver/config/fail2ban-jail.cf`:
```text
[postfix]
enabled = true
filter = postfix
logpath = /var/log/mail/mail.log
maxretry = 3
findtime = 600
bantime = 3600
```

### 3. Setup DMARC Reporting
```bash
#!/bin/bash
# setup-dmarc.sh

# Install OpenDMARC
docker-compose exec mailserver apt-get update
docker-compose exec mailserver apt-get install -y opendmarc

# Configure OpenDMARC
cat > mailserver/config/opendmarc.conf << EOF
AuthservID OpenDMARC
PidFile /var/run/opendmarc/opendmarc.pid
RejectFailures false
Syslog true
EOF
```

## Monitoring & Maintenance

### 1. Monitoring Script
```bash
#!/bin/bash
# monitor-mail.sh

# Check mail queue
docker-compose exec mailserver postqueue -p

# Check mail logs
docker-compose exec mailserver grep -i "error\|warning" /var/log/mail/mail.log

# Check SpamAssassin status
docker-compose exec mailserver sa-check_spamd

# Check DKIM status
docker-compose exec mailserver opendkim-testkey -d yourdomain.com -s mail
```

### 2. Maintenance Tasks
```bash
#!/bin/bash
# mail-maintenance.sh

# Cleanup old mails
docker-compose exec mailserver find /var/mail -type f -mtime +90 -delete

# Update SpamAssassin rules
docker-compose exec mailserver sa-update

# Restart services
docker-compose restart mailserver

# Verify services
docker-compose exec mailserver postfix status
docker-compose exec mailserver dovecot status
```

### 3. Backup Mail Data
```bash
#!/bin/bash
# backup-mail.sh

BACKUP_DIR="/path/to/backups/mail"
DATE=$(date +%Y%m%d)

# Create backup directory
mkdir -p "$BACKUP_DIR/$DATE"

# Backup mail data
tar -czf "$BACKUP_DIR/$DATE/mail-data.tar.gz" mailserver/mail-data

# Backup configuration
tar -czf "$BACKUP_DIR/$DATE/mail-config.tar.gz" mailserver/config

# Clean old backups (keep last 30 days)
find "$BACKUP_DIR" -type d -mtime +30 -exec rm -rf {} \;
```

## Troubleshooting

### Common Issues

1. **Email Not Sending**
```bash
# Check mail queue
docker-compose exec mailserver postqueue -p

# Check logs
docker-compose exec mailserver tail -f /var/log/mail/mail.log

# Test SMTP connection
telnet mail.yourdomain.com 25
```

2. **Spam Issues**
```bash
# Check SpamAssassin score
docker-compose exec mailserver spamassassin --test-mode < /path/to/email.txt

# Update spam rules
docker-compose exec mailserver sa-update
```

3. **SSL Problems**
```bash
# Test SSL configuration
openssl s_client -connect mail.yourdomain.com:587 -starttls smtp

# Check certificate
docker-compose exec mailserver certbot certificates
```

### Mail Server Checklist
- [ ] DNS records properly configured
- [ ] SSL certificates installed and valid
- [ ] DKIM keys generated and added to DNS
- [ ] SpamAssassin configured and running
- [ ] Fail2ban active and monitoring
- [ ] Regular backups configured
- [ ] Monitoring in place
- [ ] Test emails successful
- [ ] Pixelfed integration verified
```
