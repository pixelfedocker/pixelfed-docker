# Security Configuration Guide

This guide covers essential security measures and best practices for securing your Pixelfed instance.

## Table of Contents
- [Basic Security Setup](#basic-security-setup)
- [Firewall Configuration](#firewall-configuration)
- [Access Control](#access-control)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Database Security](#database-security)
- [File System Security](#file-system-security)
- [Monitoring & Intrusion Detection](#monitoring--intrusion-detection)
- [Security Updates & Patches](#security-updates--patches)

## Basic Security Setup

### 1. System Hardening Script
```bash
#!/bin/bash
# harden-system.sh

# Update system
apt update && apt upgrade -y

# Install security packages
apt install -y \
    fail2ban \
    ufw \
    unattended-upgrades \
    auditd \
    rkhunter \
    lynis \
    acl \
    apparmor \
    apparmor-utils

# Configure automatic updates
cat > /etc/apt/apt.conf.d/50unattended-upgrades << EOF
Unattended-Upgrade::Allowed-Origins {
    "\${distro_id}:\${distro_codename}-security";
    "\${distro_id}:\${distro_codename}-updates";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
EOF

# Enable automatic updates
dpkg-reconfigure -plow unattended-upgrades

# Secure shared memory
echo "none /run/shm tmpfs defaults,noexec,nosuid 0 0" >> /etc/fstab

# Disable root SSH login
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Restart SSH service
systemctl restart sshd
```

### 2. Docker Security Configuration
```yaml
# docker-compose.yml security additions
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - ./storage:/var/www/storage:rw
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 1m
      timeout: 10s
      retries: 3
```

## Firewall Configuration

### 1. UFW Setup
```bash
#!/bin/bash
# setup-firewall.sh

# Reset UFW
ufw --force reset

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (adjust port if different)
ufw allow 22/tcp

# Allow HTTP/HTTPS
ufw allow 80/tcp
ufw allow 443/tcp

# Allow Mail ports if using mail server
ufw allow 25/tcp
ufw allow 587/tcp
ufw allow 993/tcp

# Enable UFW
ufw --force enable

# Show status
ufw status verbose
```

### 2. Fail2ban Configuration
```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log

[nginx-botsearch]
enabled = true
filter = nginx-botsearch
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2
```

## Access Control

### 1. User Management
```bash
#!/bin/bash
# user-management.sh

# Create service account
useradd -r -s /bin/false pixelfed

# Set proper permissions
chown -R pixelfed:pixelfed /var/www/pixelfed
chmod -R 755 /var/www/pixelfed

# Setup ACLs
setfacl -R -m u:www-data:rx /var/www/pixelfed
setfacl -R -m u:pixelfed:rwx /var/www/pixelfed/storage
```

### 2. SSH Hardening
```bash
# /etc/ssh/sshd_config
Protocol 2
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
MaxAuthTries 3
LoginGraceTime 20
```

## SSL/TLS Configuration

### 1. Strong SSL Configuration
```nginx
# /etc/nginx/conf.d/ssl.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 1.0.0.1 valid=300s;
resolver_timeout 5s;
```

### 2. HSTS Configuration
```nginx
add_header Strict-Transport-Security "max-age=63072000" always;
```

## Database Security

### 1. MySQL Hardening
```sql
-- Remove anonymous users
DELETE FROM mysql.user WHERE User='';

-- Remove remote root
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');

-- Remove test database
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';

-- Set password policy
SET GLOBAL validate_password.policy=STRONG;
SET GLOBAL validate_password.length=12;

-- Flush privileges
FLUSH PRIVILEGES;
```

### 2. Database Backup Security
```bash
#!/bin/bash
# secure-backup.sh

BACKUP_PATH="/path/to/backups"
ENCRYPTION_KEY="/path/to/backup-key.gpg"

# Backup and encrypt database
mysqldump -u root -p"${DB_ROOT_PASSWORD}" pixelfed | \
    gpg --encrypt --recipient backup@yourdomain.com > \
    "$BACKUP_PATH/pixelfed-$(date +%Y%m%d).sql.gpg"

# Set secure permissions
chmod 600 "$BACKUP_PATH"/*.gpg
```

## File System Security

### 1. Storage Permissions
```bash
#!/bin/bash
# secure-storage.sh

# Set proper ownership
chown -R www-data:www-data storage
chmod -R 755 storage

# Secure sensitive directories
chmod 750 storage/app/private
chmod 750 storage/framework/sessions

# Set POSIX ACLs
setfacl -R -m u:www-data:rx storage
setfacl -R -m u:pixelfed:rwx storage
```

### 2. File Monitoring
```bash
#!/bin/bash
# monitor-files.sh

# Install AIDE
apt install -y aide

# Initialize AIDE database
aideinit

# Setup daily checks
cat > /etc/cron.daily/aide-check << EOF
#!/bin/bash
aide --check | mail -s "AIDE Report" admin@yourdomain.com
EOF

chmod +x /etc/cron.daily/aide-check
```

## Monitoring & Intrusion Detection

### 1. Setup Intrusion Detection
```bash
#!/bin/bash
# setup-ids.sh

# Install OSSEC
wget -q -O - https://www.atomicorp.com/RPM-GPG-KEY.atomicorp.txt | apt-key add -
wget -q -O - https://updates.atomicorp.com/installers/atomic | sh

apt update
apt install -y ossec-hids-server

# Configure OSSEC
cat > /var/ossec/etc/ossec.conf << EOF
<ossec_config>
  <global>
    <email_notification>yes</email_notification>
    <email_to>admin@yourdomain.com</email_to>
    <smtp_server>localhost</smtp_server>
    <email_from>ossec@yourdomain.com</email_from>
  </global>

  <rules>
    <include>rules_config.xml</include>
    <include>pam_rules.xml</include>
    <include>sshd_rules.xml</include>
    <include>nginx_rules.xml</include>
  </rules>

  <syscheck>
    <frequency>7200</frequency>
    <directories check_all="yes">/etc,/usr/bin,/usr/sbin</directories>
    <directories check_all="yes">/var/www/pixelfed</directories>
  </syscheck>
</ossec_config>
EOF

# Start OSSEC
systemctl start ossec
```

### 2. Monitoring Script
```bash
#!/bin/bash
# security-monitor.sh

# Check for failed login attempts
grep "Failed password" /var/log/auth.log

# Check for unauthorized sudo usage
grep "sudo:" /var/log/auth.log

# Check for modified system files
aide --check

# Check running processes
ps aux | grep -v '[[]'

# Check open ports
netstat -tulpn

# Check Docker container status
docker ps -a

# Send report
mail -s "Security Report" admin@yourdomain.com
```

## Security Updates & Patches

### 1. Automatic Updates Configuration
```bash
#!/bin/bash
# configure-updates.sh

# Configure unattended upgrades
cat > /etc/apt/apt.conf.d/20auto-upgrades << EOF
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF

# Enable security updates only
cat > /etc/apt/apt.conf.d/50unattended-upgrades << EOF
Unattended-Upgrade::Allowed-Origins {
    "\${distro_id}:\${distro_codename}-security";
};
Unattended-Upgrade::Package-Blacklist {
};
EOF
```

### 2. Security Audit Script
```bash
#!/bin/bash
# security-audit.sh

# Run system security audit
lynis audit system > /var/log/security-audit.log

# Check for rootkits
rkhunter --check --skip-keypress

# Check for vulnerable packages
sudo apt list --upgradable

# Generate report
{
    echo "Security Audit Report"
    echo "===================="
    echo
    echo "Vulnerable Packages:"
    apt list --upgradable
    echo
    echo "Last Failed Login Attempts:"
    last -f /var/log/btmp | head -n 5
    echo
    echo "Lynis Warnings:"
    grep "warning" /var/log/security-audit.log
} | mail -s "Security Audit Report" admin@yourdomain.com
```

### Security Checklist
- [ ] System hardening completed
- [ ] Firewall configured and active
- [ ] Fail2ban installed and running
- [ ] SSL/TLS properly configured
- [ ] Database secured and encrypted backups enabled
- [ ] File permissions properly set
- [ ] Intrusion detection system active
- [ ] Automatic updates configured
- [ ] Regular security audits scheduled
- [ ] Monitoring alerts configured
- [ ] Backup encryption implemented
- [ ] Access controls reviewed
```
