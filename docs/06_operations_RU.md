# Руководство по эксплуатации

**Язык**: [English](./06_operations.md) | [简体中文](./06_operations_CN.md) | [繁體中文](./06_operations_TW.md) | Русский | [فارسی](./06_operations_FA.md)

Настоящий документ предназначен для системных администраторов и охватывает вопросы повседневной эксплуатации, резервного копирования, мониторинга и оптимизации производительности.

---

## Управление службами

### Основные операции

```bash
# Запустить службу
sudo systemctl start ezpanel

# Остановить службу
sudo systemctl stop ezpanel

# Перезапустить службу
sudo systemctl restart ezpanel

# Проверить статус
sudo systemctl status ezpanel

# Включить автозапуск
sudo systemctl enable ezpanel

# Просмотр журнала в реальном времени
sudo journalctl -u ezpanel -f

# Просмотр последних 100 строк журнала
sudo journalctl -u ezpanel -n 100
```

### Управление службой XrayM

```bash
sudo systemctl start xraym
sudo systemctl stop xraym
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## Управление журналами

### Расположение журналов

| Служба | Источник журнала |
|--------|-----------------|
| EzPanel | Журнал systemd (`journalctl -u ezpanel`) или `/var/log/ezpanel/app.log` |
| XrayM | Журнал systemd (`journalctl -u xraym`) |
| Nginx | `/var/log/nginx/access.log` / `/var/log/nginx/error.log` |
| MySQL | `/var/log/mysql/error.log` |

### Настройка уровня журналирования

Отредактируйте `/etc/ezpanel/config.yaml`:

```yaml
log:
  level: "warn"    # debug / info / warn / error
```

Для производственной среды рекомендуется `warn`; временно переключайтесь на `debug` при отладке, затем возвращайте обратно.

### Ротация журналов

При выводе журналов в файл рекомендуется настроить logrotate:

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

## Резервное копирование базы данных

### Ручное резервное копирование

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
echo "Резервное копирование завершено: $BACKUP_DIR/ezpanel_$DATE.sql.gz"
```

### Автоматическое резервное копирование по расписанию

Создайте скрипт резервного копирования `/usr/local/bin/ezpanel-backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR=/backup/ezpanel
DATE=$(date +%Y%m%d_%H%M%S)
DB_PASS="your_password"

mkdir -p $BACKUP_DIR

# Резервная копия базы данных
mysqldump -u ezpanel -p"$DB_PASS" ezpanel \
  --single-transaction --quick \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Резервная копия файла конфигурации
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/ezpanel/

# Резервная копия загруженных файлов
tar -czf $BACKUP_DIR/upload_$DATE.tar.gz /var/lib/ezpanel/upload/

# Хранить только последние 14 дней
find $BACKUP_DIR -name "*.gz" -mtime +14 -delete

echo "$(date): Резервное копирование завершено" >> /var/log/ezpanel-backup.log
```

```bash
chmod +x /usr/local/bin/ezpanel-backup.sh

# Запускать ежедневно в 3:00
echo "0 3 * * * root /usr/local/bin/ezpanel-backup.sh" > /etc/cron.d/ezpanel-backup
```

### Восстановление базы данных

```bash
gunzip -c /backup/ezpanel/db_20240101_030000.sql.gz | mysql -u ezpanel -p ezpanel
```

---

## Резервное копирование файлов конфигурации

```bash
# Создать резервную копию
tar -czf config_backup_$(date +%Y%m%d).tar.gz /etc/ezpanel/

# Восстановить
tar -xzf config_backup_20240101.tar.gz -C /
```

---

## Оптимизация производительности

### Включение кэша Redis

Redis существенно снижает нагрузку на базу данных. Рекомендуется включить в производственной среде:

```yaml
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0
```

```bash
# Настроить политику памяти Redis
echo "maxmemory 256mb" | sudo tee -a /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis
```

### Пул соединений с базой данных

Настройте в соответствии с конфигурацией сервера:

```yaml
database:
  max_idle_conns: 10     # Количество неактивных соединений
  max_open_conns: 100    # Максимальное количество соединений (не превышать max_connections MySQL)
```

Просмотр максимального числа соединений MySQL:
```sql
SHOW VARIABLES LIKE 'max_connections';
```

### Оптимизация MySQL

Отредактируйте `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
[mysqld]
innodb_buffer_pool_size = 512M   # Установите 50-70% от доступной памяти
innodb_log_file_size = 128M
max_connections = 200
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

```bash
sudo systemctl restart mysql
```

### Кэширование статических ресурсов в Nginx

```nginx
location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    proxy_pass http://127.0.0.1:7088;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

---

## Проверка работоспособности

### Проверка вручную

```bash
# Проверка работоспособности API
curl -s http://localhost:7088/api/v1/health

# Проверка подключения к базе данных
mysql -u ezpanel -p -e "SELECT 1;" ezpanel

# Проверка подключения к Redis
redis-cli ping
```

### Скрипт автоматического мониторинга

Создайте `/usr/local/bin/ezpanel-health.sh`:

```bash
#!/bin/bash
HEALTH_URL="http://localhost:7088/api/v1/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 $HEALTH_URL)

if [ "$RESPONSE" != "200" ]; then
    echo "$(date): Проверка работоспособности не пройдена (HTTP $RESPONSE). Перезапуск..." >> /var/log/ezpanel-health.log
    systemctl restart ezpanel
fi
```

```bash
chmod +x /usr/local/bin/ezpanel-health.sh
# Проверять каждые 5 минут
echo "*/5 * * * * root /usr/local/bin/ezpanel-health.sh" > /etc/cron.d/ezpanel-health
```

---

## Настройка брандмауэра

### Принцип минимально необходимых открытых портов

| Порт | Назначение | Рекомендация |
|------|-----------|--------------|
| 22 | SSH | Перенести на нестандартный порт, ограничить по IP-адресам источника |
| 80 | HTTP | Открыть (перенаправление на HTTPS) |
| 443 | HTTPS | Открыть |
| 7088 | Прямой доступ к EzPanel | Закрыть при использовании обратного прокси Nginx |
| 3306 | MySQL | Только локальный доступ, не открывать наружу |
| 6379 | Redis | Только локальный доступ, не открывать наружу |

### Пример конфигурации UFW

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### Порты узлов

Порты, прослушиваемые узлами XrayM (VMess, VLESS, Trojan и т. д.), необходимо открыть в брандмауэре сервера узла.

---

## Процедура обновления

### Проверки перед обновлением

1. Создать резервную копию базы данных
2. Создать резервную копию файлов конфигурации
3. Ознакомиться с примечаниями к выпуску (Release Notes) и проверить наличие несовместимых изменений

### Выполнение обновления

См. [Установка и развёртывание — Обновление](./01_installation_RU.md#обновление).

### Проверка после обновления

```bash
# Проверить статус службы
systemctl status ezpanel

# Проверить журнал на наличие ошибок
journalctl -u ezpanel -n 50

# Тестовый вход
curl -s -X POST http://localhost:7088/api/v1/passport/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@opine.work","password":"admin123"}' | python3 -m json.tool
```

---

## Миграция данных

### Перенос на новый сервер

1. Создать резервную копию базы данных и файлов конфигурации на старом сервере
2. Установить ту же версию EzPanel на новом сервере
3. Восстановить базу данных и файлы конфигурации
4. Обновить записи DNS для указания на новый сервер
5. После проверки работоспособности нового сервера остановить старый

```bash
# Старый сервер: экспорт данных
mysqldump -u ezpanel -p ezpanel > ezpanel_full.sql
tar -czf ezpanel_config.tar.gz /etc/ezpanel/ /var/lib/ezpanel/upload/

# Новый сервер: импорт данных
mysql -u ezpanel -p ezpanel < ezpanel_full.sql
tar -xzf ezpanel_config.tar.gz -C /
systemctl restart ezpanel
```

---

## Обновление SSL-сертификатов

### Автоматическое обновление Let's Encrypt

```bash
# Тестовое обновление
sudo certbot renew --dry-run

# Просмотр сроков действия сертификатов
sudo certbot certificates
```

После установки Certbot автоматически настраивает задание по расписанию — ручное вмешательство обычно не требуется. Сертификаты обновляются автоматически за 30 дней до истечения срока действия.
