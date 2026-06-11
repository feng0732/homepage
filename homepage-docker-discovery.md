# Homepage 服务发现与 Docker 集成代码分析

本文档深度剖析 Homepage 项目中，服务探测 → 数据整理 → 页面展示的完整数据链路。

---

## 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     服务探测层 (Discovery)                        │
│  - Docker 容器标签扫描 (servicesFromDocker)             │
│  - Kubernetes Ingress/Traefik 扫描                               │
│  - services.yaml 静态配置                                    │
└────────────────────────────┬───────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     数据整理层 (Aggregation)                 │
│  - servicesResponse() 合并多源服务                              │
│  - cleanServiceGroups() 数据标准化                          │
│  - 按 group 分组 / 排序 / 去重                               │
└────────────────────────────┬───────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     页面展示层 (Presentation)                      │
│  - getStaticProps 预取数据 (SSR)                           │
│  - ServicesGroup → List → Item 组件树                        │
│  - 实时状态 API: /api/docker/status & /stats               │
│  - Docker 组件: CPU / Mem / 网络统计                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 一、服务探测层（Service Discovery）

### 1.1 Docker 连接配置

**核心文件：[docker.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js)

`getDockerArguments()` 函数负责从 `docker.yaml` 解析 Docker 服务器连接参数。

**默认连接方式：
- **Linux/Mac：默认 socketPath: `/var/run/docker.sock`
- **Windows**：默认 host: `127.0.0.1`
- **自定义配置**：支持 socket / host + TLS / headers

```javascript
// 核心配置示例（docker.yaml）：
my-docker:
  socket: /var/run/docker.sock
  swarm: false          # 是否 Docker Swarm 模式
```

**关键代码逻辑：**
- [getDockerArguments](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js#L16-L64)：解析 docker.yaml 配置
- 支持 socketPath、host:port、TLS 证书认证
- 支持 Swarm 模式标记（影响后续探测方式

---

### 1.2 Docker 容器标签发现机制

**核心文件：[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js)**

**核心函数：[servicesFromDocker()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L63-L170)

#### 工作流程：

```
Step 1: 读取 docker.yaml → 获取所有 Docker 服务器配置
   │
   ▼
Step 2: 遍历每个服务器，调用 dockerode
   │  - 普通模式：docker.listContainers({ all: true })
   │  - Swarm 模式：docker.listServices({ all: true })
   │
   ▼
Step 3: 扫描每个容器/服务的 Labels
   │  寻找以 "homepage." 前缀的 Label
   │
   ▼
Step 4: 将 Label 键值对 → 构造服务对象
   │  homepage.group=Monitoring
   │  homepage.name=Grafana
   │  homepage.href=https://grafana.example.com
   │  homepage.icon=grafana.png
   │  homepage.container=grafana
   │  homepage.server=my-docker
   │  homepage.description=Metrics Dashboard
   │
   ▼
Step 5: 按 group 分组 → 返回 mappedServiceGroups
```

#### Label 解析细节：

- **instance 过滤机制**
  - Label: `homepage.instance.prod.name=MySvc` → 如果 `instanceName=prod` 的实例才会包含此服务
  - Label: `homepage.name=MySvc` → 所有实例通用
- **shvl 库**：用于将点号路径转化为嵌套对象
  - 如 `homepage.widgets[0].type=docker` → `{ widgets: [{ type: "docker" }]`

#### 必选 Label（缺失会报错）：
- `homepage.name` — 服务名称
- `homepage.group` — 所属分组

自动注入的字段：
```javascript
constructedService = {
  container: containerName,      // 容器名（去掉前缀 "/"）
  server: serverName,      // 服务器名（来自 docker.yaml）
  weight: 0,
  type: "service"
}
```

---

### 1.3 多源服务发现

除了 Docker 外，系统还支持：

| 来源 | 函数 | 文件 |
|------|------|------|
| **静态配置** | [servicesFromConfig()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L53-L61) | service-helpers.js |
| **Kubernetes** | [servicesFromKubernetes()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L172-L229) | service-helpers.js |

---

## 二、数据整理层（Data Aggregation）

### 2.1 服务合并主入口

**核心文件：[api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js)**

**核心函数：[servicesResponse()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L158-L256)

**合并流程详解：

```javascript
// 三大来源并行加载：
1. discoveredDockerServices    ← servicesFromDocker()
2. discoveredKubernetesServices  ← servicesFromKubernetes()
3. configuredServices        ← servicesFromConfig()
```

**Step 1: 获取所有唯一 groupName 名的并集：
```javascript
const mergedGroupsNames = [...new Set([
  ...dockerGroups.map(g => g.name),
  ...k8sGroups.map(g => g.name),
  ...configGroups.map(g => g.name)
])
```

**Step 2: 按 group 合并服务：
```javascript
mergedGroup = {
  name: groupName,
  services: [
    ...dockerGroup.services,
    ...k8sGroup.services,
    ...configGroup.services
  ].sort(compareServices)  // 先按 weight，再按 name 排序
}
```

**Step 3: 应用布局排序**（settings.yaml 中定义的 layout 顺序，未在 layout 中的 group 放到 unsortedGroups

**Step 4: 剪枝空分组 `pruneEmptyGroups() — 移除没有 services 且没有 subgroups 的 group

---

### 2.2 数据标准化

**核心函数：[cleanServiceGroups()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L231-L708)

**处理内容：**

1. **类型转换**：
   - weight: string → number（如 "100" → 100
   - showStats: string → boolean（如 "true" → true

2. **Widget 白名单过滤**（安全考虑，只向前端传递明确字段：
```javascript
// 仅保留 widgets 中的这些键会被传到前端
const { fields, hideErrors, highlight, type, container, server, ... } = widgetData;
```

3. **单一 widget → widgets 数组归一化**：
```javascript
if (service.widget) {
  service.widgets.push(service.widget);
  delete service.widget;
}
```

4. **按 widget type 特化处理**：
- type="docker" → 注入 server/container 字段
- type="kubernetes" → 注入 namespace/app/podSelector
- 等等（共 40+ 种 widget 的特化处理

---

## 三、页面展示层（Presentation）

### 3.1 服务端预取（SSR + SWR 缓存

**核心文件：[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx)

**getStaticProps()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L55-L95) — 在构建时预取所有服务数据：

```javascript
export async function getStaticProps() {
  const services = await servicesResponse();  // 服务端执行，结果存入 fallback
  return {
    props: {
      fallback: {
        "/api/services": services,
        // ...
      }
    }
  };
}
```

**客户端通过 SWR 消费：
```javascript
const { data: services } = useSWR("/api/services");  // 优先使用 fallback，避免重新请求
```

---

### 3.2 组件层级结构

```
index.jsx
  └── ServicesGroup (group.jsx)      ← 可折叠分组
  │   ├── 标题栏（图标 + group 名称 + 折叠箭头
  │   └── List (list.jsx)      ← 网格/列布局
  │       └── Item (item.jsx)  ← 单个服务卡片
  │           ├── ResolvedIcon  ← 服务图标
  │           ├── 服务名称 & 描述
  │           ├── 状态标签区
  │           │   ├── Ping 状态
  │           │   ├── SiteMonitor
  │           │   ├── Status (Docker 状态徽章
  │           │   └── KubernetesStatus
  │           │   └── ProxmoxStatus
  │           └── 展开的 Stats 区域
  │               └── Docker widget (docker/component.jsx)
  │                   ├── /api/docker/status/[container]/[server]
  │                   └── /api/docker/stats/[container]/[server]
  │                       └── Block × 4 (CPU / Mem / RX / TX)
  └── Widgets (通用)
```

---

### 3.3 Docker 状态徽章（Status Badge

**核心文件：[status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx)**

通过 useSWR 轮询 `/api/docker/status/${service.container}/${service.server}

**状态映射表：**

| 返回状态 | 展示文本 | 颜色 |
|--------|-------|-----|
| running + healthy | healthy | 绿色 |
| running + starting | starting | 蓝色 |
| running + unhealthy | unhealthy | 橙色 |
| running (无 health) | running | 绿色 |
| partial x/y | partial x/y | 橙色 |
| exited | exited | 橙色 |
| not found | not found | 橙色 |
| error | error | 红色 |

**显示样式：
- **label 模式（文字标签
- **dot 模式（圆点指示器

---

### 3.4 Docker 统计组件

**核心文件：[component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx)

**数据来源（SWR 并行请求两个 API：
```javascript
const { data: statusData } = useSWR(`/api/docker/status/${widget.container}/${widget.server || ""}`);
const { data: statsData } = useSWR(`/api/docker/stats/${widget.container}/${widget.server || ""}`);
```

**数据计算辅助函数：[stats-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js)**

| 指标 | 算法 |
|-----|-----|
| **CPU 使用率 | `(cpuDelta / systemDelta * online_cpus * 100` |
| **内存使用** | `usage - total_inactive_file（参考 Docker CLI 算法 |
| **网络 RX/TX | 遍历所有网卡累加 |

---

### 3.5 API 端点详解

#### 3.5.1 状态 API

**文件：[status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/status/[...service].js)

**路由参数：`[containerName, containerServer] = req.query.service`

**执行流程：**

```
1. getDockerArguments(server) → 建立 Docker 连接
2. docker.listContainers({ all: true }) → 列出所有容器
3. 查找 containerName
   │
   ├─ 找到 → container.inspect() → 返回 { status, health }
   │
   └─ 没找到 && Swarm 模式
       ├─ docker.getService(name).inspect()
       ├─ docker.listTasks({ filters: { service: [name] } })
       └─ Replicated 模式：tasks.length / replicas
       │   → "running 2/2" 或 "partial 1/2"
       └─ Global 模式：找本地容器 → inspect
```

#### 3.5.2 统计 API

**文件：[stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/stats/[...service].js)**

流程类似，核心调用：
```javascript
const stats = await container.stats({ stream: false });  // 一次性获取而非流式
return res.status(200).json({ stats });
```

---

## 四、完整数据流时序图

```
┌──────────┐     getStaticProps()
│  Next.js │───────┐
│  Server  │       │
└──────────┘       ▼
           servicesResponse()
           ┌─────────────────────────────────┐
           │ 1. servicesFromDocker()    │
           │    - 扫描 Docker Labels │
           │ 2. servicesFromK8s()      │
           │ 3. servicesFromConfig()  │
           │ 4. 合并 / 排序 / 清理  │
           └────────────┬────────────┘
                        │ 注入 SWR fallback
                        ▼
┌────────────────────────────────────┐
│       浏览器 (Client)            │
│  useSWR("/api/services")  │
└────────────┬───────────────┘
             │ 渲染 ServicesGroup
             ▼
     ┌───────────────────────┐
     │  Item (服务卡片    │
     │  - 显示 Status 徽章 │
     └────────┬──────────────┘
              │ 用户点击展开
              ▼
    ┌─────────────────────────────┐
    │  Docker Component       │
    │  useSWR(/status)    │
    │  useSWR(/stats)     │
    │  - 计算 CPU/Mem/RX/TX │
    └─────────────────────────────┘
              │
              ▼
    ┌──────────────────────────────┐
    │  /api/docker/status/.. │
    │  /api/docker/stats/..  │
    │    ┌──────────────┐
    │    │ dockerode    │
    │    │ Docker API    │
    │    └──────────────┘
    └──────────────────────────────┘
```

---

## 五、关键设计要点

### 5.1 标签驱动的配置
- 所有服务发现通过容器 Labels：`homepage.` 前缀
- 多层 fallback：config > instance > 默认
- 多实例隔离：`instance.<name>.

### 5.2 安全白名单机制
- `cleanServiceGroups()` 中的 widgets 字段白名单
- 仅向前端暴露必要字段，过滤敏感配置

### 5.3 容错设计
- 单个 Docker 服务器失败 → 不影响其他
- 配置缺失 → 优雅降级（error 处理
- 容器不存在 → 404 而非 crash

### 5.4 性能优化
- SSR 预取 + SWR 缓存
- SWR fallback 首屏无闪烁
- 按需加载（点击展开才请求 stats API
- 配置 hash 检测变更自动刷新

### 5.5 Swarm 支持
- 区分普通容器 vs Swarm Service 两种模式
- Replicated 模式显示副本数
- Global 模式优先本地容器

---

## 六、代码关联速查表

| 功能模块 | 关键文件 | 关键函数/组件 |
|--------|---------|-------------|
| Docker 连接配置 | [docker.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js) | `getDockerArguments()` |
| Docker 服务发现 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromDocker()` |
| 服务合并整理 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `servicesResponse()` |
| 数据标准化 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `cleanServiceGroups()` |
| 页面入口 | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx) | `getStaticProps()` |
| 服务分组组件 | [group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx) | `ServicesGroup` |
| 服务列表 | [list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/list.jsx) | `List` |
| 单个服务卡片 | [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/item.jsx) | `Item` |
| 状态徽章 | [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx) | `Status` |
| Docker 统计组件 | [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx) | `Component` (docker) |
| 统计计算 | [stats-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js) | `calculateCPUPercent` 等 |
| 状态 API | [...service].js (status)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/status/[...service].js) | `handler` |
| 统计 API | [...service].js (stats)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/stats/[...service].js) | `handler` |
| 配置骨架 | [docker.yaml](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/skeleton/docker.yaml) | 配置示例 |
