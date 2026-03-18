# Clash Verge Rev (mihomo) + Tailscale 共存配置指南

> 环境：Linux / Clash Verge Rev + mihomo 内核 / TUN 模式 + 系统代理 + 全局代理
>
> 目标：Tailscale 设备间（100.x.x.x）的 SSH、文件传输、远程桌面等流量直连不绕路，其余所有流量照常走 Clash 代理。

---

## Part 1 — 原理概述

### 问题根源

Clash Verge 开启 TUN + fake-ip DNS 后，会在三个层面拦截 Tailscale 流量：

| 层面 | 具体表现 | 后果 |
|------|---------|------|
| **DNS** | fake-ip 模式把 `controlplane.tailscale.com` 解析为 `198.18.0.x` 假 IP | tailscaled 无法注册到控制服务器，`tailscale up` 卡死 |
| **TUN 路由** | Clash 的 TUN 虚拟网卡 (`Meta`) 通过 `auto-route` 捕获所有流量，包括 `100.64.0.0/10` | Tailscale 设备间的 P2P 数据被 TUN 截获，经 Clash 处理后产生不必要的延迟或路由失败 |
| **代理规则** | 全局模式下所有流量走代理；rule 模式下如果没有 Tailscale 的 DIRECT 规则，也会走代理 | Tailscale 流量被代理转发，丧失点对点直连优势 |

### 解决思路（共两处修改，互不耦合）

**修改 A — Clash 侧（Script 增强脚本）**

通过 Clash Verge 的 **Script 增强配置** 在每次生成运行配置时自动注入三项设置：

1. **DNS `fake-ip-filter`**：将 `*.tailscale.com` / `*.tailscale.io` 加入白名单，使这些域名返回真实 IP 而非假 IP
2. **TUN `route-exclude-address`**：将 `100.64.0.0/10`（Tailscale IPv4）和 `fd7a:115c:a1e0::/48`（Tailscale IPv6）排除出 TUN 路由，让这些 IP 走系统原生路由表（即 tailscale0 接口）
3. **规则 prepend**：在 rules 最前面插入 `DOMAIN-SUFFIX,tailscale.com,DIRECT` 等规则和 `PROCESS-NAME,tailscaled,DIRECT`（rule 模式下生效）

使用 Script 而非直接改配置文件的原因：订阅更新会覆盖 `clash-verge.yaml`，但 Script 增强在每次配置生成时重新执行，**永久有效**。

**修改 B — tailscaled 侧（systemd 代理环境变量）**

tailscaled 守护进程需要连接 `controlplane.tailscale.com` 进行注册和密钥交换（控制面）。由于 Clash 的 fake-ip DNS 会劫持该域名，且 tailscaled 不走系统代理，需要通过 systemd override 给它设置 `HTTP_PROXY` / `HTTPS_PROXY` 指向 Clash 的 HTTP 代理端口。

`NO_PROXY` 中排除 `100.64.0.0/10` 等 Tailscale 网段，确保 **数据面（WireGuard P2P 隧道）完全不走代理**。

### 最终效果

```
┌─────────────────────────────────────────────┐
│            系统所有流量                       │
│                                             │
│  ┌─ Tailscale IP (100.x.x.x) ──→ tailscale0 接口 ──→ P2P 直连（零额外开销）
│  │   - route-exclude-address 绕过 TUN
│  │   - DIRECT 规则绕过代理
│  │
│  ├─ tailscaled 控制面 ──→ HTTP_PROXY ──→ Clash 代理 ──→ controlplane.tailscale.com
│  │   - systemd 环境变量
│  │
│  └─ 其他所有流量 ──→ Clash TUN ──→ 全局代理（不受任何影响）
│
└─────────────────────────────────────────────┘
```

---

## Part 2 — 详细操作步骤

> **重要：全程不需要重启 Clash 服务或 systemctl restart clash-verge-service。所有 Clash 配置通过 GUI 应用生效。**

### 前置信息

Clash Verge Rev 配置目录（`~` 代表当前用户的 home 目录）：

```
~/.local/share/io.github.clash-verge-rev.clash-verge-rev/
├── clash-verge.yaml          # 运行时配置（自动生成，不要手动改）
├── dns_config.yaml           # DNS 配置模板
├── verge.yaml                # Clash Verge 应用设置
├── profiles.yaml             # 订阅和增强配置的关联关系
└── profiles/
    ├── <订阅UID>.yaml        # 远程订阅配置
    ├── <ScriptUID>.js        # ← Script 增强脚本（我们要改的）
    ├── <MergeUID>.yaml       # Merge 增强
    └── <RulesUID>.yaml       # Rules 增强
```

**如何找到你的 Script 文件名**：打开 `profiles.yaml`，找到你正在使用的订阅条目（`current` 字段指向的 UID），查看其 `option.script` 字段值，对应 `profiles/` 下的 `<该值>.js` 就是要修改的文件。

以本机 (junjie-NUC14RVS) 为例：订阅 UID 为 `R9uZsinpwqAb`，关联的 Script 为 `sL1TlO2A3Y5Y`，所以修改 `profiles/sL1TlO2A3Y5Y.js`。**不同机器上这些 UID 不同，必须自行确认。**

### 步骤 1：备份

```bash
BACKUP_DIR=~/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge-rev-backup/before-tailscale-$(date +%Y%m%d)
mkdir -p "$BACKUP_DIR"
cp ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml "$BACKUP_DIR/"
cp ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/dns_config.yaml "$BACKUP_DIR/"
# 找到你的 Script 增强文件名（查看 profiles.yaml 中 script 字段对应的 uid）
cp ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/profiles/<你的Script文件>.js "$BACKUP_DIR/"
```

### 步骤 2：修改 Script 增强脚本

找到你订阅关联的 Script 增强文件（本机为 `profiles/sL1TlO2A3Y5Y.js`），将内容替换为：

```javascript
// Define main function (script entry)

function main(config, profileName) {
  // --- Tailscale compatibility ---

  // 1. DNS: add Tailscale domains to fake-ip-filter so they resolve to real IPs
  if (!config.dns) config.dns = {};
  if (!config.dns["fake-ip-filter"]) config.dns["fake-ip-filter"] = [];

  var tailscaleDnsFilters = [
    "+.tailscale.com",
    "+.tailscale.io",
    "*.tailscale.com",
    "*.tailscale.io",
  ];
  for (var i = 0; i < tailscaleDnsFilters.length; i++) {
    if (config.dns["fake-ip-filter"].indexOf(tailscaleDnsFilters[i]) === -1) {
      config.dns["fake-ip-filter"].push(tailscaleDnsFilters[i]);
    }
  }

  // 2. TUN: exclude Tailscale IP ranges from TUN routing
  if (!config.tun) config.tun = {};
  if (!config.tun["route-exclude-address"]) {
    config.tun["route-exclude-address"] = [];
  }

  var tailscaleRouteExcludes = ["100.64.0.0/10", "fd7a:115c:a1e0::/48"];
  for (var j = 0; j < tailscaleRouteExcludes.length; j++) {
    if (
      config.tun["route-exclude-address"].indexOf(
        tailscaleRouteExcludes[j]
      ) === -1
    ) {
      config.tun["route-exclude-address"].push(tailscaleRouteExcludes[j]);
    }
  }

  // 3. Rules: prepend Tailscale DIRECT rules (effective when mode=rule)
  if (!config.rules) config.rules = [];

  var tailscaleRules = [
    "DOMAIN-SUFFIX,tailscale.com,DIRECT",
    "DOMAIN-SUFFIX,tailscale.io,DIRECT",
    "PROCESS-NAME,tailscaled,DIRECT",
    "IP-CIDR,100.64.0.0/10,DIRECT,no-resolve",
    "IP-CIDR6,fd7a:115c:a1e0::/48,DIRECT,no-resolve",
  ];
  for (var k = tailscaleRules.length - 1; k >= 0; k--) {
    if (config.rules.indexOf(tailscaleRules[k]) === -1) {
      config.rules.unshift(tailscaleRules[k]);
    }
  }

  return config;
}
```

本机 (junjie-NUC14RVS) 参考文件：`~/.local/share/io.github.clash-verge-rev.clash-verge-rev/profiles/sL1TlO2A3Y5Y.js`

### 步骤 3：在 Clash Verge GUI 中应用配置

打开 Clash Verge Rev → 左侧「配置」→ 找到你的订阅 → **右键选择"使用"或点击激活**。

这一步会触发 Script 增强重新执行，生成新的 `clash-verge.yaml`。

**验证**：在终端运行以下命令确认注入成功：

```bash
grep -n 'route-exclude\|fake-ip-filter.*tailscale\|tailscale\|PROCESS-NAME,tailscaled' \
  ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml
```

应看到 `route-exclude-address`、`+.tailscale.com`、`DOMAIN-SUFFIX,tailscale.com,DIRECT`、`PROCESS-NAME,tailscaled,DIRECT` 等条目。

### 步骤 4：配置 tailscaled 代理

```bash
sudo mkdir -p /etc/systemd/system/tailscaled.service.d
sudo tee /etc/systemd/system/tailscaled.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1,::1,100.64.0.0/10,fd7a:115c:a1e0::/48,192.168.0.0/16,10.0.0.0/8"
EOF
sudo systemctl daemon-reload
sudo systemctl restart tailscaled
```

参考文件：`/etc/systemd/system/tailscaled.service.d/proxy.conf`（所有机器路径相同）

> 端口 `7897` 对应 Clash Verge 的 `mixed-port`，如果你改过端口需要同步修改。

### 步骤 5：启动 Tailscale

```bash
sudo tailscale up
```

### 步骤 6：验证

```bash
# 1. 确认 Tailscale 在线
tailscale status

# 2. 确认 DNS 不再返回假 IP（应看到 192.x.x.x 真实 IP，而非 198.18.x.x）
nslookup controlplane.tailscale.com

# 3. 确认设备间连通
tailscale ping <对端Tailscale IP>

# 4. 确认其他流量仍走代理
curl -s https://www.google.com -o /dev/null -w "HTTP %{http_code}\n"
```

---

## 附录：注意事项

1. **绝对不要**通过 SSH 远程执行 `systemctl restart clash-verge-service`。这会破坏 Clash Verge GUI 与服务之间的 Unix socket 通信管道，导致 GUI 报 "IO error: 没有那个文件或目录"。所有 Clash 配置变更都应通过 GUI 操作生效。

2. 订阅更新不会覆盖 Script 增强脚本。Script 是独立文件，每次配置生成时自动执行。

3. 如需回滚，恢复备份的 Script 文件并在 GUI 中重新应用配置即可，无需重启任何系统服务。

4. `HTTP_PROXY` 只影响 tailscaled 的控制面（与 controlplane.tailscale.com 的 HTTPS 通信），不影响 WireGuard 数据面。P2P 传输始终走 UDP 直连。

5. 每台运行 Clash Verge + Tailscale 的机器都需要独立执行以上步骤。Script 文件的 UID 不同机器各不相同，需查看各自的 `profiles.yaml` 确认。
