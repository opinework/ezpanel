# 節點對接

**語言**: [English](./03_node-setup.md) | [简体中文](./03_node-setup_CN.md) | 繁體中文 | [Русский](./03_node-setup_RU.md) | [فارسی](./03_node-setup_FA.md)

EzPanel 支援 XrayM 和 XrayR 兩種節點客戶端，推薦使用 XrayM。

![節點入站設定](../images/inbound.png)

---

## XrayM vs XrayR 比較

| 特性 | XrayR | XrayM（推薦） |
|------|-------|--------------|
| 多協議支援 | 單程序僅支援單協議 | 單程序同時支援多協議 |
| 多連接埠支援 | 每個連接埠需獨立節點 | 單節點多連接埠 |
| 節點監控 | 不支援 | CPU / 記憶體 / 網路 / 線上使用者 |
| 設定管理 | 手動維護多個設定檔 | 面板自動下發所有協議設定 |
| 資源佔用 | 多協議需多個程序 | 單程序，資源佔用低 |
| 熱更新 | 不支援 | 設定變更自動生效，無需重啟 |

**推薦場景**：
- 需要同時提供多種協議（VMess + VLESS + Trojan 等）
- 需要即時節點監控
- 希望簡化維運、降低資源佔用

---

## 在面板中建立節點

進入 **管理後台 → 系統管理 → 節點管理**，點擊「新建節點」：

### 基本資訊

| 欄位 | 說明 |
|------|------|
| 節點名稱 | 顯示給使用者的名稱，如「香港 01」 |
| 節點位址 | 節點伺服器的 IP 或網域名稱 |
| 節點類型 | 選擇 XrayM 或 XrayR |
| 節點 ID | 建立後自動分配，填寫到節點設定檔中 |

### 協議設定（XrayM）

切換到「協議設定」標籤，點擊「快速新增協議」選擇預設組合：

| 組合 | 包含協議 | 說明 |
|------|----------|------|
| 常用協議 | VMess(WS) + VLESS(WS) + Trojan(WS) + SS | 推薦，涵蓋主流客戶端 |
| VMess 組合 | VMess(TCP/WS/gRPC) | 相容性最好 |
| VLESS 組合 | VLESS(TCP/WS/gRPC/REALITY) | 效能最優 |
| Trojan 組合 | Trojan(TCP/WS/gRPC) | 偽裝性好 |
| Shadowsocks | SS(AES-256-GCM / 2022-BLAKE3) | 輕量 |

> 連接埠會自動分配，可在建立後手動修改。重複連接埠會自動跳過。

---

## 安裝 XrayM

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

### 通用二進位

```bash
wget https://github.com/opinework/xraym/releases/latest/download/xraym-linux-amd64.tar.gz
tar -xzf xraym-linux-amd64.tar.gz
sudo cp xraym /usr/bin/
sudo chmod +x /usr/bin/xraym
```

---

## 設定 XrayM

複製範例設定並編輯：

```bash
sudo cp /etc/xraym/config.example.yml /etc/xraym/config.yml
sudo nano /etc/xraym/config.yml
```

### 最小設定（自動模式，推薦）

```yaml
Log:
  Level: warning

Nodes:
  - PanelType: "EzPanel"
    ApiConfig:
      ApiHost: "https://panel.example.com"   # 面板位址
      ApiKey: "your-node-api-key"            # 節點通訊密鑰
      NodeID: 1                              # 節點 ID
    ControllerConfig:
      ListenIP: "0.0.0.0"
      UpdatePeriodic: 60
```

`ApiKey` 在管理後台 → 系統設定 → 節點通訊密鑰中查看。

### 多節點設定

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

### TLS 憑證設定

**方式一：本地憑證檔案**

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

**方式二：DNS 自動申請（ACME）**

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

支援的 DNS 提供商：`cloudflare`、`alidns`、`dnspod`、`namesilo` 等。

### 自動限速設定

```yaml
ControllerConfig:
  UpdatePeriodic: 60
  AutoSpeedLimitConfig:
    Limit: 100         # 觸發閾值（Mbps），0 禁用
    WarnTimes: 3       # 連續超出多少次後限速（0 立即限速）
    LimitSpeed: 10     # 限速至（Mbps）
    LimitDuration: 30  # 限速持續時間（分鐘）
```

觸發條件：`60秒內流量 > 100 Mbps × 60s × 125000 = 750 MB`

### 全域裝置限制（需要 Redis）

```yaml
ControllerConfig:
  GlobalDeviceLimitConfig:
    Enable: true
    RedisAddr: "127.0.0.1:6379"
    RedisPassword: ""
```

---

## 啟動 XrayM

```bash
sudo systemctl enable --now xraym

# 查看狀態
sudo systemctl status xraym

# 查看日誌
sudo journalctl -u xraym -f
```

成功啟動後，日誌中會出現類似以下內容：

```
INFO  XrayM started, node: 香港 01 (ID: 1)
INFO  Inbound VMess:443 started
INFO  Inbound Trojan:8443 started
```

---

## 節點監控

XrayM 每 60 秒自動向面板上報一次節點狀態。

![節點監控](../images/monitor.png)

**監控指標**：
- CPU 使用率
- 記憶體使用量
- 磁碟使用情況
- 即時網路上傳 / 下載速率
- 線上使用者數
- 累計流量統計

**查看方式**：管理後台 → 系統管理 → 健康檢查

點擊節點卡片可查看詳細圖表，資料每 10 秒自動刷新。

> XrayR 不支援節點監控功能，僅統計流量。

---

## 使用 XrayR（不推薦）

> XrayR 單程序僅支援單節點單連接埠單協議。多協議需要分別啟動多個程序。

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

多協議需為每種協議建立獨立的節點和設定檔，並啟動多個 XrayR 服務。

---

## 產生 VLESS REALITY 金鑰對

```bash
xraym x25519
```

輸出範例：

```
Private key: <base64-private-key>
Public key:  <base64-public-key>
```

將 Public key 填入面板的節點協議設定中，Private key 儲存在伺服器本地（不需要填入面板）。
