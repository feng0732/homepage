# Homepage Plex 与媒体服务 Widget 协作代码理解

本文档从代码实现角度梳理 Homepage 项目中 Plex 以及同类媒体服务（Emby、Jellyfin、Tautulli）Widget 的数据获取、字段映射和卡片渲染的完整协作流程。

## 一、整体架构概览

媒体服务 Widget 的协作分为 **5 层**，数据流向为单向链式：

```
配置层 (YAML) → 注册层 (widgets.js + components.js) → 数据获取层 (proxy + useWidgetAPI)
       ↓
字段映射层 (widget.mappings + proxy 内部处理) → 渲染层 (Container + Block + 自定义组件)
```

关键文件索引：

| 层级 | 文件 | 作用 |
|------|------|------|
| 配置解析 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/config/service-helpers.js) | 解析 services.yaml，清洗 widget 字段白名单 |
| Widget 注册 | [widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/widgets.js) | 注册所有 widget 的 api 模板 + proxyHandler + mappings |
| 组件注册 | [components.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/components.js) | 动态 import 所有 widget 的 React 渲染组件 |
| 代理路由 | [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/pages/api/services/proxy.js) | Next.js API 路由，分发请求到具体 proxyHandler |
| 通用代理 | [generic.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/proxy/handlers/generic.js) | 默认 proxyHandler，处理 HTTP 请求、鉴权、map 变换 |
| API 辅助 | [api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/proxy/api-helpers.js) | URL 模板替换、代理 URL 构造 |
| 前端 Hook | [use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/proxy/use-widget-api.js) | 基于 SWR 封装，轮询拉取代理数据 |
| 渲染容器 | [container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget/container.jsx) | Widget 外层容器，处理 fields 过滤、错误隐藏、高亮上下文 |
| 渲染块 | [block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget/block.jsx) | 单个数据块（label + value），支持高亮样式 |
| 服务卡片 | [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/item.jsx) | 服务级卡片，渲染 icon + 标题 + 多个 widget |
| Widget 调度 | [widget.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget.jsx) | 根据 widget.type 从 components.js 查找并渲染 |

---

## 二、Plex Widget 专项分析

### 2.1 Plex Widget 定义：[plex/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/plex/widget.js)

```javascript
const widget = {
  api: "{url}{endpoint}?X-Plex-Token={key}",
  proxyHandler: plexProxyHandler,

  mappings: {
    unified: {
      endpoint: "/",
    },
  },
};
```

**核心要点**：
- `api` 是一个 URL 模板字符串，使用 `{placeholder}` 语法，实际替换在 `formatApiCall()` 中完成
- Plex **不使用** `genericProxyHandler`，而是自定义 `plexProxyHandler`（因为 Plex 返回 XML，且需要多接口聚合）
- `mappings.unified.endpoint = "/"` 看起来是占位符，实际完全由 proxy 内部决定调用哪些真实 Plex API

### 2.2 Plex 数据获取：[plex/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/plex/proxy.js)

这是 Plex Widget 的核心，完成了「多接口聚合 + XML→JSON 转码 + 多级缓存」。

#### 2.2.1 多接口调用时序

```
plexProxyHandler(req, res)
├─ 1. fetchFromPlexAPI("/status/sessions")   → 获取当前播放流数量（每次必调）
├─ 2. 查缓存 libraries
│   └─ 未命中 → fetchFromPlexAPI("/library/sections")   → 获取所有媒体库列表（缓存 6h）
└─ 3. 查缓存 albums/movies/tv
    └─ 未命中 → 遍历 movie/show/artist 类型库，并行调用：
        ├─ /library/sections/{key}/all         → 电影/电视剧
        └─ /library/sections/{key}/albums      → 音乐专辑
        → 累加计数（缓存 10min）
```

#### 2.2.2 XML 转 JSON

Plex API 返回 XML，使用 `xml-js` 的 `xml2json()` 转码。转换后的数据结构：

```
apiData.MediaContainer._attributes.size          ← 流数量
apiData.MediaContainer.Directory[]               ← 库列表数组
  Directory[]._attributes = { key, type, title }
```

注意 `fetchFromPlexAPI` 的请求头：
```javascript
"X-Plex-Container-Start": "0",
"X-Plex-Container-Size": "500",  // 最多拉 500 条，分页控制
```

#### 2.2.3 缓存策略（memory-cache）

使用 `memory-cache` NPM 包，缓存键以 `proxyName + service + index` 区分多实例：

| 缓存键后缀 | 内容 | 过期时间 |
|-----------|------|---------|
| `libraries.{service}.{index}` | 媒体库列表 | 6 小时 |
| `albums.{service}.{index}` | 专辑计数 | 10 分钟 |
| `movies.{service}.{index}` | 电影计数 | 10 分钟 |
| `tv.{service}.{index}` | 剧集计数 | 10 分钟 |
| **streams** | 播放流数量 | **无缓存，每次重新获取** |

#### 2.2.4 最终输出数据结构

代理层返回给前端的是已经聚合好的干净数据：

```javascript
{
  streams: 2,     // 当前活跃播放流
  albums: 150,    // 专辑总数
  movies: 800,    // 电影总数
  tv: 3000        // 剧集/节目总数
}
```

### 2.3 Plex 字段映射

Plex 的字段映射完全在 `plexProxyHandler` **内部完成**，不依赖 `mappings.map` 回调。映射关系表：

| Plex XML 原始字段 | 聚合逻辑 | 输出字段 |
|-------------------|---------|---------|
| `/status/sessions` → `MediaContainer._attributes.size` | 直接读取 | `streams` |
| `/library/sections/{movieKey}/all` → `MediaContainer._attributes[totalSize/size]` | 所有 movie 类型库累加 | `movies` |
| `/library/sections/{showKey}/all` → `MediaContainer._attributes[totalSize/size]` | 所有 show 类型库累加 | `tv` |
| `/library/sections/{artistKey}/albums` → 同上 | 所有 artist 类型库累加 | `albums` |

> **设计意图**：Plex 需要多接口聚合 + XML 特殊处理，因此完全自定义 proxy，放弃了通用 `mappings.map` 机制。

### 2.4 Plex 卡片渲染：[plex/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/plex/component.jsx)

```jsx
export default function Component({ service }) {
  const { widget } = service;

  const { data: plexData, error: plexAPIError } = useWidgetAPI(widget, "unified", {
    refreshInterval: 5000,  // 5 秒轮询
  });

  return (
    <Container service={service}>
      <Block label="plex.streams" value={t("common.number", { value: plexData.streams })} />
      <Block label="plex.albums"  value={t("common.number", { value: plexData.albums })} />
      <Block label="plex.movies"  value={t("common.number", { value: plexData.movies })} />
      <Block label="plex.tv"      value={t("common.number", { value: plexData.tv })} />
    </Container>
  );
}
```

**核心要点**：
1. 只调用一次 `useWidgetAPI(widget, "unified")` → 对应 `mappings.unified`
2. 渲染采用 **4 个 Block 横向排列** 的最基础样式，无自定义复杂 UI
3. 数值使用 `t("common.number", { value })` 做本地化千分位格式化
4. 加载态：`value` 为 `undefined` 时 Block 自动显示骨架脉冲动画（`animate-pulse`）

---

## 三、对比：同类媒体服务的协作差异

Homepage 中有 4 个媒体服务 Widget，实现策略各有不同：

| Widget | API 格式 | Proxy 类型 | 映射方式 | 播放流展示 | 媒体计数展示 |
|--------|---------|-----------|---------|-----------|-------------|
| **Plex** | XML | 自定义 `plexProxyHandler` | Proxy 内部聚合 | ❌ 只显示数量 | ✅ 4 个 Block |
| **Emby** | JSON | 通用 `genericProxyHandler` | `mappings` 声明式 | ✅ 进度条列表 | ✅ 4 个 Block（可选） |
| **Jellyfin** | JSON | 自定义 `jellyfinProxyHandler`（加 Authorization 头） | `mappings` 声明式 + V1/V2 双版本 | ✅ 进度条列表 | ✅ 4 个 Block（可选） |
| **Tautulli** | JSON（Plex 统计增强版） | 通用 `genericProxyHandler` | `mappings` 声明式 | ✅ 进度条列表 | ❌ |

### 3.1 Emby 典型映射模式：[emby/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/emby/widget.js)

```javascript
mappings: {
  Sessions: { endpoint: "Sessions" },
  Count:    { endpoint: "Items/Counts" },
  Unpause: {
    method: "POST",
    endpoint: "Sessions/{sessionId}/Playing/Unpause",
    segments: ["sessionId"],   // ← URL 路径参数白名单
  },
  Pause: { /* 类似 */ },
}
```

前端同时调用 2 个 endpoint：
```javascript
useWidgetAPI(widget, "Sessions", { refreshInterval: 5000 })  // 活跃流
useWidgetAPI(widget, "Count",    { refreshInterval: 60000 }) // 媒体库计数（1min）
```

**对比 Plex**：
- Emby 直接声明 2 个独立 endpoint，前端分别请求
- Plex 将多个 API 聚合为一个 `unified` endpoint，在 proxy 内部合并返回
- Jellyfin/Emby 展示更丰富：播放进度条、播放/暂停控制按钮、转码指示图标

### 3.2 Jellyfin 自定义 Header：[jellyfin/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/widgets/jellyfin/proxy.js)

Jellyfin V2 API 要求特殊的 `Authorization: MediaBrowser ...` 头，因此自定义 proxy：

```javascript
const authHeader = `MediaBrowser Token="${widget.key}", Client="Homepage", Device="Homepage", ...`;
const headers = { Authorization: authHeader };
// 然后走与 genericProxyHandler 相同的 httpProxy + validateWidgetData + map 流程
```

---

## 四、数据获取层深入：从 Hook 到代理 API

### 4.1 useWidgetAPI Hook：[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/proxy/use-widget-api.js)

这是连接 React 组件与后端代理的核心 Hook，基于 `swr` 库实现：

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;  // 轮询间隔
  }
  let url = formatProxyUrl(widget, ...options);
  // endpoint 为空串时返回 null → 不发送请求（用于 disable 场景）
  if (options[0] === "") url = null;
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

### 4.2 代理 URL 构造：[api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/proxy/api-helpers.js)

```javascript
formatProxyUrl(widget, "unified")
  → getURLSearchParams(widget, "unified")
  → `/api/services/proxy?group=Media&service=Plex&index=0&endpoint=unified`
```

前端 **从不直接调用 Plex 等真实 API**，所有请求都通过 `/api/services/proxy` 中转，这是出于：
- **CORS 规避**：浏览器不直接跨域
- **鉴权安全**：`key/token` 只在服务端使用，不下发到前端
- **数据加工**：聚合、转码（XML→JSON）、缓存

### 4.3 代理分发路由：[pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/pages/api/services/proxy.js)

关键分发逻辑：

```
请求到达 → 从 query 取 group/service/index
        → getServiceWidget() 从配置中取出 widget 配置（含 type, url, key, fields...）
        → widgets[type] 取出对应 widget 定义
        → widget.proxyHandler || genericProxyHandler
        → mappings[endpoint] 存在：
             1. 替换 method/body/headers/segments
             2. 将 endpoint 从逻辑名替换为真实 API 路径
             3. 调用具体 proxyHandler(req, res, mapping.map)
        → 否则：如果 allowedEndpoints(正则)匹配也可以通过
```

**mappings 机制详解**：

| mappings 字段 | 用途 |
|--------------|------|
| `endpoint` | 真实 API 路径，替换 query 中的逻辑 endpoint 名 |
| `method` | 强制 HTTP 方法（如 POST） |
| `segments` | URL 路径参数白名单（`/Sessions/{sessionId}/...`），防路径穿越 |
| `params` | 必传 query 参数白名单 |
| `optionalParams` | 可选 query 参数白名单 |
| `map` | 响应数据变换函数 `(data) => newData`，在 proxyHandler 返回前调用 |
| `headers` | 自定义请求头 |
| `body` | POST 请求体 |

### 4.4 配置字段白名单：[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/utils/config/service-helpers.js) `cleanServiceGroups()`

这是一个**非常关键但容易忽略**的安全层。从 YAML 解析出的 widget 配置不会全部下发到前端，而是通过**白名单提取**：

```javascript
cleanedService.widgets = cleanedService.widgets.map((widgetData, index) => {
  const {
    type, fields, hideErrors, highlight,      // 通用
    enableBlocks, enableNowPlaying,            // emby/jellyfin 专用
    enableUser, expandOneStreamToTwoRows, ...  // 流媒体播放专用
    ...  // 100+ 分 widget 类型的白名单字段
  } = widgetData;

  const widget = {
    type, fields, hide_errors: hideErrors,
    service_name, service_group, index,
    // 条件性加入的字段
  };
  if (["emby", "jellyfin"].includes(type)) {
    if (enableMediaControl !== undefined) widget.enableMediaControl = ...
  }
  // ... 每种 widget 类型有其专属的配置拷贝逻辑
  return widget;
});
```

**设计意义**：
- `url`、`key`（API Token）、`username`、`password` 等敏感字段**绝对不下发前端**
- 前端只能拿到展示/行为配置，无法获取任何凭证

---

## 五、卡片渲染层：Container + Block 体系

### 5.1 Widget 渲染调度：[components/services/widget.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget.jsx)

```javascript
export default function Widget({ widget, service }) {
  const ServiceWidget = components[widget.type];  // 从 components.js 按 type 查找
  const fullService = { ...service, widget };
  return <ServiceWidget service={fullService} />;  // 交给具体 widget 组件
}
```

调用链：[item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/item.jsx#L188) → `service.widgets.map(w => <Widget widget={w} service={service} />)`

### 5.2 Container：[container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget/container.jsx)

Container 是所有 Widget 组件的外层，提供了 3 个横切能力：

#### 5.2.1 fields 字段过滤

用户可在 YAML 中配置 `fields: ["plex.movies", "streams"]` 只显示特定 Block：
```javascript
visibleChildren = childrenArray.filter((child) =>
  fields.some((field) => {
    let fullField = field.includes(".") ? field : `${type}.${field}`;
    // 匹配 child.props.label 或 child.props.field
    return fullField === (child.props.field || child.props.label);
  })
);
```

匹配规则：
- `plex.movies` 完全匹配 → `label="plex.movies"` 的 Block 显示
- `movies` 自动补全为 `plex.movies`
- 对有别名的 widget（如 `overseerr`→`seerr`）会再做一次别名替换匹配

#### 5.2.2 错误隐藏

```javascript
if (error) {
  if (settings.hideErrors || service.widget.hide_errors) return null;
  return <Error service={service} error={error} />;
}
```

#### 5.2.3 高亮上下文

通过 `BlockHighlightContext.Provider` 下发高亮配置给内部所有 Block。

### 5.3 Block：[block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/198-homepage/src/components/services/widget/block.jsx)

最小渲染单元，结构为：

```
┌──────────────────────────┐  ← bg-theme-200/50 rounded-sm
│  1,234                   │  ← value: font-thin text-sm
│  MOVIES                  │  ← label(t()): font-bold text-xs uppercase
└──────────────────────────┘
```

特殊状态：
- `value === undefined` → 显示 `-` + 骨架 `animate-pulse`（加载中）
- 高亮：根据 `highlightConfig` 评估匹配，加高亮 class，可 `valueOnly` 只高亮数值

---

## 六、完整数据流转时序图（以 Plex 为例）

```
用户浏览器                                  Next.js 服务端                                Plex 服务器
   │                                            │                                            │
   │ ① 加载首页，services 配置下发（含 widget）  │                                            │
   │   （字段已清洗，无 url/key）                │                                            │
   │                                            │                                            │
   │ ② useWidgetAPI(widget, "unified")          │                                            │
   │    GET /api/services/proxy                 │                                            │
   │    ?group=Media&service=Plex               │                                            │
   │    &index=0&endpoint=unified               │───► handler(req, res)                       │
   │                                            │    1. getServiceWidget() 取完整配置         │
   │                                            │       （含 url/key，敏感字段在服务端）       │
   │                                            │    2. widgets["plex"].proxyHandler          │
   │                                            │       = plexProxyHandler                    │
   │                                            │    3. mappings.unified.endpoint="/"         │
   │                                            │       → plexProxyHandler 内部忽略           │
   │                                            │    4. plexProxyHandler(req, res)            │
   │                                            │       ├─ fetchFromPlexAPI("/status/...")─────┼───► GET /status/sessions
   │                                            │       │   headers: X-Plex-Token={key}        │◄──┐ XML Response
   │                                            │       │   xml2json → size=2                  │   │
   │                                            │       ├─ 查 libraries 缓存                  │   │
   │                                            │       │  (未命中) GET /library/sections ──────┼───► GET /library/sections
   │                                            │       │   → 缓存 6h                          │◄──┐
   │                                            │       ├─ 查 counts 缓存                      │   │
   │                                            │       │  (未命中) 并行 N 个 GET /sections/...─┼───► ...
   │                                            │       │   → 缓存 10min                       │◄──┘
   │                                            │       └─ 聚合 {streams, albums, movies, tv} │
   │                                            │                                            │
   │◄──────────── 200 JSON ────────────────────┤                                            │
   │  { streams:2, movies:800, ... }            │                                            │
   │                                            │                                            │
   │ ③ SWR 收到 data，触发重渲染                │                                            │
   │    Container + 4 个 Block                  │                                            │
   │    显示 4 个数值块                          │                                            │
   │                                            │                                            │
   │ ④ 每 5s 刷新一次（refreshInterval:5000）    │──────── 同 ②，只重取 streams（缓存命中）─────┤
```

---

## 七、Plex "不显眼"的代码层面原因总结

| 维度 | Plex | Emby/Jellyfin/Tautulli |
|------|------|----------------------|
| **流展示** | ❌ 只显示 streams 数字 | ✅ 有播放进度条、播放控制、标题滚动显示 |
| **调用次数** | 1 次 `unified` 请求（proxy 内部聚合） | 2 次独立请求（Sessions + Count） |
| **Proxy 复杂度** | 高（自定义 XML 处理 + 4 级缓存） | 低（通用 proxy 即可） |
| **配置项** | 少（仅 fields/hideErrors 通用项） | 多（enableBlocks/enableNowPlaying/enableUser/showEpisodeNumber 等 6+） |
| **渲染复杂度** | 4 Block 基础组件 | Block 列表 + 自定义 SessionEntry 组件 + 播放控制按钮 |
| **Proxy 内部聚合** | ✅ 是 | ❌ 否，各自独立 endpoint |

从代码角度看，Plex Widget 的"不显眼"是**故意为之的设计选择**：
- Plex 官方 API 返回 XML 且需要多库聚合，在 proxy 层做了大量复杂度隐藏
- 组件展示层保持了最简形式，与 Emby/Jellyfin 丰富的播放流 UI 形成鲜明对比
- 如果要让 Plex 也展示活跃播放流详情（类似 Tautulli），需要：
  1. 在 `plexProxyHandler` 中解析 `/status/sessions` 的完整 Session 详情（当前只用了 `size`）
  2. 在 component 中增加类似 Emby 的 `SessionEntry` 渲染逻辑
  3. 注意 Plex XML 的 Session 结构与 Emby JSON 结构不同，需要新的字段映射

---

## 八、关键调用链速查表

### 8.1 配置加载链

```
services.yaml
  → servicesFromConfig() / servicesFromDocker() / servicesFromKubernetes()
    → parseServicesToGroups()
      → cleanServiceGroups()  ← 字段白名单清洗，widget 配置结构化
        → 前端拿到 service + widgets[]
```

### 8.2 数据请求链

```
Component (e.g. plex/component.jsx)
  → useWidgetAPI(widget, endpoint, { refreshInterval })
    → formatProxyUrl() → /api/services/proxy?...&endpoint=XXX
      → SWR useSWR() → 自动轮询 + 缓存
        → pages/api/services/proxy.js handler
          → getServiceWidget() → 取服务端完整配置（含 url/key）
          → widgets[type] → 取 widget 定义
          → mappings[endpoint] → 逻辑名→真实路径
          → proxyHandler(req, res, map) → 真实 API 调用
            → httpProxy() → 实际 HTTP 请求
            → validateWidgetData() → 数据校验
            → map?.(data) → 可选字段变换
          ← res.send(data)
```

### 8.3 渲染链

```
components/services/item.jsx
  → service.widgets.map(widget)
    → components/services/widget.jsx <Widget widget={w} service={service}>
      → components[widget.type] → e.g. plex/component.jsx
        → Container（字段过滤 + 错误处理 + 高亮）
          → Block × N （或自定义组件，如 Emby 的 SessionEntry）
            → t(label) 国际化 + t("common.number") 本地化
```
