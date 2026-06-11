# Homepage Info-Bar Ping / SiteMonitor 探测刷新节奏分析

本文档详细分析 Info-Bar 区域（服务卡片右上角状态指示器）中 Ping 和 SiteMonitor 两类探测的刷新节奏，重点结合 SWR 库的默认行为，澄清页面隐藏、离线状态、重新聚焦时的实际表现与常见误解。

---

## 一、架构总览

服务卡片的右上角状态指示器有 3 类，其中与"主动探测"相关的是两类：

| 指示器类型 | 组件文件 | API 端点 | 探测方式 | 刷新周期 |
|------------|----------|----------|----------|----------|
| **Ping** | [components/services/ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/ping.jsx) | `/api/ping` | 服务端 ICMP `ping.probe(hostname)` | 30s |
| **SiteMonitor** | [components/services/site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/site-monitor.jsx) | `/api/siteMonitor` | 服务端 HTTP HEAD（失败重试 GET） | 30s |
| Docker Status | [components/services/status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/status.jsx) | `/api/docker/status/...` | 读 Docker daemon 状态 | 无固定周期（SWR 默认） |

两类探测组件代码结构几乎一致，都是直接调用 `useSWR(url, { refreshInterval: 30000 })`，未显式配置 `revalidateOnFocus`、`revalidateOnReconnect` 等选项，因此全部沿用 **SWR v2.4.1 的默认值**。

---

## 二、SWR v2.4 默认行为速查（与探测节奏强相关）

项目依赖版本：[package.json#L41](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/package.json#L41) — `"swr": "^2.4.1"`

项目全局 SWR 配置（未覆盖任何节奏选项，全默认）：
[pages/index.jsx#L186](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L186)
```javascript
<SWRConfig value={{
  fallback,
  fetcher: (resource, init) => fetch(resource, init).then((res) => res.json())
}}>
```

以下是 SWR v2.4 中**影响刷新节奏**的关键配置默认值：

| 配置项 | 默认值 | 含义 |
|--------|--------|------|
| `revalidateOnFocus` | `true` | 窗口/标签页**重新获得焦点**时自动重验证 |
| `revalidateOnReconnect` | `true` | 浏览器**从离线恢复到在线**时自动重验证 |
| `revalidateOnMount` | `true` | 组件挂载时（首次出现）自动请求（但有 fallback/缓存则不一定发网络请求） |
| `refreshInterval` | `undefined`（关闭） | 固定周期轮询，Ping/SiteMonitor 显式设为 `30000` |
| `refreshWhenHidden` | `false` | **页面不可见时是否继续按 refreshInterval 轮询**（默认停） |
| `refreshWhenOffline` | `false` | **浏览器离线时是否继续按 refreshInterval 轮询**（默认停） |
| `dedupingInterval` | `2000`（2s） | 2 秒内同一 key 的多次请求被去重，只发一次网络请求 |
| `focusThrottleInterval` | `5000`（5s） | 连续 focus 事件之间的最小间隔，避免频繁切标签时刷屏请求 |

> **重要结论先行**：Ping/SiteMonitor 显式只设了 `refreshInterval: 30000`，其余全部走默认值。因此页面隐藏时**不轮询**、离线时**不轮询**、重新聚焦时**额外触发一次**请求（但受 5s focus 节流约束）。

---

## 三、三类场景的精确行为分析

### 场景 1：页面隐藏（切换到其他标签页 / 最小化浏览器）

#### 触发条件
浏览器触发 `visibilitychange` 事件且 `document.hidden === true`。注意这与 `window.blur` 不同：
- `visibilitychange`：**标签页不可见**（切标签、最小化、其他应用全屏遮挡）
- `window.blur`：**焦点丢失**（点击地址栏、打开 DevTools、弹窗）— 不影响可见性

SWR 使用的是 **Page Visibility API**（`visibilitychange`），不是 focus/blur 事件。

#### Ping/SiteMonitor 的实际表现

**`refreshWhenHidden: false`（默认） + `refreshInterval: 30000`** → 页面隐藏期间，**30 秒定时器被完全暂停**。

时序示意（假设 T+10s 切走，T+70s 切回）：
```
T+0s   [visible]  首次挂载  → 请求 #1 (挂载触发)
T+10s  [hidden]   切走标签   → refreshInterval 定时器 **暂停**
T+30s  [hidden]   到了轮询点 → ❌ 不触发（refreshWhenHidden=false）
T+60s  [hidden]   到了轮询点 → ❌ 不触发
T+70s  [visible]  切回标签   → ✅ 触发请求 #2（revalidateOnFocus=true）
T+100s [visible]             → ✅ 触发请求 #3（refreshInterval 恢复计时，距 T+70s 过了 30s）
```

#### 常见误解澄清

| 误解 | 实际情况 |
|------|----------|
| "隐藏时也每 30 秒探测一次" | ❌ `refreshWhenHidden` 默认为 `false`，隐藏期间定时器**完全挂起** |
| "隐藏时定时器继续走，只是不发请求" | ❌ 不是"发请求被拦截"，而是 SWR 内部根本不启动下一轮 `setTimeout`。`refreshWhenHidden` 的实现是：当 `document.hidden` 时直接 `return`，不安排下一次定时。 |
| "切回来时会把欠的请求都补上" | ❌ 不会"补偿"。只发一次 revalidateOnFocus 触发的请求，之后恢复正常 30s 节奏。 |

---

### 场景 2：浏览器离线状态（断网 / 飞行模式）

#### 触发条件
`navigator.onLine === false` 并触发 `offline` 事件。SWR 通过监听 `window.offline` / `window.online` 事件感知。

#### Ping/SiteMonitor 的实际表现

**`refreshWhenOffline: false`（默认） + `refreshInterval: 30000`** → 离线期间，**30 秒定时器同样被暂停**。

时序示意（假设 T+20s 断网，T+140s 恢复网络）：
```
T+0s   [online]   挂载     → 请求 #1
T+20s  [offline]  断网     → refreshInterval 定时器暂停
T+30s  [offline]  轮询点   → ❌ 不触发
T+60s  [offline]  轮询点   → ❌ 不触发
...
T+140s [online]   恢复网络 → ✅ 触发请求 #2（revalidateOnReconnect=true）
T+170s [online]           → ✅ 触发请求 #3（refreshInterval 恢复计时）
```

#### 离线期间请求尝试的情况
有一个例外：如果用户在离线状态下执行了会触发 revalidate 的操作（如手动 `mutate()` 被代码调用、组件首次挂载），SWR 仍会调用 fetcher。但由于浏览器离线，`fetch()` 会立即失败并进入 error 状态。

对于 Ping/SiteMonitor，组件通常在断网前已挂载，因此离线期间**不会主动触发新请求**。

---

### 场景 3：重新聚焦（切回标签页 / 从其他应用切回浏览器）

#### 触发条件
页面从 `document.hidden === true` → `false`（`visibilitychange`），或者窗口从 blur → focus（`window.focus`）。SWR 两者都监听。

#### Ping/SiteMonitor 的实际表现

**`revalidateOnFocus: true`（默认） + `focusThrottleInterval: 5000`（默认 5s）** → 每次切回标签页会**额外触发一次请求**，但 5 秒内多次切回只会发一次。

时序示意（用户疯狂切换标签）：
```
T+0s   挂载           → 请求 #1 (挂载)
T+5s   切走 → 切回    → 请求 #2 (revalidateOnFocus)
T+7s   切走 → 切回    → ❌ 被 focusThrottleInterval=5s 吞掉（距 T+5s 仅 2s）
T+11s  切走 → 切回    → ✅ 请求 #3（距 T+5s 已 6s，超过 5s 节流窗口）
T+35s  无操作         → ✅ 请求 #4（refreshInterval 30s 到期，从 T+5s 算起）
```

#### 与 `useWindowFocus` 的区分
项目中有一个自定义 Hook [utils/hooks/window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/hooks/window-focus.js)，**只在首页 Index 组件使用**：

```javascript
// pages/index.jsx#L98-L108
const windowFocused = useWindowFocus();
useEffect(() => {
  if (windowFocused) {
    mutateHash();  // 只触发 /api/hash 的重验证，用于检测配置文件变化
  }
}, [windowFocused, mutateHash]);
```

这个 Hook 监听的是 **`window.focus` / `window.blur`**（焦点事件），而 SWR 的 revalidateOnFocus 监听的是 **`visibilitychange` + `focus`**（可见性 + 焦点）。两者是**独立并行**的两套机制：

| 维度 | `useWindowFocus`（自定义） | SWR `revalidateOnFocus`（内置） |
|------|---------------------------|----------------------------------|
| 监听事件 | `window.focus` / `blur` | `visibilitychange` + `window.focus` |
| 触发内容 | 仅 `mutateHash()`（刷新 `/api/hash`） | **所有 useSWR key** 都会被考虑重验证 |
| 节流 | 无 | `focusThrottleInterval: 5000` |
| 影响 Ping/SiteMonitor | ❌ 不影响 | ✅ 每次切回都会触发（5s 内去重） |

> **关键澄清**：`useWindowFocus` 与 Ping/SiteMonitor 的刷新节奏**没有直接关系**。它只负责检测"配置文件 hash 是否变化 → 触发整页 reload"。Ping/SiteMonitor 在聚焦时的额外请求来自 SWR 内置的 `revalidateOnFocus`。

---

## 四、Ping 探测服务端实现

[pages/api/ping.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/api/ping.js)

### 探测流程
```javascript
// 从 YAML 配置中解析 ping 字段
const { ping: pingHostOrURL } = serviceItem;

// 向后兼容：支持 http://hostname 形式，提取 hostname
let hostname = pingHostOrURL;
try {
  hostname = new URL(pingHostOrURL).hostname;
} catch (e) {}

// 调用 npm ping 库（系统级 ICMP echo，非 HTTP）
const response = await ping.probe(hostname);
// 返回: { host, alive, time, ... }
res.status(200).json(response);
```

### 错误分支
- `getServiceItem` 找不到服务 → 400 `{error: "Unable to find service..."}`
- 未配置 ping 字段 → 400 `{error: "No ping host given"}`
- `ping.probe` 抛异常（权限不足、系统不支持 ICMP）→ 400 `{error: "Error attempting ping..."}`

前端 [ping.jsx#L15](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/ping.jsx#L15) 将这些 400 响应当作 `error` 处理，显示红色"ERROR"。

---

## 五、SiteMonitor 探测服务端实现

[pages/api/siteMonitor.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/api/siteMonitor.js)

### 探测流程（两段式重试）
```javascript
let startTime = performance.now();
let [status] = await httpProxy(monitorURL, { method: "HEAD" });
let endTime = performance.now();

// 第一段：HEAD 请求失败（>403 视为异常，如 404/500/连接拒绝）
if (status > 403) {
  // 第二段：降级为 GET 请求重试（有些服务只支持 GET）
  startTime = performance.now();
  [status] = await httpProxy(monitorURL);  // 默认 GET
  endTime = performance.now();
}

return res.status(200).json({
  status,       // 最终 HTTP 状态码
  latency: endTime - startTime,  // 最终请求耗时（ms）
});
```

### 状态码判定规则
前端 [site-monitor.jsx#L22-L47](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/site-monitor.jsx#L22-L47)：
- `status > 403` → 红色 DOWN（显示状态码数字）
- `status ≤ 403` → 绿色 UP（显示延迟，单位 ms）
  - 说明：200/301/302/401/403 都视为"服务可达"

### 注意
- SiteMonitor 的 API **永远返回 200 OK**，真实 HTTP 状态码放在响应体 `status` 字段中
- 前端检查 error 时同时检查 SWR 的 `error`（网络层）和 `data.error`（业务层，如 400 参数错误）

---

## 六、完整刷新节奏决策树

以一次探测请求（Ping 或 SiteMonitor 完全一致）的生命周期为例：

```
组件挂载 (首次渲染)
  │
  ├─ SWR fallback 中有缓存? ──Yes──► 先用 fallback 展示，后台 revalidate（revalidateOnMount=true）
  │                                    (实际是否发请求取决于 dedupingInterval 是否命中)
  │
  └─ No fallback ──► 立即发请求 #1
                       │
                       ▼
              启动 30s refreshInterval 定时器
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     定时器到期?    页面可见性?    浏览器在线?
          │            │            │
        Yes<────►hidden?◄─────►offline?
          │            │            │
          │          Yes/No       Yes/No
          │            │            │
          │            │            ├─ Yes → 暂停定时器 (refreshWhenOffline=false)
          │            │            │
          │            ├─ Yes → 暂停定时器 (refreshWhenHidden=false)
          │            │
          │            └─ No ──┐
          │                    │
          └────────────────────┼──► 发请求
                               │
                       ┌───────┴────────┐
                       ▼                ▼
              5s 内 focused?        2s 内重复 key?
                       │                │
                      Yes               Yes
                       │                │
                 节流不发请求     dedup 只发一次
                       │                │
                       └───────┬────────┘
                               ▼
                      正常发请求 + 延迟展示
```

---

## 七、配置自定义节奏（开发者视角）

如果要调整 Ping/SiteMonitor 的刷新节奏，修改 `components/services/ping.jsx` 或 `site-monitor.jsx` 中的 `useSWR` 第二个参数：

```javascript
// 示例：页面隐藏时也轮询（但离线仍暂停）
useSWR(url, {
  refreshInterval: 30000,
  refreshWhenHidden: true,     // 切到后台标签也每 30s 探测一次
})

// 示例：聚焦时不要额外请求（纯定时轮询）
useSWR(url, {
  refreshInterval: 30000,
  revalidateOnFocus: false,    // 切回标签时不额外触发
})

// 示例：离线也继续发请求（通常没必要，除非用了自定义 fetcher）
useSWR(url, {
  refreshInterval: 30000,
  refreshWhenOffline: true,    // navigator.onLine=false 时也尝试发
})

// 示例：调整 focus 节流窗口
useSWR(url, {
  refreshInterval: 30000,
  focusThrottleInterval: 1000, // 1s 内多次切换只发一次
})
```

这些配置也可以在全局 `SWRConfig` 中统一设置，对所有 useSWR 生效。

---

## 八、常见误解速查表

| 问题 | 答案 | 依据 |
|------|------|------|
| 切到其他标签 2 分钟，再切回来，会发几次请求？ | **1 次**（revalidateOnFocus 触发）+ 正常节奏恢复 | `refreshWhenHidden=false` + `revalidateOnFocus=true` |
| 断网 5 分钟恢复后，Ping 会补发探测吗？ | **不会补发**，只发 1 次重连触发的请求 | `refreshWhenOffline=false` + `revalidateOnReconnect=true` |
| 频繁切标签（每秒一次）会刷屏请求吗？ | **不会**，5 秒节流窗口内只发一次 | `focusThrottleInterval=5000`（默认） |
| `useWindowFocus` 会影响 Ping 刷新吗？ | **不会**，它只触发 `/api/hash` 的 mutate | [index.jsx#L104-L108](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L104-L108) |
| 页面隐藏时 30s 定时器是"继续走但不发"还是"直接暂停"？ | **直接暂停**，不安排下一次 setTimeout | SWR `refreshWhenHidden` 源码实现 |
| Ping 和 SiteMonitor 的刷新节奏完全一样吗？ | **完全一致**，都是 `refreshInterval: 30000` + 全默认 SWR 行为 | [ping.jsx#L6-L8](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/ping.jsx#L6-L8) + [site-monitor.jsx#L6-L8](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/site-monitor.jsx#L6-L8) |
