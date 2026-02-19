# 維運指南

**語言**: [English](./06_operations.md) | [简体中文](./06_operations_CN.md) | 繁體中文 | [Русский](./06_operations_RU.md) | [فارسی](./06_operations_FA.md)

本文件面向伺服器維運人員，涵蓋日常維運、備份、監控、效能調優等內容。

---

## 服務管理

### 基本操作

```bash
# 啟動服務
sudo systemctl start ezpanel

# 停止服務
sudo systemctl stop ezpanel

# 重啟服務
sudo systemctl restart ezpanel

# 查看狀態
sudo systemctl status ezpanel

# 開機自啟
sudo systemctl enable ezpanel

# 即時日誌
sudo journalctl -u ezpanel -f

# 查看最近 100 行日誌
sudo journalctl -u ezpanel -n 100
```

### XrayM 服務管理

```bash
sudo systemctl start xraym
sudo systemctl stop xraym
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## 日誌管理

### 日誌位置

| 服務 | 日誌來源 |
|------|----------|
| EzPanel | systemd journal（`journalctl -u ezpanel`）或 `/var/log/ezpanel/app.log` |
| XrayM | systemd journal（`journalctl -u xraym`） |
| Nginx | `/var/log/nginx/access.log` / `/var/log/nginx/error.log` |
| MySQL | `/var/log/mysql/error.log` |

### 調整日誌等級

編輯 `/etc/ezpanel/config.yaml`：

```yaml
log:
  level: "warn"    # debug / info / warn / error
```

正式環境推薦使用 `warn`，除錯時臨時改為 `debug`，問題排查後改回。

### 日誌歸檔

如設定輸出到檔案，建議設定 logrotate：

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

## 資料庫備份

### 手動備份

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
echo "備份完成: $BACKUP_DIR/ezpanel_$DATE.sql.gz"
```

### 自動定時備份

建立備份腳本 `/usr/local/bin/ezpanel-backup.sh`：

```bash
#!/bin/bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)
DB_PASS="your_password"

mkdir -p $BACKUP_DIR

# 資料庫備份
mysqldump -u ezpanel -p"$DB_PASS" ezpanel \
  --single-transaction --quick \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 設定檔備份
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/ezpanel/

# 上傳檔案備份
tar -czf $BACKUP_DIR/upload_$DATE.tar.gz /var/lib/ezpanel/upload/

# 只保留最近 14 天
find $BACKUP_DIR -name "*.gz" -mtime +14 -delete

echo "$(date): 備份完成" >> /var/log/ezpanel-backup.log
```

```bash
chmod +x /usr/local/bin/ezpanel-backup.sh

# 每天凌晨 3 點執行
echo "0 3 * * * root /usr/local/bin/ezpanel-backup.sh" > /etc/cron.d/ezpanel-backup
```

### 恢復資料庫

```bash
gunzip -c /backup/ezpanel/db_20240101_030000.sql.gz | mysql -u ezpanel -p ezpanel
```

---

## 設定檔備份

```bash
# 備份
tar -czf config_backup_$(date +%Y%m%d).tar.gz /etc/ezpanel/

# 恢復
tar -xzf config_backup_20240101.tar.gz -C /
```

---

## 效能調優

### 啟用 Redis 快取

Redis 顯著降低資料庫壓力，建議在正式環境啟用：

```yaml
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0
```

```bash
# 設定 Redis 記憶體策略
echo "maxmemory 256mb" | sudo tee -a /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis
```

### 資料庫連線池

根據伺服器設定調整：

```yaml
database:
  max_idle_conns: 10     # 閒置連線數
  max_open_conns: 100    # 最大連線數（不超過 MySQL max_connections）
```

查看 MySQL 最大連線數：
```sql
SHOW VARIABLES LIKE 'max_connections';
```

### MySQL 調優

編輯 `/etc/mysql/mysql.conf.d/mysqld.cnf`：

```ini
[mysqld]
innodb_buffer_pool_size = 512M   # 設為可用記憶體的 50-70%
innodb_log_file_size = 128M
max_connections = 200
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

```bash
sudo systemctl restart mysql
```

### Nginx 靜態資源快取

```nginx
location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    proxy_pass http://127.0.0.1:7088;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

---

## 健康檢查

### 手動檢查

```bash
# API 健康檢查
curl -s http://localhost:7088/api/v1/health

# 檢查資料庫連線
mysql -u ezpanel -p -e "SELECT 1;" ezpanel

# 檢查 Redis 連線
redis-cli ping
```

### 自動監控腳本

建立 `/usr/local/bin/ezpanel-health.sh`：

```bash
#!/bin/bash
HEALTH_URL="http://localhost:7088/api/v1/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 $HEALTH_URL)

if [ "$RESPONSE" != "200" ]; then
    echo "$(date): 健康檢查失敗 (HTTP $RESPONSE)，正在重啟..." >> /var/log/ezpanel-health.log
    systemctl restart ezpanel
fi
```

```bash
chmod +x /usr/local/bin/ezpanel-health.sh
# 每 5 分鐘檢查一次
echo "*/5 * * * * root /usr/local/bin/ezpanel-health.sh" > /etc/cron.d/ezpanel-health
```

---

## 防火牆設定

### 最小連接埠開放原則

| 連接埠 | 用途 | 建議 |
|--------|------|------|
| 22 | SSH | 改為非標準連接埠，限制來源 IP |
| 80 | HTTP | 開放（重定向到 HTTPS） |
| 443 | HTTPS | 開放 |
| 7088 | EzPanel 直接訪問 | 使用 Nginx 反代則關閉 |
| 3306 | MySQL | 僅本地訪問，不對外開放 |
| 6379 | Redis | 僅本地訪問，不對外開放 |

### UFW 設定範例

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### 節點連接埠

XrayM 節點監聽的連接埠（VMess、VLESS、Trojan 等）需要在節點伺服器的防火牆中開放。

---

## 升級流程

### 升級前檢查

1. 備份資料庫
2. 備份設定檔
3. 查看 Release Notes，注意是否有不相容變更

### 執行升級

參見 [安裝部署 - 升級](./01_installation_TW.md#升級)。

### 升級後驗證

```bash
# 檢查服務狀態
systemctl status ezpanel

# 檢查日誌是否有錯誤
journalctl -u ezpanel -n 50

# 測試登入
curl -s -X POST http://localhost:7088/api/v1/passport/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@opine.work","password":"admin123"}' | python3 -m json.tool
```

---

## 資料遷移

### 遷移到新伺服器

1. 在舊伺服器備份資料庫和設定檔
2. 在新伺服器安裝相同版本的 EzPanel
3. 恢復資料庫和設定檔
4. 更新 DNS 指向新伺服器
5. 驗證新伺服器執行正常後停止舊伺服器

```bash
# 舊伺服器：匯出資料
mysqldump -u ezpanel -p ezpanel > ezpanel_full.sql
tar -czf ezpanel_config.tar.gz /etc/ezpanel/ /var/lib/ezpanel/upload/

# 新伺服器：匯入資料
mysql -u ezpanel -p ezpanel < ezpanel_full.sql
tar -xzf ezpanel_config.tar.gz -C /
systemctl restart ezpanel
```

---

## 憑證續期

### Let's Encrypt 自動續期

```bash
# 測試續期
sudo certbot renew --dry-run

# 查看憑證到期時間
sudo certbot certificates
```

Certbot 安裝後會自動設定定時任務，通常無需手動操作。憑證到期前 30 天會自動續期。
