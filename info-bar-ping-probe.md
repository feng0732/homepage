# 信息条与 Ping / SiteMonitor 探测协作机制

本文档梳理 Homepage 项目中"信息条"（服务卡片右上角的状态标签）与 Ping / SiteMonitor 探测之间的完整协作流程，涵盖：探测触发、状态汇总、信息条展示和刷新节奏。

---

## 一、整体架构概览

```
用户配置 (services.yaml)
  └─ service.ping / service.siteMonitor 字段
       │
       ▼
服务数据加载 (service-helpers.js → api-response.js → /api/services)
  └─ 前端 SWR 拿到 services 列表，每条 service 带 ping / siteMonitor 属性
       │
       ▼
Item 组件 (item.jsx)
  ├─ service.ping 为真 → 渲染 <Ping> 组件
  └─ service.siteMonitor 为真 → 渲染 <SiteMonitor> 组件
       │
       ▼
前端 SWR 轮询 (refreshInterval: 30000ms)
  ├─ <Ping> → GET /api/ping?groupName=…&serviceName=…
  └─ <SiteMonitor> → GET /api/siteMonitor?groupName=…&serviceName=…
       │
       ▼
后端 API Handler
  ├─ /api/ping → 读取 service 配置 → 调用 ping.probe(hostname) → 返回 { alive, time }
  └─ /api/siteMonitor → 读取 service 配置 → 发 HEAD/GET 请求 → 返回 { status, latency }
       │
       ▼
前端根据返回数据渲染状态标签（文字 / 圆点 / 延迟毫秒数）
```

---

## 二、配置入口：探测从哪里来

服务在 `services.yaml`（或 Docker label / Kubernetes annotation）中可声明两种探测：

```yaml
- My Group:
    - Sonarr:
        href: http://sonarr.host/
        ping: sonarr.host              # ICMP ping 探测

    - Radarr:
        href: http://radarr.host/
        siteMonitor: http://radarr.host/   # HTTP 探测（HEAD → GET 回退）
```

- `ping`：值为主机名或 URL。若为 URL，后端会自动提取 `hostname`。
- `siteMonitor`：值为完整 URL，后端对其发起 HTTP 请求。

两者可同时配置，也可单独配置；都不配则无状态标签。

配置经过 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/config/service-helpers.js) 的 `servicesFromConfig()` / `servicesFromDocker()` / `servicesFromKubernetes()` 解析后，作为 service 对象的属性传递到前端。

---

## 三、前端入口：Item 组件的条件渲染

关键文件：[item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/item.jsx#L87-L104)

```jsx
<div className="absolute top-0 right-0 flex flex-row justify-end … service-tags">
  {service.ping && (
    <div className="shrink-0 flex items-center justify-center service-tag service-ping">
      <Ping groupName={groupName} serviceName={service.name} style={statusStyle} />
    </div>
  )}

  {service.siteMonitor && (
    <div className="shrink-0 flex items-center justify-center service-tag service-site-monitor">
      <SiteMonitor groupName={groupName} serviceName={service.name} style={statusStyle} />
    </div>
  )}
</div>
```

要点：
- **条件渲染**：只有 `service.ping` 为真值时才挂载 `<Ping>` 组件；`siteMonitor` 同理。
- **位置**：信息条绝对定位于服务卡片的右上角（`absolute top-0 right-0`），多个标签水平排列。
- **statusStyle 透传**：`statusStyle` 来自全局设置或服务级覆盖，决定标签的视觉风格。

`statusStyle` 的取值逻辑（[item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/item.jsx#L20)）：

```jsx
const statusStyle = service.statusStyle !== undefined ? service.statusStyle : settings.statusStyle;
```

优先级：服务自身 `statusStyle` > 全局 `settings.statusStyle`。

---

## 四、Ping 组件：ICMP 探测前端

关键文件：[ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.jsx)

### 4.1 数据获取

```jsx
const { data, error } = useSWR(
  `/api/ping?${new URLSearchParams({ groupName, serviceName }).toString()}`,
  { refreshInterval: 30000 }
);
```

- 使用 SWR 库发起请求，URL 为 `/api/ping?groupName=…&serviceName=…`。
- **刷新间隔**：`refreshInterval: 30000`，即每 30 秒自动重新请求。

### 4.2 状态判定与视觉映射

| 条件 | statusText | colorClass | statusTitle |
|---|---|---|---|
| `error` 存在 | `ping.error` | `text-rose-500` | `Ping Error` |
| `data` 为空（加载中） | `ping.ping` | `text-black/20 opacity-20` | `Ping Not Available` |
| `data.alive === false` | `ping.down` | `text-rose-500/80` | `Ping Down` |
| `data.alive === true` + `style === "basic"` | `ping.up` | `text-emerald-500/80` | `Ping Up (xx ms)` |
| `data.alive === true` + 其他 style | `xx ms`（延迟数值） | `text-emerald-500/80 lowercase` | `Ping Up (xx ms)` |

### 4.3 style 变体

- **默认**：显示 8px 加粗大写文字标签（如 `123` ms 或 `DOWN`）。
- **`"basic"`**：存活时只显示 "UP" 文字，不展示延迟数值。
- **`"dot"`**：隐藏文字，渲染一个 12×12px 的圆点（`rounded-full h-3 w-3`），颜色从 `text-*` 映射为 `bg-*`。

---

## 五、SiteMonitor 组件：HTTP 探测前端

关键文件：[site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.jsx)

### 5.1 数据获取

```jsx
const { data, error } = useSWR(
  `/api/siteMonitor?${new URLSearchParams({ groupName, serviceName }).toString()}`,
  { refreshInterval: 30000 }
);
```

与 Ping 相同的 SWR 模式，30 秒刷新。

### 5.2 状态判定与视觉映射

| 条件 | statusText | colorClass | statusTitle |
|---|---|---|---|
| `error` 或 `data.error` | `siteMonitor.error` | `text-rose-500` | `HTTP status Error` |
| `data` 为空（加载中） | `siteMonitor.response` | `text-black/20 opacity-20` | `HTTP status Not Available` |
| `data.status > 403` | `siteMonitor.down` / 状态码 | `text-rose-500/80` | `HTTP status {code}` |
| `data.status ≤ 403` + `style === "basic"` | `siteMonitor.up` | `text-emerald-500/80` | `HTTP status {code} (xx ms)` |
| `data.status ≤ 403` + 其他 style | `xx ms`（延迟数值） | `text-emerald-500/80 lowercase` | `HTTP status {code} (xx ms)` |

**注意**：SiteMonitor 的"下线"判定是 HTTP 状态码 > 403，而非网络不可达。401/403 等仍视为"在线"。

### 5.3 style 变体

与 Ping 完全一致：默认文字 / basic 只显示 UP / dot 圆点。

---

## 六、后端 API：探测执行

### 6.1 `/api/ping` — ICMP Ping

关键文件：[ping.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/api/ping.js)

```
请求: GET /api/ping?groupName=G&serviceName=S
流程:
  1. 从 query 取 groupName、serviceName
  2. 调用 getServiceItem(groupName, serviceName) 查找服务配置
     → 依次搜索: services.yaml → Docker labels → Kubernetes annotations
  3. 若找不到服务 → 400 "Unable to find service"
  4. 取 service.ping 字段；若为空 → 400 "No ping host given"
  5. 尝试将 ping 值当 URL 解析取 hostname（兼容旧配置 http://…）
     若非 URL 则直接使用原值作为 hostname
  6. 调用 ping.probe(hostname)（底层为 node-ping 库的 ICMP 探测）
  7. 成功 → 200 { alive: true/false, time: <ms>, … }
     异常 → 400 "Error attempting ping"
```

`getServiceItem` 的查找链路（[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/config/service-helpers.js#L727-L752)）：
1. `servicesFromConfig()` → 在 `services.yaml` 中查找
2. `servicesFromDocker()` → 在 Docker 容器标签中查找
3. `servicesFromKubernetes()` → 在 K8s 资源中查找

### 6.2 `/api/siteMonitor` — HTTP 探测

关键文件：[siteMonitor.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/api/siteMonitor.js)

```
请求: GET /api/siteMonitor?groupName=G&serviceName=S
流程:
  1. 同样查找服务配置，取 service.siteMonitor 字段
  2. 若无 siteMonitor URL → 400 "No http monitor URL given"
  3. 先发 HEAD 请求，用 performance.now() 计时
  4. 若状态码 > 403，改用 GET 重试（兼容拒绝 HEAD 的服务器）
  5. 成功 → 200 { status: <httpStatus>, latency: <ms> }
     异常 → 400 "Error attempting http monitor"
```

---

## 七、刷新节奏与数据流时序

### 7.1 SWR 版本与全局配置

项目使用 SWR v2.4.1（见 [package.json](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/package.json#L41)）。

SWR 的全局配置有两层：

**App 级**（[\_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/_app.jsx#L75-L79)）：
```jsx
<SWRConfig
  value={{
    fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
  }}
>
```

**页面级**（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/index.jsx#L186)）：
```jsx
<SWRConfig value={{ fallback, fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()) }}>
```

两层配置均只声明了 `fetcher` 和（页面级的）`fallback`，**未显式覆盖任何 SWR 行为开关**，因此所有组件均采用 SWR v2 的**默认值**。

Ping/SiteMonitor 组件自身的 SWR 调用（[ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.jsx#L6-L8)、[site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.jsx#L6-L8)）：
```jsx
const { data, error } = useSWR(url, { refreshInterval: 30000 });
```
只显式设置了 `refreshInterval: 30000`，其余也走 SWR 默认。

### 7.2 SWR v2 与探测相关的默认行为

下表列出与 Ping / SiteMonitor 探测请求直接相关的 SWR v2 默认行为（基于 SWR v2.1.4 源码 `useSWRHandler` 中的决策逻辑，参考 UNPKG 分发代码）：

| SWR 选项 | 默认值 | 含义 | 对 Ping/SiteMonitor 的影响 |
|---|---|---|---|
| `revalidateOnFocus` | `true` | 页面重新获得焦点时触发重新验证 | 切回标签页或窗口时，会立即触发一次探测请求 |
| `revalidateOnReconnect` | `true` | 网络恢复在线时触发重新验证 | 网络断开→恢复后，立即触发一次探测请求 |
| `refreshWhenHidden` | `false` | 页面隐藏时是否允许按 `refreshInterval` 发起请求 | 切到后台标签页时，定时器到期后**跳过**请求 |
| `refreshWhenOffline` | `false` | 浏览器离线时是否允许按 `refreshInterval` 发起请求 | 断网时，定时器到期后**跳过**请求 |
| `focusThrottleInterval` | `5000` (ms) | 焦点重验的最小间隔（节流） | 5 秒内连续获得焦点，最多只触发 1 次额外探测 |
| `dedupingInterval` | `2000` (ms) | 相同 key 请求的去重窗口 | 2 秒内同一服务的多次请求会合并为一次 |
| `revalidateIfStale` | `true` | 有陈旧缓存数据时组件挂载是否重验 | 组件再次挂载且缓存存在旧数据时，仍会重新发起探测 |
| `revalidateOnMount` | `undefined` | 组件挂载时是否强制重验 | 见下方"revalidateOnMount 的确切含义" |

#### 7.2.1 `revalidateOnMount` 的确切含义

SWR v2 中 `revalidateOnMount` 的默认值是 **`undefined`**，而非简单的 `true` 或 `false`。其真实决策逻辑（来自 SWR 源码 `useSWRHandler` 的 `shouldStartRequest` 判断链）如下：

```
组件挂载时，是否发起首次请求？

1. 如果 revalidateOnMount 被显式设为 true/false
   → 直接使用该值，结束判断。

2. 否则（undefined，即默认情况）：
   a. 如果 key 为空或没有 fetcher → 不请求。
   b. 如果 isPaused() 返回 true → 不请求。
   c. 如果启用了 suspense 模式 → 不请求（由 Suspense 处理）。
   d. 如果 revalidateIfStale 被显式设置 → 使用该值。
   e. 其他全部情况 → 返回 true（请求）。
```

对 Ping / SiteMonitor 的实际影响：
- **首次访问页面**（缓存为空）：`revalidateOnMount` 未显式设置 → 命中兜底分支 → **true** → 组件挂载立即发请求。
- **路由切换后回到页面**（组件卸载又重新挂载，缓存中有旧数据即 "stale 数据"）：`revalidateIfStale` 默认 `true` → **true** → 组件挂载立即重新探测，旧数据先展示，新结果回来后替换。
- 简言之：在本项目的默认配置下，Ping/SiteMonitor 组件**每次挂载都会立即发起一次探测请求**。

#### 7.2.2 页面隐藏或离线时：定时器与跳过请求的关联

`refreshWhenHidden=false` 和 `refreshWhenOffline=false` 的作用方式需要澄清——**它们不是暂停定时器，而是定时器到期后检查条件再决定是否跳过请求**。

SWR 内部使用 `setInterval` 注册 `refreshInterval` 计时器，计时器在组件挂载时启动、卸载时清除，不因页面隐藏或离线而暂停。但在每次计时器回调被触发时，SWR 会先调用内部的 `isActive()` 函数：

```js
// SWR 源码中的 isActive
const isActive = () => getConfig().isVisible() && getConfig().isOnline();
```

- `isVisible()` 通过 `document.visibilityState !== 'hidden'` 或 `!document.hidden` 判断（当 `refreshWhenHidden=false` 时，页面隐藏会使 `isVisible()` 返回 false）。
- `isOnline()` 通过 `navigator.onLine` 判断（当 `refreshWhenOffline=false` 时，离线会使 `isOnline()` 返回 false）。

当 `isActive()` 返回 false 时，SWR 会**跳过本次定时请求**，但 `setInterval` 本身不会被清除，仍会按 30 秒的节奏继续触发回调——只是每次都被跳过。

对 Ping / SiteMonitor 的实际影响：
- 页面隐藏或离线期间：每隔 30 秒仍有一次回调 → 检查 `isActive()` → false → **跳过请求**，不消耗网络。
- 页面重新可见或重新在线：
  - **不等下一个 30 秒到点**，而是立刻由 `revalidateOnFocus` 或 `revalidateOnReconnect` 事件驱动一次即时请求。
  - 下一次 30 秒定时请求仍按**原有的时间节奏**发生（因为 interval 从未暂停）。例如在第 5 秒切后台，第 35 秒 interval 触发但被跳过，第 40 秒切回前台会立刻请求，第 65 秒 interval 又会再次请求。
- 这与"定时器暂停、从恢复时重新计时"的行为**不同**。

### 7.3 各场景下的探测行为详解

#### 场景 1：正常前台浏览（理想情况）

```
时间轴 (秒)
0          30         60         90         …
│          │          │          │
├─ 组件挂载，立即请求 ─┤          │          │
│  (revalidateOnMount 默认 → true) │         │          │
│          ├─ refreshInterval 触发 ─┤          │
│          │          ├─ refreshInterval 触发 ─┤
│          │          │          │
▼          ▼          ▼          ▼
请求间隔精确 30 秒
```

- `revalidateOnMount` 未显式设置 → 默认决策链兜底返回 **true** → 组件首次挂载立即发请求。
- `refreshInterval=30000` → 之后每隔 30 秒周期性请求。
- `isActive()` 返回 true（可见 + 在线）→ 每次定时器到期都真实发起请求。

#### 场景 2：页面被切到后台（隐藏）

```
前台       切到后台                           切回前台
   │           │                                │
   ├─ 请求 ───┤ 第35s interval到期              ├─ 立即请求 (revalidateOnFocus)
   │           │  → isActive()=false → 跳过    │  + focusThrottle 5s 节流
   │           │  第65s interval到期→ 跳过 …   │
   │           │                                │
   ▼           ▼                                ▼
          refreshWhenHidden=false           focusThrottleInterval=5s
          → 后台期间 interval 仍运行          → 5 秒内重复切回最多发 1 次
            但每次到期都检查 isActive()
            不可见则跳过，不消耗网络
```

- `refreshWhenHidden=false` → 标签页隐藏（如最小化、切到其他 Tab）时，`setInterval` **仍按 30 秒节奏持续触发**，但每次回调中 `isActive()` 因 `isVisible()=false` 返回 false → **跳过本次请求**。
- 重新切回前台：
  - `revalidateOnFocus=true` → 立即触发一次额外的探测请求（不等下一个 30 秒到点）。
  - `focusThrottleInterval=5000` → 如果用户在 5 秒内快速切出又切回，SWR 会节流，最多发 1 次焦点重验请求，避免探测风暴。
  - 30 秒定时器**没有重新计时**，仍按原有节奏继续。例如在第 5 秒切后台，第 35 秒和第 65 秒的 interval 都会到期但被跳过，第 40 秒切回前台会立刻触发一次焦点请求，第 65 秒的 interval 又会再次触发请求。

**注意**：项目中存在一个 `useWindowFocus` hook（[window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/hooks/window-focus.js)），但它只被 `Index` 组件用于检测窗口聚焦后主动 `mutateHash()`（检测配置变更），**与 Ping / SiteMonitor 的 SWR 行为无关**。Ping/SiteMonitor 的聚焦重验完全由 SWR 内部默认机制驱动。

#### 场景 3：浏览器离线（断网）

```
在线         离线                                重新在线
   │           │                                   │
   ├─ 请求 ───┤ 第35s interval到期                  ├─ 立即请求 (revalidateOnReconnect)
   │           │  → isActive()=false → 跳过        │
   │           │  第65s interval到期→ 跳过 …       │
   │           │                                   │
   ▼           ▼                                   ▼
          refreshWhenOffline=false
          → 离线期间 interval 仍运行
            但每次到期都检查 isActive()
            不在线则跳过，不消耗网络
```

- `refreshWhenOffline=false` → `navigator.onLine === false` 期间，`setInterval` 仍按 30 秒节奏触发，但 `isActive()` 因 `isOnline()=false` 返回 false → **跳过本次请求**。
- 网络恢复（`online` 事件）时：
  - `revalidateOnReconnect=true` → 立即触发一次探测请求，尽快反映恢复后的真实状态。
  - 30 秒定时器**没有重新计时**，仍按原有节奏继续。

#### 场景 4：多个相同服务（SWR 去重）

- `dedupingInterval=2000` → 若在 2 秒内对同一 URL（相同 `groupName` + `serviceName`）发起多次请求，SWR 只会实际发出 1 次网络请求，其他订阅者共享结果。
- 在 Homepage 场景中，每个服务只渲染一个 Ping / SiteMonitor 组件，因此通常不会触发去重；但如果同服务被多个 Tab 或布局重复引用，此机制可以避免重复探测。

#### 场景 5：服务端渲染（SSR）与首次加载

- 页面通过 `getStaticProps()`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/index.jsx#L55-L95)）预取 `/api/services`、`/api/bookmarks`、`/api/widgets` 等基础数据，放入 SWR `fallback`。
- **注意**：Ping (`/api/ping`) 和 SiteMonitor (`/api/siteMonitor`) 的数据**不在 SSR fallback 中**。
- 因此：
  - 首屏渲染时 Ping / SiteMonitor 组件没有缓存数据 → 进入 "加载中" 灰色状态。
  - 客户端 hydration 后组件挂载：`revalidateOnMount` 为 `undefined` → 决策链最终返回 **true**（有 `revalidateIfStale=true` 默认兜底）→ 立即发起第一次探测请求。
  - 30 秒 `setInterval` 在组件挂载 effect 中启动，从挂载时刻起每 30 秒触发一次回调。

### 7.4 完整时序图

```
用户打开页面
    │
    ├─ SSR 返回 HTML（无 Ping/SiteMonitor 数据）
    │
    ▼
客户端 hydration
    │
    ├─ Ping 组件挂载
    │   └─ revalidateOnMount=undefined → 决策链返回 true
    │      → 立即 GET /api/ping?…  (第 1 次)
    │   └─ setInterval 启动（每 30s 触发回调）
    │
    ├─ SiteMonitor 组件挂载
    │   └─ revalidateOnMount=undefined → 决策链返回 true
    │      → 立即 GET /api/siteMonitor?…  (第 1 次)
    │   └─ setInterval 启动（每 30s 触发回调）
    │
    │  【第 5 秒 - 用户切到其他标签页 - 页面隐藏】
    │
    │  【第 30 秒 - interval 回调到期】
    │   └─ isActive()=false（isVisible=false）→ 跳过请求
    │
    │  【第 60 秒 - interval 回调到期】
    │   └─ isActive()=false → 跳过请求
    │
    │  【第 70 秒 - 切回前台 - 页面重新聚焦】
    │   ├─ revalidateOnFocus=true
    │   │   → 立即 GET /api/ping?…  (第 2 次，不等 30s)
    │   │   → 立即 GET /api/siteMonitor?…  (第 2 次)
    │   ├─ focusThrottleInterval=5s → 5 秒内切回不重复请求
    │   └─ setInterval 继续按原节奏（第 90 秒再次回调）
    │
    │  【第 90 秒 - interval 回调到期】
    │   └─ isActive()=true → GET /api/ping?…  (第 3 次)
    │                        → GET /api/siteMonitor?…  (第 3 次)
    │
    │  【网络短暂断开又恢复】
    │   ├─ 离线期间 interval 到期 → isActive()=false（isOnline=false）→ 跳过
    │   ├─ revalidateOnReconnect=true
    │   │   → 立即 GET /api/ping?…  (第 4 次)
    │   │   → 立即 GET /api/siteMonitor?…  (第 4 次)
    │   └─ setInterval 继续按原节奏
    │
    │  【正常前台，无人操作】
    │   ├─ 每 30s → GET /api/ping?…  + GET /api/siteMonitor?…
    │   └─ ……
    │
    ▼
探测请求并非严格每 30 秒一次，
而是"30 秒定时（到期时 isActive 检查后可能跳过）
    + 聚焦触发（revalidateOnFocus）
    + 重连触发（revalidateOnReconnect）"三者的叠加
```

#### 7.4.1 Ping / SiteMonitor 实际触发时机汇总

综合以上所有机制，Ping / SiteMonitor 的探测请求会在以下**六种**时机被触发：

| # | 触发时机 | 驱动来源 | 实际行为 |
|---|---|---|---|
| 1 | 组件首次挂载（页面打开或路由进入） | `revalidateOnMount=undefined` → 默认决策链返回 `true` | 立即发起一次探测，无任何缓存时同时显示加载态 |
| 2 | 组件重新挂载（路由切回）且缓存有 stale 数据 | `revalidateIfStale=true`（默认） | 先展示旧数据，同时立即发起一次探测，新结果回来后替换 |
| 3 | 每 30 秒定时到期且页面可见、在线 | `refreshInterval=30000` + `isActive()=true` | 每 30 秒发起一次周期性探测 |
| 4 | 每 30 秒定时到期但页面隐藏或离线 | `refreshInterval=30000` + `isActive()=false` | **跳过本次请求**，不消耗网络；定时器仍继续计时 |
| 5 | 页面重新获得焦点（切回标签页/窗口） | `revalidateOnFocus=true`（默认）+ `focusThrottleInterval=5000` | 立即发起一次探测；5 秒内重复聚焦仅发一次 |
| 6 | 网络从离线恢复在线 | `revalidateOnReconnect=true`（默认） | 立即发起一次探测，确保恢复后状态及时更新 |

### 7.5 小结

刷新节奏的真实特征：

1. **基础周期**：每组件独立 `refreshInterval: 30000`（30 秒），硬编码不可配。定时器自组件挂载时启动，组件卸载时清除，**不因页面隐藏或离线而暂停**。
2. **后台跳过而非暂停**：页面隐藏或离线时，定时器仍每 30 秒触发回调，但通过 `isActive()`（`isVisible() && isOnline()`）检查后**跳过请求**，不消耗网络资源。
3. **事件驱动即时刷新**：页面重新获得焦点（`revalidateOnFocus`）或网络恢复（`revalidateOnReconnect`）时，SWR 会立即发起探测请求，不等下一个定时到点，确保状态及时更新。
4. **挂载即请求**：`revalidateOnMount` 默认值为 `undefined`，在本项目配置下，组件每次挂载（无论首次还是重新挂载）都会决策为 `true`，立即发起探测。
5. **防风暴保护**：`focusThrottleInterval=5000` 防止快速切换标签页造成探测风暴；`dedupingInterval=2000` 合并 2 秒窗口内对同一 key 的重复请求。
6. **首屏行为**：Ping/SiteMonitor 不在 SSR fallback 中，首屏显示灰色加载态，hydration 后立即开始探测。

---

## 八、信息条的视觉层级

```
┌─────────────────────────────────────────┐
│ [icon]  Service Name           [标签区] │
│         Description text                 │
│                                  ┌─────┐│
│                                  │Ping ││ ← service-ping
│                                  └─────┘│
│                                  ┌─────┐│
│                                  │ HTTP ││ ← service-site-monitor
│                                  └─────┘│
└─────────────────────────────────────────┘
```

- 标签区位于卡片右上角（`absolute top-0 right-0`），不影响卡片内容布局。
- 多个标签水平排列，`gap-2` 间距（dot 模式下 `gap-0`）。
- 标签组件只做展示，不接收点击事件（与 Docker status 标签不同，后者可点击展开统计信息）。
- 标签使用 `rounded-b-[3px]` 实现底部微圆角效果，视觉上贴合卡片顶边。

---

## 九、三种 style 模式对比

| 特性 | 默认 (无 style) | basic | dot |
|---|---|---|---|
| **Ping 在线** | 显示延迟毫秒数 | 显示 "UP" | 显示绿色圆点 |
| **Ping 离线** | 显示 "DOWN" | 显示 "DOWN" | 显示红色圆点 |
| **SiteMonitor 在线** | 显示延迟毫秒数 | 显示 "UP" | 显示绿色圆点 |
| **SiteMonitor 离线** | 显示 HTTP 状态码 | 显示 "DOWN" | 显示红色圆点 |
| **加载中** | 灰色 "Ping"/"Response" | 同左 | 灰色圆点 |
| **错误** | 红色 "Error" | 同左 | 红色圆点 |
| **文字样式** | 8px 加粗大写 | 同左 | 无文字 |
| **颜色映射** | `text-*` | `text-*` | `bg-*`（CSS 类名替换） |

---

## 十、数据结构一览

### Ping API 响应

```json
// 成功
{ "alive": true, "time": 12.3, "host": "example.com", … }

// 失败
{ "alive": false, "time": 0 }
```

`time` 单位为毫秒，来自底层 `ping.probe()` 返回值。

### SiteMonitor API 响应

```json
// 成功
{ "status": 200, "latency": 45.67 }

// 服务端错误（仍然成功返回，status > 403）
{ "status": 500, "latency": 12.3 }
```

`latency` 单位为毫秒，由 `performance.now()` 差值计算。

---

## 十一、关键文件索引

| 文件 | 职责 |
|---|---|
| [ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.jsx) | Ping 前端组件：SWR 轮询 + 状态渲染 |
| [site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.jsx) | SiteMonitor 前端组件：SWR 轮询 + 状态渲染 |
| [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/item.jsx) | 服务卡片：条件挂载 Ping / SiteMonitor |
| [ping.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/api/ping.js) | Ping 后端 API：ICMP 探测 |
| [siteMonitor.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/api/siteMonitor.js) | SiteMonitor 后端 API：HTTP 探测 |
| [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/config/service-helpers.js) | 服务配置解析 + getServiceItem 查找链 |
| [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/config/api-response.js) | 服务数据汇总与排序，输出给 /api/services |
| [settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/contexts/settings.jsx) | 全局设置上下文（含 statusStyle） |
| [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/_app.jsx) | 应用级 SWRConfig（全局 fetcher，使用默认行为） |
| [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/index.jsx) | 页面级 SWRConfig + SSR fallback + useWindowFocus |
| [window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/hooks/window-focus.js) | 窗口焦点 hook（仅用于 hash 检测，与探测 SWR 无关） |
| [package.json](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/package.json) | 项目依赖：swr v2.4.1 |
| [ping.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.test.jsx) | Ping 组件单元测试 |
| [site-monitor.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.test.jsx) | SiteMonitor 组件单元测试 |
| [ping.test.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/__tests__/pages/api/ping.test.js) | Ping API 单元测试 |
