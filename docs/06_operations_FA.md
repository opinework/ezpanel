# راهنمای عملیات

**زبان**: [English](./06_operations.md) | [简体中文](./06_operations_CN.md) | [繁體中文](./06_operations_TW.md) | [Русский](./06_operations_RU.md) | فارسی

این سند برای مدیران سرور است و موضوعات عملیات روزانه، پشتیبان‌گیری، مانیتورینگ و بهینه‌سازی عملکرد را پوشش می‌دهد.

---

## مدیریت سرویس

### عملیات پایه

```bash
# راه‌اندازی سرویس
sudo systemctl start ezpanel

# توقف سرویس
sudo systemctl stop ezpanel

# راه‌اندازی مجدد سرویس
sudo systemctl restart ezpanel

# مشاهده وضعیت
sudo systemctl status ezpanel

# فعال‌سازی راه‌اندازی خودکار
sudo systemctl enable ezpanel

# لاگ بلادرنگ
sudo journalctl -u ezpanel -f

# مشاهده ۱۰۰ خط آخر لاگ
sudo journalctl -u ezpanel -n 100
```

### مدیریت سرویس XrayM

```bash
sudo systemctl start xraym
sudo systemctl stop xraym
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## مدیریت لاگ

### مکان لاگ‌ها

| سرویس | منبع لاگ |
|-------|----------|
| EzPanel | journal سیستم (`journalctl -u ezpanel`) یا `/var/log/ezpanel/app.log` |
| XrayM | journal سیستم (`journalctl -u xraym`) |
| Nginx | `/var/log/nginx/access.log` / `/var/log/nginx/error.log` |
| MySQL | `/var/log/mysql/error.log` |

### تنظیم سطح لاگ

فایل `/etc/ezpanel/config.yaml` را ویرایش کنید:

```yaml
log:
  level: "warn"    # debug / info / warn / error
```

در محیط تولید از `warn` استفاده کنید؛ هنگام اشکال‌زدایی به `debug` تغییر دهید و پس از حل مشکل برگردانید.

### بایگانی لاگ

اگر خروجی را به فایل تنظیم کرده‌اید، پیکربندی logrotate توصیه می‌شود:

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

## پشتیبان‌گیری از پایگاه داده

### پشتیبان‌گیری دستی

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
echo "پشتیبان‌گیری کامل شد: $BACKUP_DIR/ezpanel_$DATE.sql.gz"
```

### پشتیبان‌گیری خودکار زمان‌بندی‌شده

اسکریپت پشتیبان‌گیری `/usr/local/bin/ezpanel-backup.sh` را ایجاد کنید:

```bash
#!/bin/bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)
DB_PASS="your_password"

mkdir -p $BACKUP_DIR

# پشتیبان‌گیری پایگاه داده
mysqldump -u ezpanel -p"$DB_PASS" ezpanel \
  --single-transaction --quick \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# پشتیبان‌گیری فایل پیکربندی
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/ezpanel/

# پشتیبان‌گیری فایل‌های آپلودشده
tar -czf $BACKUP_DIR/upload_$DATE.tar.gz /var/lib/ezpanel/upload/

# فقط ۱۴ روز اخیر را نگه دارید
find $BACKUP_DIR -name "*.gz" -mtime +14 -delete

echo "$(date): پشتیبان‌گیری کامل شد" >> /var/log/ezpanel-backup.log
```

```bash
chmod +x /usr/local/bin/ezpanel-backup.sh

# هر روز ساعت ۳ صبح اجرا شود
echo "0 3 * * * root /usr/local/bin/ezpanel-backup.sh" > /etc/cron.d/ezpanel-backup
```

### بازیابی پایگاه داده

```bash
gunzip -c /backup/ezpanel/db_20240101_030000.sql.gz | mysql -u ezpanel -p ezpanel
```

---

## پشتیبان‌گیری از فایل‌های پیکربندی

```bash
# پشتیبان‌گیری
tar -czf config_backup_$(date +%Y%m%d).tar.gz /etc/ezpanel/

# بازیابی
tar -xzf config_backup_20240101.tar.gz -C /
```

---

## بهینه‌سازی عملکرد

### فعال‌سازی کش Redis

Redis به‌طور قابل‌توجهی فشار پایگاه داده را کاهش می‌دهد. در محیط تولید فعال‌سازی آن توصیه می‌شود:

```yaml
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0
```

```bash
# پیکربندی سیاست حافظه Redis
echo "maxmemory 256mb" | sudo tee -a /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis
```

### استخر اتصالات پایگاه داده

بر اساس پیکربندی سرور تنظیم کنید:

```yaml
database:
  max_idle_conns: 10     # تعداد اتصالات بیکار
  max_open_conns: 100    # حداکثر اتصالات (بیشتر از max_connections MySQL نباشد)
```

مشاهده حداکثر اتصالات MySQL:
```sql
SHOW VARIABLES LIKE 'max_connections';
```

### بهینه‌سازی MySQL

فایل `/etc/mysql/mysql.conf.d/mysqld.cnf` را ویرایش کنید:

```ini
[mysqld]
innodb_buffer_pool_size = 512M   # ۵۰ تا ۷۰ درصد حافظه موجود را تنظیم کنید
innodb_log_file_size = 128M
max_connections = 200
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

```bash
sudo systemctl restart mysql
```

### کش منابع استاتیک Nginx

```nginx
location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    proxy_pass http://127.0.0.1:7088;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

---

## بررسی سلامت

### بررسی دستی

```bash
# بررسی سلامت API
curl -s http://localhost:7088/api/v1/health

# بررسی اتصال پایگاه داده
mysql -u ezpanel -p -e "SELECT 1;" ezpanel

# بررسی اتصال Redis
redis-cli ping
```

### اسکریپت مانیتورینگ خودکار

فایل `/usr/local/bin/ezpanel-health.sh` را ایجاد کنید:

```bash
#!/bin/bash
HEALTH_URL="http://localhost:7088/api/v1/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 $HEALTH_URL)

if [ "$RESPONSE" != "200" ]; then
    echo "$(date): بررسی سلامت ناموفق بود (HTTP $RESPONSE)، در حال راه‌اندازی مجدد..." >> /var/log/ezpanel-health.log
    systemctl restart ezpanel
fi
```

```bash
chmod +x /usr/local/bin/ezpanel-health.sh
# هر ۵ دقیقه یک‌بار بررسی کنید
echo "*/5 * * * * root /usr/local/bin/ezpanel-health.sh" > /etc/cron.d/ezpanel-health
```

---

## پیکربندی فایروال

### اصل حداقل پورت‌های باز

| پورت | کاربرد | توصیه |
|-----|--------|-------|
| 22 | SSH | به پورت غیراستاندارد منتقل کنید، IP منبع را محدود کنید |
| 80 | HTTP | باز کنید (هدایت به HTTPS) |
| 443 | HTTPS | باز کنید |
| 7088 | دسترسی مستقیم به EzPanel | در صورت استفاده از پروکسی معکوس Nginx ببندید |
| 3306 | MySQL | فقط دسترسی محلی، به بیرون باز نکنید |
| 6379 | Redis | فقط دسترسی محلی، به بیرون باز نکنید |

### نمونه پیکربندی UFW

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### پورت‌های گره

پورت‌هایی که گره‌های XrayM روی آن‌ها گوش می‌دهند (VMess، VLESS، Trojan و غیره) باید در فایروال سرور گره باز شوند.

---

## روند بروزرسانی

### بررسی‌های قبل از بروزرسانی

1. پشتیبان‌گیری از پایگاه داده
2. پشتیبان‌گیری از فایل‌های پیکربندی
3. مشاهده یادداشت‌های انتشار (Release Notes) و توجه به تغییرات ناسازگار

### انجام بروزرسانی

به [نصب و راه‌اندازی - بروزرسانی](./01_installation_FA.md#بروزرسانی) مراجعه کنید.

### تأیید پس از بروزرسانی

```bash
# بررسی وضعیت سرویس
systemctl status ezpanel

# بررسی لاگ برای خطاها
journalctl -u ezpanel -n 50

# تست ورود
curl -s -X POST http://localhost:7088/api/v1/passport/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@opine.work","password":"admin123"}' | python3 -m json.tool
```

---

## مهاجرت داده‌ها

### انتقال به سرور جدید

1. پشتیبان‌گیری از پایگاه داده و فایل‌های پیکربندی روی سرور قدیمی
2. نصب همان نسخه EzPanel روی سرور جدید
3. بازیابی پایگاه داده و فایل‌های پیکربندی
4. به‌روزرسانی DNS برای اشاره به سرور جدید
5. پس از تأیید عملکرد صحیح سرور جدید، سرور قدیمی را متوقف کنید

```bash
# سرور قدیمی: خروجی داده‌ها
mysqldump -u ezpanel -p ezpanel > ezpanel_full.sql
tar -czf ezpanel_config.tar.gz /etc/ezpanel/ /var/lib/ezpanel/upload/

# سرور جدید: ورودی داده‌ها
mysql -u ezpanel -p ezpanel < ezpanel_full.sql
tar -xzf ezpanel_config.tar.gz -C /
systemctl restart ezpanel
```

---

## تجدید گواهی

### تجدید خودکار Let's Encrypt

```bash
# تست تجدید
sudo certbot renew --dry-run

# مشاهده تاریخ انقضای گواهی‌ها
sudo certbot certificates
```

پس از نصب Certbot، یک وظیفه زمان‌بندی‌شده به‌صورت خودکار پیکربندی می‌شود — معمولاً نیازی به عملیات دستی نیست. گواهی‌ها ۳۰ روز قبل از انقضا به‌صورت خودکار تجدید می‌شوند.
