# SWR 刷新与数据流分析

## 概述

本项目是基于 Next.js 的主页应用，使用 SWR (stale-while-revalidate) 作为前端数据获取和缓存策略。本文档重点梳理**配置刷新链路**、**SWR 补水刷新机制**，以及**服务端与浏览器各自的职责分工**。

---

## 一、配置刷新完整链路

配置刷新是一条从「文件变更检测」到「页面重新生成」再到「浏览器全量刷新」的完整闭环。

### 1.1 链路总览

```
配置文件变更
     │
     ▼ 服务端计算
  /api/hash → SHA256(7个配置文件 + buildTime)
     │
     ▼ 浏览器检测
  窗口聚焦 → mutateHash() → 获取新 hash
     │
     ▼ hash 比对
  localStorage 旧 hash ≠ 新 hash ?
     │
     ▼ 是
  调用 /api/revalidate
     │
     ▼ 服务端 ISR
  res.revalidate("/") → 后台重新生成首页静态页面
     │
     ▼ 浏览器
  window.location.reload() → 全页刷新
```

### 1.2 哈希计算（服务端）

哈希计算在 `src/pages/api/hash.js` 中实现：

```javascript
const configs = [
  "docker.yaml",
  "settings.yaml",
  "services.yaml",
  "bookmarks.yaml",
  "widgets.yaml",
  "custom.css",
  "custom.js",
];

function hash(buffer) {
  const hashSum = createHash("sha256");
  hashSum.update(buffer);
  return hashSum.digest("hex");
}

export default async function handler(req, res) {
  const hashes = configs.map((config) => {
    checkAndCopyConfig(config);
    const configYaml = join(CONF_DIR, config);
    return hash(readFileSync(configYaml, "utf8"));
  });

  const buildTime = process.env.HOMEPAGE_BUILDTIME?.length
    ? process.env.HOMEPAGE_BUILDTIME
    : "";

  const combinedHash = hash(hashes.join("") + buildTime);
  res.send({ hash: combinedHash });
}
```

**关键要点**：
- 监控 **7 个配置文件**：docker.yaml、settings.yaml、services.yaml、bookmarks.yaml、widgets.yaml、custom.css、custom.js
- 使用 **SHA-256** 算法计算每个文件的哈希
- 最终哈希 = 所有文件哈希拼接 + buildTime 环境变量，再做一次哈希
- `HOMEPAGE_BUILDTIME` 确保容器重启/重建后也会触发刷新
- **每次请求 `/api/hash` 都会实时计算**，不缓存结果

### 1.3 哈希变更检测（浏览器端）

检测逻辑在 `src/pages/index.jsx` 的 `Index` 组件中：

```javascript
function Index({ initialSettings, fallback }) {
  const windowFocused = useWindowFocus();
  const [stale, setStale] = useState(false);
  const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

  // 窗口聚焦时主动刷新 hash
  useEffect(() => {
    if (windowFocused) {
      mutateHash();
    }
  }, [windowFocused, mutateHash]);

  // hash 变化时触发重新验证和页面刷新
  useEffect(() => {
    if (hashData) {
      if (typeof window !== "undefined") {
        const previousHash = localStorage.getItem("hash");

        if (!previousHash) {
          localStorage.setItem("hash", hashData.hash);
        }

        if (previousHash && previousHash !== hashData.hash) {
          setStale(true);
          localStorage.setItem("hash", hashData.hash);

          fetch("/api/revalidate").then((res) => {
            if (res.ok) {
              window.location.reload();
            }
          });
        }
      }
    }
  }, [hashData]);
```

**触发时机**：
1. **页面首次加载**：SWR 自动获取 `/api/hash`，存入 localStorage 作为基准
2. **窗口聚焦时**：`useWindowFocus` 检测到焦点，调用 `mutateHash()` 强制重新获取
3. **网络恢复时**：SWR 内置的 revalidateOnReconnect 自动刷新

**比对逻辑**：
- 从 localStorage 读取上次保存的 `hash`
- 与新获取的 hash 对比
- 如果不一致 → 标记页面为 stale → 调用 ISR → 全页刷新

### 1.4 ISR 按需重新生成（服务端）

调用 `/api/revalidate` 触发 Next.js 增量静态重新生成：

```javascript
// src/pages/api/revalidate.js
export default async function handler(req, res) {
  try {
    await res.revalidate("/");
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}
```

**ISR 工作原理**：
- `res.revalidate("/")` 是 Next.js 提供的 API
- 它会**在后台**重新运行 `getStaticProps()` 生成新的静态页面
- 旧页面继续对外服务，直到新页面生成完成
- 生成成功后，下一次请求就会返回新页面
- 这是「按需」（on-demand）重新生成，不是定时的

**触发方式有两种**：
1. **自动检测**：hash 变化时自动调用
2. **手动触发**：点击页面右下角的刷新按钮（`src/components/toggles/revalidate.jsx`）

### 1.5 全页刷新（浏览器端）

ISR 触发成功后，浏览器执行全页刷新：

```javascript
fetch("/api/revalidate").then((res) => {
  if (res.ok) {
    window.location.reload();
  }
});
```

**为什么需要全页刷新**：
- 配置变更可能影响页面结构（新增/删除服务、布局变化等）
- SWR 只能刷新数据，不能改变页面结构
- 全页刷新确保所有配置（包括静态生成部分）都更新

---

## 二、SWR 补水刷新机制

补水（Hydration + Revalidate）是 SWR 的核心特性：先用缓存数据渲染，再后台刷新。

### 2.1 三层 SWR 配置层级

项目中有三层 SWR 配置，层层叠加：

**第一层：全局配置（_app.jsx）**

```javascript
// src/pages/_app.jsx
<SWRConfig
  value={{
    fetcher: (resource, init) =>
      fetch(resource, init).then((res) => res.json()),
  }}
>
```

作用：全局默认 fetcher，所有 SWR 调用共享。

**第二层：页面级 fallback（index.jsx）**

```javascript
// src/pages/index.jsx
<SWRConfig
  value={{
    fallback,
    fetcher: (resource, init) =>
      fetch(resource, init).then((res) => res.json()),
  }}
>
```

作用：注入静态生成预获取的数据作为 SWR 初始缓存。

**第三层：组件级配置（各 widget 组件）**

```javascript
// 例如 useWidgetAPI 或直接 useSWR
useSWR(url, { refreshInterval: 30000 });
```

作用：每个组件独立配置刷新间隔等参数。

### 2.2 首屏三阶段渲染流程

```
┌──────────────────────────────────────────────────────────────┐
│                  阶段 1: 静态生成 (SSG)                      │
│  构建时 / ISR 重新生成时执行 getStaticProps()                │
│    → 服务发现（Docker/K8s/配置文件）                        │
│    → 读取 widgets、bookmarks                                │
│    → 数据打包进 fallback 对象                               │
│    → 渲染完整 HTML 静态页面                                 │
│    → fallback 数据序列化到 HTML 中（__NEXT_DATA__）         │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   阶段 2: Hydration 补水                    │
│  浏览器加载 HTML 和 JS                                      │
│    → React 接管页面（hydrate）                              │
│    → SWR 从 fallback 中读取缓存数据                         │
│    → 组件立即使用缓存数据渲染（无 loading 状态）             │
│    → 用户看到完整页面，无需等待 API 请求                     │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                阶段 3: 后台 Revalidate                      │
│  SWR 在后台发起重新验证请求                                  │
│    → 请求 /api/services、/api/bookmarks 等                  │
│    → 服务端重新执行服务发现，返回最新数据                    │
│    → SWR 更新缓存                                           │
│    → 组件使用新数据重渲染（如果数据变了）                    │
│    → 用户可能无感知，也可能看到数据变化                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 fallback 数据结构

`getStaticProps` 返回的 fallback 包含：

```javascript
fallback: {
  "/api/services": services,     // 服务列表（含 widget 配置）
  "/api/bookmarks": bookmarks,   // 书签列表
  "/api/widgets": widgets,       // 信息组件列表
  "/api/hash": false,            // hash 初始值
}
```

这些数据被注入到 `SWRConfig` 中，当组件调用 `useSWR("/api/services")` 时：
- 如果 fallback 中有对应 key → 立即返回缓存数据
- 同时后台发起 revalidate 请求
- 请求完成后更新缓存和组件

### 2.4 Widget 数据的补水策略

注意：**所有 widget 的具体 API 数据都不在 fallback 中**，只有配置元数据在。

```
fallback 中有的：
  /api/services → 服务 + 服务 Widget 配置元数据
  /api/widgets → 信息组件列表 + 配置元数据

fallback 中没有的（需要客户端异步请求）：

  服务 Widget 数据:
    /api/services/proxy?group=...&service=... → 真实业务数据

  信息组件数据:
    /api/widgets/weather?... → 天气数据
    /api/widgets/stocks?... → 股票数据
    /api/widgets/resources?type=cpu → 资源数据
    ...

  状态类数据:
    /api/ping?... → ping 状态
    /api/docker/status/... → docker 状态
```

**原因**：
- Widget 数据是实时的（CPU、内存、股价等），静态生成时的缓存很快过期
- Widget 数量多，全部预获取会拖慢静态生成速度
- 采用「骨架屏 + 异步加载」的体验更好

### 2.5 刷新触发时机汇总

| 触发方式 | 触发源 | 说明 |
|---------|--------|------|
| 静态生成预填充 | getStaticProps | 构建/重新生成时执行，数据注入 fallback |
| 组件挂载 | useSWR 初始化 | 组件首次渲染时发起请求 |
| 轮询刷新 | refreshInterval | 按固定间隔周期性刷新 |
| 窗口聚焦 | revalidateOnFocus | 标签页从后台切回前台时刷新 |
| 网络恢复 | revalidateOnReconnect | 浏览器从离线恢复在线时刷新 |
| 手动 mutate | mutate() | 编程式强制刷新 |
| 配置变更 | hash 检测 → ISR → reload | 配置文件变化时全页刷新 |

---

## 三、服务端与浏览器职责分工

### 3.1 服务端职责

**服务端承担的工作**：

| 模块 | 职责 | 所在文件 |
|------|------|----------|
| 服务发现 | 从 Docker/K8s/配置文件中发现服务和 widget | `src/utils/config/service-helpers.js` |
| 数据清洗 | 白名单过滤，移除敏感字段（密码、API Key） | `src/utils/config/service-helpers.js` 中 `cleanServiceGroups()` |
| 哈希计算 | 计算配置文件的 SHA-256 哈希 | `src/pages/api/hash.js` |
| ISR 触发生成 | 按需重新生成静态页面 | `src/pages/api/revalidate.js` |
| 服务 Widget 代理 | 转发服务 Widget 请求到真实后端，处理认证 | `src/pages/api/services/proxy.js` |
| 信息组件 API | 天气、股票、资源等独立 API 端点 | `src/pages/api/widgets/*.js` |
| 服务端缓存 | memory-cache 缓存第三方 API 响应 | `src/utils/proxy/http.js` 中 `cachedRequest()` |
| 数据映射 | 精简、转换、过滤 API 返回数据 | 各 `src/widgets/<type>/widget.js` 的 map 函数 |
| 静态生成 | 预获取数据，渲染完整 HTML | `src/pages/index.jsx` 的 `getStaticProps()` |

**服务端特点**：
- 有文件系统访问权限（读取 yaml 配置）
- 有 Docker/K8s API 访问权限
- 持有敏感凭据（API Key、密码）
- 可以直接调用内部服务 API
- 负责安全过滤，确保前端拿不到敏感数据
- 服务端缓存为所有用户共享

### 3.2 浏览器端职责

**浏览器承担的工作**：

| 模块 | 职责 | 所在文件 |
|------|------|----------|
| SWR 缓存管理 | 存储 API 数据，管理缓存过期 | `swr` 库 + 各组件 |
| 轮询调度 | 按 refreshInterval 定时刷新 | SWR 内置 + `useWidgetAPI` + 信息组件 |
| 窗口聚焦检测 | 监听 focus/blur 事件 | `src/utils/hooks/window-focus.js` |
| 哈希比对 | 检测配置变更，触发刷新 | `src/pages/index.jsx` |
| 页面渲染 | React 组件渲染、状态管理 | 各组件 |
| 用户交互 | 点击、输入等交互处理 | 各组件 |
| 本地存储 | 保存 hash 到 localStorage | `src/pages/index.jsx` |

**浏览器特点**：
- 无敏感凭据，所有数据请求通过服务端中转
- 只持有展示所需的数据
- 负责实时交互和动画
- SWR 缓存随页面刷新丢失
- 服务 Widget 和信息组件都使用 SWR 管理前端缓存

### 3.3 数据流向图

```
┌───────────────────────────────────────────────────────────────┐
│                          服务端                                │
│                                                               │
│  配置文件 ──→ 服务发现 ──→ 数据清洗 ──→ 静态生成              │
│     │          │              │             │                 │
│     │          │              │         fallback              │
│     │          │              │             ↓                 │
│     │          │              └──────→ HTML + 数据            │
│     │          │                            │                 │
│     │          ├───────────→ /api/services │                 │
│     │          │              /api/widgets  │                 │
│     │          │              /api/hash     │                 │
│     │          │              /api/ping     │                 │
│     │          │                              │                │
│     │          │ 服务 Widget 代理路径         │ 信息组件路径    │
│     │          ▼                              ▼                │
│     │   /api/services/proxy          /api/widgets/*           │
│     │          │                      │  天气、股票、资源等    │
│     │          │                      │                        │
│     │          ▼                      ▼                        │
│     │   代理处理器         服务端缓存 (cachedRequest)          │
│     │   (generic等)        部分信息组件使用                    │
│     │          │                      │                        │
│     └──────────┴──────────────────────┘                        │
│                        │                                       │
│                        ▼                                       │
│              第三方 API / 本机系统信息                         │
└────────────────────────┬───────────────────────────────────────┘
                         │ HTTP 请求
┌────────────────────────┴───────────────────────────────────────┐
│                          浏览器                                │
│                                                                 │
│  HTML + fallback ──→ SWR 缓存初始化                           │
│                          │                                      │
│                          ▼                                      │
│                      组件渲染（补水）                           │
│                          │                                      │
│                          ▼                                      │
│                   后台 revalidate                              │
│                          │                                      │
│                          ▼                                      │
│                   SWR 缓存更新                                  │
│                          │                                      │
│                          ▼                                      │
│                   组件重渲染                                    │
│                                                                 │
│  窗口聚焦 ──→ mutateHash() ──→ hash 比对 ──→ 全页刷新         │
│                                                                 │
│  服务 Widget: useWidgetAPI Hook                                │
│  信息组件: 直接 useSWR                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、两套 Widget 系统与数据路径

项目中存在两套独立的 widget 系统，数据获取路径和缓存策略各不相同。

### 4.1 两套 Widget 系统对比

| 维度 | 服务 Widget (Service Widgets) | 信息组件 (Information Widgets) |
|------|-----------------------------|-------------------------------|
| 位置 | `src/widgets/<type>/` | `src/components/widgets/<type>/` |
| API 路径 | `/api/services/proxy?…` | `/api/widgets/<type>?…` |
| 获取方式 | `useWidgetAPI` Hook | 直接 `useSWR` |
| 数据来源 | 第三方服务 API（Sonarr、Radarr 等） | 公开 API 或本机系统信息 |
| 服务端缓存 | 否（每次透传） | 部分有（cachedRequest） |
| 数量 | 150+ 种服务 | 天气、股票、资源、Glances 等十余种 |

### 4.2 缓存键生成规则

SWR 使用 URL 字符串作为缓存键。

**基础数据缓存键**：

| 数据 | 缓存键 | 来源 |
|------|--------|------|
| 服务列表 | `/api/services` | 静态生成 fallback + 客户端 revalidate |
| 书签列表 | `/api/bookmarks` | 静态生成 fallback + 客户端 revalidate |
| 信息组件列表 | `/api/widgets` | 静态生成 fallback + 客户端 revalidate |
| 配置校验 | `/api/validate` | 仅客户端 |
| 配置哈希 | `/api/hash` | 仅客户端 |

**服务 Widget 缓存键**：

格式：
```
/api/services/proxy?group=<group>&service=<service>&index=<index>&endpoint=<endpoint>[&query=<json>]
```

生成函数在 `src/utils/proxy/api-helpers.js` 的 `formatProxyUrl()`：

```javascript
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  if (queryParams) {
    params.append("query", JSON.stringify(queryParams));
  }
  return `/api/services/proxy?${params.toString()}`;
}
```

唯一性保证：`group + service + index + endpoint` 四维组合，确保每个 widget 实例的每个端点都有独立缓存。

**信息组件缓存键**：

| 组件 | 缓存键格式 |
|------|-----------|
| 天气 | `/api/widgets/weather?latitude=...&longitude=...&lang=...` |
| OpenMeteo | `/api/widgets/openmeteo?latitude=...&longitude=...&units=...` |
| 股票 | `/api/widgets/stocks?watchlist=...&provider=...` |
| 资源 CPU | `/api/widgets/resources?type=cpu` |
| 资源内存 | `/api/widgets/resources?type=memory` |
| 资源磁盘 | `/api/widgets/resources?type=disk&target=...` |
| 资源网络 | `/api/widgets/resources?type=network&interfaceName=...` |
| Glances | `/api/widgets/glances?index=...&cpu=...` |
| Kubernetes | `/api/widgets/kubernetes?...` |
| Longhorn | `/api/widgets/longhorn?...` |

**状态类缓存键**：

| 类型 | 缓存键格式 |
|------|-----------|
| Ping 状态 | `/api/ping?groupName=xxx&serviceName=xxx` |
| Docker 状态 | `/api/docker/status/<container>/<server>` |
| Docker 统计 | `/api/docker/stats/<container>/<server>` |

### 4.3 各类数据刷新间隔

| 数据类型 | 刷新间隔 | 配置位置 |
|----------|---------|----------|
| 服务/书签/信息组件列表 | 无（仅挂载+聚焦时刷新） | `src/pages/index.jsx` |
| 配置 hash | 无（窗口聚焦时手动 mutate） | `src/pages/index.jsx` |
| Ping 状态 | 30000ms (30s) | `src/components/services/ping.jsx` |
| 服务 Widget（Sonarr/Radarr 等） | 无（默认 SWR 行为） | `src/widgets/*/component.jsx` |
| Jellyfin 播放中 | 5000ms (5s) | `src/widgets/jellyfin/component.jsx` |
| Jellyfin 未播放 | 60000ms (60s) | `src/widgets/jellyfin/component.jsx` |
| Glances（服务 Widget） | 1000ms (1s) 起，可配置 | `src/widgets/glances/metrics/*.jsx` |
| Prometheus Metric | 10000ms (10s) 默认 | `src/widgets/prometheusmetric/component.jsx` |
| 天气组件 | 无（依赖服务端缓存 + SWR 默认） | `src/components/widgets/weather/weather.jsx` |
| 股票组件 | 无（依赖服务端缓存 + SWR 默认） | `src/components/widgets/stocks/stocks.jsx` |
| 资源组件（CPU/内存等） | 1500ms 默认，最小 1000ms | `src/components/widgets/resources/resources.jsx` |

### 4.4 服务 Widget 统一 Hook：useWidgetAPI

服务 Widget 的数据获取统一通过 `src/utils/proxy/use-widget-api.js`（信息组件不使用此 Hook）：

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
- 从 options 中提取 `refreshInterval`
- endpoint 为空字符串时返回 `url = null`，SWR 跳过请求
- 将 API 返回的 `error` 字段提升为顶层错误
- 仅适用于服务 Widget，信息组件直接使用 `useSWR`

### 4.5 信息组件数据获取方式

信息组件直接使用 `useSWR` 调用独立的 API 端点，不经过 `/api/services/proxy` 代理。

**天气组件示例**（`src/components/widgets/weather/weather.jsx`）：

```javascript
const { data, error } = useSWR(
  `/api/widgets/weather?${new URLSearchParams({ lang: i18n.language, ...options }).toString()}`,
);
```

**资源组件示例**（`src/components/widgets/resources/cpu.jsx`）：

```javascript
const { data, error } = useSWR(`/api/widgets/resources?type=cpu`, {
  refreshInterval: refresh, // 默认 1500ms
});
```

**股票组件示例**（`src/components/widgets/stocks/stocks.jsx`）：

```javascript
const { data, error } = useSWR(
  `/api/widgets/stocks?${new URLSearchParams({ lang: i18n.language, ...options }).toString()}`,
);
```

---

## 五、服务端缓存机制

### 5.1 cachedRequest 函数

服务端缓存实现在 `src/utils/proxy/http.js` 的 `cachedRequest` 函数中，基于 `memory-cache` 库：

```javascript
import cache from "memory-cache";

export async function cachedRequest(url, duration = 5, ua = "homepage") {
  const cached = cache.get(url);

  if (cached) {
    return cached;
  }

  const options = {
    headers: {
      "User-Agent": ua,
      Accept: "application/json",
    },
  };
  let [, , data] = await httpProxy(url, options);
  if (Buffer.isBuffer(data)) {
    try {
      data = JSON.parse(Buffer.from(data).toString());
    } catch (e) {
      data = Buffer.from(data).toString();
    }
  }
  cache.put(url, data, duration * 1000 * 60);
  return data;
}
```

**工作原理**：
1. 以请求 URL 作为缓存 key
2. 命中缓存直接返回，不发起真实请求
3. 未命中则通过 `httpProxy` 发起请求
4. 将返回数据解析为 JSON（或字符串）后存入缓存
5. 缓存有效期为 `duration * 1000 * 60` 毫秒，即 duration 的单位是**分钟**
6. 默认缓存时间为 5 分钟

### 5.2 哪些数据走服务端缓存

| 组件/API | 是否使用 cachedRequest | 默认缓存时长 | 所在文件 |
|---------|---------------------|-------------|----------|
| 天气 (weatherapi) | ✅ 是 | 由 query 参数 `cache` 决定 | `src/pages/api/widgets/weather.js` |
| OpenMeteo | ✅ 是 | 由 query 参数 `cache` 决定 | `src/pages/api/widgets/openmeteo.js` |
| OpenWeatherMap | ✅ 是 | 由 query 参数 `cache` 决定 | `src/pages/api/widgets/openweathermap.js` |
| 股票 (Finnhub) | ✅ 是 | 1 分钟 | `src/pages/api/widgets/stocks.js` |
| 资源 (resources) | ❌ 否 | 直接读本机系统信息 | `src/pages/api/widgets/resources.js` |
| Glances (信息组件) | ❌ 否 | 每次透传调用 | `src/pages/api/widgets/glances.js` |
| Kubernetes | ❌ 否 | 每次透传调用 | `src/pages/api/widgets/kubernetes.js` |
| Longhorn | ❌ 否 | 每次透传调用 | `src/pages/api/widgets/longhorn.js` |
| 服务 Widget (proxy) | ❌ 否 | 每次透传调用 | `src/pages/api/services/proxy.js` |
| Ping | ❌ 否 | 每次实时检测 | `src/pages/api/ping.js` |

### 5.3 服务端缓存与 SWR 缓存的关系

存在两层缓存，各司其职：

```
浏览器请求
    │
    ▼ 第一层：SWR 前端缓存（内存）
    │  key: URL 字符串
    │  命中：立即返回，不发请求
    │  未命中：发起到服务端的 HTTP 请求
    │
    ▼ 第二层：服务端 memory-cache（内存）
       key: 真实 API 的 URL
       命中：直接返回缓存数据
       未命中：发起真实 HTTP 请求到第三方 API
```

**两层缓存的区别**：

| 维度 | SWR 前端缓存 | 服务端 memory-cache |
|------|-------------|-------------------|
| 位置 | 浏览器内存 | Node.js 服务端内存 |
| 缓存 key | 代理 API 的 URL | 真实第三方 API 的 URL |
| 有效期 | 随页面刷新丢失，受 SWR 策略控制 | 固定时长（分钟级） |
| 作用范围 | 单个用户浏览器 | 所有用户共享 |
| 数据内容 | 经过映射/清洗的前端数据 | 原始 API 响应数据 |
| 更新机制 | revalidateOnFocus、refreshInterval 等 | TTL 过期自动失效 |

---

## 六、服务发现与数据清洗

### 6.1 三种服务来源

在 `src/utils/config/api-response.js` 的 `servicesResponse()` 中合并：

| 来源 | 描述 | 核心函数 |
|------|------|----------|
| 配置文件 | 从 services.yaml 静态读取 | `servicesFromConfig()` |
| Docker 发现 | 扫描容器标签 homepage.* | `servicesFromDocker()` |
| K8s 发现 | 扫描 Ingress/TraefikIngress/HTTPRoute | `servicesFromKubernetes()` |

### 6.2 数据白名单清洗

在 `src/utils/config/service-helpers.js` 的 `cleanServiceGroups()` 中进行白名单过滤。widget 配置只保留安全字段，敏感字段（url、username、password、apikey 等）**不会**发送到前端。

前端拿到的 widget 只包含展示所需的元数据，真实 API 请求通过服务端代理转发。

---

## 七、服务 Widget 代理 API 数据流

服务 Widget（Sonarr、Radarr、Jellyfin 等 150+ 种）的数据请求通过服务端代理转发。信息组件（天气、股票、资源等）走独立 API 端点，不经过此代理。

### 7.1 代理请求流程

```
前端组件
   │
   ▼ useWidgetAPI(widget, "endpoint")
   │
   ▼ 生成缓存键: /api/services/proxy?group=...&service=...&index=...&endpoint=...
   │
   ▼ SWR 发起 fetch（命中缓存则直接返回）
   │
┌──▼──────────────────────────────────────┐
│  /api/services/proxy (服务端 API 路由)  │
└──┬──────────────────────────────────────┘
   │
   ▼ getServiceWidget(group, service, index)
   │  从配置/服务发现中获取完整 widget 配置（含 URL、密钥）
   │
   ▼ 查找 widgets[type] 定义
   │  获取 api 模板、proxyHandler、mappings
   │
   ▼ endpoint 映射
   │  逻辑名 → 真实 API 路径
   │
   ▼ formatApiCall() 渲染 URL 模板
   │  {url}/api/v3/{endpoint}?apikey={key}
   │
   ▼ httpProxy() 发起真实 HTTP 请求
   │
   ▼ validateWidgetData() 数据验证
   │
   ▼ map function 数据映射（精简、转换）
   │
   ▼ 返回 JSON 响应
   │
   ▼ SWR 更新缓存
   │
   ▼ 前端组件重渲染
```

### 7.2 Widget 定义示例

每个 widget 在 `widgets/<type>/widget.js` 中定义，以 sonarr 为例：

```javascript
const widget = {
  api: "{url}/api/v3/{endpoint}?apikey={key}",
  proxyHandler: genericProxyHandler,

  mappings: {
    series: {
      endpoint: "series",
      map: (data) => asJson(data).map((entry) => ({
        title: entry.title,
        id: entry.id,
      })),
    },
    queue: {
      endpoint: "queue",
      validate: ["totalRecords"],
    },
  },
};
```

`map` 函数的作用：
- 精简返回数据，只保留前端需要的字段
- 减少网络传输量
- 将数据处理逻辑放在服务端

---

## 八、关键代码文件索引

### 8.1 核心文件

| 文件路径 | 作用 |
|---------|------|
| `src/pages/_app.jsx` | SWR 全局配置，默认 fetcher |
| `src/pages/index.jsx` | 首页、getStaticProps、hash 检测、fallback 注入 |
| `src/pages/api/hash.js` | 配置文件哈希计算 |
| `src/pages/api/revalidate.js` | ISR 按需重新生成入口 |
| `src/utils/config/api-response.js` | 服务发现 + 信息组件响应处理 |
| `src/utils/config/service-helpers.js` | 服务发现核心逻辑、数据清洗 |
| `src/utils/config/config.js` | 配置文件读取、环境变量替换 |
| `src/utils/hooks/window-focus.js` | 窗口聚焦检测 Hook |
| `src/components/toggles/revalidate.jsx` | 手动刷新按钮 |

### 8.2 服务 Widget 相关

| 文件路径 | 作用 |
|---------|------|
| `src/pages/api/services/index.js` | 服务列表 API |
| `src/pages/api/services/proxy.js` | 服务 Widget 代理 API 路由 |
| `src/utils/proxy/use-widget-api.js` | 服务 Widget 数据获取 Hook |
| `src/utils/proxy/api-helpers.js` | 缓存键生成、URL 模板渲染 |
| `src/utils/proxy/handlers/generic.js` | 通用代理处理器 |
| `src/components/services/ping.jsx` | Ping 组件（轮询示例） |
| `src/widgets/*/` | 150+ 种服务 Widget 定义与组件 |

### 8.3 信息组件相关

| 文件路径 | 作用 |
|---------|------|
| `src/pages/api/widgets/index.js` | 信息组件列表 API |
| `src/pages/api/widgets/weather.js` | 天气 API（weatherapi） |
| `src/pages/api/widgets/openmeteo.js` | OpenMeteo 天气 API |
| `src/pages/api/widgets/openweathermap.js` | OpenWeatherMap API |
| `src/pages/api/widgets/stocks.js` | 股票 API（Finnhub） |
| `src/pages/api/widgets/resources.js` | 本机资源 API（CPU/内存/磁盘等） |
| `src/pages/api/widgets/glances.js` | Glances 信息组件 API |
| `src/pages/api/widgets/kubernetes.js` | Kubernetes 信息组件 API |
| `src/pages/api/widgets/longhorn.js` | Longhorn 信息组件 API |
| `src/components/widgets/*/` | 信息组件前端实现 |

### 8.4 缓存与 HTTP 相关

| 文件路径 | 作用 |
|---------|------|
| `src/utils/proxy/http.js` | HTTP 请求、服务端缓存（cachedRequest） |
| `src/utils/proxy/http.test.js` | HTTP 模块测试 |

---

## 九、设计特点总结

### 9.1 分层缓存策略

项目共有四层缓存/数据保留机制：

1. **SSG + ISR 层**：页面结构级缓存，配置变更时触发重新生成
2. **SWR 客户端层**：前端数据级缓存，自动刷新，所有 API 共享
3. **服务端 memory-cache 层**：部分信息组件使用（天气、股票等），多用户共享
4. **服务端代理透传**：服务 Widget 和大部分信息组件不缓存，每次请求都透传

### 9.2 两套 Widget 系统

| 维度 | 服务 Widget | 信息组件 |
|------|------------|---------|
| API 路径 | `/api/services/proxy` | `/api/widgets/<type>` |
| 前端 Hook | `useWidgetAPI` | 直接 `useSWR` |
| 服务端缓存 | 否 | 部分有 |
| 数量 | 150+ | 十余种 |
| 位置 | 服务卡片内 | 页面顶部/独立区域 |

### 9.3 渐进式加载体验

- 首屏：静态生成预渲染，完整内容立即可见
- 补水：SWR 用 fallback 数据立即渲染
- 刷新：后台静默更新，有变化时平滑过渡
- 实时：Widget 按各自间隔轮询更新

### 9.4 安全设计

- 敏感配置永不离开服务端
- 前端只拿到展示所需的白名单字段
- 服务 Widget 和信息组件都通过服务端中转
- URL 中敏感参数在错误信息中脱敏

### 9.5 刷新策略权衡

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| SSG + ISR | SEO 友好、首屏快 | 结构变更需全页刷新 | 页面结构、布局 |
| SWR fallback | 无闪烁、体验好 | 初始数据可能过期 | 列表、元数据 |
| 轮询刷新 | 实时性可控 | 资源消耗随间隔降低而增加 | 指标、状态 |
| 窗口聚焦刷新 | 用户关注时才更新 | 后台可能显示过期数据 | 非关键数据 |
| 服务端 memory-cache | 多用户共享、减少第三方调用 | 占用服务端内存、数据可能陈旧 | 天气、股票等变化较慢的数据 |
| hash 检测 + 全页刷新 | 确保配置一致 | 全页刷新有闪烁 | 配置变更 |
