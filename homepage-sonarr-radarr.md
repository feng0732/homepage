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

## 十一、相关文件索引

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
| [src/widgets/widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/widgets.js) | Widget 定义注册表 |
| [src/widgets/components.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/components.js) | Widget 组件注册表（懒加载） |
