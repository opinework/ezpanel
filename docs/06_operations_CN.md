# 运维指南

**语言**: [English](./06_operations.md) | 简体中文 | [繁體中文](./06_operations_TW.md) | [Русский](./06_operations_RU.md) | [فارسی](./06_operations_FA.md)

本文档面向服务器运维人员，涵盖日常运维、备份、监控、性能调优等内容。

---

## 服务管理

### 基本操作

```bash
# 启动服务
sudo systemctl start ezpanel

# 停止服务
sudo systemctl stop ezpanel

# 重启服务
sudo systemctl restart ezpanel

# 查看状态
sudo systemctl status ezpanel

# 开机自启
sudo systemctl enable ezpanel

# 实时日志
sudo journalctl -u ezpanel -f

# 查看最近 100 行日志
sudo journalctl -u ezpanel -n 100
```

### XrayM 服务管理

```bash
sudo systemctl start xraym
sudo systemctl stop xraym
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## 日志管理

### 日志位置

| 服务 | 日志来源 |
|------|----------|
| EzPanel | systemd journal（`journalctl -u ezpanel`）或 `/var/log/ezpanel/app.log` |
| XrayM | systemd journal（`journalctl -u xraym`） |
| Nginx | `/var/log/nginx/access.log` / `/var/log/nginx/error.log` |
| MySQL | `/var/log/mysql/error.log` |

### 调整日志级别

编辑 `/etc/ezpanel/config.yaml`：

```yaml
log:
  level: "warn"    # debug / info / warn / error
```

生产环境推荐使用 `warn`，调试时临时改为 `debug`，问题排查后改回。

### 日志归档

如配置输出到文件，建议配置 logrotate：

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

## 数据库备份

### 手动备份

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
echo "备份完成: $BACKUP_DIR/ezpanel_$DATE.sql.gz"
```

### 自动定时备份

创建备份脚本 `/usr/local/bin/ezpanel-backup.sh`：

```bash
#!/bin/bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)
DB_PASS="your_password"

mkdir -p $BACKUP_DIR

# 数据库备份
mysqldump -u ezpanel -p"$DB_PASS" ezpanel \
  --single-transaction --quick \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 配置文件备份
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/ezpanel/

# 上传文件备份
tar -czf $BACKUP_DIR/upload_$DATE.tar.gz /var/lib/ezpanel/upload/

# 只保留最近 14 天
find $BACKUP_DIR -name "*.gz" -mtime +14 -delete

echo "$(date): 备份完成" >> /var/log/ezpanel-backup.log
```

```bash
chmod +x /usr/local/bin/ezpanel-backup.sh

# 每天凌晨 3 点执行
echo "0 3 * * * root /usr/local/bin/ezpanel-backup.sh" > /etc/cron.d/ezpanel-backup
```

### 恢复数据库

```bash
gunzip -c /backup/ezpanel/db_20240101_030000.sql.gz | mysql -u ezpanel -p ezpanel
```

---

## 配置文件备份

```bash
# 备份
tar -czf config_backup_$(date +%Y%m%d).tar.gz /etc/ezpanel/

# 恢复
tar -xzf config_backup_20240101.tar.gz -C /
```

---

## 性能调优

### 启用 Redis 缓存

Redis 显著降低数据库压力，建议在生产环境启用：

```yaml
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0
```

```bash
# 配置 Redis 内存策略
echo "maxmemory 256mb" | sudo tee -a /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis
```

### 数据库连接池

根据服务器配置调整：

```yaml
database:
  max_idle_conns: 10     # 空闲连接数
  max_open_conns: 100    # 最大连接数（不超过 MySQL max_connections）
```

查看 MySQL 最大连接数：
```sql
SHOW VARIABLES LIKE 'max_connections';
```

### MySQL 调优

编辑 `/etc/mysql/mysql.conf.d/mysqld.cnf`：

```ini
[mysqld]
innodb_buffer_pool_size = 512M   # 设为可用内存的 50-70%
innodb_log_file_size = 128M
max_connections = 200
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

```bash
sudo systemctl restart mysql
```

### Nginx 静态资源缓存

```nginx
location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    proxy_pass http://127.0.0.1:7088;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

---

## 健康检查

### 手动检查

```bash
# API 健康检查
curl -s http://localhost:7088/api/v1/health

# 检查数据库连接
mysql -u ezpanel -p -e "SELECT 1;" ezpanel

# 检查 Redis 连接
redis-cli ping
```

### 自动监控脚本

创建 `/usr/local/bin/ezpanel-health.sh`：

```bash
#!/bin/bash
HEALTH_URL="http://localhost:7088/api/v1/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 $HEALTH_URL)

if [ "$RESPONSE" != "200" ]; then
    echo "$(date): 健康检查失败 (HTTP $RESPONSE)，正在重启..." >> /var/log/ezpanel-health.log
    systemctl restart ezpanel
fi
```

```bash
chmod +x /usr/local/bin/ezpanel-health.sh
# 每 5 分钟检查一次
echo "*/5 * * * * root /usr/local/bin/ezpanel-health.sh" > /etc/cron.d/ezpanel-health
```

---

## 防火墙配置

### 最小端口开放原则

| 端口 | 用途 | 建议 |
|------|------|------|
| 22 | SSH | 改为非标准端口，限制来源 IP |
| 80 | HTTP | 开放（重定向到 HTTPS） |
| 443 | HTTPS | 开放 |
| 7088 | EzPanel 直接访问 | 使用 Nginx 反代则关闭 |
| 3306 | MySQL | 仅本地访问，不对外开放 |
| 6379 | Redis | 仅本地访问，不对外开放 |

### UFW 配置示例

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### 节点端口

XrayM 节点监听的端口（VMess、VLESS、Trojan 等）需要在节点服务器的防火墙中开放。

---

## 升级流程

### 升级前检查

1. 备份数据库
2. 备份配置文件
3. 查看 Release Notes，注意是否有不兼容变更

### 执行升级

参见 [安装部署 - 升级](./01_installation_CN.md#升级)。

### 升级后验证

```bash
# 检查服务状态
systemctl status ezpanel

# 检查日志是否有错误
journalctl -u ezpanel -n 50

# 测试登录
curl -s -X POST http://localhost:7088/api/v1/passport/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@opine.work","password":"admin123"}' | python3 -m json.tool
```

---

## 数据迁移

### 迁移到新服务器

1. 在旧服务器备份数据库和配置文件
2. 在新服务器安装相同版本的 EzPanel
3. 恢复数据库和配置文件
4. 更新 DNS 指向新服务器
5. 验证新服务器运行正常后停止旧服务器

```bash
# 旧服务器：导出数据
mysqldump -u ezpanel -p ezpanel > ezpanel_full.sql
tar -czf ezpanel_config.tar.gz /etc/ezpanel/ /var/lib/ezpanel/upload/

# 新服务器：导入数据
mysql -u ezpanel -p ezpanel < ezpanel_full.sql
tar -xzf ezpanel_config.tar.gz -C /
systemctl restart ezpanel
```

---

## 证书续期

### Let's Encrypt 自动续期

```bash
# 测试续期
sudo certbot renew --dry-run

# 查看证书到期时间
sudo certbot certificates
```

Certbot 安装后会自动配置定时任务，通常无需手动操作。证书到期前 30 天会自动续期。
