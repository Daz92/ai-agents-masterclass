# Complete Guide: Exposing Open WebUI to Internet (macOS)

## Overview
This guide describes how to securely expose Open WebUI to the internet from your macOS system using Nginx as a reverse proxy with SSL encryption.

## Prerequisites
- macOS system
- Docker Desktop installed and running
- Administrative access
- A registered domain name with DNS access
- Open WebUI running in Docker (port 3000)

## Common Issues & Solutions
1. **SSL Certificate Permission Issues**
   - Symptoms: nginx unable to read SSL certificates
   - Solution: Correct permissions for SSL directories and files
   ```bash
   sudo chmod 755 /etc/letsencrypt/live
   sudo chmod 755 /etc/letsencrypt/archive
   sudo chmod 755 /etc/letsencrypt/live/your-domain.com
   sudo chmod 755 /etc/letsencrypt/archive/your-domain.com
   sudo chmod 644 /etc/letsencrypt/archive/your-domain.com/*.pem
   ```

2. **Port Conflicts**
   - Symptoms: "Address already in use" errors
   - Solution: Check and kill processes using required ports
   ```bash
   sudo lsof -i :8081
   sudo lsof -i :8082
   sudo pkill -f nginx
   ```

3. **Router Configuration**
   - Symptoms: Site unreachable from external networks
   - Solution: Proper port forwarding in router
     - Forward external port 80 → internal port 8082
     - Forward external port 443 → internal port 8081
     - Use correct local IP (check with `ifconfig`)

## Step-by-Step Setup

### 1. Initial Cleanup
```bash
# Stop any running nginx
brew services stop nginx

# Kill any nginx processes
sudo pkill -f nginx

# Remove old configurations
rm -rf /opt/homebrew/etc/nginx/nginx.conf
rm -rf /opt/homebrew/etc/nginx/conf.d/*

# Clean up SSL related files (if any)
sudo rm -rf /etc/letsencrypt

# Remove any port forwarding rules
sudo pfctl -F all -f /etc/pf.conf

# Clean up SSL related files (if any)
sudo rm -rf /etc/letsencrypt
```

### 2. Install Required Software
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Nginx and Certbot
brew install nginx
brew install certbot
```

### 3. Prepare Directory Structure
```bash
sudo mkdir -p /opt/homebrew/var/log/nginx
sudo mkdir -p /opt/homebrew/var/run/nginx
sudo mkdir -p /opt/homebrew/etc/nginx/conf.d

sudo chown -R $(whoami) /opt/homebrew/var/log/nginx
sudo chown -R $(whoami) /opt/homebrew/var/run/nginx
sudo chown -R $(whoami) /opt/homebrew/etc/nginx
```

### 4. Configure Nginx
```bash
cat > /opt/homebrew/etc/nginx/nginx.conf <<EOL
worker_processes  1;
error_log  /opt/homebrew/var/log/nginx/error.log;
pid        /opt/homebrew/var/run/nginx/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    client_max_body_size 0;

    access_log  /opt/homebrew/var/log/nginx/access.log;
    error_log   /opt/homebrew/var/log/nginx/error.log;

    sendfile        on;
    keepalive_timeout  65;

    # WebSocket support
    map \$http_upgrade \$connection_upgrade {
        default upgrade;
        ''      close;
    }

    # HTTP Server (Redirect to HTTPS)
    server {
        listen       8082;
        server_name  your-domain.com;
        return 301 https://\$server_name\$request_uri;
    }

    # HTTPS Server (SSL configuration will be added by certbot)
    server {
        listen       8081;           # certbot will modify this to include ssl
        server_name  your-domain.com;

        # Note: SSL certificate paths will be automatically added by certbot
        # when running: sudo certbot --nginx -d your-domain.com

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
        }
    }
}
EOL
```

### 5. Set Up Port Forwarding
```bash
# Create port forwarding rules
sudo tee /etc/pf.anchors/nginx <<EOL
rdr pass on lo0 inet proto tcp from any to any port 80 -> 127.0.0.1 port 8082
rdr pass on lo0 inet proto tcp from any to any port 443 -> 127.0.0.1 port 8081
EOL

# Enable the rules
sudo pfctl -F all
sudo pfctl -ef /etc/pf.anchors/nginx
```

### 6. Configure SSL with Certbot
```bash
sudo certbot --nginx -d your-domain.com
```

### 7. Configure Router
1. Get your local IP:
```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
# Note the IP address on your main network (usually 192.168.1.x)
```

2. Access router admin interface (typically 192.168.1.1)
3. Add port forwarding rules:
   - External Port: 80 → Internal IP: [Your Local IP], Internal Port: 8082
   - External Port: 443 → Internal IP: [Your Local IP], Internal Port: 8081

### 8. Verify Setup
```bash
# Check nginx configuration
nginx -t

# Start nginx
brew services start nginx

# Verify ports are listening
sudo lsof -i -P | grep nginx

# Test local access
curl http://localhost:8082
curl -k https://localhost:8081

# Check certificate status
sudo certbot certificates
```

## Troubleshooting

### Common Error Messages and Solutions

1. **"Address already in use"**
```bash
sudo lsof -i :8081  # Check what's using the port
sudo lsof -i :8082
sudo pkill -f nginx  # Kill nginx processes
brew services restart nginx
```

2. **SSL Certificate Issues**
```bash
# Check certificate permissions
sudo ls -l /etc/letsencrypt/live/your-domain.com/
sudo ls -l /etc/letsencrypt/archive/your-domain.com/

# Fix permissions
sudo chmod -R 755 /etc/letsencrypt/live
sudo chmod -R 755 /etc/letsencrypt/archive
```

3. **Site Unreachable**
- Verify router port forwarding
- Check nginx logs: `tail -f /opt/homebrew/var/log/nginx/error.log`
- Verify DNS propagation: `dig your-domain.com`
- Check port status: Use online port checker tool

## Maintenance
- Update software: `brew upgrade nginx certbot`
- Renew certificate: `sudo certbot renew`
- Monitor logs: `tail -f /opt/homebrew/var/log/nginx/error.log`

## Version History
v3.0 - (2024-02-16): Added comprehensive troubleshooting guide
v2.2 - (2024-02-16): Updated Nginx configuration with improved WebSocket support
v2.1 - (2024-02-16): Reordered steps for logical flow
v2.0 - (2024-02-16): Added cleanup section
v1.0 - Initial Release