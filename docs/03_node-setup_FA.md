# اتصال گره

**زبان**: [English](./03_node-setup.md) | [简体中文](./03_node-setup_CN.md) | [繁體中文](./03_node-setup_TW.md) | [Русский](./03_node-setup_RU.md) | فارسی

EzPanel از دو کلاینت گره XrayM و XrayR پشتیبانی می‌کند. استفاده از XrayM توصیه می‌شود.

![پیکربندی ورودی گره](../images/inbound.png)

---

## مقایسه XrayM و XrayR

| ویژگی | XrayR | XrayM (توصیه‌شده) |
|-------|-------|------------------|
| پشتیبانی از چند پروتکل | یک فرآیند فقط از یک پروتکل پشتیبانی می‌کند | یک فرآیند از چند پروتکل به‌طور همزمان پشتیبانی می‌کند |
| پشتیبانی از چند پورت | هر پورت نیاز به گره مستقل دارد | چند پورت روی یک گره |
| مانیتورینگ گره | پشتیبانی نمی‌شود | CPU / حافظه / شبکه / کاربران آنلاین |
| مدیریت پیکربندی | نگهداری دستی چند فایل پیکربندی | پنل به‌صورت خودکار تمام پیکربندی‌های پروتکل را توزیع می‌کند |
| مصرف منابع | چند فرآیند برای چند پروتکل | یک فرآیند، مصرف منابع کم |
| به‌روزرسانی گرم | پشتیبانی نمی‌شود | تغییرات پیکربندی بدون راه‌اندازی مجدد اعمال می‌شوند |

**سناریوهای توصیه‌شده**:
- نیاز به ارائه چند پروتکل همزمان (VMess + VLESS + Trojan و غیره)
- نیاز به مانیتورینگ بلادرنگ گره
- ساده‌سازی عملیات و کاهش مصرف منابع

---

## ایجاد گره در پنل مدیریت

به **پنل مدیریت ← مدیریت سیستم ← مدیریت گره** بروید و روی «ایجاد گره» کلیک کنید:

### اطلاعات پایه

| فیلد | توضیح |
|-----|-------|
| نام گره | نامی که به کاربران نمایش داده می‌شود، مثلاً «هنگ‌کنگ ۰۱» |
| آدرس گره | آدرس IP یا نام دامنه سرور گره |
| نوع گره | XrayM یا XrayR را انتخاب کنید |
| شناسه گره | پس از ایجاد به‌صورت خودکار تخصیص می‌یابد؛ در فایل پیکربندی گره وارد کنید |

### پیکربندی پروتکل (XrayM)

به تب «پیکربندی پروتکل» بروید و روی «افزودن سریع پروتکل» کلیک کنید تا ترکیب پیش‌تنظیم را انتخاب نمایید:

| ترکیب | پروتکل‌های شامل‌شده | توضیح |
|-------|--------------------|----|
| پروتکل‌های رایج | VMess(WS) + VLESS(WS) + Trojan(WS) + SS | توصیه‌شده؛ پوشش کلاینت‌های اصلی |
| ترکیب VMess | VMess(TCP/WS/gRPC) | بهترین سازگاری |
| ترکیب VLESS | VLESS(TCP/WS/gRPC/REALITY) | بهترین عملکرد |
| ترکیب Trojan | Trojan(TCP/WS/gRPC) | مخفی‌سازی خوب |
| Shadowsocks | SS(AES-256-GCM / 2022-BLAKE3) | سبک |

> پورت‌ها به‌صورت خودکار تخصیص می‌یابند و می‌توان پس از ایجاد آن‌ها را دستی تغییر داد. پورت‌های تکراری به‌صورت خودکار رد می‌شوند.

---

## نصب XrayM

### Arch Linux

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym-x86_64.pkg.tar.zst
sudo pacman -U xraym-*.pkg.tar.zst
```

### Debian / Ubuntu

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym_amd64.deb
sudo dpkg -i xraym_*.deb
sudo apt-get install -f
```

### فایل باینری عمومی

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym-linux-amd64.tar.gz
tar -xzf xraym-linux-amd64.tar.gz
sudo cp xraym /usr/bin/
sudo chmod +x /usr/bin/xraym
```

---

## پیکربندی XrayM

نمونه پیکربندی را کپی و ویرایش کنید:

```bash
sudo cp /etc/xraym/config.example.yml /etc/xraym/config.yml
sudo nano /etc/xraym/config.yml
```

### پیکربندی حداقلی (حالت خودکار، توصیه‌شده)

```yaml
Log:
  Level: warning

Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"   # آدرس پنل
      ApiKey: "your-node-api-key"            # کلید API گره
      NodeID: 1                              # شناسه گره
    ControllerConfig:
      ListenIP: "0.0.0.0"
      UpdatePeriodic: 60
```

`ApiKey` را در پنل مدیریت ← تنظیمات سیستم ← کلید API گره پیدا کنید.

### پیکربندی چند گره

```yaml
Log:
  Level: warning

Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"
      ApiKey: "your-node-api-key"
      NodeID: 1
    ControllerConfig:
      ListenIP: "0.0.0.0"

  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"
      ApiKey: "your-node-api-key"
      NodeID: 2
    ControllerConfig:
      ListenIP: "0.0.0.0"
```

### پیکربندی گواهی TLS

**روش اول: فایل گواهی محلی**

```yaml
Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"
      ApiKey: "your-key"
      NodeID: 1
    ControllerConfig:
      ListenIP: "0.0.0.0"
      CertConfig:
        CertMode: "file"
        CertFile: "/etc/ssl/certs/node.crt"
        KeyFile: "/etc/ssl/private/node.key"
```

**روش دوم: دریافت خودکار از طریق DNS (ACME)**

```yaml
ControllerConfig:
  CertConfig:
    CertMode: "dns"
    CertDomain: "node.example.com"
    Provider: "cloudflare"
    Email: "admin@example.com"
    DNSEnv:
      CF_API_EMAIL: "admin@example.com"
      CF_API_KEY: "your-cloudflare-api-key"
```

ارائه‌دهندگان DNS پشتیبانی‌شده: `cloudflare`، `alidns`، `dnspod`، `namesilo` و غیره.

### پیکربندی محدودیت سرعت خودکار

```yaml
ControllerConfig:
  UpdatePeriodic: 60
  AutoSpeedLimitConfig:
    Limit: 100         # آستانه فعال‌سازی (مگابیت‌بر‌ثانیه)، 0 غیرفعال
    WarnTimes: 3       # تعداد دفعات متوالی تجاوز قبل از محدودیت (0 محدودیت فوری)
    LimitSpeed: 10     # سرعت محدودیت (مگابیت‌بر‌ثانیه)
    LimitDuration: 30  # مدت محدودیت (دقیقه)
```

شرط فعال‌سازی: `ترافیک در ۶۰ ثانیه > 100 Mbps × 60s × 125000 = 750 MB`

### محدودیت جهانی دستگاه (نیاز به Redis)

```yaml
ControllerConfig:
  GlobalDeviceLimitConfig:
    Enable: true
    RedisAddr: "127.0.0.1:6379"
    RedisPassword: ""
```

---

## راه‌اندازی XrayM

```bash
sudo systemctl enable --now xraym

# مشاهده وضعیت
sudo systemctl status xraym

# مشاهده لاگ
sudo journalctl -u xraym -f
```

پس از راه‌اندازی موفق، خطوطی مشابه زیر در لاگ ظاهر می‌شوند:

```
INFO  XrayM started, node: 香港 01 (ID: 1)
INFO  Inbound VMess:443 started
INFO  Inbound Trojan:8443 started
```

---

## مانیتورینگ گره

XrayM هر ۶۰ ثانیه یک‌بار وضعیت گره را به پنل ارسال می‌کند.

![مانیتورینگ گره](../images/monitor.png)

**شاخص‌های مانیتورینگ**:
- مصرف CPU
- مصرف حافظه
- وضعیت دیسک
- سرعت آپلود / دانلود بلادرنگ
- تعداد کاربران آنلاین
- آمار ترافیک تجمعی

**نحوه مشاهده**: پنل مدیریت ← مدیریت سیستم ← بررسی سلامت

برای مشاهده نمودارهای تفصیلی روی کارت گره کلیک کنید؛ داده‌ها هر ۱۰ ثانیه به‌صورت خودکار به‌روز می‌شوند.

> XrayR از قابلیت مانیتورینگ گره پشتیبانی نمی‌کند و فقط ترافیک را جمع‌آوری می‌کند.

---

## استفاده از XrayR (توصیه‌نمی‌شود)

> یک فرآیند XrayR فقط از یک گره، یک پورت و یک پروتکل پشتیبانی می‌کند. برای چند پروتکل باید فرآیندهای جداگانه اجرا شوند.

```yaml
# /etc/XrayR/config.yml
Log:
  Level: warning

Nodes:
  - PanelType: "NewV2board"
    ApiConfig:
      ApiHost: "https://panel.example.com"
      ApiKey: "your-node-api-key"
      NodeID: 1
      NodeType: V2ray     # V2ray / Trojan / Shadowsocks
      Timeout: 30
    ControllerConfig:
      ListenIP: 0.0.0.0
      UpdatePeriodic: 60
```

برای چند پروتکل باید برای هر پروتکل یک گره و فایل پیکربندی جداگانه ایجاد کنید و چندین سرویس XrayR راه‌اندازی نمایید.

---

## تولید جفت کلید VLESS REALITY

```bash
xraym x25519
```

نمونه خروجی:

```
Private key: <base64-private-key>
Public key:  <base64-public-key>
```

Public key را در پیکربندی پروتکل گره در پنل وارد کنید. Private key را روی سرور محلی نگه دارید (نیازی به وارد کردن در پنل نیست).
