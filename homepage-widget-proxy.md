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
    config.refreshInterval = options[1]?.refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  if (options[0] === "") {
    url = null;
  }
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

调用时通过 `formatProxyUrl` 构建后端代理 URL，格式为：
```
/api/services/proxy?group=<group>&service=<name>&index=<idx>&endpoint=<endpoint>&query=<json>
```

辅助函数位于 [api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/api-helpers.js#L31-L49)：
- `getURLSearchParams`: 组装查询参数（group, service, index, endpoint）
- `formatProxyUrl`: 生成完整代理 URL，query 参数会被 JSON 序列化
- `formatApiCall`: 模板字符串替换（将 `{url}`、`{key}`、`{endpoint}` 等占位符替换为实际值）

---

### 2. API 入口层

核心文件：[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/pages/api/services/proxy.js)

这是所有 Widget 代理请求的统一入口（Next.js API Route），**核心决策分支**如下：

#### 步骤 1：解析请求并定位 Widget 配置
```javascript
const { service, group, index } = req.query;
const serviceWidget = await getServiceWidget(group, service, index);
let type = serviceWidget?.type;

// 特例：类型别名改写
if (type === "calendar") type = "ical";
else if (service === "unifi_console" && group === "unifi_console") type = "unifi_console";

const widget = widgets[type];
```

`getServiceWidget` 定义在 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/config/service-helpers.js#L754-L761)，会依次从 `services.yaml` → Docker 标签 → Kubernetes Ingress 三个来源查找服务配置。

#### 步骤 2：选择 Proxy Handler
```javascript
const serviceProxyHandler = widget.proxyHandler || genericProxyHandler;
```

优先级：Widget 自定义 `proxyHandler` > 通用 `genericProxyHandler`

#### 步骤 3：**关键分叉点**：无 endpoint 快速委托 + calendar 特例

```javascript
if (serviceProxyHandler instanceof Function) {
  // ★ 快速返回分支：跳过 mappings 处理，直接交给 handler
  // 条件一：请求没有 endpoint 参数（完全自定义的代理，如 pihole、deluge）
  // 条件二：handler 是 calendarProxyHandler（calendar 即使有 endpoint，也跳过 mappings，因为它的 endpoint 是 integration name）
  if (!req.query.endpoint || serviceProxyHandler === calendarProxyHandler) {
    return await serviceProxyHandler(req, res);  // ← 注意：不传 map 参数
  }
  // ...否则继续走 mappings 映射逻辑
}
```

**⚠️ 这是容易读错的关键点**——**自定义 proxyHandler 不一定都走快速分支**：

| 分支条件 | 场景示例 | 后续行为 |
|---------|---------|---------|
| `!req.query.endpoint` | Pi-hole、Deluge（完全无 mappings，前端不传 endpoint） | **跳过所有 mappings 处理**，handler 自己负责全部逻辑，**map 参数为 undefined** |
| `serviceProxyHandler === calendarProxyHandler` | Calendar（endpoint 是集成名称而非 API 路径） | 即使请求带了 endpoint 参数，**也跳过 mappings**，直接委托给 calendarProxyHandler 内部用 endpoint 去匹配 integrations 数组 |
| 其他情况（有 endpoint 且非 calendar） | **TrueNAS**、Plex、FreshRSS、Sonarr、Radarr 等 | 继续执行下面的 endpoint 映射逻辑，**传递 map 参数** |

calendar proxy handler 在 [calendar/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/calendar/proxy.js#L7-L40) 内部用 `req.query.endpoint` 去匹配 `widget.integrations` 数组中的 `name` 字段，这就是为什么它需要特例化（mapping 的 endpoint 是真实路径，而 calendar 的 endpoint 只是集成项名称）。

**⚠️ 自定义 proxyHandler 的路径选择**：
- **有 endpoint + 非 calendar** → 走 mapping 分支，**传 map 参数**（TrueNAS、Plex、FreshRSS 属于这一类）
- **无 endpoint 或 calendar** → 走快速分支，**不传 map 参数**（Deluge、Pi-hole、Calendar 属于这一类）

#### 步骤 4：处理 Endpoint Mapping（端点映射）

这是入口层最核心的逻辑，将前端传入的**不透明 endpoint 名称**映射为真实 API 路径，**同时改写 req 对象的多个字段**：

```javascript
if (widget?.mappings) {
  const mapping = widget?.mappings?.[req.query.endpoint];
  const mappingParams = mapping?.params;
  const optionalParams = mapping?.optionalParams;
  const map = mapping?.map;                    // 提取响应数据转换函数
  const endpoint = mapping?.endpoint;          // 真实 API 路径
  const endpointProxy = mapping?.proxyHandler || serviceProxyHandler;
```

**⚠️ 请求方法校验**（不是改写，是准入校验）：
```javascript
if (mapping?.method && mapping.method !== req.method) {
  return res.status(403).json({ error: "Unsupported method" });
}
```
校验失败直接返回 403，不往下执行。

**⚠️ 请求方法与请求体的改写（直接修改 req 对象）**：
```javascript
// 方法改写：若 mapping 指定了 method 则覆盖，否则默认 GET
req.method = mapping?.method || "GET";

// 请求体改写：mapping.body 会完全替换 req.body（用于固定 POST body 的场景，如 JSON-RPC）
if (mapping?.body) req.body = mapping?.body;

// endpoint 路径改写：前端传入的逻辑名 → 真实 API 路径
req.query.endpoint = endpoint;
```

**⚠️ Segments 路径段注入（路径遍历防护后替换 endpoint 中的占位符）**：
```javascript
if (req.query.segments) {
  const segments = JSON.parse(req.query.segments);
  // 安全校验：
  // 1. segment key 必须在 mapping.segments 白名单内
  // 2. segment value 不得包含 /、\、..（防止路径遍历攻击）
  // 校验失败直接 403
  req.query.endpoint = formatApiCall(endpoint, segments); // ← 第二次改写 endpoint
}
```

**⚠️ 查询参数白名单过滤**：
```javascript
if (req.query.query && (mappingParams || optionalParams)) {
  const queryParams = JSON.parse(req.query.query);
  const filteredOptionalParams = optionalParams
    ? optionalParams.filter(p => queryParams[p] !== undefined)
    : [];
  const params = (mappingParams || []).concat(filteredOptionalParams);
  const query = new URLSearchParams(params.map(p => [p, queryParams[p]]));
  req.query.endpoint = `${req.query.endpoint}?${query}`; // ← 第三次改写 endpoint，追加 query string
}
```

**⚠️ 请求头注入（不是直接改写 req.headers，而是通过临时字段传递）**：
```javascript
if (mapping?.headers) {
  req.extraHeaders = mapping.headers; // ← 挂在 req 的自定义字段上，handler 里再合并
}
```

**最终委托调用（二选一）**：
```javascript
if (endpointProxy instanceof Function) {
  return await endpointProxy(req, res, map); // 1️⃣ endpoint 专用 handler + map
}
return await serviceProxyHandler(req, res, map); // 2️⃣ widget 级 handler + map
```
**注意**：与快速返回分支不同，这里**传递了 map 参数**。即使 `serviceProxyHandler` 是自定义的（如 TrueNAS、Plex、FreshRSS），也会收到 map。

#### 步骤 5：正则白名单（Mappings 的兜底方案）

如果 Widget 没有定义 `mappings`，也可以用正则匹配：
```javascript
if (widget.allowedEndpoints instanceof RegExp) {
  if (widget.allowedEndpoints.test(req.query.endpoint)) {
    return await serviceProxyHandler(req, res);  // ← 注意：不传 map 参数
  }
}
```
命中正则也会跳过 mappings，但同样不传递 map。

---

### 3. Widget 注册层

核心文件：[widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/widgets.js)

这是一个集中注册文件，导入了所有 Widget 的配置并导出为字典。每个 Widget 目录下必须有 `widget.js`。

**⚠️ 自定义 proxyHandler 按"入口分支路径"分为 5 大类**：

#### 类型 A：使用 generic Handler（最简洁，走 mapping 分支）
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

#### 类型 B：使用 credentialed Handler（带 30+ 种内置认证，走 mapping 分支）
与 A 结构相同，只是 `proxyHandler` 换成 `credentialedProxyHandler`。

#### 类型 C：**走 mapping 分支的半复用自定义 proxy —— TrueNAS**
**⚠️ 这是最特殊也最有代表性的自定义 proxy**，详见后文。[truenas/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/widget.js)：
```javascript
import truenasProxyHandler from "./proxy";
const widget = {
  api: "{url}/api/v2.0/{endpoint}",
  wsAPI: "{url}/api/current",
  proxyHandler: truenasProxyHandler,
  mappings: {
    alerts: {
      endpoint: "alert/list",
      wsMethod: "alert.list",
      map: (data) => {
        if (Array.isArray(data)) {
          return { pending: data.filter((item) => item?.dismissed === false).length };
        }
        return { pending: jsonArrayFilter(data, (item) => item?.dismissed === false).length };
      },
    },
    status: {
      endpoint: "system/info",
      wsMethod: "system.info",
      validate: ["loadavg", "uptime_seconds"],
    },
    pools: {
      endpoint: "pool",
      wsMethod: "pool.query",
      map: (data) => {
        const list = Array.isArray(data) ? data : asJson(data);
        return list.map((entry) => ({ id: entry.name, name: entry.name, healthy: entry.healthy }));
      },
    },
    dataset: {
      endpoint: "pool/dataset",
      wsMethod: "pool.dataset.query",
    },
  },
};
```
关键特征：
- `mappings` 中每个 endpoint 除了 `endpoint`（REST 路径）还有 `wsMethod`（WebSocket 方法名）
- `proxyHandler` 是自定义的，但**走 mapping 分支**（有 endpoint，非 calendar）
- 入口层会传 `map` 参数给它

#### 类型 D：名义上有 mappings 但实际忽略的自定义 proxy —— Plex、FreshRSS
以 [plex/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/plex/widget.js) 为例：
```javascript
import plexProxyHandler from "./proxy";
const widget = {
  api: "{url}{endpoint}?X-Plex-Token={key}",
  proxyHandler: plexProxyHandler,
  mappings: { unified: { endpoint: "/" } },
};
```
以 [freshrss/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/freshrss/widget.js) 为例：
```javascript
import freshrssProxyHandler from "./proxy";
const widget = {
  api: "{url}/api/greader.php/{endpoint}?output=json",
  proxyHandler: freshrssProxyHandler,
  mappings: { info: { endpoint: "/" } },
};
```
关键特征：
- 名义上有 `mappings` 和 `endpoint`，**走 mapping 分支**，入口层会传 map 参数
- 但函数签名是 `(req, res)`（不接收第三个参数，JS 会忽略多余参数）
- 内部完全不使用 `req.query.endpoint`（自己硬编码调用多个真实 API）
- `mappings` 仅用于通过入口层的合法性检查（否则会返回 403 "Unmapped proxy request"）

#### 类型 E：走快速分支的完全自定义 proxy —— Deluge、Pi-hole
以 [deluge/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/deluge/widget.js) 为例：
```javascript
import delugeProxyHandler from "./proxy";
const widget = {
  api: "{url}/json",
  proxyHandler: delugeProxyHandler,
  // 注意：无 mappings，也无 {endpoint} 占位符
};
```
以 [pihole/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/pihole/widget.js) 为例：
```javascript
import piholeProxyHandler from "./proxy";
const widget = {
  api: "{url}/api/{endpoint}",
  apiv5: "{url}/admin/api.php?{endpoint}&auth={key}",
  proxyHandler: piholeProxyHandler,
  // 注意：无 mappings
};
```
关键特征：
- 无 `mappings`（或 mappings 存在但前端不传 endpoint）
- 走 `!req.query.endpoint` 快速分支
- 函数签名 `(req, res)`，无 map 参数

#### 类型 F：走 calendar 特例分支 —— Calendar
[calendar/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/calendar/widget.js)：
```javascript
import calendarProxyHandler from "./proxy";
const widget = {
  api: "{url}",
  proxyHandler: calendarProxyHandler,
  // 注意：没有 mappings，endpoint 含义完全由 handler 自解释
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

**请求头三层合并顺序**（后者覆盖前者）：
```javascript
const headers = {
  ...(widgets[widget.type].headers ?? {}),    // 第 1 层：widget.js 中配置的默认 headers
  ...(widget.headers ?? {}),                  // 第 2 层：用户 services.yaml 中配置的 headers
  ...(req.extraHeaders ?? {}),                // 第 3 层：mapping.headers（入口层注入的）
};
```
`req.extraHeaders` 是入口层在处理 endpoint mapping 时写入的自定义临时字段，**不是标准的 req.headers**。

**Basic Auth 自动生成（额外追加，会覆盖前面的同名 Header）**：
```javascript
if (widget.username && widget.password) {
  headers.Authorization = `Basic ${Buffer.from(`${widget.username}:${widget.password}`).toString("base64")}`;
}
```

**请求体优先级（三选一）**：
```javascript
if (req.body) {                      // 优先级 1：入口层 mapping.body 改写后的值
  params.body = req.body;
} else if (widget.requestBody) {     // 优先级 2：用户配置的 requestBody
  if (typeof widget.requestBody === "object") {
    params.body = JSON.stringify(widget.requestBody);
  } else {
    params.body = widget.requestBody;
  }
}
// 优先级 3：无 body
```

**HTTP 方法来源**：
```javascript
method: widget.method ?? req.method  // 用户配置优先，其次是入口层改写过的 req.method
```

**⚠️ 返回校验与 map 转换的顺序（先 validate，后 map）**：
```javascript
if (status === 200) {
  // 第 1 步：validateWidgetData 校验原始响应（含 validate 字段的 mapping 才会触发）
  if (!validateWidgetData(widget, endpoint, resultData)) {
    return res.status(status).json({ error: { message: "Invalid data", ... } });
  }
  // 第 2 步：validate 通过后才执行 map 转换
  if (map) resultData = map(resultData);
}
```
**这是一个重要的隐含契约**：
- `validate` 校验的是**目标服务返回的原始数据结构**（JSON parse 后，但未 map 前）
- `map` 函数拿到的是**已经 validate 通过**的原始数据
- 如果顺序反过来（先 map 后 validate），validate 就得基于转换后的结构写，会非常反直觉

#### 4.2 credentialedProxyHandler —— 带认证的通用代理

核心文件：[credentialed.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/credentialed.js)

**与 generic 的差异**：
| 方面 | generic | credentialed |
|------|---------|--------------|
| 默认 Content-Type | 无 | 默认 `application/json` |
| 认证头 | 仅 Basic Auth（有 username/password 时） | 30+ 种类型的 if/else 链，按需生成 Bearer / X-API-Key / 自定义 Header 等 |
| 请求方法 | `widget.method ?? req.method` | 固定 `req.method`（入口层已改写） |
| HTTP 错误码返回 Invalid data | `status(status)`（原状态码） | 固定 `status(500)` |
| withCredentials 标志 | 无 | 有 `withCredentials: true`, `credentials: "include"` |

**认证方式速查表**（部分）：

| 认证方式 | 适用 Widget 示例 |
|---------|----------------|
| `Bearer {key}` | argocd, authentik, tailscale, firefly, netalertx 等 ~20 种 |
| `X-API-Key: {key}` | 默认兜底（sabnzbd, radarr, prowlarr 等 *Arr 家族） |
| `X-API-Token: {key}` | autobrr, jellystat |
| `Token {key}` | tubearchivist, paperlessngx |
| `X-Auth-Token: {key}` | miniflux |
| `PRIVATE-TOKEN: {key}` | gitlab |
| `Basic auth` | truenas (无 key 时，v1 REST 模式), nextcloud (无 key 时), glances, esphome 等 |
| `PVEAPIToken={user}={pass}` | proxmox |
| `PBSAPIToken={user}:{pass}` | proxmoxbackupserver |
| `X-CMC_PRO_API_KEY` | coinmarketcap |
| `X-Finnhub-Token` | stocks (finnhub provider) |

**⚠️ 返回校验与 map 顺序**：与 generic 完全一致，**先 validateWidgetData，后 map 转换**。

**⚠️ TrueNAS v1 直接复用**：`return credentialedProxyHandler(req, res, map)`，把入口层传递的 map 原样转发，同时利用 credentialed 内置的 Basic Auth（无 key 时）。

#### 4.3 jsonrpcProxyHandler —— JSON-RPC 协议代理（完整 Handler）

核心文件：[jsonrpc.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/jsonrpc.js#L66-L90)

**完整通用 handler**，通过 `proxyHandler: jsonrpcProxyHandler` 直接使用：
- 接收 `req.query.endpoint` 作为 JSON-RPC 的 **method** 名
- 从 `widget.mappings` 反向查找匹配的 mapping（`find` 其 `endpoint` 等于 method），取出 mapping 中声明的 `params` 作为 JSON-RPC 调用参数
- 通常由入口层先把逻辑 endpoint 映射成真实 JSON-RPC method，再把请求交给该 handler；API URL 直接用 `{url}`（不含 endpoint 占位符）
- 返回值：`res.status(status).end(data)`（`.end()` 而非 `.send()`，因为 data 已是 JSON 字符串）

#### 4.3.1 sendJsonRpcRequest —— JSON-RPC 底层工具函数（被 Deluge 等复用）

**⚠️ 这是容易混淆的两个概念**：jsonrpc 模块导出了**两个东西**：

| 导出 | 类型 | 用途 |
|------|------|------|
| `jsonrpcProxyHandler`（default export） | **完整 Handler** | 可直接作为 `widget.proxyHandler`，适用于"只需调一个 JSON-RPC method 且无需登录态"的简单场景 |
| `sendJsonRpcRequest`（named export） | **工具函数** | 供自定义 proxy.js 内部调用，适用于"需要多次调用、需要错误码判断、需要登录重试"等复杂场景（如 Deluge） |

`sendJsonRpcRequest` 函数签名：
```javascript
export async function sendJsonRpcRequest(url, method, params, widget)
// 返回: [statusCode, contentType, jsonStringifiedResponse]
```

内部处理细节：
- 自动添加 Basic Auth（若有 username/password）或 Bearer Auth（若有 key）
- 用 `json-rpc-2.0` 库构建标准 JSON-RPC 请求体
- 特殊容错：当响应中 `json.id === null` 时补 1；`json.error && json.result === null` 时将 result 置 undefined（配合库的错误识别机制）
- **JSON-RPC 业务错误不抛 HTTP 错误**：`JSONRPCErrorException` 会被捕获后包装成 `{ result: null, error: {...} }` 以 HTTP 200 返回，交由上层判断

#### 4.4 synologyProxyHandler —— Synology DSM 专用

核心文件：[synology.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/synology.js)

Synology API 的特殊流程：
1. `getApiInfo`: 调 `SYNO.API.Info` 查询每个 API 的 `cgiPath` 和 `maxVersion`（结果用 memory-cache 缓存，key 含 apiName 和 service 名）
2. 拼接真实 URL（含 apiName、apiMethod、cgiPath、maxVersion 占位符）
3. 首次请求
4. 若响应 `success !== true`（如 Session 过期）：
   - 调 `SYNO.API.Auth` 登录获取 Cookie（走全局 CookieJar）
   - 重新执行原请求
5. 若仍失败 → `toError` 将错误码翻译为可读信息（如 106=Session timeout），返回 500

适用于 DiskStation、DownloadStation 等群晖服务。**此 handler 不走 generic/credentialed 的 validate+map 流程**，因为响应格式是 Synology 自有的 `{ success, data, error }`。

#### 4.5 unifi.js —— UniFi 系列通用处理器工厂

核心文件：[unifi.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/handlers/unifi.js)

这是一个**高阶函数（工厂模式）**，不是直接使用的 handler。接收配置参数后返回真正的 handler：

```javascript
export default function createUnifiProxyHandler({
  proxyName,              // 日志名称（区分 unifi / unifi_drive 等）
  resolveWidget,          // 自定义 Widget 解析函数（如 unifi_console info widget 需要特殊处理）
  resolveRequestContext,  // 解析 prefix（UDMP 需 /proxy/network）、headers、csrfToken
  getLoginEndpoint,       // 返回登录端点路径（prefix 不同路径不同）
  shouldAttemptLogin,     // 判断是否需要登录（有 widget.key 时直接用 API Key，不登录）
})
```

核心逻辑：
1. 用 resolveWidget 获取配置 → resolveRequestContext 确定 prefix 和初始 headers
2. 构建 URL（含 `{prefix}` 占位符），调 `httpProxy`
3. 若返回 **401** 且 `shouldAttemptLogin` 为 true：
   - 从响应头取 `x-csrf-token`
   - POST `auth/login`（或 `login`）提交用户名密码+rememberMe
   - 成功后将 Cookie 写入全局 CookieJar
   - 携带新 Cookie 重试原始请求
4. 返回结果

**UniFi 系列的复杂性**：不同硬件型号（UDMP vs 普通 Cloud Key/UDM）API 前缀不同，UDMP 需要 `/proxy/network`，需要通过探测根路径的响应头（`x-csrf-token` 或 `access-control-expose-headers`）来自动识别。

---

### 5. 通用工具层

#### 5.1 http.js —— 底层 HTTP 客户端

核心文件：[http.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/http.js)

功能：
- 基于 `follow-redirects` 库，自动跟踪重定向，重定向过程中通过 `beforeRedirect` 钩子同步更新 CookieJar
- 自动解压 gzip/deflate 响应（zlib 出错时 fallback 到原始流）
- **自定义 DNS 解析**：先试系统 `dns.lookup`，ENOTFOUND/EAI_NONAME 时 fallback 到 `dns.resolve4`→`dns.resolve6`（解决 Alpine/musl 在 k8s 中的 DNS 问题）
- HTTP/HTTPS Agent 缓存（key 含协议和是否禁用 IPv6）+ keepAlive，lookup 指向自定义 DNS 函数
- 可选内存缓存：`cachedRequest(url, durationMinutes)`，缓存前自动尝试 JSON.parse Buffer
- 统一错误格式：`{ error: { message, url, rawError } }`，URL 中 apikey/token/access_token 等参数会被脱敏

**返回值约定**（元组，不是对象）：
```javascript
[statusCode, contentType, dataBuffer, responseHeaders, params]
```

#### 5.2 cookie-jar.js —— Cookie 管理

核心文件：[cookie-jar.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/cookie-jar.js)

基于 `tough-cookie` 的全局单例 CookieJar（模块级变量 `cookieJar`）：
- `setCookieHeader(url, params, { overwrite })`: 请求前从 Jar 取对应 URL 的 Cookie 加到 headers，默认 Header 名是 `Cookie`（可通过 `params.cookieHeader` 自定义）
- `addCookieToJar(url, headers)`: 响应后解析 `set-cookie` 头（数组或单值均支持），所有 Cookie 的 MaxAge 统一强制为 1 小时，`ignoreError: true` 吞掉非法 cookie 错误

#### 5.3 validate-widget-data.js —— 响应校验

核心文件：[validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/utils/proxy/validate-widget-data.js)

工作流程：
1. 通过 `widget.type` 和传入的 `endpoint`（**真实 API 路径**，不是逻辑名），反向查找 mapping 对象
   ```javascript
   // 注意：反向查找！用 endpoint 路径去匹配 mapping.endpoint
   const mappingEntry = Object.values(widgets[widget.type].mappings).find(
     (mapping) => mapping.endpoint === endpoint
   );
   ```
   **⚠️ TrueNAS 的 WebSocket 路径**调用它时传入的 `endpoint` 已经是真实路径（如 `system/info`），所以能正确匹配到 `mappings.status` 并拿到 `validate: ["loadavg", "uptime_seconds"]`。
2. 若 `mapping.allowEmpty === true` 且 data 是空 Buffer → 直接返回 true
3. Buffer → JSON parse（第一次直接 parse，失败则去空白后再 parse）
4. 遍历 `mapping.validate` 数组中的每个 key，若 JSON 中该 key 为 undefined 则标记 invalid
5. invalid 时 `logger.error` 详细日志（含期望字段、parse 错误、原始 data）

**⚠️ 缺少 validate 字段时的行为**：
```javascript
// validate-widget-data.js L33-37
if (dataParsed && Object.entries(dataParsed).length) {
  mapping?.validate?.forEach((key) => {   // ← 可选链 ?.forEach
    if (dataParsed[key] === undefined) {
      valid = false;
    }
  });
}
```
当 mapping 中没有 `validate` 字段时（如 TrueNAS 的 `alerts`、`pools`、`dataset`），`mapping?.validate` 为 `undefined`，`undefined?.forEach(...)` 是空操作，`valid` 保持 `true`，函数直接返回 `true`。

**也就是说：缺少 validate 字段 = 不做任何字段校验 = 始终通过**。这是"约定优于配置"的体现——只在你关心特定字段时才声明 validate，否则只检查数据可解析且非空。

对于 TrueNAS WebSocket 路径，`sendMethod` 返回的 data 已经是 JavaScript 对象（`waitForEvent` 中 `parseJson: true` 自动解析了），不是 Buffer。因此 `Buffer.isBuffer(data)` 为 false，跳过 JSON parse 步骤，直接进入 validate 检查——如果没有 validate 字段，直接通过。

**⚠️ 调用位置**：
- generic/credentialed：HTTP 200 时调用，validate 失败直接返回错误
- **TrueNAS WebSocket 路径**：自己显式调用，位置在 `sendMethod` 拿到 data 之后、`map` 之前（顺序与 generic 一致）

#### 5.4 api-helpers.js —— 杂项辅助

- `formatApiCall(url, args)`: 用正则 `/{.*?}/g` 两次遍历替换模板占位符，`{url}` 特殊处理去掉末尾 `/`
- `asJson(data)`: Buffer/string → JSON parse
- `jsonArrayTransform / jsonArrayFilter`: 数组专用的转换/过滤工具
- `sanitizeErrorURL(url)`: URL 脱敏（apikey、api_key、token、t、access_token、auth 替换为 `***`，search 和 hash 均处理）
- `parseVersionForUrl`: API 版本号安全解析（字符串整数 / 非负整数）
- `formatProxyUrl / getURLSearchParams`: 前端→后端代理 URL 构建

---

### 6. 具体 Widget 代理层（自定义 proxy.js）

当通用 Handler 无法满足需求时（需要多 API 聚合、特殊认证、协议转换、版本分流等），Widget 可以编写自己的 `proxy.js`。

**⚠️ 自定义 proxy.js 分类全景（按入口分支 + 对通用层的复用程度）**：

| 类别 | 代表 | 入口分支 | 函数签名 | 是否接收/使用 map | 是否复用通用层 | 核心特征 |
|------|------|---------|----------|------------------|---------------|---------|
| **第一类：版本分流 + 协议混合 + 半复用（最复杂）** | **TrueNAS** | **mapping 分支** | `(req, res, map)` | ✅ 接收并使用（v1 转传给 credentialed，v2 自己调） | ✅ v1 直接 `return credentialedProxyHandler(...)`；v2 自己调 `validateWidgetData` + `map` | v1 REST / v2 WebSocket 双模式，半复用半自主 |
| **第二类：名义上走 mapping 但实际忽略** | Plex、FreshRSS | mapping 分支 | `(req, res)` | ❌ 不接收（JS 忽略第三个参数） | ❌ 完全自主，只复用 `httpProxy` | mappings 仅用于通过合法性检查 |
| **第三类：走快速分支（无 endpoint）** | Deluge、Pi-hole | 快速分支 | `(req, res)` | ❌ 不接收 | ⚠️ Deluge 复用 `sendJsonRpcRequest` 工具函数；Pi-hole 完全自主 | 无 mappings，前端不传 endpoint |
| **第四类：calendar 特例分支** | Calendar | 快速分支 | `(req, res)` | ❌ 不接收 | ❌ 只复用 `httpProxy` | endpoint 是集成名称而非 API 路径 |

以下按复杂度从高到低详细说明：

---

#### 模式 F：版本分流 + 协议混合 + 半复用 credentialed（TrueNAS）

**⚠️ 这是架构最精巧、最值得研究的自定义 proxy**，代码见 [truenas/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js)

##### 函数签名（显式接收 map 参数）
```javascript
export default async function truenasProxyHandler(req, res, map) {
  const { group, service, endpoint, index } = req.query;
  // endpoint 已经是入口层改写后的真实 REST 路径，如 "system/info"
  // ...
}
```

##### 核心流程

**① 版本分流** [truenas/proxy.js#L122-L126](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js#L122-L126)
```javascript
const version = Number(widget.version ?? 1);
if (Number.isNaN(version) || version < 2) {
  // Use legacy REST proxy for version 1
  return credentialedProxyHandler(req, res, map);  // ★ 直接委托给通用 handler！
}
// version >= 2：走 WebSocket 流程
```

**⚠️ v1 REST 模式的关键点**：
- 直接 `return credentialedProxyHandler(req, res, map)`，把入口层传递的 `map` **原样转发**
- 此时 `req` 已经被入口层改写过（method、body、endpoint 路径、extraHeaders 等）
- 复用了 credentialed 内置的 Basic Auth（无 key 时）、三层请求头合并、validate+map 顺序
- 这种模式相当于"在自定义 proxy 中打了个洞，直接走回通用流水线"

**② v2+ WebSocket 模式**

**②-1 反向查找 mapping 取 wsMethod** [truenas/proxy.js#L128-L134](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js#L128-L134)
```javascript
const mappingEntry = Object.values(widgets[widget.type].mappings).find(
  (mapping) => mapping.endpoint === endpoint  // endpoint 已经是真实路径，如 "system/info"
);
const wsMethod = mappingEntry.wsMethod;  // → "system.info"
```
入口层已经帮它做了 endpoint 名称→路径的转换，但 TrueNAS 需要的是 `wsMethod` 字段，所以**自己再反向查找一次 mapping**。这是一种"半复用"——入口层处理了 segments/query 参数和安全校验，但协议相关的字段（wsMethod）还得自己取。

**②-2 WebSocket 连接与鉴权**

WebSocket URL 构建：
```javascript
const wsUrl = new URL(formatApiCall(widgets[widget.type].wsAPI, { ...widget }));
const useSecure = wsUrl.protocol === "https:" || Boolean(widget.key); // API key 要求 wss
wsUrl.protocol = useSecure ? "wss:" : "ws:";
const ws = new WebSocket(wsUrl, { rejectUnauthorized: false });
```

鉴权流程 [truenas/proxy.js#L84-L102](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js#L84-L102)：
```javascript
async function authenticate(ws, widget) {
  if (widget?.key) {
    // 优先用 API Key
    const apiKeyResult = await sendMethod(ws, "auth.login_with_api_key", [widget.key]);
    if (apiKeyResult === true) return;
  }
  // 失败 fallback 到用户名密码
  if (widget?.username && widget?.password) {
    const loginResult = await sendMethod(ws, "auth.login", [widget.username, widget.password]);
    if (loginResult === true) return;
  }
  throw new Error("TrueNAS authentication failed");
}
```

JSON-RPC over WebSocket 工具函数 [truenas/proxy.js#L69-L82](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js#L69-L82)：
```javascript
let nextId = 1;
async function sendMethod(ws, method, params = []) {
  const id = nextId++;
  const payload = { jsonrpc: "2.0", id, method, params };  // JSON-RPC 2.0 格式
  ws.send(JSON.stringify(payload));
  return waitForEvent(ws, (message) => {
    if (message?.id !== id) return undefined;  // 按 id 匹配响应
    if (message?.error) return new Error(message.error?.message);
    return message?.result ?? message;
  });
}
```

**②-3 validate 与 map 的位置（顺序与 generic 一致）** [truenas/proxy.js#L150-L156](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/proxy.js#L150-L156)
```javascript
data = await sendMethod(ws, wsMethod);

// ★ 第一步：validate（与 generic 完全相同的调用，传入真实 endpoint 路径）
if (!validateWidgetData(widget, endpoint, data)) {
  return res.status(500).json({ error: { message: "Invalid data", ... } });
}

// ★ 第二步：map（与 generic 顺序完全一致）
if (map) data = map(data);

return res.status(200).json(data);
```
**⚠️ 重要一致点**：validate 和 map 的调用顺序与 generic/credentialed 完全相同——**先 validate，后 map**。这保持了整个代理层的契约一致性。

**⚠️ TrueNAS 各 endpoint 的 validate/map 行为差异**：

| 逻辑 endpoint | 真实路径 | wsMethod | validate | map | validateWidgetData 行为 |
|--------------|---------|----------|----------|-----|------------------------|
| `alerts` | `alert/list` | `alert.list` | 无 | ✅ 有 | `mapping.validate` 为 undefined → `?.forEach` 空操作 → 直接通过 |
| `status` | `system/info` | `system.info` | `["loadavg", "uptime_seconds"]` | 无 | 检查响应中 `loadavg` 和 `uptime_seconds` 两个 key 是否都存在 |
| `pools` | `pool` | `pool.query` | 无 | ✅ 有 | 同 alerts，无 validate → 直接通过 |
| `dataset` | `pool/dataset` | `pool.dataset.query` | 无 | 无 | 无 validate 也无 map → 原样返回 |

**alerts 的 map 转换逻辑**（[widget.js#L14-L19](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/widget.js#L14-L19)）：
```javascript
map: (data) => {
  if (Array.isArray(data)) {
    // WebSocket 返回的是原生数组
    return { pending: data.filter((item) => item?.dismissed === false).length };
  }
  // REST 返回的可能是 Buffer → asJson 后的对象，需要 jsonArrayFilter 辅助
  return { pending: jsonArrayFilter(data, (item) => item?.dismissed === false).length };
}
```
- 输入：告警对象数组 `[{ dismissed: false, ... }, { dismissed: true, ... }, ...]`
- 输出：`{ pending: <未 dismissed 的数量> }`
- 两个分支分别处理数组数据（WebSocket 原生）和类 Buffer 数据（REST 兼容），过滤逻辑相同

**pools 的 map 转换逻辑**（[widget.js#L29-L36](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/truenas/widget.js#L29-L36)）：
```javascript
map: (data) => {
  const list = Array.isArray(data) ? data : asJson(data);
  return list.map((entry) => ({ id: entry.name, name: entry.name, healthy: entry.healthy }));
}
```
- 输入：存储池对象数组 `[{ name: "tank", healthy: true, ... }, ...]`
- 输出：`[{ id: "tank", name: "tank", healthy: true }, ...]`（只保留前端需要的 3 个字段）
- 同样有 Array.isArray 兼容处理

##### TrueNAS 设计总结
TrueNAS 是自定义 proxy 中**对通用层复用程度最高**的：
- v1：100% 复用，直接 `return credentialedProxyHandler(...)`
- v2：复用 `validateWidgetData` 工具函数 + `map` 转换函数 + 入口层的参数处理/安全校验
- 仅协议（WebSocket）和认证流程是完全自定义的

---

#### 模式 A：多 API 聚合并缓存（Plex，名义上走 mapping 但实际忽略）

**代表**：[plex/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/plex/proxy.js)

函数签名：
```javascript
export default async function plexProxyHandler(req, res) {  // ★ 不接收 map
  const widget = await getWidget(req);
  const { service, index } = req.query;
  // 注意：根本不读 req.query.endpoint！
  // ...
}
```

特点：
- 名义上走 mapping 分支（有 endpoint），但入口层传的 map 被忽略
- `req.query.endpoint` 被入口层改写为 "/"，但 Plex 根本不读它
- 内部硬编码调 3 类共 N 个真实 API（`/status/sessions`、`/library/sections`、各媒体库条目数）
- `fetchFromPlexAPI` 封装：Plex 返回 XML，用 `xml-js` 转 JSON
- memory-cache 分级缓存：`librariesCacheKey`（6 小时）、`albumsCacheKey/tvCacheKey/moviesCacheKey`（10 分钟）
- 对不同类型的媒体库并行 `Promise.all` 调对应条目数 API，累加计数
- 最终手动构造 `{ streams, albums, movies, tv }` 返回

---

#### 模式 B：版本分支 + 协议转换（Pi-hole，走快速分支）

**代表**：[pihole/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/pihole/proxy.js)

函数签名：
```javascript
export default async function piholeProxyHandler(req, res) {  // ★ 不接收 map
  const { group, service, index } = req.query;
  // 注意：无 endpoint 参数
  // ...
}
```

特点：
- 走 `!req.query.endpoint` 快速分支（前端不传 endpoint）
- `widget.version < 6`（v5）：直接调 `apiv5` 模板 URL，query 参数传 auth token，响应原样透传
- `widget.version >= 6`（v6）：完全不同的流程——POST `/api/auth` 拿 `session.sid` → 缓存 sid（有效期为 API 返回的 validity 秒数）→ `X-FTL-SID` 请求头调业务 API → 手动字段映射，把 v6 响应格式转换为 v5 兼容格式返回前端
- 前端组件不需要关心后端版本，数据接口统一

---

#### 模式 C：登录态管理（401 自动重试）+ 多 API 聚合（FreshRSS，名义上走 mapping 但实际忽略）

**代表**：[freshrss/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/freshrss/proxy.js)

函数签名：
```javascript
export default async function freshrssProxyHandler(req, res) {  // ★ 不接收 map
  const { group, service, index } = req.query;
  // 注意：根本不读 req.query.endpoint！
  // ...
}
```

特点：
- 名义上走 mapping 分支（有 endpoint="info"），但入口层传的 map 被忽略
- `req.query.endpoint` 被入口层改写为 "/"，但 FreshRSS 根本不读它
- 使用 Google Account Login 协议（`accounts/ClientLogin`，`application/x-www-form-urlencoded` POST）
- `apiCall` 封装了「调业务 API → 401 时重新登录 → 重试」的循环
- 缓存 token（无过期时间，永久保存直到 401 触发刷新）
- 最终聚合两个 API：`subscription/list`（取订阅数） + `unread-count`（取最大未读数）

---

#### 模式 D：复用 sendJsonRpcRequest + 自定义错误码 + 登录重试（Deluge，走快速分支）

**⚠️ 与 jsonrpcProxyHandler 的关系重点区分**：

**代表**：[deluge/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/deluge/proxy.js)

| 维度 | jsonrpcProxyHandler（完整 Handler） | deluge/proxy.js（自定义 proxy） |
|------|-----------------------------------|--------------------------------|
| 身份 | 可直接当 proxyHandler 使用的完整 handler | 独立自定义 handler，走入口层快速分支 |
| 与 jsonrpc 模块关系 | **就是**该模块的 default export | **import 了**该模块的 `sendJsonRpcRequest` 工具函数 |
| 适用场景 | 简单场景：一个 endpoint → 一个 JSON-RPC method，无认证逻辑 | 复杂场景：固定 method（`web.update_ui`），需要对 JSON-RPC 错误码做语义翻译，需要根据错误码触发登录流程 |
| JSON-RPC method 来源 | `req.query.endpoint` | 写死的常量 `dataMethod = "web.update_ui"` 和 `loginMethod = "auth.login"` |
| 认证流程 | 仅处理 HTTP Basic/Bearer（sendJsonRpcRequest 内置） | 额外：收到 JSON-RPC 错误码 1（未登录）→ 调 `auth.login` → 重试 |
| 返回行为 | `res.status().end(data)` | `res.status().end(data)`（相同） |
| map 参数 | 走 mapping 分支，接收并使用 | 走快速分支，无 map |

Deluge 的核心流程：
1. 调 `sendRpc(url, "web.update_ui", dataParams)` → 内部是 `sendJsonRpcRequest`
2. `sendRpc` 在 `sendJsonRpcRequest` 返回的基础上**再做一层 JSON-RPC 错误码翻译**：code === 1 → 转 HTTP 403（未登录），其他错误 → HTTP 500
3. 若 HTTP 403 → 调 `login(url, widget.password)`（`auth.login` method）→ 成功后**重试**数据请求
4. 注意：`sendJsonRpcRequest` 已把 JSON 字符串化过，Deluge 返回时再次 `JSON.parse` 判断 `json.error`，最后又 `end(data)` 直接写回字符串——两次 parse 纯为判断错误码

---

#### 模式 E：集成 URL 直连（calendar 特例分支）

**代表**：[calendar/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/196-homepage/src/widgets/calendar/proxy.js)

**⚠️ 这是入口层特例分支 `serviceProxyHandler === calendarProxyHandler` 的对应实现**：

函数签名：
```javascript
export default async function calendarProxyHandler(req, res) {  // ★ 不接收 map
  const { group, service, index, endpoint } = req.query;
  // endpoint 是集成名称，不是 API 路径！
  // ...
}
```

特点：
- `endpoint` 不是 API 路径，而是用户配置中 `integrations` 数组的 `name` 字段
- proxy handler 内部 `widget.integrations.find(i => i.name === endpoint)` 查找集成配置
- 从集成配置中取真实 URL（iCal 链接），直接 `httpProxy(integration.url)`
- 特殊处理：Outlook 日历要求 User-Agent 头，手动加 `gethomepage/{version}`
- 返回：`{ data: icsFileContentString }`（把原始 ics 文本包成 JSON，前端用 ical.js 解析）

Calendar 是唯一"endpoint 含义非 API 路径"的 widget，因此必须在入口层特例化。

---

## 三、关键设计模式总结

### 1. 模板方法 + 策略模式
- `genericProxyHandler` 定义了代理请求的骨架流程（URL 构造 → 头合并 → 发请求 → 校验 → 转换 → 返回）
- 各个 `proxyHandler`（credentialed、jsonrpc、synology、unifi 工厂产物、各 widget 自定义）是不同的策略实现
- Widget 可通过 `widget.proxyHandler` 自由选择策略，也可在 endpoint 级别覆盖 `mapping.proxyHandler`

### 2. 工厂模式
`createUnifiProxyHandler` 是典型的高阶函数工厂，注入 5 个配置点生成定制化 handler。UniFi/UniFi Drive/UDMP 之间共用登录重试和 Cookie 逻辑，差异点全部通过参数注入。

### 3. 配置驱动 + 渐进式复杂度
- **层 1（90% 场景）**：`widget.js` 仅配置 `api` + `proxyHandler: genericProxyHandler` + `mappings`，零代码
- **层 2（认证复杂场景）**：换用 `credentialedProxyHandler`，认证逻辑集中维护（新增一种认证只需改 credentialed.js 一个文件）
- **层 3（协议特殊场景）**：使用 `jsonrpcProxyHandler` / `synologyProxyHandler` 等专用 handler
- **层 4（半复用场景）**：像 TrueNAS 一样，部分版本/路径复用通用 handler，部分自主实现
- **层 5（完全特殊场景）**：写自定义 `proxy.js`，可复用 `httpProxy` / `sendJsonRpcRequest` 等底层工具；可走 mapping 分支，也可走快速分支完全掌控流程

### 4. 入口层委托 + 请求对象改写
入口层 `pages/api/services/proxy.js` 不是简单的路由分发，而是**主动改写 `req` 对象的字段**（`method`、`body`、`query.endpoint`、`extraHeaders`），把 mapping 中的声明式配置"预编译"到请求对象上，后续 handler 只需直接读这些字段即可正常工作。

### 5. 隐含契约一致性
**先 validate 后 map** 的顺序在整个代理层保持一致：
- generic.js：`validateWidgetData` → `if (map) resultData = map(resultData)`
- credentialed.js：完全相同的顺序
- truenas/proxy.js（WebSocket 路径）：`validateWidgetData` → `if (map) data = map(data)`（自己显式调用，顺序一致）

这个契约确保了 `validate` 永远基于目标服务的原始数据结构，`map` 永远基于校验通过的数据。

### 6. 安全机制（分层防御）
- 入口层 endpoint 白名单（mapping 或正则），不允许任意路径代理
- Segments key 白名单 + value 路径遍历防护（`/`、`\`、`..` 禁止）
- Query 参数白名单（mapping.params + optionalParams 双重过滤）
- HTTP 方法校验（mapping.method 不匹配直接 403）
- 错误信息中的 URL 自动脱敏（apikey/token 等替换为 `***`）
- generic/credentialed 中 `validateWidgetData` 确保目标服务返回的数据结构正确，避免下游组件崩溃

---

## 四、请求链路完整示例

### 示例 1：Sonarr queue 端点（走 mapping 分支 + generic + validate + map）

```
前端 SonarrWidget 组件
  ↓ useWidgetAPI(widget, "queue")
  ↓ formatProxyUrl → /api/services/proxy?group=...&service=sonarr&index=0&endpoint=queue
  ↓ HTTP GET
pages/api/services/proxy.js (入口 L29)
  ↓ 有 endpoint 且 handler 不是 calendar → 不进快速分支
  ↓ widgets["sonarr"].mappings["queue"] → { endpoint: "queue", validate: ["totalRecords"], map: undefined }
  ↓ mapping.method 未设置 → 不改写 req.method（保留 GET）
  ↓ mapping.body 未设置 → 不改写 req.body
  ↓ req.query.endpoint = "queue"（真实路径 = 逻辑名，巧合而已）
  ↓ segments/query 未传 → 不改写
  ↓ mapping.headers 未设置 → req.extraHeaders = undefined
  ↓ 调用 genericProxyHandler(req, res, map=undefined)
genericProxyHandler
  ↓ formatApiCall("{url}/api/v3/{endpoint}?apikey={key}", { endpoint:"queue", ...widget })
  ↓ → https://sonarr.example.com/api/v3/queue?apikey=xxx
  ↓ headers 三层合并（空）+ 无 username/password → 无额外 Headers
  ↓ method = widget.method ?? req.method = GET
  ↓ req.body 和 widget.requestBody 均无 → 无 Body
  ↓ httpProxy(url, { method, headers })
http.js
  ↓ follow-redirects 发起 HTTPS 请求，自动解压
  ↓ 返回 [200, "application/json", <Buffer>]
genericProxyHandler (继续)
  ↓ status === 200
  ↓ validateWidgetData → 检查 totalRecords 是否存在于响应 JSON
  ↓ validate 通过 → map 为 undefined → 不转换
  ↓ res.status(200).send(buffer)
  ↓
前端 SWR 缓存 → 组件渲染
```

### 示例 2：TrueNAS status 端点（走 mapping 分支 + 自定义 proxy + 版本分流 + v2 WebSocket）

```
前端 TrueNASWidget 组件
  ↓ useWidgetAPI(widget, "status")
  ↓ formatProxyUrl → /api/services/proxy?group=...&service=truenas&index=0&endpoint=status
  ↓ HTTP GET
pages/api/services/proxy.js (入口 L29)
  ↓ 有 endpoint 且 handler 不是 calendar → 不进快速分支
  ↓ widgets["truenas"].mappings["status"] → { endpoint: "system/info", wsMethod: "system.info", validate: ["loadavg", "uptime_seconds"] }
  ↓ mapping.method 未设置 → req.method = "GET"
  ↓ mapping.body 未设置 → 不改写
  ↓ req.query.endpoint = "system/info"（逻辑名 status → 真实 REST 路径 system/info）
  ↓ segments/query 未传 → 不改写
  ↓ map = undefined（此 endpoint 无 map 函数）
  ↓ 调用 truenasProxyHandler(req, res, map=undefined)
truenasProxyHandler
  ↓ getServiceWidget 获取服务配置（含 version=2）
  ↓ version >= 2 → 不走 REST，走 WebSocket
  ↓ Object.values(mappings).find(m => m.endpoint === "system/info") → 取 wsMethod = "system.info"
  ↓ formatApiCall("{url}/api/current", widget) → https://truenas.example.com/api/current
  ↓ protocol 改 wss://（因为有 key）
  ↓ new WebSocket(wsUrl) → waitForEvent "open"
  ↓ authenticate(ws, widget)
  │   ← sendMethod(ws, "auth.login_with_api_key", [key]) → true
  ↓ sendMethod(ws, "system.info") → { loadavg: [...], uptime_seconds: 12345, ... }
  ↓ validateWidgetData(widget, "system/info", data) → 检查 loadavg 和 uptime_seconds 字段存在
  ↓ validate 通过 → map 为 undefined → 不转换
  ↓ res.status(200).json(data)
  ↓
前端 SWR 缓存 → 组件渲染
```

### 示例 3：TrueNAS alerts 端点（v2 WebSocket，无 validate + 有 map）

```
...（入口层处理同上）
  ↓ widgets["truenas"].mappings["alerts"] → { endpoint: "alert/list", wsMethod: "alert.list", map: (data) => {...} }
  ↓ req.query.endpoint = "alert/list"
  ↓ map = mapping.map（有转换函数）
  ↓ 调用 truenasProxyHandler(req, res, map=fn)
truenasProxyHandler
  ...（WebSocket 连接和鉴权同上）
  ↓ sendMethod(ws, "alert.list") → [{ dismissed: false, ... }, { dismissed: true, ... }, ...]
  ↓ validateWidgetData(widget, "alert/list", data) → 此 mapping 无 validate 字段 → mapping?.validate?.forEach 空操作 → 直接通过
  ↓ map(data) → { pending: 1 }（Array.isArray 分支：filter(item => item.dismissed === false).length）
  ↓ res.status(200).json({ pending: 1 })
```

### 示例 4：TrueNAS v1 REST 模式（直接复用 credentialed）

```
...（入口层处理同上）
  ↓ widget.version = 1
truenasProxyHandler
  ↓ version < 2 → return credentialedProxyHandler(req, res, map)
credentialedProxyHandler
  ↓ formatApiCall("{url}/api/v2.0/{endpoint}", { endpoint: "system/info", ... })
  ↓ → https://truenas.example.com/api/v2.0/system/info
  ↓ 因为无 key，自动加 Basic Auth 头（username + password）
  ↓ headers 三层合并 + application/json
  ↓ httpProxy(...) → 返回 [200, ...]
  ↓ validateWidgetData → 检查 loadavg 和 uptime_seconds
  ↓ validate 通过 → map（如果有）
  ↓ res.status(200).json(data)
```
**注意**：整个过程 truenasProxyHandler 没有做任何自定义处理，相当于"透明代理"直接把请求转发给 credentialed。

### 示例 5：Deluge（走快速分支 + 自定义 proxy + 复用 sendJsonRpcRequest + 登录重试）

```
前端 DelugeWidget 组件
  ↓ useWidgetAPI(widget, "")（空 endpoint，或不传）
  ↓ formatProxyUrl → /api/services/proxy?group=...&service=deluge&index=0（无 endpoint 参数）
  ↓ HTTP GET
pages/api/services/proxy.js (入口 L31)
  ↓ !req.query.endpoint → TRUE → 走快速分支
  ↓ 调用 delugeProxyHandler(req, res)  ★ 注意：无 map 参数
delugeProxyHandler
  ↓ getServiceWidget 获取服务配置
  ↓ formatApiCall("{url}/json", widget) → https://deluge.example.com/json
  ↓ sendRpc(url, "web.update_ui", dataParams)
  │   ↓ sendJsonRpcRequest(url, method, params)
  │       ↓ 构造 JSON-RPC 请求（POST + Basic Auth）
  │       ↓ httpProxy(...) → 返回 [200, "application/json", <Buffer>]
  │       ↓ JSON.parse(data).error.code === 1（未登录）
  │   ↓ sendRpc 转成 HTTP 403
  ↓ status === 403 → 触发登录流程
  ↓ login(url, widget.password) → sendRpc(url, "auth.login", [password])
  ↓ 登录成功（200）→ 再次 sendRpc(url, dataMethod, dataParams)
  ↓ 这次成功 → res.status(200).end(data)  ★ 注意：.end() 直接写字符串
  ↓
前端 SWR 缓存 → 组件渲染
```
