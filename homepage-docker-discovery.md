# Homepage 服务发现与 Docker 集成代码分析

本文档基于源代码事实，深度剖析 Homepage 项目中「服务探测 → 数据整理 → 页面展示」的完整数据链路。
重点聚焦：**布局分组空组展示的精确边界** 和 **Kubernetes 接口错误从后端到前端组件的完整传播路径**。

---

## 核心发现摘要（本次最新修正）

1. **SWR Fetcher 不检查 `res.ok`**：HTTP 4xx/5xx 的错误信息在 `data.error` 中，而非 `error` 对象中
2. **K8s Component 缺失 `data.error` 检测**：API 返回 500 `{ error: "..." }` 时，不进 error 分支，而是显示 "offline" 或抛出 TypeError
3. **K8s Status 徽章缺失 `data.error` 检测**：API 返回 500 时，显示灰色 "unknown"，而非红色 "error"
4. **K8s Stats API 返回 `{ error: "..." }` 时，`statsData.stats` 为 undefined**，访问 `statsData.stats.cpuLimit` 会抛 TypeError（潜在 Bug）
5. **Error 组件的 `error.data.error` 检查永远不命中**：fetcher 不构造这种结构
6. **空组展示由 `pruneEmptyGroups()` 最终决定**：layout 定义的空组如果没有子组或服务，会被剪掉

---

## 整体架构三层模型

```
┌──────────────────────────────────────────────────────────────────────┐
│  第一层：服务探测层 (Discovery)                                    │
│                                                                    │
│  来源一：Docker 容器/服务标签扫描                                   │
│    - 入口：servicesFromDocker()                                   │
│    - 前缀：homepage.*                                              │
│    - 对象：容器 (listContainers) / Swarm 服务 (listServices)      │
│                                                                    │
│  来源二：Kubernetes 资源注解扫描                                   │
│    - 入口：servicesFromKubernetes()                               │
│    - 前缀：gethomepage.dev/*                                      │
│    - 对象：Ingress / Traefik IngressRoute / HTTPRoute             │
│    - 前置条件：gethomepage.dev/enabled = "true"                   │
│                                                                    │
│  来源三：静态配置文件                                              │
│    - 入口：servicesFromConfig()                                   │
│    - 文件：services.yaml                                           │
└────────────────────────────┬──────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  第二层：数据整理层 (Aggregation)                                  │
│                                                                    │
│  入口：servicesResponse()                                          │
│    1. 三来源分别 cleanServiceGroups() 标准化                      │
│    2. 按 group 名取并集 → 合并 services 数组                       │
│    3. layout 定义但无服务的空 group 也被合并进来                    │
│    4. 按 weight + name 排序                                        │
│    5. pruneEmptyGroups() 递归剪枝空分组                            │
│                                                                    │
│  ★ 空分组是否展示取决于剪枝后的状态（见第二章详述）                │
│                                                                    │
│  输出结构：Group[]                                                 │
│    { name, services: Service[], groups: Group[] }                 │
└────────────────────────────┬──────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  第三层：页面展示层 (Presentation)                                 │
│                                                                    │
│  首屏数据：getStaticProps() (SSR)                                  │
│    → servicesResponse()                                            │
│    → 注入 SWR fallback                                             │
│    → 无额外 HTTP 请求，首屏直出                                     │
│                                                                    │
│  客户端数据：useSWR("/api/services")                               │
│    → 优先使用 fallback                                              │
│    → 后台可重新验证 (revalidate)                                   │
│                                                                    │
│  实时状态（按需加载）：                                              │
│    Docker Status:   /api/docker/status/[container]/[server]      │
│    Docker Stats:    /api/docker/stats/[container]/[server]       │
│    K8s Status:      /api/kubernetes/status/[ns]/[app]             │
│    K8s Stats:       /api/kubernetes/stats/[ns]/[app]              │
│                                                                    │
│  ★ K8s 接口错误传播路径已完全拆解（见第三章详述）                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 一、服务探测层（Service Discovery）

### 1.1 Docker 连接配置

**核心文件：** [docker.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js)

**核心函数：** [getDockerArguments(server)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js#L16-L64)

#### 默认连接参数

由 [getDefaultDockerArgs()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js#L8-L14) 根据平台判断：

| 平台 | 默认连接方式 |
|------|------------|
| Linux / macOS | `{ socketPath: "/var/run/docker.sock" }` |
| Windows | `{ host: "127.0.0.1" }` |

**返回值结构：**
```javascript
// 配置文件解析时返回：
{ conn: { socketPath?, host?, port?, ... }, swarm: true|false }

// server 参数为空时返回默认参数（无 conn 外层包装）
{ socketPath: "/var/run/docker.sock" }  // 或 { host: "127.0.0.1" }
```

---

### 1.2 Docker 容器标签发现（精确边界）

**核心文件：** [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js)

**核心函数：** [servicesFromDocker()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L63-L170)

#### 标签发现的边界条件（代码事实）

| 条件 | 结果 | 代码位置 |
|------|------|---------|
| Label 不以 `homepage.` 开头 | 跳过 | [service-helpers.js L99](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L99) |
| `homepage.instance.xxx` 但 instanceName 不匹配 | 跳过该 Label（不是跳过整个容器） | [service-helpers.js L103-L104](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L103-L104) |
| `homepage.instance.{name}.xxx` 且 instanceName 匹配 | 保留，去掉 `instance.{name}.` 前缀 | [service-helpers.js L101-L102](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L101-L102) |
| 没有 `homepage.name` **或** 没有 `homepage.group` | 整个容器被丢弃（log error） | [service-helpers.js L123-L131](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L123-L131) |
| 容器连接失败 / 返回非数组 | 该服务器返回空 services | [service-helpers.js L89-L91](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L89-L91) |
| Swarm 模式下 Label 位置 | 从 `Spec.Labels` 取，而非顶层 Labels | [service-helpers.js L95](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L95) |
| 容器名的 `/` 前缀 | 自动去掉（replace(/^\//, "")） | [service-helpers.js L109](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L109) |

#### 自动注入的字段（初始 constructedService）

```javascript
{
  container: containerName.replace(/^\//, ""),
  server: serverName,
  weight: 0,
  type: "service"
}
```

#### shvl 路径转换

使用 [shvl.set()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js#L38-L64) 将点号+方括号路径转为嵌套对象。
安全保护：拦截 `__proto__` / `constructor` / `prototype`，防止原型污染。见 [shvl.js L46](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js#L46)

---

### 1.3 Kubernetes 资源注解发现

**核心文件：**
- 入口：[servicesFromKubernetes()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L172-L229)
- 资源处理：[resource-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js)
- 配置：[kubernetes.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/kubernetes.js)

#### 连接模式（由 kubernetes.yaml 中 mode 决定）

| mode | 加载方式 | 代码位置 |
|------|--------|---------|
| `cluster` | `kc.loadFromCluster()`（集群内 ServiceAccount） | [kubernetes.js L23](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/kubernetes.js#L23) |
| `default` | `kc.loadFromDefault()`（kubeconfig） | [kubernetes.js L26](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/kubernetes.js#L26) |
| `disabled` / 未配置 | 返回 null，不启用 K8s 发现 | [kubernetes.js L28-L31](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/kubernetes.js#L28-L31) |

#### 扫描的三类资源

| 资源类型 | 开关（kubernetes.yaml） | 实现文件 |
|---------|---------------------|---------|
| **Ingress** (networking.k8s.io/v1) | `ingress: true`（默认 true） | [ingress-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js) |
| **Traefik IngressRoute** (CRD) | `traefik: true` | [traefik-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js) |
| **HTTPRoute** (Gateway API) | `gateway: true` | [httproute-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js) |

#### 发现前置条件（isDiscoverable）

**函数：** [isDiscoverable(resource, instanceName)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js#L79-L87)

资源必须同时满足：
1. **存在 annotations**
2. **`gethomepage.dev/enabled === "true"`**（字符串 "true"，不是布尔值）
3. **实例匹配**（三选一）：
   - 没有 `gethomepage.dev/instance` → 通过
   - `gethomepage.dev/instance === instanceName` → 通过
   - 存在 `gethomepage.dev/instance.{instanceName}` 键 → 通过

#### URL 自动推导规则

| 资源类型 | 推导逻辑 | 失败时 |
|---------|---------|-------|
| **Ingress** | `http(s)://{host}{path}`，有 TLS 则 https | - |
| **HTTPRoute** | `{schema}://{hostname}{path}`，schema 从 Gateway listener.protocol 获取 | Gateway 查询失败 → 默认 `"http"` |
| **Traefik** | **不自动推导**，必须有 `gethomepage.dev/href` 才会被纳入 | 无 href → 该资源被过滤掉 |

---

### 1.4 静态配置文件来源

**函数：** [servicesFromConfig()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L53-L61)

- 从 `services.yaml` 读取，通过 `parseServicesToGroups()` 将 YAML 结构转为 Group 数组
- **唯一支持嵌套 group** 的来源（Docker/K8s 发现的服务只有扁平 group）
- 每个 service 自动赋默认 weight：`(index + 1) * 100`

---

## 二、数据整理层（Data Aggregation）— 重点：布局分组与空组边界

### 2.1 服务合并主入口

**核心文件：** [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js)

**核心函数：** [servicesResponse()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L158-L256)

#### 合并流程（精确步骤）

```
Step 1: 加载三来源数据（各自 try/catch，互不影响）
   │
   ├─ discoveredDockerServices
   │     = cleanServiceGroups(await servicesFromDocker())
   │     失败 → console.error + 返回 []
   │
   ├─ discoveredKubernetesServices
   │     = cleanServiceGroups(await servicesFromKubernetes())
   │     失败 → console.error + 返回 []
   │
   └─ configuredServices
         = cleanServiceGroups(await servicesFromConfig())
         失败 → console.error + 返回 []

Step 2: 获取所有 group 名的并集
   mergedGroupsNames = [...new Set([...docker, ...k8s, ...config].map(g => g.name))]

Step 3: 如果有 settings.layout
   │  将 layout 中定义的空 group 合并进 configuredServices
   │  （即使该 group 在 services.yaml 中不存在）
   │
   ├─ convertLayoutGroupToGroup() 为每个 layout key 创建空 group
   │   结构：{ name, services: [], groups: [] }
   │   若 layout value 是对象 → 递归创建嵌套空 subgroup
   │
   └─ mergeLayoutGroupsIntoConfigured() 合并到 configuredServices
       ├─ group 已存在于 configuredServices → 合并子组
       └─ group 不存在 → push 空组进 configuredServices

Step 4: 遍历每个 groupName，执行合并
   │
   ├─ 分别从三来源找同名 group
   │   （findGroupByName 支持深层搜索，设 parent 属性）
   │
   ├─ 合并 services：[...docker, ...k8s, ...config].filter(Boolean).sort()
   │
   ├─ 合并 groups：[...configuredGroup.groups]（仅 config 有嵌套）
   │
   └─ 排序策略：
        ├─ group 在 layout 中 → sortedGroups[layoutIndex] = mergedGroup
        ├─ group 有 parent → mergeSubgroups() + ensureParentGroupExists()
        └─ 其他 → unsortedGroups

Step 5: 结果 = sortedGroups.filter(g => g) + unsortedGroups
        （sortedGroups 中未赋值的 slot 是 undefined，需 filter 去掉）

Step 6: pruneEmptyGroups() 递归剪枝 ← ★ 空组的最终决定者
```

---

### 2.2 ★ 空分组展示的精确边界（代码逐行验证）

这是最容易误解的部分。代码注释写「handles cases where groups are only defined in layout」，但最终展示由 `pruneEmptyGroups()` 决定。

#### pruneEmptyGroups() 精确逻辑

**函数：** [pruneEmptyGroups()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L123-L134)

```javascript
function pruneEmptyGroups(groups) {
  return groups.filter((group) => {
    if (group.services.length === 0 && group.groups.length === 0) {
      return false;           // ★ 同时满足两个空条件 → 被剪掉
    }
    if (group.groups.length > 0) {
      group.groups = pruneEmptyGroups(group.groups);  // 递归剪枝子组
    }
    return true;              // 有 services 或有子组 → 保留
  });
}
```

**判定规则（代码事实）：**

| services | groups | 递归剪枝后 groups | 最终结果 |
|----------|--------|-----------------|---------|
| `[]` | `[]` | `[]` | **被剪掉**（不展示） |
| `[svc]` | `[]` | `[]` | **保留**（有服务） |
| `[]` | `[subgroup]` | 视子组情况 | 取决于子组 |
| `[]` | `[subgroup(services:[])]` | `[]` | **被剪掉**（子组先被剪，自身也空了） |
| `[]` | `[subgroup(services:[svc])]` | `[subgroup]` | **保留**（子组存活，自身非空） |

#### 六种场景推演

**场景 A：layout 定义了空组，但 Docker/K8s 都没发现该 group 的服务**

```
settings.yaml layout:  { "Infrastructure": null }
services.yaml / Docker / K8s: 无任何 "Infrastructure" 服务

执行流程：
1. convertLayoutGroupToGroup("Infrastructure", null)
   → { name: "Infrastructure", services: [], groups: [] }
2. mergeLayoutGroupsIntoConfigured → push 到 configuredServices
3. mergedGroupsNames 包含 "Infrastructure"
4. 合并后 mergedGroup = { name: "Infrastructure", services: [], groups: [] }
5. pruneEmptyGroups → services.length===0 && groups.length===0 → 被剪掉

结果：★ 不展示
```

**场景 B：layout 定义了含嵌套子组的空组，子组也没有服务**

```
settings.yaml layout:  { "Infrastructure": { "Monitoring": null } }
Docker / K8s: 无 "Monitoring" 服务

执行流程：
1. convertLayoutGroupToGroup("Infrastructure", { Monitoring: null })
   → { name: "Infrastructure", services: [], groups: [
       { name: "Monitoring", services: [], groups: [] }
     ] }
2. 合并进 configuredServices
3. "Infrastructure" 出现在 mergedGroupsNames
4. mergedGroup = { name: "Infrastructure", services: [], groups: [空Monitoring] }
5. pruneEmptyGroups 递归：
   - Monitoring: services=[] && groups=[] → 被剪
   - Infrastructure: services=[] && groups=[]（子组被剪后变空）→ 被剪

结果：★ 不展示
```

**场景 C：layout 定义了嵌套子组，子组有 Docker 发现的服务**

```
settings.yaml layout:  { "Infrastructure": { "Monitoring": null } }
Docker: 容器带 homepage.group=Monitoring

执行流程：
1. convertLayoutGroupToGroup 创建空 Infrastructure + 空 Monitoring
2. 合并进 configuredServices
3. "Infrastructure" 和 "Monitoring" 都在 mergedGroupsNames
4. findGroupByName(configuredServices, "Monitoring") 深层搜索找到，设 parent="Infrastructure"
5. Monitoring 的 mergedGroup 包含 Docker 发现的服务
6. mergeSubgroups 将 Docker 服务注入 Infrastructure 下的 Monitoring 子组
7. ensureParentGroupExists 将 Infrastructure 加入 sortedGroups
8. pruneEmptyGroups：
   - Monitoring: services 非空 → 保留
   - Infrastructure: groups 非空 → 保留

结果：★ 展示（Infrastructure > Monitoring > 服务列表）
```

**场景 D：layout 定义了组，该组有静态配置的服务**

```
settings.yaml layout:  { "Infrastructure": null }
services.yaml: 有 Infrastructure group 的服务

执行流程：
1. mergeLayoutGroupsIntoConfigured → Infrastructure 已存在，不重复添加
2. 合并后 mergedGroup.services 非空
3. pruneEmptyGroups → services 非空 → 保留

结果：★ 展示
```

**场景 E：Docker/K8s 发现了一个不在 layout 中的 group**

```
Docker: 容器带 homepage.group=NewService
settings.yaml layout: 不包含 "NewService"

执行流程：
1. "NewService" 出现在 mergedGroupsNames
2. configuredGroup = findGroupByName(configuredServices, "NewService") → null
   → 用 { services: [], groups: [] } 替代
3. mergedGroup = { name: "NewService", services: [Docker服务], groups: [] }
4. layoutIndex = -1 → 放入 unsortedGroups
5. pruneEmptyGroups → services 非空 → 保留

结果：★ 展示（排在 layout 定义的组之后）
```

**场景 F：layout 和 Docker 定义了同名组，服务合并**

```
settings.yaml layout:  { "Infrastructure": null }
services.yaml: Infrastructure 有 2 个服务
Docker: 容器带 homepage.group=Infrastructure 有 1 个服务

执行流程：
1. mergeLayoutGroupsIntoConfigured → Infrastructure 已存在，跳过
2. 合并 mergedGroup.services = [...Docker, ...Config] = 3 个服务
3. 按 layout 索引放入 sortedGroups
4. pruneEmptyGroups → services 非空 → 保留

结果：★ 展示（3 个服务合并，按 weight+name 排序）
```

#### 渲染层的空 services 容错

**组件：** [group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx)

如果 `pruneEmptyGroups` 之后某个 group 的 services 为空但 groups 非空（有存活的子组），渲染层的处理：

```jsx
// group.jsx L82-L108
<List services={group.services} ... />          {/* services=[] 时渲染空 <ul> */}
{group.groups?.length > 0 && (
  group.groups.map((subgroup) => (
    <ServicesGroup group={subgroup} ... />      {/* 递归渲染子组 */}
  ))
)}
```

- `List` 组件收到空 services → 渲染空的 `<ul>`（无 `<Item>` 子元素）
- 子组正常递归渲染
- **视觉上**：组标题显示，但内容区只有子组，没有服务卡片

---

### 2.3 数据标准化（cleanServiceGroups）

**函数：** [cleanServiceGroups()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L231-L708)

核心目的：类型统一 + Widget 字段白名单安全过滤。

**白名单机制**：解构赋值方式，仅明确列出的字段（约 90 个）会被重新组装回 widget 对象，恶意字段在此步被过滤。

---

## 三、页面展示层 — 重点：Kubernetes 接口错误完整传播路径

### 3.1 首屏数据链路

**getStaticProps()** 在构建时直接调用 `servicesResponse()`（不走 HTTP），结果注入 SWR fallback。客户端 `useSWR("/api/services")` 优先使用 fallback，后台自动 revalidate。

---

### 3.2 ★ SWR Fetcher 的关键设计（最易误解的代码事实）

**全局 fetcher 定义：** [index.jsx L186](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L186) / [_app.jsx L77](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/_app.jsx#L77)

```jsx
<SWRConfig value={{
  fallback,
  fetcher: (resource, init) => fetch(resource, init).then((res) => res.json())
}}>
```

**★ 代码事实**：fetcher **没有检查 `res.ok` 或 `res.status`**！

这意味着：
- **HTTP 2xx**：`res.json()` 正常解析 → `data` = 解析后的对象，`error` = undefined
- **HTTP 4xx/5xx**：`res.json()` 仍会尝试解析 body → 只要是有效 JSON，`data` = 解析后的对象，`error` = undefined
- **网络错误**（DNS 失败 / CORS / 网络断开）：`fetch()` reject → `error` = Error 对象，`data` = undefined
- **JSON 解析失败**（body 不是有效 JSON）：`res.json()` reject → `error` = SyntaxError 对象，`data` = undefined

**四种 HTTP 响应场景的 SWR 结果对照表：**

| HTTP 状态 | JSON Body | `data` (SWR) | `error` (SWR) |
|----------|-----------|-------------|--------------|
| 200 | `{ status: "running" }` | `{ status: "running" }` | undefined |
| 404 | `{ status: "not found" }` | `{ status: "not found" }` | undefined |
| 500 | `{ error: "No kubernetes configuration" }` | `{ error: "No kubernetes configuration" }` | undefined |
| 网络断开 | - | undefined | TypeError 对象 |

> **这是之前分析最大的错误修正**：HTTP 4xx/5xx 的错误信息在 `data.error` 中，而不是 `error` 对象中！
> `error` 对象只有在 fetch 本身失败（网络层）时才会被设置。

---

### 3.3 ★ Kubernetes 资源错误降级（完整路径追踪）

K8s 的错误路径横跨三层：**发现层 → API 端点 → 渲染组件**，每层有独立的降级策略。

#### 第一层：服务发现阶段

**入口：** [servicesFromKubernetes()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L172-L229)

```
整体 try/catch → 失败时 throw e
↑ 被 servicesResponse() 的 try/catch 捕获 → discoveredKubernetesServices = []
```

三类资源列表**并行**获取：`Promise.all([listIngress(), listTraefikIngress(), listHttpRoute()])`

##### Ingress 列表错误处理

**函数：** [listIngress()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js#L9-L26)

| 错误场景 | 行为 | 代码位置 |
|---------|------|---------|
| kubernetes.yaml 中 `ingress: false` | 直接返回 `[]`，不发请求 | [L11](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js#L11) |
| `listIngressForAllNamespaces()` 失败 | catch → logger.error + 返回 null | [L18-L22](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js#L18-L22) |
| 返回 null | `ingressData?.items ?? []` → 返回 `[]` | [L23](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js#L23) |

**降级结果**：Ingress 列表失败 → 该类资源贡献 0 个服务，不影响 Traefik/HTTPRoute

##### Traefik IngressRoute 列表错误处理

**函数：** [listTraefikIngress()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L9-L68)

| 错误场景 | 行为 | 代码位置 |
|---------|------|---------|
| kubernetes.yaml 中 `traefik` 为空 | 直接返回 `[]` | [L13](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L13) |
| `traefik.containo.us` CRD 不存在 | checkCRD 返回 false，API 失败时不 log error | [L15](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L15) |
| `traefik.containo.us` API 失败 | checkCRD 为 true → log error；否则静默；返回 `[]` | [L24-L36](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L24-L36) |
| `traefik.io` API 失败 | 同上逻辑 | [L44-L56](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L44-L56) |
| 合并后列表无 `gethomepage.dev/href` 注解的资源 | 过滤后为空，返回 `[]` | [L61-L64](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L61-L64) |

**关键设计**：`checkCRD()` 先检测 CRD 是否存在，避免对不存在的 CRD 产生无意义的 404 错误日志。

**降级结果**：Traefik 列表失败 → 贡献 0 个服务。两个 API Group 互相独立，一个失败另一个仍可工作。

##### HTTPRoute 列表错误处理

**函数：** [listHttpRoute()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L9-L55)

| 错误场景 | 行为 | 代码位置 |
|---------|------|---------|
| kubernetes.yaml 中 `gateway` 为空 | 直接返回 `[]` | [L15](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L15) |
| `listNamespace()` 失败 | catch → logger.error + 返回 null | [L37-L41](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L37-L41) |
| namespaces 为 null | **跳过整个 httproute 查询**，返回 `[]` | [L43](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L43) |
| 单个 namespace 的 httproute 查询失败 | catch → logger.error + 返回 null | [L28-L32](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L28-L32) |
| null 结果过滤 | `filter((httpRoute) => httpRoute)` 去掉 | [L51](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js#L51) |

**降级结果**：namespace 列表失败 → 整个 HTTPRoute 发现跳过。单个 namespace 失败 → 不影响其他 namespace。

---

#### 第二层：API 端点阶段（客户端按需请求）

##### K8s 状态 API — HTTP 状态与 JSON 映射

**文件：** [pages/api/kubernetes/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js)

| 错误场景 | HTTP 状态 | JSON Body | 代码位置 |
|---------|----------|-----------|---------|
| 缺少 namespace/appName | 400 | `{ error: "kubernetes query parameters are required" }` | [L13-L17](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L13-L17) |
| getKubeConfig() 返回 null | 500 | `{ error: "No kubernetes configuration" }` | [L22-L26](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L22-L26) |
| listNamespacedPod 失败 | 500 | `{ error: "Error communicating with kubernetes" }` | [L34-L42](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L34-L42) |
| Pod 列表为空 | 404 | `{ status: "not found" }` | [L46-L51](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L46-L51) |
| 所有 Pod 都 Ready | 200 | `{ status: "running" }` | [L56-L57](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L56-L57) |
| 部分 Pod Ready | 200 | `{ status: "partial" }` | [L58-L59](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L58-L59) |
| 无 Pod Ready | 200 | `{ status: "down" }` | [L55](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L55) |
| 未知异常 | 500 | `{ error: "unknown error" }` | [L64-L68](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js#L64-L68) |

##### K8s 统计 API — HTTP 状态与 JSON 映射

**文件：** [pages/api/kubernetes/stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js)

| 错误场景 | HTTP 状态 | JSON Body | 代码位置 |
|---------|----------|-----------|---------|
| 缺少 namespace/appName | 400 | `{ error: "..." }` | [L14-L18](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L14-L18) |
| getKubeConfig() 返回 null | 500 | `{ error: "No kubernetes configuration" }` | [L23-L27](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L23-L27) |
| listNamespacedPod 失败 | 500 | `{ error: "Error communicating with kubernetes" }` | [L37-L45](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L37-L45) |
| Pod 列表为空 | 404 | `{ error: "no pods found..." }` | [L49-L53](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L49-L53) |
| getPodMetrics 失败 (404) | 200 | `{ stats: { cpu: 0, mem: 0, cpuLimit, memLimit, cpuUsage: 0, memUsage: 0 } }` | [L71-L101](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L71-L101) |
| getPodMetrics 失败 (非 404) | 200 | `{ stats: { cpu: 0, mem: 0, ... } }`（静默降级） | [L76-L78](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L76-L78) |
| 未知异常 | 500 | `{ error: "unknown error" }` | [L104-L108](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js#L104-L108) |

**★ 关键降级行为**：metrics-server 不可用时，stats API 仍返回 200，但 cpu/mem 值为 0。
这导致前端不会显示错误，而是显示 **0% CPU / 0 bytes Mem**，用户可能误以为服务确实没有资源消耗。

---

#### 第三层：渲染组件阶段 — 完整传播路径

##### ★ K8s Stats API 错误 → K8s Component 的精确传播

这是最复杂的路径。**后端 HTTP 状态 → SWR 结果 → 组件分支判断 → 最终展示** 五步走：

| 后端场景 | HTTP 状态 | JSON Body | SWR `data` | SWR `error` | 进入哪个组件分支 | 最终展示 |
|---------|----------|-----------|-----------|------------|----------------|---------|
| **缺少参数** | 400 | `{ error: "..." }` | `{ error: "..." }` | undefined | `statsError` false<br>`statusData.status` 正常（假设 status API 正常）→ 不进 offline 分支<br>`statsData` 非空 → 不进骨架分支<br>`statsData.stats` = undefined<br>`statsData.stats.cpuLimit` → **抛出 TypeError** | React 渲染错误（可能被 ErrorBoundary 捕获） |
| **无 K8s 配置** | 500 | `{ error: "No kubernetes configuration" }` | `{ error: "..." }` | undefined | 同上 → **TypeError** | 同上 |
| **Pod 不存在** | 404 | `{ error: "no pods found..." }` | `{ error: "..." }` | undefined | 同上 → **TypeError** | 同上 |
| **metrics 不可用** | 200 | `{ stats: { cpu:0, mem:0, ... } }` | `{ stats: { cpu:0, mem:0, ... } }` | undefined | 正常渲染分支 | 显示 **0% CPU / 0 bytes**（静默降级） |
| **正常** | 200 | `{ stats: { cpu, mem, ... } }` | `{ stats: { cpu, mem, ... } }` | undefined | 正常渲染分支 | 正常显示指标 |
| **网络断开** | - | - | undefined | TypeError 对象 | `if (statsError)` → 进入 error 分支 | Container → Error 组件（或 hideErrors → null） |
| **JSON 解析失败** | 200（无效 JSON） | `<html>...</html>` | undefined | SyntaxError 对象 | `if (statsError)` → 进入 error 分支 | Container → Error 组件 |

**代码逐行验证：** [widgets/kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/kubernetes/component.jsx)

```jsx
// L11-L17: 发起 SWR 请求
const { data: statusData, error: statusError } = useSWR(`/api/kubernetes/status/...`);
const { data: statsData, error: statsError } = useSWR(`/api/kubernetes/stats/...`);

// L19-L21: 只检测 SWR error（网络级失败）
// ★ 注意：statsData.error（业务级错误）完全不检测！
if (statsError || statusError) {
  return <Container service={service} error={statsError ?? statusError ?? statusData} />;
}

// L23-L32: 检测 statusData.status 不是 running/partial
// 当 status API 返回 500 { error: "..." } 时：
//   statusData = { error: "..." }, statusData.status = undefined
//   !undefined = true → 进入此分支，显示 "offline"
if (statusData && (!statusData.status || 
    !(statusData.status.includes("running") || statusData.status.includes("partial")))) {
  return <Container><Block label="widget.status" value="docker.offline" /></Container>;
}

// L34-L41: 数据仍在加载
if (!statsData || !statusData) {
  return <Container service={service}>
    <Block label="docker.cpu" />
    <Block label="docker.mem" />
  </Container>;
}

// L43-L64: 正常渲染
// 当 stats API 返回 500 { error: "..." } 时：
//   statsData = { error: "..." }, statsData.stats = undefined
//   statsData.stats.cpuLimit → 抛出 TypeError！
return (
  <Container service={service}>
    {(statsData.stats.cpuLimit && (...)) || (
      <Block ... value={statsData.stats.cpu} />
    )}
    <Block ... value={statsData.stats.mem} />
  </Container>
);
```

##### ★ K8s Status API 错误 → K8s Component 的精确传播

| 后端场景 | HTTP 状态 | JSON Body | SWR `data` | SWR `error` | 进入哪个组件分支 | 最终展示 |
|---------|----------|-----------|-----------|------------|----------------|---------|
| **缺少参数** | 400 | `{ error: "..." }` | `{ error: "..." }` | undefined | `statusError` false<br>`statusData.status` = undefined → 进入 offline 分支 | "offline" 状态 Block |
| **无 K8s 配置** | 500 | `{ error: "No kubernetes configuration" }` | `{ error: "..." }` | undefined | 同上 → offline 分支 | "offline" 状态 Block |
| **Pod 不存在** | 404 | `{ status: "not found" }` | `{ status: "not found" }` | undefined | `statusData.status = "not found"` → 不含 "running" → offline 分支 | "offline" 状态 Block |
| **无 Pod Ready** | 200 | `{ status: "down" }` | `{ status: "down" }` | undefined | 同上 → offline 分支 | "offline" 状态 Block |
| **正常** | 200 | `{ status: "running" }` | `{ status: "running" }` | undefined | 正常渲染分支 | 正常显示指标 |
| **网络断开** | - | - | undefined | TypeError 对象 | `if (statusError)` → error 分支 | Container → Error 组件 |

> **关键发现**：K8s Status API 返回 4xx/5xx `{ error: "..." }` 时，K8s Component **不进 error 分支**，而是显示 **"offline"**！
> 因为它只检查 `statusError`（SWR error 对象），不检查 `statusData.error`（JSON body 中的 error 字段）。

##### Docker 组件对比（多一层检测）

**文件：** [widgets/docker/component.jsx L19](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx#L19)

```jsx
// ★ Docker 比 K8s 多检测 statsData?.error 和 statusData?.error（业务级错误）
if (statsError || statsData?.error || statusError || statusData?.error) {
  return <Container service={service} error={finalError} />;
}
```

| 对比项 | K8s Component | Docker Component |
|--------|--------------|-----------------|
| 检测 `error` (SWR 网络级) | ✅ | ✅ |
| 检测 `data.error` (业务级) | ❌ | ✅ |
| API 返回 500 `{ error: "..." }` 时 | 显示 "offline" 或抛 TypeError | 显示 Error 组件 |

##### K8s Status 徽章组件的传播路径

**文件：** [kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/kubernetes-status.jsx)

| 后端场景 | JSON Body | `data` | `error` | 状态文本 | 颜色 |
|---------|-----------|--------|---------|---------|------|
| **缺少参数/配置** | `{ error: "..." }` | `{ error: "..." }` | undefined | `docker.unknown` | 灰色 |
| **Pod 不存在** | `{ status: "not found" }` | `{ status: "not found" }` | undefined | `not found` | 橙色 |
| **无 Pod Ready** | `{ status: "down" }` | `{ status: "down" }` | undefined | `down` | 橙色 |
| **部分 Ready** | `{ status: "partial" }` | `{ status: "partial" }` | undefined | `partial` | 橙色 |
| **正常** | `{ status: "running" }` | `{ status: "running" }` | undefined | `running` | 绿色 |
| **网络断开** | - | undefined | TypeError | `docker.error` | 红色 |

**代码逻辑：**
```jsx
if (error) {                  // 只有网络错误才进这里
  statusLabel = t("docker.error");
  colorClass = "text-rose-500/80";
} else if (data) {
  if (data.status === "running") { /* 绿色 */ }
  if (data.status === "not found" || data.status === "down" || data.status === "partial") { /* 橙色 */ }
  // ★ data.error 场景：data.status = undefined → 两个 if 都不进，保持默认 "unknown" 灰色
}
```

> **关键发现**：K8s Status API 返回 4xx/5xx `{ error: "..." }` 时，徽章显示 **灰色 "unknown"**，而不是红色 "error"！

##### Container 错误展示组件

**文件：** [widget/container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/widget/container.jsx)

```jsx
// L24-L29: 错误处理
if (error) {
  if (settings.hideErrors || service.widget.hide_errors) {
    return null;              // ★ 全局或 widget 级别隐藏错误 → 完全不渲染
  }
  return <Error service={service} error={error} />;  // 否则显示可展开的错误详情
}
```

**两个级别的错误隐藏：**
1. `settings.hideErrors` — 全局设置
2. `service.widget.hide_errors` — 单个 widget 设置（Docker Label: `homepage.widgets[0].hideErrors=true`）

##### Error 组件

**文件：** [widget/error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/widget/error.jsx)

```jsx
// L21-L23: 解包 error.data.error
// ★ 代码事实：error.data.error 永远不存在！
// 因为本项目的 fetcher 没有构造这种结构。只有自定义 fetcher 手动 throw { data: ... } 才会有。
if (error?.data?.error) {
  error = error.data.error;
}
```

**显示内容：**
- 渲染为可展开的 `<details>` 元素
- 显示 API error message、请求 URL、raw error、response data
- 错误对象归一化：string → `{ message }`, number → `{ message: "Error N" }`

---

### 3.4 ★ 错误降级全景决策树（修正后）

```
K8s 服务（从发现到展示）
  │
  ├─ 发现阶段
  │   ├─ kubernetes.yaml 配置错误 / K8s 不可达
  │   │   → servicesFromKubernetes() 抛异常
  │   │   → servicesResponse catch → discoveredKubernetesServices = []
  │   │   → ★ 该 Homepage 实例完全无 K8s 服务，不影响 Docker/Config
  │   │
  │   ├─ Ingress API 失败
  │   │   → listIngress 返回 []
  │   │   → ★ 仅丢失 Ingress 服务，Traefik/HTTPRoute 仍可工作
  │   │
  │   ├─ Traefik API 失败（两个 Group 都失败）
  │   │   → listTraefikIngress 返回 []
  │   │   → ★ 仅丢失 Traefik 服务
  │   │
  │   └─ HTTPRoute namespace 列表失败
  │       → listHttpRoute 返回 []
  │       → ★ 仅丢失 HTTPRoute 服务
  │
  ├─ API 端点阶段（客户端按需请求）
  │   │
  │   ├─ Status API 场景：
  │   │   ├─ HTTP 4xx/5xx { error: "..." }
  │   │   │   → data = { error: "..." }, error = undefined
  │   │   │   → K8s Component: data.status undefined → offline 分支 → "offline"
  │   │   │   → K8s Status 徽章: data.status undefined → 默认 → 灰色 "unknown"
  │   │   │
  │   │   ├─ HTTP 404 { status: "not found" }
  │   │   │   → data = { status: "not found" }, error = undefined
  │   │   │   → K8s Component: status 不含 "running" → offline 分支 → "offline"
  │   │   │   → K8s Status 徽章: status="not found" → 橙色
  │   │   │
  │   │   ├─ 网络断开
  │   │   │   → data = undefined, error = TypeError
  │   │   │   → K8s Component: error 分支 → Container → Error 组件
  │   │   │   → K8s Status 徽章: error 分支 → 红色 "error"
  │   │   │
  │   │   └─ 正常 → 显示 running/partial 状态
  │   │
  │   └─ Stats API 场景：
  │       ├─ HTTP 4xx/5xx { error: "..." }
  │       │   → data = { error: "..." }, error = undefined
  │       │   → K8s Component: data.stats undefined → data.stats.cpuLimit 抛出 TypeError
  │       │   → ★ React 渲染错误（潜在 Bug）
  │       │
  │       ├─ metrics-server 不可用
  │       │   → data = { stats: { cpu:0, mem:0, ... } }, error = undefined
  │       │   → 正常渲染，显示 0% CPU / 0 bytes（静默降级）
  │       │
  │       ├─ 网络断开
  │       │   → data = undefined, error = TypeError
  │       │   → error 分支 → Container → Error 组件
  │       │
  │       └─ 正常 → 显示 CPU / Mem 指标
  │
  └─ 渲染阶段
      ├─ error + hideErrors → 不渲染
      ├─ error + 不隐藏 → Error 详情组件
      ├─ status=down/not found/error(业务级) → "offline"
      ├─ status=running/partial → 显示指标
      └─ 数据加载中 → 骨架屏
```

---

### 3.5 Docker 与 K8s 错误处理对比（修正后）

| 对比维度 | Docker | Kubernetes |
|---------|--------|-----------|
| **SWR fetcher** | 共享全局 fetcher，**不检查 `res.ok`** | 同 |
| **HTTP 4xx/5xx → SWR** | `data.error` 有值，`error` = undefined | 同 |
| **API error 检测** | ✅ 检测 `statsData?.error` + `statusData?.error`（业务级） | ❌ 仅检测 `error`（网络级），不检测 `data.error` |
| **API 返回 500 `{ error: "..." }`** | 进入 error 分支 → Error 组件 | K8s Component: offline 分支（status API）/ TypeError（stats API）<br>K8s Status 徽章: 灰色 "unknown" |
| **发现阶段容错** | 单服务器失败 → 该服务器返回 []，不影响其他 | 三类资源并行，一类失败不影响其他 |
| **metrics 不可用** | 不适用（Docker stats 直连） | ✅ 静默返回 0 值，前端显示 0% CPU / 0 bytes |
| **容器/Pod 不存在** | API 返回 `{ status: "not found" }` → offline | API 返回 404 `{ status: "not found" }` → offline |
| **offline 判断** | status 不含 "running"/"partial" | 同 |
| **骨架屏 Block 数** | 4 个（CPU/Mem/RX/TX） | 2 个（CPU/Mem） |
| **错误隐藏** | `hideErrors` 或 `widget.hide_errors` | 同 |
| **`error.data.error` 检查** | - | Error 组件有此检查，但 **永远不会命中**（fetcher 不构造此结构） |
| **潜在 Bug** | - | K8s Component stats API 返回 `{ error: "..." }` 时 → `statsData.stats.cpuLimit` 抛 **TypeError** |

---

### 3.6 组件层级结构

```
pages/index.jsx (Home 组件)
  │
  └── ServicesGroup ([group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx))
      │  可折叠分组，Disclosure + Transition 动画
      │
      ├── 标题栏（可选 icon + group name + 折叠箭头）
      │
      └── List ([list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/list.jsx))
          │  services=[] → 渲染空 <ul>
          │
          └── Item ([item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/item.jsx))
              ├── ResolvedIcon — 服务图标
              ├── 服务名称 & 描述
              ├── 状态标签区
              │   ├── Status [Docker]  ← service.container 存在时
              │   ├── KubernetesStatus ← service.app 存在时
              │   └── 其他...
              └── 展开的 Stats 区域
                  ├── Docker Component → Block × 4
                  ├── K8s Component → Block × 2
                  └── 其他 widget...

      └── 子组递归渲染（group.groups?.length > 0）
          └── ServicesGroup (isSubgroup=true)
```

---

## 四、完整数据流时序图（含错误路径精确标注）

```
┌───────────────┐
│ Next.js Build │
└───────┬───────┘
        │ getStaticProps()
        ▼
┌─────────────────────────────────────┐
│ servicesResponse()                  │
│  ├─ servicesFromDocker()            │  try/catch → 失败 = []
│  ├─ servicesFromKubernetes()        │  try/catch → 失败 = []
│  │   ├─ listIngress()              │  catch → []
│  │   ├─ listTraefikIngress()       │  catch → []
│  │   └─ listHttpRoute()            │  catch → []
│  ├─ servicesFromConfig()            │  try/catch → 失败 = []
│  ├─ cleanServiceGroups() × 3        │
│  ├─ layout 空组合并                  │
│  ├─ 按 group 合并 + 排序            │
│  └─ pruneEmptyGroups() 递归剪枝     │  ← 决定空组是否展示
└───────────────┬─────────────────────┘
                │ SWR fallback
                ▼
        ┌───────────────┐
        │  浏览器首屏    │  零请求直出
        └───────┬───────┘
                │ useSWR revalidate
                ▼
        ServicesGroup 渲染
        └─ Item
            ├─ KubernetesStatus 徽章
            │  └─ useSWR(/api/kubernetes/status)
            │       ├─ HTTP 200 { status: "running" } → 绿色 running
            │       ├─ HTTP 4xx/5xx { error: "..." } → data.error 有值
            │       │   → data.status = undefined → 灰色 unknown
            │       ├─ HTTP 404 { status: "not found" } → 橙色 not found
            │       ├─ 网络断开 → error 对象 → 红色 error
            │       └─ JSON 解析失败 → error 对象 → 红色 error
            │
            └─ Stats 展开
               └─ K8s Component
                  ├─ useSWR(/api/kubernetes/status)
                  │  ├─ HTTP 4xx/5xx { error } → statusData.status undefined → offline
                  │  └─ 网络断开 → error 对象 → Container → Error 组件
                  │
                  └─ useSWR(/api/kubernetes/stats)
                     ├─ HTTP 200 { stats: { cpu, mem, ... } } → 正常渲染
                     ├─ HTTP 200 { stats: { cpu:0, mem:0 } } → 0% CPU / 0 bytes（静默降级）
                     ├─ HTTP 4xx/5xx { error } → data.error 有值
                     │   → data.stats undefined
                     │   → data.stats.cpuLimit → TypeError（潜在 Bug）
                     ├─ 网络断开 → error 对象 → Container → Error 组件
                     └─ error + hideErrors → null
```

---

## 五、关键设计要点

### 5.1 空分组展示规则

- `convertLayoutGroupToGroup()` 允许 layout 定义无服务的空组
- `mergeLayoutGroupsIntoConfigured()` 将空组合入 configuredServices
- **但 `pruneEmptyGroups()` 是最终决定者**：services 和 groups 都空的组会被剪掉
- 只有子组中有服务时，父空组才会存活
- 渲染层：services=[] 的组会渲染空 `<ul>`，但不会报错

### 5.2 ★ SWR Fetcher 设计（最关键的代码事实）

- **全局 fetcher 不检查 `res.ok` 或 `res.status`**
- HTTP 4xx/5xx 的错误在 `data.error` 中，而不是 `error` 对象中
- `error` 对象只有在 **fetch 网络失败** 或 **JSON 解析失败** 时才会被设置
- 这是整个错误传播路径的**根基**，影响所有组件的判断逻辑

### 5.3 ★ K8s 组件的三层错误检测缺失

1. **业务级错误检测缺失**：K8s Component 不检测 `data.error`，只检测 `error`
   - Status API 返回 4xx/5xx → 显示 "offline" 而非错误
   - Stats API 返回 4xx/5xx → 抛出 TypeError（潜在 Bug）

2. **Status 徽章业务级错误检测缺失**：`data.error` 场景下 `data.status` = undefined → 显示灰色 "unknown"

3. **`error.data.error` 检查永远不命中**：Error 组件有此代码，但 fetcher 不构造这种结构

### 5.4 K8s 错误的分层降级

| 层级 | 粒度 | 降级策略 |
|-----|------|---------|
| 整体发现 | K8s 全局 | 抛异常 → servicesResponse catch → [] |
| 资源类型 | Ingress / Traefik / HTTPRoute | 各自 catch → []，互不影响 |
| Namespace | HTTPRoute 的 namespace | 单个 namespace 失败 → filter 掉 null |
| API 请求（Status） | 4xx/5xx `{ error }` | data.error 有值，error 无值 → 显示 offline / unknown |
| API 请求（Status） | 网络断开 | error 有值 → 显示红色 error |
| API 请求（Stats） | 4xx/5xx `{ error }` | data.error 有值，error 无值 → 抛 TypeError（潜在 Bug） |
| API 请求（Stats） | metrics 不可用 | 返回 `{ stats: { cpu:0, mem:0 } }` → 显示 0 值（静默降级） |
| API 请求（Stats） | 网络断开 | error 有值 → Error 组件 |
| 渲染 | Container | hideErrors → null；否则 → Error 详情 |

### 5.5 Docker 与 K8s 错误处理的关键差异

| 差异点 | Docker | Kubernetes |
|--------|--------|-----------|
| `data.error` 检测 | ✅ 检测 `statsData?.error` + `statusData?.error` | ❌ 不检测 |
| API 返回 500 `{ error }` | 进入 error 分支 → Error 组件 | Status: offline / unknown<br>Stats: 抛 TypeError |
| 静默降级场景 | 无 | metrics-server 不可用 → 0 值显示 |

### 5.6 安全设计

- Widget 字段白名单：`cleanServiceGroups()` 解构赋值过滤
- shvl 原型污染防护：拦截 `__proto__` / `constructor` / `prototype`
- 环境变量替换白名单：仅 `HOMEPAGE_VAR_` 和 `HOMEPAGE_FILE_` 前缀

### 5.7 性能优化

- SSR 预取 + SWR fallback：首屏零额外请求
- 按需加载：stats 数据只有展开卡片时才请求
- SWR 缓存：相同 key 的请求共享缓存
- K8s 资源并行获取：`Promise.all`

---

## 六、代码关联速查表

| 功能模块 | 文件路径 | 关键函数/组件 |
|---------|---------|-------------|
| SWR 全局 fetcher | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L186) / [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/_app.jsx#L77) | `fetcher` |
| Docker 连接配置 | [docker.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js) | `getDockerArguments()` |
| Docker 服务发现 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromDocker()` |
| K8s 服务发现 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromKubernetes()` |
| K8s 资源发现判断 | [resource-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js) | `isDiscoverable()` |
| K8s 服务构造 | [resource-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js) | `constructedServiceFromResource()` |
| K8s Ingress 列表 | [ingress-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js) | `listIngress()` |
| K8s Traefik 列表 | [traefik-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js) | `listTraefikIngress()` |
| K8s HTTPRoute 列表 | [httproute-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js) | `listHttpRoute()` |
| 服务合并主入口 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `servicesResponse()` |
| 空组创建 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `convertLayoutGroupToGroup()` |
| 空组合并 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `mergeLayoutGroupsIntoConfigured()` |
| 空组剪枝 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `pruneEmptyGroups()` |
| 子组合并 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `mergeSubgroups()` |
| 数据标准化 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `cleanServiceGroups()` |
| 页面入口 SSR | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx) | `getStaticProps()` |
| 服务分组组件 | [group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx) | `ServicesGroup` |
| Docker 状态徽章 | [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx) | `Status` |
| K8s 状态徽章 | [kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/kubernetes-status.jsx) | `KubernetesStatus` |
| Docker 统计组件 | [docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx) | `Component` |
| K8s 统计组件 | [kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/kubernetes/component.jsx) | `Component` |
| Widget 错误容器 | [container.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/widget/container.jsx) | `Container` |
| Widget 错误展示 | [error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/widget/error.jsx) | `Error` |
| K8s Status API | [status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/status/[...service].js) | `handler` |
| K8s Stats API | [stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/kubernetes/stats/[...service].js) | `handler` |
| Docker Status API | [status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/status/[...service].js) | `handler` |
| Docker Stats API | [stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/stats/[...service].js) | `handler` |
| shvl 路径工具 | [shvl.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js) | `set()` |
