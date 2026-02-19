# نصب و راه‌اندازی

**زبان**: [English](./01_installation.md) | [简体中文](./01_installation_CN.md) | [繁體中文](./01_installation_TW.md) | [Русский](./01_installation_RU.md) | فارسی

## پیش‌نیازهای سیستم

### پیش‌نیازهای سخت‌افزاری

| مورد | حداقل | توصیه‌شده |
|------|-------|-----------|
| CPU | ۱ هسته | ۲ هسته+ |
| حافظه RAM | ۱ گیگابایت | ۲ گیگابایت+ |
| دیسک | ۱۰ گیگابایت | ۲۰ گیگابایت+ SSD |

### پیش‌نیازهای نرم‌افزاری

- **سیستم‌عامل**: Linux x86_64 / aarch64 (توصیه‌شده: Arch Linux / Debian 12 / Ubuntu 22.04)
- **پایگاه داده**: MySQL 8.0+ یا MariaDB 10.5+
- **کش**: Redis 6.0+ (اختیاری، برای بهینه‌سازی عملکرد)

---

## آماده‌سازی قبل از نصب

### ۱. ایجاد پایگاه داده

EzPanel به‌صورت خودکار پایگاه داده ایجاد نمی‌کند و باید این کار را دستی انجام دهید:

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

### ۲. نصب Redis (اختیاری)

Redis یک مؤلفه اختیاری است. فعال‌سازی آن عملکرد سیستم را بهبود می‌بخشد (کش، محدودیت جهانی دستگاه و غیره).

```bash
# Debian / Ubuntu
sudo apt install redis-server
sudo systemctl enable --now redis-server

# Arch Linux
sudo pacman -S redis
sudo systemctl enable --now redis
```

---

## نصب EzPanel

### روش اول: بسته Arch Linux

```bash
# دانلود بسته نصبی
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-x86_64.pkg.tar.zst

# نصب
sudo pacman -U ezpanel-*.pkg.tar.zst
```

### روش دوم: بسته Debian / Ubuntu

```bash
# دانلود بسته نصبی
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel_amd64.deb

# نصب
sudo dpkg -i ezpanel_*.deb
sudo apt-get install -f
```

### روش سوم: فایل باینری عمومی (Tarball)

برای سایر توزیع‌های Linux:

```bash
# دانلود و استخراج
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-linux-amd64.tar.gz
tar -xzf ezpanel-linux-amd64.tar.gz
cd ezpanel-*

# نصب فایل باینری
sudo cp ezpanel /usr/bin/
sudo chmod +x /usr/bin/ezpanel

# ایجاد دایرکتوری پیکربندی و زمان اجرا
sudo mkdir -p /etc/ezpanel /run/ezpanel /var/lib/ezpanel /var/log/ezpanel

# ایجاد کاربر سیستمی
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/ezpanel ezpanel
sudo chown ezpanel:ezpanel /var/lib/ezpanel /var/log/ezpanel /run/ezpanel

# کپی فایل پیکربندی
sudo cp config.yaml.example /etc/ezpanel/config.yaml
sudo chown ezpanel:ezpanel /etc/ezpanel/config.yaml
sudo chmod 600 /etc/ezpanel/config.yaml

# نصب سرویس systemd
sudo cp ezpanel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## پیکربندی اولیه

فایل پیکربندی `/etc/ezpanel/config.yaml` را ویرایش کنید و حداقل اطلاعات اتصال به پایگاه داده و کلید JWT را وارد نمایید:

```yaml
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"   # با رمز واقعی جایگزین کنید
  dbname: "ezpanel"

jwt:
  secret: "change-this-to-a-random-secret"  # حتماً تغییر دهید!
  expire: 168
```

برای توضیحات کامل پیکربندی به [راهنمای پیکربندی](./02_configuration_FA.md) مراجعه کنید.

---

## راه‌اندازی سرویس

```bash
sudo systemctl enable --now ezpanel

# مشاهده وضعیت اجرا
sudo systemctl status ezpanel

# مشاهده لاگ راه‌اندازی
sudo journalctl -u ezpanel -f
```

در اولین راه‌اندازی، GORM به‌صورت خودکار تمام جداول پایگاه داده را ایجاد می‌کند و حساب مدیر پیش‌فرض را مقداردهی می‌نماید.

**مدیر پیش‌فرض**:
- ایمیل: `admin@opine.work`
- رمز عبور: `admin123`

> **هشدار امنیتی**: پس از ورود، فوراً رمز عبور و ایمیل پیش‌فرض را تغییر دهید.

---

## پیکربندی پروکسی معکوس Nginx (توصیه‌شده)

در محیط تولید، توصیه می‌شود پروکسی معکوس Nginx را در مقابل EzPanel راه‌اندازی کنید تا از HTTPS و بهینه‌سازی منابع استاتیک پشتیبانی شود.

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

    # کش منابع استاتیک
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

### دریافت گواهی SSL

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com
```

---

## بروزرسانی

### بروزرسانی بسته Arch Linux / Debian

```bash
# دانلود نسخه جدید و نصب مجدد
sudo pacman -U ezpanel-new-version.pkg.tar.zst    # Arch
sudo dpkg -i ezpanel_new_version_amd64.deb         # Debian
```

پس از نصب، سرویس به‌صورت خودکار راه‌اندازی مجدد می‌شود.

### بروزرسانی فایل باینری

```bash
# توقف سرویس
sudo systemctl stop ezpanel

# پشتیبان‌گیری از نسخه قدیمی
sudo cp /usr/bin/ezpanel /usr/bin/ezpanel.bak

# جایگزینی با نسخه جدید
sudo cp ezpanel-new /usr/bin/ezpanel
sudo chmod +x /usr/bin/ezpanel

# راه‌اندازی سرویس
sudo systemctl start ezpanel
sudo journalctl -u ezpanel -f  # مشاهده لاگ راه‌اندازی
```

---

## نصب نسخه حرفه‌ای (Pro / Enterprise)

نسخه حرفه‌ای به‌عنوان یک فرآیند مستقل اجرا می‌شود و از طریق Unix Socket با فرآیند اصلی EzPanel ارتباط برقرار می‌کند.

### پیش‌نیازها

- نسخه رایگان EzPanel به‌درستی در حال اجرا باشد
- فایل مجوز معتبر `license.json` در اختیار داشته باشید

### مراحل نصب

```bash
# نصب ezpanel-pro (مانند نصب EzPanel)
sudo dpkg -i ezpanel-pro_*.deb    # Debian/Ubuntu
sudo pacman -U ezpanel-pro-*.pkg.tar.zst  # Arch

# قرار دادن فایل مجوز
sudo cp license.json /etc/ezpanel/license.json
sudo chown ezpanel:ezpanel /etc/ezpanel/license.json
sudo chmod 600 /etc/ezpanel/license.json
```

### فعال‌سازی ماژول تجاری در فایل پیکربندی

فایل `/etc/ezpanel/config.yaml` را ویرایش کرده و خطوط زیر را از حالت کامنت خارج کنید:

```yaml
commercial:
  pro_binary_path: "/usr/bin/ezpanel-pro"
  license_path: "/etc/ezpanel/license.json"
  socket_path: "/run/ezpanel/commercial.sock"
```

### راه‌اندازی مجدد سرویس

```bash
sudo systemctl restart ezpanel
sudo journalctl -u ezpanel -f
```

هنگام راه‌اندازی فرآیند اصلی EzPanel، فرآیند فرزند `ezpanel-pro` به‌صورت خودکار اجرا می‌شود. ظاهر شدن پیام `commercial module started` در لاگ نشان‌دهنده فعال‌سازی موفق نسخه حرفه‌ای است.
