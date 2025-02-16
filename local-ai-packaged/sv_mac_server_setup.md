# Complete Guide: Exposing Open WebUI to Internet (macOS)

## Overview
A comprehensive guide to securely expose Open WebUI to the internet from your macOS system using Nginx as a reverse proxy with SSL encryption.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Directory Setup](#directory-setup)
4. [Domain Configuration](#domain-configuration)
5. [Nginx Configuration](#nginx-configuration)
6. [SSL Certificate Setup](#ssl-certificate-setup)
7. [Service Management](#service-management)
8. [Security and Maintenance](#security-and-maintenance)
9. [Troubleshooting](#troubleshooting)

## Prerequisites
- macOS system
- Docker Desktop installed and running
- Administrative access
- A registered domain name with DNS access
- Open WebUI running in Docker (port 3000:8080)

## Installation

### 1. Install Required Software
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Nginx and Certbot
brew install nginx
brew install certbot
```

## Directory Setup

### 1. Create Required Directories
```bash
# Create necessary directories
sudo mkdir -p /opt/homebrew/etc/nginx/conf.d
sudo mkdir -p /opt/homebrew/var/log/nginx
sudo mkdir -p /opt/homebrew/var/run/nginx

# Set correct permissions
sudo chown -R $(whoami) /opt/homebrew/etc/nginx
sudo chown -R $(whoami) /opt/homebrew/var/log/nginx
sudo chown -R $(whoami) /opt/homebrew/var/run/nginx
sudo chown -R $(whoami) /opt/homebrew/Cellar/nginx
sudo chown -R $(whoami) /opt/homebrew/opt/nginx
sudo chown -R $(whoami) /opt/homebrew/var/homebrew/linked/nginx
```

## Domain Configuration

### 1. Get Server IP
```bash
curl ifconfig.me
```

### 2. Configure DNS
Add an A record in your domain provider's DNS settings:
- Type: A
- Host: your desired subdomain (e.g., 'openwebui')
- Value: Your server's public IP
- TTL: Lowest available (typically 300 seconds)

### 3. Verify DNS Propagation
```bash
dig your-subdomain.yourdomain.com
```

## Nginx Configuration

### 1. Create Main Configuration
```bash
# Backup existing config if present
cp /opt/homebrew/etc/nginx/nginx.conf /opt/homebrew/etc/nginx/nginx.conf.backup 2>/dev/null

# Create new main config
cat > /opt/homebrew/etc/nginx/nginx.conf <<EOL
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    client_max_body_size 0;

    sendfile        on;
    keepalive_timeout  65;

    # WebSocket support
    map \$http_upgrade \$connection_upgrade {
        default upgrade;
        ''      close;
    }

    include /opt/homebrew/etc/nginx/conf.d/*.conf;
}
EOL
```

### 2. Create Open WebUI Configuration
```bash
# Replace ACTUAL_DOMAIN with your domain (e.g., open-webui.dcphotozon.com)
DOMAIN="ACTUAL_DOMAIN"

cat > /opt/homebrew/etc/nginx/conf.d/openwebui.conf <<EOL
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;  # Your actual domain

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection \$connection_upgrade;
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        # WebSocket timeout settings
        proxy_read_timeout 300;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
    }
}
EOL
```

### 3. Enable Port 80/443 Access
Since we're running nginx as a non-root user, we need to allow it to bind to privileged ports:
```bash
sudo tee /Library/LaunchDaemons/com.nginx.privileged-ports.plist > /dev/null <<EOL
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nginx.privileged-ports</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/sh</string>
        <string>-c</string>
        <string>echo "net.inet.ip.portrange.reservedhigh=0" | sudo tee /etc/sysctl.conf && sudo sysctl -w net.inet.ip.portrange.reservedhigh=0</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOL

sudo launchctl load -w /Library/LaunchDaemons/com.nginx.privileged-ports.plist
```

### 4. Test Configuration
```bash
nginx -t
```

## Port Forwarding Setup

### 1. Get Your Local IP
```bash
# Find your local IP address
ifconfig | grep "inet " | grep -v 127.0.0.1
```

### 2. Configure Router Port Forwarding
1. Access your router's admin interface
2. Locate port forwarding settings
3. Add two port forwarding rules:
   - Forward port 80 to your local IP
   - Forward port 443 to your local IP

### 3. Verify Port Configuration
```bash
# Find your local IP address
ifconfig | grep "inet " | grep -v 127.0.0.1
# Expected output something like:
# inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255

# Check if nginx is listening on port 80
sudo lsof -i :80
# Expected output something like:
# COMMAND  PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
# nginx   1234 user    6u  IPv4 0xc123456789012345      0t0  TCP *:http (LISTEN)

# Test local port 80 access
nc -zv localhost 80
# Expected output:
# Connection to localhost port 80 [tcp/http] succeeded!

# Optional: Check all listening ports
netstat -an | grep LISTEN
# Should show entries for port 80 and 443 like:
# tcp4       0      0  *.80      *.*    LISTEN
```

✅ If you see similar outputs, your local configuration is correct.
❌ If outputs differ, check nginx status and port configurations.

After verifying local access, test internet accessibility:
1. Visit: https://www.yougetsignal.com/tools/open-ports/
2. Enter your public IP
3. Check ports 80 and 443
4. Status should show as "Open" for both ports

### 4. Prepare and Obtain SSL Certificate
```bash
# Create symbolic link for certbot to find nginx configuration
sudo mkdir -p /usr/local/etc
sudo ln -s /opt/homebrew/etc/nginx /usr/local/etc/nginx

# Verify the symbolic link
ls -l /usr/local/etc/nginx

# Obtain the certificate
sudo certbot --nginx -d $DOMAIN  # Use your actual domain (e.g., open-webui.dcphotozon.com)
```

When running the certbot command, you'll be prompted for:
1. An email address for:
   - Certificate expiration notifications
   - Security alerts
   - Issues with your certificates
2. Agreement to Let's Encrypt's Terms of Service
3. Whether to share your email with EFF (Electronic Frontier Foundation)

Example response to email prompt:
```
your.email@example.com
```


### 3. Fix Certbot Permissions
```bash
sudo chown -R $(whoami) /opt/homebrew/etc/nginx
```

## Service Management

### 1. Start Nginx
```bash
brew services start nginx
```

### 2. Verify Status
```bash
brew services list
curl -I http://localhost
```

## Security and Maintenance

### 1. Regular Updates
```bash
# Update software
brew update
brew upgrade nginx certbot

# Update Docker containers
docker-compose pull
docker-compose up -d
```

### 2. Certificate Renewal
Add to your user's crontab (not root's):
```bash
(crontab -l 2>/dev/null; echo "0 0 1 * * /opt/homebrew/bin/certbot renew --quiet") | crontab -
```

### 3. Backup Configuration
```bash
# Create backup directory
mkdir -p ~/nginx-backups
cp -r /opt/homebrew/etc/nginx ~/nginx-backups/nginx-$(date +%Y%m%d)
```

## Troubleshooting

### Common Issues

1. **Permission Denied**
```bash
# Fix ownership
sudo chown -R $(whoami) /opt/homebrew/etc/nginx
sudo chown -R $(whoami) /opt/homebrew/var/log/nginx
sudo chown -R $(whoami) /opt/homebrew/var/run/nginx
```

2. **Port Binding Issues**
```bash
# Verify port settings
sudo sysctl net.inet.ip.portrange.reservedhigh
# Should return 0

# Check port usage
sudo lsof -i :80
sudo lsof -i :443
```

3. **SSL Certificate Issues**
```bash
# Test renewal
certbot renew --dry-run
```

4. **502 Bad Gateway**
```bash
# Check logs
tail -f /opt/homebrew/var/log/nginx/error.log

# Verify Open WebUI is running
docker ps
curl http://localhost:3000
```

## Additional Notes
- Keep regular backups of configurations
- Monitor logs for unusual activity
- Check certificate expiration dates regularly
- Test configuration changes in a staging environment

## Version History
v1.1 - Updated (2024-02-16)
- Added proper permission handling
- Added port binding solution for non-root nginx
- Improved security configurations
- Enhanced troubleshooting section

v1.0 - Initial Release (2024-02-16)