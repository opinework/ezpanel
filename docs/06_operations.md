# Operations Guide

**Language**: English | [简体中文](./06_operations_CN.md) | [繁體中文](./06_operations_TW.md) | [Русский](./06_operations_RU.md) | [فارسی](./06_operations_FA.md)

This document is intended for server operations staff and covers day-to-day operations, backups, monitoring, and performance tuning.

---

## Service Management

### Basic Operations

```bash
# Start the service
sudo systemctl start ezpanel

# Stop the service
sudo systemctl stop ezpanel

# Restart the service
sudo systemctl restart ezpanel

# Check the status
sudo systemctl status ezpanel

# Enable on boot
sudo systemctl enable ezpanel

# Follow live logs
sudo journalctl -u ezpanel -f

# View the last 100 lines of logs
sudo journalctl -u ezpanel -n 100
```

### XrayM Service Management

```bash
sudo systemctl start xraym
sudo systemctl stop xraym
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## Log Management

### Log Locations

| Service | Log Source |
|---------|------------|
| EzPanel | systemd journal (`journalctl -u ezpanel`) or `/var/log/ezpanel/app.log` |
| XrayM | systemd journal (`journalctl -u xraym`) |
| Nginx | `/var/log/nginx/access.log` / `/var/log/nginx/error.log` |
| MySQL | `/var/log/mysql/error.log` |

### Adjusting the Log Level

Edit `/etc/ezpanel/config.yaml`:

```yaml
log:
  level: "warn"    # debug / info / warn / error
```

Use `warn` in production. Temporarily switch to `debug` for troubleshooting and revert afterward.

### Log Rotation

If output is configured to a file, it is recommended to set up logrotate:

```bash
# /etc/logrotate.d/ezpanel
/var/log/ezpanel/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
        systemctl reload ezpanel 2>/dev/null || true
    endscript
}
```

---

## Database Backup

### Manual Backup

```bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

mysqldump -u ezpanel -p ezpanel \
  --single-transaction \
  --quick \
  --lock-tables=false \
  > $BACKUP_DIR/ezpanel_$DATE.sql

gzip $BACKUP_DIR/ezpanel_$DATE.sql
echo "Backup complete: $BACKUP_DIR/ezpanel_$DATE.sql.gz"
```

### Scheduled Automatic Backup

Create the backup script `/usr/local/bin/ezpanel-backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)
DB_PASS="your_password"

mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u ezpanel -p"$DB_PASS" ezpanel \
  --single-transaction --quick \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Config file backup
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/ezpanel/

# Upload file backup
tar -czf $BACKUP_DIR/upload_$DATE.tar.gz /var/lib/ezpanel/upload/

# Retain only the last 14 days
find $BACKUP_DIR -name "*.gz" -mtime +14 -delete

echo "$(date): Backup complete" >> /var/log/ezpanel-backup.log
```

```bash
chmod +x /usr/local/bin/ezpanel-backup.sh

# Run at 3:00 AM every day
echo "0 3 * * * root /usr/local/bin/ezpanel-backup.sh" > /etc/cron.d/ezpanel-backup
```

### Restoring the Database

```bash
gunzip -c /backup/ezpanel/db_20240101_030000.sql.gz | mysql -u ezpanel -p ezpanel
```

---

## Config File Backup

```bash
# Backup
tar -czf config_backup_$(date +%Y%m%d).tar.gz /etc/ezpanel/

# Restore
tar -xzf config_backup_20240101.tar.gz -C /
```

---

## Performance Tuning

### Enabling Redis Cache

Redis significantly reduces database load. It is recommended to enable it in production:

```yaml
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0
```

```bash
# Configure Redis memory policy
echo "maxmemory 256mb" | sudo tee -a /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis
```

### Database Connection Pool

Adjust based on server capacity:

```yaml
database:
  max_idle_conns: 10     # Idle connections
  max_open_conns: 100    # Maximum connections (should not exceed MySQL max_connections)
```

Check MySQL's maximum connections:
```sql
SHOW VARIABLES LIKE 'max_connections';
```

### MySQL Tuning

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
[mysqld]
innodb_buffer_pool_size = 512M   # Set to 50-70% of available memory
innodb_log_file_size = 128M
max_connections = 200
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

```bash
sudo systemctl restart mysql
```

### Nginx Static Asset Caching

```nginx
location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    proxy_pass http://127.0.0.1:7088;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

---

## Health Checks

### Manual Checks

```bash
# API health check
curl -s http://localhost:7088/api/v1/health

# Check database connection
mysql -u ezpanel -p -e "SELECT 1;" ezpanel

# Check Redis connection
redis-cli ping
```

### Automated Monitoring Script

Create `/usr/local/bin/ezpanel-health.sh`:

```bash
#!/bin/bash
HEALTH_URL="http://localhost:7088/api/v1/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 $HEALTH_URL)

if [ "$RESPONSE" != "200" ]; then
    echo "$(date): Health check failed (HTTP $RESPONSE), restarting..." >> /var/log/ezpanel-health.log
    systemctl restart ezpanel
fi
```

```bash
chmod +x /usr/local/bin/ezpanel-health.sh
# Check every 5 minutes
echo "*/5 * * * * root /usr/local/bin/ezpanel-health.sh" > /etc/cron.d/ezpanel-health
```

---

## Firewall Configuration

### Minimum Port Exposure Principle

| Port | Purpose | Recommendation |
|------|---------|----------------|
| 22 | SSH | Change to a non-standard port; restrict source IPs |
| 80 | HTTP | Open (redirects to HTTPS) |
| 443 | HTTPS | Open |
| 7088 | EzPanel direct access | Close if using Nginx reverse proxy |
| 3306 | MySQL | Local access only; do not expose externally |
| 6379 | Redis | Local access only; do not expose externally |

### UFW Configuration Example

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### Node Ports

The ports that XrayM nodes listen on (VMess, VLESS, Trojan, etc.) must be opened in the firewall on the node servers.

---

## Upgrade Process

### Pre-Upgrade Checklist

1. Back up the database
2. Back up the config files
3. Review the Release Notes and check for any breaking changes

### Performing the Upgrade

See [Installation - Upgrading](./01_installation.md#upgrading).

### Post-Upgrade Verification

```bash
# Check service status
systemctl status ezpanel

# Check logs for errors
journalctl -u ezpanel -n 50

# Test login
curl -s -X POST http://localhost:7088/api/v1/passport/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@opine.work","password":"admin123"}' | python3 -m json.tool
```

---

## Data Migration

### Migrating to a New Server

1. Back up the database and config files on the old server
2. Install the same version of EzPanel on the new server
3. Restore the database and config files
4. Update DNS to point to the new server
5. Stop the old server after verifying that the new server is running correctly

```bash
# Old server: export data
mysqldump -u ezpanel -p ezpanel > ezpanel_full.sql
tar -czf ezpanel_config.tar.gz /etc/ezpanel/ /var/lib/ezpanel/upload/

# New server: import data
mysql -u ezpanel -p ezpanel < ezpanel_full.sql
tar -xzf ezpanel_config.tar.gz -C /
systemctl restart ezpanel
```

---

## Certificate Renewal

### Let's Encrypt Automatic Renewal

```bash
# Test renewal
sudo certbot renew --dry-run

# View certificate expiration dates
sudo certbot certificates
```

After Certbot is installed, it automatically configures a cron job. Certificates are renewed automatically 30 days before expiration — no manual action is normally required.
