# Homepage 身份认证代理协议分析

## 一、整体架构概览

Homepage 是一个基于 Next.js 的自托管仪表盘应用，核心功能之一是通过**服务端代理（Server-side Proxy）**代替前端直接访问各个后端服务的 API。这一设计避免了在浏览器中暴露 API 密钥和凭据，同时由代理层统一处理认证头注入、登录态管理和访问控制。

### 请求流全景

```
浏览器前端
  │  useWidgetAPI → SWR → /api/services/proxy?group=X&service=Y&endpoint=Z
  ▼
Next.js Middleware (middleware.js)
  │  Host 校验
  ▼
API 路由处理器 (pages/api/services/proxy.js)
  │  解析 widget 类型 → 选择 proxyHandler
  ▼
Proxy Handler 层
  │  credentialed / generic / synology / jsonrpc / unifi / 自定义 widget proxy
  │  注入认证头 / 管理登录态 / Cookie Jar
  ▼
HTTP 底层 (utils/proxy/http.js)
  │  httpProxy() → follow-redirects → 目标服务
  ▼
目标后端服务 API
```

---

## 二、认证头传递机制

认证头（Auth Headers）的构建发生在 **Proxy Handler 层**，核心逻辑集中在 [credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js)。

### 2.1 认证头构建的三层合并

[credentialed.js#L33-L38](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L33-L38) 中，headers 按优先级从低到高合并：

```js
const headers = {
  "Content-Type": "application/json",
  ...(widgets[widget.type].headers ?? {}),   // 1. widget 类型默认头
  ...(widget.headers ?? {}),                  // 2. 用户配置的自定义头
  ...(req.extraHeaders ?? {}),               // 3. mapping 中定义的额外头
};
```

- **第一层**：widget 类型注册时的默认 headers（如 `withheaders` 类型会带 `X-Widget: 1`）
- **第二层**：用户在 `services.yaml` 中为某个服务配置的 `headers` 字段，可覆盖类型默认值
- **第三层**：`req.extraHeaders`，由 [proxy.js#L88-L90](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L88-L90) 在 endpoint mapping 中通过 `mapping.headers` 注入

### 2.2 按 widget 类型分发的认证头策略

[credentialed.js#L40-L140](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L40-L140) 中，根据 `widget.type` 选择不同的认证头格式：

| widget 类型 | 认证头格式 | 说明 |
|---|---|---|
| `stocks` (finnhub) | `X-Finnhub-Token: {key}` | 从 settings.providers 获取 |
| `coinmarketcap` | `X-CMC_PRO_API_KEY: {key}` | API Key 方式 |
| `gotify` | `X-gotify-Key: {key}` | API Key 方式 |
| `checkmk` | `Authorization: Bearer {username} {password}` | Bearer 格式 |
| `argocd`, `authentik`, `mealie`, `vikunja` 等 | `Authorization: Bearer {key}` | 通用 Bearer Token |
| `truenas` | `Bearer {key}` 或 `Basic {user:pass}` | 有 key 优先用 Bearer，否则 Basic Auth |
| `ntfy` | `Bearer {key}` 或 `Basic {user:pass}` | 同上 |
| `proxmox` | `PVEAPIToken={username}={password}` | Proxmox 专用 Token 格式 |
| `proxmoxbackupserver` | `PBSAPIToken={username}:{password}` | PBS 专用格式，同时删除 Content-Type |
| `autobrr`, `jellystat` | `X-API-Token: {key}` | 自定义 Header Key |
| `tubearchivist` | `Authorization: Token {key}` | Django 风格 Token |
| `miniflux` | `X-Auth-Token: {key}` | 自定义 Header Key |
| `nextcloud` | `NC-Token: {key}` 或 `Basic {user:pass}` | Nextcloud 专用 Token 或 Basic Auth |
| `paperlessngx` | `Token {key}` 或 `Basic {user:pass}` | Django Token 或 Basic Auth |
| `azuredevops` | `Basic {base64("$:{key}")}` | 用 `$` 作为用户名 |
| `glances` | `Basic {user:pass}` | 仅 Basic Auth |
| `gitlab` | `PRIVATE-TOKEN: {key}` | GitLab 专用 Header |
| `trilium` | `Authorization: {key}` | 直接传 key（无 Bearer 前缀） |
| `esphome` | `Basic {user:pass}` 或 `Cookie: authenticated={key}` | 有密码用 Basic，否则用 Cookie |
| `wgeasy` | `Basic {user:pass}` 或 `Authorization: {password}` | 有用户名用 Basic，否则直接传密码 |
| `speedtest` | `Bearer {key}`（v1 不需要） | 可选 Bearer |
| 其他（默认 fallback） | `X-API-Key: {key}` | 通用 API Key Header |

### 2.3 Basic Auth 辅助函数

[basicAuthHeader()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js#L11-L13) 将 `username:password` 编码为 Base64：

```js
function basicAuthHeader(widget) {
  return `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.4 generic handler 的认证头

[generic.js#L28-L36](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js#L28-L36) 使用更简单的策略：如果配置了 `username` 和 `password`，就自动使用 Basic Auth：

```js
if (widget.username && widget.password) {
  headers.Authorization = `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

### 2.5 Jellyfin 的特殊认证头

[jellyfin/proxy.js#L29-L35](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/jellyfin/proxy.js#L29-L35) 使用 Emby/Jellyfin 专用的 `MediaBrowser Token` 格式：

```js
const authHeader = `MediaBrowser Token="${encodeURIComponent(widget.key)}", Client="Homepage", Device="Homepage", DeviceId="${deviceId}", Version="1.0.0"`;
```

---

## 三、代理转发逻辑

### 3.1 API 路由入口

[pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js) 是所有代理请求的入口，处理流程：

1. **解析参数**：从 `req.query` 获取 `service`、`group`、`index`、`endpoint`
2. **获取 widget 配置**：调用 [getServiceWidget()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/service-helpers.js#L754-L761) 从 services.yaml / Docker labels / Kubernetes 资源中查找 widget 定义
3. **确定 widget 类型**：做少量类型映射（如 `calendar` → `ical`，`unifi_console` 特殊处理）
4. **查找 proxy handler**：优先使用 widget 自定义的 `proxyHandler`，否则 fallback 到 [genericProxyHandler](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js)
5. **endpoint mapping**：如果 widget 定义了 `mappings`，则将前端的 opaque endpoint 映射为实际 API endpoint

### 3.2 Endpoint Mapping 机制

[proxy.js#L36-L97](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L36-L97)：

```js
if (widget?.mappings) {
  const mapping = widget?.mappings?.[req.query.endpoint];
  const endpoint = mapping?.endpoint;
  const endpointProxy = mapping?.proxyHandler || serviceProxyHandler;
  // ...
  req.query.endpoint = endpoint;
  // 处理 segments（URL 路径参数）
  // 处理 query params（白名单过滤）
  // 注入 mapping 级别的额外 headers
  return await endpointProxy(req, res, map);
}
```

mapping 支持以下能力：
- **endpoint 映射**：前端使用语义化的 endpoint 名（如 `speed`），映射到真实 API 路径（如 `api/v2/transfer/info`）
- **segments**：URL 路径参数，会进行安全校验（禁止 `/`、`\`、`..`）
- **query params**：白名单过滤，只允许 mapping 中声明的参数传递
- **method 覆盖**：mapping 可指定 HTTP 方法
- **body 覆盖**：mapping 可指定请求体
- **额外 headers**：通过 `mapping.headers` 注入到 `req.extraHeaders`
- **自定义 proxyHandler**：mapping 可指定独立的代理处理器
- **map 函数**：对响应数据进行转换

### 3.3 allowedEndpoints 正则白名单

[proxy.js#L99-L103](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js#L99-L103)：

```js
if (widget.allowedEndpoints instanceof RegExp) {
  if (widget.allowedEndpoints.test(req.query.endpoint)) {
    return await serviceProxyHandler(req, res);
  }
}
```

没有 mapping 的请求会被 `allowedEndpoints` 正则校验，不匹配则返回 403。

### 3.4 HTTP 底层实现

[http.js#L252-L293](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L252-L293) 中 `httpProxy()` 是最终发请求的函数：

- 使用 `follow-redirects` 库处理 HTTP/HTTPS 重定向
- 自定义 DNS 解析（Alpine/musl 兼容）：先尝试 `dns.lookup`，失败后 fallback 到 `dns.resolve`
- Agent 缓存：`keepAlive: true`，支持 IPv6 禁用选项
- SSL 证书验证：`rejectUnauthorized: false`（自签名证书兼容）
- 返回值格式：`[statusCode, contentType, data, responseHeaders, params]`

### 3.5 Cookie Jar 机制

[cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js) 使用 `tough-cookie` 库实现 Cookie 自动管理：

- **setCookieHeader()**：在请求发出前，从 Cookie Jar 中查找匹配 URL 的 Cookie 并注入到请求头
- **addCookieToJar()**：收到响应后，将 `Set-Cookie` 头存入 Cookie Jar，设置 1 小时 Max-Age
- **beforeRedirect 钩子**：[http.js#L19-L22](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js#L19-L22) 在重定向过程中自动处理 Cookie

这使得需要 Session Cookie 的服务（如 UniFi、Synology）可以自动维持会话。

### 3.6 前端如何发起代理请求

[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/use-widget-api.js) 封装了前端调用：

```js
export default function useWidgetAPI(widget, ...options) {
  let url = formatProxyUrl(widget, ...options);
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

[formatProxyUrl()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js#L43-L49) 生成代理 URL：

```js
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  return `/api/services/proxy?${params.toString()}`;
}
```

前端不直接访问后端服务，而是通过 `/api/services/proxy` 统一入口。

---

## 四、登录态判断逻辑

登录态管理存在两种主要模式：**无状态（Stateless）**和**有状态会话（Stateful Session）**。

### 4.1 无状态模式 — credentialed handler

[credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js) 不做登录流程，每次请求都直接注入认证头（API Key / Bearer Token / Basic Auth）。后端服务根据认证头验证身份，无需维护会话。

### 4.2 有状态会话模式 — 各 widget 自定义 proxy

许多服务需要先登录获取 session，再携带 session 访问 API。通用模式如下：

```
1. 尝试请求 API
2. 如果返回 401/403 → 判定为未登录
3. 调用 login() 获取 session（token / cookie）
4. 将 session 缓存到 memory-cache
5. 携带 session 重新请求 API
6. 如果 session 过期 → 重新 login 并重试
```

#### 4.2.1 qBittorrent — Cookie 会话模式

[qbittorrent/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/qbittorrent/proxy.js)：

- 先无认证请求 API
- 收到 403 时触发 login（POST 表单到 `/api/v2/auth/login`）
- 登录成功后 Cookie 自动存入 Cookie Jar
- 重新请求 API 时 `httpProxy` 会自动携带 Cookie

#### 4.2.2 Homebridge — Bearer Token + 缓存模式

[homebridge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/homebridge/proxy.js)：

- 从 `memory-cache` 取 token，用 `Authorization: Bearer {token}` 请求
- 401/403 时重新 login，获取 `access_token` 和 `expires_in`
- Token 缓存时间 = `expires_in - 5分钟`（提前刷新）
- 登录重试一次

#### 4.2.3 FreshRSS — GoogleLogin Auth 模式

[freshrss/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/freshrss/proxy.js)：

- 使用 `Authorization: GoogleLogin auth={token}` 格式
- 401 时重新登录（POST 到 `accounts/ClientLogin`），从响应中解析 `Auth=` 行提取 token
- Token 缓存到 memory-cache

#### 4.2.4 Synology — SID 会话模式

[synology.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js)：

- 先请求 API，检查 `json.success !== true`
- 登录流程：先查询 `SYNO.API.Info` 获取认证 API 路径和版本，再构造登录 URL
- 登录成功后 Synology 服务端维护 SID（通过 Cookie），后续请求自动携带
- API Info 信息也做了缓存

#### 4.2.5 UniFi — Cookie + CSRF Token 模式

[unifi.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/unifi.js)：

- 使用工厂函数 `createUnifiProxyHandler()` 创建处理器
- 401 时触发 login（POST JSON 到 `auth/login`）
- 支持 CSRF Token：从 `X-CSRF-TOKEN` 响应头获取，登录时回传
- 登录成功后将 Cookie 存入 Cookie Jar，后续请求自动携带
- 支持配置 `shouldAttemptLogin` 条件（如有 key 则不登录）

#### 4.2.6 Deluge — JSON-RPC 会话模式

[deluge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/deluge/proxy.js)：

- 使用 JSON-RPC 协议（通过 [jsonrpc.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/jsonrpc.js) 发送请求）
- 403 时调用 `auth.login` RPC 方法登录
- 登录后 Cookie 自动维护

#### 4.2.7 Omada — Token + Cookie 双重缓存模式

[omada/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/widgets/omada/proxy.js)：

- 登录后将 `token` 和 `cookieHeader` 同时缓存
- 缓存 55 分钟（Token 通常 1 小时有效）
- 后续请求同时携带 `Authorization` 头和 Cookie

#### 4.2.8 通用 Token 缓存模式（NPM、Beszel、Dispatcharr 等）

这类服务的 proxy 遵循统一模式：

```js
async function login(loginUrl, username, password, service) {
  const authResponse = await httpProxy(loginUrl, { method: "POST", ... });
  const data = JSON.parse(authResponse[2]);
  if (status === 200) {
    cache.put(`${tokenCacheKey}.${service}`, data.token, expiration);
  }
  return [status, data.token];
}
```

- 登录获取 token → 缓存到 `memory-cache` → 后续请求用 `Authorization: Bearer {token}`
- Token 过期时（401/403）→ 重新登录获取新 token

### 4.3 登录态判断总结

| 判断方式 | 代表服务 | 机制 |
|---|---|---|
| HTTP 状态码 403 | qBittorrent, Deluge | 收到 403 即触发登录 |
| HTTP 状态码 401 | Homebridge, FreshRSS, UniFi | 收到 401 即触发登录 |
| 业务层 success 字段 | Synology | `json.success !== true` 判定失败 |
| 业务层 error code | OpenWRT | `json.error.code === -32002` 判定未授权 |
| 无需登录态 | 多数 API Key 类服务 | 每次请求自带认证头 |

---

## 五、服务访问控制

### 5.1 Middleware 层 — Host 校验

[middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js) 是第一道防线，只对 `/api/:path*` 路由生效：

```
1. 获取请求的 Host 头
2. 构建允许列表：localhost + 127.0.0.1 + [::1] + HOMEPAGE_ALLOWED_HOSTS 环境变量
3. 如果 HOMEPAGE_ALLOWED_HOSTS === "*"，允许所有 Host
4. 否则，Host 不在允许列表中 → 返回 400
```

这防止了 DNS 重绑定攻击和未授权的跨域 API 调用。

### 5.2 API 路由层 — Widget 存在性校验

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js) 中多层校验：

1. **group 和 service 必须存在**，否则 400
2. **widget 必须存在**：[getServiceWidget()](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/service-helpers.js#L754-L761) 从配置中查找，找不到返回 false → 400
3. **widget 类型必须注册**：`widgets[type]` 不存在 → 403
4. **widget 必须有 API 定义**：`widgets[widget.type].api` 不存在 → 403
5. **endpoint 必须有映射或被允许**：无 mapping 且不匹配 `allowedEndpoints` → 403
6. **HTTP 方法必须匹配**：mapping 指定了 method 但请求方法不一致 → 403
7. **segment 值安全校验**：禁止 `/`、`\`、`..` → 403

### 5.3 Proxy Handler 层 — 凭据校验

[credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js) 和 [generic.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js) 中：

- Widget 不存在 → 400
- Widget 类型不支持 API → 403

### 5.4 响应数据校验

[validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js)：

- 检查响应是否能解析为 JSON
- 检查 mapping 中 `validate` 字段声明的必需 key 是否存在
- `allowEmpty` 配置允许空 Buffer 响应
- 校验失败返回 500

### 5.5 敏感信息脱敏

[api-helpers.js#L71-L78](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js#L71-L78) 中 `sanitizeErrorURL()` 在返回错误信息给前端前，将 URL 中的敏感参数（`apikey`, `api_key`, `token`, `t`, `access_token`, `auth`）替换为 `***`：

```js
["apikey", "api_key", "token", "t", "access_token", "auth"].forEach((key) => {
  if (url.searchParams.has(key)) url.searchParams.set(key, "***");
});
```

### 5.6 访问控制流程图

```
请求进入 /api/services/proxy
  │
  ├─ middleware.js: Host 校验
  │    └─ 失败 → 400 "Host validation failed"
  │
  ├─ proxy.js: 解析 group/service
  │    └─ 缺失 → 403 "Unknown proxy service type"
  │
  ├─ getServiceWidget(): 查找 widget 配置
  │    └─ 不存在 → 400/403
  │
  ├─ widgets[type] 存在性检查
  │    └─ 不存在 → 403 "Unknown proxy service type"
  │
  ├─ widget.api 存在性检查
  │    └─ 不存在 → 403 "Service does not support API calls"
  │
  ├─ endpoint mapping / allowedEndpoints 校验
  │    ├─ 无 mapping 且不匹配 → 403 "Unmapped proxy request"
  │    ├─ 方法不匹配 → 403 "Unsupported method"
  │    └─ segment 不合法 → 403 "Unsupported segment"
  │
  ├─ Proxy Handler: 注入认证头 → httpProxy() → 目标服务
  │    ├─ 401/403 → 尝试登录重试
  │    └─ 其他错误 → 返回错误（URL 脱敏）
  │
  └─ validateWidgetData(): 响应数据校验
       └─ 无效 → 500 "Invalid data"
```

---

## 六、核心文件索引

| 文件 | 职责 |
|---|---|
| [middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/middleware.js) | Next.js 中间件，Host 白名单校验 |
| [pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/pages/api/services/proxy.js) | 代理 API 入口路由，endpoint 映射和分发 |
| [utils/proxy/handlers/credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/credentialed.js) | 凭据型代理处理器，认证头构建和注入 |
| [utils/proxy/handlers/generic.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/generic.js) | 通用代理处理器，Basic Auth |
| [utils/proxy/handlers/synology.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/synology.js) | Synology 专用代理，SID 会话管理 |
| [utils/proxy/handlers/jsonrpc.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/jsonrpc.js) | JSON-RPC 代理处理器，Basic/Bearer Auth |
| [utils/proxy/handlers/unifi.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/handlers/unifi.js) | UniFi 工厂函数，Cookie + CSRF Token |
| [utils/proxy/http.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/http.js) | HTTP 底层请求，DNS 回退，Agent 缓存，gzip 解压 |
| [utils/proxy/cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/cookie-jar.js) | Cookie 自动管理，tough-cookie |
| [utils/proxy/api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/api-helpers.js) | URL 格式化，代理 URL 生成，敏感信息脱敏 |
| [utils/proxy/validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/validate-widget-data.js) | 响应数据结构校验 |
| [utils/proxy/use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/proxy/use-widget-api.js) | 前端 SWR Hook，发起代理请求 |
| [utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/service-helpers.js) | 服务配置加载和 widget 查找 |
| [utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/208-homepage/src/utils/config/config.js) | 配置文件管理，环境变量替换 |
