# 配置说明

**语言**: [English](./02_configuration.md) | 简体中文 | [繁體中文](./02_configuration_TW.md) | [Русский](./02_configuration_RU.md) | [فارسی](./02_configuration_FA.md)

EzPanel 的主配置文件位于 `/etc/ezpanel/config.yaml`。

---

## 完整配置参考

```yaml
# ================================
# EzPanel 配置文件
# ================================

# 服务器配置
server:
  host: "0.0.0.0"        # 监听地址，0.0.0.0 表示所有网卡
  port: 7088              # 监听端口，默认 7088
  mode: "release"         # 运行模式: debug / release / test

# 数据库配置（必填）
database:
  host: "127.0.0.1"
  port: 3306
  user: "ezpanel"
  password: "your_password"
  dbname: "ezpanel"
  charset: "utf8mb4"
  max_idle_conns: 10      # 最大空闲连接数
  max_open_conns: 100     # 最大打开连接数

# Redis 配置（可选）
redis:
  host: "127.0.0.1"
  port: 6379
  password: ""
  db: 0

# JWT 配置
jwt:
  secret: "change-this-to-a-random-string"  # 务必修改！
  expire: 168             # Token 过期时间（小时），168 = 7 天
  issuer: "ezpanel"

# 日志配置
log:
  level: "warn"           # 日志级别: debug / info / warn / error
  format: "console"       # 输出格式: json / console
  output: "stdout"        # 输出位置: stdout / file / both
  file_path: "/var/log/ezpanel/app.log"

# Swagger API 文档
swagger:
  enable: false           # 生产环境建议关闭

# 节点监控（XrayM）
node:
  report_metrics: true    # 启用节点监控数据上报

# 上传配置
upload:
  path: "/var/lib/ezpanel/upload"
  max_size: 10            # 单文件最大大小（MB）
  allowed_types:
    - image/jpeg
    - image/png
    - image/gif
    - image/webp

# CORS 跨域配置
cors:
  allow_origins:
    - "*"                 # 生产环境改为实际域名，如 https://panel.example.com
  allow_credentials: true
  max_age: 12

# 商业模块（Pro / Enterprise 版）
# commercial:
#   pro_binary_path: "/usr/bin/ezpanel-pro"
#   license_path: "/etc/ezpanel/license.json"
#   socket_path: "/run/ezpanel/commercial.sock"
```

---

## 配置项说明

### server

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `host` | `0.0.0.0` | 监听地址。如只允许本地访问（Nginx 反代），可改为 `127.0.0.1` |
| `port` | `7088` | 监听端口 |
| `mode` | `release` | `debug` 模式会输出详细日志，生产环境使用 `release` |

### database

| 配置项 | 说明 |
|--------|------|
| `host` | MySQL 主机地址 |
| `port` | MySQL 端口，默认 3306 |
| `user` | 数据库用户名 |
| `password` | 数据库密码 |
| `dbname` | 数据库名（需提前手动创建） |
| `max_open_conns` | 最大连接数，根据服务器配置调整，建议 50-200 |

### redis

Redis 为可选组件。未配置时系统仍可正常运行，但以下功能不可用：

- 分布式缓存加速
- XrayM 跨节点全局设备限制

### jwt

| 配置项 | 说明 |
|--------|------|
| `secret` | JWT 签名密钥，**必须修改为随机字符串**，泄露会导致安全风险 |
| `expire` | Token 有效期（小时），修改后现有 Token 不受影响直到过期 |

生成随机密钥：
```bash
openssl rand -hex 32
```

### log

| 级别 | 说明 | 适用场景 |
|------|------|----------|
| `debug` | 输出所有日志，包含 SQL 语句 | 本地开发调试 |
| `info` | 输出一般信息 | 测试环境 |
| `warn` | 仅输出警告和错误 | **生产环境推荐** |
| `error` | 仅输出错误 | 稳定生产环境 |

---

## 后台系统设置

以下配置项已移至 **管理后台 → 系统设置**，无需在配置文件中填写：

### 站点信息

- 站点名称、Logo
- 站点 URL、订阅 URL
- 注册设置（是否开放注册、邀请码）

### 邮件（SMTP）

- SMTP 服务器地址和端口
- 用户名、密码
- 发件人地址
- 邮件模板管理

### Telegram Bot（可选）

1. 在 [@BotFather](https://t.me/BotFather) 创建 Bot，获取 Token
2. 在管理后台填入 Bot Token
3. 点击「设置 Webhook」完成配置

用户可使用以下命令绑定账号：
- `/bindaccount` — 发起绑定
- `/bindcode <验证码>` — 完成绑定
- `/bindstatus` — 查看绑定状态

### 支付网关（Pro 版）

支付配置在管理后台 → 支付设置中管理，支持：

| 类型 | 说明 |
|------|------|
| EPay | 聚合支付平台，支持支付宝、微信等，推荐 |
| 支付宝当面付 | 扫码支付 |
| 支付宝网页支付 | 跳转支付 |
| 微信支付 | 原生二维码支付 |
| Stripe | 国际信用卡支付 |

### 节点通信密钥

节点与面板通信所用的 API Key，在管理后台 → 系统设置 → 节点通信密钥中查看和修改。

---

## 多实例部署

如需在同一台服务器运行多个面板实例，只需修改每个实例的端口和配置目录：

```yaml
# 实例 2 的配置
server:
  port: 7089

database:
  dbname: "ezpanel2"
```

并创建对应的 systemd 服务文件即可。
