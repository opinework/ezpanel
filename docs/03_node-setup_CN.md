# 节点对接

**语言**: [English](./03_node-setup.md) | 简体中文 | [繁體中文](./03_node-setup_TW.md) | [Русский](./03_node-setup_RU.md) | [فارسی](./03_node-setup_FA.md)

EzPanel 支持 XrayM 和 XrayR 两种节点客户端，推荐使用 XrayM。

![节点入站配置](../images/inbound.png)

---

## XrayM vs XrayR 对比

| 特性 | XrayR | XrayM（推荐） |
|------|-------|--------------|
| 多协议支持 | 单进程仅支持单协议 | 单进程同时支持多协议 |
| 多端口支持 | 每端口需独立节点 | 单节点多端口 |
| 节点监控 | 不支持 | CPU / 内存 / 网络 / 在线用户 |
| 配置管理 | 手动维护多个配置文件 | 面板自动下发所有协议配置 |
| 资源占用 | 多协议需多个进程 | 单进程，资源占用低 |
| 热更新 | 不支持 | 配置变更自动生效，无需重启 |

**推荐场景**：
- 需要同时提供多种协议（VMess + VLESS + Trojan 等）
- 需要实时节点监控
- 希望简化运维、降低资源占用

---

## 在面板中创建节点

进入 **管理后台 → 系统管理 → 节点管理**，点击「新建节点」：

### 基本信息

| 字段 | 说明 |
|------|------|
| 节点名称 | 显示给用户的名称，如「香港 01」 |
| 节点地址 | 节点服务器的 IP 或域名 |
| 节点类型 | 选择 XrayM 或 XrayR |
| 节点 ID | 创建后自动分配，填写到节点配置文件中 |

### 协议配置（XrayM）

切换到「协议配置」标签，点击「快速添加协议」选择预设组合：

| 组合 | 包含协议 | 说明 |
|------|----------|------|
| 常用协议 | VMess(WS) + VLESS(WS) + Trojan(WS) + SS | 推荐，覆盖主流客户端 |
| VMess 组合 | VMess(TCP/WS/gRPC) | 兼容性最好 |
| VLESS 组合 | VLESS(TCP/WS/gRPC/REALITY) | 性能最优 |
| Trojan 组合 | Trojan(TCP/WS/gRPC) | 伪装性好 |
| Shadowsocks | SS(AES-256-GCM / 2022-BLAKE3) | 轻量 |

> 端口会自动分配，可在创建后手动修改。重复端口会自动跳过。

---

## 安装 XrayM

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

### 通用二进制

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym-linux-amd64.tar.gz
tar -xzf xraym-linux-amd64.tar.gz
sudo cp xraym /usr/bin/
sudo chmod +x /usr/bin/xraym
```

---

## 配置 XrayM

复制示例配置并编辑：

```bash
sudo cp /etc/xraym/config.example.yml /etc/xraym/config.yml
sudo nano /etc/xraym/config.yml
```

### 最小配置（自动模式，推荐）

```yaml
Log:
  Level: warning

Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"   # 面板地址
      ApiKey: "your-node-api-key"            # 节点通信密钥
      NodeID: 1                              # 节点 ID
    ControllerConfig:
      ListenIP: "0.0.0.0"
      UpdatePeriodic: 60
```

`ApiKey` 在管理后台 → 系统设置 → 节点通信密钥中查看。

### 多节点配置

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

### TLS 证书配置

**方式一：本地证书文件**

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

**方式二：DNS 自动申请（ACME）**

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

支持的 DNS 提供商：`cloudflare`、`alidns`、`dnspod`、`namesilo` 等。

### 自动限速配置

```yaml
ControllerConfig:
  UpdatePeriodic: 60
  AutoSpeedLimitConfig:
    Limit: 100         # 触发阈值（Mbps），0 禁用
    WarnTimes: 3       # 连续超出多少次后限速（0 立即限速）
    LimitSpeed: 10     # 限速至（Mbps）
    LimitDuration: 30  # 限速持续时间（分钟）
```

触发条件：`60秒内流量 > 100 Mbps × 60s × 125000 = 750 MB`

### 全局设备限制（需要 Redis）

```yaml
ControllerConfig:
  GlobalDeviceLimitConfig:
    Enable: true
    RedisAddr: "127.0.0.1:6379"
    RedisPassword: ""
```

---

## 启动 XrayM

```bash
sudo systemctl enable --now xraym

# 查看状态
sudo systemctl status xraym

# 查看日志
sudo journalctl -u xraym -f
```

成功启动后，日志中会出现类似以下内容：

```
INFO  XrayM started, node: 香港 01 (ID: 1)
INFO  Inbound VMess:443 started
INFO  Inbound Trojan:8443 started
```

---

## 节点监控

XrayM 每 60 秒自动向面板上报一次节点状态。

![节点监控](../images/monitor.png)

**监控指标**：
- CPU 使用率
- 内存使用量
- 磁盘使用情况
- 实时网络上传 / 下载速率
- 在线用户数
- 累计流量统计

**查看方式**：管理后台 → 系统管理 → 健康检查

点击节点卡片可查看详细图表，数据每 10 秒自动刷新。

> XrayR 不支持节点监控功能，仅统计流量。

---

## 使用 XrayR（不推荐）

> XrayR 单进程仅支持单节点单端口单协议。多协议需要分别启动多个进程。

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

多协议需为每种协议创建独立的节点和配置文件，并启动多个 XrayR 服务。

---

## 生成 VLESS REALITY 密钥对

```bash
xraym x25519
```

输出示例：

```
Private key: <base64-private-key>
Public key:  <base64-public-key>
```

将 Public key 填入面板的节点协议配置中，Private key 保存在服务器本地（不需要填入面板）。
