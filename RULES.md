# 内联分流规则说明（本仓库定制）

> 除 37 组远程规则集（blackmatrix7/ios_rule_script）外，本工具生成的
> Clash / Surge / Quantumult X / sing-box 配置都注入了三段**内联规则**，
> 全部位于 `GEOIP,CN` / `MATCH` 兜底之前，不依赖远程规则集下载，开机即用。

## 1. FORCE_PROXY_DOMAINS —— 强制走代理

| 域名 | 原因 |
|---|---|
| `workbuddy.ai` | **2026-09-15 新增**。WorkBuddy 国际版部署在海外，国内直连往往打不开或卡死（TLS 握手超时），必须走代理(PROXY)才能稳定访问。加入此清单后，生成的 Clash/Surge/QX/sing-box 配置会在 `GEOIP,CN` / `MATCH` 兜底之前就把 `workbuddy.ai` 及其子域路由到代理。**注意区分**：`workbuddy.cn`(国内版)仍在下方「国内 AI 工具直连」清单走 DIRECT，两者互不冲突——一个国内直连、一个海外代理。 |
| (无) | 原 `bestvirtualgoods.com` 实测:大陆可直连 Cloudflare 边缘(TCP 通,526 属边缘 TLS/源站校验问题),走 Clash 代理(7890/7897)反而连接直接失败(000),当前节点链到该 Cloudflare 站不通。故该域已移入下方直连白名单,避免自定义模型「一直转圈 / Empty response」。此清单保留,用于将来确有「必须走代理才通」的域名。 |

## 2. FORCE_PROXY_DOMAINS — DNS 说明

`workbuddy.ai`（国际版）**不需要**加入 `fake-ip-filter`。

原因：该域名走代理（PROXY），DNS 也由代理出口解析，fake-ip 模式下由 Clash 自动返回 fake-ip（如 `198.18.x.x`），客户端拿着 fake-ip 通过代理通道访问海外节点，Clash 再在代理侧用真实 DNS 解析 —— 这条链路是完整可用的。而国内直连域名（如 `workbuddy.cn`、`traework.cn`）则必须用真实 IP 才能直接 TCP 到服务器，所以它们才需要加入 `fake-ip-filter`。

如果你同时使用国内版和国际版，只需在客户端输入域名即可，规则会自动按 `DOMAIN-SUFFIX` 决定走 DIRECT 还是 PROXY。

## 3. DOMESTIC_AI_DIRECT_DOMAINS —— 国内 AI 工具强制直连

| 域名 | 归属 |
|---|---|
| `traework.cn` / `trae.cn` | 字节 TRAE(含 SOLO/Work 桌面端,请求经其国内后端转发) |
| `workbuddy.cn` | WorkBuddy 云服务 |
| `bestvirtualgoods.com` | WorkBuddy 模型上游中转 API(Cloudflare 后)。**2026-09-13 实测直连可达、代理 000,由 FORCE_PROXY 改为 DIRECT** |
| `volces.com` / `volcengine.com` | 火山引擎方舟(豆包模型) |
| `deepseek.com` | DeepSeek 官方 API |
| `dashscope.aliyuncs.com` | 阿里云通义 |
| `bigmodel.cn` | 智谱 GLM |
| `moonshot.cn` | 月之暗面 Kimi |
| `siliconflow.cn` | 硅基流动 |

**解决的典型问题**：Trae / WorkBuddy 添加自定义模型时「一直转圈 /
Empty response / errCode -1」——多为到国内服务器的请求被误代理，
绕境外节点后延迟飙升甚至握手失败。

## 4. APPLE_DIRECT_DOMAINS —— Apple 认证域名直连

| 域名 | 用途 |
|---|---|
| `apple.com` | 含 `gsa.apple.com`（AltStore/Sideloadly anisette 认证）、`ocsp.apple.com`（证书状态） |
| `icloud.com` | iCloud 服务 |
| `mzstatic.com` | App Store 静态资源 |
| `apple-dns.net` | Apple 私有 DNS |

**解决的典型问题**：
- AltStore 刷新/安装报 `The data couldn't be read because it isn't in the correct format`（代理篡改 anisette 响应）。
- iOS 免费签名证书「信任闪回」——运营商劫持 `ocsp.apple.com` 解析，配合下方 DNS 段用真实 IP 解析修复。

## 5. DNS 抗污染段（配合上述白名单生效）

本网络实测：Cloudflare / Google 的 DoH 被阻断，Quad9（9.9.9.9）是唯一干净解析源。
生成的 `dns:` 段要点：

- `respect-rules: false`：DNS 不跟分流走，防「代理未就绪先走代理解析」死锁。
- `proxy-server-nameserver` / `direct-nameserver`：节点域名与直连域名强制 Quad9 DoH `#DIRECT`，拿真 IP。
- `fallback`：Quad9 DoH/DoT `#PROXY`，国外域名经代理出口解析，配合 `fallback-filter`（geoip CN + google/github/openai/youtube 等域名白名单）。
- `enhanced-mode: fake-ip` + `fake-ip-filter` 放行 `+.apple.com`、`ocsp.apple.com`、
  `+.traework.cn`、`+.trae.cn`、`+.workbuddy.cn` —— 这些域名用真实 IP，直接修复 OCSP 劫持导致的证书信任闪退。
- 注意：`fallback` 走代理，若 PROXY 组没选/断开，国外域名会解析失败（国内不受影响）。

## 6. 节点负载均衡开关(2026-09-13 新增)

UI 右上角「⚖ 负载均衡」按钮,默认**关闭**。开启后,「自动选择」策略组
从 `url-test`(取最低延迟节点)切换为负载均衡轮询:

| 客户端 | 关闭(默认) | 开启 |
|---|---|---|
| Clash / Mihomo | `url-test` + gstatic 204 + 300s | `load-balance` + `strategy: round-robin` |
| sing-box | `urltest` | `load_balance` + `strategy: round-robin` |
| Surge | `PROXY = select`(手动) | `PROXY = url-latency-basis`(延迟轮询) |
| Quantumult X | `static=PROXY` | 不支持,保持不变 |

**适用场景**:聚合了大量节点(多家机场/自建)做多线程下载、跑测速时,
轮询能摊开并发连接、提升总带宽与容错。

**注意事项**(为什么默认关闭):
- 轮询会频繁更换出口 IP,可能破坏需要保持会话/相同出口的登录态、
  Cloudflare 人机验证、流媒体区域锁。
- 你的内联规则(RULES.md 1-4)设计目标是「AI/Apple 直连、其余走 PROXY」,
  load-balance 只影响 PROXY 组内部选点,不影响直连白名单。
- 若单个节点延迟远高于其他节点,轮询会拖慢整体体验;此时建议仍用 url-test。

## 规则输出顺序(自上而下,先匹配先生效)

```
1. FORCE_PROXY_DOMAINS      → PROXY   （workbuddy.ai 国际版，海外部署，必须代理）
2. RULE-SET × 37            → 远程规则集
3. INLINE_DIRECT_DOMAINS    → DIRECT  （Apple + 国内 AI 域名，含 workbuddy.cn 国内版）
4. GEOIP,CN,DIRECT
5. MATCH,PROXY
```

## 维护提示

- 新增需要直连的国内服务：改 `DOMESTIC_AI_DIRECT_DOMAINS`（写父域名，`DOMAIN-SUFFIX` 自动覆盖子域）。
- 新增必须走代理的站点：改 `FORCE_PROXY_DOMAINS`。
- 改完跑一次生成，把 Clash 输出贴到 Clash Verge 预览确认 `dns:` 与 `rules:` 无红字即可。
- sing-box 格式使用 `domain_suffix` 等价实现；Surge/QX 语法分别对应 `DOMAIN-SUFFIX` / `host-suffix`。
