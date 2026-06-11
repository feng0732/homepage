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

## 十二、相关文件索引

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
| [src/components/services/widget/container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/container.jsx) | Widget 容器（错误拦截 + 字段过滤 + 高亮） |
| [src/components/services/widget/error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/components/services/widget/error.jsx) | 错误展示组件（容错格式化 + 可折叠面板） |
| [src/widgets/widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/widgets.js) | Widget 定义注册表 |
| [src/widgets/components.js](file:///d:/fz/0601/solo-dogfeeding/code/197-homepage/src/widgets/components.js) | Widget 组件注册表（懒加载） |
