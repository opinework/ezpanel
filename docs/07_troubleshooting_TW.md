# 常見問題與排查

**語言**: [English](./07_troubleshooting.md) | [简体中文](./07_troubleshooting_CN.md) | 繁體中文 | [Русский](./07_troubleshooting_RU.md) | [فارسی](./07_troubleshooting_FA.md)

遇到問題時，首先查看服務日誌往往能快速定位原因：

```bash
# EzPanel 日誌
sudo journalctl -u ezpanel -n 100 --no-pager

# XrayM 日誌
sudo journalctl -u xraym -n 100 --no-pager

# Nginx 錯誤日誌
sudo tail -n 50 /var/log/nginx/error.log
```

---

## 安裝與啟動問題

### 服務無法啟動

**症狀**：`systemctl status ezpanel` 顯示 `failed` 或 `activating`

**排查步驟**：

```bash
sudo journalctl -u ezpanel -n 50
```

常見原因：

| 錯誤訊息 | 原因 | 解決方案 |
|----------|------|----------|
| `dial tcp: connection refused` | 資料庫未啟動 | `systemctl start mysql` |
| `Access denied for user` | 資料庫密碼錯誤 | 檢查 `config.yaml` 中的 `database.password` |
| `Unknown database` | 資料庫未建立 | 手動建立資料庫，參見 [安裝部署](./01_installation_TW.md) |
| `bind: address already in use` | 連接埠被佔用 | 修改 `server.port` 或釋放佔用程序 |
| `invalid configuration` | 設定檔格式錯誤 | 檢查 YAML 縮排，避免使用 Tab |

### 資料庫未自動建立資料表

EzPanel 會在首次啟動時自動建表，但需確認：

1. 資料庫使用者有 `CREATE TABLE` 權限
2. 資料庫已提前建立（系統不會自動建立資料庫本身）
3. 日誌中沒有資料庫連線錯誤

```sql
-- 檢查使用者權限
SHOW GRANTS FOR 'ezpanel'@'localhost';
```

---

## 登入與認證問題

### 無法登入 / 忘記密碼

使用預設管理員帳號嘗試：
- 電子郵件: `admin@opine.work`
- 密碼: `admin123`

如預設帳號也無法登入，可直接修改資料庫密碼：

```bash
# 產生新密碼的 bcrypt 雜湊（需安裝 htpasswd）
htpasswd -nbBC 10 "" "new_password" | tr -d ':\n'

# 或使用 Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'new_password', bcrypt.gensalt(10)).decode())"
```

```sql
UPDATE users SET password = '$2a$10$...' WHERE email = 'admin@opine.work';
```

### Token 頻繁失效

原因：伺服器重啟後 JWT 密鑰可能變化，或 Token 過期時間設定過短。

解決：
1. 確認 `jwt.secret` 在 `config.yaml` 中已固定（不要使用隨機產生的）
2. 適當增大 `jwt.expire`（預設 168 小時 = 7 天）

### 2FA 驗證碼無效

可能原因：伺服器時間與手機時間不同步（TOTP 依賴時間精確度）

```bash
# 同步伺服器時間
sudo timedatectl set-ntp true
timedatectl status
```

---

## 資料庫問題

### 資料庫連線失敗

```bash
# 檢查 MySQL 是否執行
systemctl status mysql

# 測試連線
mysql -u ezpanel -p -h 127.0.0.1 ezpanel -e "SELECT 1;"
```

常見原因：
- MySQL 服務未啟動
- 使用者名稱 / 密碼錯誤
- 資料庫名稱不存在
- 防火牆阻止了 3306 連接埠（遠端連線時）

### 資料庫效能慢

```sql
-- 查看慢查詢
SHOW VARIABLES LIKE 'slow_query_log';
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 查看當前連線
SHOW PROCESSLIST;

-- 查看 InnoDB 狀態
SHOW ENGINE INNODB STATUS\G
```

調整 `innodb_buffer_pool_size`（建議為記憶體的 50-70%）並重啟 MySQL。

---

## 節點對接問題

### 節點無法連線面板

**排查步驟**：

1. 檢查面板位址和 API Key 是否正確
2. 在節點伺服器上測試網路連通性：
   ```bash
   curl -v https://panel.example.com/api/v1/server/UniProxy/config?node_id=1 \
     -H "Authorization: Bearer your-node-api-key"
   ```
3. 檢查面板是否允許來自節點 IP 的請求（防火牆 / Cloudflare 設定）
4. 查看 XrayM 日誌：
   ```bash
   journalctl -u xraym -f
   ```

### 節點監控資料不顯示

1. 確認使用的是 XrayM（XrayR 不支援監控）
2. 確認 `config.yaml` 中 `node.report_metrics: true`
3. 查看 XrayM 日誌中是否有 `report` 相關錯誤

### 使用者無法連線節點

排查順序：

1. 確認節點服務在執行：`systemctl status xraym`
2. 確認節點連接埠已在防火牆開放
3. 檢查訂閱連結是否已更新（客戶端重新拉取訂閱）
4. 查看 XrayM 日誌中的連線錯誤

```bash
# 測試連接埠是否監聽
ss -tlnp | grep 443

# 測試連接埠連通性（從客戶端測試）
nc -zv node.example.com 443
```

### XrayM 設定熱更新不生效

XrayM 監聽設定檔變化並自動重載。如未生效：

```bash
# 手動重啟
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## 介面與功能問題

### 頁面顯示 502 Bad Gateway

原因：EzPanel 服務未執行或 Nginx 代理設定錯誤。

```bash
# 檢查 EzPanel 是否執行
systemctl status ezpanel
curl http://127.0.0.1:7088/api/v1/health

# 檢查 Nginx 設定
nginx -t
```

### 檔案上傳失敗

1. 檢查上傳目錄權限：
   ```bash
   ls -la /var/lib/ezpanel/upload/
   # 應為 ezpanel:ezpanel 所有
   sudo chown -R ezpanel:ezpanel /var/lib/ezpanel/upload/
   ```
2. 檢查上傳大小限制：
   - `config.yaml` 中的 `upload.max_size`
   - Nginx 中的 `client_max_body_size`：
     ```nginx
     client_max_body_size 20m;
     ```

### 電子郵件發送失敗

1. 檢查 SMTP 設定（管理後台 → 系統設定 → 電子郵件設定）
2. 使用「發送測試郵件」功能驗證
3. 常見問題：
   - Gmail 需要使用「應用程式專用密碼」而非帳號密碼
   - 部分服務商需要開啟 SMTP 服務
   - 檢查伺服器是否被電子郵件服務商 IP 黑名單

### 專業版功能未顯示

1. 確認 `ezpanel-pro` 已安裝：
   ```bash
   which ezpanel-pro
   systemctl status ezpanel
   ```
2. 確認 `config.yaml` 中 `commercial` 設定已取消註解
3. 確認授權檔案存在且格式正確：
   ```bash
   cat /etc/ezpanel/license.json
   ```
4. 查看日誌中是否有授權相關錯誤：
   ```bash
   journalctl -u ezpanel -n 50 | grep -i "license\|commercial"
   ```

---

## 效能問題

### 面板回應慢

**診斷**：

```bash
# 查看系統負載
top
htop

# 查看記憶體使用
free -h

# 查看磁碟 I/O
iostat -x 1

# 查看慢 API（開啟 debug 日誌後）
journalctl -u ezpanel | grep "latency"
```

**常見最佳化措施**：

1. 啟用 Redis 快取
2. 調低日誌等級為 `warn`
3. 增大資料庫連線池
4. 為 MySQL 調整 `innodb_buffer_pool_size`
5. 確認磁碟 I/O 正常（SSD 優於 HDD）

### 記憶體不足

```bash
# 查看記憶體佔用
ps aux --sort=-%mem | head -20
```

- EzPanel 主程序正常佔用約 50-200 MB
- 如記憶體不足，考慮升級伺服器或關閉 Swagger（`swagger.enable: false`）

---

## 重設管理員帳號

如所有管理員帳號都無法訪問，可以直接修改資料庫：

```sql
-- 查找管理員帳號
SELECT id, email, is_admin FROM users WHERE is_admin = 1;

-- 重設密碼為 admin123
UPDATE users
SET password = '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi'
WHERE email = 'admin@opine.work';

-- 解封帳號
UPDATE users SET banned = 0 WHERE email = 'admin@opine.work';
```

> 密碼雜湊 `$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi` 對應明文 `password`，登入後請立即修改。

---

## 取得協助

EzPanel 社群版官方不提供技術支援。如需協助，可以：

1. 加入 Telegram 群組 [@OpineWorkOfficial](https://t.me/OpineWorkOfficial) 向社群尋求協助
2. 購買專業版或商業版以取得官方支援，詳見 [https://opine.work](https://opine.work)
