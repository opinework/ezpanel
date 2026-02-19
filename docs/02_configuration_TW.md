# 設定說明

**語言**: [English](./02_configuration.md) | [简体中文](./02_configuration_CN.md) | 繁體中文 | [Русский](./02_configuration_RU.md) | [فارسی](./02_configuration_FA.md)

EzPanel 的主設定檔位於 `/etc/ezpanel/config.yaml`。

---

## 完整設定參考

```yaml
# ================================
# EzPanel 配置文件
# ================================

# 伺服器設定
server:
  host: "0.0.0.0"        # 監聽位址，0.0.0.0 表示所有網路介面
  port: 7088              # 監聽埠，預設 7088
  mode: "release"         # 執行模式: debug / release / test

# 資料庫設定（必填）
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"
  dbname: "ezpanel"
  charset: "utf8mb4"
  max_idle_conns: 10      # 最大閒置連線數
  max_open_conns: 100     # 最大開啟連線數

# Redis 設定（可選）
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0

# JWT 設定
jwt:
  secret: "change-this-to-a-random-string"  # 務必修改！
  expire: 168             # Token 過期時間（小時），168 = 7 天
  issuer: "ezpanel"

# 日誌設定
log:
  level: "warn"           # 日誌等級: debug / info / warn / error
  format: "console"       # 輸出格式: json / console
  output: "stdout"        # 輸出位置: stdout / file / both
  file_path: "/var/log/ezpanel/app.log"

# Swagger API 文件
swagger:
  enable: false           # 正式環境建議關閉

# 節點監控（XrayM）
node:
  report_metrics: true    # 啟用節點監控資料上報

# 上傳設定
upload:
  path: "/var/lib/ezpanel/upload"
  max_size: 10            # 單檔案最大大小（MB）
  allowed_types:
    - image/jpeg
    - image/png
    - image/gif
    - image/webp

# CORS 跨域設定
cors:
  allow_origins:
    - "*"                 # 正式環境改為實際網域，如 https://panel.example.com
  allow_credentials: true
  max_age: 12

# 商業模組（Pro / Enterprise 版）
# commercial:
#   pro_binary_path: "/usr/bin/ezpanel-pro"
#   license_path: "/etc/ezpanel/license.json"
#   socket_path: "/run/ezpanel/commercial.sock"
```

---

## 設定項說明

### server

| 設定項 | 預設值 | 說明 |
|--------|--------|------|
| `host` | `0.0.0.0` | 監聽位址。如只允許本地訪問（Nginx 反代），可改為 `127.0.0.1` |
| `port` | `7088` | 監聽埠 |
| `mode` | `release` | `debug` 模式會輸出詳細日誌，正式環境使用 `release` |

### database

| 設定項 | 說明 |
|--------|------|
| `host` | MySQL 主機位址 |
| `port` | MySQL 連接埠，預設 3306 |
| `user` | 資料庫使用者名稱 |
| `password` | 資料庫密碼 |
| `dbname` | 資料庫名稱（需提前手動建立） |
| `max_open_conns` | 最大連線數，根據伺服器設定調整，建議 50-200 |

### redis

Redis 為可選元件。未設定時系統仍可正常執行，但以下功能不可用：

- 分散式快取加速
- XrayM 跨節點全域裝置限制

### jwt

| 設定項 | 說明 |
|--------|------|
| `secret` | JWT 簽名密鑰，**必須修改為隨機字串**，洩露會導致安全風險 |
| `expire` | Token 有效期（小時），修改後現有 Token 不受影響直到過期 |

產生隨機密鑰：
```bash
openssl rand -hex 32
```

### log

| 等級 | 說明 | 適用場景 |
|------|------|----------|
| `debug` | 輸出所有日誌，包含 SQL 語句 | 本地開發除錯 |
| `info` | 輸出一般資訊 | 測試環境 |
| `warn` | 僅輸出警告和錯誤 | **正式環境推薦** |
| `error` | 僅輸出錯誤 | 穩定正式環境 |

---

## 後台系統設定

以下設定項已移至 **管理後台 → 系統設定**，無需在設定檔中填寫：

### 站台資訊

- 站台名稱、Logo
- 站台 URL、訂閱 URL
- 註冊設定（是否開放註冊、邀請碼）

### 電子郵件（SMTP）

- SMTP 伺服器位址和連接埠
- 使用者名稱、密碼
- 寄件人位址
- 郵件範本管理

### Telegram Bot（可選）

1. 在 [@BotFather](https://t.me/BotFather) 建立 Bot，取得 Token
2. 在管理後台填入 Bot Token
3. 點擊「設定 Webhook」完成設定

使用者可使用以下指令綁定帳號：
- `/bindaccount` — 發起綁定
- `/bindcode <驗證碼>` — 完成綁定
- `/bindstatus` — 查看綁定狀態

### 付款閘道（Pro 版）

付款設定在管理後台 → 付款設定中管理，支援：

| 類型 | 說明 |
|------|------|
| EPay | 聚合付款平台，支援支付寶、微信等，推薦 |
| 支付寶當面付 | 掃碼付款 |
| 支付寶網頁付款 | 跳轉付款 |
| 微信支付 | 原生二維碼付款 |
| Stripe | 國際信用卡付款 |

### 節點通訊密鑰

節點與面板通訊所用的 API Key，在管理後台 → 系統設定 → 節點通訊密鑰中查看和修改。

---

## 多實例部署

如需在同一台伺服器執行多個面板實例，只需修改每個實例的連接埠和設定目錄：

```yaml
# 實例 2 的設定
server:
  port: 7089

database:
  dbname: "ezpanel2"
```

並建立對應的 systemd 服務檔案即可。
