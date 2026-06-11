# Homepage 身份认证代理协议分析（v3 - 公开可复核版）

> **项目信息**：Homepage v1.13.1 | 仓库：[gethomepage/homepage](https://github.com/gethomepage/homepage)
>
> **引用格式说明**：本文档所有代码引用均采用「仓库相对路径 + GitHub 公开链接」格式，可直接在浏览器中打开验证。例如：
> `[src/utils/proxy/handlers/credentialed.js#L11-L13](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L11-L13)`

---

## 一、整体架构概览

Homepage 是基于 Next.js 的自托管仪表盘应用，核心设计是**服务端代理（Server-side Proxy）**模式：前端不直接访问后端服务 API，而是通过 `/api/services/proxy` 统一入口，由服务端代理层注入认证头、管理登录态、转发请求。

### 请求流全景

```
浏览器前端
  │  [src/utils/proxy/use-widget-api.js#L5-L16](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/use-widget-api.js#L5-L16)
  │  useWidgetAPI → SWR → /api/services/proxy?group=X&service=Y&endpoint=Z
  ▼
Next.js Middleware
  │  [src/middleware.js#L3-L19](https://github.com/gethomepage/homepage/blob/main/src/middleware.js#L3-L19)
  │  Host 白名单校验
  ▼
API 路由入口
  │  [src/pages/api/services/proxy.js#L10-L115](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L10-L115)
  │  解析 widget 类型 → 选择 proxyHandler → endpoint mapping
  ▼
Proxy Handler 层
  │  credentialed / generic / synology / jsonrpc / unifi / 各 widget 自定义 proxy
  │  注入认证头 / 管理登录态 / Cookie Jar
  ▼
HTTP 底层
  │  [src/utils/proxy/http.js#L252-L293](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L252-L293)
  │  httpProxy() → follow-redirects → 目标服务 API
  ▼
目标后端服务
```

---

## 二、认证头传递机制

认证头构建发生在 **Proxy Handler 层**，核心逻辑集中在 `credentialed.js`。

### 2.1 认证头构建的三层合并

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L33-L38](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L33-L38)

```javascript
const headers = {
  "Content-Type": "application/json",
  ...(widgets[widget.type].headers ?? {}),   // 1. widget 类型默认头
  ...(widget.headers ?? {}),                  // 2. 用户配置的自定义头
  ...(req.extraHeaders ?? {}),               // 3. mapping 中定义的额外头
};
```

**合并优先级**（从低到高）：
1. **第一层**：widget 类型注册时的默认 `headers`（如 `withheaders` 类型自带 `X-Widget: 1`）
2. **第二层**：用户在 `services.yaml` 中为服务配置的 `headers` 字段，可覆盖类型默认值
3. **第三层**：`req.extraHeaders`，由 [src/pages/api/services/proxy.js#L88-L90](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L88-L90) 在 endpoint mapping 中通过 `mapping.headers` 注入

### 2.2 按 widget 类型分发的认证头策略

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L40-L140](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L40-L140)

| widget 类型 | 认证头格式 | 代码定位 |
|---|---|---|
| `stocks` (finnhub) | `X-Finnhub-Token: {key}` | [src/utils/proxy/handlers/credentialed.js#L40-L44](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L40-L44) |
| `coinmarketcap` | `X-CMC_PRO_API_KEY: {key}` | [src/utils/proxy/handlers/credentialed.js#L45-L46](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L45-L46) |
| `gotify` | `X-gotify-Key: {key}` | [src/utils/proxy/handlers/credentialed.js#L47-L48](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L47-L48) |
| `checkmk` | `Authorization: Bearer {username} {password}` | [src/utils/proxy/handlers/credentialed.js#L49-L51](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L49-L51) |
| `argocd`, `authentik`, `mealie`, `vikunja` 等 20+ 类型 | `Authorization: Bearer {key}` | [src/utils/proxy/handlers/credentialed.js#L52-L73](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L52-L73) |
| `truenas` | `Bearer {key}` 或 `Basic {user:pass}` | [src/utils/proxy/handlers/credentialed.js#L74-L79](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L74-L79) |
| `ntfy` | `Bearer {key}` 或 `Basic {user:pass}` | [src/utils/proxy/handlers/credentialed.js#L80-L85](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L80-L85) |
| `proxmox` | `PVEAPIToken={username}={password}` | [src/utils/proxy/handlers/credentialed.js#L86-L87](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L86-L87) |
| `proxmoxbackupserver` | `PBSAPIToken={username}:{password}` | [src/utils/proxy/handlers/credentialed.js#L88-L90](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L88-L90) |
| `autobrr`, `jellystat` | `X-API-Token: {key}` | [src/utils/proxy/handlers/credentialed.js#L91-L92](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L91-L92) |
| `tubearchivist` | `Authorization: Token {key}` | [src/utils/proxy/handlers/credentialed.js#L93-L94](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L93-L94) |
| `miniflux` | `X-Auth-Token: {key}` | [src/utils/proxy/handlers/credentialed.js#L95-L96](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L95-L96) |
| `nextcloud` | `NC-Token: {key}` 或 `Basic {user:pass}` | [src/utils/proxy/handlers/credentialed.js#L97-L102](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L97-L102) |
| `paperlessngx` | `Token {key}` 或 `Basic {user:pass}` | [src/utils/proxy/handlers/credentialed.js#L103-L108](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L103-L108) |
| `azuredevops` | `Basic {base64("$:{key}")}` | [src/utils/proxy/handlers/credentialed.js#L109-L110](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L109-L110) |
| `glances` | `Basic {user:pass}` | [src/utils/proxy/handlers/credentialed.js#L111-L112](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L111-L112) |
| `plantit` | `Key: {key}` | [src/utils/proxy/handlers/credentialed.js#L113-L114](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L113-L114) |
| `myspeed` | `Password: {password}` | [src/utils/proxy/handlers/credentialed.js#L115-L116](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L115-L116) |
| `esphome` | `Basic {user:pass}` 或 `Cookie: authenticated={key}` | [src/utils/proxy/handlers/credentialed.js#L117-L122](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L117-L122) |
| `wgeasy` | `Basic {user:pass}` 或 `Authorization: {password}` | [src/utils/proxy/handlers/credentialed.js#L123-L128](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L123-L128) |
| `trilium` | `Authorization: {key}` | [src/utils/proxy/handlers/credentialed.js#L129-L130](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L129-L130) |
| `gitlab` | `PRIVATE-TOKEN: {key}` | [src/utils/proxy/handlers/credentialed.js#L131-L132](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L131-L132) |
| `speedtest` | `Bearer {key}`（v1 不需要） | [src/utils/proxy/handlers/credentialed.js#L133-L137](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L133-L137) |
| **其他（默认 fallback）** | `X-API-Key: {key}` | [src/utils/proxy/handlers/credentialed.js#L138-L140](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L138-L140) |

### 2.3 Basic Auth 辅助函数

**代码位置**：[src/utils/proxy/handlers/credentialed.js#L11-L13](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L11-L13)

```javascript
function basicAuthHeader(widget) {
  return `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.4 generic handler 的认证头

**代码位置**：[src/utils/proxy/handlers/generic.js#L28-L36](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/generic.js#L28-L36)

```javascript
const headers = {
  ...(widgets[widget.type].headers ?? {}),
  ...(widget.headers ?? {}),
  ...(req.extraHeaders ?? {}),
};

if (widget.username && widget.password) {
  headers.Authorization = `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.5 Jellyfin 的特殊认证头（MediaBrowser Token）

**代码位置**：[src/widgets/jellyfin/proxy.js#L27-L35](https://github.com/gethomepage/homepage/blob/main/src/widgets/jellyfin/proxy.js#L27-L35)

```javascript
const deviceIdRaw = widget.deviceId ?? `${widget.service_group || "group"}-${widget.service_name || "service"}`;
const deviceId = encodeURIComponent(deviceIdRaw);
const authHeader = `MediaBrowser Token="${encodeURIComponent(
  widget.key,
)}", Client="Homepage", Device="Homepage", DeviceId="${deviceId}", Version="1.0.0"`;
```

---

## 三、代理转发逻辑

### 3.1 API 路由入口

**代码位置**：[src/pages/api/services/proxy.js#L10-L115](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L10-L115)

处理流程：
1. **解析参数**：从 `req.query` 获取 `service`、`group`、`index`（[L12](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L12)）
2. **获取 widget 配置**：调用 `getServiceWidget()` 查找配置（[L13](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L13)）
3. **确定 widget 类型**：特殊映射（`calendar` → `ical`，`unifi_console` 特殊处理）（[L16-L18](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L16-L18)）
4. **查找 proxy handler**：优先 `widget.proxyHandler`，否则 `genericProxyHandler`（[L27](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L27)）
5. **endpoint mapping**：将前端 opaque endpoint 映射为真实 API endpoint（[L36-L97](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L36-L97)）

### 3.2 Endpoint Mapping 机制

**代码位置**：[src/pages/api/services/proxy.js#L36-L97](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L36-L97)

```javascript
if (widget?.mappings) {
  const mapping = widget?.mappings?.[req.query.endpoint];
  const mappingParams = mapping?.params;
  const optionalParams = mapping?.optionalParams;
  const map = mapping?.map;
  const endpoint = mapping?.endpoint;
  const endpointProxy = mapping?.proxyHandler || serviceProxyHandler;

  // 方法校验
  if (mapping?.method && mapping.method !== req.method) {
    return res.status(403).json({ error: "Unsupported method" });
  }

  // endpoint 存在性校验
  if (!endpoint) {
    return res.status(403).json({ error: "Unsupported service endpoint" });
  }

  req.method = mapping?.method || "GET";
  if (mapping?.body) req.body = mapping?.body;
  req.query.endpoint = endpoint;

  // 路径参数安全校验：禁止 / \ ..
  if (req.query.segments) {
    const segments = JSON.parse(req.query.segments);
    let validSegments = true;
    Object.keys(segments).forEach((key) => {
      if (!mapping.segments.includes(key)) {
        validSegments = false;
      } else if (segments[key].includes("/") || segments[key].includes("\\") || segments[key].includes("..")) {
        validSegments = false;
      }
    });
    if (!validSegments) return res.status(403).json({ error: "Unsupported segment" });
    req.query.endpoint = formatApiCall(endpoint, segments);
  }

  // 查询参数白名单过滤
  if (req.query.query && (mappingParams || optionalParams)) {
    // ... 白名单过滤逻辑
  }

  // mapping 级别的额外 headers
  if (mapping?.headers) {
    req.extraHeaders = mapping.headers;
  }

  return await endpointProxy(req, res, map);
}
```

mapping 支持的能力清单（已核对）：

| 能力 | 代码定位 |
|---|---|
| endpoint 语义化映射 | [src/pages/api/services/proxy.js#L41](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L41) |
| HTTP 方法覆盖与校验 | [src/pages/api/services/proxy.js#L44-L47](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L44-L47) |
| 请求体覆盖 | [src/pages/api/services/proxy.js#L54-L55](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L54-L55) |
| 路径 segments 参数 + 安全校验 | [src/pages/api/services/proxy.js#L58-L72](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L58-L72) |
| 查询参数白名单过滤 | [src/pages/api/services/proxy.js#L74-L86](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L74-L86) |
| mapping 级额外 headers | [src/pages/api/services/proxy.js#L88-L90](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L88-L90) |
| mapping 级自定义 proxyHandler | [src/pages/api/services/proxy.js#L42](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L42) |
| 响应数据 map 转换 | [src/pages/api/services/proxy.js#L40](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L40) |

### 3.3 allowedEndpoints 正则白名单

**代码位置**：[src/pages/api/services/proxy.js#L99-L103](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L99-L103)

```javascript
if (widget.allowedEndpoints instanceof RegExp) {
  if (widget.allowedEndpoints.test(req.query.endpoint)) {
    return await serviceProxyHandler(req, res);
  }
}
```

无 mapping 的请求必须匹配 `allowedEndpoints` 正则，否则返回 403（[L105-L106](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L105-L106)）。

### 3.4 HTTP 底层实现

**代码位置**：[src/utils/proxy/http.js#L252-L293](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L252-L293)

`httpProxy()` 核心特性（已核对）：

| 特性 | 代码定位 |
|---|---|
| 使用 `follow-redirects` 处理重定向 | [src/utils/proxy/http.js#L5](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L5) |
| 重定向过程自动处理 Cookie | [src/utils/proxy/http.js#L19-L22](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L19-L22) |
| gzip/deflate 自动解压 | [src/utils/proxy/http.js#L35-L53](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L35-L53) |
| 响应收到后存入 Cookie Jar | [src/utils/proxy/http.js#L60](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L60) |
| 自定义 DNS 回退（Alpine/musl 兼容） | [src/utils/proxy/http.js#L111-L225](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L111-L225) |
| Agent 缓存（keepAlive: true） | [src/utils/proxy/http.js#L228-L250](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L228-L250) |
| 自签名证书兼容（rejectUnauthorized: false） | [src/utils/proxy/http.js#L245](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L245) |
| IPv6 禁用支持 | [src/utils/proxy/http.js#L254](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L254) |

**返回值格式**：`[statusCode, contentType, data, responseHeaders, params]`
- 正常路径：[src/utils/proxy/http.js#L61-L62](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L61-L62) → 经 [L270](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L270) 返回
- 错误路径：[src/utils/proxy/http.js#L281-L292](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js#L281-L292)

### 3.5 Cookie Jar 机制

**代码位置**：[src/utils/proxy/cookie-jar.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js)

| 函数 | 职责 | 代码定位 |
|---|---|---|
| `setCookieHeader()` | 请求前从 Jar 取 Cookie 注入头 | [src/utils/proxy/cookie-jar.js#L5-L15](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js#L5-L15) |
| `addCookieToJar()` | 响应后将 `Set-Cookie` 存入 Jar | [src/utils/proxy/cookie-jar.js#L17-L41](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js#L17-L41) |

**Cookie 有效期**：存入 Jar 时设置 1 小时 Max-Age（60 * 60 秒）
- 数组 Cookie：[src/utils/proxy/cookie-jar.js#L29](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js#L29)
- 单个 Cookie：[src/utils/proxy/cookie-jar.js#L34](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js#L34)

### 3.6 前端发起代理请求的封装

**代码位置**：[src/utils/proxy/use-widget-api.js#L5-L16](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/use-widget-api.js#L5-L16)

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  if (options[0] === "") {
    url = null;
  }
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

**代理 URL 生成**：[src/utils/proxy/api-helpers.js#L43-L49](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/api-helpers.js#L43-L49)

```javascript
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  if (queryParams) {
    params.append("query", JSON.stringify(queryParams));
  }
  return `/api/services/proxy?${params.toString()}`;
}
```

---

## 四、登录态判断逻辑（逐个与真实代码对齐）

登录态管理分两大类：**无状态模式**（每次带认证头）和**有状态会话模式**（先登录获取 session，再携带 session 访问）。

### 4.1 无状态模式 — credentialed handler

**代码位置**：[src/utils/proxy/handlers/credentialed.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js)

特点：不做登录流程，每次请求直接注入认证头（API Key / Bearer Token / Basic Auth）。无登录态判断逻辑。

### 4.2 有状态会话模式 — 各 widget 自定义 proxy

通用模式：
```
1. 尝试请求 API
2. 根据判断条件判定未登录
3. 调用 login() 获取 session
4. 将 session 缓存（memory-cache / cookie-jar）
5. 携带 session 重新请求 API
6. 再次失败则返回错误
```

以下按判断条件分类，**每个都已与真实代码逐行对齐**：

---

#### 4.2.1 基于 HTTP 状态码 401 判断

| 服务 | 判断代码定位 | 登录代码定位 |
|---|---|---|
| **Homebridge** | [src/widgets/homebridge/proxy.js#L53](https://github.com/gethomepage/homepage/blob/main/src/widgets/homebridge/proxy.js#L53) `status === 401 \|\| status === 403` | [src/widgets/homebridge/proxy.js#L13-L36](https://github.com/gethomepage/homepage/blob/main/src/widgets/homebridge/proxy.js#L13-L36) |
| **FreshRSS** | [src/widgets/freshrss/proxy.js#L56](https://github.com/gethomepage/homepage/blob/main/src/widgets/freshrss/proxy.js#L56) `status === 401` | [src/widgets/freshrss/proxy.js#L13-L41](https://github.com/gethomepage/homepage/blob/main/src/widgets/freshrss/proxy.js#L13-L41) |
| **Flood** | [src/widgets/flood/proxy.js#L48](https://github.com/gethomepage/homepage/blob/main/src/widgets/flood/proxy.js#L48) `status === 401` | [src/widgets/flood/proxy.js#L8-L27](https://github.com/gethomepage/homepage/blob/main/src/widgets/flood/proxy.js#L8-L27) |
| **Crowdsec** | [src/widgets/crowdsec/proxy.js#L87](https://github.com/gethomepage/homepage/blob/main/src/widgets/crowdsec/proxy.js#L87) `status === 401` | [src/widgets/crowdsec/proxy.js#L13-L47](https://github.com/gethomepage/homepage/blob/main/src/widgets/crowdsec/proxy.js#L13-L47) |
| **UniFi（工厂函数）** | [src/utils/proxy/handlers/unifi.js#L75](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/unifi.js#L75) `status === 401 && shouldAttemptLogin(...)` | [src/utils/proxy/handlers/unifi.js#L14-L27](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/unifi.js#L14-L27) |
| **OpenMediaVault** | [src/widgets/openmediavault/proxy.js#L121](https://github.com/gethomepage/homepage/blob/main/src/widgets/openmediavault/proxy.js#L121) `resp.status === 401` | [src/widgets/openmediavault/proxy.js#L61-L83](https://github.com/gethomepage/homepage/blob/main/src/widgets/openmediavault/proxy.js#L61-L83) |
| **OpenWRT** | [src/widgets/openwrt/proxy.js#L115](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L115) `status === 401` | [src/widgets/openwrt/proxy.js#L42-L51](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L42-L51) |

**Homebridge 401 判断代码片段**（[src/widgets/homebridge/proxy.js#L53-L63](https://github.com/gethomepage/homepage/blob/main/src/widgets/homebridge/proxy.js#L53-L63)）：
```javascript
if (status === 401 || status === 403) {
  logger.debug("Homebridge API rejected the request, attempting to obtain new session token");
  const { accessToken } = await login(widget, service);
  headers.Authorization = `Bearer ${accessToken}`;
  [status, contentType, data, responseHeaders] = await httpProxy(url, { method, headers });
}
```

---

#### 4.2.2 基于 HTTP 状态码 403 判断

| 服务 | 判断代码定位 | 登录代码定位 |
|---|---|---|
| **qBittorrent** | [src/widgets/qbittorrent/proxy.js#L41](https://github.com/gethomepage/homepage/blob/main/src/widgets/qbittorrent/proxy.js#L41) `status === 403` | [src/widgets/qbittorrent/proxy.js#L8-L20](https://github.com/gethomepage/homepage/blob/main/src/widgets/qbittorrent/proxy.js#L8-L20) |
| **Deluge** | [src/widgets/deluge/proxy.js#L62](https://github.com/gethomepage/homepage/blob/main/src/widgets/deluge/proxy.js#L62) `status === 403` | [src/widgets/deluge/proxy.js#L39-L41](https://github.com/gethomepage/homepage/blob/main/src/widgets/deluge/proxy.js#L39-L41) |
| **NPM（Nginx Proxy Manager）** | [src/widgets/npm/proxy.js#L72](https://github.com/gethomepage/homepage/blob/main/src/widgets/npm/proxy.js#L72) `status === 403` | [src/widgets/npm/proxy.js#L13-L36](https://github.com/gethomepage/homepage/blob/main/src/widgets/npm/proxy.js#L13-L36) |
| **pyLoad** | [src/widgets/pyload/proxy.js#L138](https://github.com/gethomepage/homepage/blob/main/src/widgets/pyload/proxy.js#L138) `status === 403 \|\| status === 401 \|\| (status === 400 && data?.error?.includes("CSRF token"))` | [src/widgets/pyload/proxy.js#L76-L97](https://github.com/gethomepage/homepage/blob/main/src/widgets/pyload/proxy.js#L76-L97) |

**qBittorrent 403 判断代码片段**（[src/widgets/qbittorrent/proxy.js#L41-L55](https://github.com/gethomepage/homepage/blob/main/src/widgets/qbittorrent/proxy.js#L41-L55)）：
```javascript
if (status === 403) {
  [status, data] = await login(widget);
  if (![200, 204].includes(status)) {
    return res.status(status).end(data);
  }
  if (status === 200 && data.toString() !== "Ok.") {
    return res.status(401).end(data);
  }
  [status, contentType, data] = await httpProxy(url, params);
}
```

---

#### 4.2.3 基于 HTTP 状态码 409（CSRF Token）判断

| 服务 | 判断代码定位 | 处理代码定位 |
|---|---|---|
| **Transmission** | [src/widgets/transmission/proxy.js#L57](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js#L57) `status === 409` | [src/widgets/transmission/proxy.js#L59-L68](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js#L59-L68) |

**Transmission 409 判断代码片段**（[src/widgets/transmission/proxy.js#L57-L68](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js#L57-L68)）：
```javascript
if (status === 409) {
  logger.debug("Transmission is rejecting the request, but returning a CSRF token");
  headers[csrfHeaderName] = responseHeaders[csrfHeaderName];
  cache.put(`${headerCacheKey}.${service}`, headers);
  // retry the request, now with the CSRF token
  [status, contentType, data, responseHeaders] = await httpProxy(url, { method, auth, body, headers });
}
```

---

#### 4.2.4 基于业务层 success 字段判断

| 服务 | 判断代码定位 | 登录代码定位 |
|---|---|---|
| **Synology** | [src/utils/proxy/handlers/synology.js#L169](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/synology.js#L169) `json?.success !== true` | [src/utils/proxy/handlers/synology.js#L17-L41](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/synology.js#L17-L41) |
| **OpenMediaVault** | [src/widgets/openmediavault/proxy.js#L76](https://github.com/gethomepage/homepage/blob/main/src/widgets/openmediavault/proxy.js#L76) `json.response.authenticated !== true`（login 内部校验） | [src/widgets/openmediavault/proxy.js#L61-L83](https://github.com/gethomepage/homepage/blob/main/src/widgets/openmediavault/proxy.js#L61-L83) |
| **UniFi（登录响应校验）** | [src/utils/proxy/handlers/unifi.js#L9-L12](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/unifi.js#L9-L12) `json?.meta?.rc === "ok" \|\| json?.login_time \|\| json?.update_time` | [src/utils/proxy/handlers/unifi.js#L94-L97](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/unifi.js#L94-L97) |

**Synology success 字段判断代码片段**（[src/utils/proxy/handlers/synology.js#L168-L173](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/synology.js#L168-L173)）：
```javascript
let json = asJson(data);
if (json?.success !== true) {
  logger.debug(`Attempting login to ${serviceWidget.type}`);
  [status, contentType, data] = await handleUnsuccessfulResponse(serviceWidget, url, service);
  json = asJson(data);
}
```

---

#### 4.2.5 基于业务层 error code 判断

| 服务 | 判断代码定位 | 登录代码定位 |
|---|---|---|
| **OpenWRT** | [src/widgets/openwrt/proxy.js#L37-L40](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L37-L40) `json?.error?.code === -32002` | [src/widgets/openwrt/proxy.js#L42-L51](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L42-L51) |
| **Omada** | [src/widgets/omada/proxy.js#L16-L18](https://github.com/gethomepage/homepage/blob/main/src/widgets/omada/proxy.js#L16-L18) `status === 401 \|\| status === 403 \|\| responseData?.errorCode > 0` | [src/widgets/omada/proxy.js#L37-L73](https://github.com/gethomepage/homepage/blob/main/src/widgets/omada/proxy.js#L37-L73) |
| **Deluge** | [src/widgets/deluge/proxy.js#L30-L31](https://github.com/gethomepage/homepage/blob/main/src/widgets/deluge/proxy.js#L30-L31) `json.error.code === 1` → 返回 403 | [src/widgets/deluge/proxy.js#L39-L41](https://github.com/gethomepage/homepage/blob/main/src/widgets/deluge/proxy.js#L39-L41) |

**OpenWRT error code 判断代码片段**（[src/widgets/openwrt/proxy.js#L37-L40](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L37-L40)）：
```javascript
function isUnauthorized(data) {
  const json = JSON.parse(data.toString());
  return json?.error?.code === -32002;
}
```

---

#### 4.2.6 基于业务层响应内容 + 空数组判断

| 服务 | 判断代码定位 | 登录代码定位 |
|---|---|---|
| **Beszel** | [src/widgets/beszel/proxy.js#L75-L86](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js#L75-L86) `[400, 403].includes(status) \|\| isEmpty` | [src/widgets/beszel/proxy.js#L13-L34](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js#L13-L34) |
| **Dispatcharr** | [src/widgets/dispatcharr/proxy.js#L76-L86](https://github.com/gethomepage/homepage/blob/main/src/widgets/dispatcharr/proxy.js#L76-L86) `[400, 401, 403].includes(status) \|\| isEmpty` | [src/widgets/dispatcharr/proxy.js#L13-L37](https://github.com/gethomepage/homepage/blob/main/src/widgets/dispatcharr/proxy.js#L13-L37) |

**Beszel 空数组判断代码片段**（[src/widgets/beszel/proxy.js#L75-L86](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js#L75-L86)）：
```javascript
const badRequest = [400, 403].includes(status);
const text = data.toString("utf-8");
let isEmpty = false;
try {
  const json = JSON.parse(text);
  isEmpty = Array.isArray(json.items) && json.items.length === 0;
} catch (err) {
  logger.debug("Failed to parse Beszel response JSON:", err);
}
if (badRequest || isEmpty) {
  // 重新登录并重试
}
```

---

#### 4.2.7 无登录态判断（每次都登录）

| 服务 | 代码定位 | 说明 |
|---|---|---|
| **Filebrowser** | [src/widgets/filebrowser/proxy.js#L51](https://github.com/gethomepage/homepage/blob/main/src/widgets/filebrowser/proxy.js#L51) | 每次请求前先 login 获取 token，无缓存机制 |

**Filebrowser 每次登录代码片段**（[src/widgets/filebrowser/proxy.js#L51-L54](https://github.com/gethomepage/homepage/blob/main/src/widgets/filebrowser/proxy.js#L51-L54)）：
```javascript
const token = await login(widget, service);
if (!token) {
  return res.status(500).json({ error: "Failed to authenticate with Filebrowser" });
}
```

---

### 4.3 Token 缓存策略汇总（已核对行号）

| 服务 | 缓存 Key 模式 | 缓存时长 | 代码定位 |
|---|---|---|---|
| **Homebridge** | `${sessionTokenCacheKey}.${service}` | `expires_in * 1000 - 5 * 60 * 1000`（提前 5 分钟） | [src/widgets/homebridge/proxy.js#L29](https://github.com/gethomepage/homepage/blob/main/src/widgets/homebridge/proxy.js#L29) |
| **NPM** | `${tokenCacheKey}.${service}` | `expiration - 5 * 60 * 1000`（提前 5 分钟） | [src/widgets/npm/proxy.js#L30](https://github.com/gethomepage/homepage/blob/main/src/widgets/npm/proxy.js#L30) |
| **Omada** | `[sessionCacheKey, group, service, index??"0"].join(".")` | `55 * 60 * 1000`（55 分钟） | [src/widgets/omada/proxy.js#L62-L69](https://github.com/gethomepage/homepage/blob/main/src/widgets/omada/proxy.js#L62-L69) |
| **pyLoad** | `${sessionCacheKey}.${service}` | 默认无过期 / pyload-ng 为 23 小时 | [src/widgets/pyload/proxy.js#L91](https://github.com/gethomepage/homepage/blob/main/src/widgets/pyload/proxy.js#L91) |
| **Crowdsec** | `${sessionTokenCacheKey}.${service}` | 服务器返回的 expire 时间差 | [src/widgets/crowdsec/proxy.js#L43-L44](https://github.com/gethomepage/homepage/blob/main/src/widgets/crowdsec/proxy.js#L43-L44) |
| **FreshRSS** | `${sessionTokenCacheKey}.${service}` | 默认无过期 | [src/widgets/freshrss/proxy.js#L34](https://github.com/gethomepage/homepage/blob/main/src/widgets/freshrss/proxy.js#L34) |
| **Beszel** | `${tokenCacheKey}.${service}` | 默认无过期 | [src/widgets/beszel/proxy.js#L28](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js#L28) |
| **Dispatcharr** | `${tokenCacheKey}.${service}` | 默认无过期 | [src/widgets/dispatcharr/proxy.js#L28](https://github.com/gethomepage/homepage/blob/main/src/widgets/dispatcharr/proxy.js#L28) |
| **Transmission** | `${headerCacheKey}.${service}` | 默认无过期（仅缓存 CSRF headers） | [src/widgets/transmission/proxy.js#L28-L34](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js#L28-L34) |

### 4.4 登录态判断总结表

| 判断方式 | 代表服务 | 代码定位 |
|---|---|---|
| HTTP 401 | Homebridge, FreshRSS, Flood, Crowdsec, UniFi, OMV, OpenWRT | 见上表 |
| HTTP 403 | qBittorrent, Deluge, NPM, pyLoad | 见上表 |
| HTTP 409 | Transmission | [src/widgets/transmission/proxy.js#L57](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js#L57) |
| 业务层 success 字段 | Synology, OpenMediaVault, UniFi 登录响应 | [src/utils/proxy/handlers/synology.js#L169](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/synology.js#L169) |
| 业务层 error code | OpenWRT (-32002), Omada (errorCode > 0), Deluge (code=1) | [src/widgets/openwrt/proxy.js#L39](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js#L39) |
| 状态码 + 空数组 | Beszel, Dispatcharr | [src/widgets/beszel/proxy.js#L75](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js#L75) |
| 无状态模式 | 多数 API Key 类服务 | [src/utils/proxy/handlers/credentialed.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js) |

---

## 五、服务访问控制

### 5.1 Middleware 层 — Host 白名单校验

**代码位置**：[src/middleware.js#L3-L19](https://github.com/gethomepage/homepage/blob/main/src/middleware.js#L3-L19)

生效范围：`/api/:path*`（[src/middleware.js#L21-L23](https://github.com/gethomepage/homepage/blob/main/src/middleware.js#L21-L23)）

```javascript
export function middleware(req) {
  const host = req.headers.get("host");
  const port = process.env.PORT || 3000;
  let allowedHosts = [`localhost:${port}`, `127.0.0.1:${port}`, `[::1]:${port}`];
  const allowAll = process.env.HOMEPAGE_ALLOWED_HOSTS === "*";
  if (process.env.HOMEPAGE_ALLOWED_HOSTS) {
    allowedHosts = allowedHosts.concat(process.env.HOMEPAGE_ALLOWED_HOSTS.split(","));
  }
  if (!allowAll && (!host || !allowedHosts.includes(host))) {
    return NextResponse.json({ error: "Host validation failed..." }, { status: 400 });
  }
  return NextResponse.next();
}
```

**安全目的**：防止 DNS 重绑定攻击和未授权的跨域 API 调用。

### 5.2 API 路由层 — 多层校验

**代码位置**：[src/pages/api/services/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js)

| 校验项 | 失败状态码 | 代码定位 |
|---|---|---|
| widget 类型已注册（`!widget`） | 403 | [src/pages/api/services/proxy.js#L22-L25](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L22-L25) |
| endpoint 有 mapping 或匹配 allowedEndpoints | 403 | [src/pages/api/services/proxy.js#L49-L52](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L49-L52)、[L99-L106](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L99-L106) |
| HTTP 方法匹配 mapping | 403 | [src/pages/api/services/proxy.js#L44-L47](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L44-L47) |
| segment 值安全（禁止 / \ ..） | 403 | [src/pages/api/services/proxy.js#L65-L70](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L65-L70) |
| widget 配置存在（credentialed/generic 内） | 400 | [src/utils/proxy/handlers/credentialed.js#L21-L24](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L21-L24) |
| widget 有 API 定义 | 403 | [src/utils/proxy/handlers/credentialed.js#L26-L28](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js#L26-L28) |

### 5.3 响应数据校验

**代码位置**：[src/utils/proxy/validate-widget-data.js#L6-L49](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js#L6-L49)

校验内容：
1. **allowEmpty 例外**：mapping 配置了 `allowEmpty` 且 Buffer 为空 → 直接通过（[L16](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js#L16)）
2. **Buffer 可解析为 JSON**：两次尝试解析（第二次先 strip whitespace）（[L18-L29](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js#L18-L29)）
3. **必需 key 存在**：mapping.validate 声明的 key 必须全部存在（[L32-L38](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js#L32-L38)）

校验失败返回 500 `Invalid data`。

### 5.4 敏感信息脱敏

**代码位置**：[src/utils/proxy/api-helpers.js#L71-L79](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/api-helpers.js#L71-L79)

```javascript
export function sanitizeErrorURL(errorURL) {
  const url = new URL(errorURL);
  ["apikey", "api_key", "token", "t", "access_token", "auth"].forEach((key) => {
    if (url.searchParams.has(key)) url.searchParams.set(key, "***");
    if (url.hash.includes(key)) url.hash = url.hash.replace(new RegExp(`${key}=[^&]+`), `${key}=***`);
  });
  return url.toString();
}
```

脱敏的参数名（同时处理 query string 和 hash）：
`apikey`, `api_key`, `token`, `t`, `access_token`, `auth`

### 5.5 访问控制完整流程图

```
请求进入 /api/services/proxy
  │
  ├─ [src/middleware.js#L3-L19](https://github.com/gethomepage/homepage/blob/main/src/middleware.js#L3-L19)
  │    Host 校验 → 失败 → 400 "Host validation failed"
  │
  ├─ [src/pages/api/services/proxy.js#L10-L115](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js#L10-L115)
  │    ├─ widget 类型已注册 → 未注册 → 403 "Unknown proxy service type"
  │    ├─ endpoint 有 mapping / 匹配 allowedEndpoints → 不匹配 → 403
  │    ├─ HTTP 方法匹配 → 不匹配 → 403 "Unsupported method"
  │    └─ segment 值安全 → 不安全 → 403 "Unsupported segment"
  │
  ├─ Proxy Handler（credentialed/generic 等）
  │    ├─ widget 配置存在 → 不存在 → 400
  │    ├─ widget 有 API 定义 → 无 API → 403
  │    └─ 注入认证头 → httpProxy() → 目标服务
  │         ├─ 401/403/409 → 尝试登录重试
  │         └─ 其他错误 → 返回错误（URL 已脱敏）
  │
  └─ [src/utils/proxy/validate-widget-data.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js)
       响应数据校验 → 无效 → 500 "Invalid data"
```

---

## 六、核心文件索引（公开可复核）

| 文件 | 核心职责 | GitHub 链接 |
|---|---|---|
| **middleware.js** | Next.js 中间件，Host 白名单校验 | [src/middleware.js](https://github.com/gethomepage/homepage/blob/main/src/middleware.js) |
| **proxy.js**（API 路由） | 代理 API 入口，endpoint 映射和分发 | [src/pages/api/services/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/pages/api/services/proxy.js) |
| **credentialed.js** | 凭据型代理处理器，20+ 种认证头构建 | [src/utils/proxy/handlers/credentialed.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/credentialed.js) |
| **generic.js** | 通用代理处理器，Basic Auth | [src/utils/proxy/handlers/generic.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/generic.js) |
| **synology.js** | Synology 专用代理，SID 会话管理 | [src/utils/proxy/handlers/synology.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/synology.js) |
| **jsonrpc.js** | JSON-RPC 代理处理器 | [src/utils/proxy/handlers/jsonrpc.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/jsonrpc.js) |
| **unifi.js** | UniFi 工厂函数，Cookie + CSRF Token | [src/utils/proxy/handlers/unifi.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/handlers/unifi.js) |
| **http.js** | HTTP 底层请求，DNS 回退，Agent 缓存 | [src/utils/proxy/http.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/http.js) |
| **cookie-jar.js** | Cookie 自动管理（tough-cookie） | [src/utils/proxy/cookie-jar.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/cookie-jar.js) |
| **api-helpers.js** | URL 格式化，代理 URL 生成，敏感信息脱敏 | [src/utils/proxy/api-helpers.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/api-helpers.js) |
| **validate-widget-data.js** | 响应数据结构校验 | [src/utils/proxy/validate-widget-data.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/validate-widget-data.js) |
| **use-widget-api.js** | 前端 SWR Hook，发起代理请求 | [src/utils/proxy/use-widget-api.js](https://github.com/gethomepage/homepage/blob/main/src/utils/proxy/use-widget-api.js) |
| **service-helpers.js** | 服务配置加载和 widget 查找 | [src/utils/config/service-helpers.js](https://github.com/gethomepage/homepage/blob/main/src/utils/config/service-helpers.js) |
| **config.js** | 配置文件管理，环境变量替换 | [src/utils/config/config.js](https://github.com/gethomepage/homepage/blob/main/src/utils/config/config.js) |

### 自定义 proxy handler 的代表服务索引

| 服务 | proxy 文件 | 登录态判断方式 | GitHub 链接 |
|---|---|---|---|
| qBittorrent | qbittorrent/proxy.js | HTTP 403 | [src/widgets/qbittorrent/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/qbittorrent/proxy.js) |
| Deluge | deluge/proxy.js | HTTP 403（JSON-RPC） | [src/widgets/deluge/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/deluge/proxy.js) |
| Homebridge | homebridge/proxy.js | HTTP 401/403 | [src/widgets/homebridge/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/homebridge/proxy.js) |
| FreshRSS | freshrss/proxy.js | HTTP 401 | [src/widgets/freshrss/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/freshrss/proxy.js) |
| Flood | flood/proxy.js | HTTP 401 | [src/widgets/flood/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/flood/proxy.js) |
| Crowdsec | crowdsec/proxy.js | HTTP 401 | [src/widgets/crowdsec/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/crowdsec/proxy.js) |
| Jellyfin | jellyfin/proxy.js | 无状态（MediaBrowser Token） | [src/widgets/jellyfin/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/jellyfin/proxy.js) |
| Transmission | transmission/proxy.js | HTTP 409（CSRF） | [src/widgets/transmission/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/transmission/proxy.js) |
| Omada | omada/proxy.js | 401/403 + errorCode > 0 | [src/widgets/omada/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/omada/proxy.js) |
| NPM | npm/proxy.js | HTTP 403 | [src/widgets/npm/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/npm/proxy.js) |
| Beszel | beszel/proxy.js | 400/403 + 空数组 | [src/widgets/beszel/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/beszel/proxy.js) |
| Dispatcharr | dispatcharr/proxy.js | 400/401/403 + 空数组 | [src/widgets/dispatcharr/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/dispatcharr/proxy.js) |
| OpenWRT | openwrt/proxy.js | error.code === -32002 | [src/widgets/openwrt/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/openwrt/proxy.js) |
| pyLoad | pyload/proxy.js | 401/403 + CSRF token | [src/widgets/pyload/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/pyload/proxy.js) |
| OpenMediaVault | openmediavault/proxy.js | HTTP 401 + authenticated 字段 | [src/widgets/openmediavault/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/openmediavault/proxy.js) |
| UniFi / UniFi Drive | unifi/proxy.js · unifi_drive/proxy.js | 工厂函数 createUnifiProxyHandler（HTTP 401） | [src/widgets/unifi/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/unifi/proxy.js) · [src/widgets/unifi_drive/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/unifi_drive/proxy.js) |
| Filebrowser | filebrowser/proxy.js | 每次都登录（无缓存） | [src/widgets/filebrowser/proxy.js](https://github.com/gethomepage/homepage/blob/main/src/widgets/filebrowser/proxy.js) |
