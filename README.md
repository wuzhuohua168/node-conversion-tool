# 多客户端订阅转换器

一个纯前端的节点订阅转换工具:粘贴节点链接或 Base64 订阅内容,**全部在浏览器本地完成解析与转换**(不上传任何数据),一键生成 6 种客户端格式。

支持协议:`ss` / `ssr` / `vmess` / `vless`(含 Reality)/ `trojan` / `hysteria` / `hysteria2` / `tuic` / `wireguard` / `socks5` / `http(s)`。

## 功能

- **输入**:一行一个节点链接、整段 Base64 订阅(从订阅 URL 复制的内容)、Clash YAML / sing-box JSON 完整配置(自动抽取 `proxies` / outbounds 中的节点)。
- **输出格式**(tab 切换):
  | 客户端 | 格式 | 说明 |
  |---|---|---|
  | Clash | YAML | 完整配置:节点 + dns 抗污染 + 37 组远程规则集 + 三段内联规则 |
  | Surge | conf | [Proxy]/[Proxy Group]/[Rule] |
  | Shadowrocket | txt | 仅节点(Base64) |
  | Quantumult X | conf | [server_local]/[policy]/[filter_local] |
  | sing-box | JSON | 完整配置;远程规则集用 MetaCubeX .srs 二进制 |
  | Base64 | txt | 通用节点订阅 |
- **输出内容模式**:`节点 + 规则`(完整配置)或 `仅节点`(裸节点列表)。
- **节点负载均衡开关**(右上角 ⚖,默认关):
  - Clash/Mihomo:`load-balance` + `round-robin`
  - sing-box:`load_balance` + `round-robin`
  - Surge:`url-latency-basis`
  - 适合聚合了大量节点做多线程下载时摊开连接;默认关闭以保护登录态/Apple 认证(详见 [RULES.md](RULES.md) 第 6 节)。
- **去重统计、诊断面板**:解析失败的行会标注行号/原因,节点卡片显示类型、SNI/流特征。

## 分流规则设计(详见 [RULES.md](RULES.md))

生成配置内置**三段内联规则**,优先级高于 `GEOIP,CN` / `MATCH` 兜底,不依赖远程规则集即可生效:

1. **FORCE_PROXY_DOMAINS** —— 强制走代理的域名(当前含 `workbuddy.ai`)。WorkBuddy 国际版部署在海外,国内直接访问往往打不开或卡死,必须走代理。与下方的国内版 `workbuddy.cn` 直连互不冲突。
2. **国内 AI 工具/模型 API 直连** —— `traework.cn`、`trae.cn`、`workbuddy.cn`(国内版)、`bestvirtualgoods.com`、`volces.com`、`deepseek.com`、`dashscope.aliyuncs.com`、`bigmodel.cn`、`moonshot.cn`、`siliconflow.cn` 等强制 DIRECT,解决 Trae / WorkBuddy 添加自定义模型「一直转圈 / Empty response / errCode -1」。
3. **Apple 认证域名直连** —— `apple.com`、`icloud.com`、`mzstatic.com`、`apple-dns.net` 强制 DIRECT,防止 AltServer/Sideloadly 侧载签名被代理干扰、iOS 证书「信任闪退」。

配套 **DNS 抗污染段**:Quad9 DoH/DoT 走 DIRECT 拿真 IP,`fake-ip` 模式 + `fake-ip-filter` 放行 Apple/AI 域名,`fallback-filter` 处理境外域名解析。

规则输出顺序:`FORCE_PROXY` → 37 组 RULE-SET → INLINE_DIRECT → `GEOIP,CN,DIRECT` → `MATCH,PROXY`。

## 使用

1. `git clone https://github.com/wuzhuohua168/node-conversion-tool.git`
2. 直接用浏览器打开 `index.html`(无需安装依赖、无需服务器)。也可以 `python3 -m http.server` 或任意静态托管。

## 本地运行 / 修改

- 单文件应用,所有逻辑在 `index.html` 的 `<script>` 内。
- 改完自检:浏览器打开页面 → 填示例 → 解析 → 切换格式/负载均衡 → 复制到目标客户端验证。
- **注意**:JS 字符串里如果出现英文撇号(如 `couldn't`、`it's`),在单引号字符串内必须转义(`\'`),否则整个 `<script>` 解析失败、页面白屏。

## 隐私

所有解析与转换均发生在浏览器本地,订阅内容不会发送到任何服务器。唯一的例外:生成的完整配置会引用远程规则集 URL(blackmatrix7 / MetaCubeX),由客户端在导入配置后自行拉取规则更新。

## License

MIT(如未指定,请以仓库实际 LICENSE 为准)。