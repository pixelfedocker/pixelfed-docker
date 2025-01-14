```markdown
# Network & SSL Configuration

This guide covers the network setup, domain configuration, and SSL certificate installation for your Pixelfed instance.

## Table of Contents
- [DNS Configuration](#dns-configuration)
- [Nginx Setup](#nginx-setup)
- [SSL Certificates](#ssl-certificates)
- [Security Headers](#security-headers)
- [Performance Optimization](#performance-optimization)

## DNS Configuration

### Required DNS Records

1. **A Record**
```
Type: A
Host: pixelfed
Value: YOUR_SERVER_IP
TTL: 3600
```

2. **AAAA Record** (if IPv6 is available)
```
Type: AAAA
Host: pixelfed
Value: YOUR_IPv6_ADDRESS
TTL: 3600
```

3. **WWW Subdomain** (optional)
```
Type: CNAME
Host: www
Value: pixelfed.yourdomain.com
TTL: 3600
```

### Verify DNS Propagation
```bash
# Check A record
dig pixelfed.yourdomain.com A

# Check AAAA record (if applicable)
dig pixelfed.yourdomain.com AAAA

# Check DNS propagation
curl -I pixelfed.yourdomain.com
```

## Nginx Setup

### 1. Basic Configuration
Create `/etc/nginx/conf.d/pixelfed.conf`:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name pixelfed.yourdomain.com;
    root /var/www/public;
    
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    # Media handling
    location /storage {
        alias /var/www/storage/app/public;
        try_files $uri $uri/ =404;
        expires 1y;
        access_log off;
        add_header Cache-Control "public";
    }

    # Deny access to sensitive files
    location ~ \.(env|log|git|yml|xml|json)$ {
        deny all;
    }
}
```

### 2. Test Configuration
```bash
# Test Nginx configuration
sudo nginx -t

# Restart Nginx if test succeeds
sudo systemctl restart nginx
```

## SSL Certificates

### 1. Install Certbot
```bash
# Install Certbot and Nginx plugin
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

### 2. Obtain SSL Certificate
```bash
# Stop Nginx
sudo systemctl stop nginx

# Get certificate
sudo certbot certonly --standalone \
    -d pixelfed.yourdomain.com \
    --agree-tos \
    --email your@email.com \
    --rsa-key-size 4096

# Start Nginx
sudo systemctl start nginx
```

### 3. SSL-Enabled Nginx Configuration
Update `/etc/nginx/conf.d/pixelfed.conf`:
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name pixelfed.yourdomain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/pixelfed.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pixelfed.yourdomain.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/pixelfed.yourdomain.com/chain.pem;

    # SSL Security Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 1.0.0.1 valid=300s;
    resolver_timeout 5s;

    # Root directory and index
    root /var/www/public;
    index index.php;

    # Previous location blocks remain the same
    # ... (copy from basic configuration)
}

# HTTP redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name pixelfed.yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

## Security Headers

### 1. Add Security Headers
Add these inside the HTTPS server block:
```nginx
# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

## Performance Optimization

### 1. Enable Compression
Add to the HTTPS server block:
```nginx
# Compression
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;
```

### 2. Browser Caching
Add to appropriate location blocks:
```nginx
# Static files caching
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg)$ {
    expires 365d;
    add_header Cache-Control "public, no-transform";
}
```

### 3. FastCGI Caching
Add before server blocks:
```nginx
# FastCGI cache settings
fastcgi_cache_path /tmp/nginx_cache levels=1:2 keys_zone=PIXELFED:100m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout http_500 http_503;
fastcgi_cache_valid 200 60m;
```

## SSL Maintenance

### 1. Auto-renewal Setup
```bash
# Test auto-renewal
sudo certbot renew --dry-run

# Check renewal timer
sudo systemctl status certbot.timer
```

### 2. SSL Monitoring Script
Create `scripts/check-ssl.sh`:
```bash
#!/bin/bash

DOMAIN="pixelfed.yourdomain.com"
EXPIRY=$(date -d "$(openssl s_client -connect ${DOMAIN}:443 \
    -servername ${DOMAIN} 2>/dev/null \
    | openssl x509 -text \
    | grep "Not After" \
    | cut -d: -f2-)" +%s)

NOW=$(date +%s)
DAYS=$(( ($EXPIRY - $NOW) / 86400 ))

if [ $DAYS -lt 30 ]; then
    echo "WARNING: SSL certificate for ${DOMAIN} expires in ${DAYS} days"
    # Add notification logic here
fi
```

## Troubleshooting

### Common Issues

1. **Certificate Renewal Failures**
```bash
# Check Certbot logs
sudo journalctl -u certbot

# Manual renewal attempt
sudo certbot renew --force-renewal
```

2. **Nginx Configuration Errors**
```bash
# Check syntax
sudo nginx -t

# Check error logs
sudo tail -f /var/log/nginx/error.log
```

3. **SSL Connection Issues**
```bash
# Test SSL connection
openssl s_client -connect pixelfed.yourdomain.com:443 -servername pixelfed.yourdomain.com

# Check SSL configuration
curl -I https://pixelfed.yourdomain.com
```

### SSL Best Practices Checklist
- [ ] Use strong SSL protocols (TLS 1.2, 1.3)
- [ ] Enable HSTS
- [ ] Configure proper security headers
- [ ] Set up automatic renewals
- [ ] Monitor certificate expiration
- [ ] Keep Nginx and Certbot updated
- [ ] Regular security audits
```
