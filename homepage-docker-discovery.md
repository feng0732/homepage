# Homepage 服务发现与 Docker 集成代码分析

本文档基于源代码事实，深度剖析 Homepage 项目中「服务探测 → 数据整理 → 页面展示」的完整数据链路。
所有结论均可在对应代码位置直接验证。

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
│    3. 按 weight + name 排序                                        │
│    4. 应用 settings.yaml 中 layout 的顺序                         │
│    5. pruneEmptyGroups() 剪枝空分组                                │
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
│    → /api/services 路由再次调用 servicesResponse()                │
│                                                                    │
│  实时状态（按需加载）：                                              │
│    Docker Status:   /api/docker/status/[container]/[server]      │
│    Docker Stats:    /api/docker/stats/[container]/[server]       │
│    K8s Status:      /api/kubernetes/status/[ns]/[app]             │
│    K8s Stats:       /api/kubernetes/stats/[ns]/[app]              │
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

#### docker.yaml 配置结构

```yaml
# 示例 docker.yaml
my-docker:           # 服务器名（自定义）
  socket: /var/run/docker.sock   # 方式一：socket
  swarm: false                   # 是否 Swarm 模式

my-remote-docker:    # 方式二：远程主机
  host: 192.168.1.100
  port: 2376
  tls:                        # TLS 认证（可选）
    caFile: ca.pem
    certFile: cert.pem
    keyFile: key.pem
  protocol: https
  headers:                    # 自定义 header（可选）
    Authorization: Bearer xxx
```

**返回值结构：**
```javascript
{
  conn: { socketPath?, host?, port?, ca?, cert?, key?, protocol?, headers? },
  swarm: true | false   // 仅在配置了 swarm 时返回
}
```

> **代码事实**：当 server 参数为空时，`getDockerArguments()` 返回的是默认参数对象（没有 `conn` 外层包装），但在 API 端点中使用时会取 `.conn`。需要注意 `getDefaultDockerArgs()` 返回的是扁平对象，而配置文件解析返回的是 `{ conn, swarm }` 结构。

---

### 1.2 Docker 容器标签发现（精确边界）

**核心文件：** [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js)

**核心函数：** [servicesFromDocker()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L63-L170)

#### 执行流程

```
Step 1: 读取 docker.yaml → 获取所有服务器配置
   │
   ▼
Step 2: 遍历每个服务器，并行调用 dockerode
   │  - 普通模式：docker.listContainers({ all: true })
   │  - Swarm 模式：docker.listServices({ all: true })
   │  - 如果返回不是数组（连接失败可能返回 Buffer）→ 返回空数组
   │
   ▼
Step 3: 对每个容器/服务，扫描所有 Labels
   │  对每个 Label 键值对执行：
   │
   │  3.1 前缀检查：label.startsWith("homepage.")
   │    → 不匹配 → 跳过
   │
   │  3.2 去掉前缀：value = label.replace("homepage.", "")
   │
   │  3.3 Instance 过滤（仅当 settings.instanceName 存在时有效）：
   │    │
   │    ├─ value 以 "instance.{instanceName}." 开头
   │    │    → 去掉该前缀，保留后续路径
   │    │
   │    ├─ value 以 "instance." 开头（但实例名不匹配）
   │    │    → 跳过当前 Label（return）
   │    │
   │    └─ 其他情况 → 保留原值
   │
   │  3.4 首次匹配时初始化 constructedService：
   │       { container, server, weight: 0, type: "service" }
   │
   │  3.5 值处理：
   │    - substituteEnvironmentVars() 替换 {{HOMEPAGE_VAR_*}} / {{HOMEPAGE_FILE_*}}
   │    - 如果路径是 "widget.version" 或匹配 /^widgets\[\d+\]\.version$/
   │      → parseVersionForUrl() 处理版本号
   │
   │  3.6 shvl.set(constructedService, valuePath, substitutedVal)
   │       → 将点号路径转为嵌套对象
   │
   ▼
Step 4: 有效性校验
   │  条件：constructedService 存在 且 name 存在 且 group 存在
   │  → 不满足 → log error 并返回 null（该容器被过滤掉）
   │
   ▼
Step 5: 按 group 分组，输出 mappedServiceGroups
       → [{ name: groupName, services: [...] }]
```

#### 标签发现的边界条件（代码事实）

| 条件 | 结果 | 代码位置 |
|------|------|---------|
| Label 不以 `homepage.` 开头 | 跳过 | [service-helpers.js L99](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L99) |
| `homepage.instance.xxx` 但 instanceName 不匹配 | 跳过该 Label | [service-helpers.js L103-L104](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L103-L104) |
| `homepage.instance.{name}.xxx` 且 instanceName 匹配 | 保留，去掉 `instance.{name}.` 前缀 | [service-helpers.js L101-L102](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L101-L102) |
| 没有 `homepage.name` 或没有 `homepage.group` | 整个容器被丢弃（log error） | [service-helpers.js L123-L131](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L123-L131) |
| 容器连接失败 / 返回非数组 | 该服务器返回空 services | [service-helpers.js L89-L91](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L89-L91) |
| Swarm 模式下 Label 位置 | 从 `Spec.Labels` 取，而非顶层 Labels | [service-helpers.js L95](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L95) |
| 容器名的 `/` 前缀 | 自动去掉（replace(/^\//, "")） | [service-helpers.js L109](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L109) |

#### 自动注入的字段（初始 constructedService）

```javascript
{
  container: containerName.replace(/^\//, ""),   // 容器名（去掉开头 /）
  server: serverName,     // 来自 docker.yaml 的服务器名
  weight: 0,              // 默认权重
  type: "service"         // 标记类型
}
```

#### shvl 路径转换示例

使用 [shvl.set()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js#L38-L64) 将点号+方括号路径转为嵌套对象：

| Label 键（去掉 homepage. 后） | 生成的对象结构 |
|-----------------------------|--------------|
| `name` | `{ name: "value" }` |
| `group` | `{ group: "value" }` |
| `widget.type` | `{ widget: { type: "value" } }` |
| `widgets[0].type` | `{ widgets: [{ type: "value" }] }` |
| `widgets[1].version` | `{ widgets: [undefined, { version: "value" }] }` |

> **安全保护**：shvl.set 会拦截 `__proto__` / `constructor` / `prototype` 等危险键名，防止原型污染。
> 见 [shvl.js L46](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js#L46)

---

### 1.3 Kubernetes 资源注解发现

**核心文件：**
- 入口：[service-helpers.js → servicesFromKubernetes()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L172-L229)
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

> **代码事实**：Traefik IngressRoute 有两个 API Group 都尝试：`traefik.containo.us` 和 `traefik.io`。
> 见 [traefik-list.js L18-L58](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L18-L58)

#### 发现前置条件（isDiscoverable）

**函数：** [isDiscoverable(resource, instanceName)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js#L79-L87)

资源必须同时满足：

1. **存在 annotations**：`resource.metadata.annotations` 不为空
2. **启用标记**：`gethomepage.dev/enabled === "true"`（字符串 "true"，不是布尔值）
3. **实例匹配**（三选一）：
   - 没有 `gethomepage.dev/instance` 注解 → 通过
   - `gethomepage.dev/instance === instanceName` → 通过
   - 存在 `gethomepage.dev/instance.{instanceName}` 注解键 → 通过

> **注意**：K8s 的注解前缀是 `gethomepage.dev/`（斜杠），与 Docker 的 `homepage.`（点号）不同。
> 常量定义：[kubernetes.js L58-L59](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/kubernetes.js#L58-L59)

#### 服务对象构造

**函数：** [constructedServiceFromResource(resource)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js#L89-L137)

**基础字段（有默认值）：**

| 字段 | 注解键 | 默认值 |
|------|-------|-------|
| `app` | `gethomepage.dev/app` | `resource.metadata.name` |
| `namespace` | - | `resource.metadata.namespace` |
| `href` | `gethomepage.dev/href` | 自动从资源规则推导 URL |
| `name` | `gethomepage.dev/name` | `resource.metadata.name` |
| `group` | `gethomepage.dev/group` | `"Kubernetes"` |
| `weight` | `gethomepage.dev/weight` | `"0"` |
| `icon` | `gethomepage.dev/icon` | `""` |
| `description` | `gethomepage.dev/description` | `""` |
| `external` | `gethomepage.dev/external` | `false` |
| `type` | - | `"service"` |

**额外可选字段（存在才注入）：**
- `podSelector` ← `gethomepage.dev/pod-selector`
- `ping` ← `gethomepage.dev/ping`
- `siteMonitor` ← `gethomepage.dev/siteMonitor`
- `statusStyle` ← `gethomepage.dev/statusStyle`

**Widget 字段（通过 gethomepage.dev/widget.* 前缀批量处理）：**
```javascript
Object.keys(annotations).forEach((annotation) => {
  if (annotation.startsWith(ANNOTATION_WIDGET_BASE)) {  // gethomepage.dev/widget.
    shvl.set(constructedService, annotation.replace("gethomepage.dev/", ""), value);
  }
});
```

最后对整个对象做 `substituteEnvironmentVars()` 环境变量替换（JSON 序列化后再替换）。

#### URL 自动推导规则

| 资源类型 | 推导逻辑 |
|---------|---------|
| **Ingress** | `http(s)://{host}{path}`，有 TLS 则 https |
| **HTTPRoute** | `{schema}://{hostname}{path}`，schema 从关联 Gateway 的 listener.protocol 获取 |
| **Traefik** | **不自动推导**，必须有 `gethomepage.dev/href` 注解才会被纳入（见 [traefik-list.js L61-L63](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js#L61-L63)） |

---

### 1.4 静态配置文件来源

**函数：** [servicesFromConfig()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L53-L61)

- 从 `services.yaml` 读取
- 通过 `parseServicesToGroups()` 将 YAML 结构转为 Group 数组
- 支持嵌套 group（子分组）
- 每个 service 自动赋默认 weight：`(index + 1) * 100`

---

## 二、数据整理层（Data Aggregation）

### 2.1 服务合并主入口

**核心文件：** [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js)

**核心函数：** [servicesResponse()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L158-L256)

#### 合并流程（精确步骤）

```
Step 1: 加载三来源数据（各自 try/catch，互不影响）
   │
   ├─ discoveredDockerServices
   │     = cleanServiceGroups(await servicesFromDocker())
   │     失败 → []
   │
   ├─ discoveredKubernetesServices
   │     = cleanServiceGroups(await servicesFromKubernetes())
   │     失败 → []
   │
   └─ configuredServices
         = cleanServiceGroups(await servicesFromConfig())
         失败 → []

Step 2: 获取所有 group 名的并集
   mergedGroupsNames = [...new Set([...docker, ...k8s, ...config].map(g => g.name))]

Step 3: 如果有 settings.layout
   → 将 layout 中定义的空 group 合并进 configuredServices
   → 作用：即使 group 里没有服务，也能在页面上显示分组标题

Step 4: 遍历每个 groupName，执行合并
   │
   ├─ 分别从三来源找同名 group（找不到则用 { services: [] } 替代）
   │
   ├─ 合并 services：
   │    [...docker.services, ...k8s.services, ...config.services]
   │      .filter(Boolean)
   │      .sort(compareServices)   // 先 weight 升序，再 name 字典序
   │
   ├─ 合并 groups（嵌套子组）：
   │    [...configuredGroup.groups]   // 仅来自 config，Docker/K8s 没有嵌套
   │
   └─ 排序：
        ├─ group 名在 layout 中 → 按 layout 索引放 sortedGroups
        ├─ group 有 parent → 合并到父 group 的 services
        └─ 其他 → 放 unsortedGroups

Step 5: 结果 = sortedGroups（去空值） + unsortedGroups

Step 6: pruneEmptyGroups() 剪枝
       → 移除 services 和 groups 都为空的 group（递归）
```

#### 关于嵌套 groups 的关键事实

- **只有 services.yaml 静态配置支持嵌套 group**（子组）
- Docker 和 Kubernetes 发现的服务都是「扁平的」，只有一层 group
- 嵌套合并通过 `mergeSubgroups()` + `findGroupByName(..., parent)` 完成
- 见 [api-response.js L99-L121](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L99-L121)

#### 排序规则

**函数：** [compareServices()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js#L19-L25)

```javascript
function compareServices(service1, service2) {
  const comp = service1.weight - service2.weight;
  if (comp !== 0) return comp;      // 先按 weight 升序
  return service1.name.localeCompare(service2.name);  // 再按名称字典序
}
```

---

### 2.2 数据标准化（cleanServiceGroups）

**函数：** [cleanServiceGroups()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js#L231-L708)

这是数据从服务端到前端前的最后一道「清洗」，核心目的有两个：
1. **类型统一**：将 Label/Annotation 中的字符串值转为正确类型
2. **安全过滤**：Widget 字段白名单，只向前端暴露必要字段

#### 标准化处理项

| 处理项 | 逻辑 |
|-------|------|
| `weight` 类型转换 | string → parseInt，失败则设为 0 |
| `showStats` 类型转换 | string → JSON.parse（"true" → true） |
| `widget` → `widgets` 归一化 | 单个 widget 对象 → 推入 widgets 数组 |
| `widgets` 数组初始化 | 没有 widgets 则设为空数组 `[]` |
| Widget 字段白名单 | 仅保留明确列出的字段（约 90 个字段） |
| `fields` 类型转换 | string → JSON.parse（失败则 null） |
| `highlight` 类型转换 | string → JSON.parse |
| 各 widget 特有字段类型转换 | 如 enableQueue、bitratePrecision 等 |

#### Widget 白名单机制（代码事实）

白名单是**解构赋值**方式实现的，解构出来的变量才会被重新组装回 widget 对象：

```javascript
const {
  fields, hideErrors, highlight, type,     // 通用
  env,              // arcane
  repositoryId,     // azuredevops
  systemId,         // beszel
  container, server,  // docker
  ...               // 约 40+ 种 widget 的字段
} = widgetData;

// 组装新 widget 对象
const widget = {
  type, fields: fieldsList || null, hide_errors: hideErrors || false,
  service_name: service.name, service_group: serviceGroup.name, index,
  // ... 按 type 特化注入
};
```

> **安全意义**：即使 Label 中注入了恶意字段，也会在白名单这一步被过滤掉，不会传到前端。

---

## 三、页面展示层（Presentation）

### 3.1 首屏数据链路（SSR + SWR Fallback）

**核心文件：** [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx)

#### 服务端：getStaticProps()

**位置：** [getStaticProps()](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L55-L95)

```javascript
export async function getStaticProps() {
  const services = await servicesResponse();   // 服务端直接调用，不走 HTTP
  const bookmarks = await bookmarksResponse();
  const widgets = await widgetsResponse();

  return {
    props: {
      initialSettings: settings,
      fallback: {
        "/api/services": services,     // SWR fallback 数据
        "/api/bookmarks": bookmarks,
        "/api/widgets": widgets,
        "/api/hash": false,
      },
      ...i18nTranslations,
    }
  };
}
```

**关键点：**
- 这是 **Next.js Static Site Generation** (getStaticProps)
- 直接在 Node 环境调用 `servicesResponse()`，**不经过 API 路由**
- 结果作为 SWR 的 fallback 数据注入页面

#### 客户端：SWR 消费

**SWRConfig 注入：** [index.jsx L186](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L186)

```jsx
<SWRConfig value={{ fallback, fetcher: (resource, init) => fetch(resource, init).then(res => res.json()) }}>
  <Home initialSettings={initialSettings} />
</SWRConfig>
```

**Home 组件中使用：** [index.jsx L226-L228](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx#L226-L228)

```javascript
const { data: services } = useSWR("/api/services");
const { data: bookmarks } = useSWR("/api/bookmarks");
const { data: widgets } = useSWR("/api/widgets");
```

**数据优先级：**
1. 优先使用 `fallback` 中的数据（首屏无请求、无闪烁）
2. SWR 后台会自动重新验证（发送请求到 `/api/services`）
3. 如果数据有变化则更新视图

#### API 路由：/api/services

**文件：** [pages/api/services/index.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/services/index.js)

```javascript
import { servicesResponse } from "utils/config/api-response";

export default async function handler(req, res) {
  res.send(await servicesResponse());
}
```

> **代码事实**：API 路由和 getStaticProps 调用的是同一个 `servicesResponse()` 函数。
> 区别在于：getStaticProps 在构建时/ISR 重新生成时执行，API 路由在客户端重新验证时按需执行。

---

### 3.2 组件层级结构

```
pages/index.jsx (Home 组件)
  │
  └── 遍历 services 数组
      │
      └── ServicesGroup ([group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx))
          │  功能：可折叠分组，带标题栏和展开/收起动画
          │
          ├── 标题栏（可选 icon + group name + 折叠箭头）
          │
          └── List ([list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/list.jsx))
              │  功能：网格/列布局
              │
              └── Item ([item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/item.jsx))
                  │  功能：单个服务卡片
                  │
                  ├── ResolvedIcon — 服务图标
                  ├── 服务名称 & 描述
                  │
                  ├── 状态标签区（右上角）
                  │   ├── Ping 状态 (ping.jsx)
                  │   ├── SiteMonitor (site-monitor.jsx)
                  │   ├── Status [Docker] (status.jsx)    ← service.container 存在时显示
                  │   ├── KubernetesStatus (kubernetes-status.jsx)  ← service.app 存在时显示
                  │   └── ProxmoxStatus (proxmox-status.jsx)
                  │
                  └── 展开的 Stats 区域（点击状态标签时展开/收起）
                      │
                      ├── Docker Component ([widgets/docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx))
                      │   ├── /api/docker/status/[container]/[server]
                      │   └── /api/docker/stats/[container]/[server]
                      │       └── Block × 4（CPU / Mem / RX / TX）
                      │
                      ├── Kubernetes Component ([widgets/kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/kubernetes/component.jsx))
                      │   ├── /api/kubernetes/status/[ns]/[app]
                      │   └── /api/kubernetes/stats/[ns]/[app]
                      │       └── Block × 2（CPU / Mem）
                      │
                      └── 其他 widget...
```

---

### 3.3 Docker 状态徽章（Status Badge）

**文件：** [components/services/status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx)

**数据来源：**
```javascript
const { data, error } = useSWR(`/api/docker/status/${service.container}/${service.server || ""}`);
```

#### 状态映射表（代码精确对应）

| 返回状态 | 展示文本 | 颜色 class | 代码位置 |
|---------|---------|-----------|---------|
| `error` | `docker.error` | `text-rose-500/80` | [status.jsx L13-L15](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L13-L15) |
| `running` + 无 health | `docker.running` | `text-emerald-500/80` | [status.jsx L17-L21](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L17-L21) |
| `running` + `healthy` | `docker.healthy` | `text-emerald-500/80` | [status.jsx L22-L24](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L22-L24) |
| `running` + `starting` | `docker.starting` | `text-blue-500/80` | [status.jsx L25-L28](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L25-L28) |
| `running` + `unhealthy` | `docker.unhealthy` | `text-orange-400/50` | [status.jsx L30-L33](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L30-L33) |
| `not found` | `docker.not_found` | `text-orange-400/50` | [status.jsx L37-L38](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L37-L38) |
| `exited` | `docker.exited` | `text-orange-400/50` | [status.jsx L39](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L39) |
| `partial x/y` | `docker.partial` + x/y | `text-orange-400/50` | [status.jsx L40](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L40) |
| 默认（未知） | `docker.unknown` | `text-black/20` | [status.jsx L9](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx#L9) |

**显示样式：**
- **label 模式**（默认）：显示文字标签，`text-[8px] font-bold uppercase`
- **dot 模式**：显示圆点，将 `text-` 替换为 `bg-`，移除透明度

---

### 3.4 Docker 统计组件

**文件：** [widgets/docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx)

**并行请求两个 API：**
```javascript
const { data: statusData } = useSWR(`/api/docker/status/${widget.container}/${widget.server || ""}`);
const { data: statsData } = useSWR(`/api/docker/stats/${widget.container}/${widget.server || ""}`);
```

**展示逻辑分支：**

| 条件 | 显示内容 |
|------|---------|
| 任意错误 | `Container` 组件 + error 提示 |
| 状态存在但不是 running / partial | 显示 `docker.offline` |
| 数据加载中 | 显示 4 个 Block 的骨架（只有 label，无 value） |
| 数据正常 | 显示 CPU / Mem / RX / TX 四个指标 |

#### 指标计算（stats-helpers.js）

**文件：** [stats-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js)

| 指标 | 算法 | 代码行 |
|-----|------|-------|
| **CPU 使用率** | `(cpuDelta / systemDelta) * online_cpus * 100.0` | [L1-L11](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js#L1-L11) |
| **内存使用** | `usage - total_inactive_file`（参考 Docker CLI 算法） | [L13-L18](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js#L13-L18) |
| **网络 RX/TX** | 遍历所有网卡累加 rx_bytes / tx_bytes | [L20-L33](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js#L20-L33) |

> **内存算法来源**：注释标明参考 docker/cli 的 stats_helpers.go L239。
> 兼容两种 stats 格式：新格式 `total_inactive_file` 和旧格式 `stats.inactive_file`。

---

### 3.5 API 端点详解

#### 3.5.1 Docker 状态 API

**文件：** [pages/api/docker/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/status/[...service].js)

**路由参数解析：**
```javascript
const { service } = req.query;
const [containerName, containerServer] = service;  // catch-all 路由，数组形式
```

**执行流程：**

```
1. getDockerArguments(containerServer) → 获取连接参数
2. new Docker(dockerArgs.conn) → 建立连接
3. docker.listContainers({ all: true }) → 列出所有容器
4. 验证返回值是数组（不是 Buffer 等异常值）
5. 扁平化所有容器名（去掉开头的 /）
6. 查找 containerName：
   │
   ├─ 找到 → container.inspect()
   │       → 返回 { status: info.State.Status, health: info.State.Health?.Status }
   │
   └─ 未找到 && dockerArgs.swarm → 尝试 Swarm Service 模式
       │
       ├─ docker.getService(name).inspect() → 获取服务信息
       │  失败 → 404 "not found"
       │
       ├─ docker.listTasks({ filters: { service: [name], "desired-state": ["running"] } })
       │
       ├─ Replicated 模式：
       │    tasks.length === replicas → "running x/y"
       │    tasks.length > 0        → "partial x/y"
       │    tasks.length === 0      → 404
       │
       └─ Global 模式：
            优先找本地容器（localContainerIDs 包含的 task）
            → 有容器 → inspect → 返回 status/health
            → 没容器 → 返回 task.Status.State
```

#### 3.5.2 Docker 统计 API

**文件：** [pages/api/docker/stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/stats/[...service].js)

流程与状态 API 类似，核心调用：
```javascript
const stats = await container.stats({ stream: false });  // 一次性获取，非流式
return res.status(200).json({ stats });
```

Swarm 模式下，优先选择本地运行的容器来获取 stats（只有本地节点的容器才能获取 stats）。

---

## 四、完整数据流时序图（代码级）

```
┌───────────────┐
│ Next.js Build │  或 ISR 重新生成
│  (SSR 阶段)   │
└───────┬───────┘
        │ getStaticProps()
        ▼
┌─────────────────────────────────────┐
│ servicesResponse()                  │
│  ├─ servicesFromDocker()            │  ← dockerode → Docker API
│  ├─ servicesFromKubernetes()        │  ← @kubernetes/client-node → K8s API
│  ├─ servicesFromConfig()            │  ← fs.readFile → services.yaml
│  ├─ cleanServiceGroups() × 3        │
│  ├─ 按 group 合并 + 排序            │
│  └─ pruneEmptyGroups()              │
└───────────────┬─────────────────────┘
                │ 注入 SWR fallback
                ▼
┌─────────────────────────────────────┐
│  浏览器 (首屏 HTML)                  │
│  SWR fallback 已填充 → 无请求直出    │
└───────────────┬─────────────────────┘
                │ useSWR("/api/services")
                │ （优先用 fallback，后台 revalidate）
                ▼
┌─────────────────────────────────────┐
│  /api/services (按需触发)            │
│  → 再次调用 servicesResponse()       │
└───────────────┬─────────────────────┘
                ▼
        ServicesGroup 渲染
        └─ List
            └─ Item
                ├─ Status 徽章
                │  └─ useSWR(/api/docker/status)
                │       → [...service].js → dockerode → Docker API
                │
                └─ 点击展开 Stats
                   └─ Docker Component
                      ├─ useSWR(/api/docker/status)
                      └─ useSWR(/api/docker/stats)
                         └─ stats-helpers.js 计算指标
```

---

## 五、关键设计要点（代码事实）

### 5.1 标签/注解驱动的发现

- **Docker**：`homepage.` 前缀 + shvl 路径展开
- **Kubernetes**：`gethomepage.dev/` 前缀 + `enabled=true` 前置条件
- 两者都支持 `instance` 多实例隔离
- 都通过 shvl 支持嵌套对象路径（点号+方括号语法）

### 5.2 安全设计

- **Widget 字段白名单**：`cleanServiceGroups()` 中解构赋值，仅明确列出的字段会传到前端
- **shvl 原型污染防护**：拦截 `__proto__` / `constructor` / `prototype`
- **环境变量替换白名单**：仅 `HOMEPAGE_VAR_` 和 `HOMEPAGE_FILE_` 前缀的变量会被替换

### 5.3 容错设计

- 三个服务来源各自独立 try/catch，一个失败不影响其他
- 单个 Docker 服务器失败 → 该服务器返回空数组，不影响全局
- 容器不存在 → 返回 404 + "not found"，不抛异常
- K8s CRD 不存在 → 静默返回空（checkCRD 先检测）
- 配置文件解析失败 → 优雅降级，控制台输出错误

### 5.4 性能优化

- **SSR 预取 + SWR fallback**：首屏零额外请求
- **按需加载**：stats 数据只有展开卡片时才请求
- **listContainers({ all: true }) 一次拉全量**：批量获取，逐个处理
- **SWR 缓存**：相同 key 的请求共享缓存（如 Status 和 Docker Component 都请求 status API，只发一次）

### 5.5 Swarm 双模支持

- 普通容器模式 → `listContainers` + `container.inspect()/stats()`
- Swarm 服务模式 → `listServices` + 从 `Spec.Labels` 取标签
- Swarm 状态查询区分 Replicated / Global 两种模式
- Swarm 下 stats 优先找本地节点容器（因为只能获取本地容器的 stats）

### 5.6 布局与分组

- `settings.yaml` 的 layout 决定 group 显示顺序
- layout 中定义但 services 中不存在的 group 也会显示（空组）
- 支持嵌套 group（仅 services.yaml 静态配置）
- `pruneEmptyGroups()` 确保最终没有完全空的分组

---

## 六、代码关联速查表

| 功能模块 | 文件路径 | 关键函数/组件 |
|---------|---------|-------------|
| Docker 连接配置 | [docker.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/docker.js) | `getDockerArguments()` |
| Docker 服务发现 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromDocker()` |
| K8s 服务发现 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromKubernetes()` |
| K8s 资源发现判断 | [resource-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js) | `isDiscoverable()` |
| K8s 服务构造 | [resource-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/resource-helpers.js) | `constructedServiceFromResource()` |
| K8s Ingress 列表 | [ingress-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/ingress-list.js) | `listIngress()` |
| K8s Traefik 列表 | [traefik-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/traefik-list.js) | `listTraefikIngress()` |
| K8s HTTPRoute 列表 | [httproute-list.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/kubernetes/httproute-list.js) | `listHttpRoute()` |
| 静态配置加载 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `servicesFromConfig()` |
| 服务合并主入口 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/api-response.js) | `servicesResponse()` |
| 数据标准化 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/service-helpers.js) | `cleanServiceGroups()` |
| shvl 路径工具 | [shvl.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/utils/config/shvl.js) | `get()`, `set()` |
| 页面入口 SSR | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/index.jsx) | `getStaticProps()` |
| Services API 路由 | [services/index.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/services/index.js) | `handler` |
| 服务分组组件 | [group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/group.jsx) | `ServicesGroup` |
| 服务列表组件 | [list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/list.jsx) | `List` |
| 单个服务卡片 | [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/item.jsx) | `Item` |
| Docker 状态徽章 | [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/status.jsx) | `Status` |
| K8s 状态徽章 | [kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/components/services/kubernetes-status.jsx) | `KubernetesStatus` |
| Docker 统计组件 | [component.jsx (docker)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/component.jsx) | `Component` |
| 统计计算辅助 | [stats-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/widgets/docker/stats-helpers.js) | `calculateCPUPercent` 等 |
| Docker 状态 API | [...service].js (status)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/status/[...service].js) | `handler` |
| Docker 统计 API | [...service].js (stats)](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/pages/api/docker/stats/[...service].js) | `handler` |
| Docker 配置骨架 | [docker.yaml](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/skeleton/docker.yaml) | 配置示例 |
| K8s 配置骨架 | [kubernetes.yaml](file:///d:/fz/0601/solo-dogfeeding/code/193-homepage/src/skeleton/kubernetes.yaml) | 配置示例 |
