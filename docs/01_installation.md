# Installation

**Language**: English | [简体中文](./01_installation_CN.md) | [繁體中文](./01_installation_TW.md) | [Русский](./01_installation_RU.md) | [فارسی](./01_installation_FA.md)

## System Requirements

### Hardware Requirements

| Item | Minimum | Recommended |
|------|---------|-------------|
| CPU | 1 core | 2 cores+ |
| RAM | 1 GB | 2 GB+ |
| Disk | 10 GB | 20 GB+ SSD |

### Software Requirements

- **Operating System**: Linux x86_64 / aarch64 (Arch Linux / Debian 12 / Ubuntu 22.04 recommended)
- **Database**: MySQL 8.0+ or MariaDB 10.5+
- **Cache**: Redis 6.0+ (optional, for performance optimization)

---

## Pre-Installation

### 1. Create a Database

EzPanel does not create the database automatically — you must do it manually:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE ezpanel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'ezpanel'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON ezpanel.* TO 'ezpanel'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 2. Install Redis (Optional)

Redis is an optional component. Enabling it improves system performance (caching, global device limits, etc.).

```bash
# Debian / Ubuntu
sudo apt install redis-server
sudo systemctl enable --now redis-server

# Arch Linux
sudo pacman -S redis
sudo systemctl enable --now redis
```

---

## Installing EzPanel

### Method 1: Arch Linux Package

```bash
# Download the package
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-x86_64.pkg.tar.zst

# Install
sudo pacman -U ezpanel-*.pkg.tar.zst
```

### Method 2: Debian / Ubuntu Package

```bash
# Download the package
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel_amd64.deb

# Install
sudo dpkg -i ezpanel_*.deb
sudo apt-get install -f
```

### Method 3: Generic Binary (Tarball)

For other Linux distributions:

```bash
# Download and extract
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-linux-amd64.tar.gz
tar -xzf ezpanel-linux-amd64.tar.gz
cd ezpanel-*

# Install the binary
sudo cp ezpanel /usr/bin/
sudo chmod +x /usr/bin/ezpanel

# Create config, runtime, and data directories
sudo mkdir -p /etc/ezpanel /run/ezpanel /var/lib/ezpanel /var/log/ezpanel

# Create a system user
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/ezpanel ezpanel
sudo chown ezpanel:ezpanel /var/lib/ezpanel /var/log/ezpanel /run/ezpanel

# Copy the config file
sudo cp config.yaml.example /etc/ezpanel/config.yaml
sudo chown ezpanel:ezpanel /etc/ezpanel/config.yaml
sudo chmod 600 /etc/ezpanel/config.yaml

# Install the systemd service
sudo cp ezpanel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## Initial Configuration

Edit `/etc/ezpanel/config.yaml` and fill in at minimum the database connection details and the JWT secret:

```yaml
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"   # Replace with your actual password
  dbname: "ezpanel"

jwt:
  secret: "change-this-to-a-random-secret"  # Must be changed!
  expire: 168
```

For detailed configuration options, see [Configuration](./02_configuration.md).

---

## Starting the Service

```bash
sudo systemctl enable --now ezpanel

# Check the running status
sudo systemctl status ezpanel

# View startup logs
sudo journalctl -u ezpanel -f
```

On first startup, GORM will automatically create all database tables and initialize the default administrator account.

**Default Administrator**:
- Email: `admin@opine.work`
- Password: `admin123`

> **Security Notice**: Please change the default password and email immediately after logging in.

---

## Configuring Nginx as a Reverse Proxy (Recommended)

In production, it is recommended to deploy Nginx in front of EzPanel to enable HTTPS and static asset optimization.

```nginx
server {
    listen 80;
    server_name panel.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name panel.example.com;

    ssl_certificate     /etc/letsencrypt/live/panel.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/panel.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Static asset caching
    location ~* \.(js|css|png|jpg|svg|woff2?)$ {
        proxy_pass http://127.0.0.1:7088;
        proxy_set_header Host $host;
        expires 7d;
        add_header Cache-Control "public";
    }

    location / {
        proxy_pass http://127.0.0.1:7088;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Obtaining an SSL Certificate

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com
```

---

## Upgrading

### Arch Linux / Debian Package Upgrade

```bash
# Download the new version package, then reinstall
sudo pacman -U ezpanel-new-version.pkg.tar.zst    # Arch
sudo dpkg -i ezpanel_new_version_amd64.deb         # Debian
```

The service will restart automatically after installation.

### Binary Upgrade

```bash
# Stop the service
sudo systemctl stop ezpanel

# Back up the old version
sudo cp /usr/bin/ezpanel /usr/bin/ezpanel.bak

# Replace with the new version
sudo cp ezpanel-new /usr/bin/ezpanel
sudo chmod +x /usr/bin/ezpanel

# Start the service
sudo systemctl start ezpanel
sudo journalctl -u ezpanel -f  # Watch the startup logs
```

---

## Installing the Pro / Enterprise Edition

The Pro edition runs as a separate process and communicates with the main EzPanel process via a Unix socket.

### Prerequisites

- EzPanel Community edition is already running normally
- You have a valid `license.json` license file

### Installation Steps

```bash
# Install ezpanel-pro (same method as EzPanel itself)
sudo dpkg -i ezpanel-pro_*.deb    # Debian/Ubuntu
sudo pacman -U ezpanel-pro-*.pkg.tar.zst  # Arch

# Place the license file
sudo cp license.json /etc/ezpanel/license.json
sudo chown ezpanel:ezpanel /etc/ezpanel/license.json
sudo chmod 600 /etc/ezpanel/license.json
```

### Enable the Commercial Module in the Config File

Edit `/etc/ezpanel/config.yaml` and uncomment the following:

```yaml
commercial:
  pro_binary_path: "/usr/bin/ezpanel-pro"
  license_path: "/etc/ezpanel/license.json"
  socket_path: "/run/ezpanel/commercial.sock"
```

### Restart the Service

```bash
sudo systemctl restart ezpanel
sudo journalctl -u ezpanel -f
```

The EzPanel main process will automatically launch the `ezpanel-pro` subprocess on startup. When `commercial module started` appears in the logs, the Pro edition is active.
