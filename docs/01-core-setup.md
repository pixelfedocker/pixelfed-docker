```markdown
# Core Setup Guide

This guide covers the basic setup of your Pixelfed instance, including system preparation, Docker installation, and initial configuration.

## Table of Contents
- [System Requirements](#system-requirements)
- [Basic System Setup](#basic-system-setup)
- [Docker Installation](#docker-installation)
- [Pixelfed Configuration](#pixelfed-configuration)
- [Directory Structure](#directory-structure)

## System Requirements

### Minimum Specifications
- CPU: 2 cores
- RAM: 2GB
- Storage: 20GB
- OS: Ubuntu 24.04 LTS or newer
- Domain name pointing to your server

### Recommended Specifications
- CPU: 4 cores
- RAM: 4GB
- Storage: 40GB+ (depends on media storage needs)
- SSD storage for better performance

## Basic System Setup

### 1. Update System
```bash
# Update package list and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y \
    curl \
    git \
    nginx \
    ufw \
    certbot \
    python3-certbot-nginx \
    fail2ban \
    unzip
```

### 2. Configure System Settings
```bash
# Set system hostname
sudo hostnamectl set-hostname pixelfed.yourdomain.com

# Set timezone
sudo timedatectl set-timezone UTC

# Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

## Docker Installation

### 1. Install Docker
```bash
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Add current user to docker group
sudo usermod -aG docker $USER

# IMPORTANT: Log out and log back in for group changes to take effect
```

### 2. Install Docker Compose
```bash
# Download latest version
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose

# Make executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
docker --version
docker-compose --version
```

## Pixelfed Configuration

### 1. Create Directory Structure
```bash
# Create main directory
mkdir pixelfed
cd pixelfed

# Create required subdirectories
mkdir -p {storage,database,nginx,redis}
```

### 2. Create Docker Compose File
Create `docker-compose.yml`:
```yaml
version: '3'
services:
  app:
    image: pixelfed/pixelfed:latest
    restart: always
    env_file: .env
    volumes:
      - ./storage:/var/www/storage
      - ./php.ini:/usr/local/etc/php/php.ini:ro
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: pixelfed
      MYSQL_USER: pixelfed
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - ./database:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password

  redis:
    image: redis:alpine
    restart: always
    volumes:
      - ./redis:/data

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx:/etc/nginx/conf.d
      - ./storage/app/public:/var/www/storage/app/public:ro
    depends_on:
      - app
```

### 3. Create Environment File
Create `.env` file:
```env
# Application
APP_NAME=Pixelfed
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://pixelfed.yourdomain.com
APP_DOMAIN="yourdomain.com"

# Database
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=pixelfed
DB_USERNAME=pixelfed
DB_PASSWORD=your-secure-password
DB_ROOT_PASSWORD=your-secure-root-password

# Redis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

# Pixelfed Specific
OPEN_REGISTRATION=true
ENFORCE_EMAIL_VERIFICATION=true
PF_MAX_USERS=1000
```

### 4. Initialize Application
```bash
# Start services
docker-compose up -d

# Generate app key
docker-compose exec app php artisan key:generate

# Run migrations
docker-compose exec app php artisan migrate --force

# Create admin user
docker-compose exec app php artisan user:create
```

## Directory Structure
After setup, your directory should look like this:
```
pixelfed/
├── docker-compose.yml
├── .env
├── storage/
│   ├── app/
│   ├── framework/
│   └── logs/
├── database/
├── nginx/
│   └── pixelfed.conf
├── redis/
└── php.ini
```

## Next Steps
- Proceed to [Network & SSL Setup](02-network-ssl.md) to configure your domain and SSL certificates
- Configure email settings for user notifications
- Set up backup procedures

## Troubleshooting

### Common Issues
1. **Permission Issues**
```bash
# Fix storage permissions
docker-compose exec app chown -R www-data:www-data storage
```

2. **Database Connection Issues**
```bash
# Check database connection
docker-compose exec app php artisan db:show
```

3. **Container Status**
```bash
# Check container status
docker-compose ps

# View container logs
docker-compose logs -f app
```

### Getting Help
- Check the [official documentation](https://docs.pixelfed.org/)
- Join the [Discord community](https://discord.gg/pixelfed)
- Search [GitHub Issues](https://github.com/pixelfed/pixelfed/issues)
```
