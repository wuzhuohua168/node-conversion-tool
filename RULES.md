# 内联分流规则说明（本仓库定制）

> 除 37 组远程规则集（blackmatrix7/ios_rule_script）外，本工具生成的
> Clash / Surge / Quantumult X / sing-box 配置都注入了三段**内联规则**，
> 全部位于 `GEOIP,CN` / `MATCH` 兜底之前，不依赖远程规则集下载，开机即用。

## 1. FORCE_PROXY_DOMAINS —— 强制走代理

| 域名 | 原因 |
|---|---|
| `bestvirtualgoods.com` | WorkBuddy 的模型经本机 `lb_proxy`（127.0.0.1:8787）转发到这里的中转 API，该站点架在 Cloudflare 后。若被 GEOIP 误判为国内直连，模型调用整体失败；强制 PROXY 保证任何规则顺序下都能通。 |

## 2. DOMESTIC_AI_DIRECT_DOMAINS —— 国内 AI 工具强制直连

| 域名 | 归属 |
|---|---|
| `traework.cn` / `trae.cn` | 字节 TRAE（含 SOLO/Work 桌面端，请求经其国内后端转发） |
| `workbuddy.cn` | WorkBuddy 云服务 |
| `volces.com` / `volcengine.com` | 火山引擎方舟（豆包模型） |
| `deepseek.com` | DeepSeek 官方 API |
| `dashscope.aliyuncs.com` | 阿里云通义 |
| `bigmodel.cn` | 智谱 GLM |
| `moonshot.cn` | 月之暗面 Kimi |
| `siliconflow.cn` | 硅基流动 |

**解决的典型问题**：Trae / WorkBuddy 添加自定义模型时「一直转圈 /
Empty response / errCode -1」——多为到国内服务器的请求被误代理，
绕境外节点后延迟飙升甚至握手失败。

## 3. APPLE_DIRECT_DOMAINS —— Apple 认证域名直连

| 域名 | 用途 |
|---|---|
| `apple.com` | 含 `gsa.apple.com`（AltStore/Sideloadly anisette 认证）、`ocsp.apple.com`（证书状态） |
| `icloud.com` | iCloud 服务 |
| `mzstatic.com` | App Store 静态资源 |
| `apple-dns.net` | Apple 私有 DNS |

**解决的典型问题**：
- AltStore 刷新/安装报 `The data couldn't be read because it isn't in the correct format`（代理篡改 anisette 响应）。
- iOS 免费签名证书「信任闪回」——运营商劫持 `ocsp.apple.com` 解析，配合下方 DNS 段用真实 IP 解析修复。

## 4. DNS 抗污染段（配合上述白名单生效）

本网络实测：Cloudflare / Google 的 DoH 被阻断，Quad9（9.9.9.9）是唯一干净解析源。
生成的 `dns:` 段要点：

- `respect-rules: false`：DNS 不跟分流走，防「代理未就绪先走代理解析」死锁。
- `proxy-server-nameserver` / `direct-nameserver`：节点域名与直连域名强制 Quad9 DoH `#DIRECT`，拿真 IP。
- `fallback`：Quad9 DoH/DoT `#PROXY`，国外域名经代理出口解析，配合 `fallback-filter`（geoip CN + google/github/openai/youtube 等域名白名单）。
- `enhanced-mode: fake-ip` + `fake-ip-filter` 放行 `+.apple.com`、`ocsp.apple.com`、
  `+.traework.cn`、`+.trae.cn`、`+.workbuddy.cn` —— 这些域名用真实 IP，直接修复 OCSP 劫持导致的证书信任闪退。
- 注意：`fallback` 走代理，若 PROXY 组没选/断开，国外域名会解析失败（国内不受影响）。

## 规则输出顺序（自上而下，先匹配先生效）

```
1. FORCE_PROXY_DOMAINS      → PROXY   （模型中转，必须代理）
2. RULE-SET × 37            → 远程规则集
3. INLINE_DIRECT_DOMAINS    → DIRECT  （Apple + 国内 AI 域名）
4. GEOIP,CN,DIRECT
5. MATCH,PROXY
```

## 维护提示

- 新增需要直连的国内服务：改 `DOMESTIC_AI_DIRECT_DOMAINS`（写父域名，`DOMAIN-SUFFIX` 自动覆盖子域）。
- 新增必须走代理的站点：改 `FORCE_PROXY_DOMAINS`。
- 改完跑一次生成，把 Clash 输出贴到 Clash Verge 预览确认 `dns:` 与 `rules:` 无红字即可。
- sing-box 格式使用 `domain_suffix` 等价实现；Surge/QX 语法分别对应 `DOMAIN-SUFFIX` / `host-suffix`。
