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

| 层级 | 位置 | 作用 |
|------|------|------|
| 配置解析 | `src/utils/config/service-helpers.js` | 解析 services.yaml，清洗 widget 字段白名单 |
| Widget 注册 | `src/widgets/widgets.js` | 注册所有 widget 的 api 模板 + proxyHandler + mappings |
| 组件注册 | `src/widgets/components.js` | 动态 import 所有 widget 的 React 渲染组件 |
| 代理路由 | `src/pages/api/services/proxy.js` | Next.js API 路由，分发请求到具体 proxyHandler |
| 通用代理 | `src/utils/proxy/handlers/generic.js` | 默认 proxyHandler，处理 HTTP 请求、鉴权、map 变换 |
| API 辅助 | `src/utils/proxy/api-helpers.js` | URL 模板替换、代理 URL 构造 |
| 前端 Hook | `src/utils/proxy/use-widget-api.js` | 基于 SWR 封装，轮询拉取代理数据 |
| 渲染容器 | `src/components/services/widget/container.jsx` | Widget 外层容器，处理 fields 过滤、错误隐藏、高亮上下文 |
| 渲染块 | `src/components/services/widget/block.jsx` | 单个数据块（label + value），支持高亮样式 |
| 服务卡片 | `src/components/services/item.jsx` | 服务级卡片，渲染 icon + 标题 + 多个 widget |
| Widget 调度 | `src/components/services/widget.jsx` | 根据 widget.type 从 components.js 查找并渲染 |

---

## 二、Plex Widget 专项分析

### 2.1 Plex Widget 定义：`src/widgets/plex/widget.js`

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

### 2.2 Plex 数据获取：`src/widgets/plex/proxy.js`

这是 Plex Widget 的核心，完成了「多接口聚合 + XML→JSON 转码 + 多级缓存」。

#### 2.2.1 多接口调用时序

```
plexProxyHandler(req, res)                           ← proxy.js L64
├─ 1. fetchFromPlexAPI("/status/sessions")           ← proxy.js L75，获取当前播放会话（每次必调）
├─ 2. 查缓存 libraries                               ← proxy.js L87
│   └─ 未命中 → fetchFromPlexAPI("/library/sections") ← proxy.js L90，获取所有媒体库列表（缓存 6h）
└─ 3. 查缓存 albums/movies/tv                        ← proxy.js L97-L99
    └─ 未命中 → 遍历 movie/show/artist 类型库，并行调用：
        ├─ /library/sections/{key}/all                ← proxy.js L109，电影/电视剧
        └─ /library/sections/{key}/albums             ← proxy.js L110，音乐专辑
        → 累加计数（缓存 10min）
```

#### 2.2.2 XML 转 JSON

Plex API 返回 XML，在 `fetchFromPlexAPI`（proxy.js L35-L62）中通过 `xml-js` 的 `xml2json()` 转码，再 `JSON.parse` 得到对象。转换后的数据结构：

```
apiData.MediaContainer._attributes.size          ← 会话数量（字符串类型）
apiData.MediaContainer.Directory[]               ← 库列表数组
  Directory[]._attributes = { key, type, title }
```

注意 `fetchFromPlexAPI` 的请求头（proxy.js L44-L48）：
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
| **streams** | 播放会话数量 | **无缓存，每次重新获取** |

#### 2.2.4 最终输出数据结构

代理层返回给前端的是已经聚合好的数据（proxy.js L130-L135）：

```javascript
{
  streams: "2",   // 字符串，来自 xml2json 的 _attributes.size（proxy.js L84）
  albums: 150,    // 数字，经 parseInt 转换（proxy.js L114）
  movies: 800,    // 数字，经 parseInt 转换（proxy.js L114）
  tv: 3000        // 数字，经 parseInt 转换（proxy.js L114）
}
```

> ⚠️ **类型不一致**：`streams` 直接赋值 `apiData.MediaContainer._attributes.size`，而 XML 属性在 `xml2json({ compact: true })` 后始终为字符串；`albums/movies/tv` 则通过 `parseInt(..., 10)` 转为数字。当 Plex 无活跃会话时 `apiData.MediaContainer` 可能不存在，`streams` 将保持 `undefined`。

### 2.3 Plex 字段映射

Plex 的字段映射完全在 `plexProxyHandler` **内部完成**，不依赖 `mappings.map` 回调。映射关系表：

| 输出字段 | API 来源 | 原始 XML 属性 | 取值方式 | 类型 |
|---------|---------|-------------|---------|------|
| `streams` | `/status/sessions` | `MediaContainer._attributes.size` | 直接赋值（proxy.js L84） | **string** 或 `undefined` |
| `movies` | `/library/sections/{movieKey}/all` | `MediaContainer._attributes[totalSize/size]` | `parseInt` 后累加（proxy.js L114） | **number** |
| `tv` | `/library/sections/{showKey}/all` | 同上 | `parseInt` 后累加 | **number** |
| `albums` | `/library/sections/{artistKey}/albums` | 同上 | `parseInt` 后累加 | **number** |

`sizeProp` 的选择逻辑（proxy.js L113）：Plex API 在某些版本返回 `totalSize`，某些版本返回 `size`，代码优先取 `totalSize`，不存在时回退到 `size`。

> **设计意图**：Plex 需要多接口聚合 + XML 特殊处理，因此完全自定义 proxy，放弃了通用 `mappings.map` 机制。

### 2.4 Plex 卡片渲染：`src/widgets/plex/component.jsx`

```jsx
export default function Component({ service }) {
  const { widget } = service;

  const { data: plexData, error: plexAPIError } = useWidgetAPI(widget, "unified", {
    refreshInterval: 5000,  // 5 秒轮询
  });

  if (plexAPIError) {
    return <Container service={service} error={plexAPIError} />;
  }

  if (!plexData) {
    return (
      <Container service={service}>
        <Block label="plex.streams" />
        <Block label="plex.albums" />
        <Block label="plex.movies" />
        <Block label="plex.tv" />
      </Container>
    );
  }

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
3. 数值使用 `t("common.number", { value })` 做本地化千分位格式化（该函数会自动处理字符串到数字的转换，因此 `streams` 为字符串时不会出错）
4. 加载态：`value` 为 `undefined` 时 Block 自动显示骨架脉冲动画（`animate-pulse`）
5. 三种渲染状态：error → Container 仅显示错误；无数据 → Block 无 value（脉冲加载）；有数据 → 正常显示

---

## 三、对比：同类媒体服务的协作差异

Homepage 中有 4 个媒体服务 Widget，实现策略各有不同：

| Widget | API 格式 | Proxy 类型 | 映射方式 | 播放流展示 | 媒体计数展示 |
|--------|---------|-----------|---------|-----------|-------------|
| **Plex** | XML | 自定义 `plexProxyHandler` | Proxy 内部聚合 | ❌ 只显示数量 | ✅ 4 个 Block |
| **Emby** | JSON | 通用 `genericProxyHandler` | `mappings` 声明式 | ✅ 进度条列表 | ✅ 4 个 Block（可选） |
| **Jellyfin** | JSON | 自定义 `jellyfinProxyHandler`（加 Authorization 头） | `mappings` 声明式 + V1/V2 双版本 | ✅ 进度条列表 | ✅ 4 个 Block（可选） |
| **Tautulli** | JSON（Plex 统计增强版） | 通用 `genericProxyHandler` | `mappings` 声明式 | ✅ 进度条列表 | ❌ |

### 3.1 Emby 典型映射模式：`src/widgets/emby/widget.js`

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

### 3.2 Jellyfin 自定义 Header：`src/widgets/jellyfin/proxy.js`

Jellyfin V2 API 要求特殊的 `Authorization: MediaBrowser ...` 头，因此自定义 proxy：

```javascript
const authHeader = `MediaBrowser Token="${widget.key}", Client="Homepage", Device="Homepage", ...`;
const headers = { Authorization: authHeader };
// 然后走与 genericProxyHandler 相同的 httpProxy + validateWidgetData + map 流程
```

---

## 四、数据获取层深入：从 Hook 到代理 API

### 4.1 useWidgetAPI Hook：`src/utils/proxy/use-widget-api.js`

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

### 4.2 代理 URL 构造：`src/utils/proxy/api-helpers.js`

```
formatProxyUrl(widget, "unified")
  → getURLSearchParams(widget, "unified")
  → /api/services/proxy?group=Media&service=Plex&index=0&endpoint=unified
```

前端 **从不直接调用 Plex 等真实 API**，所有请求都通过 `/api/services/proxy` 中转，这是出于：
- **CORS 规避**：浏览器不直接跨域
- **鉴权安全**：`key/token` 只在服务端使用，不下发到前端
- **数据加工**：聚合、转码（XML→JSON）、缓存

### 4.3 代理分发路由：`src/pages/api/services/proxy.js`

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

### 4.4 配置字段白名单：`src/utils/config/service-helpers.js` `cleanServiceGroups()`

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

### 5.1 Widget 渲染调度：`src/components/services/widget.jsx`

```javascript
export default function Widget({ widget, service }) {
  const ServiceWidget = components[widget.type];  // 从 components.js 按 type 查找
  const fullService = { ...service, widget };
  return <ServiceWidget service={fullService} />;  // 交给具体 widget 组件
}
```

调用链：`src/components/services/item.jsx` L188 → `service.widgets.map(w => <Widget widget={w} service={service} />)`

### 5.2 Container：`src/components/services/widget/container.jsx`

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

### 5.3 Block：`src/components/services/widget/block.jsx`

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
   │    &index=0&endpoint=unified               │───► proxy.js handler(req, res)             │
   │                                            │    1. getServiceWidget() 取完整配置         │
   │                                            │       （含 url/key，敏感字段在服务端）       │
   │                                            │    2. widgets["plex"].proxyHandler          │
   │                                            │       = plexProxyHandler                    │
   │                                            │    3. mappings.unified.endpoint="/"         │
   │                                            │       → plexProxyHandler 内部忽略           │
   │                                            │    4. plexProxyHandler(req, res)            │
   │                                            │       ├─ fetchFromPlexAPI("/status/...")─────┼───► GET /status/sessions
   │                                            │       │   headers: X-Plex-Token={key}        │◄──┐ XML Response
   │                                            │       │   xml2json → size="2" (string)       │   │
   │                                            │       ├─ 查 libraries 缓存                  │   │
   │                                            │       │  (未命中) GET /library/sections ──────┼───► GET /library/sections
   │                                            │       │   → 缓存 6h                          │◄──┐
   │                                            │       ├─ 查 counts 缓存                      │   │
   │                                            │       │  (未命中) 并行 N 个 GET /sections/...─┼───► ...
   │                                            │       │   → parseInt 后累加, 缓存 10min      │◄──┘
   │                                            │       └─ 聚合 {streams, albums, movies, tv} │
   │                                            │                                            │
   │◄──────────── 200 JSON ────────────────────┤                                            │
   │  { streams:"2", movies:800, ... }          │                                            │
   │                                            │                                            │
   │ ③ SWR 收到 data，触发重渲染                │                                            │
   │    Container + 4 个 Block                  │                                            │
   │    t("common.number") 格式化显示            │                                            │
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
| **输出类型** | `streams` 为 string，其余为 number | 全部为 JSON 原生类型 |

从代码角度看，Plex Widget 的"不显眼"是**设计选择 + 技术限制的叠加**：
- Plex 官方 API 返回 XML 且需要多库聚合，在 proxy 层做了大量复杂度隐藏
- `/status/sessions` 虽然返回完整的 Session 列表，但 proxy 只取了 `_attributes.size`（活跃会话数量），丢弃了所有播放详情
- 组件展示层保持了最简形式，与 Emby/Jellyfin 丰富的播放流 UI 形成鲜明对比
- 如果要让 Plex 也展示活跃播放流详情（类似 Tautulli），需要：
  1. 在 `plexProxyHandler` 中解析 `/status/sessions` 的完整 Session 详情（当前只用了 `_attributes.size`，见 proxy.js L84）
  2. 在 component 中增加类似 Emby 的 `SessionEntry` 渲染逻辑
  3. 注意 Plex XML 的 Session 结构（`Video._attributes`）与 Emby JSON 结构（`NowPlayingItem`）不同，需要新的字段映射

---

## 八、关键调用链速查表

### 8.1 配置加载链

```
services.yaml
  → service-helpers.js: servicesFromConfig() / servicesFromDocker() / servicesFromKubernetes()
    → service-helpers.js: parseServicesToGroups()
      → service-helpers.js: cleanServiceGroups()  ← 字段白名单清洗，widget 配置结构化
        → 前端拿到 service + widgets[]
```

### 8.2 数据请求链

```
src/widgets/plex/component.jsx
  → use-widget-api.js: useWidgetAPI(widget, endpoint, { refreshInterval })
    → api-helpers.js: formatProxyUrl() → /api/services/proxy?...&endpoint=XXX
      → use-widget-api.js: SWR useSWR() → 自动轮询 + 缓存
        → src/pages/api/services/proxy.js: handler
          → service-helpers.js: getServiceWidget() → 取服务端完整配置（含 url/key）
          → widgets.js: widgets[type] → 取 widget 定义
          → widget.mappings[endpoint] → 逻辑名→真实路径
          → proxyHandler(req, res, map) → 真实 API 调用
            → http.js: httpProxy() → 实际 HTTP 请求
            → validate-widget-data.js: validateWidgetData() → 数据校验
            → map?.(data) → 可选字段变换
          ← res.send(data)
```

### 8.3 渲染链

```
src/components/services/item.jsx
  → service.widgets.map(widget)
    → src/components/services/widget.jsx: <Widget widget={w} service={service}>
      → components.js: components[widget.type] → e.g. src/widgets/plex/component.jsx
        → container.jsx: Container（字段过滤 + 错误处理 + 高亮）
          → block.jsx: Block × N （或自定义组件，如 Emby 的 SessionEntry）
            → next-i18next: t(label) 国际化 + t("common.number") 本地化
```
