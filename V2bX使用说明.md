# V2bX 使用说明

## 一、项目简介

V2bX 是基于 Xray-core 的 V2board 节点服务端，修改自 XrayR，永久开源免费。支持单实例对接多面板、多节点，内置限速、限设备、审计、证书自动申请等功能。

**支持的协议：** VMess、VLESS、Trojan、Shadowsocks、Hysteria1/2、TUIC、AnyTLS

**支持的内核：** Xray、Sing-box、Hysteria2

---

## 二、安装

### 一键脚本安装（推荐）

```bash
wget -N https://raw.githubusercontent.com/akiker233/V2bX-script/master/install.sh && bash install.sh
```

配置文件目录：`/etc/V2bX`

**更新：**

```bash
v2bx update
```

### 手动编译安装

需要 Go 1.24.0+：

```bash
go build -tags "with_xray with_sing with_hysteria2" -o v2bx .
```

---

## 三、配置文件说明

配置文件默认为 `/etc/V2bX/config.json`，JSON 格式，包含三个顶级字段：`Log`、`Cores`、`Nodes`。

### 3.1 完整结构示例

```json
{
  "Log": {
    "Level": "warn"
  },
  "Cores": [
    {
      "Type": "xray",
      "Log": { "Level": "error" }
    }
  ],
  "Nodes": [
    {
      "Name": "node1",
      "Core": "xray",
      "ApiHost": "https://your-panel.com",
      "ApiKey": "your-api-key",
      "NodeID": 1,
      "NodeType": "shadowsocks",
      "CertConfig": {
        "CertMode": "none"
      }
    }
  ]
}
```

### 3.2 Log 配置

| 字段 | 说明 | 可选值 |
|------|------|--------|
| `Level` | 日志级别 | `info` / `warn` / `error` / `none` |

### 3.3 Cores 配置

#### Xray 内核

```json
{
  "Type": "xray",
  "Log": { "Level": "error", "AccessPath": "", "ErrorPath": "" },
  "AssetPath": "",
  "DnsConfigPath": "",
  "RouteConfigPath": "",
  "InboundConfigPath": "",
  "OutboundConfigPath": "",
  "ConnectionConfig": {
    "handshake": 4,
    "connIdle": 300,
    "uplinkOnly": 2,
    "downlinkOnly": 5,
    "bufferSize": 4
  }
}
```

#### Sing-box 内核

```json
{
  "Type": "sing",
  "Name": "sing1",
  "Log": { "Level": "error", "Timestamp": true },
  "NTP": { "Enable": true, "Server": "time.apple.com", "ServerPort": 0 },
  "OriginalPath": "/etc/V2bX/sing_origin.json"
}
```

> `OriginalPath` 指定 Sing-box 自定义配置文件路径，可在其中自定义 DNS、路由、出入站等，详见第七节。

### 3.4 Nodes 节点配置

#### 通用字段

| 字段 | 说明 |
|------|------|
| `Name` | 节点标识名，不可重复 |
| `Core` | 使用的内核类型：`xray` / `sing` |
| `CoreName` | 同类多内核时指定名称 |
| `ApiHost` | 面板 API 地址 |
| `ApiKey` | 面板 Token |
| `NodeID` | 节点 ID |
| `NodeType` | 节点类型，如 `shadowsocks`、`hysteria2` |
| `ListenIP` | 监听 IP，默认 `0.0.0.0` |
| `SendIP` | 出站 IP（IPv4 填 `0.0.0.0`，IPv6 填 `::`） |
| `EnableTFO` | 是否开启 TCP Fast Open |
| `DeviceOnlineMinTraffic` | 设备计数流量最低阈值（kB），低于此值不计为在线 |
| `MinReportTraffic` | 最低上报流量阈值（kB） |

#### Xray 专属字段

| 字段 | 说明 |
|------|------|
| `EnableDNS` | 启用自定义 DNS |
| `DNSType` | DNS 策略：`AsIs` / `UseIP` / `UseIPv4` / `UseIPv6` |
| `EnableUot` | UDP over TCP |
| `DisableSniffing` | 禁用流量嗅探 |
| `EnableFallback` | 启用回落 |
| `FallBackConfigs` | 回落配置（SNI / Alpn / Path / Dest 等） |

#### Sing-box 专属：多路复用

```json
"MultiplexConfig": {
  "Enable": true,
  "Padding": true,
  "Brutal": {
    "Enable": true,
    "UpMbps": 500,
    "DownMbps": 500
  }
}
```

---

## 四、证书配置

在节点的 `CertConfig` 中配置：

```json
"CertConfig": {
  "CertMode": "dns",
  "CertDomain": "your.domain.com",
  "CertFile": "/etc/V2bX/cert/cert.pem",
  "KeyFile": "/etc/V2bX/cert/cert.key",
  "Email": "admin@example.com",
  "Provider": "cloudflare",
  "DNSEnv": {
    "CF_DNS_API_TOKEN": "your-token"
  }
}
```

| `CertMode` | 说明 |
|------------|------|
| `none` | 不启用 TLS |
| `http` | 通过 80 端口 HTTP 验证自动申请 |
| `dns` | 通过 DNS 验证自动申请，需配置 `Provider` 和 `DNSEnv` |
| `self` | 使用指定路径的证书文件，不存在则自签 |

`Provider` 的值对应 [lego 文档](https://go-acme.github.io/lego/dns/) 中各 DNS 服务商的 CLI 标志名，例如 `cloudflare`、`alidns`。

> Xray 内核独有字段 `RejectUnknownSni: true`：开启后拒绝 SNI 与证书域名不匹配的 TLS 握手。

---

## 五、限速与限制功能

### 5.1 用户限速

在 V2board 面板的订阅套餐中设置速度上限，单位 Mbps，设为 `0` 表示不限速。

### 5.2 本地节点限速

通过节点的 `LimitConfig` 配置，会覆盖面板下发的节点限速，对该节点所有用户生效：

```json
"LimitConfig": {
  "SpeedLimit": 100
}
```

### 5.3 动态限速

用户流量超过阈值后自动触发限速，到期后恢复：

```json
"LimitConfig": {
  "EnableDynamicSpeedLimit": true,
  "DynamicSpeedLimitConfig": {
    "Periodic": 60,
    "Traffic": 1000,
    "SpeedLimit": 10,
    "ExpireTime": 60
  }
}
```

| 字段 | 说明 |
|------|------|
| `Periodic` | 检查周期（秒） |
| `Traffic` | 触发阈值（MB） |
| `SpeedLimit` | 触发后限速值（Mbps） |
| `ExpireTime` | 限速持续时长（秒） |

### 5.4 设备连接限制

在 V2board 套餐中设置设备数上限（大于 0 时生效）。

- **宽松模式**：同一 IP 无论连接几个节点，只算一台设备
- 建议最低设为 `2`，设为 `1` 在切换节点时容易误封
- IP 数据跨后端同步约需 2 分钟，短暂超连接属正常现象

---

## 六、审计规则

在 V2board 面板中填写正则表达式来封锁特定域名，例如：

```
(^|\.)360\.cn$
```

- 支持正则匹配域名
- 支持直接填写 IP 地址封锁
- BT 协议封锁通过自定义路由实现（见第七节）

---

## 七、Xray 内核高级配置

通过在 Cores 中指定对应路径，可单独自定义 DNS、路由、入站、出站。节点 Tag 格式为 `[ApiHost]-NodeType:NodeID`，例如：`[https://panel.com/]-shadowsocks:12`。

### 7.1 自定义 DNS（`DnsConfigPath`）

创建 `dns.json`，参考 [Xray DNS 文档](https://xtls.github.io/config/dns.html)：

```json
{
  "servers": [
    {
      "address": "1.1.1.1",
      "domains": ["netflix.com"],
      "port": 53
    },
    "8.8.8.8"
  ]
}
```

节点配置中同时设置：

```json
"EnableDNS": true,
"DNSType": "UseIP"
```

- IPv6 优先：`SendIP` 设为 `"::"` + `DNSType` 设为 `"UseIP"`
- IPv4 优先：`SendIP` 设为 `"0.0.0.0"` + `DNSType` 设为 `"UseIP"`

### 7.2 自定义路由（`RouteConfigPath`）

创建 `route.json`，封锁 BT、私网 IP、Netflix 解锁示例：

```json
{
  "rules": [
    { "type": "field", "ip": ["geoip:private"], "outboundTag": "block" },
    { "type": "field", "protocol": ["bittorrent"], "outboundTag": "block" },
    { "type": "field", "domain": ["geosite:netflix"], "outboundTag": "netflix-out" }
  ]
}
```

> geo 规则需将 `geoip.dat` 和 `geosite.dat` 放在 `config.json` 同目录下。

### 7.3 自定义入站（`InboundConfigPath`）

创建 `custom_inbound.json`，格式参考 Xray inbound 配置：

```json
[
  {
    "tag": "socks-in",
    "port": 1234,
    "protocol": "socks"
  }
]
```

### 7.4 自定义出站（`OutboundConfigPath`）

创建 `custom_outbound.json`，示例（IPv4/IPv6 分流 + 黑洞）：

```json
[
  { "tag": "IPv4-out", "protocol": "freedom", "settings": { "domainStrategy": "UseIPv4" } },
  { "tag": "IPv6-out", "protocol": "freedom", "settings": { "domainStrategy": "UseIPv6" } },
  { "tag": "block",    "protocol": "blackhole" }
]
```

---

## 八、Sing-box 内核自定义配置

在 Core 的 `OriginalPath` 中指定自定义配置文件（如 `/etc/V2bX/sing_origin.json`），V2bX 会将其与面板下发的节点信息合并。

示例（含 WARP 出站 + 广告拦截）：

```json
{
  "dns": {
    "servers": [{ "address": "1.1.1.1", "strategy": "prefer_ipv4" }]
  },
  "outbounds": [
    { "type": "direct", "tag": "direct", "domain_strategy": "prefer_ipv4" },
    {
      "type": "wireguard",
      "tag": "warp",
      "server": "engage.cloudflareclient.com",
      "server_port": 2408
    }
  ],
  "route": {
    "rules": [
      { "ip_is_private": true, "action": "reject" },
      { "rule_set": "geosite-ads", "action": "reject" },
      { "rule_set": ["geoip-cn", "geosite-cn"], "outbound": "warp" }
    ]
  }
}
```

节点 Tag 同样遵循 `[ApiHost]-NodeType:NodeID` 格式。

---

## 九、引用外部配置

节点配置支持通过 `Include` 引用本地或远程 JSON 文件，便于多节点复用：

```json
{ "Include": "../example/node1.json" }
{ "Include": "http://127.0.0.1:11451/node1.json" }
```

---

## 十、常用命令

```bash
v2bx update             # 更新到最新版本
systemctl start v2bx    # 启动
systemctl stop v2bx     # 停止
systemctl restart v2bx  # 重启
systemctl status v2bx   # 查看状态
journalctl -u v2bx -f   # 实时查看日志
```

> 配置文件修改后 V2bX 会自动检测并热重载，通常无需手动重启。
