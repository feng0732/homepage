# SWR 刷新与数据流分析

## 概述

本项目是一个基于 Next.js 的主页应用，使用 SWR (stale-while-revalidate) 作为前端数据获取和缓存策略。本文档梳理了服务发现、缓存键生成、轮询更新与页面重绘之间的完整数据流关系。

---

## 一、SWR 全局配置

### 1.1 顶层配置

全局 SWR 配置在 [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/_app.jsx#L75-L79) 中定义，提供默认 fetcher：

```jsx
<SWRConfig
  value={{
    fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
  }}
>
```

### 1.2 页面级 Fallback 配置

在首页 [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L186) 中，通过 `getStaticProps` 预获取数据并注入 SWR fallback 缓存，实现 SSR 首屏渲染：

```jsx
<SWRConfig value={{ fallback, fetcher: ... }}>
```

fallback 数据包含：
- `/api/services` - 服务列表
- `/api/bookmarks` - 书签列表
- `/api/widgets` - 组件列表
- `/api/hash` - 配置哈希

---

## 二、服务发现机制

服务发现是数据流的起点，决定了哪些服务和 widget 需要被展示和监控。

### 2.1 三种服务来源

在 [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/api-response.js#L158-L256) 的 `servicesResponse()` 函数中，合并三种服务来源：

| 来源 | 描述 | 核心文件 |
|------|------|----------|
| 配置文件 | 从 `services.yaml` 静态配置读取 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/service-helpers.js#L53-L61) `servicesFromConfig()` |
| Docker 发现 | 通过 Docker API 扫描容器标签 `homepage.*` | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/service-helpers.js#L63-L170) `servicesFromDocker()` |
| Kubernetes 发现 | 扫描 Ingress / TraefikIngress / HTTPRoute 资源的注解 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/service-helpers.js#L172-L229) `servicesFromKubernetes()` |

### 2.2 服务发现调用时机

服务发现**仅在服务端**执行，触发时机包括：

1. **构建时**：`getStaticProps` 在构建时预获取服务列表
2. **API 请求时**：每次调用 `/api/services` 都会重新执行服务发现
3. **配置变更时**：通过 hash 检测配置文件变化，触发 ISR 重新验证

### 2.3 服务数据清洗

在 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/service-helpers.js#L231-L708) 的 `cleanServiceGroups()` 中，对服务数据进行清洗和白名单过滤，只将安全的字段发送到前端，避免敏感信息（如密码、API Key）泄露。

---

## 三、缓存键（Cache Key）生成机制

SWR 使用 URL 字符串作为缓存键。不同类型的数据有不同的缓存键生成规则。

### 3.1 基础数据缓存键

| 数据类型 | 缓存键格式 | 定义位置 |
|----------|------------|----------|
| 服务列表 | `/api/services` | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L226) |
| 书签列表 | `/api/bookmarks` | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L227) |
| Widget 列表 | `/api/widgets` | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L228) |
| 配置验证 | `/api/validate` | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L100) |
| 配置哈希 | `/api/hash` | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L102) |

### 3.2 Widget API 缓存键

Widget 数据通过代理 API 获取，缓存键由 [formatProxyUrl()](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/proxy/api-helpers.js#L43-L49) 生成：

```javascript
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  if (queryParams) {
    params.append("query", JSON.stringify(queryParams));
  }
  return `/api/services/proxy?${params.toString()}`;
}
```

生成的缓存键格式：
```
/api/services/proxy?group=<group>&service=<service>&index=<index>&endpoint=<endpoint>[&query=<json>]
```

关键参数：
- `group` - 服务所属组名
- `service` - 服务名称
- `index` - 同服务下多个 widget 的索引
- `endpoint` - 具体 API 端点（映射后的）
- `query` - 附加查询参数（JSON 序列化）

### 3.3 状态类缓存键

| 状态类型 | 缓存键格式 | 定义位置 |
|----------|------------|----------|
| Ping 状态 | `/api/ping?groupName=xxx&serviceName=xxx` | [ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/components/services/ping.jsx#L6-L8) |
| Docker 状态 | `/api/docker/status/<container>/<server>` | [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/components/services/status.jsx#L7) |
| Docker 统计 | `/api/docker/stats/<container>/<server>` | [docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/docker/component.jsx#L17) |
| 站点监控 | `/api/siteMonitor?...` | [site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/components/services/site-monitor.jsx) |

### 3.4 缓存键的唯一性设计

每个 widget 实例通过 `group + service + index + endpoint` 的组合确保缓存键唯一。即使同一服务下有多个相同类型的 widget，也会因为 `index` 不同而拥有独立的缓存。

---

## 四、轮询更新机制

SWR 通过 `refreshInterval` 选项实现周期性数据刷新。

### 4.1 useWidgetAPI Hook

统一的 widget 数据获取 Hook 在 [use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/proxy/use-widget-api.js)：

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

关键特性：
- 从 options 中提取 `refreshInterval` 传递给 SWR
- 当 endpoint 为空字符串时，设置 `url = null`，SWR 将跳过请求
- 将 API 返回数据中的 `error` 提升为顶层错误

### 4.2 各类数据刷新间隔

| 数据类型 | 默认刷新间隔 | 配置位置 |
|----------|-------------|----------|
| Ping 状态 | 30000ms (30s) | [ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/components/services/ping.jsx#L7) |
| Sonarr/Radarr 等 | 无（使用 SWR 默认） | 各 widget component |
| Jellyfin 播放中 | 5000ms (5s) | [jellyfin/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/jellyfin/component.jsx) |
| Glances 指标 | 可变，有默认值 | [glances/metrics/*.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/glances/metrics/) |
| Prometheus Metric | 10000ms (10s) | [prometheusmetric/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/prometheusmetric/component.jsx#L55) |
| iFrame | 可配置，最小 1000ms | [iframe/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/iframe/component.jsx) |

### 4.3 条件刷新

某些 widget 根据状态动态调整刷新间隔：

**Jellyfin/Emby 示例**：
- 播放中：5000ms 刷新（实时更新播放信息）
- 未播放：60000ms 刷新（降低频率）

### 4.4 SWR 内置刷新策略

除了显式的 `refreshInterval`，SWR 还提供以下自动刷新机制：

1. **窗口聚焦刷新**：当浏览器标签页重新获得焦点时，自动重新验证数据
2. **网络恢复刷新**：网络从离线恢复到在线时自动刷新
3. **组件挂载刷新**：组件首次挂载时刷新

---

## 五、页面重绘流程

### 5.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          服务端 (Node.js)                           │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────────────┐   │
│  │ services.  │→│ 服务发现模块  │→ │  servicesResponse()      │   │
│  │ yaml等    │   │ (Docker/K8s) │   │  合并、排序、清洗       │   │
│  └──────────┘   └──────────────┘   └───────────┬──────────────┘   │
│                                                  │                  │
│                                        /api/services               │
│                                        /api/services/proxy        │
│                                        /api/ping 等 API           │
└──────────────────────────────────────────┬─────────────────────────┘
                                           │ HTTP 请求
┌──────────────────────────────────────────▼─────────────────────────┐
│                         前端 (React 组件)                          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                    SWR 缓存层                              │    │
│  │  • key: URL 字符串                                        │    │
│  │  • value: data/error/isValidating                         │    │
│  │  • fallback: SSR 预填充数据                               │    │
│  └───────────────────────┬───────────────────────────────────┘    │
│                          │                                        │
│          ┌───────────────┼───────────────┐                        │
│          ▼               ▼               ▼                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                  │
│  │ Widget 组件 │ │ Ping 组件   │ │ Status 组件 │  ...              │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘                  │
│         │               │               │                          │
│         └───────────────┼───────────────┘                          │
│                         ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                    React 重绘                              │    │
│  │  • data 变化 → 组件 re-render                              │    │
│  │  • error 变化 → 错误状态展示                               │    │
│  │  • 骨架屏 → 数据填充的过渡效果                             │    │
│  └───────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

### 5.2 首屏渲染流程

1. **SSR 阶段**（服务端）：
   - `getStaticProps` 执行服务发现，获取 services/bookmarks/widgets 数据
   - 数据注入 SWR fallback 缓存
   - 页面 HTML 直接渲染完整内容

2. **Hydration 阶段**（前端）：
   - React 接管页面，SWR 从 fallback 中读取缓存数据
   - 组件立即使用缓存数据渲染，无闪烁
   - SWR 在后台发起重新验证请求（revalidate）

3. **更新阶段**：
   - 重新验证请求返回新数据
   - SWR 更新缓存
   - 依赖该缓存键的组件自动重渲染

### 5.3 组件重绘触发时机

SWR 数据变化会触发组件重绘，具体包括：

| 触发原因 | 描述 |
|----------|------|
| 初始数据加载 | 组件挂载后首次获取数据 |
| 轮询刷新 | `refreshInterval` 到期后刷新 |
| 窗口聚焦 | 标签页从后台切回前台 |
| 网络恢复 | 浏览器从离线状态恢复 |
| 手动 mutate | 调用 `mutate()` 函数强制刷新 |
| 缓存失效 | 配置 hash 变化触发页面重载 |

### 5.4 配置变更检测与全页刷新

在 [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx#L102-L131) 中实现了配置变更检测：

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

useEffect(() => {
  if (windowFocused) {
    mutateHash();
  }
}, [windowFocused, mutateHash]);

useEffect(() => {
  if (hashData && previousHash && previousHash !== hashData.hash) {
    setStale(true);
    fetch("/api/revalidate").then((res) => {
      if (res.ok) {
        window.location.reload();
      }
    });
  }
}, [hashData]);
```

流程：
1. 窗口聚焦时重新获取配置 hash
2. 与 localStorage 中存储的旧 hash 对比
3. 如果 hash 变化，调用 `/api/revalidate` 触发 ISR 重新生成页面
4. 然后执行 `window.location.reload()` 全页刷新

---

## 六、代理 API 数据流

Widget 的数据请求不直接调用第三方 API，而是通过服务端代理转发。

### 6.1 代理请求流程

```
前端组件
    │
    ▼ useWidgetAPI(widget, "endpoint")
    │
    ▼ 生成缓存键: /api/services/proxy?group=...&service=...&index=...&endpoint=...
    │
    ▼ SWR 发起 fetch 请求
    │
┌───▼──────────────────────────────────────┐
│  /api/services/proxy (Next.js API Route) │
│  [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/api/services/proxy.js) │
└───┬──────────────────────────────────────┘
    │
    ▼ getServiceWidget(group, service, index)
    │  从配置/服务发现中获取 widget 配置（含 URL、密钥等）
    │
    ▼ 查找 widget 定义 (widgets[type])
    │  获取 api 模板、proxyHandler、mappings 等
    │
    ▼ endpoint 映射
    │  前端传的 "wanted/missing" → 实际 "api/v3/wanted/missing"
    │
    ▼ 调用 proxyHandler (genericProxyHandler 等)
    │
    ▼ formatApiCall() 渲染 API URL 模板
    │  {url}/api/v3/{endpoint}?apikey={key}
    │
    ▼ httpProxy() 发起真实 HTTP 请求到目标服务
    │
    ▼ 数据验证 (validateWidgetData)
    │
    ▼ 数据映射 (map function)
    │
    ▼ 返回 JSON 响应
    │
    ▼ SWR 更新缓存
    │
    ▼ 前端组件重渲染
```

### 6.2 Widget 定义结构

每个 widget 在 `widgets/<type>/widget.js` 中定义，例如 [sonarr/widget.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/widgets/sonarr/widget.js)：

```javascript
const widget = {
  api: "{url}/api/v3/{endpoint}?apikey={key}",  // API URL 模板
  proxyHandler: genericProxyHandler,             // 代理处理器
  mappings: {                                    // 端点映射
    series: { endpoint: "series", map: (data) => ... },
    queue: { endpoint: "queue", validate: ["totalRecords"] },
    ...
  },
};
```

### 6.3 数据映射（map）

`mappings` 中的 `map` 函数用于：
- 精简返回数据，只保留前端需要的字段
- 转换数据格式，方便前端使用
- 排序、过滤等数据处理

这样可以减少网络传输数据量，并将数据处理逻辑放在服务端。

---

## 七、关键代码文件索引

| 文件 | 作用 |
|------|------|
| [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/_app.jsx) | SWR 全局配置 |
| [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/index.jsx) | 首页 SWR 使用、配置变更检测 |
| [use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/proxy/use-widget-api.js) | Widget 数据获取 Hook |
| [api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/proxy/api-helpers.js) | 缓存键生成、URL 模板渲染 |
| [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/api-response.js) | 服务发现响应处理 |
| [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/config/service-helpers.js) | 服务发现核心逻辑 |
| [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/pages/api/services/proxy.js) | Widget 代理 API 路由 |
| [generic.js](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/utils/proxy/handlers/generic.js) | 通用代理处理器 |
| [ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/206-homepage/src/components/services/ping.jsx) | Ping 组件（轮询示例） |

---

## 八、总结

### 8.1 数据流核心路径

```
配置文件/Docker/K8s
       │
       ▼ 服务发现（服务端）
  servicesResponse()
       │
       ▼ 清洗 + 白名单过滤
  cleanServiceGroups()
       │
       ▼ HTTP API 响应
  /api/services, /api/services/proxy
       │
       ▼ SWR 缓存（前端）
  key: URL 字符串
       │
       ▼ 轮询 / 聚焦 / 网络恢复
  refreshInterval + SWR 内置机制
       │
       ▼ React 组件重绘
  data/error 变化 → re-render
```

### 8.2 设计特点

1. **分层缓存**：SWR 客户端缓存 + 服务端可能的内存缓存
2. **渐进式加载**：SSR fallback → 后台 revalidate → 数据更新
3. **按需请求**：组件挂载时才发起请求，未挂载的 widget 不消耗资源
4. **统一代理**：所有第三方 API 通过服务端代理，统一处理认证、错误、数据映射
5. **安全设计**：敏感配置不发送到前端，通过服务端代理转发

### 8.3 刷新策略权衡

- **轮询间隔**：不同 widget 根据实时性需求设置不同间隔，平衡实时性和资源消耗
- **窗口聚焦刷新**：用户关注时自动刷新，后台时暂停，优化用户体验
- **配置 hash 检测**：平衡配置变更的实时性和全页刷新的成本
