# Подключение узлов

**Язык**: [English](./03_node-setup.md) | [简体中文](./03_node-setup_CN.md) | [繁體中文](./03_node-setup_TW.md) | Русский | [فارسی](./03_node-setup_FA.md)

EzPanel поддерживает два клиента для узлов — XrayM и XrayR. Рекомендуется использовать XrayM.

![Настройка входящих соединений узла](../images/inbound.png)

---

## Сравнение XrayM и XrayR

| Характеристика | XrayR | XrayM (рекомендуется) |
|----------------|-------|-----------------------|
| Поддержка нескольких протоколов | Один процесс поддерживает только один протокол | Один процесс поддерживает несколько протоколов одновременно |
| Поддержка нескольких портов | Каждый порт требует отдельного узла | Несколько портов на один узел |
| Мониторинг узла | Не поддерживается | CPU / Память / Сеть / Онлайн-пользователи |
| Управление конфигурацией | Ручное ведение нескольких файлов конфигурации | Панель автоматически распространяет все конфигурации протоколов |
| Потребление ресурсов | Несколько процессов для нескольких протоколов | Один процесс, низкое потребление ресурсов |
| Горячее обновление | Не поддерживается | Изменения конфигурации вступают в силу автоматически без перезапуска |

**Рекомендуемые сценарии**:
- Требуется поддержка нескольких протоколов одновременно (VMess + VLESS + Trojan и др.)
- Требуется мониторинг узлов в реальном времени
- Упрощение операций и снижение потребления ресурсов

---

## Создание узла в панели управления

Перейдите в **Панель управления → Системное администрирование → Управление узлами** и нажмите «Создать узел»:

### Основная информация

| Поле | Описание |
|------|----------|
| Имя узла | Имя, отображаемое пользователям, например «Гонконг 01» |
| Адрес узла | IP-адрес или доменное имя сервера узла |
| Тип узла | Выберите XrayM или XrayR |
| ID узла | Назначается автоматически после создания; укажите его в файле конфигурации узла |

### Настройка протоколов (XrayM)

Перейдите на вкладку «Конфигурация протоколов» и нажмите «Быстрое добавление протокола» для выбора предустановленного набора:

| Набор | Включаемые протоколы | Описание |
|-------|----------------------|----------|
| Основные протоколы | VMess(WS) + VLESS(WS) + Trojan(WS) + SS | Рекомендуется; охватывает основные клиенты |
| Набор VMess | VMess(TCP/WS/gRPC) | Максимальная совместимость |
| Набор VLESS | VLESS(TCP/WS/gRPC/REALITY) | Оптимальная производительность |
| Набор Trojan | Trojan(TCP/WS/gRPC) | Хорошая маскировка |
| Shadowsocks | SS(AES-256-GCM / 2022-BLAKE3) | Лёгкий вариант |

> Порты назначаются автоматически; их можно изменить вручную после создания. Дублирующиеся порты пропускаются автоматически.

---

## Установка XrayM

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

### Универсальный бинарный файл

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym-linux-amd64.tar.gz
tar -xzf xraym-linux-amd64.tar.gz
sudo cp xraym /usr/bin/
sudo chmod +x /usr/bin/xraym
```

---

## Настройка XrayM

Скопируйте пример конфигурации и отредактируйте его:

```bash
sudo cp /etc/xraym/config.example.yml /etc/xraym/config.yml
sudo nano /etc/xraym/config.yml
```

### Минимальная конфигурация (автоматический режим, рекомендуется)

```yaml
Log:
  Level: warning

Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"   # Адрес панели
      ApiKey: "your-node-api-key"            # Ключ API узла
      NodeID: 1                              # ID узла
    ControllerConfig:
      ListenIP: "0.0.0.0"
      UpdatePeriodic: 60
```

`ApiKey` можно найти в разделе «Панель управления → Системные настройки → Ключ API узла».

### Конфигурация нескольких узлов

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

### Настройка TLS-сертификатов

**Способ 1: Локальный файл сертификата**

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

**Способ 2: Автоматическое получение через DNS (ACME)**

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

Поддерживаемые DNS-провайдеры: `cloudflare`, `alidns`, `dnspod`, `namesilo` и другие.

### Конфигурация автоматического ограничения скорости

```yaml
ControllerConfig:
  UpdatePeriodic: 60
  AutoSpeedLimitConfig:
    Limit: 100         # Порог срабатывания (Мбит/с), 0 — отключено
    WarnTimes: 3       # Количество последовательных превышений до ограничения (0 — немедленное ограничение)
    LimitSpeed: 10     # Скорость ограничения (Мбит/с)
    LimitDuration: 30  # Длительность ограничения (в минутах)
```

Условие срабатывания: `трафик за 60 с > 100 Мбит/с × 60 с × 125000 = 750 МБ`

### Глобальное ограничение устройств (требуется Redis)

```yaml
ControllerConfig:
  GlobalDeviceLimitConfig:
    Enable: true
    RedisAddr: "127.0.0.1:6379"
    RedisPassword: ""
```

---

## Запуск XrayM

```bash
sudo systemctl enable --now xraym

# Проверить статус
sudo systemctl status xraym

# Просмотр журнала
sudo journalctl -u xraym -f
```

После успешного запуска в журнале появятся записи вида:

```
INFO  XrayM started, node: 香港 01 (ID: 1)
INFO  Inbound VMess:443 started
INFO  Inbound Trojan:8443 started
```

---

## Мониторинг узлов

XrayM автоматически отправляет статус узла в панель каждые 60 секунд.

![Мониторинг узла](../images/monitor.png)

**Отслеживаемые показатели**:
- Загрузка ЦП
- Использование памяти
- Состояние диска
- Скорость загрузки / выгрузки в реальном времени
- Количество онлайн-пользователей
- Статистика суммарного трафика

**Просмотр**: Панель управления → Системное администрирование → Проверка работоспособности

Нажмите на карточку узла для просмотра подробных графиков; данные обновляются автоматически каждые 10 секунд.

> XrayR не поддерживает мониторинг узлов — собирается только статистика трафика.

---

## Использование XrayR (не рекомендуется)

> Один процесс XrayR поддерживает только один узел, один порт и один протокол. Для нескольких протоколов необходимо запускать несколько процессов.

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

Для нескольких протоколов необходимо создать отдельный узел и файл конфигурации для каждого протокола и запустить несколько служб XrayR.

---

## Генерация ключевой пары VLESS REALITY

```bash
xraym x25519
```

Пример вывода:

```
Private key: <base64-private-key>
Public key:  <base64-public-key>
```

Введите Public key в конфигурацию протокола узла в панели. Private key храните на сервере локально (в панель вводить не требуется).
