# 安装部署

**语言**: [English](./01_installation.md) | 简体中文 | [繁體中文](./01_installation_TW.md) | [Русский](./01_installation_RU.md) | [فارسی](./01_installation_FA.md)

## 系统要求

### 硬件要求

| 项目 | 最低 | 推荐 |
|------|------|------|
| CPU | 1 核 | 2 核+ |
| 内存 | 1 GB | 2 GB+ |
| 磁盘 | 10 GB | 20 GB+ SSD |

### 软件要求

- **操作系统**: Linux x86_64 / aarch64（推荐 Arch Linux / Debian 12 / Ubuntu 22.04）
- **数据库**: MySQL 8.0+ 或 MariaDB 10.5+
- **缓存**: Redis 6.0+（可选，用于性能优化）

---

## 安装前准备

### 1. 创建数据库

EzPanel 不会自动创建数据库，需要手动操作：

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

### 2. 安装 Redis（可选）

Redis 为可选组件，启用后可提升系统性能（缓存、全局设备限制等）。

```bash
# Debian / Ubuntu
sudo apt install redis-server
sudo systemctl enable --now redis-server

# Arch Linux
sudo pacman -S redis
sudo systemctl enable --now redis
```

---

## 安装 EzPanel

### 方式一：Arch Linux 软件包

```bash
# 下载安装包
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-x86_64.pkg.tar.zst

# 安装
sudo pacman -U ezpanel-*.pkg.tar.zst
```

### 方式二：Debian / Ubuntu 软件包

```bash
# 下载安装包
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel_amd64.deb

# 安装
sudo dpkg -i ezpanel_*.deb
sudo apt-get install -f
```

### 方式三：通用二进制（Tarball）

适用于其他 Linux 发行版：

```bash
# 下载并解压
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-linux-amd64.tar.gz
tar -xzf ezpanel-linux-amd64.tar.gz
cd ezpanel-*

# 安装二进制
sudo cp ezpanel /usr/bin/
sudo chmod +x /usr/bin/ezpanel

# 创建配置目录和运行时目录
sudo mkdir -p /etc/ezpanel /run/ezpanel /var/lib/ezpanel /var/log/ezpanel

# 创建系统用户
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/ezpanel ezpanel
sudo chown ezpanel:ezpanel /var/lib/ezpanel /var/log/ezpanel /run/ezpanel

# 复制配置文件
sudo cp config.yaml.example /etc/ezpanel/config.yaml
sudo chown ezpanel:ezpanel /etc/ezpanel/config.yaml
sudo chmod 600 /etc/ezpanel/config.yaml

# 安装 systemd 服务
sudo cp ezpanel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## 初始配置

编辑配置文件 `/etc/ezpanel/config.yaml`，至少填写数据库连接信息和 JWT 密钥：

```yaml
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"   # 改为实际密码
  dbname: "ezpanel"

jwt:
  secret: "change-this-to-a-random-secret"  # 务必修改！
  expire: 168
```

详细配置说明参见 [配置说明](./02_configuration.md)。

---

## 启动服务

```bash
sudo systemctl enable --now ezpanel

# 查看运行状态
sudo systemctl status ezpanel

# 查看启动日志
sudo journalctl -u ezpanel -f
```

首次启动时，GORM 会自动创建所有数据库表，并初始化默认管理员账号。

**默认管理员**:
- 邮箱: `admin@opine.work`
- 密码: `admin123`

> **安全提醒**: 登录后请立即修改默认密码和邮箱。

---

## 配置 Nginx 反向代理（推荐）

生产环境建议在 EzPanel 前端部署 Nginx 反向代理，以支持 HTTPS 和静态资源优化。

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

    # 静态资源缓存
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

### 申请 SSL 证书

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com
```

---

## 升级

### Arch Linux / Debian 包升级

```bash
# 下载新版本安装包，然后重新安装
sudo pacman -U ezpanel-new-version.pkg.tar.zst    # Arch
sudo dpkg -i ezpanel_new_version_amd64.deb         # Debian
```

安装完成后服务会自动重启。

### 二进制升级

```bash
# 停止服务
sudo systemctl stop ezpanel

# 备份旧版本
sudo cp /usr/bin/ezpanel /usr/bin/ezpanel.bak

# 替换新版本
sudo cp ezpanel-new /usr/bin/ezpanel
sudo chmod +x /usr/bin/ezpanel

# 启动服务
sudo systemctl start ezpanel
sudo journalctl -u ezpanel -f  # 观察启动日志
```

---

## 安装专业版（Pro / Enterprise）

专业版作为独立进程运行，通过 Unix Socket 与 EzPanel 主进程通信。

### 前提条件

- EzPanel 社区版已正常运行
- 已获取有效的 `license.json` 授权文件

### 安装步骤

```bash
# 安装 ezpanel-pro（与 EzPanel 安装方式相同）
sudo dpkg -i ezpanel-pro_*.deb    # Debian/Ubuntu
sudo pacman -U ezpanel-pro-*.pkg.tar.zst  # Arch

# 放置授权文件
sudo cp license.json /etc/ezpanel/license.json
sudo chown ezpanel:ezpanel /etc/ezpanel/license.json
sudo chmod 600 /etc/ezpanel/license.json
```

### 在配置文件中启用商业模块

编辑 `/etc/ezpanel/config.yaml`，取消以下注释：

```yaml
commercial:
  pro_binary_path: "/usr/bin/ezpanel-pro"
  license_path: "/etc/ezpanel/license.json"
  socket_path: "/run/ezpanel/commercial.sock"
```

### 重启服务

```bash
sudo systemctl restart ezpanel
sudo journalctl -u ezpanel -f
```

EzPanel 主进程启动时会自动拉起 `ezpanel-pro` 子进程。日志中出现 `commercial module started` 表示专业版已激活。
