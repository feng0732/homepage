# Homepage 身份认证代理协议分析（v2 - 可复核版）

> 本文档所有代码引用均指向项目真实文件的具体行号，可直接点击跳转验证。

---

## 一、整体架构概览

Homepage 是基于 Next.js 的自托管仪表盘应用，核心设计是**服务端代理（Server-side Proxy）**模式：前端不直接访问后端服务 API，而是通过 `/api/services/proxy` 统一入口，由服务端代理层注入认证头、管理登录态、转发请求。

### 请求流全景（带真实代码引用）

```
浏览器前端
  │  [use-widget-api.js#L5-L16](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/use-widget-api.js#L5-L16)
  │  useWidgetAPI → SWR → /api/services/proxy?group=X&service=Y&endpoint=Z
  ▼
Next.js Middleware
  │  [middleware.js#L3-L19](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js#L3-L19)
  │  Host 白名单校验
  ▼
API 路由入口
  │  [pages/api/services/proxy.js#L10-L115](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L10-L115)
  │  解析 widget 类型 → 选择 proxyHandler → endpoint mapping
  ▼
Proxy Handler 层
  │  credentialed / generic / synology / jsonrpc / unifi / 各 widget 自定义 proxy
  │  注入认证头 / 管理登录态 / Cookie Jar
  ▼
HTTP 底层
  │  [utils/proxy/http.js#L252-L293](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L252-L293)
  │  httpProxy() → follow-redirects → 目标服务 API
  ▼
目标后端服务
```

---

## 二、认证头传递机制

认证头构建发生在 **Proxy Handler 层**，核心逻辑集中在 [credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js)。

### 2.1 认证头构建的三层合并

**代码位置**：[credentialed.js#L33-L38](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L33-L38)

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
3. **第三层**：`req.extraHeaders`，由 [proxy.js#L88-L90](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L88-L90) 在 endpoint mapping 中通过 `mapping.headers` 注入

### 2.2 按 widget 类型分发的认证头策略

**代码位置**：[credentialed.js#L40-L140](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L40-L140)

| widget 类型 | 认证头格式 | 代码定位 |
|---|---|---|
| `stocks` (finnhub) | `X-Finnhub-Token: {key}` | [credentialed.js#L40-L44](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L40-L44) |
| `coinmarketcap` | `X-CMC_PRO_API_KEY: {key}` | [credentialed.js#L45-L46](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L45-L46) |
| `gotify` | `X-gotify-Key: {key}` | [credentialed.js#L47-L48](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L47-L48) |
| `checkmk` | `Authorization: Bearer {username} {password}` | [credentialed.js#L49-L51](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L49-L51) |
| `argocd`, `authentik`, `mealie`, `vikunja` 等 20+ 类型 | `Authorization: Bearer {key}` | [credentialed.js#L52-L73](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L52-L73) |
| `truenas` | `Bearer {key}` 或 `Basic {user:pass}` | [credentialed.js#L74-L79](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L74-L79) |
| `ntfy` | `Bearer {key}` 或 `Basic {user:pass}` | [credentialed.js#L80-L85](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L80-L85) |
| `proxmox` | `PVEAPIToken={username}={password}` | [credentialed.js#L86-L87](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L86-L87) |
| `proxmoxbackupserver` | `PBSAPIToken={username}:{password}` | [credentialed.js#L88-L90](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L88-L90) |
| `autobrr`, `jellystat` | `X-API-Token: {key}` | [credentialed.js#L91-L92](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L91-L92) |
| `tubearchivist` | `Authorization: Token {key}` | [credentialed.js#L93-L94](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L93-L94) |
| `miniflux` | `X-Auth-Token: {key}` | [credentialed.js#L95-L96](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L95-L96) |
| `nextcloud` | `NC-Token: {key}` 或 `Basic {user:pass}` | [credentialed.js#L97-L102](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L97-L102) |
| `paperlessngx` | `Token {key}` 或 `Basic {user:pass}` | [credentialed.js#L103-L108](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L103-L108) |
| `azuredevops` | `Basic {base64("$:{key}")}` | [credentialed.js#L109-L110](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L109-L110) |
| `glances` | `Basic {user:pass}` | [credentialed.js#L111-L112](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L111-L112) |
| `plantit` | `Key: {key}` | [credentialed.js#L113-L114](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L113-L114) |
| `myspeed` | `Password: {password}` | [credentialed.js#L115-L116](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L115-L116) |
| `esphome` | `Basic {user:pass}` 或 `Cookie: authenticated={key}` | [credentialed.js#L117-L122](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L117-L122) |
| `wgeasy` | `Basic {user:pass}` 或 `Authorization: {password}` | [credentialed.js#L123-L128](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L123-L128) |
| `trilium` | `Authorization: {key}` | [credentialed.js#L129-L130](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L129-L130) |
| `gitlab` | `PRIVATE-TOKEN: {key}` | [credentialed.js#L131-L132](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L131-L132) |
| `speedtest` | `Bearer {key}`（v1 不需要） | [credentialed.js#L133-L137](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L133-L137) |
| **其他（默认 fallback）** | `X-API-Key: {key}` | [credentialed.js#L138-L140](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L138-L140) |

### 2.3 Basic Auth 辅助函数

**代码位置**：[credentialed.js#L11-L13](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L11-L13)

```javascript
function basicAuthHeader(widget) {
  return `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.4 generic handler 的认证头

**代码位置**：[generic.js#L28-L36](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js#L28-L36)

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

**代码位置**：[jellyfin/proxy.js#L27-L35](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/jellyfin/proxy.js#L27-L35)

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

**代码位置**：[pages/api/services/proxy.js#L10-L115](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L10-L115)

处理流程：
1. **解析参数**：从 `req.query` 获取 `service`、`group`、`index`、`endpoint`（[L12](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L12)）
2. **获取 widget 配置**：调用 [getServiceWidget()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/service-helpers.js#L754-L761) 查找配置（[L13](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L13)）
3. **确定 widget 类型**：特殊映射（`calendar` → `ical`，`unifi_console` 特殊处理）（[L16-L18](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L16-L18)）
4. **查找 proxy handler**：优先 `widget.proxyHandler`，否则 `genericProxyHandler`（[L27](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L27)）
5. **endpoint mapping**：将前端 opaque endpoint 映射为真实 API endpoint（[L36-L97](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L36-L97)）

### 3.2 Endpoint Mapping 机制

**代码位置**：[proxy.js#L36-L97](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L36-L97)

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
  }

  // 查询参数白名单过滤
  if (req.query.query && (mappingParams || optionalParams)) {
    const queryParams = JSON.parse(req.query.query);
    let params = mappingParams ? mappingParams.concat(filteredOptionalParams) : filteredOptionalParams;
    const query = new URLSearchParams(params.map((p) => [p, queryParams[p]]));
  }

  // mapping 级别的额外 headers
  if (mapping?.headers) {
    req.extraHeaders = mapping.headers;
  }

  return await endpointProxy(req, res, map);
}
```

mapping 支持的能力清单（可复核）：
| 能力 | 代码定位 |
|---|---|
| endpoint 语义化映射 | [L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L41) |
| HTTP 方法覆盖与校验 | [L44-L47](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L44-L47) |
| 请求体覆盖 | [L54-L55](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L54-L55) |
| 路径 segments 参数 + 安全校验 | [L58-L72](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L58-L72) |
| 查询参数白名单过滤 | [L74-L86](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L74-L86) |
| mapping 级额外 headers | [L88-L90](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L88-L90) |
| mapping 级自定义 proxyHandler | [L42](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L42) |
| 响应数据 map 转换 | [L40](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L40) |

### 3.3 allowedEndpoints 正则白名单

**代码位置**：[proxy.js#L99-L103](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L99-L103)

```javascript
if (widget.allowedEndpoints instanceof RegExp) {
  if (widget.allowedEndpoints.test(req.query.endpoint)) {
    return await serviceProxyHandler(req, res);
  }
}
```

无 mapping 的请求必须匹配 `allowedEndpoints` 正则，否则返回 403。

### 3.4 HTTP 底层实现

**代码位置**：[http.js#L252-L293](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L252-L293)

`httpProxy()` 核心特性（可复核）：
| 特性 | 代码定位 |
|---|---|
| 使用 `follow-redirects` 处理重定向 | [L5](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L5) |
| 自定义 DNS 回退（Alpine/musl 兼容） | [L111-L225](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L111-L225) |
| Agent 缓存（keepAlive） | [L228-L250](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L228-L250) |
| 自签名证书兼容（rejectUnauthorized: false） | [L245](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L245) |
| gzip/deflate 自动解压 | [L35-L53](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L35-L53) |
| IPv6 禁用支持 | [L254](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L254) |

**返回值格式**：`[statusCode, contentType, data, responseHeaders, params]`（[L61-L62](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L61-L62)、[L270](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L270)）

### 3.5 Cookie Jar 机制

**代码位置**：[cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js)

| 函数 | 职责 | 代码定位 |
|---|---|---|
| `setCookieHeader()` | 请求前从 Jar 取 Cookie 注入头 | [cookie-jar.js#L5-L15](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js#L5-L15) |
| `addCookieToJar()` | 响应后将 `Set-Cookie` 存入 Jar | [cookie-jar.js#L17-L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js#L17-L41) |
| `beforeRedirect` 钩子 | 重定向时自动处理 Cookie | [http.js#L19-L22](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L19-L22) |

**Cookie 有效期**：存入 Jar 时设置 1 小时 Max-Age（[cookie-jar.js#L29](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js#L29)、[L34](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js#L34)）

### 3.6 前端发起代理请求的封装

**代码位置**：[use-widget-api.js#L5-L16](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/use-widget-api.js#L5-L16)

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

**代理 URL 生成**：[api-helpers.js#L43-L49](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js#L43-L49)

```javascript
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  return `/api/services/proxy?${params.toString()}`;
}
```

---

## 四、登录态判断逻辑（逐个与真实代码对齐）

登录态管理分两大类：**无状态模式**（每次带认证头）和**有状态会话模式**（先登录获取 session，再携带 session 访问）。

### 4.1 无状态模式 — credentialed handler

**代码位置**：[credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js)

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

以下按判断条件分类，**每个都有真实代码对齐**：

---

#### 4.2.1 基于 HTTP 状态码 401 判断

| 服务 | 判断代码 | 登录代码 |
|---|---|---|
| **Homebridge** | [homebridge/proxy.js#L53](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js#L53) `status === 401 || status === 403` | [homebridge/proxy.js#L13-L36](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js#L13-L36) |
| **FreshRSS** | [freshrss/proxy.js#L56](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/freshrss/proxy.js#L56) `status === 401` | [freshrss/proxy.js#L13-L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/freshrss/proxy.js#L13-L41) |
| **Flood** | [flood/proxy.js#L48](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/flood/proxy.js#L48) `status === 401` | [flood/proxy.js#L8-L27](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/flood/proxy.js#L8-L27) |
| **Crowdsec** | [crowdsec/proxy.js#L87](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/crowdsec/proxy.js#L87) `status === 401` | [crowdsec/proxy.js#L13-L47](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/crowdsec/proxy.js#L13-L47) |
| **UniFi（工厂函数）** | [unifi.js#L75](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/unifi.js#L75) `status === 401` | [unifi.js#L14-L27](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/unifi.js#L14-L27) |
| **OpenMediaVault** | [openmediavault/proxy.js#L121](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openmediavault/proxy.js#L121) `resp.status === 401` | [openmediavault/proxy.js#L61-L83](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openmediavault/proxy.js#L61-L83) |
| **OpenWRT** | [openwrt/proxy.js#L115](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L115) `status === 401` | [openwrt/proxy.js#L42-L51](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L42-L51) |

**Homebridge 401 判断代码片段**（[homebridge/proxy.js#L53-L63](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js#L53-L63)）：
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

| 服务 | 判断代码 | 登录代码 |
|---|---|---|
| **qBittorrent** | [qbittorrent/proxy.js#L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/qbittorrent/proxy.js#L41) `status === 403` | [qbittorrent/proxy.js#L8-L20](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/qbittorrent/proxy.js#L8-L20) |
| **Deluge** | [deluge/proxy.js#L62](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/deluge/proxy.js#L62) `status === 403` | [deluge/proxy.js#L39-L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/deluge/proxy.js#L39-L41) |
| **NPM（Nginx Proxy Manager）** | [npm/proxy.js#L72](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/npm/proxy.js#L72) `status === 403` | [npm/proxy.js#L13-L36](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/npm/proxy.js#L13-L36) |
| **pyLoad** | [pyload/proxy.js#L138](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/pyload/proxy.js#L138) `status === 403 \|\| status === 401` | [pyload/proxy.js#L76-L97](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/pyload/proxy.js#L76-L97) |

**qBittorrent 403 判断代码片段**（[qbittorrent/proxy.js#L41-L55](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/qbittorrent/proxy.js#L41-L55)）：
```javascript
if (status === 403) {
  [status, data] = await login(widget);
  if (![200, 204].includes(status)) {
    return res.status(status).end(data);
  }
  [status, contentType, data] = await httpProxy(url, params);
}
```

---

#### 4.2.3 基于 HTTP 状态码 409（CSRF Token）判断

| 服务 | 判断代码 | 登录/处理代码 |
|---|---|---|
| **Transmission** | [transmission/proxy.js#L57](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js#L57) `status === 409` | [transmission/proxy.js#L59-L68](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js#L59-L68) |

**Transmission 409 判断代码片段**（[transmission/proxy.js#L57-L68](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js#L57-L68)）：
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

| 服务 | 判断代码 | 登录代码 |
|---|---|---|
| **Synology** | [synology.js#L169](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js#L169) `json?.success !== true` | [synology.js#L17-L41](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js#L17-L41) |
| **OpenMediaVault** | [openmediavault/proxy.js#L76](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openmediavault/proxy.js#L76) `json.response.authenticated !== true` | [openmediavault/proxy.js#L61-L83](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openmediavault/proxy.js#L61-L83) |

**Synology success 字段判断代码片段**（[synology.js#L168-L173](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js#L168-L173)）：
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

| 服务 | 判断代码 | 登录代码 |
|---|---|---|
| **OpenWRT** | [openwrt/proxy.js#L37-L40](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L37-L40) `json?.error?.code === -32002` | [openwrt/proxy.js#L42-L51](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L42-L51) |
| **Omada** | [omada/proxy.js#L16-L18](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/omada/proxy.js#L16-L18) `status === 401 \|\| status === 403 \|\| responseData?.errorCode > 0` | [omada/proxy.js#L37-L73](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/omada/proxy.js#L37-L73) |

**OpenWRT error code 判断代码片段**（[openwrt/proxy.js#L37-L40](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L37-L40)）：
```javascript
function isUnauthorized(data) {
  const json = JSON.parse(data.toString());
  return json?.error?.code === -32002;
}
```

---

#### 4.2.6 基于业务层响应内容 + 空数组判断

| 服务 | 判断代码 | 登录代码 |
|---|---|---|
| **Beszel** | [beszel/proxy.js#L75](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js#L75) `[400, 403].includes(status) \|\| isEmpty` | [beszel/proxy.js#L13-L34](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js#L13-L34) |
| **Dispatcharr** | [dispatcharr/proxy.js#L76](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/dispatcharr/proxy.js#L76) `[400, 401, 403].includes(status) \|\| isEmpty` | [dispatcharr/proxy.js#L13-L37](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/dispatcharr/proxy.js#L13-L37) |

**Beszel 空数组判断代码片段**（[beszel/proxy.js#L75-L84](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js#L75-L84)）：
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
```

---

#### 4.2.7 无登录态判断（每次都登录）

| 服务 | 代码位置 | 说明 |
|---|---|---|
| **Filebrowser** | [filebrowser/proxy.js#L51](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/filebrowser/proxy.js#L51) | 每次请求前先 login 获取 token，无缓存 |

**Filebrowser 每次登录代码片段**（[filebrowser/proxy.js#L51-L54](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/filebrowser/proxy.js#L51-L54)）：
```javascript
const token = await login(widget, service);
if (!token) {
  return res.status(500).json({ error: "Failed to authenticate with Filebrowser" });
}
```

---

### 4.3 Token 缓存策略汇总

| 服务 | 缓存 Key 模式 | 缓存时长 | 代码定位 |
|---|---|---|---|
| **Homebridge** | `${sessionTokenCacheKey}.${service}` | `expires_in * 1000 - 5 * 60 * 1000` | [homebridge/proxy.js#L29](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js#L29) |
| **NPM** | `${tokenCacheKey}.${service}` | `expiration - 5 * 60 * 1000` | [npm/proxy.js#L30](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/npm/proxy.js#L30) |
| **Omada** | `${sessionCacheKey}.${group}.${service}.${index}` | `55 * 60 * 1000`（55分钟） | [omada/proxy.js#L62-L69](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/omada/proxy.js#L62-L69) |
| **pyLoad** | `${sessionCacheKey}.${service}` | 默认无过期 / pyload-ng 为 23 小时 | [pyload/proxy.js#L91](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/pyload/proxy.js#L91) |
| **Crowdsec** | `${sessionTokenCacheKey}.${service}` | 服务器返回的 expire 时间 | [crowdsec/proxy.js#L43-L44](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/crowdsec/proxy.js#L43-L44) |
| **FreshRSS** | `${sessionTokenCacheKey}.${service}` | 默认无过期 | [freshrss/proxy.js#L34](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/freshrss/proxy.js#L34) |
| **Beszel** | `${tokenCacheKey}.${service}` | 默认无过期 | [beszel/proxy.js#L28](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js#L28) |
| **Dispatcharr** | `${tokenCacheKey}.${service}` | 默认无过期 | [dispatcharr/proxy.js#L28](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/dispatcharr/proxy.js#L28) |
| **Transmission** | `${headerCacheKey}.${service}` | 默认无过期（仅缓存 CSRF headers） | [transmission/proxy.js#L28-L34](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js#L28-L34) |

**提前刷新策略**：Homebridge 和 NPM 都采用「过期时间 - 5 分钟」的策略，确保 token 在真正过期前刷新。

### 4.4 登录态判断总结表

| 判断方式 | 代表服务 | 代码定位 |
|---|---|---|
| HTTP 401 | Homebridge, FreshRSS, Flood, Crowdsec, UniFi, OMV | 见上表 |
| HTTP 403 | qBittorrent, Deluge, NPM, pyLoad | 见上表 |
| HTTP 409 | Transmission | [transmission/proxy.js#L57](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js#L57) |
| 业务层 success 字段 | Synology, OpenMediaVault | [synology.js#L169](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js#L169) |
| 业务层 error code | OpenWRT (-32002), Omada (errorCode > 0) | [openwrt/proxy.js#L39](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js#L39) |
| 业务层响应内容 + 空数组 | Beszel, Dispatcharr | [beszel/proxy.js#L75](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js#L75) |
| 无状态模式 | 多数 API Key 类服务 | [credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js) |

---

## 五、服务访问控制

### 5.1 Middleware 层 — Host 白名单校验

**代码位置**：[middleware.js#L3-L19](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js#L3-L19)

生效范围：`/api/:path*`（[middleware.js#L22](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js#L22)）

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

**代码位置**：[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js)

| 校验项 | 失败状态码 | 代码定位 |
|---|---|---|
| group/service 必填 | 400 | [proxy.js#L185](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L185) |
| widget 配置存在 | 400 | [credentialed.js#L21-L24](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L21-L24) |
| widget 类型已注册 | 403 | [proxy.js#L22-L25](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L22-L25) |
| widget 有 API 定义 | 403 | [credentialed.js#L26-L28](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L26-L28) |
| endpoint 有 mapping 或匹配 allowedEndpoints | 403 | [proxy.js#L49-L52](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L49-L52)、[L99-L103](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L99-L103) |
| HTTP 方法匹配 mapping | 403 | [proxy.js#L44-L47](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L44-L47) |
| segment 值安全（禁止 / \ ..） | 403 | [proxy.js#L65-L68](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L65-L68) |

### 5.3 响应数据校验

**代码位置**：[validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js)

校验内容：
1. 响应可解析为 JSON（[L18-L29](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js#L18-L29)）
2. mapping.validate 声明的必需 key 存在（[L32-L38](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js#L32-L38)）
3. `allowEmpty` 配置允许空 Buffer（[L16](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js#L16)）

校验失败返回 500 `Invalid data`。

### 5.4 敏感信息脱敏

**代码位置**：[api-helpers.js#L71-L78](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js#L71-L78)

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

脱敏的参数名：`apikey`, `api_key`, `token`, `t`, `access_token`, `auth`

### 5.5 访问控制完整流程图

```
请求进入 /api/services/proxy
  │
  ├─ [middleware.js#L3-L19](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js#L3-L19)
  │    Host 校验 → 失败 → 400 "Host validation failed"
  │
  ├─ [proxy.js#L10-L115](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L10-L115)
  │    ├─ group/service 必填 → 缺失 → 400
  │    ├─ widget 配置存在 → 不存在 → 400
  │    ├─ widget 类型已注册 → 未注册 → 403
  │    ├─ widget 有 API 定义 → 无 API → 403
  │    ├─ endpoint 有 mapping / 匹配 allowedEndpoints → 不匹配 → 403
  │    ├─ HTTP 方法匹配 → 不匹配 → 403
  │    └─ segment 值安全 → 不安全 → 403
  │
  ├─ Proxy Handler 注入认证头 → [httpProxy()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L252-L293)
  │    ├─ 401/403/409 → 尝试登录重试
  │    └─ 其他错误 → 返回错误（URL 已脱敏）
  │
  └─ [validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js)
       响应数据校验 → 无效 → 500 "Invalid data"
```

---

## 六、核心文件索引（可点击跳转）

| 文件 | 核心职责 | 链接 |
|---|---|---|
| **middleware.js** | Next.js 中间件，Host 白名单校验 | [middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js) |
| **proxy.js**（API 路由） | 代理 API 入口，endpoint 映射和分发 | [pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js) |
| **credentialed.js** | 凭据型代理处理器，20+ 种认证头构建 | [utils/proxy/handlers/credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js) |
| **generic.js** | 通用代理处理器，Basic Auth | [utils/proxy/handlers/generic.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js) |
| **synology.js** | Synology 专用代理，SID 会话管理 | [utils/proxy/handlers/synology.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js) |
| **jsonrpc.js** | JSON-RPC 代理处理器 | [utils/proxy/handlers/jsonrpc.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/jsonrpc.js) |
| **unifi.js** | UniFi 工厂函数，Cookie + CSRF Token | [utils/proxy/handlers/unifi.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/unifi.js) |
| **http.js** | HTTP 底层请求，DNS 回退，Agent 缓存 | [utils/proxy/http.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js) |
| **cookie-jar.js** | Cookie 自动管理（tough-cookie） | [utils/proxy/cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js) |
| **api-helpers.js** | URL 格式化，代理 URL 生成，敏感信息脱敏 | [utils/proxy/api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js) |
| **validate-widget-data.js** | 响应数据结构校验 | [utils/proxy/validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js) |
| **use-widget-api.js** | 前端 SWR Hook，发起代理请求 | [utils/proxy/use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/use-widget-api.js) |
| **service-helpers.js** | 服务配置加载和 widget 查找 | [utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/service-helpers.js) |
| **config.js** | 配置文件管理，环境变量替换 | [utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/config.js) |

### 自定义 proxy handler 的代表服务

| 服务 | proxy 文件 | 登录态判断方式 |
|---|---|---|
| qBittorrent | [widgets/qbittorrent/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/qbittorrent/proxy.js) | HTTP 403 |
| Deluge | [widgets/deluge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/deluge/proxy.js) | HTTP 403（JSON-RPC） |
| Homebridge | [widgets/homebridge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js) | HTTP 401/403 |
| FreshRSS | [widgets/freshrss/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/freshrss/proxy.js) | HTTP 401 |
| Flood | [widgets/flood/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/flood/proxy.js) | HTTP 401 |
| Crowdsec | [widgets/crowdsec/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/crowdsec/proxy.js) | HTTP 401 |
| Jellyfin | [widgets/jellyfin/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/jellyfin/proxy.js) | 无状态（MediaBrowser Token） |
| Transmission | [widgets/transmission/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/transmission/proxy.js) | HTTP 409（CSRF） |
| Omada | [widgets/omada/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/omada/proxy.js) | 401/403 + errorCode |
| NPM | [widgets/npm/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/npm/proxy.js) | HTTP 403 |
| Beszel | [widgets/beszel/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/beszel/proxy.js) | 400/403 + 空数组 |
| Dispatcharr | [widgets/dispatcharr/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/dispatcharr/proxy.js) | 400/401/403 + 空数组 |
| OpenWRT | [widgets/openwrt/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openwrt/proxy.js) | error.code === -32002 |
| pyLoad | [widgets/pyload/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/pyload/proxy.js) | 401/403 + CSRF token |
| OpenMediaVault | [widgets/openmediavault/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/openmediavault/proxy.js) | HTTP 401 + authenticated 字段 |
| UniFi | [widgets/unifi/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/unifi/proxy.js) | 工厂函数 createUnifiProxyHandler |
| UniFi Drive | [widgets/unifi_drive/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/unifi_drive/proxy.js) | 工厂函数 createUnifiProxyHandler |
| Filebrowser | [widgets/filebrowser/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/filebrowser/proxy.js) | 每次都登录 |
