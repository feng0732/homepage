# Sonarr / Radarr 集成代码梳理

## 一、整体架构概览

Sonarr（剧集管理）和 Radarr（电影管理）在 Homepage 项目中的集成采用 **分层架构**，从配置接入到数据展示分为以下 6 层：

```
┌───────────────────────────────────────────────────────────────────┐
│  ① 配置接入层  service-helpers.js                                 │
│     (services.yaml / Docker Labels / Kubernetes Ingress)          │
├───────────────────────────────────────────────────────────────────┤
│  ② Widget 定义层  sonarr/widget.js + radarr/widget.js             │
│     (API 模板 / Endpoint Mappings / 数据转换函数)                 │
├───────────────────────────────────────────────────────────────────┤
│  ③ 服务端代理入口  pages/api/services/proxy.js                    │
│     (Endpoint 映射解析 / 参数过滤 / Handler 分发)                 │
├───────────────────────────────────────────────────────────────────┤
│  ④ 代理处理器层  utils/proxy/handlers/generic.js                  │
│     (URL 拼接 / 认证注入 / HTTP 请求 / 数据校验 + 转换)           │
├───────────────────────────────────────────────────────────────────┤
│  ⑤ 前端数据 Hook  utils/proxy/use-widget-api.js                   │
│     (SWR 缓存 + 自动刷新 / 请求 URL 构造)                         │
├───────────────────────────────────────────────────────────────────┤
│  ⑥ UI 组件层  sonarr/component.jsx + radarr/component.jsx         │
│     (数据聚合 / 状态渲染 / 下载队列 / 日历集成)                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 二、第 ① 层：配置接入 —— 服务如何被 Homepage 识别

核心文件：[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js)

### 2.1 三种服务发现方式

| 方式 | 入口函数 | 配置来源 |
|------|----------|----------|
| 静态配置 | `servicesFromConfig()` (L53-L61) | `config/services.yaml` |
| Docker 自动发现 | `servicesFromDocker()` (L63-L170) | 容器标签 `homepage.*` |
| Kubernetes 发现 | `servicesFromKubernetes()` (L172-L229) | Ingress 注解 |

### 2.2 配置数据结构（services.yaml 示例）

```yaml
- 媒体管理:
    - Sonarr:
        href: http://sonarr:8989
        icon: sonarr.png
        widget:
          type: sonarr
          url: http://sonarr:8989
          key: YOUR_API_KEY
          enableQueue: true      # 是否显示下载队列
    - Radarr:
        href: http://radarr:7878
        widget:
          type: radarr
          url: http://radarr:7878
          key: YOUR_API_KEY
          enableQueue: true
```

### 2.3 配置清洗：`cleanServiceGroups()` (L231-L708)

这是 Sonarr/Radarr 配置处理的关键函数，执行以下操作：

**① Widget 字段白名单** (L253-L439)：只向前端传递安全字段，`url` 和 `key` 等敏感字段 **不会** 发送到浏览器。

**② Sonarr/Radarr 专属参数处理** (L555-L557)：

```javascript
if (["sonarr", "radarr"].includes(type)) {
  if (enableQueue !== undefined) widget.enableQueue = JSON.parse(enableQueue);
}
```

最终输出给前端的 Widget 对象结构：
```javascript
{
  type: "sonarr",             // 或 "radarr"
  fields: null,
  hide_errors: false,
  service_name: "Sonarr",
  service_group: "媒体管理",
  index: 0,
  enableQueue: true           // 可选
}
```

> **安全设计要点**：API Key 仅存在于服务端内存中，通过 `getServiceWidget()` (L754-L761) 在代理请求时按需读取，永不暴露到前端。

---

## 三、第 ② 层：Widget 定义 —— 声明式 API 契约

### 3.1 Sonarr Widget 定义

文件：[sonarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/widget.js)

```javascript
const widget = {
  api: "{url}/api/v3/{endpoint}?apikey={key}",   // API URL 模板
  proxyHandler: genericProxyHandler,              // 使用通用处理器

  mappings: {
    // 1. 获取剧集列表
    series: {
      endpoint: "series",
      map: (data) => asJson(data).map((entry) => ({
        title: entry.title,
        id: entry.id,
      })),
    },
    // 2. 获取队列总数
    queue: {
      endpoint: "queue",
      validate: ["totalRecords"],    // 响应必须包含该字段
    },
    // 3. 获取缺失/想要的剧集
    "wanted/missing": {
      endpoint: "wanted/missing",
      validate: ["totalRecords"],
    },
    // 4. 获取队列详情（带下载进度）
    "queue/details": {
      endpoint: "queue/details",
      map: (data) => asJson(data)
        .map((entry) => ({
          trackedDownloadState: entry.trackedDownloadState,
          trackedDownloadStatus: entry.trackedDownloadStatus,
          timeLeft: entry.timeleft,
          size: entry.size,
          sizeLeft: entry.sizeleft,
          seriesId: entry.seriesId,
          episodeTitle: entry.episode?.title ?? entry.title,
          episodeId: entry.episodeId ?? entry.id,
          status: entry.status,
        }))
        .sort((a, b) => { /* 下载中优先，完成度高优先 */ }),
    },
    // 5. 日历数据
    calendar: {
      endpoint: "calendar",
      params: ["start", "end", "unmonitored", "includeSeries",
               "includeEpisodeFile", "includeEpisodeImages"],
    },
  },
};
```

### 3.2 Radarr Widget 定义

文件：[radarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/radarr/widget.js)

```javascript
const widget = {
  api: "{url}/api/v3/{endpoint}?apikey={key}",
  proxyHandler: genericProxyHandler,

  mappings: {
    // 1. 获取电影统计（在 Widget 内部聚合 wanted/have/missing）
    movie: {
      endpoint: "movie",
      map: (data) => ({
        wanted: jsonArrayFilter(data, (item) => 
          item.monitored && !item.hasFile && item.isAvailable).length,
        have: jsonArrayFilter(data, (item) => item.hasFile).length,
        missing: jsonArrayFilter(data, (item) => 
          item.monitored && !item.hasFile).length,
        all: asJson(data).map((entry) => ({
          title: entry.title,
          id: entry.id,
        })),
      }),
    },
    // 2. 队列状态
    "queue/status": {
      endpoint: "queue/status",
      validate: ["totalCount"],
    },
    // 3. 队列详情
    "queue/details": {
      endpoint: "queue/details",
      map: (data) => asJson(data)
        .map((entry) => ({
          trackedDownloadState: entry.trackedDownloadState,
          trackedDownloadStatus: entry.trackedDownloadStatus,
          timeLeft: entry.timeleft,
          size: entry.size,
          sizeLeft: entry.sizeleft,
          movieId: entry.movieId ?? entry.id,
          status: entry.status,
        }))
        .sort(/* 同 Sonarr 排序逻辑 */),
    },
    // 4. 日历数据
    calendar: {
      endpoint: "calendar",
      params: ["start", "end", "unmonitored"],
    },
  },
};
```

### 3.3 Widget 注册

文件：[widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/widgets.js) (L112-L120, L272-L280)

```javascript
import radarr from "./radarr/widget";
import sonarr from "./sonarr/widget";

const widgets = {
  radarr,     // key 与 widget.type 对应
  sonarr,
  // ... 其他 widget
};
```

文件：[components.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/components.js) (L122, L130)

```javascript
const components = {
  radarr: dynamic(() => import("./radarr/component")),   // 懒加载
  sonarr: dynamic(() => import("./sonarr/component")),
};
```

---

## 四、第 ③ 层：服务端代理入口 —— 请求路由中枢

文件：[pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js)

### 4.1 完整请求处理流程

```
前端请求 /api/services/proxy?group=媒体管理&service=Sonarr&index=0&endpoint=queue
    │
    ▼
① getServiceWidget(group, service, index)
   └─► 从 services.yaml / Docker / K8s 中读取完整 Widget 配置（含 url 和 key）
    │
    ▼
② widgets[type]  →  查找 sonarr/radarr 的 widget 定义对象
    │
    ▼
③ widget.mappings[endpoint]  →  解析 "queue" → { endpoint: "queue", validate: [...] }
    │
    ▼
④ 处理 query 参数（如果 mapping 声明了 params）
   └─► 白名单过滤，只允许 mapping.params 中声明的参数通过
    │
    ▼
⑤ 调用 proxyHandler(req, res, map)
   └─► 通常是 genericProxyHandler，传入 mapping.map 转换函数
```

### 4.2 关键安全机制

- **Endpoint 白名单** (L36-L52)：只允许访问 `widget.mappings` 中声明的 endpoint
- **Query 参数白名单** (L74-L86)：只有 `mapping.params` 和 `mapping.optionalParams` 列出的参数才会转发
- **Segments 路径注入防护** (L58-L72)：禁止 `../`、`/`、`\` 等路径穿越字符
- **Method 校验** (L44-L47)：mapping 指定的 HTTP method 必须匹配

---

## 五、第 ④ 层：通用代理处理器 —— HTTP 请求执行

文件：[handlers/generic.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/handlers/generic.js)

### 5.1 处理流程 (genericProxyHandler, L10-L99)

```javascript
// ① 构造真实请求 URL
let urlString = formatApiCall(
  widgets[widget.type].api,           // "{url}/api/v3/{endpoint}?apikey={key}"
  { endpoint, ...widget }             // 注入 url, key, endpoint 变量
).replace(/(?<=\?.*)\?/g, "&");      // 处理重复 ? 符号

// ② 构造请求头
const headers = {
  ...(widgets[widget.type].headers ?? {}),
  ...(widget.headers ?? {}),
};
// Basic Auth 注入
if (widget.username && widget.password) {
  headers.Authorization = `Basic ${Buffer.from(...).toString("base64")}`;
}

// ③ 发起真实 HTTP 请求到 Sonarr/Radarr 服务器
const [status, contentType, data] = await httpProxy(url, params);

// ④ 数据校验（validate 字段）
if (!validateWidgetData(widget, endpoint, resultData)) {
  return res.status(200).json({ error: { message: "Invalid data", ... } });
}

// ⑤ 应用 mapping.map 转换函数
if (map) resultData = map(resultData);

// ⑥ 脱敏错误 URL（隐藏 API Key）
if (resultData.error?.url) {
  resultData.error.url = sanitizeErrorURL(url);
}

// ⑦ 返回前端
return res.status(status).send(resultData);
```

### 5.2 URL 模板替换机制

文件：[api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/api-helpers.js) `formatApiCall()` (L1-L13)

模板：`"{url}/api/v3/{endpoint}?apikey={key}"`

替换逻辑：
1. 正则 `/\{.*?\}/g` 匹配所有 `{xxx}` 占位符
2. 从 args 对象中取对应值
3. `{url}` 会去掉末尾斜杠
4. 执行 **两次替换**，支持嵌套依赖

### 5.3 错误 URL 脱敏

`sanitizeErrorURL()` (L71-L78)：将 URL 中的 `apikey`, `api_key`, `token`, `access_token`, `auth` 等参数值替换为 `***`，避免日志泄露。

---

## 六、第 ⑤ 层：前端数据 Hook —— SWR 驱动的数据获取

文件：[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/use-widget-api.js)

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;  // 自定义刷新间隔
  }
  // 构造代理 API URL
  let url = formatProxyUrl(widget, ...options);
  // formatProxyUrl 输出示例：
  // /api/services/proxy?group=媒体管理&service=Sonarr&index=0&endpoint=queue

  if (options[0] === "") url = null;   // endpoint 为空则不请求

  const { data, error, mutate } = useSWR(url, config);
  // SWR 特性：自动重试、窗口聚焦刷新、缓存共享、去重请求

  return {
    data,
    error: data?.error ?? error,       // 优先显示 API 返回的业务错误
    mutate                              // 手动刷新函数
  };
}
```

`formatProxyUrl()` 定义在 [api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/api-helpers.js#L43-L49)：

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

## 七、第 ⑥ 层：UI 组件层 —— 数据展示与交互

### 7.1 Sonarr 组件

文件：[sonarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx)

#### 并行请求 4 个 Endpoint (L28-L31)

```javascript
const { data: wantedData, error: wantedError } = useWidgetAPI(widget, "wanted/missing");
const { data: queuedData, error: queuedError }   = useWidgetAPI(widget, "queue");
const { data: seriesData, error: seriesError }   = useWidgetAPI(widget, "series");
const { data: queueDetailsData, error: queueDetailsError } = useWidgetAPI(widget, "queue/details");
```

#### 三态渲染

| 状态 | 条件 | 渲染内容 |
|------|------|----------|
| 错误态 | 任一请求 error | `<Container error={finalError} />` |
| 加载态 | 任一 data 为 null | 3 个 `<Block label="..." />` 占位骨架 |
| 就绪态 | 全部数据就绪 | 统计 Block + 队列列表 |

#### 统计数据展示 (L63-L67)

```
┌─────────────────────────────────────┐
│  wanted    │  queued    │  series   │
│  共 12 条  │  共 3 条   │  共 45 部 │
└─────────────────────────────────────┘
```

对应代码：
```jsx
<Block label="sonarr.wanted" value={t("common.number", { value: wantedData.totalRecords })} />
<Block label="sonarr.queued" value={t("common.number", { value: queuedData.totalRecords })} />
<Block label="sonarr.series" value={t("common.number", { value: seriesData.length })} />
```

#### 下载队列渲染 (L68-L77)

当 `enableQueue=true` 且队列非空时，渲染每个队列条目：

```jsx
queueDetailsData.map((queueEntry) => (
  <QueueEntry
    progress={getProgress(queueEntry.sizeLeft, queueEntry.size)}
    timeLeft={queueEntry.timeLeft}
    title={getTitle(queueEntry, seriesData) ?? t("sonarr.unknown")}
    activity={formatDownloadState(queueEntry.trackedDownloadState)}
    key={`${queueEntry.seriesId}-${queueEntry.episodeId}`}
  />
))
```

`getTitle()` (L14-L22)：通过 `seriesId` 关联 `seriesData` 查出剧集名，拼接成 `{剧集名}: {单集标题}`。

`formatDownloadState()` (L33-L42)：将状态码转可读文本，如 `importPending` → `"import pending"`。

### 7.2 Radarr 组件

文件：[radarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/radarr/component.jsx)

#### 并行请求 3 个 Endpoint (L18-L20)

```javascript
const { data: moviesData, error: moviesError }           = useWidgetAPI(widget, "movie");
const { data: queuedData, error: queuedError }           = useWidgetAPI(widget, "queue/status");
const { data: queueDetailsData, error: queueDetailsError } = useWidgetAPI(widget, "queue/details");
```

> **注意**：Radarr 的 `movie` 接口在 Widget 层已做了数据聚合，返回 `{wanted, have, missing, all}`，因此前端少一个请求。

#### 统计 Block (L53-L58)

```
┌──────────────────────────────────────────────────┐
│  wanted    │  missing   │  queued    │  movies   │
│  共 8 条   │  共 15 条  │  共 2 条   │  共 120 部│
└──────────────────────────────────────────────────┘
```

对应代码：
```jsx
<Block label="radarr.wanted"  value={t("common.number", { value: moviesData.wanted })} />
<Block label="radarr.missing" value={t("common.number", { value: moviesData.missing })} />
<Block label="radarr.queued"  value={t("common.number", { value: queuedData.totalCount })} />
<Block label="radarr.movies"  value={t("common.number", { value: moviesData.have })} />
```

#### 队列关联电影名 (L59-L68)

通过 `movieId` 在 `moviesData.all` 中查找电影标题：
```jsx
title={moviesData.all.find((entry) => entry.id === queueEntry.movieId)?.title ?? t("radarr.unknown")}
```

### 7.3 Sonarr 日历集成

文件：[calendar/integrations/sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx)

#### 请求日历数据 (L8-L14)

```javascript
const { data: sonarrData, error: sonarrError } = useWidgetAPI(config, "calendar", {
  ...params,                              // 父组件传入的 start, end
  includeSeries: "true",
  includeEpisodeFile: "false",
  includeEpisodeImages: "false",
  ...(config?.params ?? {}),              // 用户自定义参数覆盖
});
```

#### 事件构造 (L23-L34)

```javascript
sonarrData?.forEach((event) => {
  eventsToAdd[title] = {
    title: event.series.title,                   // 剧集名
    date: DateTime.fromISO(event.airDateUtc),    // 播出时间
    color: config?.color ?? "teal",
    isCompleted: event.hasFile,                  // 是否已下载
    additional: `S${event.seasonNumber} E${event.episodeNumber}`,
    url: `${config.baseUrl}/series/${event.series.titleSlug}`,  // 跳转到 Sonarr
  };
});
```

### 7.4 Radarr 日历集成

文件：[calendar/integrations/radarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.jsx)

#### 多日期事件类型

一部电影可能生成 3 个日历事件：

| 事件类型 | 数据字段 | 默认颜色 |
|----------|----------|----------|
| 影院上映 | `inCinemas` | amber (琥珀) |
| 实体碟发行 | `physicalRelease` | cyan (青色) |
| 数字版发行 | `digitalRelease` | emerald (翠绿) |

---

## 八、完整数据链路图（以 Sonarr "queue" 请求为例）

```
┌───────────────┐
│  Browser      │  useWidgetAPI(widget, "queue")
│  (React)      │        │
└───────┬───────┘        │  构造 URL: /api/services/proxy?group=X&service=Y&index=0&endpoint=queue
        │                ▼
        │       ┌───────────────────────────┐
        │       │  SWR (useSWR)             │
        │       │  - 缓存去重               │
        │       │  - 窗口聚焦刷新           │
        │       │  - 自动重试               │
        │       └───────────┬───────────────┘
        │                   │ HTTP GET
        ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│  Next.js API Route  /api/services/proxy.js                       │
│    ├─ getServiceWidget() → 读取 {url, key, enableQueue, ...}     │
│    ├─ widgets["sonarr"].mappings["queue"] → {endpoint, validate} │
│    └─ 调用 genericProxyHandler(req, res, undefined)              │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  genericProxyHandler  (handlers/generic.js)                      │
│    ├─ formatApiCall()                                            │
│    │     "{url}/api/v3/{endpoint}?apikey={key}"                  │
│    │      → "http://sonarr:8989/api/v3/queue?apikey=xxx"         │
│    ├─ 注入 Basic Auth Header (如果配置)                          │
│    ├─ httpProxy() → 真实请求 Sonarr 服务器                        │
│    ├─ validateWidgetData() → 检查 totalRecords 字段存在          │
│    └─ 响应 { totalRecords: 3 }                                   │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌───────────────┐
│  Browser      │  SWR 缓存更新 → 组件 re-render
│  (React)      │        │
└───────┬───────┘        │  { data: { totalRecords: 3 } }
        │                ▼
        │       ┌───────────────────────────┐
        │       │  Sonarr Component         │
        │       │  <Block label="queued"    │
        │       │         value="共 3 条" />│
        │       └───────────────────────────┘
        ▼
   用户界面渲染
```

---

## 九、Sonarr 与 Radarr 的设计差异对比

| 维度 | Sonarr | Radarr | 原因 |
|------|--------|--------|------|
| **核心资源** | Series（剧集）→ Episode | Movie（电影） | Sonarr 多一层嵌套结构 |
| **统计 API** | 分开请求 `wanted/missing`、`queue`、`series` | 合并到 `movie` 接口一次返回 | 电影列表可在前端聚合统计 |
| **请求数** | 4 个并行请求 | 3 个并行请求 | Sonarr 多一个 series 列表用于关联剧集名 |
| **队列关联键** | `seriesId` + `episodeId` | `movieId` | Sonarr 需要双层关联 |
| **队列标题格式** | `{剧集名}: {单集标题}` | `{电影名}` | Sonarr 一集对应一个条目 |
| **日历事件** | 1 条记录 = 1 个事件 | 1 条记录 = 最多 3 个事件（影院/实体/数字） | 电影有多个发行日期 |
| **日历默认色** | teal (蓝绿) | amber/cyan/emerald (三色区分事件类型) | 同上 |

---

## 十、设计模式与最佳实践总结

### 10.1 声明式 Widget 契约
- 每个 Widget 用纯 JS 对象声明 API 模板、Endpoint 映射、数据转换函数
- 新增 Widget 只需加 `widget.js` + `component.jsx`，无需修改框架代码
- 符合 **开闭原则**

### 10.2 敏感信息服务端保留
- API Key、密码等仅在服务端内存中流转
- 前端只拿到 `{service_group, service_name, index}` 三元组标识符
- 通过服务端代理转发请求，避免 CORS 和密钥泄露

### 10.3 Opaque Endpoint（不透明端点）
- 前端使用 `queue`、`wanted/missing` 等语义化标识
- 服务端通过 `mappings` 翻译为真实 API 路径
- 解耦前端与后端 API，便于升级和兼容

### 10.4 SWR 客户端缓存
- 相同 `(group, service, index, endpoint)` 请求自动共享缓存
- 窗口切换聚焦时自动刷新
- 失败自动指数退避重试
- 多个组件可订阅同一数据源

### 10.5 Mapping 数据转换
- 数据转换集中在 Widget 定义层的 `map()` 函数
- 前端组件直接消费结构化数据，无需处理原始 API 差异
- 字段裁剪减小传输体积（Sonarr queue/details 只保留 9 个字段）

### 10.6 多层次白名单安全
| 层级 | 防护措施 | 文件位置 |
|------|----------|----------|
| Endpoint | `widget.mappings` 白名单 | proxy.js L36-L52 |
| Query 参数 | `mapping.params` / `optionalParams` 白名单 | proxy.js L74-L86 |
| Path 分段 | 禁止 `../` `/` `\` 等特殊字符 | proxy.js L58-L72 |
| HTTP Method | `mapping.method` 校验 | proxy.js L44-L47 |
| 错误脱敏 | 隐藏 URL 中的 apikey/token | generic.js L56-L58 |

---

## 十一、边界流程补充分析

### 11.1 队列开关 `enableQueue` 关闭时的完整行为

#### 问题：`enableQueue=false` 或未设置时，`queue/details` 请求是否还会发出？

**答案：会。** 这是一个容易被误解的关键行为。

以 Sonarr 为例，[component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx#L28-L31) 的 4 个 `useWidgetAPI` 调用是 **无条件执行** 的：

```javascript
// L28-L31：无论 enableQueue 设置如何，这些 Hook 都会执行
const { data: wantedData, error: wantedError } = useWidgetAPI(widget, "wanted/missing");
const { data: queuedData, error: queuedError } = useWidgetAPI(widget, "queue");
const { data: seriesData, error: seriesError } = useWidgetAPI(widget, "series");
const { data: queueDetailsData, error: queueDetailsError } = useWidgetAPI(widget, "queue/details");
```

`enableQueue` 的判断发生在数据请求 **之后**，在 [L59](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx#L59)：

```javascript
const enableQueue = widget?.enableQueue && Array.isArray(queueDetailsData) && queueDetailsData.length > 0;
```

这是一个 **三重条件** 的短路求值：

| 条件 | 含义 | 值为 false 的情况 |
|------|------|-------------------|
| `widget?.enableQueue` | 配置中是否开启队列 | 未设置 `enableQueue` 或设为 `false` |
| `Array.isArray(queueDetailsData)` | 接口返回的是否为数组 | 接口返回空对象、null、或出错 |
| `queueDetailsData.length > 0` | 队列中是否有内容 | 队列为空 |

**完整行为流程图：**

```
enableQueue 未设置 / false
        │
        ▼
  4 个 useWidgetAPI 仍然全部执行
  (SWR 仍会向 /api/services/proxy 发起请求)
        │
        ├── 全部成功 → enableQueue 为 false → 只渲染 3 个 Block，不渲染 QueueEntry
        │
        ├── queue/details 返回错误 → 整个组件进入错误态！
        │    └─ 即使 enableQueue=false，queueDetailsError 仍会触发 L44-L47 的错误渲染
        │
        └── 其他接口出错 → 同样进入错误态
```

#### ⚠️ 设计隐患

这意味着即使用户不关心队列信息（`enableQueue=false`），如果 `queue/details` 接口本身出错（如 Sonarr 服务端版本不兼容），**整个 Widget 都会显示错误**，包括本应正常显示的 wanted / queued / series 数据。

修复思路：如果 `enableQueue` 为 false，可以在 `queue/details` 出错时不将其视为致命错误，或使用条件式请求（SWR 支持 `url = null` 跳过请求）。

---

### 11.2 错误状态的完整处理链路

错误从产生到展示经过多层处理，每层都有不同行为。

#### 11.2.1 错误产生源

| 来源 | 代码位置 | 错误结构 |
|------|----------|----------|
| HTTP 错误（4xx/5xx） | [generic.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/handlers/generic.js#L75-L91) L75-L91 | `{ error: { message: "HTTP Error", url: "***", data: ... } }` |
| 数据校验失败 | [generic.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/handlers/generic.js#L61-L65) L61-L65 | `{ error: { message: "Invalid data", url: "***", data: ... } }` |
| 不支持的 endpoint | [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js#L49-L52) L49-L52 | `{ error: "Unsupported service endpoint" }` (纯字符串) |
| 网络超时/连接失败 | SWR 内部 fetch 异常 | 原生 Error 对象 |
| Sonarr/Radarr 业务错误 | API 返回非 200 | 取决于上游服务 |

#### 11.2.2 useWidgetAPI 的错误提升

[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/use-widget-api.js#L16) L16：

```javascript
return { data, error: data?.error ?? error, mutate };
```

关键逻辑：**API 返回的业务错误（`data.error`）优先于 SWR 的网络错误（`error`）**。

这意味着当服务端返回 `{ error: { message: "HTTP Error", ... } }` 时：
- `data` = `{ error: { message: "HTTP Error", ... } }`（非 null）
- `error` = SWR 层面的 undefined（fetch 本身成功）
- 合并结果：`error` = `data.error` = `{ message: "HTTP Error", ... }`

#### 11.2.3 Widget 组件的错误聚合

Sonarr [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx#L44-L47) L44-L47：

```javascript
if (wantedError || queuedError || seriesError || queueDetailsError) {
  const finalError = wantedError ?? queuedError ?? seriesError ?? queueDetailsError;
  return <Container service={service} error={finalError} />;
}
```

**任一请求失败即整体报错**，且只展示第一个非空错误（按 wanted → queued → series → queueDetails 优先级）。

#### 11.2.4 Container 的错误拦截

[container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/container.jsx#L24-L29) L24-L29：

```javascript
if (error) {
  if (settings.hideErrors || service.widget.hide_errors) {
    return null;    // 静默吞掉错误，不渲染任何内容
  }
  return <Error service={service} error={error} />;
}
```

这里有 **双重隐藏机制**：

| 机制 | 作用域 | 来源 |
|------|--------|------|
| `settings.hideErrors` | 全局（所有 Widget） | 用户全局设置 |
| `service.widget.hide_errors` | 单个 Widget | YAML 配置中的 `hideErrors: true` |

两者满足其一即完全隐藏错误，Widget 区域变为空白。

#### 11.2.5 Error 组件的容错格式化

[error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/error.jsx#L12-L23) L12-L23：

```javascript
if (typeof error === "string") {
  error = { message: error };       // "Unsupported service endpoint" → { message: "..." }
} else if (typeof error === "number") {
  error = { message: `Error ${error}` };  // HTTP 状态码数字 → { message: "Error 403" }
}

if (error?.data?.error) {
  error = error.data.error;         // 嵌套提取
}
```

最终展示的可折叠详情面板包含：
- **message**：错误描述
- **url**：脱敏后的请求 URL
- **rawError**：原始异常
- **data**：响应数据

#### 11.2.6 完整错误链路图

```
Sonarr/Radarr API 返回 500
        │
        ▼
genericProxyHandler
  ├─ httpProxy() 返回 [500, "application/json", data]
  ├─ sanitizeErrorURL() 脱敏 URL 中的 apikey
  └─ res.status(500).json({ error: { message: "HTTP Error", url: "https://sonarr:8989/api/v3/queue?apikey=***" } })
        │
        ▼
SWR fetch 成功（HTTP 200 from proxy），但 data 包含 error 字段
        │
        ▼
useWidgetAPI 提升错误：error = data.error = { message: "HTTP Error", url: "..." }
        │
        ▼
Sonarr Component: queueDetailsError 为 truthy
  └─ finalError = wantedError ?? queuedError ?? seriesError ?? queueDetailsError
     (如果 wanted/queued/series 也出错，取第一个非空)
        │
        ▼
Container 检查隐藏设置
  ├─ hideErrors (全局) 或 hide_errors (Widget 级) → return null (空白)
  └─ 不隐藏 → <Error error={finalError} />
        │
        ▼
Error 组件
  ├─ 格式化 error 对象（容错 string/number/嵌套）
  └─ 渲染可折叠 <details> 面板
```

---

### 11.3 日历配置的覆盖与隐藏错误机制

#### 11.3.1 日历 Widget 的配置合并链

[calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L70-L86) L70-L86：

```javascript
const integrations = useMemo(
  () =>
    widget.integrations
      ?.filter((integration) => integration?.type)
      .map((integration) => ({
        service: dynamic(() => import(`./integrations/${integration.type}.jsx`)),
        widget: { ...widget, ...integration },   // ← 关键：integration 覆盖 widget
      })) ?? [],
  [widget],
);
```

合并顺序 `{ ...widget, ...integration }` 意味着：

```yaml
widget:
  type: calendar
  url: http://sonarr:8989        # widget 层
  key: API_KEY                    # widget 层
  integrations:
    - type: sonarr
      url: http://sonarr-new:8989  # integration 层，覆盖 widget.url
      color: teal                   # integration 层，新增
      params:                       # integration 层，新增
        includeSeries: "true"
```

合并后传给 Sonarr 集成组件的 `config` 对象：
```javascript
{
  type: "sonarr",                    // integration.type 覆盖 widget.type
  url: "http://sonarr-new:8989",    // integration.url 覆盖 widget.url
  key: "API_KEY",                   // 保留 widget.key
  color: "teal",                    // integration 新增
  params: { includeSeries: "true" },// integration 新增
  service_name: "...",              // 保留 widget 的
  service_group: "...",             // 保留 widget 的
}
```

> **注意**：`integration.type` 会覆盖 `widget.type`（`calendar` → `sonarr`），这使得 `useWidgetAPI(config, "calendar")` 能正确找到 `widgets["sonarr"].mappings["calendar"]` 来构造请求。

#### 11.3.2 日历 API 请求参数的三层合并

以 [sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L8-L14) L8-L14 为例：

```javascript
const { data: sonarrData, error: sonarrError } = useWidgetAPI(config, "calendar", {
  ...params,                       // 第 1 层：日历组件传入的 start/end/unmonitored
  includeSeries: "true",          // 第 2 层：Sonarr 集成硬编码的默认值
  includeEpisodeFile: "false",
  includeEpisodeImages: "false",
  ...(config?.params ?? {}),       // 第 3 层：用户在 integration 中自定义的 params
});
```

**覆盖优先级**（后者覆盖前者）：

| 层级 | 来源 | 示例 |
|------|------|------|
| 第 1 层 | 日历父组件计算的日期范围 | `start: "2026-03-11"`, `end: "2026-09-11"` |
| 第 2 层 | Sonarr 集成内置的默认参数 | `includeSeries: "true"` |
| 第 3 层 | 用户 YAML 配置的 `params` | 可覆盖以上所有参数 |

最终传到代理层的 `query` 参数经过 [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js#L74-L86) L74-L86 的 **白名单过滤**：

```javascript
// widget.mappings.calendar.params = ["start", "end", "unmonitored", "includeSeries", ...]
if (req.query.query && (mappingParams || optionalParams)) {
  const queryParams = JSON.parse(req.query.query);
  let params = [];
  if (mappingParams) params = params.concat(mappingParams);       // 白名单
  if (filteredOptionalParams) params = params.concat(filteredOptionalParams);
  const query = new URLSearchParams(params.map((p) => [p, queryParams[p]]));
  req.query.endpoint = `${req.query.endpoint}?${query}`;
}
```

只转发 `mapping.params` 中声明的参数名，**用户自定义的 `params` 中如果有未声明参数会被静默丢弃**。

#### 11.3.3 日历集成的隐藏错误机制

[sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L39-L40) L39-L40：

```javascript
const error = sonarrError ?? sonarrData?.error;
return error && !hideErrors && <Error error={{ message: `${config.type}: ${error.message ?? error}` }} />;
```

[Radarr 同理](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.jsx#L64-L65) L64-L65：

```javascript
const error = radarrError ?? radarrData?.error;
return error && !hideErrors && <Error error={{ message: `${config.type}: ${error.message ?? error}` }} />;
```

**与 Widget 组件的错误处理有三点关键差异：**

| 维度 | Widget 组件 (sonarr/component.jsx) | 日历集成 (calendar/integrations/sonarr.jsx) |
|------|-------------------------------------|----------------------------------------------|
| 错误影响范围 | **整个 Widget 停止渲染**，只显示错误 | **仅显示错误提示，不影响日历其他集成的事件** |
| 隐藏方式 | `settings.hideErrors` + `widget.hide_errors` | 仅 `hideErrors` 参数（来自 `settings.hideErrors`） |
| 数据流 | 错误时 return 后，不执行后续渲染 | 错误时 useEffect 直接 return，不影响已有的 `events` 状态 |

**日历集成错误不阻断其他集成的原因：**

1. 每个集成是独立的 React 组件，各自调用自己的 `useWidgetAPI`
2. 各集成通过 `setEvents((prev) => ({ ...prev, ...eventsToAdd }))` 合并到共享 `events` 状态
3. 如果 Sonarr 集成出错，`useEffect` 中 `if (!sonarrData || sonarrError) return;` 会跳过事件添加，但已由 Radarr 等其他集成添加的事件 **不受影响**
4. 日历组件的 `<Container>` 不会收到错误，所以日历视图照常渲染

```
┌─ Calendar Component ──────────────────────────────┐
│                                                    │
│  ┌─ Sonarr Integration ─┐  ┌─ Radarr Integration ┐│
│  │  error → 跳过事件添加  │  │  成功 → 添加事件    ││
│  │  显示 Error 提示      │  │                     ││
│  └───────────────────────┘  └─────────────────────┘│
│                                                    │
│  events = { ...radarrEvents }  ← Sonarr 的被跳过  │
│                                                    │
│  ┌─ Monthly / Agenda ──────────────────────────┐  │
│  │  只展示 Radarr 的事件，Sonarr 错误不影响渲染  │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

#### 11.3.4 `hideErrors` 的传递路径

[calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L97-L107) L97-L107：

```javascript
<Integration
  config={integration.widget}
  params={params}
  setEvents={setEvents}
  hideErrors={settings.hideErrors}   // ← 全局设置，非 widget 级
/>
```

**注意**：日历集成只传了 `settings.hideErrors`，**没有传** `widget.hide_errors`。这与 Widget 组件的 Container 有所不同：
- Container 同时检查 `settings.hideErrors || service.widget.hide_errors`
- 日历集成只检查 `hideErrors`（仅全局设置）

这意味着即使某个日历 integration 配置了 `hideErrors: true`，在日历集成层面仍然可能显示错误（因为 `widget.hide_errors` 没有传递到 integration 的 `hideErrors` prop）。不过配置层已将 `hideErrors` 转为 `hide_errors` 字段传给前端，而日历集成代码只检查 `hideErrors` prop，所以 **日历集成的 widget 级错误隐藏实际上是失效的**。

---

### 11.4 跳转链接缺少字段时的处理

#### 11.4.1 Sonarr 日历链接

[sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L32) L32：

```javascript
url: config?.baseUrl && event.series.titleSlug && `${config.baseUrl}/series/${event.series.titleSlug}`,
```

这是一个 **三重短路求值**，逐一检查：

| 条件 | 缺失场景 | 结果 |
|------|----------|------|
| `config?.baseUrl` 为 falsy | 日历 integration 未配置 `baseUrl` | `url` = `undefined`（或 `false`） |
| `event.series.titleSlug` 为 falsy | Sonarr API 返回的事件中 `series.titleSlug` 缺失 | `url` = `undefined`（或 `false`） |
| 两者都存在 | 正常情况 | `url` = `"https://sonarr.example/series/show"` |

**URL 为 falsy 时的表现**：日历组件在渲染事件时，如果 `url` 为 falsy，不会生成可点击的链接，事件只显示文本。

#### 11.4.2 Radarr 日历链接

[radarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.jsx#L25) L25：

```javascript
const url = config?.baseUrl && event.titleSlug && `${config.baseUrl}/movie/${event.titleSlug}`;
```

逻辑与 Sonarr 一致，但有一个区别：**`url` 变量在循环外定义**，被 3 种事件类型（影院/实体/数字）**共享**。如果 `baseUrl` 或 `titleSlug` 缺失，**同一电影的所有事件都不可点击**。

#### 11.4.3 `baseUrl` 的来源追踪

`baseUrl` 不在 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js) 的白名单中（不在 L256-L439 的解构列表中），说明它 **不是从 services.yaml 的 widget 配置直接传给前端的**。

`baseUrl` 的实际来源是日历组件的配置合并：

[calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L83) L83：

```javascript
widget: { ...widget, ...integration },
```

而 `integration` 来自 `widget.integrations` 数组中的元素。在配置清洗阶段 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L623-L635) L623-L635：

```javascript
if (type === "calendar") {
  if (integrations) {
    if (Array.isArray(integrations)) {
      widget.integrations = integrations.map((integration) => {
        if (!integration || typeof integration !== "object") {
          return integration;
        }
        const { url, ...integrationWithoutUrl } = integration;   // ← 删除了 url！
        return integrationWithoutUrl;
      });
    }
  }
}
```

**关键发现：日历集成配置中的 `url` 字段在配置清洗时被主动删除了！**

这意味着传给前端的 integration 对象中 **没有 `url`**，只有 `type`、`key` 等字段。那 `baseUrl` 从哪来？

查看测试文件 [sonarr.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.test.jsx#L32) L32：

```javascript
config={{ type: "sonarr", baseUrl: "https://sonarr.example", color: "teal" }}
```

测试中 `baseUrl` 是直接传入的。但实际运行时，在配置清洗中 `url` 被删除后，`baseUrl` 字段并未被显式添加。

因此 **在实际运行中**，如果没有在 integration 配置中显式设置 `baseUrl`（作为独立于 `url` 的字段），`config.baseUrl` 将为 `undefined`，导致 **所有日历事件都不可点击**。

这是设计上的一个有意行为：由于服务端代理机制使得前端不知道 Sonarr/Radarr 的真实 URL（只通过 `/api/services/proxy` 中转），日历事件默认就无法直接链接到 Sonarr/Radarr 的 Web 界面。用户需要额外配置 `baseUrl` 字段才能启用跳转链接。

#### 11.4.4 `titleSlug` 缺失时的防御

Sonarr 的 `event.series.titleSlug` 和 Radarr 的 `event.titleSlug` 依赖于上游 API 返回数据的完整性：

- **Sonarr**：由于 `calendar` mapping 声明了 `includeSeries: "true"`，`event.series` 对象通常存在
- **Radarr**：`event.titleSlug` 是电影资源的标准字段，通常存在

但如果 API 返回异常数据导致字段缺失，短路求值确保 `url` 为 `undefined` 而非拼接出无效 URL（如 `https://sonarr/series/undefined`）。

#### 11.4.5 Sonarr 队列标题的防御性处理

[sonarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx#L14-L22) L14-L22：

```javascript
function getTitle(queueEntry, seriesData) {
  let title = "";
  const seriesTitle = seriesData.find((entry) => entry.id === queueEntry.seriesId)?.title;
  if (seriesTitle) title += `${seriesTitle}: `;
  const { episodeTitle } = queueEntry;
  if (episodeTitle) title += episodeTitle;
  if (title === "") return null;
  return title;
}
```

防御逻辑：

| 场景 | seriesTitle | episodeTitle | 返回值 | 使用处 |
|------|-------------|--------------|--------|--------|
| 正常 | "Breaking Bad" | "Pilot" | `"Breaking Bad: Pilot"` | QueueEntry title |
| 缺少剧集名 | "Breaking Bad" | undefined | `"Breaking Bad: "` | QueueEntry title |
| 缺少剧集标题 | undefined | "Pilot" | `"Pilot"` | QueueEntry title |
| 都缺失 | undefined | undefined | `null` | 降级为 `t("sonarr.unknown")` |

而 Radarr 的标题查找方式不同（[radarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/radarr/component.jsx#L64) L64）：

```javascript
title={moviesData.all.find((entry) => entry.id === queueEntry.movieId)?.title ?? t("radarr.unknown")}
```

使用 `?.` 可选链 + `??` 空值合并，如果 `find()` 返回 `undefined` 或找到的对象 `title` 为 `undefined`，都降级为 `t("radarr.unknown")`。

---

### 11.5 边界场景速查表

| 场景 | 行为 | 影响 |
|------|------|------|
| `enableQueue` 未设置 | 4 个请求全部发出，队列详情不渲染 | 浪费一次 `queue/details` 请求 |
| `enableQueue=false` 且 `queue/details` 出错 | 整个 Widget 进入错误态 | 正常统计也被隐藏 |
| `enableQueue=true` 但队列为空 | `queueDetailsData.length === 0`，不渲染 QueueEntry | 正常，无额外影响 |
| 日历 integration 的 `url` 被清洗删除 | `baseUrl` 为 undefined，事件不可点击 | 需用户额外配置 `baseUrl` |
| 日历 integration 的 `titleSlug` 缺失 | `url` 为 undefined，单条事件不可点击 | 不影响其他事件 |
| Sonarr 队列条目缺少 seriesId | `getTitle` 返回 `null`，降级为 `"unknown"` | 不影响其他条目 |
| Radarr 队列条目 movieId 不匹配 | `find` 返回 `undefined`，降级为 `"unknown"` | 不影响其他条目 |
| Sonarr 日历错误 | 该集成事件不添加，显示 Error 提示 | 不影响 Radarr 等其他集成 |
| `settings.hideErrors=true` | Widget 错误和日历集成错误都隐藏 | 用户无法感知异常 |
| `widget.hide_errors=true` | Widget 错误隐藏，日历集成错误 **可能仍显示** | 日历集成未传递此字段 |
| `event.series` 为 null (Sonarr) | `event.series.title` 抛异常 | ⚠️ **未防御，可能导致集成崩溃** |

> **最后一条**是真正的隐患：[sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L24) L24 的 `event.series.title` 如果 `event.series` 为 `null`，将抛出 `TypeError: Cannot read properties of null`。虽然 `includeSeries: "true"` 通常保证 `series` 存在，但极端情况下（如 Sonarr API 返回不完整数据）可能触发此问题。Radarr 不受影响，因为它直接访问 `event.title`，不依赖嵌套对象。

---

## 十二、日历集成的服务定位机制

日历集成是 Homepage 中最复杂的服务定位场景：日历 Widget 本身是一个独立服务，但它的集成项需要引用 **其他已配置的服务**（如 Sonarr、Radarr）来获取日历数据。本节深入分析这一"服务中引用服务"的定位链路，包括配置清洗的边界、序号（index）的传递方式、以及 type 在前端动态组件和服务端的不同作用。

### 12.1 两种集成模式

日历 Widget 的 integrations 支持两种完全不同的模式，它们的服务定位逻辑截然不同：

| 模式 | 典型类型 | 数据来源 | 代理方式 |
|------|----------|----------|----------|
| **服务引用模式** | `sonarr`, `radarr`, `lidarr`, `readarr` | 引用 Homepage 中已配置的其他 Widget 服务 | 走通用代理 `genericProxyHandler` |
| **直接 URL 模式** | `ical` | 集成项自身携带 URL 配置 | 走日历专属代理 `calendarProxyHandler` |

Sonarr / Radarr 属于 **服务引用模式**，这是本节分析的重点。

### 12.2 配置清洗的两层边界

理解服务定位的前提是区分**两套配置数据**：一套给前端用（经过清洗），一套给服务端内部用（原始完整）。

#### 12.2.1 两套配置数据的来源

| 配置集 | 生成函数 | 使用方 | 包含敏感信息 |
|--------|----------|--------|-------------|
| 原始配置 | `servicesFromConfig()` / `servicesFromDocker()` / `servicesFromKubernetes()` | 服务端 `getServiceWidget()` | 是（含 url、key、password 等） |
| 清洗后配置 | `cleanServiceGroups()` | 前端 React 组件 | 否（白名单过滤） |

两者是**独立生成**的：服务端代理调用 `getServiceWidget()` 时，直接读 YAML 原始配置，不走 `cleanServiceGroups()`。这意味着：
- 传给前端的 integration 没有 `url`
- 但服务端 `getServiceWidget()` 返回的 integration **仍然有** `url`（iCal 模式需要用）

#### 12.2.2 前端配置的白名单清洗

传给前端的 widget 数据经过 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L253-L439) L253-L439 的白名单解构：

```javascript
const {
  // all widgets
  fields, hideErrors, highlight, type,

  // calendar
  firstDayInWeek, integrations, maxEvents, showTime, previousDays, view, timezone,

  // sonarr, radarr
  enableQueue,

  // ... 其他 widget 类型的字段
} = widgetData;
```

只有在这个解构列表中的字段才会被保留，传给前端。`integrations` 作为 calendar 类型的字段，**整体被保留**。

#### 12.2.3 integration 的二次处理：仅删除真实地址

在白名单保留的基础上，calendar 的 integrations 会经历 [二次处理](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L622-L642) L622-L642：

```javascript
if (type === "calendar") {
  if (integrations) {
    if (Array.isArray(integrations)) {
      widget.integrations = integrations.map((integration) => {
        if (!integration || typeof integration !== "object") {
          return integration;
        }
        const { url, ...integrationWithoutUrl } = integration;   // 只删除 url
        return integrationWithoutUrl;
      });
    }
  }
  // 日历自有字段：有值才设置
  if (firstDayInWeek) widget.firstDayInWeek = firstDayInWeek;
  if (view) widget.view = view;
  if (maxEvents) widget.maxEvents = maxEvents;
  // ... showTime, previousDays, timezone 等同理
}
```

**关键事实：integration 只删除 `url` 字段，其他所有字段原样保留。**

保留的字段包括（但不限于）：
- `type`：集成类型（sonarr / radarr / ical 等）
- `service_group`：被引用服务所在组
- `service_name`：被引用服务名称
- `name`：集成的显示名称（iCal 模式必填）
- `color`：事件颜色
- `baseUrl`：跳转链接前缀
- `params`：额外请求参数
- `index`：**如果配置了的话**（见 12.4 节讨论）

> 为什么只删 `url`？因为：
> - 服务引用模式（sonarr/radarr）的 URL 来自被引用服务的配置，integration 中本来就没有 url
> - iCal 模式的 url 是敏感信息（可能包含 token），不应暴露给前端
> - 其他字段（service_group、service_name、color 等）都是非敏感的展示或定位参数

### 12.3 前端配置合并：日历字段保留与集成字段覆盖

[calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L70-L86) L70-L86 的合并逻辑：

```javascript
const integrations = useMemo(
  () =>
    widget.integrations
      ?.filter((integration) => integration?.type)
      .map((integration) => ({
        service: dynamic(() => import(`./integrations/${integration.type}.jsx`)),
        widget: { ...widget, ...integration },   // 展开日历 widget，再展开 integration 覆盖
      })) ?? [],
  [widget],
);
```

#### 12.3.1 合并的三个分区

`{ ...日历widget, ...integration }` 的合并结果可以分为三类：

| 分区 | 字段示例 | 来源 | 合并后行为 |
|------|----------|------|------------|
| **日历独有字段** | `firstDayInWeek`, `maxEvents`, `view`, `showTime`, `previousDays`, `timezone`, `integrations` | 日历 widget | 保留（integration 通常不设置这些） |
| **通用重叠字段** | `type`, `service_group`, `service_name`, `index`, `hide_errors`, `fields` | 两者都有 | integration 覆盖日历 widget（定位切换的核心） |
| **集成独有字段** | `color`, `baseUrl`, `params`, `name` | integration | 新增（日历 widget 没有这些） |

#### 12.3.2 合并前后的完整字段对比

以 Sonarr 集成为例：

| 字段 | 日历 widget (合并前) | integration (合并前) | 合并后结果 |
|------|---------------------|---------------------|------------|
| **type** | `"calendar"` | `"sonarr"` | `"sonarr"` ← 被覆盖 |
| **service_group** | 日历所在组（如 `"媒体中心"`） | 被引用服务组（如 `"媒体管理"`） | `"媒体管理"` ← 被覆盖 |
| **service_name** | 日历服务名（如 `"日历面板"`） | 被引用服务名（如 `"Sonarr"`） | `"Sonarr"` ← 被覆盖 |
| **index** | `0`（日历 widget 自身序号） | 通常未设置 | `0` ← 沿用日历的 |
| **hide_errors** | `false`（日历的设置） | 通常未设置 | `false` ← 沿用日历的 |
| **fields** | `null` | 通常未设置 | `null` ← 沿用日历的 |
| **firstDayInWeek** | `"monday"` | 未设置 | `"monday"` ← 保留 |
| **view** | `"monthly"` | 未设置 | `"monthly"` ← 保留 |
| **maxEvents** | `10` | 未设置 | `10` ← 保留 |
| **timezone** | `"Asia/Shanghai"` | 未设置 | `"Asia/Shanghai"` ← 保留 |
| **integrations** | 数组（所有集成） | 未设置 | 数组 ← 保留 |
| **color** | 无 | `"teal"` | `"teal"` ← 新增 |
| **baseUrl** | 无 | `"https://sonarr.example.com"` | `"https://sonarr.example.com"` ← 新增 |
| **params** | 无 | `{ unmonitored: true }` | `{ unmonitored: true }` ← 新增 |

> **注意**：日历面板的字段（firstDayInWeek、view、maxEvents 等）在合并后**仍然保留**在集成组件的 `config` 对象中。虽然 Sonarr/Radarr 集成组件通常不会使用这些字段，但它们是可用的。

### 12.4 序号（index）的边界：能否指定目标 widget？

#### 12.4.1 技术上：可以，但未文档化

integration 的字段除了 `url` 都保留，所以**如果用户在 YAML 中给 integration 加上 `index` 字段，它会被保留并最终覆盖日历 widget 的 index**。

示例：
```yaml
widget:
  type: calendar
  integrations:
    - type: sonarr
      service_group: 媒体管理
      service_name: Sonarr
      index: 1   # 引用 Sonarr 服务的第 2 个 widget（未文档化的用法）
```

合并后 `widget.index = 1`，然后 `formatProxyUrl` 会把 `index=1` 传给代理接口，服务端 `getServiceWidget(group, service, index)` 就能找到对应序号的 widget。

#### 12.4.2 实践中：默认用日历自身的 index

绝大多数场景下，integration 不配置 `index`，所以：
- 合并后 index 沿用日历 widget 自己的 index（通常是 0）
- 这隐含了一个假设：**被引用服务的 widget 序号和日历服务的 widget 序号相同**

对于只有一个 widget 的服务（绝大多数情况），这个假设成立。但如果被引用服务有多个 widget，而日历服务只有一个，想引用被引用服务的第 2 个 widget，就需要显式在 integration 中设置 `index`。

#### 12.4.3 设计意图：简化配置

这是有意的设计简化——服务引用模式下，用户只需要配置 `service_group` 和 `service_name`，就能定位到目标服务。`index` 作为高级参数，不暴露在文档中，避免增加配置复杂度。

### 12.5 type 的双重作用：前端动态加载 vs 服务端类型读取

`type` 字段在前端和服务端扮演完全不同的角色，且**服务端不依赖前端传来的 type**。

#### 12.5.1 前端：type 用于动态组件加载

两处使用 `integration.type`：

**① 决定加载哪个集成组件** ([calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L76-L81) L76-L81)：
```javascript
service: dynamic(() => import(`./integrations/${integration.type}.jsx`))
// type = "sonarr" → 加载 ./integrations/sonarr.jsx
// type = "radarr" → 加载 ./integrations/radarr.jsx
// type = "ical"   → 加载 ./integrations/ical.jsx
```

**② 错误提示中显示服务类型**（如 [sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L40) L40）：
```javascript
<Error error={{ message: `${config.type}: ${error.message ?? error}` }} />
// 显示 "sonarr: xxx error"
```

此外，合并后的 `config.type` 也是集成组件内部逻辑可能使用的标识。

#### 12.5.2 服务端：type 从配置中重新读取

服务端代理**不信任**前端传来的 type。它通过 `group` + `service` + `index` 定位到服务后，从**服务端自身的配置**中读取 type：

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js#L12-L20) L12-L20：
```javascript
const { service, group, index } = req.query;
const serviceWidget = await getServiceWidget(group, service, index);
let type = serviceWidget?.type;   // ← 从服务端配置读取，不是从请求参数读

if (type === "calendar") type = "ical";   // 特殊处理

const widget = widgets[type];
```

**注意**：请求参数中没有 `type`，只有 `group`、`service`、`index`、`endpoint`。服务端完全通过 group/service/index 定位服务，然后从配置中获取 type。

#### 12.5.3 类型边界的两次转换

`type` 的值在链路中发生两次变化，但含义完全不同：

| 转换 | 位置 | 触发原因 | 转换方向 |
|------|------|----------|----------|
| 第一次 | 前端日历组件合并配置 | integration.type 覆盖 widget.type | `"calendar"` → `"sonarr"`（或 `"radarr"`、`"ical"`） |
| 第二次 | 服务端 proxy.js | calendar 类型特殊处理（别名） | `"calendar"` → `"ical"` （仅当 type 是 calendar 时） |

**为什么第二次转换只对 calendar 生效？**

因为 widgets 注册表中：
- `ical: calendar`（iCal 是日历的别名，共享 calendar widget 定义）
- 但 `sonarr`、`radarr` 都是独立注册的 widget，有自己的 mappings 和 proxyHandler

当服务端读取到 type 为 `"sonarr"` 时，直接查找 `widgets["sonarr"]`，不会触发 calendar→ical 的转换。这是 Sonarr/Radarr 日历集成能走通的关键——前端已经通过 service_group/service_name 把请求"转发"给了 Sonarr 服务，服务端看到的就是一个标准的 Sonarr widget 请求。

### 12.6 服务端代理的请求路由（五步定位法）

请求到达 [pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js) 后，经历以下定位步骤：

#### 第一步：group + service + index → 服务配置对象

```javascript
const { service, group, index } = req.query;
const serviceWidget = await getServiceWidget(group, service, index);
let type = serviceWidget?.type;
```

- 输入：`group="媒体管理"`, `service="Sonarr"`, `index=0`
- 输出：Sonarr 服务的第 0 个 widget 配置（含完整的 url、key 等敏感字段）
- type 来自服务端配置，不是前端参数

> 这是服务引用模式的核心：前端通过 `service_group` + `service_name` 把"日历集成"请求转化为对"Sonarr 服务"的请求。

#### 第二步：type → Widget 定义对象

```javascript
if (type === "calendar") type = "ical";   // 仅 calendar 类型重映射
const widget = widgets[type];
```

- 输入：`type = "sonarr"`
- 输出：`widgets["sonarr"]` → Sonarr Widget 定义对象

如果是 iCal 集成，流程不同：
- 前端 type 是 `"ical"`，请求的 group/service 是**日历服务自身**的
- 服务端读取到 type = `"calendar"`（日历服务的 type），然后重映射为 `"ical"`
- `widgets["ical"]` 指向 calendar widget，使用 `calendarProxyHandler`

#### 第三步：endpoint → mapping 配置

```javascript
const mapping = widget?.mappings?.[req.query.endpoint];
// widget.mappings["calendar"] → { endpoint: "calendar", params: [...], ... }
```

Sonarr 的 `calendar` mapping 定义在 [sonarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/widget.js#L61-L65) L61-L65：

```javascript
calendar: {
  endpoint: "calendar",
  params: ["start", "end", "unmonitored", "includeSeries", 
           "includeEpisodeFile", "includeEpisodeImages"],
},
```

#### 第四步：query 参数白名单过滤

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js#L74-L86) L74-L86：

```javascript
if (req.query.query && (mappingParams || optionalParams)) {
  const queryParams = JSON.parse(req.query.query);
  // 只保留 mapping.params 和 optionalParams 中声明的参数名
  const params = [...(mappingParams || []), ...filteredOptionalParams];
  const query = new URLSearchParams(params.map((p) => [p, queryParams[p]]));
  req.query.endpoint = `${req.query.endpoint}?${query}`;
}
```

前端传来的参数（`start`, `end`, `includeSeries` 等）只有在 mapping 的 params 白名单中才会转发到真实 API。

#### 第五步：调用代理处理器

```javascript
return await serviceProxyHandler(req, res, map);
// serviceProxyHandler = genericProxyHandler
// map = undefined (calendar mapping 没有 map 函数)
```

`genericProxyHandler` 执行实际的 HTTP 请求：
- 根据 `{url}/api/v3/{endpoint}?apikey={key}` 模板构造完整 URL
- 注入 API Key 和 Basic Auth
- 发起请求到真实的 Sonarr 服务器
- 数据校验和转换后返回前端

### 12.7 iCal 模式的特殊定位（对比理解）

为了更清晰地理解 Sonarr/Radarr 的服务引用模式，下面对比 iCal（直接 URL 模式）的定位方式：

| 维度 | Sonarr/Radarr（服务引用模式） | iCal（直接 URL 模式） |
|------|-----------------------------|----------------------|
| **数据来源** | 引用其他已配置的 Widget 服务 | 集成项自带 URL |
| **定位方式** | `service_group` + `service_name` + `index` → 找到目标服务 → 读 type | `name` 字段匹配 integration → 取 integration.url |
| **请求中的 group/service** | 被引用服务的（如 "媒体管理/Sonarr"） | 日历服务自身的（如 "媒体中心/日历面板"） |
| **服务端 type** | `"sonarr"`（从被引用服务配置读取） | `"calendar"` → 重映射为 `"ical"`（从日历服务配置读取） |
| **Widget 定义** | `widgets["sonarr"]` | `widgets["ical"]`（即 calendar） |
| **Proxy Handler** | `genericProxyHandler` | `calendarProxyHandler` |
| **endpoint 含义** | Sonarr API 的 endpoint 名（如 "calendar"） | integration 的 `name`（用于在 integrations 数组中查找） |
| **URL 来源** | 被引用服务配置的 `widget.url` | integration 自身的 `url` 字段（服务端可见，前端不可见） |
| **API Key** | 使用被引用服务的 key | 无（或包含在 URL 中） |

iCal 模式的服务端定位逻辑在 [calendar/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/proxy.js#L7-L17) L7-L17：

```javascript
const { group, service, endpoint, index } = req.query;

if (group && service) {
  const widget = await getServiceWidget(group, service, index);  // 找到日历服务的配置
  const integration = widget.integrations?.find((i) => i.name === endpoint);  // endpoint 当 name 用
  if (integration) {
    if (!integration.url) {  // 服务端能看到 integration.url（前端看不到）
      return res.status(403).json({ error: "No integration URL specified" });
    }
    // ... 用 integration.url 发起请求
  }
}
```

### 12.8 完整服务定位链路图

```
┌─ services.yaml 原始配置 ───────────────────────────────────┐
│  （服务端 getServiceWidget 直接读这里）                       │
│                                                            │
│  媒体管理组:                                                 │
│    Sonarr 服务: { widget: { type: sonarr, url, key } }      │
│                                                            │
│  媒体中心组:                                                 │
│    日历面板服务: {                                           │
│      widget: {                                              │
│        type: calendar,                                      │
│        maxEvents: 10,                                       │
│        integrations: [                                      │
│          { type: sonarr,                                    │
│            service_group: 媒体管理,  ← 定位指针              │
│            service_name: Sonarr,     ← 定位指针              │
│            url: ...(iCal 模式才有), ← 服务端可见             │
│            color: teal }                                    │
│        ]                                                    │
│      }                                                      │
│    }                                                        │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ 配置清洗 cleanServiceGroups() ────────────────────────────┐
│  （给前端用的配置，白名单过滤）                               │
│                                                            │
│  - integrations 数组整体保留                                 │
│  - 每个 integration 只删除 url 字段                          │
│  - 日历独有字段（firstDayInWeek, maxEvents, view...）保留    │
│  - 通用字段（type, service_group, service_name, index...）保留│
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ 前端：日历组件 (calendar/component.jsx) ──────────────────┐
│                                                            │
│  对每个 integration:                                        │
│    { ...日历widget, ...integration }                      │
│                                                            │
│  结果：                                                     │
│    - type: sonarr（覆盖 calendar）                          │
│    - service_group: 媒体管理（覆盖日历自身组）                │
│    - service_name: Sonarr（覆盖日历自身名）                   │
│    - index: 0（沿用日历的，除非 integration 设置了 index）   │
│    - firstDayInWeek, maxEvents, view...（保留日历字段）      │
│    - color, baseUrl, params...（integration 新增字段）       │
│                                                            │
│  按 type 动态加载集成组件: ./integrations/sonarr.jsx         │
│  传入 config = 合并后的 widget 对象                           │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ 前端：Sonarr 集成组件 (sonarr.jsx) ──────────────────────┐
│                                                            │
│  useWidgetAPI(config, "calendar", { start, end, ... })    │
│    ↓                                                        │
│  formatProxyUrl() → /api/services/proxy                    │
│    ?group=媒体管理&service=Sonarr&index=0&endpoint=calendar │
│                                                            │
│  ↑ 关键：group/service 是 Sonarr 的，不是日历的！             │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ 服务端：代理入口 (proxy.js) ──────────────────────────────┐
│                                                            │
│  ① getServiceWidget("媒体管理", "Sonarr", 0)               │
│     → 读原始配置，找到 Sonarr 服务的 widget                  │
│     → type = "sonarr"（从服务端配置读取，不来自请求）          │
│                                                            │
│  ② widgets["sonarr"] → Sonarr Widget 定义                   │
│     → proxyHandler = genericProxyHandler                   │
│                                                            │
│  ③ mappings["calendar"] → 映射配置                          │
│     endpoint: "calendar"                                   │
│     params: [start, end, unmonitored, ...]                 │
│                                                            │
│  ④ query 参数白名单过滤                                      │
│                                                            │
│  ⑤ genericProxyHandler                                      │
│     → 用 Sonarr 的 url + key 构造真实请求                     │
│     → 返回日历数据数组                                       │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    前端收到数据，渲染日历事件
```

### 12.9 设计特点与边界总结

#### 12.9.1 配置清洗的精确边界

- **只删 `url`**：integration 只删除真实地址字段，其他字段完整保留
- **两套配置独立**：前端看到的是清洗后的，服务端代理读的是原始配置
- **日历字段保留**：合并后日历的显示配置（view、maxEvents 等）仍然在 config 中可用

#### 12.9.2 序号的隐式假设

- `index` 可以在 integration 中指定（技术上可行），但未文档化
- 默认沿用日历服务自身的 index（通常为 0）
- 隐含假设：被引用服务的 widget 序号与日历服务相同

#### 12.9.3 type 的两种角色

| 角色 | 位置 | 作用 |
|------|------|------|
| 前端 type | integration.type | 动态加载哪个集成组件；错误提示标识 |
| 服务端 type | getServiceWidget 返回的配置.type | 决定用哪个 Widget 定义（mappings、proxyHandler） |

服务端不相信前端传来的 type，始终通过 group/service/index 从配置中重新读取。这是安全设计的一部分——前端无法通过篡改 type 来访问未授权的 Widget 类型。

---

## 十三、配置清洗中的 URL 移除与 baseUrl 传递：跳转链接的统一来源

本节解决一个核心问题：配置清洗中**为什么只删 `url` 而保留 `baseUrl`**，以及日历事件中的跳转链接从哪来、如何拼接、缺失时如何防御。

### 13.1 `url` 与 `baseUrl` 的语义区别

两者虽然都含有 URL，但用途和安全性完全不同：

| 字段 | 语义 | 用途 | 是否含敏感信息 | 所在层级 |
|------|------|------|---------------|----------|
| `url` | 服务的 **API 基地址** | 服务端代理向 Sonarr/Radarr 发起 HTTP 请求 | 是（通常含内网地址、端口、可能带 token） | Widget 顶层（`widget.url`） |
| `baseUrl` | 服务的 **Web 界面地址** | 前端浏览器跳转到 Sonarr/Radarr 的网页 | 否（通常是公网可访问的域名） | integration 内部（`integration.baseUrl`） |

```yaml
# 完整配置示例
- 媒体管理:
    - Sonarr:
        href: http://sonarr:8989          # 浏览器点击服务卡片跳转（前端可用）
        widget:
          url: http://sonarr:8989          # API 基地址（服务端代理用，敏感）
          type: sonarr
          key: YOUR_API_KEY                # API 密钥（服务端代理用，敏感）

- 媒体中心:
    - 日历面板:
        widget:
          type: calendar
          integrations:
            - type: sonarr
              service_group: 媒体管理
              service_name: Sonarr
              baseUrl: https://sonarr.example.com  # Web 界面地址（前端跳转用，非敏感）
```

### 13.2 配置清洗中为何只移除 `url`

#### 13.2.1 `url` 的删除机制

[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L629) L629：

```javascript
const { url, ...integrationWithoutUrl } = integration;
return integrationWithoutUrl;
```

这段代码**只对 integration 中的 `url` 字段生效**，使用解构赋值 + rest 语法将 `url` 剥离。

#### 13.2.2 Widget 顶层的 `url` 也被过滤

Widget 顶层的 `url` 字段不在 [白名单解构](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L256-L439) L256-L439 中（`url`、`key` 等敏感字段都被排除），所以也不会传给前端。

#### 13.2.3 `baseUrl` 不被删除的原因

`baseUrl` 在配置清洗中**没有被删除**，这是有意为之：

1. **`baseUrl` 不是敏感信息**：它只是 Sonarr/Radarr 的公网 Web 地址，不含 API Key 或内网信息
2. **`baseUrl` 是前端专用的**：服务端代理从不使用 `baseUrl`，它仅用于浏览器端跳转链接
3. **`baseUrl` 在 integration 内部**：integration 的字段除了 `url` 都保留，`baseUrl` 在解构 `{ url, ...rest }` 中落在 `rest` 里

#### 13.2.4 字段过滤的完整对比

| 字段 | 在哪一层 | 白名单过滤 | 二次处理 | 最终传给前端 |
|------|----------|-----------|---------|-------------|
| `widget.url` | Widget 顶层 | ❌ 不在白名单中 | — | 不传 |
| `widget.key` | Widget 顶层 | ❌ 不在白名单中 | — | 不传 |
| `integration.url` | integration 内部 | ✅ integrations 整体保留 | ❌ 被删除 | 不传 |
| `integration.baseUrl` | integration 内部 | ✅ integrations 整体保留 | ✅ 保留 | 传递 |

### 13.3 `baseUrl` 的完整传递链路

从 YAML 配置到日历事件中的可点击链接，`baseUrl` 经过以下 4 个阶段：

```
YAML 配置          配置清洗            前端合并             事件 URL 拼接
─────────     ──────────────     ──────────────     ──────────────────
integration:   integration:       config:            event.url:
  baseUrl:       baseUrl:           baseUrl:           "https://sonarr
  "https://      "https://          "https://          .example.com
  sonarr.        sonarr.            sonarr.            /series/
  example.       example.           example.           breaking-bad"
  com"           com"               com"
       │                │                 │                   │
       │  保留          │  合并时新增      │  拼接             │
       ▼                ▼                 ▼                   ▼
  只删 url         ...integration     config.baseUrl    baseUrl + path
  保留 baseUrl     覆盖日历widget     + titleSlug       + titleSlug
```

#### 阶段一：YAML 配置

```yaml
integrations:
  - type: sonarr
    baseUrl: https://sonarr.example.com   # ← 用户显式配置
```

#### 阶段二：配置清洗

[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js#L625-L630) L625-L630：

```javascript
widget.integrations = integrations.map((integration) => {
  const { url, ...integrationWithoutUrl } = integration;
  return integrationWithoutUrl;
  // baseUrl 在 integrationWithoutUrl 中，被保留
});
```

传给前端的 integration 对象：
```javascript
{ type: "sonarr", service_group: "媒体管理", service_name: "Sonarr", baseUrl: "https://sonarr.example.com" }
```

#### 阶段三：前端合并

[calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx#L83) L83：

```javascript
widget: { ...widget, ...integration }
// 日历 widget 没有 baseUrl，integration 有 → 合并后 config.baseUrl = "https://sonarr.example.com"
```

#### 阶段四：事件 URL 拼接

**Sonarr** ([sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx#L32) L32)：

```javascript
url: config?.baseUrl && event.series.titleSlug && `${config.baseUrl}/series/${event.series.titleSlug}`
// "https://sonarr.example.com" + "/series/" + "breaking-bad"
// → "https://sonarr.example.com/series/breaking-bad"
```

**Radarr** ([radarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.jsx#L25) L25)：

```javascript
const url = config?.baseUrl && event.titleSlug && `${config.baseUrl}/movie/${event.titleSlug}`;
// "https://radarr.example.com" + "/movie/" + "inception"
// → "https://radarr.example.com/movie/inception"
```

### 13.4 跳转链接的统一来源模型

日历事件的跳转链接有三种来源，对应三种集成模式：

#### 来源一：Sonarr/Radarr 的 `baseUrl` 拼接

**适用**：服务引用模式（sonarr、radarr、lidarr、readarr）

```
链接 = baseUrl + 资源路径 + titleSlug
```

| 集成类型 | baseUrl 前缀 | 资源路径 | titleSlug 来源 | 完整链接示例 |
|----------|-------------|---------|---------------|------------|
| Sonarr | `config.baseUrl` | `/series/` | `event.series.titleSlug` | `https://sonarr.example/series/breaking-bad` |
| Radarr | `config.baseUrl` | `/movie/` | `event.titleSlug` | `https://radarr.example/movie/inception` |

**关键特点**：
- `baseUrl` 是用户在 integration 中**显式配置**的，不是从服务配置中自动获取的
- 如果不配置 `baseUrl`，日历事件**不可点击**
- `titleSlug` 来自 Sonarr/Radarr API 返回的日历数据

#### 来源二：iCal 事件的 `event.url`

**适用**：直接 URL 模式（ical）

[ical.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/ical.jsx#L143) L143：

```javascript
url: event.url    // 直接取 iCal 数据中的 URL 字段
```

**关键特点**：
- 链接直接来自 iCal 数据源（如 Google Calendar 的事件链接）
- 不需要任何前端配置
- 如果 iCal 事件本身没有 URL，则为 `undefined`，事件不可点击

#### 来源三：无链接

**适用**：以上两种来源都缺失时

- Sonarr/Radarr 未配置 `baseUrl` → `config.baseUrl` 为 `undefined` → 短路求值返回 falsy
- Sonarr/Radarr 的 API 返回事件缺少 `titleSlug` → 短路求值返回 falsy
- iCal 事件没有 `url` 属性 → `event.url` 为 `undefined`

### 13.5 跳转链接的渲染逻辑

[event.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/event.jsx#L39-L55) L39-L55：

```javascript
return event.url ? (
  <a href={event.url} target="_blank" rel="noopener noreferrer">
    {children}
  </a>
) : (
  <div>
    {children}
  </div>
);
```

| `event.url` 值 | 渲染结果 | 行为 |
|----------------|---------|------|
| `"https://sonarr.example/series/breaking-bad"` | `<a>` 标签 | 可点击，新窗口打开 |
| `undefined` | `<div>` 标签 | 不可点击，纯文本展示 |
| `""` (空字符串) | `<div>` 标签 | 不可点击（空字符串是 falsy） |
| `false` (Sonarr 短路结果) | `<div>` 标签 | 不可点击 |

### 13.6 `baseUrl` 为何不能从 `widget.url` 自动派生

一个自然的疑问是：既然 `widget.url` 已经包含 Sonarr 的地址，为什么不自动把它作为 `baseUrl` 传给前端？

**原因**：

1. **安全隔离**：`widget.url` 可能是内网地址（如 `http://192.168.1.100:8989`），不应暴露给前端；而 `baseUrl` 是用户有意配置的公网地址（如 `https://sonarr.example.com`）

2. **地址可能不同**：服务端代理使用的是内网地址（直接访问 Sonarr），而浏览器需要的可能是反向代理后的公网地址（经 CDN/认证）

3. **协议和端口可能不同**：`widget.url` 可能是 `http://sonarr:8989`，而 `baseUrl` 是 `https://sonarr.example.com`（经反向代理加了 TLS）

4. **设计原则**：每个字段只服务于一个明确的用途。`url` 用于服务端代理请求，`baseUrl` 用于前端浏览器跳转，职责分离

### 13.7 跳转链接来源的完整对照

| 集成模式 | 链接来源 | 前端配置 | 缺失时行为 |
|----------|---------|---------|-----------|
| **Sonarr** | `config.baseUrl + "/series/" + event.series.titleSlug` | integration 中的 `baseUrl` | 事件不可点击 |
| **Radarr** | `config.baseUrl + "/movie/" + event.titleSlug` | integration 中的 `baseUrl` | 事件不可点击 |
| **Lidarr** | `config.baseUrl + "/artist/" + event.artist.titleSlug` | integration 中的 `baseUrl` | 事件不可点击 |
| **Readarr** | `config.baseUrl + "/author/" + event.author.titleSlug` | integration 中的 `baseUrl` | 事件不可点击 |
| **iCal** | `event.url`（来自 iCal 数据本身） | 无需配置 | 事件不可点击 |

---

## 十四、相关文件索引

| 文件路径 | 职责 |
|----------|------|
| [src/utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/config/service-helpers.js) | 配置读取、三种服务发现、配置清洗、敏感信息保护 |
| [src/utils/proxy/api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/api-helpers.js) | URL 模板替换、代理 URL 构造、错误脱敏 |
| [src/utils/proxy/use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/use-widget-api.js) | 前端 Hook，SWR 封装 |
| [src/utils/proxy/handlers/generic.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/utils/proxy/handlers/generic.js) | 通用代理处理器，HTTP 请求执行 |
| [src/pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/pages/api/services/proxy.js) | Next.js API 路由，请求路由中枢 |
| [src/widgets/sonarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/widget.js) | Sonarr Widget 定义 |
| [src/widgets/sonarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/sonarr/component.jsx) | Sonarr UI 组件 |
| [src/widgets/radarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/radarr/widget.js) | Radarr Widget 定义 |
| [src/widgets/radarr/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/radarr/component.jsx) | Radarr UI 组件 |
| [src/widgets/calendar/integrations/sonarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.jsx) | Sonarr 日历集成 |
| [src/widgets/calendar/integrations/radarr.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.jsx) | Radarr 日历集成 |
| [src/widgets/calendar/integrations/sonarr.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/sonarr.test.jsx) | Sonarr 日历集成测试 |
| [src/widgets/calendar/integrations/radarr.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/integrations/radarr.test.jsx) | Radarr 日历集成测试 |
| [src/widgets/calendar/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/component.jsx) | 日历主组件（集成加载 + 配置合并） |
| [src/widgets/calendar/event.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/event.jsx) | 日历事件组件（根据 url 渲染为 &lt;a&gt; 或 &lt;div&gt;） |
| [src/widgets/calendar/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/calendar/proxy.js) | 日历专属代理处理器（iCal 模式 URL 直接请求） |
| [docs/widgets/services/calendar.md](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/docs/widgets/services/calendar.md) | 日历 Widget 配置文档 |
| [src/components/services/widget/container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/container.jsx) | Widget 容器（错误拦截 + 字段过滤 + 高亮） |
| [src/components/services/widget/error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/error.jsx) | 错误展示组件（容错格式化 + 可折叠面板） |
| [src/widgets/widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/widgets.js) | Widget 定义注册表 |
| [src/widgets/components.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/components.js) | Widget 组件注册表（懒加载） |
