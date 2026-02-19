# Troubleshooting

**Language**: English | [简体中文](./07_troubleshooting_CN.md) | [繁體中文](./07_troubleshooting_TW.md) | [Русский](./07_troubleshooting_RU.md) | [فارسی](./07_troubleshooting_FA.md)

When a problem occurs, checking the service logs is usually the fastest way to identify the cause:

```bash
# EzPanel logs
sudo journalctl -u ezpanel -n 100 --no-pager

# XrayM logs
sudo journalctl -u xraym -n 100 --no-pager

# Nginx error log
sudo tail -n 50 /var/log/nginx/error.log
```

---

## Installation and Startup Issues

### Service Fails to Start

**Symptom**: `systemctl status ezpanel` shows `failed` or `activating`

**Troubleshooting steps**:

```bash
sudo journalctl -u ezpanel -n 50
```

Common causes:

| Error Message | Cause | Solution |
|---------------|-------|---------|
| `dial tcp: connection refused` | Database not running | `systemctl start mysql` |
| `Access denied for user` | Wrong database password | Check `database.password` in `config.yaml` |
| `Unknown database` | Database not created | Create the database manually; see [Installation](./01_installation.md) |
| `bind: address already in use` | Port already in use | Change `server.port` or free the port |
| `invalid configuration` | Malformed config file | Check YAML indentation; avoid using tabs |

### Database Tables Not Created Automatically

EzPanel creates tables automatically on first startup, but verify:

1. The database user has `CREATE TABLE` permission
2. The database itself was created beforehand (the system will not create the database automatically)
3. There are no database connection errors in the logs

```sql
-- Check user permissions
SHOW GRANTS FOR 'ezpanel'@'localhost';
```

---

## Login and Authentication Issues

### Cannot Log In / Forgot Password

Try the default administrator credentials:
- Email: `admin@opine.work`
- Password: `admin123`

If the default account also fails, you can update the password directly in the database:

```bash
# Generate a bcrypt hash of the new password (requires htpasswd)
htpasswd -nbBC 10 "" "new_password" | tr -d ':\n'

# Or use Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'new_password', bcrypt.gensalt(10)).decode())"
```

```sql
UPDATE users SET password = '$2a$10$...' WHERE email = 'admin@opine.work';
```

### Token Expires Frequently

Cause: The JWT secret may have changed after a server restart, or the token expiry time is set too short.

Solution:
1. Confirm that `jwt.secret` is hardcoded in `config.yaml` (do not use a randomly generated value)
2. Increase `jwt.expire` as appropriate (default is 168 hours = 7 days)

### 2FA Code Invalid

Possible cause: The server time and phone time are out of sync (TOTP depends on time accuracy).

```bash
# Sync server time
sudo timedatectl set-ntp true
timedatectl status
```

---

## Database Issues

### Database Connection Failure

```bash
# Check if MySQL is running
systemctl status mysql

# Test the connection
mysql -u ezpanel -p -h 127.0.0.1 ezpanel -e "SELECT 1;"
```

Common causes:
- MySQL service is not running
- Wrong username / password
- Database name does not exist
- Firewall is blocking port 3306 (for remote connections)

### Slow Database Performance

```sql
-- View slow query settings
SHOW VARIABLES LIKE 'slow_query_log';
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- View current connections
SHOW PROCESSLIST;

-- View InnoDB status
SHOW ENGINE INNODB STATUS\G
```

Adjust `innodb_buffer_pool_size` (recommended: 50–70% of available memory) and restart MySQL.

---

## Node Connection Issues

### Node Cannot Connect to the Panel

**Troubleshooting steps**:

1. Verify the panel address and API key are correct
2. Test network connectivity from the node server:
   ```bash
   curl -v https://panel.example.com/api/v1/server/UniProxy/config?node_id=1 \
     -H "Authorization: Bearer your-node-api-key"
   ```
3. Check whether the panel allows requests from the node IP (firewall / Cloudflare settings)
4. View XrayM logs:
   ```bash
   journalctl -u xraym -f
   ```

### Node Monitoring Data Not Displayed

1. Confirm you are using XrayM (XrayR does not support monitoring)
2. Confirm `node.report_metrics: true` is set in `config.yaml`
3. Check XrayM logs for any `report`-related errors

### Users Cannot Connect to the Node

Troubleshooting order:

1. Confirm the node service is running: `systemctl status xraym`
2. Confirm the node ports are open in the firewall
3. Check that the subscription link has been updated (have the client re-fetch the subscription)
4. View connection errors in XrayM logs

```bash
# Test whether the port is listening
ss -tlnp | grep 443

# Test port connectivity (from the client side)
nc -zv node.example.com 443
```

### XrayM Config Hot Reload Not Taking Effect

XrayM listens for config file changes and reloads automatically. If it does not take effect:

```bash
# Restart manually
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## Interface and Feature Issues

### Page Shows 502 Bad Gateway

Cause: The EzPanel service is not running, or the Nginx proxy configuration is incorrect.

```bash
# Check if EzPanel is running
systemctl status ezpanel
curl http://127.0.0.1:7088/api/v1/health

# Check Nginx config
nginx -t
```

### File Upload Fails

1. Check upload directory permissions:
   ```bash
   ls -la /var/lib/ezpanel/upload/
   # Should be owned by ezpanel:ezpanel
   sudo chown -R ezpanel:ezpanel /var/lib/ezpanel/upload/
   ```
2. Check upload size limits:
   - `upload.max_size` in `config.yaml`
   - `client_max_body_size` in Nginx:
     ```nginx
     client_max_body_size 20m;
     ```

### Email Sending Fails

1. Check SMTP settings (Admin Panel → System Settings → Email Settings)
2. Use the "Send Test Email" function to verify
3. Common issues:
   - Gmail requires an "app-specific password" instead of your account password
   - Some providers require SMTP to be explicitly enabled
   - Check whether the server IP is on an email provider's blocklist

### Pro Edition Features Not Showing

1. Confirm `ezpanel-pro` is installed:
   ```bash
   which ezpanel-pro
   systemctl status ezpanel
   ```
2. Confirm the `commercial` section in `config.yaml` is uncommented
3. Confirm the license file exists and is in the correct format:
   ```bash
   cat /etc/ezpanel/license.json
   ```
4. Check logs for license-related errors:
   ```bash
   journalctl -u ezpanel -n 50 | grep -i "license\|commercial"
   ```

---

## Performance Issues

### Panel Responds Slowly

**Diagnosis**:

```bash
# Check system load
top
htop

# Check memory usage
free -h

# Check disk I/O
iostat -x 1

# Check slow APIs (after enabling debug logging)
journalctl -u ezpanel | grep "latency"
```

**Common optimizations**:

1. Enable Redis cache
2. Reduce the log level to `warn`
3. Increase the database connection pool size
4. Tune `innodb_buffer_pool_size` for MySQL
5. Verify disk I/O is healthy (SSD is preferred over HDD)

### Out of Memory

```bash
# Check memory usage
ps aux --sort=-%mem | head -20
```

- The EzPanel main process normally uses approximately 50–200 MB
- If memory is insufficient, consider upgrading the server or disabling Swagger (`swagger.enable: false`)

---

## Resetting the Administrator Account

If all administrator accounts are inaccessible, you can modify the database directly:

```sql
-- Find administrator accounts
SELECT id, email, is_admin FROM users WHERE is_admin = 1;

-- Reset password to admin123
UPDATE users
SET password = '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi'
WHERE email = 'admin@opine.work';

-- Unban the account
UPDATE users SET banned = 0 WHERE email = 'admin@opine.work';
```

> The password hash `$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi` corresponds to the plaintext `password`. Please change it immediately after logging in.

---

## Getting Help

The EzPanel Community edition does not include official technical support. For assistance, you can:

1. Join the Telegram group [@OpineWorkOfficial](https://t.me/OpineWorkOfficial) to seek help from the community
2. Purchase a Pro or Enterprise edition for official support; see [https://opine.work](https://opine.work)
