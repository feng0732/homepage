# Homepage Kubernetes 服务发现代码路径分析

本文档梳理 Homepage 项目中 Kubernetes 集群资源采集到首页渲染的完整代码链路。

## 一、整体架构总览

整个 Kubernetes 服务发现与首页映射流程分为 6 个层次：

```
┌───────────────────────────────────────────────────────────┐
│ 1. 配置加载层 (Config Loading)                            │
│    kubernetes.yaml → KubeConfig 初始化                     │
├───────────────────────────────────────────────────────────┤
│ 2. 资源采集层 (Resource Discovery)                        │
│    Ingress / Traefik IngressRoute / Gateway HTTPRoute     │
├───────────────────────────────────────────────────────────┤
│ 3. 服务构建层 (Service Construction)                      │
│    注解解析 → isDiscoverable() → constructedServiceFrom…() │
├───────────────────────────────────────────────────────────┤
│ 4. 服务聚合层 (Service Aggregation)                       │
│    K8s + Docker + services.yaml 三方合并排序               │
├───────────────────────────────────────────────────────────┤
│ 5. 前端渲染层 (Frontend Rendering)                        │
│    /api/services → SWR → ServicesGroup → Item            │
├───────────────────────────────────────────────────────────┤
│ 6. 运行时监控层 (Runtime Monitoring)                      │
│    Pod 状态 / 资源使用率 → Widget 组件实时展示            │
└───────────────────────────────────────────────────────────┘
```

---

## 二、配置加载层

### 2.1 核心文件

- `src/utils/config/kubernetes.js`
- `src/utils/config/config.js`

### 2.2 关键函数

**`getKubernetes()`** — 读取并解析 `kubernetes.yaml`

```
流程：
1. checkAndCopyConfig("kubernetes.yaml")  // 确保配置文件存在，不存在则从 skeleton 复制
2. 读取 CONF_DIR/kubernetes.yaml          // 配置目录由 HOMEPAGE_CONFIG_DIR 环境变量决定
3. substituteEnvironmentVars()             // 替换 {{HOMEPAGE_VAR_*}} 和 {{HOMEPAGE_FILE_*}} 变量
4. yaml.load()                             // 解析为 JS 对象
```

**`getKubeConfig()`** — 根据配置模式初始化 Kubernetes 客户端

```javascript
switch (config?.mode) {
  case "cluster":  kc.loadFromCluster();  break;  // 集群内 ServiceAccount
  case "default":  kc.loadFromDefault();  break;  // ~/.kube/config
  case "disabled": default: return null;
}
```

配置项中 `ingress`、`traefik`、`gateway` 三个布尔开关分别控制三种资源类型的采集。

**`checkCRD(name, kc, logger)`** — 检查 CustomResourceDefinition 是否存在，仅用于 Traefik IngressRoute 采集前的 CRD 存在性判断。通过 `ApiextensionsV1Api.readCustomResourceDefinitionStatus` 查询，403 时提示 RBAC 权限不足。Gateway API 的 HTTPRoute 采集不使用此检查，而是直接按 namespace 查询，查询失败时静默返回 null。

### 2.3 注解常量定义

```javascript
ANNOTATION_BASE = "gethomepage.dev"
ANNOTATION_WIDGET_BASE = "gethomepage.dev/widget."
HTTPROUTE_API_GROUP = "gateway.networking.k8s.io"
HTTPROUTE_API_VERSION = "v1"
```

所有 Kubernetes 资源通过 `gethomepage.dev/*` 注解与 Homepage 交互。

---

## 三、资源采集层

三种入口资源类型并行采集，汇总后统一处理，入口统一在 `src/utils/kubernetes/export.js`。

### 3.1 标准 Ingress 采集

文件：`src/utils/kubernetes/ingress-list.js`

```
NetworkingV1Api.listIngressForAllNamespaces()
  → 成功：response（客户端库已自动解析 body，资源列表在 response.items）
  → 失败：catch 后返回 null
  → 最终：ingressData?.items ?? []  // 空数组兜底
```

受 `kubernetes.yaml` 中 `ingress: true/false` 控制，为 false 时直接返回空数组。

### 3.2 Traefik IngressRoute 采集

文件：`src/utils/kubernetes/traefik-list.js`

Traefik 存在两套 CRD API Group，需要同时尝试：

```
1. checkCRD("ingressroutes.traefik.containo.us")   // 旧版
2. checkCRD("ingressroutes.traefik.io")            // 新版

3. CustomObjectsApi.listClusterCustomObject({
     group: "traefik.containo.us" | "traefik.io",
     version: "v1alpha1",
     plural: "ingressroutes"
   })
   → 两套分别请求，失败时返回空数组（仅在 CRD 确实存在时才打错误日志）

4. 合并结果后，额外过滤条件：必须带有 gethomepage.dev/href 注解
   （因为 Traefik IngressRoute 的 URL 结构复杂，无法自动推断）
```

### 3.3 Gateway API HTTPRoute 采集

文件：`src/utils/kubernetes/httproute-list.js`

Gateway API 的 HTTPRoute 是命名空间级别资源，无全集群 List API，需逐个 namespace 查询：

```
1. CoreV1Api.listNamespace()
   → 成功：items.map(ns => ns.metadata.name)
   → 失败：返回 null

2. 对每个 namespace 调用：
   CustomObjectsApi.listNamespacedCustomObject({
     group: "gateway.networking.k8s.io",
     version: "v1",
     plural: "httproutes",
     namespace: <ns>
   })
   → 失败返回 null

3. Promise.all 并行查询后 flat().filter(Boolean) 过滤掉 null
```

---

## 四、服务构建层

### 4.1 核心文件

`src/utils/kubernetes/resource-helpers.js`

### 4.2 可发现性判断

**`isDiscoverable(resource, instanceName)`**

```
判断条件（全部满足）：
1. resource.metadata.annotations 存在
2. annotations["gethomepage.dev/enabled"] === "true"
3. instance 匹配（任一满足）：
   - 无 instance 注解
   - annotations["gethomepage.dev/instance"] === instanceName
   - annotations["gethomepage.dev/instance.<instanceName>"] 存在
```

instance 机制用于多 Homepage 实例共享同一集群时的资源隔离。

### 4.3 URL 自动推断

根据资源类型不同，从 spec 中提取 URL：

**Ingress → `getUrlFromIngress()`**

```
schema = spec.tls ? "https" : "http"
host   = spec.rules[0].host
path   = spec.rules[0].http.paths[0].path
url    = `${schema}://${host}${path}`
```

**HTTPRoute → `getUrlFromHttpRoute()`**

```
1. spec.hostnames 必须存在
2. path 类型不能是 RegularExpression
3. 通过 parentRefs[0] 关联的 Gateway 资源查询监听器协议（http/https）
   → 直接使用 parentRef.namespace 作为 Gateway 所在命名空间
     （⚠️ 命名空间边界：parentRef 未写 namespace 时，
      不会自动改用 HTTPRoute 自身的命名空间，
      会将 undefined 传给 API 导致查询失败，
      catch 后协议回退为 "http"）
   → CustomObjectsApi.getNamespacedCustomObject() 获取 Gateway
   → 匹配 sectionName 或取第一个 listener 的 protocol
4. url = `${schema}://${hostnames[0]}${rules[0].matches[0].path.value}`
5. Gateway 查询失败时，协议回退为 "http"
```

用户可通过 `gethomepage.dev/href` 注解手动覆盖自动推断的 URL。

### 4.4 服务对象构建

**`constructedServiceFromResource(resource)`** — 将 K8s 资源映射为 Homepage 服务对象：

| 服务字段 | 来源（注解优先，回退到资源属性） |
|---------|--------------------------------|
| `app` | `gethomepage.dev/app` 或 `metadata.name` |
| `namespace` | `metadata.namespace` |
| `href` | `gethomepage.dev/href` 或自动推断 URL |
| `name` | `gethomepage.dev/name` 或 `metadata.name` |
| `group` | `gethomepage.dev/group` 或默认 `"Kubernetes"` |
| `weight` | `gethomepage.dev/weight` 或 `"0"` |
| `icon` | `gethomepage.dev/icon` |
| `description` | `gethomepage.dev/description` |
| `external` | `gethomepage.dev/external`（字符串 "true" 解析为布尔 true） |
| `podSelector` | `gethomepage.dev/pod-selector` |
| `ping` | `gethomepage.dev/ping` |
| `siteMonitor` | `gethomepage.dev/siteMonitor` |
| `statusStyle` | `gethomepage.dev/statusStyle` |
| `widget.*` | 所有 `gethomepage.dev/widget.<xxx>` 注解通过 shvl.set() 嵌套写入 |

最后对整个对象执行 `substituteEnvironmentVars()`，支持 `{{HOMEPAGE_VAR_*}}` 变量替换（JSON 序列化 → 替换 → 反序列化）。

---

## 五、服务聚合层

### 5.1 K8s 服务分组

文件：`src/utils/config/service-helpers.js` 中的 `servicesFromKubernetes()`

完整流程与空值/异常处理：

```
1. getSettings() 读取 instanceName
2. checkAndCopyConfig("kubernetes.yaml") 确保配置存在
3. try {
     getKubeConfig() → 为 null（disabled 模式）直接 return []
     Promise.all([ingress, traefik, httpRoute]) 并行采集
     resources = [...三类结果展开]
       → resources 为假值时 return []（代码注释标注此分支实际不可达）
     filter(isDiscoverable) → 过滤启用的资源
     map(constructedServiceFromResource) → 构建服务对象
     reduce() 按 group 字段分组
   } catch (e) {
     logger.error(e)
     throw e   // 异常向上抛出，由外层 servicesResponse 捕获
   }
```

### 5.2 三方服务合并

文件：`src/utils/config/api-response.js` 中的 `servicesResponse()`

**四个独立的 try-catch，各自失败互不影响：**

```javascript
// 1. Docker 服务
try {
  discoveredDockerServices = cleanServiceGroups(await servicesFromDocker())
} catch (e) {
  console.error("Failed to discover services, please check docker.yaml...")
  discoveredDockerServices = []   // 异常兜底为空数组
}

// 2. K8s 服务
try {
  discoveredKubernetesServices = cleanServiceGroups(await servicesFromKubernetes())
} catch (e) {
  console.error("Failed to discover services, please check kubernetes.yaml...")
  discoveredKubernetesServices = []   // 异常兜底为空数组
}

// 3. services.yaml 配置
try {
  configuredServices = cleanServiceGroups(await servicesFromConfig())
} catch (e) {
  console.error("Failed to load services.yaml, please check for errors")
  configuredServices = []   // 异常兜底为空数组
}

// 4. settings.yaml 配置
try {
  initialSettings = await getSettings()
} catch (e) {
  console.error("Failed to load settings.yaml...")
  initialSettings = {}   // 异常兜底为空对象
}
```

**合并策略：**

1. 取三者 group name 的并集（`mergedGroupsNames`）
2. 对每个 group，按 `docker → k8s → yaml` 顺序拼接 services，`.filter(Boolean)` 去除假值，然后按 `weight`（升序）→ `name`（字典序）排序
3. `groups` 子组只取 `configuredGroup.groups`（即只来自 services.yaml，K8s 和 Docker 发现的服务不支持子组）
4. 根据 `settings.yaml` 的 `layout` 配置对 group 排序
5. 支持嵌套 group（通过 `parent` 字段递归合并）
6. `pruneEmptyGroups()` 移除无服务也无子组的空组（递归处理）

### 5.3 数据清洗

**`cleanServiceGroups()`** 对最终服务对象执行：

- `weight` 字符串转数字，失败置 0
- `widget` 单例转为 `widgets` 数组
- Widget 配置白名单过滤（只保留前端需要的字段，约 100+ 个 widget 专属字段）
- `fields` / `highlight` 等 JSON 字符串字段反序列化
- `kubernetes` type widget 提取 `namespace`、`app`、`podSelector`

---

## 六、前端渲染层

### 6.1 数据获取入口

文件：`src/pages/index.jsx`

**服务端预取（SSR）** — `getStaticProps()`：

```javascript
try {
  const services  = await servicesResponse()
  const bookmarks = await bookmarksResponse()
  const widgets   = await widgetsResponse()
  // 注入 SWR fallback，首屏无需二次请求
  fallback: {
    "/api/services": services,
    "/api/bookmarks": bookmarks,
    "/api/widgets": widgets,
    "/api/hash": false,
  }
} catch (e) {
  // 整体异常兜底，全部返回空数组
  fallback: {
    "/api/services": [],
    "/api/bookmarks": [],
    "/api/widgets": [],
    "/api/hash": false,
  }
}
```

**客户端刷新** — `Home` 组件内：

```javascript
const { data: services } = useSWR("/api/services")
```

API 路由：`src/pages/api/services/index.js` 直接转发 `servicesResponse()`。

### 6.2 分组渲染

文件：`src/components/services/group.jsx`

`ServicesGroup` 组件：

- 根据 `layout.style` 决定 `row` 布局还是默认 `grid` 响应式列布局（1/2, 1/3, 1/4, 1/N）
- 支持可折叠（Disclosure + Transition 动画）、子组递归渲染
- 可选 header（图标 + 组名 + 折叠箭头）

### 6.3 服务卡片渲染

文件：`src/components/services/item.jsx`

**核心渲染逻辑（K8s 相关部分）：**

```
┌─ service 带 app 字段 → 识别为 Kubernetes 服务
│
├── 右上角状态指示器：
│    条件：service.app && !service.external
│    └─ KubernetesStatus 组件（圆点或文字标签）
│       └─ useSWR(/api/kubernetes/status/{ns}/{app})
│
└── 底部展开监控区域：
     条件：service.app  （注意：不判断 external！）
     └─ 显示条件：showStats || statsOpen
        ├─ showStats：service.showStats === false ? false : settings.showStats
        │   （来自 settings.yaml 全局配置，或 services.yaml 中该服务的单独配置；
        │    Kubernetes 注解不支持设置 showStats）
        └─ statsOpen：点击状态指示器按钮切换
           （但 external 服务没有指示器按钮，只能靠 showStats 自动展开）
```

**关键细节 — external 对 K8s 服务的影响：**

| 特性 | 条件 | external=true 时 |
|-----|------|-----------------|
| 状态指示器按钮 | `service.app && !service.external` | ❌ 不显示 |
| 底部展开区域 DOM | `service.app` | ✅ 仍然存在 |
| 通过点击展开 | 依赖指示器按钮的 onClick | ❌ 无法点击展开 |
| 通过 showStats 默认展开 | 依赖 settings.showStats 或 service.showStats | ✅ 可以 |
| Kubernetes 组件渲染 | `showStats \|\| statsOpen` | ✅ 若展开则正常显示 |

简言之：`external=true` 的 K8s 服务隐藏了状态指示器按钮，但监控面板组件本身仍在，只能通过 settings.yaml 的全局 `showStats` 或 services.yaml 中该服务的 `showStats` 配置自动展开（K8s 注解无法设置 showStats）。

另外，服务卡片还可能带有 `ping`、`siteMonitor` 等状态标签，以及 `widgets` 数组渲染的各类型 Widget。

---

## 七、运行时监控层

Kubernetes 监控分两个维度：**服务级 Pod 监控** 和 **集群级 Node 监控**。

### 7.1 服务级 Pod 状态

API 路由：`src/pages/api/kubernetes/status/[...service].js`

```
请求：GET /api/kubernetes/status/{namespace}/{app}?podSelector=...

1. labelSelector = podSelector || "app.kubernetes.io/name=<appName>"
2. CoreV1Api.listNamespacedPod({ namespace, labelSelector })
   → 失败：500 + { error }
3. pods.length === 0 → 404 + { status: "not found" }
4. 判断 Pod phase：
   - 全部 Running/Succeeded → "running"
   - 部分 Running/Succeeded → "partial"
   - 否则                   → "down"
5. 200 + { status }
```

前端组件：`src/components/services/kubernetes-status.jsx`

- 通过 `useSWR` 定时获取状态
- 支持 `dot` 和文字两种样式
- 状态颜色：running→绿色，partial/down/not found→橙色，error→红色

### 7.2 服务级 Pod 资源统计

API 路由：`src/pages/api/kubernetes/stats/[...service].js`

```
1. 同 status API，先 listNamespacedPod 获取目标 Pod
2. 累加各 container 的 resources.limits.cpu / memory（使用 parseCpu/parseMemory 解析单位）
3. Metrics.getPodMetrics(namespace) 获取 metrics.k8s.io 数据
   → 404 不报错（可能 metrics 尚未填充），返回 null
4. 按 Pod name 过滤后累加 containers 的 usage.cpu / memory
5. 计算：
   stats.cpuUsage = cpuLimit ? 100 * (cpu / cpuLimit) : 0
   stats.memUsage = memLimit ? 100 * (mem / memLimit) : 0
6. 200 + { stats: { cpu, mem, cpuLimit, memLimit, cpuUsage, memUsage } }
```

工具函数 `src/utils/kubernetes/utils.js`：

- `parseCpu()`：支持 `n`(纳核) / `u`(微核) / `m`(毫核) / 纯数字
- `parseMemory()`：支持 `Ki`/`K`/`Mi`/`M`/`Gi`/`G`，注意十进制与二进制的区别（Ki=1000, K=1024）

### 7.3 前端组件（服务级）

文件：`src/widgets/kubernetes/component.jsx`

```javascript
useSWR(`/api/kubernetes/status/${namespace}/${app}?podSelector=...`)
useSWR(`/api/kubernetes/stats/${namespace}/${app}?podSelector=...`)
```

渲染两块数据：
- **CPU**：有 limit 时显示百分比（带高亮），否则显示绝对值（4 位小数）
- **内存**：显示字节数（自动带单位，带高亮）

状态异常（非 running / partial）时，显示 "offline"。

### 7.4 集群级 Widget

API 路由：`src/pages/api/widgets/kubernetes.js`

信息栏 Widget，在顶部展示整个集群的状态：

```
1. CoreV1Api.listNode()
   - capacity.cpu / capacity.memory 累加为 total
   - conditions[type=Ready].status 判断节点健康

2. Metrics.getNodeMetrics()
   - 每个节点的 usage.cpu/memory
   - 计算 load / percent / used / free
   - 失败：500 + { error }，提示需安装 metrics-server

3. 200 + { cluster: {...}, nodes: [{name, ready, cpu:{...}, memory:{...}}, ...] }
```

前端组件：`src/components/widgets/kubernetes/kubernetes.jsx`

每 1500ms 刷新一次，支持分别控制 `cluster.show` 和 `nodes.show`。

---

## 八、完整调用链路图

```
用户访问首页
    │
    ▼
getStaticProps() [src/pages/index.jsx]
    ├── servicesResponse()        [src/utils/config/api-response.js]
    │     ├── try { servicesFromDocker() } catch → []
    │     ├── try { servicesFromKubernetes() } catch → []
    │     │     └── servicesFromKubernetes()  [src/utils/config/service-helpers.js]
    │     │           ├── getSettings() → instanceName
    │     │           ├── getKubeConfig()
    │     │           │     └── kubernetes.yaml mode: cluster|default|disabled
    │     │           ├── kubernetes.listIngress()
    │     │           │     └── NetworkingV1Api.listIngressForAllNamespaces()
    │     │           │       → 失败 → null → .items ?? []
    │     │           ├── kubernetes.listTraefikIngress()
    │     │           │     ├── checkCRD(traefik.containo.us)
    │     │           │     ├── checkCRD(traefik.io)
    │     │           │     └── CustomObjectsApi.listClusterCustomObject() ×2
    │     │           │       → 失败 → []
    │     │           ├── kubernetes.listHttpRoute()
    │     │           │     ├── CoreV1Api.listNamespace()
    │     │           │     └── CustomObjectsApi.listNamespacedCustomObject() ×N
    │     │           │       → 失败 → null → filter(Boolean) 过滤
    │     │           ├── filter(isDiscoverable)
    │     │           │     └── gethomepage.dev/enabled === "true" + instance 匹配
    │     │           ├── map(constructedServiceFromResource)
    │     │           │     ├── getUrlFromIngress / getUrlFromHttpRoute
    │     │           │     ├── 注解解析 → 服务对象
    │     │           │     ├── shvl.set() 写入 widget 嵌套字段
    │     │           │     └── substituteEnvironmentVars() 变量替换
    │     │           └── reduce() 按 group 分组
    │     │           └── catch → logger.error + throw
    │     ├── try { servicesFromConfig() } catch → []
    │     ├── try { getSettings() } catch → {}
    │     ├── 三方合并 + 排序 + layout 排序
    │     │     └── compareServices(): weight 升序 → name 字典序
    │     ├── pruneEmptyGroups()
    │     └── cleanServiceGroups()
    └── SWR fallback → props
    │
    ▼
Home 组件 [src/pages/index.jsx]
    └── useSWR("/api/services")
         │
         ▼
ServicesGroup [src/components/services/group.jsx]
    └── Services List
         └── Item [src/components/services/item.jsx]
              ├── service.app && !service.external → KubernetesStatus 按钮
              │    └── useSWR(/api/kubernetes/status/{ns}/{app})
              │         └── CoreV1Api.listNamespacedPod → phase 判断
              └── service.app → 底部展开区域（点击或 showStats 显示）
                   └── Kubernetes component [src/widgets/kubernetes/component.jsx]
                        ├── useSWR(/api/kubernetes/status/{ns}/{app})
                        └── useSWR(/api/kubernetes/stats/{ns}/{app})
                             ├── CoreV1Api.listNamespacedPod → limits 累加
                             └── Metrics.getPodMetrics → usage 累加

另外（信息栏 Widget，独立于服务）：
src/components/widgets/kubernetes/kubernetes.jsx
    └── useSWR(/api/widgets/kubernetes, refreshInterval: 1500)
         ├── CoreV1Api.listNode()
         └── Metrics.getNodeMetrics()
```

---

## 九、关键注解速查表

在 Kubernetes Ingress / HTTPRoute / IngressRoute 资源上使用以下注解控制 Homepage 行为：

| 注解 | 必填 | 说明 |
|-----|-----|------|
| `gethomepage.dev/enabled` | ✅ | 必须为 `"true"` 才会被发现 |
| `gethomepage.dev/name` | | 服务显示名称，默认取资源名 |
| `gethomepage.dev/group` | | 所属分组，默认 `"Kubernetes"` |
| `gethomepage.dev/href` | Traefik 必填 | 服务 URL，Ingress/HTTPRoute 可自动推断 |
| `gethomepage.dev/app` | | 用于关联 Pod（`app.kubernetes.io/name` 标签值），默认取资源名 |
| `gethomepage.dev/description` | | 服务描述 |
| `gethomepage.dev/icon` | | 图标 |
| `gethomepage.dev/weight` | | 排序权重（数字越小越靠前），默认 0 |
| `gethomepage.dev/ping` | | 启用 ICMP ping 检测 |
| `gethomepage.dev/siteMonitor` | | 启用 HTTP 站点监控 |
| `gethomepage.dev/pod-selector` | | 自定义 Pod label selector，覆盖默认 `app.kubernetes.io/name=<app>` |
| `gethomepage.dev/external` | | `"true"` 时隐藏 K8s 状态指示器按钮（监控面板仍可通过 settings.yaml/services.yaml 的 `showStats` 自动展开） |
| `gethomepage.dev/statusStyle` | | 状态指示器样式：`dot` 或文字 |
| `gethomepage.dev/instance` | | 多实例隔离，匹配 `settings.yaml` 的 `instanceName` |
| `gethomepage.dev/instance.<name>` | | 多实例隔离的另一种写法 |
| `gethomepage.dev/widget.<type>.<field>` | | 为服务附加 Widget，如 `widget.type=kubernetes` |
