# 安裝部署

**語言**: [English](./01_installation.md) | [简体中文](./01_installation_CN.md) | 繁體中文 | [Русский](./01_installation_RU.md) | [فارسی](./01_installation_FA.md)

## 系統需求

### 硬體需求

| 項目 | 最低 | 推薦 |
|------|------|------|
| CPU | 1 核 | 2 核+ |
| 記憶體 | 1 GB | 2 GB+ |
| 磁碟 | 10 GB | 20 GB+ SSD |

### 軟體需求

- **作業系統**: Linux x86_64 / aarch64（推薦 Arch Linux / Debian 12 / Ubuntu 22.04）
- **資料庫**: MySQL 8.0+ 或 MariaDB 10.5+
- **快取**: Redis 6.0+（可選，用於效能最佳化）

---

## 安裝前準備

### 1. 建立資料庫

EzPanel 不會自動建立資料庫，需要手動操作：

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

### 2. 安裝 Redis（可選）

Redis 為可選元件，啟用後可提升系統效能（快取、全域裝置限制等）。

```bash
# Debian / Ubuntu
sudo apt install redis-server
sudo systemctl enable --now redis-server

# Arch Linux
sudo pacman -S redis
sudo systemctl enable --now redis
```

---

## 安裝 EzPanel

### 方式一：Arch Linux 軟體包

```bash
# 下載安裝包
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-x86_64.pkg.tar.zst

# 安裝
sudo pacman -U ezpanel-*.pkg.tar.zst
```

### 方式二：Debian / Ubuntu 軟體包

```bash
# 下載安裝包
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel_amd64.deb

# 安裝
sudo dpkg -i ezpanel_*.deb
sudo apt-get install -f
```

### 方式三：通用二進位（Tarball）

適用於其他 Linux 發行版：

```bash
# 下載並解壓
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-linux-amd64.tar.gz
tar -xzf ezpanel-linux-amd64.tar.gz
cd ezpanel-*

# 安裝二進位
sudo cp ezpanel /usr/bin/
sudo chmod +x /usr/bin/ezpanel

# 建立設定目錄和執行時目錄
sudo mkdir -p /etc/ezpanel /run/ezpanel /var/lib/ezpanel /var/log/ezpanel

# 建立系統使用者
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/ezpanel ezpanel
sudo chown ezpanel:ezpanel /var/lib/ezpanel /var/log/ezpanel /run/ezpanel

# 複製設定檔
sudo cp config.yaml.example /etc/ezpanel/config.yaml
sudo chown ezpanel:ezpanel /etc/ezpanel/config.yaml
sudo chmod 600 /etc/ezpanel/config.yaml

# 安裝 systemd 服務
sudo cp ezpanel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## 初始設定

編輯設定檔 `/etc/ezpanel/config.yaml`，至少填寫資料庫連線資訊和 JWT 密鑰：

```yaml
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"   # 改為實際密碼
  dbname: "ezpanel"

jwt:
  secret: "change-this-to-a-random-secret"  # 務必修改！
  expire: 168
```

詳細設定說明參見 [設定說明](./02_configuration_TW.md)。

---

## 啟動服務

```bash
sudo systemctl enable --now ezpanel

# 查看執行狀態
sudo systemctl status ezpanel

# 查看啟動日誌
sudo journalctl -u ezpanel -f
```

首次啟動時，GORM 會自動建立所有資料庫資料表，並初始化預設管理員帳號。

**預設管理員**:
- 電子郵件: `admin@opine.work`
- 密碼: `admin123`

> **安全提醒**: 登入後請立即修改預設密碼和電子郵件。

---

## 設定 Nginx 反向代理（推薦）

正式環境建議在 EzPanel 前端部署 Nginx 反向代理，以支援 HTTPS 和靜態資源最佳化。

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

    # 靜態資源快取
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

### 申請 SSL 憑證

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com
```

---

## 升級

### Arch Linux / Debian 套件升級

```bash
# 下載新版本安裝包，然後重新安裝
sudo pacman -U ezpanel-new-version.pkg.tar.zst    # Arch
sudo dpkg -i ezpanel_new_version_amd64.deb         # Debian
```

安裝完成後服務會自動重啟。

### 二進位升級

```bash
# 停止服務
sudo systemctl stop ezpanel

# 備份舊版本
sudo cp /usr/bin/ezpanel /usr/bin/ezpanel.bak

# 替換新版本
sudo cp ezpanel-new /usr/bin/ezpanel
sudo chmod +x /usr/bin/ezpanel

# 啟動服務
sudo systemctl start ezpanel
sudo journalctl -u ezpanel -f  # 觀察啟動日誌
```

---

## 安裝專業版（Pro / Enterprise）

專業版作為獨立程序執行，透過 Unix Socket 與 EzPanel 主程序通訊。

### 前提條件

- EzPanel 社群版已正常執行
- 已取得有效的 `license.json` 授權檔案

### 安裝步驟

```bash
# 安裝 ezpanel-pro（與 EzPanel 安裝方式相同）
sudo dpkg -i ezpanel-pro_*.deb    # Debian/Ubuntu
sudo pacman -U ezpanel-pro-*.pkg.tar.zst  # Arch

# 放置授權檔案
sudo cp license.json /etc/ezpanel/license.json
sudo chown ezpanel:ezpanel /etc/ezpanel/license.json
sudo chmod 600 /etc/ezpanel/license.json
```

### 在設定檔中啟用商業模組

編輯 `/etc/ezpanel/config.yaml`，取消以下註解：

```yaml
commercial:
  pro_binary_path: "/usr/bin/ezpanel-pro"
  license_path: "/etc/ezpanel/license.json"
  socket_path: "/run/ezpanel/commercial.sock"
```

### 重啟服務

```bash
sudo systemctl restart ezpanel
sudo journalctl -u ezpanel -f
```

EzPanel 主程序啟動時會自動拉起 `ezpanel-pro` 子程序。日誌中出現 `commercial module started` 表示專業版已啟用。
