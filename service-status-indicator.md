# 服务状态指示器代码联动分析

## 一、整体架构联动

### 组件层级关系

```
item.jsx (服务项组件)
├── 根据 service 配置渲染不同的状态指示器
│   ├── service.ping → <Ping />
│   ├── service.siteMonitor → <SiteMonitor />
│   ├── service.container → <Status /> (Docker)
│   ├── service.app → <KubernetesStatus />
│   └── service.proxmoxNode + proxmoxVMID → <ProxmoxStatus />
├── 状态组件内部使用 useSWR 调用对应的 API
├── API 端点采集真实数据返回
├── 状态组件根据返回数据判断状态并映射颜色
└── 根据 style 属性决定显示文字标签还是圆点
```

### 核心入口文件

- **服务项组件**: [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/item.jsx)
- **状态展示样式**: 由 `statusStyle` 配置决定（来自全局 settings 或 service 单独配置）

---

## 二、状态采集逻辑

各类型服务通过独立的 API 端点采集状态数据：

### 2.1 Docker 容器状态采集

- **API 路径**: `/api/docker/status/[container]/[server]`
- **源码**: [docker/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/docker/status/[...service].js)
- **使用库**: `dockerode`
- **采集内容**:
  - 容器状态: `info.State.Status`
  - 健康检查状态: `info.State.Health?.Status`
- **Swarm 支持**: 检查服务副本数，返回 `running n/m` 或 `partial n/m`
- **返回示例**: `{ status: "running", health: "healthy" }`

### 2.2 Ping 网络可达性采集

- **API 路径**: `/api/ping?groupName=&serviceName=`
- **源码**: [ping.js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/ping.js)
- **使用库**: `ping`
- **采集逻辑**: `ping.probe(hostname)`
- **主机解析**: 支持 URL 格式，自动提取 hostname
- **返回示例**: `{ alive: true, time: 23.5, ... }`

### 2.3 HTTP 站点监控采集

- **API 路径**: `/api/siteMonitor?groupName=&serviceName=`
- **源码**: [siteMonitor.js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/siteMonitor.js)
- **使用库**: 内部 `httpProxy`
- **采集逻辑**:
  1. 先发送 HEAD 请求，记录响应时间
  2. 若状态码 > 403，重试 GET 请求
  3. 使用 `performance.now()` 精确计算延迟
- **返回示例**: `{ status: 200, latency: 156.32 }`

### 2.4 Kubernetes 应用状态采集

- **API 路径**: `/api/kubernetes/status/[namespace]/[app]?podSelector=`
- **源码**: [kubernetes/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/kubernetes/status/[...service].js)
- **使用库**: `@kubernetes/client-node`
- **采集逻辑**:
  - 查询指定 namespace 下匹配 label 的 Pods
  - 根据 `pod.status.phase` 判断所有/部分 Pod 是否处于 `Running` 或 `Succeeded` 阶段
  - ⚠️ 注意：判断依据是 Pod Phase 而非 Ready 条件，详见第六章 6.1 节
- **状态映射**（基于 Pod Phase）:
  - 所有 Pod phase ∈ [`Running`, `Succeeded`] → `running`
  - 部分 Pod phase ∈ [`Running`, `Succeeded`] → `partial`
  - 无 Pod 满足 → `down`
  - 无 Pod → `not found`

### 2.5 Proxmox 虚拟机状态采集

- **API 路径**: `/api/proxmox/stats/[node]/[vmid]?type=`
- **源码**: [proxmox/stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/proxmox/stats/[...service].js)
- **调用接口**: `/nodes/{node}/{type}/{vmid}/status/current`
- **支持类型**: `qemu` (默认) 或 `lxc`
- **返回示例**: `{ status: "running", cpu: 0.05, mem: 1073741824 }`

---

## 三、可达性判断逻辑

各前端组件根据 API 返回数据判断服务可达性：

### 3.1 Ping 组件判断逻辑

**源码**: [ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/ping.jsx)

| 条件 | 状态 |
|------|------|
| `error` 存在 | error |
| `!data` | not available |
| `!data.alive` | down |
| `data.alive` | up |

### 3.2 站点监控组件判断逻辑

**源码**: [site-monitor.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/site-monitor.jsx)

| 条件 | 状态 |
|------|------|
| `error || data.error` | error |
| `!data` | not available |
| `data.status > 403` | down |
| 其他情况 | up |

### 3.3 Docker 状态组件判断逻辑

**源码**: [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/status.jsx)

| 条件 | 状态 |
|------|------|
| `error` | error |
| `data.status.includes("running")` + `!data.health` | running |
| `data.status.includes("running")` + `data.health === "healthy"` | healthy |
| `data.status.includes("running")` + `data.health === "starting"` | starting |
| `data.status.includes("running")` + `data.health === "unhealthy"` | unhealthy |
| `data.status === "not found"` | not found |
| `data.status === "exited"` | exited |
| `data.status.startsWith("partial")` | partial n/m |

### 3.4 Kubernetes 状态组件判断逻辑

**源码**: [kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/kubernetes-status.jsx)

| 条件 | 状态 |
|------|------|
| `error` | error |
| `data.status === "running"` | running |
| `data.status === "not found"` | not found |
| `data.status === "down"` | down |
| `data.status === "partial"` | partial |

### 3.5 Proxmox 状态组件判断逻辑

**源码**: [proxmox-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/proxmox-status.jsx)

| 条件 | 状态 |
|------|------|
| `error` | error |
| `data.status === "running"` | running |
| `data.status === "stopped"` | stopped (显示 exited) |
| `data.status === "paused"` | paused |
| `data.status === "offline"` | offline |
| `data.status === "not found"` | not found |

---

## 四、颜色展示规则

所有状态指示器组件使用统一的颜色映射策略，通过 Tailwind CSS 类实现：

### 4.1 颜色映射表

| 状态类型 | 文字颜色类 | 圆点背景色类 | 视觉效果 |
|---------|-----------|-------------|---------|
| 默认/未知/加载中 | `text-black/20 dark:text-white/40` | `bg-black dark:bg-white` | 灰色 |
| 正常/运行中/健康/UP | `text-emerald-500/80` | `bg-emerald-500` | 绿色 |
| 启动中 | `text-blue-500/80` | `bg-blue-500` | 蓝色 |
| 异常/不健康/停止/离线/partial | `text-orange-400/50 dark:text-orange-400/80` | `bg-orange-400` | 橙色 |
| 错误/DOWN/HTTP > 403 | `text-rose-500/80` 或 `text-rose-500` | `bg-rose-500` | 红色 |

### 4.2 展示样式切换

组件支持两种展示样式，由 `style` 属性控制：

- **`basic`/默认**: 显示文字标签，使用 `text-*` 颜色类
- **`dot`**: 显示彩色圆点，自动将 `text-*` 替换为 `bg-*`，并移除透明度后缀

**颜色转换代码**（各组件通用）:
```javascript
if (style === "dot") {
  colorClass = colorClass.replace(/text-/g, "bg-").replace(/\/\d\d/g, "");
}
```

### 4.3 状态标签 CSS 类

每个状态指示器还会生成一个基于状态的 CSS 类，便于自定义样式：
- Docker: `docker-status-${statusLabel}`
- Proxmox: `proxmoxstatus-${statusLabel}`
- Kubernetes: `k8s-status`
- Ping: `ping-status`
- Site Monitor: `site-monitor-status`

---

## 五、刷新时机

数据刷新使用 **SWR (stale-while-revalidate)** 库管理。

### 5.1 SWR 全局配置

**源码**: [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/_app.jsx#L75-L79)

```javascript
<SWRConfig
  value={{
    fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
  }}
>
```

- 仅配置了全局 `fetcher`，无全局 `refreshInterval`
- 各组件独立配置刷新策略

### 5.2 各组件刷新配置

| 组件 | refreshInterval | 源码位置 |
|------|-----------------|---------|
| Ping | 30000ms (30秒) | [ping.jsx#L7](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/ping.jsx#L7-L8) |
| Site Monitor | 30000ms (30秒) | [site-monitor.jsx#L7](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/site-monitor.jsx#L7-L8) |
| Docker Status | 未设置 | - |
| Kubernetes Status | 未设置 | - |
| Proxmox Status | 未设置 | - |

### 5.3 SWR 触发刷新的时机

1. **组件挂载**: 组件首次渲染时自动请求一次
2. **周期性刷新**: 配置了 `refreshInterval` 的组件会周期性刷新
3. **窗口聚焦**: 窗口重新获得焦点时重新验证数据（SWR 默认行为）
4. **网络恢复**: 网络重新连接时重新验证数据（SWR 默认行为）
5. **手动触发**: 可通过 `mutate()` 函数手动刷新

### 5.4 useWidgetAPI 封装

**源码**: [use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/utils/proxy/use-widget-api.js)

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  // ...
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

### 5.5 配置传递链路

配置中的 `refreshInterval` 可以通过以下方式设置：

1. 在 service 的 widget 配置中直接设置 `refreshInterval`
2. 在 [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/utils/config/service-helpers.js) 中解析配置时传递
3. 支持的 widget 类型: `glances`, `customapi`, `iframe`, `prometheusmetric`

---

## 六、代码理解易误判点澄清

### 6.1 Kubernetes 状态判断：基于 Pod Phase 而非 Ready 条件

**容易误判之处**: 变量名 `someReady` / `allReady` 暗示检查的是 Pod 的 Ready 条件，但实际检查的是 **Pod Phase**。

**API 端点核心代码**（[kubernetes/status/[...service].js#L53-L60](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/kubernetes/status/[...service].js#L53-L60)）:

```javascript
const someReady = pods.find((pod) => ["Succeeded", "Running"].includes(pod.status.phase));
const allReady = pods.every((pod) => ["Succeeded", "Running"].includes(pod.status.phase));
```

**实际判断依据是 `pod.status.phase`**，而非 `pod.status.conditions` 中的 Ready 条件。

**两者的关键区别**:

| 判断维度 | Pod Phase | Ready 条件 |
|---------|-----------|-----------|
| 字段路径 | `pod.status.phase` | `pod.status.conditions[type="Ready"].status` |
| Running 含义 | 容器已创建且至少一个正在运行 | 容器正在运行 **且** 通过了就绪探针和启动探针 |
| 典型误判场景 | Pod phase=Running 但 Ready=false（正在启动、健康检查未通过） | 不会出现此误判 |
| Succeeded 含义 | 容器成功执行完毕（CronJob 等一次性任务） | N/A（Succeeded Pod 通常 Ready=false） |

**影响**: 一个 Pod 可能 `phase=Running`（状态指示器显示绿色 running）但 `Ready=false`（实际尚未就绪、无法接收流量）。这意味着 Kubernetes 状态指示器在 Pod 启动过程中可能过早地显示"正常"。

**前端组件额外说明**（[kubernetes-status.jsx#L18-L21](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/kubernetes-status.jsx#L18-L21)）:

```javascript
if (data.status === "running") {
  statusTitle = data.health ?? data.status;
  statusLabel = statusTitle;
  colorClass = "text-emerald-500/80";
}
```

前端组件在 `data.status === "running"` 时会尝试读取 `data.health`，但当前后端 API **并未返回 `health` 字段**，因此 `data.health` 始终为 `undefined`，`statusTitle` 退化为显示原始的 `"running"` 字符串。这不像 Docker 状态组件那样有 healthy/starting/unhealthy 的细分。

---

### 6.2 状态指示器刷新与普通服务小组件刷新的区别

在 [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/components/services/item.jsx) 中，每个服务卡片同时包含两类组件，它们的刷新机制**完全独立**：

#### 两类组件对比

| 维度 | 状态指示器组件 | 展开面板 Widget 组件 |
|------|--------------|-------------------|
| **所在位置** | 卡片右上角（始终可见） | 卡片下方（需点击展开） |
| **Docker** | `<Status />` | `<Docker />` |
| **Kubernetes** | `<KubernetesStatus />` | `<Kubernetes />` |
| **Proxmox** | `<ProxmoxStatus />` | `<ProxmoxVM />` |
| **Ping / SiteMonitor** | `<Ping />` / `<SiteMonitor />` | 无对应 widget |
| **useSWR 调用方式** | 直接调用 `useSWR(url)` | 直接调用 `useSWR(url)` |
| **refreshInterval** | Ping/SiteMonitor: 30s; 其他: 无 | Docker/K8s/Proxmox: 无 |

#### SWR 缓存共享与挂载时的行为

状态指示器与展开面板 widget 对同一服务使用**完全相同的 SWR URL key**：

| 类型 | 状态指示器 URL | Widget URL | 是否共享缓存 |
|------|--------------|-----------|------------|
| Docker | `/api/docker/status/{container}/{server}` | `/api/docker/status/{container}/{server}` | ✅ 共享 |
| Kubernetes | `/api/kubernetes/status/{ns}/{app}?podSelector=` | `/api/kubernetes/status/{ns}/{app}?podSelector=` | ✅ 共享 |
| Proxmox | `/api/proxmox/stats/{node}/{vmid}?type=` | `/api/proxmox/stats/{node}/{vmid}?type=` | ✅ 共享 |

SWR 以 URL 为缓存 key，相同 URL 的请求会共享同一条缓存数据。但理解以下行为至关重要：

**SWR 默认配置（[\_app.jsx#L75-L79](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/_app.jsx#L75-L79)）**：

```javascript
<SWRConfig
  value={{
    fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
  }}
>
```

项目**未覆盖** SWR 的默认行为，因此：
- `revalidateOnMount` 默认为 `true`
- `dedupingInterval` 默认为 `2000` 毫秒
- `revalidateOnFocus` 默认为 `true`

**挂载时的精确行为**：

1. **首次打开页面，状态指示器先挂载**
   - SWR 缓存中无数据，立即发起网络请求获取状态
   - 请求返回后数据写入缓存

2. **用户点击展开面板，widget 首次挂载**
   - widget 的 `useSWR` 使用完全相同的 URL key
   - **先立即返回缓存中的旧数据**（SWR 的 stale-while-revalidate 机制）
   - **然后再发起一次新的网络请求**进行重新验证（因为 `revalidateOnMount=true`）
   - ⚠️ **并不是"不会重复请求"**，而是先展示缓存 + 后台重新请求
   - 新请求返回后，状态指示器和 widget **都会同步更新**（共享缓存）

3. **展开/折叠重复操作**
   - 每次展开都是 widget 重新挂载，都会触发一次重新验证请求
   - 如果两次展开间隔小于 `dedupingInterval`（2秒），请求会被去重合并

4. **窗口聚焦时**
   - 状态指示器始终存在（始终可见），会触发重新验证
   - 重新验证的结果写入共享缓存，widget 下次展开时直接使用最新数据

#### 展开面板的共用请求与额外资源请求差异

各类型展开面板 widget 发起的请求数量不同，与状态指示器的关系也不同：

**Docker 展开面板**（[widgets/docker/component.jsx#L13-L17](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/widgets/docker/component.jsx#L13-L17)）

```javascript
// ① 共用请求（与 Status 组件共享缓存）
const { data: statusData } = useSWR(`/api/docker/status/${widget.container}/${widget.server || ""}`);
// ② 额外资源请求（仅展开面板有，状态指示器无）
const { data: statsData } = useSWR(`/api/docker/stats/${widget.container}/${widget.server || ""}`);
```

**Kubernetes 展开面板**（[widgets/kubernetes/component.jsx#L11-L17](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/widgets/kubernetes/component.jsx#L11-L17)）

```javascript
// ① 共用请求（与 KubernetesStatus 组件共享缓存）
const { data: statusData } = useSWR(`/api/kubernetes/status/${widget.namespace}/${widget.app}?${podSelectorString}`);
// ② 额外资源请求（仅展开面板有，状态指示器无）
const { data: statsData } = useSWR(`/api/kubernetes/stats/${widget.namespace}/${widget.app}?${podSelectorString}`);
```

**Proxmox 展开面板**（[widgets/proxmoxvm/component.jsx#L11](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/widgets/proxmoxvm/component.jsx#L11)）

```javascript
// 唯一请求（与 ProxmoxStatus 组件共享缓存，同时提供 stats 数据）
const { data, error } = useSWR(`/api/proxmox/stats/${widget.node}/${widget.vmid}?type=${widget.type || "qemu"}`);
```

**三者差异对比**：

| 维度 | Docker | Kubernetes | Proxmox |
|------|--------|-----------|---------|
| **共用请求数量** | 1 个（status） | 1 个（status） | 1 个（stats 兼 status） |
| **额外请求数量** | 1 个（stats） | 1 个（stats） | 0 个 |
| **展开面板总请求数** | 2 个 | 2 个 | 1 个（复用 status 请求） |
| **额外请求 API 端点** | `/api/docker/stats/...` | `/api/kubernetes/stats/...` | 无 |
| **额外请求采集内容** | CPU / 内存 / 网络流量 | CPU / 内存 / CPU限制 / 内存限制 | 无（CPU/内存已在共用请求中返回） |
| **状态指示器使用字段** | `data.status`, `data.health` | `data.status` | `data.status` |
| **Widget 额外使用字段** | `statsData.stats.memory_stats`, `statsData.stats.networks`, `statsData.stats.cpu_stats` | `statsData.stats.mem`, `statsData.stats.cpu`, `statsData.stats.cpuLimit` | `data.cpu`, `data.mem` |

**Proxmox 的特殊架构**：

Proxmox 没有独立的 status API 和 stats API。状态指示器和展开面板 widget 共用**同一个** `/api/proxmox/stats/...` 端点（[proxmox/stats/[...service].js#L73-L77](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/pages/api/proxmox/stats/[...service].js#L73-L77)）：

```javascript
return res.status(200).json({
  status: parsedData.data.status || "unknown",
  cpu: parsedData.data.cpu,
  mem: parsedData.data.mem,
});
```

这个端点同时返回 `status`（状态指示器用）、`cpu` 和 `mem`（展开面板用）。因此：
- Proxmox 状态指示器请求的数据量比 Docker/K8s 状态指示器大（额外包含 cpu/mem）
- Proxmox 展开面板不会比状态指示器产生额外的 API 请求

**展开时的资源开销总结**：

假设首次加载页面时状态指示器已完成请求，用户点击展开卡片：

| 类型 | 共用请求行为 | 额外请求行为 | 展开时新增网络请求数 |
|------|------------|------------|------------------|
| Docker | SWR 重新验证（复用缓存 + 后台刷新） | 首次请求 `/api/docker/stats/...` | 2 个（status 重验证 + stats 新请求） |
| Kubernetes | SWR 重新验证（复用缓存 + 后台刷新） | 首次请求 `/api/kubernetes/stats/...` | 2 个（status 重验证 + stats 新请求） |
| Proxmox | SWR 重新验证（复用缓存 + 后台刷新） | 无额外请求 | 1 个（唯一 URL 的重验证） |

如果用户先展开再折叠，然后 2 秒内再次展开：

| 类型 | 再次展开时的网络请求数 | 原因 |
|------|---------------------|------|
| Docker | 0 个 | 两次挂载间隔 < `dedupingInterval`（2s），请求被去重 |
| Kubernetes | 0 个 | 同上 |
| Proxmox | 0 个 | 同上 |

但两者都没有设置 `refreshInterval`，因此**日常运行中不会主动刷新**（仅依赖挂载重验证、窗口聚焦、网络恢复）

#### useWidgetAPI 与直接 useSWR 的区别

普通 widget 组件（如 glances、customapi 等）通过 [useWidgetAPI](file:///d:/fz/0601/solo-dogfeeding/code/205-homepage/src/utils/proxy/use-widget-api.js) 封装调用，支持通过配置传入 `refreshInterval`：

```javascript
// useWidgetAPI 支持从 widget 配置读取 refreshInterval
if (options && options[1]?.refreshInterval) {
  config.refreshInterval = options[1].refreshInterval;
}
```

而**所有状态指示器组件都直接使用 `useSWR`**，不经过 `useWidgetAPI` 封装，因此：
- 状态指示器的 `refreshInterval` 是**硬编码**的（Ping/SiteMonitor 各 30s）
- 其他状态指示器（Docker/K8s/Proxmox）**无法通过配置设置刷新间隔**
- Docker/K8s/Proxmox widget 同样直接使用 `useSWR`，也不支持配置 `refreshInterval`

---

### 6.3 不同指示器缺乏统一周期刷新的影响

#### 现状总结

| 指示器类型 | refreshInterval | 刷新触发方式 | 数据时效性 |
|-----------|----------------|------------|-----------|
| Ping | 30s | 自动周期刷新 + 窗口聚焦 | ✅ 较好 |
| Site Monitor | 30s | 自动周期刷新 + 窗口聚焦 | ✅ 较好 |
| Docker Status | 无 | 仅窗口聚焦/网络恢复/首次挂载 | ⚠️ 依赖用户行为 |
| Kubernetes Status | 无 | 仅窗口聚焦/网络恢复/首次挂载 | ⚠️ 依赖用户行为 |
| Proxmox Status | 无 | 仅窗口聚焦/网络恢复/首次挂载 | ⚠️ 依赖用户行为 |

#### 具体影响

**1. Docker/K8s/Proxmox 状态可能长时间不更新**

用户打开页面后，如果不切换标签页再切回来（触发窗口聚焦重验证），Docker/K8s/Proxmox 的状态指示器将一直显示首次挂载时的数据。一个容器从 running 变为 exited 后，指示器可能仍显示绿色。

**2. 同一页面不同类型指示器状态更新频率不一致**

如果服务同时配置了 `ping` 和 `container`，Ping 指示器每 30 秒自动刷新，而 Docker 状态指示器不会。用户可能看到 Ping 显示 up（绿色）但 Docker 状态指示器仍显示旧状态，造成混淆。

**3. SWR 缓存共享带来的"虚假及时性"**

虽然 Docker 状态指示器自身没有 `refreshInterval`，但它与 Docker widget（展开面板）共享 SWR 缓存。如果用户点击展开面板触发了 Docker widget 的渲染，widget 也会发起相同的 `useSWR` 请求。但由于两者都没有 `refreshInterval`，展开操作只会让 SWR 执行一次 stale-while-revalidate，不会建立持续刷新。

**4. Ping/SiteMonitor 始终占用网络资源**

Ping 和 Site Monitor 设置了 30 秒的 `refreshInterval`，即使用户长时间不操作页面，这两个组件也会持续发送请求。对于配置了大量 ping/siteMonitor 服务的页面，这会带来持续的 API 调用开销。

**5. 窗口聚焦重验证的非确定性**

Docker/K8s/Proxmox 的刷新依赖 SWR 的 `revalidateOnFocus` 默认行为。但用户切换标签页再切回的时机不可预测，无法保证状态数据在任何确定的时间窗口内更新。如果用户长时间在同一标签页操作（如编辑配置），状态指示器可能永远不刷新。

---

## 七、完整调用链路示例

以 Ping 状态指示器为例：

```
1. item.jsx 检测到 service.ping 配置
   ↓
2. 渲染 <Ping groupName="..." serviceName="..." style="dot" />
   ↓
3. Ping 组件调用 useSWR('/api/ping?groupName=...&serviceName=...', { refreshInterval: 30000 })
   ↓
4. SWR 触发 fetcher，请求 /api/ping 端点
   ↓
5. API 端点调用 getServiceItem() 获取服务配置中的 ping 地址
   ↓
6. 使用 ping.probe(hostname) 检测网络可达性
   ↓
7. 返回 { alive: true, time: 23.5 }
   ↓
8. Ping 组件根据 data.alive 判断状态为 up
   ↓
9. 映射颜色为 text-emerald-500/80
   ↓
10. style 为 dot，转换为 bg-emerald-500
   ↓
11. 渲染绿色圆点
   ↓
12. 30秒后，SWR 自动触发下一次刷新
```
