# How to Turn Your Computer into a Web Server with Docker

## 📋 Table of Contents
1. [Introduction](#introduction)
2. [Requirements](#requirements)
3. [Docker Container Setup](#docker-container-setup)
4. [Nginx Reverse Proxy](#nginx-reverse-proxy)
5. [Connect Domain](#connect-domain)
6. [Router Port Forwarding](#router-port-forwarding)
7. [SSL Certificate (HTTPS)](#ssl-certificate-https)
8. [Security Settings](#security-settings)

---

## Introduction

This guide shows you how to turn your local computer into a web server using Docker, and make it accessible from the internet with your domain name.

**Example Project:** Node.js API + MySQL Database + Nginx Reverse Proxy

---

## Requirements

### Software
- Docker & Docker Compose
- Linux/Ubuntu (or Windows/macOS)
- Text editor (VS Code recommended)

### Network
- Internet connection
- Router access (for port forwarding)
- Domain name (optional, you can use IP address)

---

## Docker Container Setup

### 1. Project Structure

```
project/
├── docker-compose.yml
├── api/
│   ├── Dockerfile
│   ├── index.js
│   └── package.json
├── etc/
│   └── nginx/
│       └── nginx.conf
└── init/
    └── 01-create-db.sql
```

### 2. Docker Compose File

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # Database
  mysql:
    image: mysql:8.0
    container_name: app-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"  # Local only, not exposed externally
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    volumes:
      - db_data:/var/lib/mysql
      - ./init:/docker-entrypoint-initdb.d

  # Application (Node.js API)
  api:
    build: ./api
    container_name: app-api
    restart: unless-stopped
    environment:
      DB_HOST: app-mysql
      DB_PORT: 3306
      DB_USER: appuser
      DB_PASS: apppass
      DB_NAME: appdb
    depends_on:
      - mysql

  # Nginx Reverse Proxy
  nginx:
    image: nginx:stable-alpine
    container_name: app-nginx
    restart: unless-stopped
    ports:
      - "80:80"      # HTTP
      - "443:443"    # HTTPS
    depends_on:
      - api
    volumes:
      - ./etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro  # SSL certificates

volumes:
  db_data:
```

### 3. Node.js API Dockerfile

**api/Dockerfile:**
```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install --production

# Copy application code
COPY . .

EXPOSE 3000
CMD ["node", "index.js"]
```

### 4. Simple API Code

**api/index.js:**
```javascript
const express = require('express');
const mysql = require('mysql2/promise');

const app = express();
app.use(express.json());

const pool = mysql.createPool({
  host: process.env.DB_HOST || 'app-mysql',
  port: process.env.DB_PORT || 3306,
  user: process.env.DB_USER || 'appuser',
  password: process.env.DB_PASS || 'apppass',
  database: process.env.DB_NAME || 'appdb',
  connectionLimit: 5
});

// Homepage
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>My API Server</title>
      <meta name="viewport" content="width=device-width, initial-scale=1">
    </head>
    <body>
      <h1>🚀 API Server Running</h1>
      <p>Server is online and working!</p>
    </body>
    </html>
  `);
});

// Database test endpoint
app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.json({ status: 'ok', database: 'connected' });
  } catch (err) {
    res.status(500).json({ status: 'error', message: err.message });
  }
});

const port = 3000;
app.listen(port, () => console.log(`API listening on ${port}`));
```

**api/package.json:**
```json
{
  "name": "api",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0",
    "mysql2": "^3.6.0"
  }
}
```

---

## Nginx Reverse Proxy

### Nginx Configuration

**etc/nginx/nginx.conf:**
```nginx
events {}
http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # Rate Limiting (DDoS Protection)
  limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
  limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

  upstream api_up {
    server app-api:3000;
  }

  # HTTP - redirect to HTTPS
  server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
  }

  # HTTPS
  server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Connection limit
    limit_conn conn_limit 10;

    location / {
      # Rate limiting: 10 requests per second
      limit_req zone=api_limit burst=20 nodelay;
      
      proxy_pass http://api_up;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

### Start Containers

```bash
# Build and start containers
docker compose up -d --build

# Check status
docker ps

# View logs
docker compose logs -f
```

---

## Connect Domain

### 1. Find Your Public IP

```bash
curl ifconfig.me
# Output: 123.45.67.89 (example)
```

### 2. Domain DNS Settings

Go to your domain provider's (GoDaddy, Namecheap, etc.) DNS management panel:

**Add A Record:**
```
Type: A
Host: @
Value: 123.45.67.89 (your public IP)
TTL: 3600
```

**For www subdomain:**
```
Type: A
Host: www
Value: 123.45.67.89
TTL: 3600
```

DNS changes can take **15 minutes to 48 hours** to propagate.

### 3. Test

```bash
# DNS propagation check
nslookup yourdomain.com

# Ping test
ping yourdomain.com
```

---

## Router Port Forwarding

### 1. Find Your Local IP

```bash
hostname -I | awk '{print $1}'
# Output: 192.168.2.27 (örnek)
```

### 2. Router Settings

Go to your router's admin panel (usually `192.168.1.1` or `192.168.2.1`)

**Add Port Forwarding Rules:**

| Service | Protocol | External Port | Internal IP    | Internal Port | Status  |
|---------|----------|---------------|----------------|---------------|---------|
| HTTP    | TCP      | 80            | 192.168.2.27   | 80            | Enabled |
| HTTPS   | TCP      | 443           | 192.168.2.27   | 443           | Enabled |

### 3. Test

Use online port checker: https://www.yougetsignal.com/tools/open-ports/

```
IP Address: 123.45.67.89
Port: 80
```

If it says "Open", you're successful! 🎉

---

## SSL Certificate (HTTPS)

### 1. Install Certbot

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y certbot

# Verify installation
certbot --version
```

### 2. Stop Nginx Container

```bash
docker stop app-nginx
```

### 3. Get SSL Certificate

```bash
sudo certbot certonly --standalone -d yourdomain.com

# Follow prompts:
# - Enter email address
# - Agree to Terms of Service
# - Share email with EFF (optional)
```

**Certificates are saved at:**
```
Certificate: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Private Key: /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

### 4. Start Nginx Again

```bash
docker compose up -d
```

### 5. Test

```bash
# HTTPS Test
curl -I https://yourdomain.com

# Should return: HTTP/1.1 200 OK
```

### 6. Auto Renewal

Certbot creates an auto-renewal timer. Check it:

```bash
sudo systemctl status certbot.timer
```

Test manual renewal:

```bash
sudo certbot renew --dry-run
```

**Note:** Certificates are valid for 90 days and auto-renew.

---

## Security Settings

### 1. Rate Limiting ✅

Already added in Nginx config:
- 10 requests per second limit
- Maximum 10 concurrent connections

### 2. Security Headers ✅

- X-Frame-Options (Clickjacking protection)
- X-Content-Type-Options (MIME sniffing protection)
- X-XSS-Protection (XSS protection)

### 3. Database Port Closed ✅

MySQL port (3306) only accessible locally, not exposed externally.

### 4. SSL/HTTPS ✅

All traffic is encrypted.

### 5. Additional Security Tips

**Enable Firewall:**
```bash
# UFW (Ubuntu)
sudo ufw enable
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp  # SSH
sudo ufw status
```

**Keep Docker Containers Updated:**
```bash
# Pull latest images
docker compose pull

# Rebuild
docker compose up -d --build
```

**Monitor Logs:**
```bash
# Nginx logs
docker logs app-nginx -f

# API logs
docker logs app-api -f

# All logs
docker compose logs -f
```

---

## Troubleshooting

### Can't Access Port 80/443

1. Check router port forwarding
2. Check ISP port blocking (some ISPs block port 80)
3. Check firewall

```bash
# Check port
sudo netstat -tulpn | grep :80
```

### Domain Not Working

1. Wait for DNS propagation (up to 48 hours)
2. Check DNS settings
3. Clear cache

```bash
# DNS flush (Ubuntu)
sudo systemd-resolve --flush-caches

# Test
nslookup yourdomain.com
```

### SSL Error

1. Check certificate paths
2. Test Nginx config

```bash
# Nginx config test
docker exec app-nginx nginx -t

# Reload nginx
docker exec app-nginx nginx -s reload
```

### Container Not Running

```bash
# Check status
docker ps -a

# View logs
docker logs app-api

# Restart
docker compose restart

# Full rebuild
docker compose down
docker compose up -d --build
```

---

## Conclusion

By following this guide, you have:

✅ Set up a containerized application with Docker  
✅ Configured Nginx reverse proxy  
✅ Connected your domain name  
✅ Set up port forwarding on router  
✅ Added free SSL certificate  
✅ Implemented security measures  
✅ Turned your local computer into a web server  

**Your application is now accessible worldwide!** 🌍

---

## Useful Commands

```bash
# Docker
docker compose up -d              # Start containers
docker compose down               # Stop containers
docker compose restart            # Restart all
docker compose logs -f            # View logs
docker ps                         # List running containers
docker exec -it app-api sh        # Enter container shell

# SSL
sudo certbot certificates         # List certificates
sudo certbot renew                # Renew certificates
sudo certbot delete               # Delete certificate

# Network
curl ifconfig.me                  # Public IP
hostname -I                       # Local IP
netstat -tulpn                    # Open ports
ping yourdomain.com               # Test domain

# System
sudo systemctl status docker      # Docker service status
sudo systemctl restart docker     # Restart Docker
df -h                             # Disk usage
free -h                           # Memory usage
```

---

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Docker Compose](https://docs.docker.com/compose/)

---

**Author:** Sezgin Yurdakul  
**Date:** November 10, 2025  

---

## License

This document is free to use, share, and modify.
