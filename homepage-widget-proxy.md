# Homepage Widget 通用代理层代码梳理

## 一、整体架构概览

Widget 代理层采用分层设计，从前端请求到后端目标服务 API 调用共经过 6 层：

```
┌──────────────────────────────────────────────────────────┐
│  1. 前端调用层 (React Hook)                               │
│     use-widget-api.js + SWR                              │
├──────────────────────────────────────────────────────────┤
│  2. API 入口层 (Next.js Route)                            │
│     pages/api/services/proxy.js                          │
├──────────────────────────────────────────────────────────┤
│  3. Widget 注册层 (配置中心)                               │
│     widgets/widgets.js + 各 widget/widget.js              │
├──────────────────────────────────────────────────────────┤
│  4. Handler 层 (通用处理器)                                │
│     utils/proxy/handlers/{generic,credentialed,...}.js   │
├──────────────────────────────────────────────────────────┤
│  5. 通用工具层 (HTTP/认证/校验)                            │
│     utils/proxy/{http,cookie-jar,validate-widget-data,   │
│                  api-helpers}.js                          │
├──────────────────────────────────────────────────────────┤
│  6. 具体 Widget 代理层 (特殊逻辑)                           │
│     widgets/{xxx}/proxy.js                                │
└──────────────────────────────────────────────────────────┘
```

---

## 二、各层详解

### 1. 前端调用层

核心文件：[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/use-widget-api.js)

这是一个基于 SWR 的 React Hook，供前端 Widget 组件调用：

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  // ...
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

调用时通过 `formatProxyUrl` 构建后端代理 URL，格式为：
```
/api/services/proxy?group=<group>&service=<name>&index=<idx>&endpoint=<endpoint>&query=<json>
```

辅助函数位于 [api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/api-helpers.js#L31-L49)：
- `getURLSearchParams`: 组装查询参数
- `formatProxyUrl`: 生成完整代理 URL
- `formatApiCall`: 模板字符串替换（将 `{url}`、`{key}`、`{endpoint}` 等占位符替换为实际值）

---

### 2. API 入口层

核心文件：[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/pages/api/services/proxy.js)

这是所有 Widget 代理请求的统一入口（Next.js API Route），核心流程如下：

#### 步骤 1：解析请求并定位 Widget 配置
```javascript
const { service, group, index } = req.query;
const serviceWidget = await getServiceWidget(group, service, index);
let type = serviceWidget?.type;
// 类型别名处理（calendar → ical，unifi_console → unifi 等）
const widget = widgets[type];
```

`getServiceWidget` 定义在 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/config/service-helpers.js#L754-L761)，会依次从 `services.yaml`、Docker 标签、Kubernetes Ingress 三个来源查找服务配置。

#### 步骤 2：选择 Proxy Handler
```javascript
const serviceProxyHandler = widget.proxyHandler || genericProxyHandler;
```

优先级：Widget 自定义 `proxyHandler` > 通用 `genericProxyHandler`

#### 步骤 3：处理 Endpoint Mapping（端点映射）
这是入口层最核心的逻辑，将前端传入的**不透明 endpoint 名称**映射为真实 API 路径：

```javascript
if (widget?.mappings) {
  const mapping = widget?.mappings?.[req.query.endpoint];
  // mapping 结构示例：
  // {
  //   endpoint: "queue/details",   // 真实 API 路径
  //   method: "GET",                // HTTP 方法
  //   params: ["start", "end"],     // 必需查询参数白名单
  //   optionalParams: ["unmonitored"], // 可选查询参数白名单
  //   segments: ["id"],             // URL 路径段（替换 {id}）
  //   headers: { ... },             // 额外请求头
  //   body: { ... },                // 固定请求体
  //   validate: ["totalRecords"],   // 响应数据字段校验
  //   allowEmpty: true,             // 允许空响应
  //   proxyHandler: xxxHandler,     // 此 endpoint 专用的 handler
  //   map: (data) => ...            // 响应数据转换函数
  // }
}
```

**安全校验**：
- `method`: 校验 HTTP 方法是否匹配
- `segments`: 校验路径段 key 是否在白名单内，且值不包含 `/`、`\`、`..`（防止路径遍历）
- `params/optionalParams`: 查询参数白名单过滤

处理完成后调用：
```javascript
return await endpointProxy(req, res, map);  // endpoint 专用 handler
// 或
return await serviceProxyHandler(req, res, map);  // widget 级 handler
```

#### 步骤 4：正则白名单（备选方案）
如果 Widget 没有定义 `mappings`，也可以用正则匹配：
```javascript
if (widget.allowedEndpoints instanceof RegExp) {
  if (widget.allowedEndpoints.test(req.query.endpoint)) {
    return await serviceProxyHandler(req, res);
  }
}
```

---

### 3. Widget 注册层

核心文件：[widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/widgets.js)

这是一个集中注册文件，导入了所有 Widget 的配置并导出为字典。每个 Widget 目录下必须有 `widget.js`，典型配置如下：

#### 类型 A：使用通用 Handler（最简洁）
以 [sonarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/sonarr/widget.js) 为例：
```javascript
import genericProxyHandler from "utils/proxy/handlers/generic";

const widget = {
  api: "{url}/api/v3/{endpoint}?apikey={key}",
  proxyHandler: genericProxyHandler,
  mappings: {
    series: { endpoint: "series", map: (data) => asJson(data).map(...) },
    queue: { endpoint: "queue", validate: ["totalRecords"] },
    calendar: { endpoint: "calendar", params: ["start", "end"] },
  },
};
export default widget;
```

#### 类型 B：使用带认证的通用 Handler
很多 Widget 使用 `credentialedProxyHandler`，它内置了 30+ 种服务的认证头生成逻辑。

#### 类型 C：完全自定义 Handler
以 [plex/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/plex/widget.js) 为例：
```javascript
import plexProxyHandler from "./proxy";
const widget = {
  api: "{url}{endpoint}?X-Plex-Token={key}",
  proxyHandler: plexProxyHandler,
  mappings: { unified: { endpoint: "/" } },
};
```

**别名机制**：`widgets.js` 中支持类型别名，如：
```javascript
ical: calendar,            // calendar 类型也叫 ical
jellyseerr: seerr,         // jellyseerr 复用 seerr 的配置
unifi_console: unifi,      // unifi_console 复用 unifi 的配置
```

---

### 4. Handler 层（通用处理器）

位于 `src/utils/proxy/handlers/`，共 5 种：

#### 4.1 genericProxyHandler —— 基础通用代理

核心文件：[generic.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/generic.js)

处理流程：
1. 获取 Widget 配置
2. 用 `formatApiCall` 将 `{url}`、`{endpoint}`、`{key}` 等占位符替换为实际值，构建目标 URL
3. 组装请求头（三层合并：widget 定义 headers → 用户配置 headers → mapping 额外 headers）
4. 若配置了 `username/password`，自动添加 Basic Auth
5. 调用 `httpProxy` 发起 HTTP 请求
6. 若有 `map` 函数，对响应数据进行转换
7. 用 `validateWidgetData` 校验响应数据结构
8. 返回结果或错误

#### 4.2 credentialedProxyHandler —— 带认证的通用代理

核心文件：[credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/credentialed.js)

在 generic 的基础上，增加了一个巨大的 `if/else if` 链，根据 `widget.type` 生成不同的认证头：

| 认证方式 | 适用 Widget 示例 |
|---------|----------------|
| `Bearer {key}` | argocd, authentik, tailscale, firefly 等 ~20 种 |
| `X-API-Key: {key}` | 默认兜底（如 sabnzbd, radarr 等） |
| `X-API-Token: {key}` | autobrr, jellystat |
| `Token {key}` | tubearchivist, paperlessngx |
| `X-Auth-Token: {key}` | miniflux |
| `PRIVATE-TOKEN: {key}` | gitlab |
| `Basic auth` | truenas (无 key 时), nextcloud (无 key 时), glances 等 |
| `PVEAPIToken={user}={pass}` | proxmox |
| `PBSAPIToken={user}:{pass}` | proxmoxbackupserver |
| `X-CMC_PRO_API_KEY` | coinmarketcap |
| `X-Finnhub-Token` | stocks (finnhub provider) |

这种设计的核心思想是：**认证逻辑集中管理**，新增 Widget 时只需在这个链上增加一行即可，无需写独立的 proxy.js。

#### 4.3 jsonrpcProxyHandler —— JSON-RPC 协议代理

核心文件：[jsonrpc.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/jsonrpc.js)

为使用 JSON-RPC 2.0 协议的服务（如 Deluge、Kodi 等）设计：
- 使用 `json-rpc-2.0` 库构建请求
- endpoint 参数被解释为 JSON-RPC 的 `method`
- mapping 中的 `params` 作为 JSON-RPC 参数
- 导出 `sendJsonRpcRequest` 供自定义 proxy.js 复用（如 deluge/proxy.js）

#### 4.4 synologyProxyHandler —— Synology DSM 专用

核心文件：[synology.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/synology.js)

Synology 的 API 有特殊的认证流程：
1. 先调用 `SYNO.API.Info` 查询各个 API 的 `cgiPath` 和 `maxVersion`（结果缓存）
2. 调用 `SYNO.API.Auth` 登录获取 Session Cookie
3. 使用 Cookie 调用实际 API
4. 自动处理常见错误码（Session 超时、权限不足等）

适用于：DiskStation、DownloadStation 等群晖服务。

#### 4.5 unifi.js —— UniFi 系列通用处理器工厂

核心文件：[unifi.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/unifi.js)

这是一个**高阶函数（工厂模式）**，接收配置参数后返回真正的 handler：

```javascript
export default function createUnifiProxyHandler({
  proxyName,              // 日志名称
  resolveWidget,          // 自定义 Widget 解析函数
  resolveRequestContext,  // 解析请求上下文（prefix、headers、csrfToken）
  getLoginEndpoint,       // 返回登录端点路径
  shouldAttemptLogin,     // 判断是否需要登录
})
```

核心逻辑：
1. 首次请求 → 判断是否已有有效 Cookie/Session
2. 若返回 401 → 尝试登录（POST 用户名密码）
3. 登录成功 → 保存 Cookie 到 CookieJar
4. 携带 Cookie 重试原始请求

UniFi 系列的复杂性在于：不同硬件型号（UDMP vs 普通 Cloud Key）API 前缀不同（`/proxy/network` vs `/`），且需要处理 CSRF Token。

---

### 5. 通用工具层

#### 5.1 http.js —— 底层 HTTP 客户端

核心文件：[http.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/http.js)

功能：
- 基于 `follow-redirects` 库，自动跟踪重定向
- 自动解压 gzip/deflate 响应
- **自定义 DNS 解析**：先尝试系统 `dns.lookup`，若失败则 fallback 到 `dns.resolve4/6`（解决 Alpine/musl 在 k8s 中的 DNS 问题）
- HTTP/HTTPS Agent 缓存 + keepAlive
- 可选内存缓存：`cachedRequest(url, durationMinutes)`
- 统一错误格式包装：`{ error: { message, url, rawError } }`，URL 中的敏感参数会被脱敏

返回值约定：
```javascript
[statusCode, contentType, dataBuffer, responseHeaders]
```

#### 5.2 cookie-jar.js —— Cookie 管理

核心文件：[cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/cookie-jar.js)

基于 `tough-cookie` 的全局单例 CookieJar：
- `setCookieHeader`: 请求前自动从 Jar 中取出对应 URL 的 Cookie 添加到请求头
- `addCookieToJar`: 响应后保存 Set-Cookie 到 Jar（默认有效期 1 小时）
- 重定向时也会自动处理 Cookie 更新

#### 5.3 validate-widget-data.js —— 响应校验

核心文件：[validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/validate-widget-data.js)

根据 mapping 中的 `validate` 字段，校验响应 JSON 中是否包含指定的 key。不包含则记录错误日志并返回 `false`。

#### 5.4 api-helpers.js —— 杂项辅助

- `formatApiCall(url, args)`: 模板替换，支持 `{key}` 占位符
- `asJson(data)`: Buffer → JSON 解析
- `sanitizeErrorURL(url)`: URL 脱敏（隐藏 apikey、token 等参数）
- `parseVersionForUrl`: API 版本号安全解析

---

### 6. 具体 Widget 代理层（自定义 proxy.js）

当通用 Handler 无法满足需求时（需要多 API 聚合、特殊认证、协议转换等），Widget 可以编写自己的 `proxy.js`。以下是几种典型模式：

#### 模式 A：多 API 聚合并缓存

**代表**：[plex/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/plex/proxy.js)

Plex 需要：
1. 调 `/status/sessions` 获取当前播放流数量
2. 调 `/library/sections` 获取媒体库列表（缓存 6 小时）
3. 对每个电影/剧集/音乐库分别调 `/library/sections/{id}/all` 获取条目数（缓存 10 分钟）
4. 最后将所有数据聚合成一个 JSON 返回

使用 `memory-cache` 做内存缓存，避免每次请求都打满 Plex API。

#### 模式 B：版本分支 + 协议转换

**代表**：[pihole/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/pihole/proxy.js)

Pi-hole v5 和 v6 API 完全不同：
- v5: `{url}/admin/api.php?summaryRaw&auth={key}`（直接 query 参数传 token）
- v6: 先 POST `/api/auth` 登录获取 session sid，再用 `X-FTL-SID` 请求头调 `/api/stats/summary`

proxy.js 中根据 `widget.version` 走不同分支，并将两种响应统一转换为相同格式返回给前端。

#### 模式 C：登录态管理（401 自动重试）

**代表**：[freshrss/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/freshrss/proxy.js)

流程：
1. 检查缓存中是否有 session token
2. 没有 → 调 `accounts/ClientLogin` 获取 token（Google Account Login 协议）
3. 携带 token 调业务 API
4. 若返回 401 → 重新登录 → 重试请求
5. 聚合多个 API 结果（subscription list + unread count）

#### 模式 D：JSON-RPC + 认证重试

**代表**：[deluge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/deluge/proxy.js)

1. 用 JSON-RPC 调 `web.update_ui` 获取种子数据
2. 若返回错误码 1（未认证）→ 调 `auth.login` 登录
3. 重试业务请求

#### 模式 E：集成 URL 直连（无后端 API）

**代表**：[calendar/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/calendar/proxy.js)

Calendar Widget 不调用后端服务 API，而是直接拉取用户配置的 iCal URL。代理层只做 URL 获取和转发，Outlook 等特殊来源额外加 User-Agent。

---

## 三、关键设计模式总结

### 1. 模板方法 + 策略模式
- `genericProxyHandler` 定义了代理请求的骨架流程
- 各个 `proxyHandler`（credentialed、jsonrpc、synology 等）是不同的策略实现
- Widget 可通过 `widget.proxyHandler` 自由选择策略

### 2. 工厂模式
`createUnifiProxyHandler` 是典型的工厂函数，接收配置参数生成定制化的 handler，避免了 UniFi/UniFi Drive/UDMP 之间的代码重复。

### 3. 配置驱动
通过 `mappings` 声明式定义端点，而不是为每个 endpoint 写代码。90% 的场景只需在 `widget.js` 中加几行配置即可。

### 4. 渐进式复杂度
- **最简单**：`widget.js` 配 `api` + `proxyHandler: genericProxyHandler` + `mappings`
- **中等**：换用 `credentialedProxyHandler` 获得认证支持
- **复杂**：写自定义 `proxy.js`，在其中复用 `httpProxy`、`sendJsonRpcRequest` 等底层工具

### 5. 安全机制
- Endpoint 白名单（mapping 或正则），防止任意 URL 代理
- Segments 路径遍历防护（禁止 `/`、`\`、`..`）
- Query 参数白名单过滤
- 错误信息中的 URL 自动脱敏（隐藏 apikey/token）

---

## 四、请求链路完整示例

以 Sonarr 的 `queue` 端点为例，完整调用链路：

```
前端 SonarrWidget 组件
  ↓ useWidgetAPI(widget, "queue")
  ↓ formatProxyUrl → /api/services/proxy?group=...&service=sonarr&index=0&endpoint=queue
  ↓ HTTP GET
pages/api/services/proxy.js (入口)
  ↓ getServiceWidget → 找到服务配置
  ↓ widgets["sonarr"] → 定位 widget.js 配置
  ↓ widget.proxyHandler → genericProxyHandler
  ↓ mappings["queue"] → { endpoint: "queue", validate: ["totalRecords"] }
  ↓ 调用 genericProxyHandler(req, res)
genericProxyHandler
  ↓ formatApiCall("{url}/api/v3/{endpoint}?apikey={key}", ...)
  ↓ 构建真实 URL: https://sonarr.example.com/api/v3/queue?apikey=xxx
  ↓ httpProxy(url, { method: "GET", headers: {} })
http.js
  ↓ follow-redirects 发起 HTTPS 请求
  ↓ 自动处理 gzip 解压
  ↓ 返回 [200, "application/json", <Buffer>]
  ↓
validateWidgetData → 检查响应中是否有 totalRecords 字段
  ↓
res.status(200).send(data) → 返回前端
  ↓
SWR 缓存数据 → 组件渲染
```
