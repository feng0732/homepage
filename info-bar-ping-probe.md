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

项目使用 **SWR v2.4.1**（见 [package.json](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/package.json#L41)，npm 记录该版本发布于 2025 年 3 月）。

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

两层配置均只声明了 `fetcher` 和（页面级的）`fallback`，**未显式覆盖任何 SWR 行为开关**，因此所有组件均采用 SWR v2.4.1 的**默认值**。

Ping/SiteMonitor 组件自身的 SWR 调用（[ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.jsx#L6-L8)、[site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.jsx#L6-L8)）：
```jsx
const { data, error } = useSWR(url, { refreshInterval: 30000 });
```
只显式设置了 `refreshInterval: 30000`，其余也走 SWR 默认。

### 7.2 SWR v2.4.1 与探测相关的默认行为

下表列出与 Ping / SiteMonitor 探测请求直接相关的 SWR v2.4.1 默认行为（参考 SWR GitHub 源码 `src/index/use-swr.ts`、`src/_internal/` 及测试用例 `test/use-swr-refresh.test.tsx`）：

| SWR 选项 | 默认值 | 含义 | 对 Ping/SiteMonitor 的影响 |
|---|---|---|---|
| `revalidateOnFocus` | `true` | 页面重新获得焦点时触发重新验证 | 切回标签页或窗口时，会立即触发一次探测请求 |
| `revalidateOnReconnect` | `true` | 网络恢复在线时触发重新验证 | 网络断开→恢复后，立即触发一次探测请求 |
| `refreshWhenHidden` | `false` | 页面隐藏时是否允许按 `refreshInterval` 发起请求 | 切到后台标签页时，**停止调度下一次 setTimeout** |
| `refreshWhenOffline` | `false` | 浏览器离线时是否允许按 `refreshInterval` 发起请求 | 断网时，**停止调度下一次 setTimeout** |
| `focusThrottleInterval` | `5000` (ms) | 焦点重验的最小间隔（节流） | 5 秒内连续获得焦点，最多只触发 1 次额外探测 |
| `dedupingInterval` | `2000` (ms) | 相同 key 请求的去重窗口 | 2 秒内同一服务的多次请求会合并为一次 |
| `revalidateIfStale` | `true` | 有陈旧缓存数据时组件挂载是否重验 | 组件再次挂载且缓存存在旧数据时，仍会重新发起探测 |
| `revalidateOnMount` | `undefined` | 组件挂载时是否强制重验 | 见下方"revalidateOnMount 的确切含义" |

#### 7.2.1 `revalidateOnMount` 的确切含义

SWR v2.4.1 中 `revalidateOnMount` 的默认值是 **`undefined`**，而非简单的 `true` 或 `false`。其真实决策逻辑（来自 SWR 源码 `useSWRHandler` 的 `shouldStartRequest` 判断链）如下：

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

#### 7.2.2 轮询机制：递归 setTimeout 而非固定 setInterval

SWR v2.4.1 的 `refreshInterval` 轮询**不是用固定节奏的 `setInterval` 实现，而是用递归 `setTimeout` 实现**。其核心流程（基于 SWR v2.1.4 源码 `core/dist/index.mjs` 中 `Polling` 部分的真实代码）：

```
组件挂载 → 轮询 effect 启动
  │
  └─ 调用 next()  ← 入口
        │
        └─ setTimeout(execute, 30000)  ← 安排第一次 execute

30 秒后 → execute() 被触发
  │
  ├─ 条件检查：
  │   !cache.error
  │   && (refreshWhenHidden || isVisible())
  │   && (refreshWhenOffline || isOnline())
  │
  ├─ 条件为 true（无错误、可见且在线）
  │     │
  │     ├─ revalidate(WITH_DEDUPE) → 发起探测请求
  │     │     │
  │     │     └─ Promise resolve → .then(next)
  │     │              │
  │     │              └─ next() → setTimeout(execute, 30000)  ← 递归调度①
  │     │
  │     └─ 条件为 false（有错误、或隐藏、或离线）
  │           │
  │           └─ // Schedule the next interval to check again.
  │              next()  ← else 分支：不发起请求，直接安排下一次检查
  │                   │
  │                   └─ setTimeout(execute, 30000)  ← 递归调度②，调度链不中断
  │
  ▼
递归 setTimeout 的关键特征：
  • 两次 execute 之间的间隔固定为 30 秒（无论是否发起请求）
  • if 分支：请求完成后再 next → 两次请求间隔 = 30s + 请求耗时
  • else 分支：直接 next → 两次检查间隔 = 精确 30s（不请求）
  • 调度链永不断裂（除非组件卸载，timer 被设为 -1）
  • 支持 refreshInterval(data) 动态计算下一次间隔
```

以下是 SWR v2.1.4 源码中的真实代码片段（来自 `core/dist/index.mjs` 的 `Polling` 部分）：

```js
// Polling
useIsomorphicLayoutEffect(()=>{
    let timer;
    function next() {
        const interval = isFunction(refreshInterval) ? refreshInterval(getCache().data) : refreshInterval;
        if (interval && timer !== -1) {
            timer = setTimeout(execute, interval);
        }
    }
    function execute() {
        // Check if it's OK to execute:
        if (!getCache().error && (refreshWhenHidden || getConfig().isVisible()) && (refreshWhenOffline || getConfig().isOnline())) {
            revalidate(WITH_DEDUPE).then(next);
        } else {
            // Schedule the next interval to check again.
            next();  // ← else next 逻辑：跳过请求，但仍安排下一次检查！
        }
    }
    next();
    return ()=>{
        if (timer) {
            clearTimeout(timer);
            timer = -1;
        }
    };
}, [refreshInterval, refreshWhenHidden, refreshWhenOffline, key]);
```

**重要结论**：
- `execute` 的条件判断不是 `isActive()`，而是 `!error && (refreshWhenHidden || visible) && (refreshWhenOffline || online)`。
- **无论条件真假，最终都会调用 `next()`** → 递归调度链**永远不会断裂**。
- 隐藏/离线时的真实行为是"跳过请求，但仍安排下一次 30 秒后的检查"，不是"停止调度"。

#### 7.2.3 页面隐藏或离线时：跳过请求 + 继续安排下一次检查

配合上述递归 `setTimeout` 机制，`refreshWhenHidden=false` 和 `refreshWhenOffline=false` 的真实作用需要重新理解——它们不是"停止调度"，也不是"固定 interval 持续运行每次跳过"，而是"**递归 setTimeout 持续调度，每次到期检查条件后决定是否发起请求，但永远安排下一次检查**"。

SWR 内部的条件判断：

```
每次 setTimeout 到期 → execute() 被触发
  │
  ├─ 条件 = !error && (refreshWhenHidden || visible) && (refreshWhenOffline || online)
  │
  ├─ true → revalidate().then(next)  ← 发起请求，完成后再等 30s 检查
  │
  └─ false → next()  ← 不请求，立即等 30s 再检查
```

对 Ping / SiteMonitor 的实际影响：
- 页面隐藏或离线期间：每隔 30 秒 `execute()` 都会被触发 → 检查条件 → false → **不发起请求**，但**立即调用 `next()` 安排下一次 30 秒后的检查**。
- 调度链从未断裂，隐藏/离线期间一直有 `setTimeout` 在运行（每次到期都安排下一次），只是不发起网络请求。
- 页面重新可见或重新在线：
  - 不需要等"恢复事件"来重建调度——因为调度本来就没停。
  - **如果恰好赶在某次 `execute` 到期时条件已经恢复**：该次检查会直接命中 true 分支，发起请求。
  - **更常见的情况**：由 `revalidateOnFocus` 或 `revalidateOnReconnect` 事件在恢复的瞬间**立即**触发一次请求（不等下一个 30 秒到点）。
  - 该请求完成后 `.then(next)` → 重新从完成时刻开始计时 30 秒。
- 这意味着：隐藏/离线期间调度持续在后台"空转"（只检查不请求），恢复时可以由事件立即触发请求，也可以等到下一次调度到期时自然请求。

### 7.3 各场景下的探测行为详解

#### 场景 1：正常前台浏览（理想情况）

```
时间轴 (秒)
0                ~0.1s 完成    30.1s          60.2s          90.3s
│                  │            │              │              │
├─ 组件挂载，立即请求 ─┤            │              │              │
│  (revalidateOnMount → true) │              │              │
│                  ├─ next() → setTimeout(execute, 30000)  │              │
│                               ├─ 到期 → isActive=true → 发起请求 (第 2 次)
│                               │              ├─ 到期 → 第 3 次请求
│                               │              │              ├─ 第 4 次请求
▼                               ▼              ▼              ▼
"完成→下一次开始"间隔 30 秒（请求耗时 ~0.1s，开始时间略有漂移）
```

- `revalidateOnMount` 未显式设置 → 默认决策链兜底返回 **true** → 组件首次挂载立即发请求（第 1 次）。
- 请求完成后调用 `next()` → `setTimeout(execute, 30000)`，开始递归调度。
- 每次 setTimeout 到期时 `!error && visible && online` 条件为 true → 发起请求 → 请求完成后再次 `setTimeout(execute, 30000)`，循环继续。
- **注意**：两次请求的"开始间隔"为 `30s + 请求耗时`，不是精确 30 秒。

#### 场景 2：页面被切到后台（隐藏）

```
前台       切到后台                                        切回前台
   │           │                                             │
   ├─ 请求 ①完成  │ 第30s setTimeout到期                     ├─ 立即请求 ② (revalidateOnFocus)
   │           │   → 条件=false（hidden）→ 不请求           │  + focusThrottle 5s 节流
   │           │   → else next() → 安排第60s 检查          │  + 请求完成 → .then(next) → 第70s+30s ③
   │           │                                             │
   │           │  第60s setTimeout到期                      │
   │           │   → 条件=false → 不请求 → next() → 第90s  │
   │           │                                             │
   │           │  …持续每30s检查，不请求…                    │
   │           │                                             │
   ▼           ▼                                             ▼
          refreshWhenHidden=false                        focusThrottleInterval=5s
          → 条件false走 else next() 分支                  → 5 秒内重复切回最多发 1 次
            跳过请求，但仍安排下一次检查                  → 事件请求完成后重新计时
            调度链不中断（与"调度停止"本质不同）
```

- `refreshWhenHidden=false` → 当某次 `setTimeout` 到期时若 `isVisible()=false`：
  - 命中 `else` 分支 → **不发起请求**，但**直接调用 `next()` 安排下一次 30 秒后的检查** → 递归调度链**不中断**。
  - 页面隐藏期间每隔 30 秒都会触发一次 `execute()` 检查，只是不发起网络请求。
- 重新切回前台：
  - `revalidateOnFocus=true` → 立即触发一次即时请求（第 2 次，完全同步于切回动作，不等下一个 30 秒到点）。
  - `focusThrottleInterval=5000` → 5 秒内频繁切出/切回，最多只发 1 次焦点重验请求。
  - 该请求完成后 → `.then(next)` → `setTimeout(execute, 30000)` → 新的 30 秒节奏从请求完成时开始。
  - **注意**：即使没有焦点事件，调度本来也在运行——下一次 `execute()` 到期时条件已恢复为 true，也会自然发起请求。

**注意**：项目中存在一个 `useWindowFocus` hook（[window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/utils/hooks/window-focus.js)），但它只被 `Index` 组件用于检测窗口聚焦后主动 `mutateHash()`（检测配置变更），**与 Ping / SiteMonitor 的 SWR 行为无关**。Ping/SiteMonitor 的聚焦重验完全由 SWR 内部默认机制驱动。

#### 场景 3：浏览器离线（断网）

```
在线         离线                                           重新在线
   │           │                                               │
   ├─ 请求 ①完成  │ 第30s setTimeout到期                       ├─ 立即请求 ② (revalidateOnReconnect)
   │           │   → 条件=false（offline）→ 不请求             │  + 请求完成 → .then(next) → 第70s+30s ③
   │           │   → else next() → 安排第60s 检查              │
   │           │                                               │
   │           │  第60s setTimeout到期                        │
   │           │   → 条件=false → 不请求 → next() → 第90s    │
   │           │                                               │
   │           │  …持续每30s检查，不请求…                      │
   │           │                                               │
   ▼           ▼                                               ▼
          refreshWhenOffline=false
          → 条件false走 else next() 分支
            跳过请求，但仍安排下一次检查
            调度链不中断
```

- `refreshWhenOffline=false` → `navigator.onLine === false` 时，若某次到期检查发现 `isOnline()=false`：
  - 命中 `else` 分支 → **不发起请求**，但**直接调用 `next()` 安排下一次检查** → 调度链**不中断**。
- 网络恢复（`online` 事件）时：
  - `revalidateOnReconnect=true` → 立即触发一次请求（第 2 次），不等下一次调度到点。
  - 请求完成后 → `.then(next)` → 重新从完成时刻开始计时 30 秒。

#### 场景 4：多个相同服务（SWR 去重）

- `dedupingInterval=2000` → 若在 2 秒内对同一 URL（相同 `groupName` + `serviceName`）发起多次请求，SWR 只会实际发出 1 次网络请求，其他订阅者共享结果。
- 在 Homepage 场景中，每个服务只渲染一个 Ping / SiteMonitor 组件，因此通常不会触发去重；但如果同服务被多个 Tab 或布局重复引用，此机制可以避免重复探测。

#### 场景 5：服务端渲染（SSR）与首次加载

- 页面通过 `getStaticProps()`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/pages/index.jsx#L55-L95)）预取 `/api/services`、`/api/bookmarks`、`/api/widgets` 等基础数据，放入 SWR `fallback`。
- **注意**：Ping (`/api/ping`) 和 SiteMonitor (`/api/siteMonitor`) 的数据**不在 SSR fallback 中**。
- 因此：
  - 首屏渲染时 Ping / SiteMonitor 组件没有缓存数据 → 进入 "加载中" 灰色状态。
  - 客户端 hydration 后组件挂载：`revalidateOnMount` 为 `undefined` → 决策链最终返回 **true**（有 `revalidateIfStale=true` 默认兜底）→ 立即发起第一次探测请求。
  - 第一次请求完成后 → `next()` → `setTimeout(execute, 30000)` → 递归轮询开始建立。

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
    │      → 立即 GET /api/ping?…  (第 1 次请求)
    │   └─ 第 1 次请求完成 → next() → setTimeout(execute, 30000)  ← 建立递归调度
    │
    ├─ SiteMonitor 组件挂载
    │   └─ revalidateOnMount=undefined → 决策链返回 true
    │      → 立即 GET /api/siteMonitor?…  (第 1 次请求)
    │   └─ 第 1 次请求完成 → next() → setTimeout(execute, 30000)  ← 建立递归调度
    │
    │  【第 ~30 秒 - setTimeout 到期 - 用户已切后台 - 页面隐藏】
    │   └─ execute() 被触发 → 条件=false（!visible）
    │      → 不发起请求
    │      → else next() → setTimeout(execute, 30000)  ← 安排第 60 秒检查，调度链不中断！
    │
    │  【第 60 秒 - setTimeout 到期 - 仍隐藏】
    │   └─ execute() 被触发 → 条件=false
    │      → 不发起请求
    │      → else next() → setTimeout(execute, 30000)  ← 安排第 90 秒检查
    │
    │  【第 70 秒 - 切回前台 - 页面重新聚焦】
    │   ├─ revalidateOnFocus=true
    │   │   → 立即 GET /api/ping?…  (第 2 次请求)
    │   │   → 立即 GET /api/siteMonitor?…  (第 2 次请求)
    │   ├─ focusThrottleInterval=5s → 5 秒内切回不重复请求
    │   └─ 第 2 次请求完成 → .then(next) → setTimeout(execute, 30000)  ← 从此时重新计时
    │
    │  【第 100 秒 - setTimeout 到期（来自第 70 秒重新计时）- 可见在线】
    │   └─ 条件=true
    │      → GET /api/ping?…  (第 3 次请求)
    │      → GET /api/siteMonitor?…  (第 3 次请求)
    │      → 请求完成 → .then(next) → setTimeout(execute, 30000)  ← 继续循环
    │
    │  【网络短暂断开又恢复】
    │   ├─ 断开期间某次 setTimeout 到期 → 条件=false → else next() → 安排下一次
    │   ├─ revalidateOnReconnect=true
    │   │   → 立即 GET /api/ping?…  (第 4 次请求)
    │   │   → 立即 GET /api/siteMonitor?…  (第 4 次请求)
    │   └─ 请求完成 → .then(next) → setTimeout(30s)  ← 重新计时
    │
    │  【正常前台，无人操作】
    │   ├─ 每次 setTimeout 到期 + 条件=true → 发起请求 → .then(next) → 再 setTimeout
    │   └─ 递归循环持续进行
    │
    ▼
探测请求的真实节奏 =
    "组件挂载即请求（第 1 次）
    + 递归 setTimeout 30s 永不中断
       （到期条件 true 则请求 + .then(next)
        到期条件 false 则 else next()，继续 30s 后检查）
    + 页面聚焦时由 revalidateOnFocus 立即请求（不等到期）
    + 网络重连时由 revalidateOnReconnect 立即请求（不等到期）"
    的复合效果
```

#### 7.4.1 Ping / SiteMonitor 实际触发时机汇总

综合以上所有机制，Ping / SiteMonitor 的探测请求会在以下**六种**时机被触发（注意"跳过请求但继续调度"与"真正触发请求"的区别）：

| # | 时机 | 驱动来源 | 是否实际发起探测请求 | 对递归调度链的影响 |
|---|---|---|---|---|
| 1 | 组件首次挂载（页面打开或路由进入） | `revalidateOnMount=undefined` → 默认决策链返回 `true` | ✅ **立即发起**（无缓存时同时显示加载态） | 请求完成后 `.then(next)` → `setTimeout(execute, 30000)` → 建立调度 |
| 2 | 组件重新挂载（路由切回）且缓存有 stale 数据 | `revalidateIfStale=true`（默认） | ✅ **立即发起**（先展示旧数据，新结果回来后替换） | 请求完成后 `.then(next)` → 建立调度 |
| 3 | setTimeout 到期且 `!error && visible && online` | `refreshInterval=30000` + 条件为 true | ✅ **发起周期性探测** | 请求完成后 `.then(next)` → `setTimeout(execute, 30000)` → 循环继续 |
| 4 | setTimeout 到期但条件为 false（隐藏/离线/有 error） | `refreshInterval=30000` + 条件为 false | ❌ **不发起** | **`else next()` → 立即安排下一次 30s 检查**，调度链**不中断** |
| 5 | 页面重新获得焦点（切回标签页/窗口） | `revalidateOnFocus=true`（默认）+ `focusThrottleInterval=5000` | ✅ **立即发起**；5 秒内重复聚焦仅发 1 次 | 请求完成后 `.then(next)` → 从此时起重新计时 30s |
| 6 | 网络从离线恢复在线 | `revalidateOnReconnect=true`（默认） | ✅ **立即发起**，确保恢复后状态及时更新 | 请求完成后 `.then(next)` → 从此时起重新计时 30s |

### 7.5 小结

刷新节奏的真实特征：

1. **SWR 版本**：项目依赖 **SWR v2.4.1**（npm 发布于 2025 年 3 月），其轮询机制经对 SWR v2.1.4 源码 `core/dist/index.mjs` 中 `Polling` 部分的真实代码验证，v2.1.x 到 v2.4.x 的核心轮询逻辑一致。
2. **轮询实现**：`refreshInterval` 底层采用**递归 `setTimeout`**，不是固定 `setInterval`。核心代码模式为 `execute()` 中的 `if (condition) { revalidate().then(next) } else { next() }`——**无论条件真假，两个分支最终都调用 `next()`** → 递归调度链**永不断裂**（除非组件卸载 `timer` 被设为 -1）。
3. **隐藏/离线时的真实行为（else next 逻辑）**：不是"停止调度"，也不是"固定 interval 跳过"，而是"**每次到期检查条件，条件 false 则命中 else 分支，跳过请求但立即调用 `next()` 安排下一次 30 秒检查**"。隐藏/离线期间调度链持续在后台"空转"（只检查不请求），每隔 30 秒有一次 `execute()` 回调，只是不发起网络请求。
4. **恢复机制**：当页面重新聚焦或网络重连时，调度链本来就没断——`revalidateOnFocus` / `revalidateOnReconnect` 事件的作用是**不等下一次调度到点，立即触发一次请求**，并在该请求完成后 `.then(next)` → 新的 30 秒节奏从恢复请求完成时**重新开始计算**（打断了原本每 30 秒的空转节奏）。
5. **条件判断细节**：不是简单的 `isActive()`，而是 `!cache.error && (refreshWhenHidden || visible) && (refreshWhenOffline || online)`。`refreshWhenHidden/Offline` 为 true 时可以绕过可见性/在线检查。
6. **挂载即请求**：`revalidateOnMount` 默认值为 `undefined`，在本项目默认配置下，组件每次挂载（无论首次还是重新挂载）都会决策为 `true`，立即发起探测。
7. **防风暴保护**：`focusThrottleInterval=5000` 防止快速切换标签页造成探测风暴；`dedupingInterval=2000` 合并 2 秒窗口内对同一 key 的重复请求。
8. **首屏行为**：Ping/SiteMonitor 不在 SSR fallback 中，首屏显示灰色加载态，hydration 后立即开始探测，请求完成后启动递归调度。

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
