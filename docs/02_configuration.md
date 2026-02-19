# Configuration

**Language**: English | [简体中文](./02_configuration_CN.md) | [繁體中文](./02_configuration_TW.md) | [Русский](./02_configuration_RU.md) | [فارسی](./02_configuration_FA.md)

The main configuration file for EzPanel is located at `/etc/ezpanel/config.yaml`.

---

## Full Configuration Reference

```yaml
# ================================
# EzPanel Configuration File
# ================================

# Server settings
server:
  host: "0.0.0.0"        # Listening address; 0.0.0.0 means all interfaces
  port: 7088              # Listening port, default 7088
  mode: "release"         # Run mode: debug / release / test

# Database settings (required)
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"
  dbname: "ezpanel"
  charset: "utf8mb4"
  max_idle_conns: 10      # Maximum idle connections
  max_open_conns: 100     # Maximum open connections

# Redis settings (optional)
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0

# JWT settings
jwt:
  secret: "change-this-to-a-random-string"  # Must be changed!
  expire: 168             # Token expiry in hours; 168 = 7 days
  issuer: "ezpanel"

# Logging settings
log:
  level: "warn"           # Log level: debug / info / warn / error
  format: "console"       # Output format: json / console
  output: "stdout"        # Output destination: stdout / file / both
  file_path: "/var/log/ezpanel/app.log"

# Swagger API documentation
swagger:
  enable: false           # Recommended to disable in production

# Node monitoring (XrayM)
node:
  report_metrics: true    # Enable node metrics reporting

# Upload settings
upload:
  path: "/var/lib/ezpanel/upload"
  max_size: 10            # Maximum file size per upload (MB)
  allowed_types:
    - image/jpeg
    - image/png
    - image/gif
    - image/webp

# CORS settings
cors:
  allow_origins:
    - "*"                 # Change to your actual domain in production, e.g. https://panel.example.com
  allow_credentials: true
  max_age: 12

# Commercial module (Pro / Enterprise edition)
# commercial:
#   pro_binary_path: "/usr/bin/ezpanel-pro"
#   license_path: "/etc/ezpanel/license.json"
#   socket_path: "/run/ezpanel/commercial.sock"
```

---

## Configuration Reference

### server

| Option | Default | Description |
|--------|---------|-------------|
| `host` | `0.0.0.0` | Listening address. Set to `127.0.0.1` if access is only through a reverse proxy (Nginx). |
| `port` | `7088` | Listening port |
| `mode` | `release` | `debug` mode outputs verbose logs; use `release` in production. |

### database

| Option | Description |
|--------|-------------|
| `host` | MySQL host address |
| `port` | MySQL port, default 3306 |
| `user` | Database username |
| `password` | Database password |
| `dbname` | Database name (must be created manually beforehand) |
| `max_open_conns` | Maximum connections; adjust based on server capacity, recommended 50–200 |

### redis

Redis is an optional component. The system will function without it, but the following features will be unavailable:

- Distributed cache acceleration
- XrayM cross-node global device limits

### jwt

| Option | Description |
|--------|-------------|
| `secret` | JWT signing key. **Must be changed to a random string.** Leaking this key is a security risk. |
| `expire` | Token validity period (hours). Changing this does not affect existing tokens until they expire. |

Generate a random key:
```bash
openssl rand -hex 32
```

### log

| Level | Description | Recommended Use |
|-------|-------------|-----------------|
| `debug` | Outputs all logs, including SQL statements | Local development and debugging |
| `info` | Outputs general information | Test environments |
| `warn` | Outputs only warnings and errors | **Recommended for production** |
| `error` | Outputs only errors | Stable production environments |

---

## Admin Panel System Settings

The following settings have been moved to **Admin Panel → System Settings** and do not need to be set in the config file:

### Site Information

- Site name, Logo
- Site URL, subscription URL
- Registration settings (open registration, invite codes)

### Email (SMTP)

- SMTP server address and port
- Username, password
- Sender address
- Email template management

### Telegram Bot (Optional)

1. Create a Bot via [@BotFather](https://t.me/BotFather) and obtain a token
2. Enter the Bot token in the admin panel
3. Click "Set Webhook" to complete the configuration

Users can bind their accounts using the following commands:
- `/bindaccount` — Initiate binding
- `/bindcode <verification_code>` — Complete binding
- `/bindstatus` — Check binding status

### Payment Gateways (Pro Edition)

Payment settings are managed in Admin Panel → Payment Settings and support:

| Type | Description |
|------|-------------|
| EPay | Aggregated payment platform supporting Alipay, WeChat Pay, etc. Recommended. |
| Alipay Face-to-Face | QR code payment |
| Alipay Web Payment | Redirect payment |
| WeChat Pay | Native QR code payment |
| Stripe | International credit card payment |

### Node API Key

The API key used for communication between nodes and the panel. View and modify it in Admin Panel → System Settings → Node API Key.

---

## Multi-Instance Deployment

To run multiple panel instances on the same server, change the port and config directory for each instance:

```yaml
# Config for instance 2
server:
  port: 7089

database:
  dbname: "ezpanel2"
```

Then create a corresponding systemd service file for each instance.
