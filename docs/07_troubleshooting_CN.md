# 常见问题与排查

**语言**: [English](./07_troubleshooting.md) | 简体中文 | [繁體中文](./07_troubleshooting_TW.md) | [Русский](./07_troubleshooting_RU.md) | [فارسی](./07_troubleshooting_FA.md)

遇到问题时，首先查看服务日志往往能快速定位原因：

```bash
# EzPanel 日志
sudo journalctl -u ezpanel -n 100 --no-pager

# XrayM 日志
sudo journalctl -u xraym -n 100 --no-pager

# Nginx 错误日志
sudo tail -n 50 /var/log/nginx/error.log
```

---

## 安装与启动问题

### 服务无法启动

**症状**：`systemctl status ezpanel` 显示 `failed` 或 `activating`

**排查步骤**：

```bash
sudo journalctl -u ezpanel -n 50
```

常见原因：

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `dial tcp: connection refused` | 数据库未启动 | `systemctl start mysql` |
| `Access denied for user` | 数据库密码错误 | 检查 `config.yaml` 中的 `database.password` |
| `Unknown database` | 数据库未创建 | 手动创建数据库，参见 [安装部署](./01_installation.md) |
| `bind: address already in use` | 端口被占用 | 修改 `server.port` 或释放占用进程 |
| `invalid configuration` | 配置文件格式错误 | 检查 YAML 缩进，避免使用 Tab |

### 数据库未自动创建表

EzPanel 会在首次启动时自动建表，但需确认：

1. 数据库用户有 `CREATE TABLE` 权限
2. 数据库已提前创建（系统不会自动创建数据库本身）
3. 日志中没有数据库连接错误

```sql
-- 检查用户权限
SHOW GRANTS FOR 'ezpanel'@'localhost';
```

---

## 登录与认证问题

### 无法登录 / 忘记密码

使用默认管理员账号尝试：
- 邮箱: `admin@opine.work`
- 密码: `admin123`

如默认账号也无法登录，可直接修改数据库密码：

```bash
# 生成新密码的 bcrypt 哈希（需安装 htpasswd）
htpasswd -nbBC 10 "" "new_password" | tr -d ':\n'

# 或使用 Python
python3 -c "import bcrypt; print(bcrypt.hashpw(b'new_password', bcrypt.gensalt(10)).decode())"
```

```sql
UPDATE users SET password = '$2a$10$...' WHERE email = 'admin@opine.work';
```

### Token 频繁失效

原因：服务器重启后 JWT 密钥可能变化，或 Token 过期时间设置过短。

解决：
1. 确认 `jwt.secret` 在 `config.yaml` 中已固定（不要使用随机生成的）
2. 适当增大 `jwt.expire`（默认 168 小时 = 7 天）

### 2FA 验证码无效

可能原因：服务器时间与手机时间不同步（TOTP 依赖时间精确度）

```bash
# 同步服务器时间
sudo timedatectl set-ntp true
timedatectl status
```

---

## 数据库问题

### 数据库连接失败

```bash
# 检查 MySQL 是否运行
systemctl status mysql

# 测试连接
mysql -u ezpanel -p -h 127.0.0.1 ezpanel -e "SELECT 1;"
```

常见原因：
- MySQL 服务未启动
- 用户名 / 密码错误
- 数据库名不存在
- 防火墙阻止了 3306 端口（远程连接时）

### 数据库性能慢

```sql
-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query_log';
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 查看当前连接
SHOW PROCESSLIST;

-- 查看 InnoDB 状态
SHOW ENGINE INNODB STATUS\G
```

调整 `innodb_buffer_pool_size`（建议为内存的 50-70%）并重启 MySQL。

---

## 节点对接问题

### 节点无法连接面板

**排查步骤**：

1. 检查面板地址和 API Key 是否正确
2. 在节点服务器上测试网络连通性：
   ```bash
   curl -v https://panel.example.com/api/v1/server/UniProxy/config?node_id=1 \
     -H "Authorization: Bearer your-node-api-key"
   ```
3. 检查面板是否允许来自节点 IP 的请求（防火墙 / Cloudflare 设置）
4. 查看 XrayM 日志：
   ```bash
   journalctl -u xraym -f
   ```

### 节点监控数据不显示

1. 确认使用的是 XrayM（XrayR 不支持监控）
2. 确认 `config.yaml` 中 `node.report_metrics: true`
3. 查看 XrayM 日志中是否有 `report` 相关错误

### 用户无法连接节点

排查顺序：

1. 确认节点服务在运行：`systemctl status xraym`
2. 确认节点端口已在防火墙开放
3. 检查订阅链接是否已更新（客户端重新拉取订阅）
4. 查看 XrayM 日志中的连接错误

```bash
# 测试端口是否监听
ss -tlnp | grep 443

# 测试端口连通性（从客户端测试）
nc -zv node.example.com 443
```

### XrayM 配置热更新不生效

XrayM 监听配置文件变化并自动重载。如未生效：

```bash
# 手动重启
sudo systemctl restart xraym
sudo journalctl -u xraym -f
```

---

## 界面与功能问题

### 页面显示 502 Bad Gateway

原因：EzPanel 服务未运行或 Nginx 代理配置错误。

```bash
# 检查 EzPanel 是否运行
systemctl status ezpanel
curl http://127.0.0.1:7088/api/v1/health

# 检查 Nginx 配置
nginx -t
```

### 文件上传失败

1. 检查上传目录权限：
   ```bash
   ls -la /var/lib/ezpanel/upload/
   # 应为 ezpanel:ezpanel 所有
   sudo chown -R ezpanel:ezpanel /var/lib/ezpanel/upload/
   ```
2. 检查上传大小限制：
   - `config.yaml` 中的 `upload.max_size`
   - Nginx 中的 `client_max_body_size`：
     ```nginx
     client_max_body_size 20m;
     ```

### 邮件发送失败

1. 检查 SMTP 配置（管理后台 → 系统设置 → 邮件设置）
2. 使用「发送测试邮件」功能验证
3. 常见问题：
   - Gmail 需要使用「应用专用密码」而非账号密码
   - 部分服务商需要开启 SMTP 服务
   - 检查服务器是否被邮件服务商 IP 黑名单

### 专业版功能未显示

1. 确认 `ezpanel-pro` 已安装：
   ```bash
   which ezpanel-pro
   systemctl status ezpanel
   ```
2. 确认 `config.yaml` 中 `commercial` 配置已取消注释
3. 确认授权文件存在且格式正确：
   ```bash
   cat /etc/ezpanel/license.json
   ```
4. 查看日志中是否有授权相关错误：
   ```bash
   journalctl -u ezpanel -n 50 | grep -i "license\|commercial"
   ```

---

## 性能问题

### 面板响应慢

**诊断**：

```bash
# 查看系统负载
top
htop

# 查看内存使用
free -h

# 查看磁盘 I/O
iostat -x 1

# 查看慢 API（开启 debug 日志后）
journalctl -u ezpanel | grep "latency"
```

**常见优化措施**：

1. 启用 Redis 缓存
2. 调低日志级别为 `warn`
3. 增大数据库连接池
4. 为 MySQL 调整 `innodb_buffer_pool_size`
5. 确认磁盘 I/O 正常（SSD 优于 HDD）

### 内存不足

```bash
# 查看内存占用
ps aux --sort=-%mem | head -20
```

- EzPanel 主进程正常占用约 50-200 MB
- 如内存不足，考虑升级服务器或关闭 Swagger（`swagger.enable: false`）

---

## 重置管理员账号

如所有管理员账号都无法访问，可以直接修改数据库：

```sql
-- 查找管理员账号
SELECT id, email, is_admin FROM users WHERE is_admin = 1;

-- 重置密码为 admin123
UPDATE users
SET password = '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi'
WHERE email = 'admin@opine.work';

-- 解封账号
UPDATE users SET banned = 0 WHERE email = 'admin@opine.work';
```

> 密码哈希 `$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi` 对应明文 `password`，登录后请立即修改。

---

## 获取帮助

EzPanel 社区版官方不提供技术支持。如需帮助，可以：

1. 加入 Telegram 群组 [@OpineWorkOfficial](https://t.me/OpineWorkOfficial) 向社区寻求帮助
2. 购买专业版或商业版以获得官方支持，详见 [https://opine.work](https://opine.work)
