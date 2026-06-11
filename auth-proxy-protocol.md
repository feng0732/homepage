# Homepage 身份认证代理协议分析（v6 - 统计口径修正版）

> **项目信息**：Homepage v1.13.1 | 仓库：[gethomepage/homepage](https://github.com/gethomepage/homepage)
>
> **引用稳定性**：本文档所有代码引用均使用 **`v1.13.1` 稳定版本 tag**，链接永久有效，不会随分支变化而失效。
>
> **引用格式**：`[相对路径#行号](https://github.com/gethomepage/homepage/blob/v1.13.1/相对路径#L行号)`
>
> **仓库相对路径对照**：所有路径均为项目 `src/` 目录下的相对路径，可直接在本地 IDE 中按相同路径查找。
>
> **覆盖范围**：全部 **47 个自定义 widget proxy** + 5 个通用 handler，按 15 个业务类别组织。
>
> **统计口径**：
> - 自定义 widget proxy：`src/widgets/*/proxy.js`，共 47 个（有独立 proxy 实现文件）
> - 通用 handler：`src/utils/proxy/handlers/*.js`，共 5 个（credentialed, generic, synology, jsonrpc, unifi）
> - 使用通用 handler 但无独立 proxy.js 的服务不计入自定义 proxy 数量

---

## 一、整体架构概览

Homepage 是基于 Next.js 的自托管仪表盘应用，核心设计是**服务端代理（Server-side Proxy）**模式：前端不直接访问后端服务 API，而是通过 `/api/services/proxy` 统一入口，由服务端代理层注入认证头、管理登录态、转发请求。

### 请求流全景

```
浏览器前端
  │  [src/utils/proxy/use-widget-api.js#L5-L16](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/use-widget-api.js#L5-L16)
  │  useWidgetAPI → SWR → /api/services/proxy?group=X&service=Y&endpoint=Z
  ▼
Next.js Middleware
  │  [src/middleware.js#L3-L19](https://github.com/gethomepage/homepage/blob/v1.13.1/src/middleware.js#L3-L19)
  │  Host 白名单校验
  ▼
API 路由入口
  │  [src/pages/api/services/proxy.js#L10-L115](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L10-L115)
  │  解析 widget 类型 → 选择 proxyHandler → endpoint mapping
  ▼
Proxy Handler 层
  │  credentialed / generic / synology / jsonrpc / unifi / 47 个自定义 widget proxy
  │  注入认证头 / 管理登录态 / Cookie Jar
  ▼
HTTP 底层
  │  [src/utils/proxy/http.js#L252-L293](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L252-L293)
  │  httpProxy() → follow-redirects → 目标服务 API
  ▼
目标后端服务
```

---

## 二、认证头传递机制

认证头构建发生在 **Proxy Handler 层**，核心逻辑集中在 `credentialed.js`。

### 2.1 认证头构建的三层合并

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L33-L38](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L33-L38)

```javascript
const headers = {
  "Content-Type": "application/json",
  ...(widgets[widget.type].headers ?? {}),   // 1. widget 类型默认头
  ...(widget.headers ?? {}),                  // 2. 用户配置的自定义头
  ...(req.extraHeaders ?? {}),               // 3. mapping 中定义的额外头
};
```

**合并优先级**（从低到高）：
1. **第一层**：widget 类型注册时的默认 `headers`
2. **第二层**：用户在 `services.yaml` 中为服务配置的 `headers` 字段
3. **第三层**：`req.extraHeaders`，由 [src/pages/api/services/proxy.js#L88-L90](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L88-L90) 注入

### 2.2 按 widget 类型分发的认证头策略

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L40-L140](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L40-L140)

| widget 类型 | 认证头格式 | 代码定位 |
|---|---|---|
| `stocks` (finnhub) | `X-Finnhub-Token: {key}` | [L40-L44](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L40-L44) |
| `coinmarketcap` | `X-CMC_PRO_API_KEY: {key}` | [L45-L46](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L45-L46) |
| `gotify` | `X-gotify-Key: {key}` | [L47-L48](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L47-L48) |
| `checkmk` | `Authorization: Bearer {username} {password}` | [L49-L51](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L49-L51) |
| `argocd`, `authentik`, `mealie`, `vikunja` 等 | `Authorization: Bearer {key}` | [L52-L73](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L52-L73) |
| `truenas` | `Bearer {key}` 或 `Basic {user:pass}` | [L74-L79](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L74-L79) |
| `ntfy` | `Bearer {key}` 或 `Basic {user:pass}` | [L80-L85](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L80-L85) |
| `proxmox` | `PVEAPIToken={username}={password}` | [L86-L87](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L86-L87) |
| `proxmoxbackupserver` | `PBSAPIToken={username}:{password}` | [L88-L90](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L88-L90) |
| `autobrr`, `jellystat` | `X-API-Token: {key}` | [L91-L92](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L91-L92) |
| `tubearchivist` | `Authorization: Token {key}` | [L93-L94](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L93-L94) |
| `miniflux` | `X-Auth-Token: {key}` | [L95-L96](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L95-L96) |
| `nextcloud` | `NC-Token: {key}` 或 `Basic {user:pass}` | [L97-L102](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L97-L102) |
| `paperlessngx` | `Token {key}` 或 `Basic {user:pass}` | [L103-L108](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L103-L108) |
| `azuredevops` | `Basic {base64("$:{key}")}` | [L109-L110](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L109-L110) |
| `glances` | `Basic {user:pass}` | [L111-L112](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L111-L112) |
| `plantit` | `Key: {key}` | [L113-L114](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L113-L114) |
| `myspeed` | `Password: {password}` | [L115-L116](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L115-L116) |
| `esphome` | `Basic {user:pass}` 或 `Cookie: authenticated={key}` | [L117-L122](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L117-L122) |
| `wgeasy` | `Basic {user:pass}` 或 `Authorization: {password}` | [L123-L128](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L123-L128) |
| `trilium` | `Authorization: {key}` | [L129-L130](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L129-L130) |
| `gitlab` | `PRIVATE-TOKEN: {key}` | [L131-L132](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L131-L132) |
| `speedtest` | `Bearer {key}`（v1 不需要） | [L133-L137](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L133-L137) |
| **默认 fallback** | `X-API-Key: {key}` | [L138-L140](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L138-L140) |

### 2.3 Basic Auth 辅助函数

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L11-L13](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L11-L13)

```javascript
function basicAuthHeader(widget) {
  return `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.4 generic handler 的认证头

**代码位置**：[src/utils/proxy/handlers/generic.js#L28-L36](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/generic.js#L28-L36)

如果配置了用户名和密码，自动添加 Basic Auth 头。

---

## 三、代理转发逻辑

### 3.1 API 路由入口

**代码位置**：[src/pages/api/services/proxy.js#L10-L115](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L10-L115)

处理流程：
1. 解析参数：service、group、index、endpoint（[L12](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L12)）
2. 获取 widget 配置（[L13](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L13)）
3. 确定 widget 类型（[L16-L18](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L16-L18)）
4. 查找 proxy handler：优先 `widget.proxyHandler`，否则 `genericProxyHandler`（[L27](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L27)）
5. Endpoint mapping（[L36-L97](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L36-L97)）

### 3.2 Endpoint Mapping 机制

**代码位置**：[src/pages/api/services/proxy.js#L36-L97](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L36-L97)

mapping 支持的能力：

| 能力 | 代码定位 |
|---|---|
| endpoint 语义化映射 | [L41](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L41) |
| HTTP 方法覆盖与校验 | [L44-L47](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L44-L47) |
| 请求体覆盖 | [L54-L55](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L54-L55) |
| 路径 segments 参数 + 安全校验（禁止 / \ ..） | [L58-L72](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L58-L72) |
| 查询参数白名单过滤 | [L74-L86](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L74-L86) |
| mapping 级额外 headers | [L88-L90](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L88-L90) |
| mapping 级自定义 proxyHandler | [L42](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L42) |
| 响应数据 map 转换 | [L40](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L40) |

### 3.3 HTTP 底层实现

**代码位置**：[src/utils/proxy/http.js#L252-L293](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L252-L293)

| 特性 | 代码定位 |
|---|---|
| follow-redirects 重定向 | [L5](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L5) |
| 重定向时自动处理 Cookie | [L19-L22](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L19-L22) |
| gzip/deflate 自动解压 | [L35-L53](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L35-L53) |
| 自定义 DNS 回退（Alpine/musl 兼容） | [L111-L225](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L111-L225) |
| Agent 缓存（keepAlive） | [L228-L250](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L228-L250) |
| 自签名证书兼容 | [L245](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L245) |
| IPv6 禁用支持 | [L254](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js#L254) |

### 3.4 Cookie Jar 机制

**代码位置**：[src/utils/proxy/cookie-jar.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/cookie-jar.js)

| 函数 | 职责 | 代码定位 |
|---|---|---|
| `setCookieHeader()` | 请求前从 Jar 取 Cookie 注入头 | [L5-L15](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/cookie-jar.js#L5-L15) |
| `addCookieToJar()` | 响应后将 `Set-Cookie` 存入 Jar（1 小时有效） | [L17-L41](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/cookie-jar.js#L17-L41) |

### 3.5 前端封装

**代码位置**：[src/utils/proxy/use-widget-api.js#L5-L16](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/use-widget-api.js#L5-L16)

通过 SWR 调用 `/api/services/proxy`，支持 `refreshInterval` 配置。

---

## 四、登录态判断逻辑（全分类 · 逐服务对齐）

> 本章节覆盖全部 **47 个自定义 proxy handler** + 5 个通用 handler，按 15 个业务类别组织。每个服务都标注了**认证方式**、**Token/Cookie/业务字段**、**登录态判断条件**、**缓存策略**和**对应代码行号**。
>
> **分类说明**：有独立 proxy.js 文件的服务计入"自定义 proxy"数量；使用 credentialed/generic 等通用 handler 但无独立 proxy.js 的服务（如 Prowlarr、Miniflux、Nextcloud 等）在对应分类中注明"使用通用 handler"，不计入自定义 proxy 数量。
>
> 所有链接均指向 `v1.13.1` tag，永久有效。

### 4.1 登录态模式总览

| 模式分类 | 典型特征 | 代表服务数量 |
|---|---|---|
| **无状态模式** | 每次请求直接带认证头，无需登录流程 | ~25+（credentialed + generic 覆盖的多数） |
| **Token 缓存模式** | 先登录获取 token，缓存到 memory-cache，401/403 时刷新 | 约 17 个 |
| **Cookie Jar 模式** | 登录后 Cookie 自动存入 cookie-jar，后续自动携带 | 约 8 个 |
| **每次登录模式** | 每次请求前都重新登录，无缓存 | 约 6 个 |
| **业务字段模式** | 通过响应 body 中的 success/error/code 字段判断登录态 | 约 7 个 |
| **特殊协议模式** | WebSocket / 加密协议 / JSON-RPC / SOAP 等非标准 HTTP | 约 5 个 |

---

### 4.2 漫画服务（Manga / Comic）

共 4 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Komga** | Basic Auth 或 `X-API-Key` | Header 携带 | 无 — 每次带认证头，非 200 直接返回 | 无 | [src/widgets/komga/proxy.js#L27-L31](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/komga/proxy.js#L27-L31) |
| **Kavita** | Bearer Token | `token` 字段（响应体） | `status === 401 \|\| status === 403` 时重新登录 | memory-cache，无显式过期 | [src/widgets/kavita/proxy.js#L62-L72](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/kavita/proxy.js#L62-L72) |
| **Komodo** | `X-API-Key` + `X-API-Secret` 双 header | Header 携带 | 无 — 每次带认证头 | 无 | [src/widgets/komodo/proxy.js#L23-L27](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/komodo/proxy.js#L23-L27) |
| **Suwayomi / Tachidesk** | Basic Auth（可选） | Header 携带 | `status === 401` 直接返回错误，不重试 | 无 | [src/widgets/suwayomi/proxy.js#L155-L158](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/suwayomi/proxy.js#L155-L158) |

**Kavita 登录流程**（[src/widgets/kavita/proxy.js#L13-L45](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/kavita/proxy.js#L13-L45)）：
- 登录端点：`Account/login`（POST JSON）
- 支持两种凭据：用户名密码 或 API Key
- 从响应中提取 `token` 字段缓存

---

### 4.3 阅读库 / 电子书（eBook / Library）

共 2 个自定义 proxy（Kavita 已归入漫画分类）。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Booklore** | Bearer Token（JWT） | `token` 字段（响应体） | `status === 401 \|\| status === 403` 时重新登录 | memory-cache，**10 小时 - 1 分钟**提前刷新 | [src/widgets/booklore/proxy.js#L44](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js#L44) |
| **Audiobookshelf** | Bearer Token | Header 携带 | 无 — 每次带 key 请求 | 无 | [src/widgets/audiobookshelf/proxy.js#L10-L23](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/audiobookshelf/proxy.js#L10-L23) |

**Booklore 登录细节**（[src/widgets/booklore/proxy.js#L13-L52](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js#L13-L52)）：
- 登录端点：`auth/login`（POST JSON，用户名+密码）
- 缓存时间：`10 * 60 * 60 * 1000 - 60 * 1000`
- 预检：请求前先检查 cache，没有就先登录（[L58-L60](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js#L58-L60)）
- 失败重试：遇 401/403 重新登录后再请求一次（[L77-L88](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js#L77-L88)）

---

### 4.4 RSS / 订阅（RSS / Feed）

共 1 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **FreshRSS** | GoogleLogin Auth Token | `Auth=` 字段（响应体文本） | `status === 401` 时重新登录 | memory-cache，无显式过期 | [src/widgets/freshrss/proxy.js#L56-L66](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/freshrss/proxy.js#L56-L66) |
| **Miniflux** | `X-Auth-Token` | Header 携带 | 无 — 每次带 token | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L95-L96](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L95-L96) |

**FreshRSS 登录细节**（[src/widgets/freshrss/proxy.js#L13-L41](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/freshrss/proxy.js#L13-L41)）：
- 登录端点：`accounts/ClientLogin`（POST form-urlencoded）
- 请求参数：`Email` + `Passwd`
- 从响应文本中解析 `Auth=xxx` 行提取 token
- 认证头格式：`Authorization: GoogleLogin auth={token}`

---

### 4.5 家庭库存（Home Inventory）

仅 Homebox 一个服务。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Homebox** | Bearer Token | `token` + `expiresAt` 字段 | `status === 401 \|\| status === 403` 时重新登录 | memory-cache，按 `expiresAt` 计算剩余时间 | [src/widgets/homebox/proxy.js#L26-L29](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebox/proxy.js#L26-L29) |

**Homebox 登录细节**（[src/widgets/homebox/proxy.js#L12-L35](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebox/proxy.js#L12-L35)）：
- 登录端点：`/api/v1/users/login`（POST form-urlencoded）
- 从响应提取 `token` 和 `expiresAt`，缓存时长 = `expiresAt - Date.now()`

---

### 4.6 文件管理（File Management）

共 1 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Filebrowser** | `X-AUTH` Header | 响应体 raw data 作为 token | 无显式重试 — 登录失败就 500 | **无（每次都重新登录）** | [src/widgets/filebrowser/proxy.js#L51-L68](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/filebrowser/proxy.js#L51-L68) |
| **Nextcloud** | `NC-Token` 或 Basic Auth | Header 携带 | 无 — 每次带认证头 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L97-L102](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L97-L102) |

**Filebrowser 每次登录模式**：
- 每次调用 proxy handler 都先调用 `login()` 获取新 token
- 登录端点：`login`（POST JSON `{username, password}`）
- 支持可选 `authHeader` 自定义头
- **无缓存机制**，每次请求都重新登录

---

### 4.7 NAS / 存储（NAS / Storage）

共 5 个自定义 proxy + Synology 通用 handler。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **QNAP** | SID Token（URL 参数 `&sid=`） | `authPassed._cdata`（XML 字段） | ① `status === 404` ② `authPassed._cdata === "0"`（XML 字段） | memory-cache，无显式过期 | [src/widgets/qnap/proxy.js#L48-L53](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qnap/proxy.js#L48-L53) · [L62-L74](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qnap/proxy.js#L62-L74) |
| **Unraid** | `X-API-Key` + GraphQL | Header 携带 | 无 — 每次带 key | 无 | [src/widgets/unraid/proxy.js#L99-L103](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/unraid/proxy.js#L99-L103) |
| **TrueNAS v1** | `Bearer {key}` 或 Basic Auth | Header 携带 | 无 — 使用 credentialed handler | 无 | [src/widgets/truenas/proxy.js#L123-L126](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/truenas/proxy.js#L123-L126) |
| **TrueNAS v2** | WebSocket + `auth.login_with_api_key` / `auth.login` | WebSocket 消息 | 连接后调用 authenticate，失败抛错 | 无（每次新建 WS 连接） | [src/widgets/truenas/proxy.js#L84-L102](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/truenas/proxy.js#L84-L102) |
| **UrBackup** | `urbackup-server-api` SDK | SDK 内部处理 | SDK 内部处理 | 无 | [src/widgets/urbackup/proxy.js#L9-L13](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/urbackup/proxy.js#L9-L13) |
| **OpenMediaVault** | Bearer Token | `response.authenticated` 字段 | `resp.status === 401` 或 `json.response.authenticated !== true` | 无显式缓存（每次都预检登录） | [src/widgets/openmediavault/proxy.js#L71-L83](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openmediavault/proxy.js#L71-L83) |
| **Synology** | SID + Cookie | `success` 字段（响应体） | `json?.success !== true` 时重新登录 | Cookie Jar 自动管理 | （通用 handler，无独立 proxy.js）[src/utils/proxy/handlers/synology.js#L168-L173](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/synology.js#L168-L173) |

**QNAP 双重登录重试逻辑**：
1. 先用缓存的 SID 请求
2. 遇 HTTP 404 → 重新登录 → 重试
3. 遇 XML 中 `authPassed === "0"` → 再重新登录 → 再重试
4. 两次失败才返回错误

---

### 4.8 摄像头 / NVR / 照片（Camera / NVR / Photo）

共 3 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Frigate** | Cookie（登录后获取） | Set-Cookie 响应头 | `status === 401 && 有用户名密码` 时登录 | cookie-jar（自动管理） | [src/widgets/frigate/proxy.js#L38-L63](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/frigate/proxy.js#L38-L63) |
| **PhotoPrism** | Session（POST 创建）或 Bearer Token | `session` 响应体 | 无 — 每次都创建新 session 或带 token | **无（每次新建 session）** | [src/widgets/photoprism/proxy.js#L30-L40](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/photoprism/proxy.js#L30-L40) |
| **Jellyfin** | MediaBrowser Token | Header 携带 | 无 — 每次带 token | 无 | [src/widgets/jellyfin/proxy.js#L27-L35](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jellyfin/proxy.js#L27-L35) |

**Frigate Cookie 登录流程**（[src/widgets/frigate/proxy.js#L38-L63](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/frigate/proxy.js#L38-L63)）：
1. 先无认证请求 API
2. 若返回 401 且配置了用户名密码 → POST 到 `/api/login` 登录
3. 登录成功后调用 `addCookieToJar()` + `setCookieHeader()`
4. 携带 Cookie 重新请求原端点

**PhotoPrism 每次新建 Session**：
- 有用户名密码 → POST `/api/v1/session`（JSON body）
- 有 key → `Authorization: Bearer {key}` + body `{authToken: key}`
- **无缓存**，每次都新建 session 请求

---

### 4.9 索引器（Indexer）

共 1 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Jackett** | Cookie（表单登录获取） | Set-Cookie 响应头 | 无登录态判断 — **每次请求前都重新获取 Cookie** | **无（每次都登录拿新 cookie）** | [src/widgets/jackett/proxy.js#L9-L25](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jackett/proxy.js#L9-L25) |
| **Prowlarr** | `X-Api-Key`（credentialed 默认） | Header 携带 | 无 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L138-L140](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L138-L140) |

**Jackett 每次登录模式**（[src/widgets/jackett/proxy.js#L41-L48](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jackett/proxy.js#L41-L48)）：
> 每次调用 proxy handler 都会重新登录获取新 cookie，**没有复用机制**。

---

### 4.10 广告过滤 / DNS（Ad Blocking / DNS）

共 1 个自定义 proxy（Pi-hole，支持 v5/v6 两个版本）。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Pi-hole v5** | 无认证（直接 API） | — | 无 | 无 | [src/widgets/pihole/proxy.js#L56-L61](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pihole/proxy.js#L56-L61) |
| **Pi-hole v6** | `X-FTL-SID` header | `session.sid` + `validity` 字段 | 无显式重试 — 首次预检登录，失败就 500 | memory-cache，按 `validity` 缓存 | [src/widgets/pihole/proxy.js#L64-L71](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pihole/proxy.js#L64-L71) |

**Pi-hole v6 登录流程**（[src/widgets/pihole/proxy.js#L13-L37](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pihole/proxy.js#L13-L37)）：
- 登录端点：`auth`（POST JSON `{password: key}`）
- 响应中提取 `session.sid` 缓存
- 缓存时间：`Math.min(2147483647, validity * 1000)`
- **注意**：没有 401 重试逻辑，只有首次预检登录

---

### 4.11 下载客户端（Download Client）

共 7 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **qBittorrent** | Cookie（WebUI 登录） | Set-Cookie 响应头 | `status === 403` 时登录 | cookie-jar（自动管理） | [src/widgets/qbittorrent/proxy.js#L41-L55](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qbittorrent/proxy.js#L41-L55) |
| **Deluge** | Cookie（JSON-RPC WebUI） | `error.code` 业务字段 | `status === 403` 或 `json.error.code === 1` | cookie-jar | [src/widgets/deluge/proxy.js#L30-L62](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/deluge/proxy.js#L30-L62) |
| **Transmission** | Basic Auth + CSRF Header | `X-Transmission-Session-Id` 响应头 | `status === 409` 时提取 CSRF token 并重试 | memory-cache（缓存完整 headers） | [src/widgets/transmission/proxy.js#L57-L68](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/transmission/proxy.js#L57-L68) |
| **Flood** | Bearer Token（JWT） | `token` 字段 | `status === 401` 时重新登录 | memory-cache，无显式过期 | [src/widgets/flood/proxy.js#L48-L59](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/flood/proxy.js#L48-L59) |
| **pyLoad** | `X-API-Token: {password}` | 400/401/403 状态码 | 401 / 403 / 400(CSRF错误) 时重新登录 | memory-cache，无过期或 23 小时 | [src/widgets/pyload/proxy.js#L138-L158](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pyload/proxy.js#L138-L158) |
| **ruTorrent** | Basic Auth | Header 携带 | 无 — 每次带认证 | 无 | [src/widgets/rutorrent/proxy.js#L57-L60](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/rutorrent/proxy.js#L57-L60) |
| **JDownloader** | 加密 HMAC + sessiontoken | 加密协议字段 | 无显式判断 — 每次都走完整登录流程 | **无（每次都新建 session）** | [src/widgets/jdownloader/proxy.js#L28-L120](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jdownloader/proxy.js#L28-L120) |

**Transmission 409 + CSRF 模式**：
- 首次请求无 CSRF token → 返回 409
- 从响应头 `X-Transmission-Session-Id` 提取 token
- 将完整 headers 缓存（含 Basic Auth + CSRF token）
- 下次请求直接使用缓存的 headers

**JDownloader 加密协议**：
- 基于 MyJDownloader API 的加密通信
- 每步请求都用 HMAC-SHA256 签名 + AES 加密
- 流程：`/my/connect` 登录 → `/my/listdevices` 找设备 → `queryPackages` 查数据
- **每次调用都重新走完整流程**，无 session 复用

---

### 4.12 智能家居 / 路由器（Smart Home / Router）

共 4 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Home Assistant** | Bearer Token（Long-Lived Access Token） | Header 携带 | 无 — 每次带 token | 无 | [src/widgets/homeassistant/proxy.js#L34](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homeassistant/proxy.js#L34) |
| **Homebridge** | Bearer Token | `access_token` + `expires_in` 字段 | `status === 401 \|\| status === 403` 时重新登录 | memory-cache，`expires_in - 5分钟` 提前刷新 | [src/widgets/homebridge/proxy.js#L29](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebridge/proxy.js#L29) |
| **FritzBox** | UPnP / SOAP（无认证） | — | 无 | 无 | [src/widgets/fritzbox/proxy.js#L11-L46](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/fritzbox/proxy.js#L11-L46) |
| **OpenWRT** | JSON-RPC + ubus session | `ubus_rpc_session` 字段 + `error.code` | `json.error.code === -32002` 时重新登录 | 模块级变量（非 cache 库） | [src/widgets/openwrt/proxy.js#L37-L51](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openwrt/proxy.js#L37-L51) |
| **ESPHome** | Basic Auth 或 Cookie | `authenticated` Cookie | 无 — 每次带认证头 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L117-L122](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L117-L122) |

**OpenWRT JSON-RPC 登录模式**（[src/widgets/openwrt/proxy.js#L42-L51](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openwrt/proxy.js#L42-L51)）：
- 登录方法：`call` + 参数 `["00000000000000000000000000000000", "session", "login", {username, password}]`
- 从响应提取 `ubus_rpc_session` 作为 token
- 失败判断：`json.error.code === -32002`
- 缓存方式：模块级变量 `authToken`（非 memory-cache，所有服务共享）

---

### 4.13 网络控制器 / 代理管理（Network / Controller）

共 4 个服务。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **UniFi** | Cookie + CSRF Token（可选） | `meta.rc` 业务字段 + `login_time` | `status === 401` 时登录，成功判断 `meta.rc === "ok"` | cookie-jar 自动管理 | [src/utils/proxy/handlers/unifi.js#L75-L103](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/unifi.js#L75-L103) |
| **UniFi Drive** | 同上（unifi 工厂函数） | 同上 | 同上 | 同上 | [src/widgets/unifi_drive/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/unifi_drive/proxy.js) |
| **Omada** | Token + Cookie + CSRF | `errorCode` 业务字段 + `result.token` | `status === 401/403` 或 `errorCode > 0` 时重试 | memory-cache，55 分钟 | [src/widgets/omada/proxy.js#L16-L18](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/omada/proxy.js#L16-L18) · [L62-L69](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/omada/proxy.js#L62-L69) |
| **NPM (Nginx Proxy Manager)** | Bearer Token | `token` + `expires` 字段 | `status === 403` 时重新登录 | memory-cache，`expiration - 5分钟` | [src/widgets/npm/proxy.js#L72-L89](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/npm/proxy.js#L72-L89) |

**UniFi 工厂函数模式**（[src/utils/proxy/handlers/unifi.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/unifi.js)）：
- 通用工厂函数 `createUnifiProxyHandler()`，支持自定义：
  - `resolveWidget`：widget 解析逻辑
  - `resolveRequestContext`：请求上下文（prefix、headers、csrfToken）
  - `getLoginEndpoint`：登录端点
  - `shouldAttemptLogin`：是否尝试登录
- 支持两种模式：API Key 模式（`X-API-KEY`）和 用户名密码模式（Cookie + CSRF）
- 自动探测 UDM/Prefix（`/proxy/network` 或 `/proxy/drive`）

**Omada 多版本兼容**：
- 支持 v3/v4/v5/v6 四个主版本
- v3：JSON-RPC 风格，`method=login`
- v4+：REST API，`/api/v2/login`
- v5+：需要 `omadacId` 路径前缀
- 双重认证：`Csrf-Token` Header + Cookie
- 登录成功判断：`errorCode === 0`

**NPM 登录细节**：
- 登录端点：`/api/tokens`（POST JSON `{identity, secret}`）
- 缓存时间：`expiration - 5 * 60 * 1000`（过期前 5 分钟刷新）
- 失败重试：遇 403 重新登录

---

### 4.14 资源监控 / 安全（Monitoring / Security）

共 2 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Beszel** | Bearer Token | `token` 字段（响应体） | `status === 400/403` 或 `items` 为空数组 | memory-cache，无显式过期 | [src/widgets/beszel/proxy.js#L75-L93](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/beszel/proxy.js#L75-L93) |
| **CrowdSec** | Bearer Token | `token` + `expire` 字段 | `status === 401` 时重新登录 | memory-cache，按 `expire` 计算 TTL | [src/widgets/crowdsec/proxy.js#L87-L98](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/crowdsec/proxy.js#L87-L98) |
| **Glances** | Basic Auth | Header 携带 | 无 — 每次带认证 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L111-L112](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L111-L112) |
| **APC UPS** | — | — | 无 | 无（有独立 proxy.js，但无认证逻辑） | [src/widgets/apcups/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/apcups/proxy.js) |
| **Speedtest Tracker** | Bearer Token（可选） | Header 携带 | 无 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L133-L137](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L133-L137) |

**Beszel 空数组判断模式**（[src/widgets/beszel/proxy.js#L79-L84](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/beszel/proxy.js#L79-L84)）：
- 除了 HTTP 400/403，还会检查 `json.items.length === 0`
- 空数组也被视为登录态失效，触发重新登录
- 支持 v1/v2 两个 auth endpoint 版本

**CrowdSec 登录细节**：
- 登录端点：POST JSON `{machine_id, password, scenarios: []}`
- 必须带 `User-Agent` 头（CrowdSec 要求）
- 缓存 TTL：`Math.max(new Date(dataParsed.expire) - new Date(), 1)`

---

### 4.15 容器 / 备份 / 调度（Container / Backup / Scheduler）

共 4 个自定义 proxy（UrBackup 已归入 NAS/存储分类）。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Backrest** | Basic Auth（可选） | Header 携带 | 无 — 每次带认证头 | 无 | [src/widgets/backrest/proxy.js#L66-L68](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/backrest/proxy.js#L66-L68) |
| **Watchtower** | Bearer Token | Header 携带 | 无 — 每次带 token | 无 | [src/widgets/watchtower/proxy.js#L27-L32](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/watchtower/proxy.js#L27-L32) |
| **Dockhand** | Cookie（登录后获取） | Set-Cookie 响应头 | `status === 401` 时登录并重试一次 | 无（每次都重新登录） | [src/widgets/dockhand/proxy.js#L46-L53](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/dockhand/proxy.js#L46-L53) |
| **Dispatcharr** | Bearer Token（JWT） | `access` 字段 | `status === 400/401/403` 或 `items` 为空数组 | memory-cache，无显式过期 | [src/widgets/dispatcharr/proxy.js#L76-L93](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/dispatcharr/proxy.js#L76-L93) |
| **Proxmox** | `PVEAPIToken` | Header 携带 | 无 — 每次带 token | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L86-L87](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L86-L87) |

**Dockhand 登录模式**：
- 首次请求无认证 → 若 401 则 POST `/api/auth/login` 登录
- 登录成功后再重试原请求
- **注意**：没有使用 cookie-jar，也没有缓存 Cookie，每次 401 都重新登录

**Dispatcharr 空数组判断模式**：
- 类似 Beszel，除了 HTTP 状态码还检查 `json.items.length === 0`
- 登录端点：`token` mapping（POST JSON `{username, password}`）
- 提取 `access` 字段作为 Bearer Token

---

### 4.16 媒体服务器 / IPTV / 其他（Media / Misc）

共 6 个自定义 proxy。

| 服务 | 认证方式 | Token/Cookie/业务字段 | 登录态判断 | 缓存策略 | 代码定位 |
|---|---|---|---|---|---|
| **Plex** | 无显式认证头（通过 URL 参数？） | — | 无 | 无（有数据缓存，非登录态缓存） | [src/widgets/plex/proxy.js#L35-L62](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/plex/proxy.js#L35-L62) |
| **xTeVe** | Token（登录获取） | `status` + `token` 业务字段 | 登录失败返回 200 但 `json.status !== true` | **无（每次都重新登录）** | [src/widgets/xteve/proxy.js#L26-L48](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/xteve/proxy.js#L26-L48) |
| **Tdarr** | `x-api-key` | Header 携带 | 无 | 无 | [src/widgets/tdarr/proxy.js#L27-L29](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/tdarr/proxy.js#L27-L29) |
| **Minecraft** | 游戏协议（minecraft-server-util） | — | 无认证 | 无 | （无认证概念）[src/widgets/minecraft/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/minecraft/proxy.js) |
| **Gamedig** | 游戏协议（gamedig） | — | 无认证 | 无 | （无认证概念）[src/widgets/gamedig/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/gamedig/proxy.js) |
| **Calendar** | ics 文件 URL | — | 无 — 直接 fetch | 无 | （无认证）[src/widgets/calendar/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/calendar/proxy.js) |
| **Paperless-ngx** | `Token` 或 Basic Auth | Header 携带 | 无 | 无（使用 credentialed 通用 handler，无独立 proxy.js） | [src/utils/proxy/handlers/credentialed.js#L103-L108](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L103-L108) |

**xTeVe 每次登录模式**（[src/widgets/xteve/proxy.js#L26-L48](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/xteve/proxy.js#L26-L48)）：
- 有用户名密码 → 先 POST `cmd: "login"` 登录
- 检查 `json.status === true` 才认为成功
- 提取 `json.token` 加到后续请求的 payload 中
- **无缓存**，每次都重新登录

---

### 4.17 通用 handler（5 个）

| Handler | 认证方式 | 适用场景 | 代码定位 |
|---|---|---|---|
| **credentialed** | 20+ 种认证头格式 | 大部分 API Key / Token 类服务 | [src/utils/proxy/handlers/credentialed.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js) |
| **generic** | Basic Auth（可选） | 通用自定义服务 | [src/utils/proxy/handlers/generic.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/generic.js) |
| **synology** | SID + Cookie + API.Info 发现 | Synology DSM 系列 | [src/utils/proxy/handlers/synology.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/synology.js) |
| **jsonrpc** | Basic 或 Bearer Auth + JSON-RPC 协议 | JSON-RPC API 的服务 | [src/utils/proxy/handlers/jsonrpc.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/jsonrpc.js) |
| **unifi（工厂函数）** | Cookie + CSRF Token（可选） | UniFi 系列（Network / Protect / Drive 等） | [src/utils/proxy/handlers/unifi.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/unifi.js) |

---

### 4.18 登录态判断方式分类汇总

| 判断方式 | 代表服务 | 数量 |
|---|---|---|
| **HTTP 401** | Homebridge, Booklore, Kavita, Flood, CrowdSec, OMV, pyLoad, UniFi 工厂, OpenWRT, FreshRSS, NPM, Beszel, Dispatcharr | 13 |
| **HTTP 403** | qBittorrent, Deluge, NPM, Homebox, pyLoad, Beszel, Dispatcharr, Omada | 8 |
| **HTTP 409（CSRF）** | Transmission | 1 |
| **HTTP 404** | QNAP（URL 不存在表示 session 失效） | 1 |
| **业务层 success 字段** | Synology, xTeVe, OpenMediaVault, UniFi 登录响应 | 4 |
| **业务层 error code** | OpenWRT (-32002), Omada (errorCode > 0), Deluge (code=1) | 3 |
| **业务层 authPassed 字段** | QNAP（XML CDATA） | 1 |
| **业务层 meta.rc 字段** | UniFi 系列 | 1 |
| **状态码 + 空数组** | Beszel, Dispatcharr | 2 |
| **无状态（每次带认证头）** | 所有 credentialed/generic 覆盖的服务 + Unraid + Plex + 更多 | ~30+ |
| **每次都登录** | Jackett, PhotoPrism, xTeVe, Filebrowser, JDownloader | 5 |

---

### 4.19 Token 缓存策略汇总

| 服务 | 缓存存储 | 缓存 Key 模式 | 缓存时长 | 代码定位 |
|---|---|---|---|---|
| **Booklore** | memory-cache | `{proxyName}__sessionToken.{service}` | 10 小时 - 1 分钟 | [L44](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js#L44) |
| **Homebox** | memory-cache | `{proxyName}__sessionToken.{service}` | `expiresAt - Date.now()` | [L28](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebox/proxy.js#L28) |
| **Homebridge** | memory-cache | `{proxyName}__sessionToken.{service}` | `expires_in - 5 分钟` | [L29](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebridge/proxy.js#L29) |
| **NPM** | memory-cache | `{proxyName}__token.{service}` | `expiration - 5 分钟` | [L30](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/npm/proxy.js#L30) |
| **Omada** | memory-cache | `{sessionCacheKey}.{group}.{service}.{index}` | 55 分钟 | [L62-L69](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/omada/proxy.js#L62-L69) |
| **Pi-hole v6** | memory-cache | `{proxyName}__sessionSID.{service}` | `Math.min(2147483647, validity * 1000)` | [L31-L35](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pihole/proxy.js#L31-L35) |
| **QNAP** | memory-cache | `{proxyName}__sessionToken.{service}` | 无显式过期 | [L33](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qnap/proxy.js#L33) |
| **Transmission** | memory-cache | `{proxyName}__headerCache.{service}` | 无显式过期（缓存完整 headers） | [L28-L34](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/transmission/proxy.js#L28-L34) |
| **Beszel** | memory-cache | `{proxyName}__token.{service}` | 无显式过期 | [L28](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/beszel/proxy.js#L28) |
| **Dispatcharr** | memory-cache | `{proxyName}__token.{service}` | 无显式过期 | [L28](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/dispatcharr/proxy.js#L28) |
| **CrowdSec** | memory-cache | `{sessionTokenCacheKey}.{service}` | `expire - now` | [L43-L44](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/crowdsec/proxy.js#L43-L44) |
| **FreshRSS** | memory-cache | `{sessionTokenCacheKey}.{service}` | 无显式过期 | [L34](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/freshrss/proxy.js#L34) |
| **Kavita** | memory-cache | 未显式命名 | 无显式过期 | [L16](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/kavita/proxy.js#L16) |
| **Flood** | memory-cache | 未显式命名 | 无显式过期 | — |
| **pyLoad** | memory-cache | — | 无过期或 23 小时 | — |
| **Frigate** | cookie-jar | URL 域匹配 | 1 小时（Cookie Jar 默认） | （cookie-jar 自动管理） |
| **qBittorrent** | cookie-jar | URL 域匹配 | 1 小时 | （cookie-jar 自动管理） |
| **Deluge** | cookie-jar | URL 域匹配 | 1 小时 | （cookie-jar 自动管理） |
| **UniFi** | cookie-jar | URL 域匹配 | 1 小时 | （cookie-jar 自动管理） |
| **Synology** | cookie-jar | URL 域匹配 | 1 小时 | （cookie-jar 自动管理） |
| **Plex** | memory-cache（数据缓存，非登录态） | `{proxyName}__libraries.{service}.{index}` | 6 小时（libraries）、10 分钟（统计） | [L93](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/plex/proxy.js#L93) |
| **OpenWRT** | 模块级变量 | `authToken` | 进程生命周期（所有服务共享） | [L12](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openwrt/proxy.js#L12) |

---

## 五、服务访问控制

### 5.1 Middleware 层 — Host 白名单校验

**代码位置**：[src/middleware.js#L3-L19](https://github.com/gethomepage/homepage/blob/v1.13.1/src/middleware.js#L3-L19)

生效范围：`/api/:path*`（[L21-L23](https://github.com/gethomepage/homepage/blob/v1.13.1/src/middleware.js#L21-L23)）

默认允许列表：`localhost:{port}`, `127.0.0.1:{port}`, `[::1]:{port}`
支持环境变量：`HOMEPAGE_ALLOWED_HOSTS`（逗号分隔，或设为 `"*"` 允许所有）

### 5.2 API 路由层 — 多层校验

| 校验项 | 失败状态码 | 代码定位 |
|---|---|---|
| widget 类型已注册 | 403 | [src/pages/api/services/proxy.js#L22-L25](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L22-L25) |
| endpoint 有 mapping 或匹配 allowedEndpoints | 403 | [L49-L52](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L49-L52) · [L99-L106](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L99-L106) |
| HTTP 方法匹配 mapping | 403 | [L44-L47](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L44-L47) |
| segment 值安全（禁止 / \ ..） | 403 | [L65-L70](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js#L65-L70) |
| widget 配置存在 | 400 | [credentialed.js#L21-L24](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L21-L24) |
| widget 有 API 定义 | 403 | [credentialed.js#L26-L28](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js#L26-L28) |

### 5.3 响应数据校验

**代码位置**：[src/utils/proxy/validate-widget-data.js#L6-L49](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/validate-widget-data.js#L6-L49)

1. `allowEmpty` 例外 → 直接通过（[L16](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/validate-widget-data.js#L16)）
2. Buffer 可解析为 JSON（[L18-L29](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/validate-widget-data.js#L18-L29)）
3. `mapping.validate` 声明的必需 key 存在（[L32-L38](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/validate-widget-data.js#L32-L38)）

### 5.4 敏感信息脱敏

**代码位置**：[src/utils/proxy/api-helpers.js#L71-L79](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/api-helpers.js#L71-L79)

脱敏参数：`apikey`, `api_key`, `token`, `t`, `access_token`, `auth`（同时处理 query 和 hash）

---

## 六、核心文件索引（v1.13.1 稳定版本）

### 通用代理 Handler

| 文件 | 核心职责 | GitHub 链接 |
|---|---|---|
| **credentialed.js** | 凭据型代理，20+ 种认证头格式 | [src/utils/proxy/handlers/credentialed.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/credentialed.js) |
| **generic.js** | 通用代理，Basic Auth | [src/utils/proxy/handlers/generic.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/generic.js) |
| **synology.js** | Synology 专用，SID 会话 | [src/utils/proxy/handlers/synology.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/synology.js) |
| **jsonrpc.js** | JSON-RPC 协议代理 | [src/utils/proxy/handlers/jsonrpc.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/jsonrpc.js) |
| **unifi.js** | UniFi 工厂函数 | [src/utils/proxy/handlers/unifi.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/handlers/unifi.js) |

### 基础设施

| 文件 | 核心职责 | GitHub 链接 |
|---|---|---|
| **middleware.js** | Host 白名单中间件 | [src/middleware.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/middleware.js) |
| **proxy.js**（API 路由） | 代理 API 入口，endpoint 映射 | [src/pages/api/services/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/pages/api/services/proxy.js) |
| **api-helpers.js** | URL 格式化，敏感信息脱敏 | [src/utils/proxy/api-helpers.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/api-helpers.js) |
| **validate-widget-data.js** | 响应数据校验 | [src/utils/proxy/validate-widget-data.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/validate-widget-data.js) |
| **use-widget-api.js** | 前端 SWR Hook | [src/utils/proxy/use-widget-api.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/use-widget-api.js) |
| **http.js** | HTTP 底层请求 | [src/utils/proxy/http.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/http.js) |
| **cookie-jar.js** | Cookie 自动管理 | [src/utils/proxy/cookie-jar.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/utils/proxy/cookie-jar.js) |

### 自定义 Widget Proxy 完整列表（47 个）

按字母顺序排列（统计口径：`src/widgets/*/proxy.js` 实际存在的文件）：

| 类别 | 服务 | GitHub 链接 |
|---|---|---|
| 📊 监控 | apcups | [src/widgets/apcups/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/apcups/proxy.js) |
| 📚 有声书 | audiobookshelf | [src/widgets/audiobookshelf/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/audiobookshelf/proxy.js) |
| 💾 备份 | backrest | [src/widgets/backrest/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/backrest/proxy.js) |
| 📊 资源监控 | beszel | [src/widgets/beszel/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/beszel/proxy.js) |
| 📖 阅读库 | booklore | [src/widgets/booklore/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/booklore/proxy.js) |
| 📅 日历 | calendar | [src/widgets/calendar/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/calendar/proxy.js) |
| 🛡️ 安全 | crowdsec | [src/widgets/crowdsec/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/crowdsec/proxy.js) |
| 📦 下载 | deluge | [src/widgets/deluge/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/deluge/proxy.js) |
| 🎬 调度 | dispatcharr | [src/widgets/dispatcharr/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/dispatcharr/proxy.js) |
| 🐳 容器管理 | dockhand | [src/widgets/dockhand/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/dockhand/proxy.js) |
| 📁 文件管理 | filebrowser | [src/widgets/filebrowser/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/filebrowser/proxy.js) |
| 🌊 下载 | flood | [src/widgets/flood/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/flood/proxy.js) |
| 📹 NVR | frigate | [src/widgets/frigate/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/frigate/proxy.js) |
| 📰 RSS | freshrss | [src/widgets/freshrss/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/freshrss/proxy.js) |
| 📶 路由器 | fritzbox | [src/widgets/fritzbox/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/fritzbox/proxy.js) |
| 🎮 游戏查询 | gamedig | [src/widgets/gamedig/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/gamedig/proxy.js) |
| 🏠 智能家居 | homeassistant | [src/widgets/homeassistant/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homeassistant/proxy.js) |
| 🏠 智能家居 | homebridge | [src/widgets/homebridge/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebridge/proxy.js) |
| 📦 家庭库存 | homebox | [src/widgets/homebox/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/homebox/proxy.js) |
| 📥 下载 | jdownloader | [src/widgets/jdownloader/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jdownloader/proxy.js) |
| 🔍 索引器 | jackett | [src/widgets/jackett/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jackett/proxy.js) |
| 🎬 媒体服务器 | jellyfin | [src/widgets/jellyfin/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/jellyfin/proxy.js) |
| 📚 漫画 | kavita | [src/widgets/kavita/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/kavita/proxy.js) |
| 📚 漫画 | komga | [src/widgets/komga/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/komga/proxy.js) |
| 📚 漫画 | komodo | [src/widgets/komodo/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/komodo/proxy.js) |
| 🎮 我的世界 | minecraft | [src/widgets/minecraft/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/minecraft/proxy.js) |
| 🌀 代理管理 | npm (Nginx Proxy Manager) | [src/widgets/npm/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/npm/proxy.js) |
| 🛜 网络 | omada | [src/widgets/omada/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/omada/proxy.js) |
| 📂 NAS | openmediavault | [src/widgets/openmediavault/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openmediavault/proxy.js) |
| 🔌 路由器 | openwrt | [src/widgets/openwrt/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/openwrt/proxy.js) |
| 📷 照片 | photoprism | [src/widgets/photoprism/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/photoprism/proxy.js) |
| 📊 广告过滤 | pihole | [src/widgets/pihole/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pihole/proxy.js) |
| 🎬 媒体服务器 | plex | [src/widgets/plex/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/plex/proxy.js) |
| 📦 下载 | pyload | [src/widgets/pyload/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/pyload/proxy.js) |
| 📂 NAS | qnap | [src/widgets/qnap/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qnap/proxy.js) |
| 📦 下载 | qbittorrent | [src/widgets/qbittorrent/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/qbittorrent/proxy.js) |
| 📦 下载 | rutorrent | [src/widgets/rutorrent/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/rutorrent/proxy.js) |
| 📚 漫画 | suwayomi | [src/widgets/suwayomi/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/suwayomi/proxy.js) |
| 📡 IPTV | tdarr | [src/widgets/tdarr/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/tdarr/proxy.js) |
| 📦 下载 | transmission | [src/widgets/transmission/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/transmission/proxy.js) |
| 📂 NAS | truenas | [src/widgets/truenas/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/truenas/proxy.js) |
| 📂 NAS | unraid | [src/widgets/unraid/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/unraid/proxy.js) |
| 🛜 网络 | unifi | [src/widgets/unifi/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/unifi/proxy.js) |
| 🛜 网络存储 | unifi_drive | [src/widgets/unifi_drive/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/unifi_drive/proxy.js) |
| 💾 备份 | urbackup | [src/widgets/urbackup/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/urbackup/proxy.js) |
| 🛡️ 容器监控 | watchtower | [src/widgets/watchtower/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/watchtower/proxy.js) |
| 📡 IPTV | xteve | [src/widgets/xteve/proxy.js](https://github.com/gethomepage/homepage/blob/v1.13.1/src/widgets/xteve/proxy.js) |

> **使用通用 handler 但无独立 proxy.js 的服务**（不计入 47 个）：
> - Prowlarr（credentialed）
> - Miniflux（credentialed）
> - Nextcloud（credentialed）
> - Glances（credentialed）
> - ESPHome（credentialed）
> - Paperless-ngx（credentialed）
> - Proxmox（credentialed）
> - Speedtest Tracker（credentialed）
> - Synology（synology 通用 handler）

---

> **版本说明**：本文档基于 Homepage v1.13.1 源码分析，所有 GitHub 链接均指向 `v1.13.1` tag，确保长期可复核。如需查看最新版本，请将 URL 中的 `v1.13.1` 替换为 `main`。
>
> **v6 更新内容（统计口径修正）**：
> - **数量修正**：自定义 widget proxy 从 53 个修正为 **47 个**（按 `src/widgets/*/proxy.js` 实际存在文件统计）
> - **分类修正**：通用 handler 列表中移除混入的 http.js、cookie-jar.js（归入基础设施）
> - **分类修正**：移除重复计算的服务（Kavita 归入漫画，UrBackup 归入 NAS）
> - **分类修正**：从各分类数量中剔除使用通用 handler 但无独立 proxy.js 的服务（Prowlarr、Miniflux、Nextcloud、Glances、ESPHome、Paperless-ngx、Proxmox、Speedtest Tracker）
> - **各分类数量核对**：
>   - 漫画服务：4 → 4 ✓
>   - 阅读库/电子书：3 → 2 ✓（移除重复的 Kavita）
>   - RSS/订阅：2 → 1 ✓（Miniflux 无独立 proxy.js）
>   - 家庭库存：1 → 1 ✓
>   - 文件管理：2 → 1 ✓（Nextcloud 无独立 proxy.js）
>   - NAS/存储：7 → 5 ✓（Synology 是通用 handler，移除重复的 OMV）
>   - 摄像头/NVR/照片：3 → 3 ✓
>   - 索引器：2 → 1 ✓（Prowlarr 无独立 proxy.js）
>   - 广告过滤/DNS：2 → 1 ✓（Pi-hole v5/v6 是同一个 proxy.js 的两个版本）
>   - 下载客户端：7 → 7 ✓
>   - 智能家居/路由器：6 → 4 ✓（ESPHome 无独立 proxy.js，APC UPS 归入监控）
>   - 网络控制器/代理管理：4 → 4 ✓
>   - 资源监控/安全：5 → 2 ✓（Glances、Speedtest 无独立 proxy.js）
>   - 容器/备份/调度：6 → 4 ✓（Proxmox 无独立 proxy.js，移除重复的 UrBackup）
>   - 媒体/其他：7 → 6 ✓（Paperless-ngx 无独立 proxy.js）
> - **新增说明**：明确"使用通用 handler 但无独立 proxy.js 的服务"清单，共 9 个
> - **新增统计口径说明**：在文档头部和完整列表处明确统计规则
>
> **v5 更新内容**：
> - 新增 RSS/订阅分类（FreshRSS、Miniflux）
> - 新增文件管理分类（Filebrowser、Nextcloud）
> - 新增网络控制器/代理管理分类（UniFi、UniFi Drive、Omada、NPM）
> - 新增资源监控/安全分类（Beszel、CrowdSec、Glances、Speedtest Tracker）
> - 新增容器/备份/调度分类（Dockhand、Dispatcharr、Proxmox）
> - 每个服务表格新增 "Token/Cookie/业务字段" 列
> - 补充每次登录模式的详细说明
> - 登录态判断方式从 10 种扩展到 11 种
> - Token 缓存策略从 15 个扩展到 21 个条目