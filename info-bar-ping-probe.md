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

```
时间轴 (秒)
0         30        60        90       …
│         │         │         │
├─ SWR 首次请求 ─┤         │         │
│  (组件挂载时)   │         │         │
│         ├─ SWR 自动刷新 ─┤         │
│         │         ├─ SWR 自动刷新 ─┤
│         │         │         │
▼         ▼         ▼         ▼
每个 <Ping> / <SiteMonitor> 组件独立维护自己的 SWR 请求
```

关键点：

1. **每组件独立轮询**：每个 `<Ping>` 和 `<SiteMonitor>` 组件各自维护一个 SWR 请求实例，互不干扰。
2. **固定 30 秒刷新**：`refreshInterval: 30000` 是硬编码的，不可通过配置修改（与 widget 的 `refreshInterval` 不同）。
3. **首次加载**：组件挂载时 SWR 立即发起请求，同时显示加载态（灰色 `Ping` / `Response` 文字）。
4. **无去抖/节流**：SWR 的 `refreshInterval` 是精确的定时器，不会因窗口焦点变化而停止（SWR 默认在窗口重新获得焦点时也会触发 revalidate，但 `refreshInterval` 是独立定时器）。
5. **缓存键**：SWR 的缓存键就是完整 URL（如 `/api/ping?groupName=g&serviceName=s`），同一服务不会重复请求。

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
| [ping.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/ping.test.jsx) | Ping 组件单元测试 |
| [site-monitor.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/components/services/site-monitor.test.jsx) | SiteMonitor 组件单元测试 |
| [ping.test.js](file:///d:/fz/0601/solo-dogfeeding/code/210-homepage/src/__tests__/pages/api/ping.test.js) | Ping API 单元测试 |
