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
  - 判断所有/部分 Pod 是否处于 Running 或 Succeeded 状态
- **状态映射**:
  - 所有 Ready → `running`
  - 部分 Ready → `partial`
  - 无 Ready → `down`
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

## 六、完整调用链路示例

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
