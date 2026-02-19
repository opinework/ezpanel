# Установка и развёртывание

**Язык**: [English](./01_installation.md) | [简体中文](./01_installation_CN.md) | [繁體中文](./01_installation_TW.md) | Русский | [فارسی](./01_installation_FA.md)

## Системные требования

### Требования к оборудованию

| Параметр | Минимум | Рекомендуется |
|----------|---------|---------------|
| CPU | 1 ядро | 2 ядра+ |
| Память | 1 ГБ | 2 ГБ+ |
| Диск | 10 ГБ | 20 ГБ+ SSD |

### Программные требования

- **Операционная система**: Linux x86_64 / aarch64 (рекомендуется Arch Linux / Debian 12 / Ubuntu 22.04)
- **База данных**: MySQL 8.0+ или MariaDB 10.5+
- **Кэш**: Redis 6.0+ (опционально, для оптимизации производительности)

---

## Подготовка к установке

### 1. Создание базы данных

EzPanel не создаёт базу данных автоматически — необходимо сделать это вручную:

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

### 2. Установка Redis (опционально)

Redis является необязательным компонентом. При его включении повышается производительность системы (кэширование, глобальное ограничение устройств и т. д.).

```bash
# Debian / Ubuntu
sudo apt install redis-server
sudo systemctl enable --now redis-server

# Arch Linux
sudo pacman -S redis
sudo systemctl enable --now redis
```

---

## Установка EzPanel

### Способ 1: Пакет для Arch Linux

```bash
# Загрузить пакет
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-x86_64.pkg.tar.zst

# Установить
sudo pacman -U ezpanel-*.pkg.tar.zst
```

### Способ 2: Пакет для Debian / Ubuntu

```bash
# Загрузить пакет
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel_amd64.deb

# Установить
sudo dpkg -i ezpanel_*.deb
sudo apt-get install -f
```

### Способ 3: Универсальный бинарный файл (Tarball)

Для других дистрибутивов Linux:

```bash
# Загрузить и распаковать
wget https://github.com/opinework/ezpanel/releases/latest/download/ezpanel-linux-amd64.tar.gz
tar -xzf ezpanel-linux-amd64.tar.gz
cd ezpanel-*

# Установить бинарный файл
sudo cp ezpanel /usr/bin/
sudo chmod +x /usr/bin/ezpanel

# Создать каталоги конфигурации и среды выполнения
sudo mkdir -p /etc/ezpanel /run/ezpanel /var/lib/ezpanel /var/log/ezpanel

# Создать системного пользователя
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/ezpanel ezpanel
sudo chown ezpanel:ezpanel /var/lib/ezpanel /var/log/ezpanel /run/ezpanel

# Скопировать файл конфигурации
sudo cp config.yaml.example /etc/ezpanel/config.yaml
sudo chown ezpanel:ezpanel /etc/ezpanel/config.yaml
sudo chmod 600 /etc/ezpanel/config.yaml

# Установить службу systemd
sudo cp ezpanel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## Начальная настройка

Отредактируйте файл конфигурации `/etc/ezpanel/config.yaml`, указав как минимум параметры подключения к базе данных и секретный ключ JWT:

```yaml
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"   # Замените на реальный пароль
  dbname: "ezpanel"

jwt:
  secret: "change-this-to-a-random-secret"  # Обязательно измените!
  expire: 168
```

Подробное описание параметров конфигурации см. в [Руководстве по настройке](./02_configuration_RU.md).

---

## Запуск службы

```bash
sudo systemctl enable --now ezpanel

# Проверить статус
sudo systemctl status ezpanel

# Просмотр журнала запуска
sudo journalctl -u ezpanel -f
```

При первом запуске GORM автоматически создаёт все таблицы базы данных и инициализирует учётную запись администратора по умолчанию.

**Учётные данные по умолчанию**:
- Email: `admin@opine.work`
- Пароль: `admin123`

> **Предупреждение безопасности**: После входа немедленно смените пароль и адрес электронной почты по умолчанию.

---

## Настройка обратного прокси Nginx (рекомендуется)

В производственной среде рекомендуется развернуть обратный прокси Nginx перед EzPanel для поддержки HTTPS и оптимизации статических ресурсов.

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

    # Кэширование статических ресурсов
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

### Получение SSL-сертификата

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com
```

---

## Обновление

### Обновление пакета Arch Linux / Debian

```bash
# Загрузить новый пакет, затем переустановить
sudo pacman -U ezpanel-new-version.pkg.tar.zst    # Arch
sudo dpkg -i ezpanel_new_version_amd64.deb         # Debian
```

После установки служба перезапускается автоматически.

### Обновление бинарного файла

```bash
# Остановить службу
sudo systemctl stop ezpanel

# Создать резервную копию старой версии
sudo cp /usr/bin/ezpanel /usr/bin/ezpanel.bak

# Заменить новой версией
sudo cp ezpanel-new /usr/bin/ezpanel
sudo chmod +x /usr/bin/ezpanel

# Запустить службу
sudo systemctl start ezpanel
sudo journalctl -u ezpanel -f  # Наблюдать за журналом запуска
```

---

## Установка профессиональной версии (Pro / Enterprise)

Профессиональная версия запускается как отдельный процесс и взаимодействует с основным процессом EzPanel через Unix Socket.

### Предварительные требования

- Сообщество EzPanel уже работает нормально
- Получен действующий файл лицензии `license.json`

### Шаги установки

```bash
# Установить ezpanel-pro (аналогично установке EzPanel)
sudo dpkg -i ezpanel-pro_*.deb    # Debian/Ubuntu
sudo pacman -U ezpanel-pro-*.pkg.tar.zst  # Arch

# Разместить файл лицензии
sudo cp license.json /etc/ezpanel/license.json
sudo chown ezpanel:ezpanel /etc/ezpanel/license.json
sudo chmod 600 /etc/ezpanel/license.json
```

### Включение коммерческого модуля в файле конфигурации

Отредактируйте `/etc/ezpanel/config.yaml`, раскомментируйте следующие строки:

```yaml
commercial:
  pro_binary_path: "/usr/bin/ezpanel-pro"
  license_path: "/etc/ezpanel/license.json"
  socket_path: "/run/ezpanel/commercial.sock"
```

### Перезапуск службы

```bash
sudo systemctl restart ezpanel
sudo journalctl -u ezpanel -f
```

При запуске основного процесса EzPanel автоматически запускает дочерний процесс `ezpanel-pro`. Наличие в журнале записи `commercial module started` означает, что профессиональная версия активирована.
