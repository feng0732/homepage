# Homepage YAML 配置体系与合并规则深度分析

## 一、整体架构概览

Homepage 的 YAML 配置系统采用 **"多文件分治 + 多源拼接 + 排序输出"** 的设计模式。核心特征是：**同名服务不会互相覆盖，而是将来自 Docker、Kubernetes 和 services.yaml 的三者结果按组进行拼接，最后统一按 weight 和 name 排序输出。**

配置体系分为三个主要层次：

1. **配置文件层**：8 个独立 YAML 文件各司其职
2. **来源发现层**：Services 支持 3 种来源（配置文件 / Docker 容器标签 / Kubernetes 资源注解）
3. **运行时层**：通过 Context + SWR 实现前端消费与热更新

核心配置处理链路分布在以下关键模块（仓库相对路径）：

| 模块 | 职责 | 文件位置 |
|------|------|----------|
| 配置基础能力 | 目录管理、文件校验、环境变量替换 | `src/utils/config/config.js` |
| 扁平路径操作 | 点号/数组路径读写（shvl 精简库） | `src/utils/config/shvl.js` |
| API 响应构造 | 多源 **拼接合并**、排序、空组裁剪 | `src/utils/config/api-response.js` |
| Services 解析 | 3 种来源的数据提取与标准化 | `src/utils/config/service-helpers.js` |
| Widgets 解析 | 配置读取与敏感字段清洗 | `src/utils/config/widget-helpers.js` |

---

## 二、配置文件清单与职责

### 2.1 八大配置文件

所有配置文件均位于 `CONF_DIR` 目录，通过环境变量 `HOMEPAGE_CONFIG_DIR` 指定，默认值为 `<cwd>/config`。首次启动时若文件不存在，会从 `src/skeleton/` 复制模板文件。

| 文件名 | 职责 | 是否参与合并 | 读取入口 |
|--------|------|-------------|----------|
| `settings.yaml` | 全局设置（主题、布局、语言、背景、provider 密钥等） | 不合并（单一来源） | `getSettings()` |
| `services.yaml` | 静态服务组定义（手动配置） | **与 Docker/K8s 拼接合并** | `servicesFromConfig()` |
| `bookmarks.yaml` | 书签组定义 | 不合并（单一来源） | `bookmarksResponse()` |
| `widgets.yaml` | 顶部信息小部件定义 | 不合并（单一来源） | `widgetsFromConfig()` |
| `docker.yaml` | Docker 服务器连接参数 | 本身不合并，驱动服务发现 | `getDockerArguments()` |
| `kubernetes.yaml` | Kubernetes 集群模式配置 | 本身不合并，驱动服务发现 | `getKubernetes()` / `getKubeConfig()` |
| `proxmox.yaml` | Proxmox 连接参数 | 不合并 | `getProxmoxConfig()` |
| `custom.css` / `custom.js` | 用户自定义样式与脚本（非 YAML） | 不合并 | `/api/config/[path]` |

模板文件位于 `src/skeleton/` 目录下，首次启动时自动复制。

### 2.2 配置文件自动初始化机制

`checkAndCopyConfig(config)` 函数（位于 `src/utils/config/config.js`）负责配置文件的生命周期管理：

**执行流程**：
1. 若 `CONF_DIR` 不存在，递归创建（`mkdirSync(..., { recursive: true })`）
2. 检查目标文件是否存在，不存在则从 `src/skeleton/<config>` 复制
3. 若文件存在，调用 `yaml.load()` 做语法校验
4. 返回值语义：
   - `true`：文件就绪且 YAML 合法
   - `{ ...e, config }`：YAML 解析出错时返回错误对象，附带配置文件名

**注意**：若 skeleton 复制失败，程序直接 `process.exit(1)` 终止。

---

## 三、环境变量替换机制

### 3.1 两种替换语法

环境变量替换发生在 **YAML 解析之前**（对原始字符串操作），支持两种前缀：

```js
// 伪代码说明替换顺序
raw = readFileSync(configFile)
substituted = substituteEnvironmentVars(raw)   // 先替换
parsed = yaml.load(substituted)               // 再解析
```

#### 类型一：`HOMEPAGE_VAR_*` —— 直接值替换

**实现位置**：`src/utils/config/config.js` 第 64-80 行

```yaml
# settings.yaml
title: "{{HOMEPAGE_VAR_MY_TITLE}}"
```

设置环境变量 `HOMEPAGE_VAR_MY_TITLE="My Dashboard"` 后，解析结果为 `title: "My Dashboard"`。

#### 类型二：`HOMEPAGE_FILE_*` —— 文件内容注入

```yaml
# services.yaml
widget:
  type: pihole
  apiKey: "{{HOMEPAGE_FILE_PIHOLE_KEY}}"
```

若 `HOMEPAGE_FILE_PIHOLE_KEY="/run/secrets/pihole"`，系统会读取该文件内容并替换占位符。此设计专为 Docker Secrets / Kubernetes Secrets 场景设计。

### 3.2 性能优化：内存缓存

`src/utils/config/config.js` 第 52-62 行使用 `memory-cache` 模块缓存环境变量过滤结果：

- **缓存键**：`homepageEnvironmentVariables`
- **缓存内容**：`Object.entries(process.env)` 过滤出前缀为 `HOMEPAGE_VAR_` 或 `HOMEPAGE_FILE_` 的条目
- **触发时机**：首次调用 `substituteEnvironmentVars()` 时构建，后续复用

---

## 四、核心配置读取流程

### 4.1 Settings 读取（含 layout 兼容转换）

**入口**：`getSettings()` —— `src/utils/config/config.js` 第 82-103 行

```
checkAndCopyConfig → 读文件 → 环境变量替换 → yaml.load → 转换 layout 格式
```

**特殊处理：layout 格式兼容**

为兼容历史版本，当 `settings.layout` 是 **数组** 时自动转换为 **对象**：

```yaml
# 旧格式（数组）—— 会被自动转换
layout:
  - GroupA:
      style: row
  - GroupB:
      style: column
```

转换后等价于：
```yaml
layout:
  GroupA:
    style: row
  GroupB:
    style: column
```

### 4.2 Services 三源读取详解

Services 是唯一涉及 **多源拼接合并** 的配置类型。三个来源独立读取，互不干扰，最后在 `servicesResponse()` 中统一按组拼接。

#### 来源 A：配置文件 `services.yaml`

**入口**：`servicesFromConfig()` —— `src/utils/config/service-helpers.js` 第 53-61 行

解析核心在 `parseServicesToGroups()` —— `src/utils/config/service-helpers.js` 第 17-51 行

**嵌套组递归解析规则**：
- 遍历 YAML 数组，每个条目是 `{ 组名: [ 条目数组 ] }`
- 条目如果是 **对象 + 值为数组**，则识别为 **子组**，递归解析
- 条目如果是 **对象 + 值为对象**，则识别为 **服务项**
- 服务项的 `weight` 默认分配：`(当前服务在组内索引 + 1) * 100`，用户显式指定的 `weight` 优先级更高

```yaml
# 嵌套示例
- Main:                # 主组
    - Child:           # 子组（因为值是数组）
        - SvcA: { href: "http://a" }
        - SvcB: { href: "http://b", weight: 5 }
    - SvcRoot: { href: "http://root" }  # 直接服务
```

解析结果结构：
```js
{
  name: "Main",
  type: "group",
  services: [ { name: "SvcRoot", weight: 100, type: "service", href: "http://root" } ],
  groups: [
    {
      name: "Child",
      type: "group",
      services: [
        { name: "SvcA", weight: 100, href: "http://a" },
        { name: "SvcB", weight: 5,   href: "http://b" }
      ],
      groups: []
    }
  ]
}
```

#### 来源 B：Docker 容器/服务标签发现

**入口**：`servicesFromDocker()` —— `src/utils/config/service-helpers.js` 第 63-170 行

**执行流程**：
1. 读取 `docker.yaml` 获取服务器列表
2. 并行连接所有 Docker 服务器（`Promise.all`），单台失败不影响整体
3. Swarm 模式调 `listServices`，普通模式调 `listContainers`
4. 过滤标签前缀为 `homepage.` 的容器
5. 使用 **shvl.set()** 将扁平标签路径还原为嵌套对象

**标签解析关键代码**（`src/utils/config/service-helpers.js` 第 98-121 行）：

```js
// 标签 homepage.href → shvl.set(obj, "href", value)
// 标签 homepage.widget.version → shvl.set(obj, "widget.version", value)
// 标签 homepage.widgets[0].type → shvl.set(obj, "widgets[0].type", value)
Object.keys(containerLabels).forEach((label) => {
  if (label.startsWith("homepage.")) {
    let value = label.replace("homepage.", "");
    // instance 过滤：instanceName 匹配才接受
    shvl.set(constructedService, value, substitutedVal);
  }
});
```

**实例隔离（instanceName）**：若 `settings.yaml` 中设置了 `instanceName: "foo"`，则：
- `homepage.instance.foo.description` 会被接受（去掉前缀后设为 `description`）
- `homepage.instance.bar.description` 会被忽略（属于其他实例）

**必需字段校验**：构造的服务必须同时包含 `name` 和 `group`，否则记录错误并丢弃。

**关键：Docker 来源服务的默认 `weight = 0`**，比 services.yaml 默认值 100+ 更小，排序时会排在前面。

#### 来源 C：Kubernetes 资源注解发现

**入口**：`servicesFromKubernetes()` —— `src/utils/config/service-helpers.js` 第 172-229 行

**支持的资源类型**：
- Ingress（标准 Kubernetes）
- Traefik IngressRoute（CRD）
- Gateway API HTTPRoute（`gateway.networking.k8s.io/v1`）

**核心逻辑**：
1. 通过 `getKubeConfig()` 根据 `kubernetes.yaml` 的 `mode` 字段选择加载方式：
   - `cluster`：集群内 ServiceAccount 认证
   - `default`：`~/.kube/config`
   - `disabled` 或空：直接返回空数组
2. 并行获取三类资源列表
3. 用 `isDiscoverable(resource, instanceName)` 过滤带 `gethomepage.dev/*` 注解的资源
4. `constructedServiceFromResource()` 提取注解构造服务对象

**关键：Kubernetes 来源服务的默认 `weight = 0`**，与 Docker 相同。

### 4.3 Bookmarks 读取

**入口**：`bookmarksResponse()` —— `src/utils/config/api-response.js` 第 27-71 行

数据结构转换：
```yaml
# 输入格式（YAML 易写格式）
- Developer:
    - Github:
        - abbr: GH
          href: https://github.com/
```

转换为：
```js
{
  name: "Developer",
  bookmarks: [ { name: "Github", abbr: "GH", href: "https://github.com/" } ]
}
```

### 4.4 Widgets 读取

**入口**：`widgetsFromConfig()` —— `src/utils/config/widget-helpers.js` 第 8-27 行

```yaml
- resources:
    cpu: true
```
转换为：
```js
{ type: "resources", options: { index: 0, cpu: true } }
```

`index` 字段根据 YAML 数组顺序自动分配，用于后续 API 匹配对应 widget 的私有参数。

---

## 五、Services 合并规则详解（核心重点）

> **核心原则：同名服务不会互相覆盖。三者结果（Docker / Kubernetes / services.yaml）按组进行数组拼接，最后统一按 weight 和 name 排序输出。**

Services 的合并逻辑集中在 `servicesResponse()` 函数 —— `src/utils/config/api-response.js` 第 158-256 行。

### 5.1 合并流程总览（7 个步骤）

```
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: 并行加载三源数据（互不干扰，各自独立）                     │
│   servicesFromDocker()    →  discoveredDockerServices[ {groupName, │
│                                                    services: [...]}]│
│   servicesFromKubernetes()→  discoveredK8sServices   [同上结构]     │
│   servicesFromConfig()    →  configuredServices      [同上结构]     │
│   任一失败 → 降级为空数组 []                                       │
├────────────────────────────────────────────────────────────────────┤
│ Step 2: 合并前清洗 cleanServiceGroups()                            │
│   - 规范化 weight 为数字（字符串→parseInt，非法值→0）              │
│   - widget 单对象追加到 widgets 数组末尾                           │
│   - 字段白名单过滤 + 按类型解析布尔/JSON/数字                       │
├────────────────────────────────────────────────────────────────────┤
│ Step 3: 构建合并组名集合 mergedGroupsNames（去重）                 │
│   new Set([                                                        │
│     ...dockerGroups.map(g→g.name),                                 │
│     ...k8sGroups.map(g→g.name),                                    │
│     ...configGroups.map(g→g.name),                                 │
│   ])                                                               │
├────────────────────────────────────────────────────────────────────┤
│ Step 4: 注入 layout-only 组与嵌套结构（services.yaml 或 layout 均可声明） │
│   convertLayoutGroupToGroup() → mergeLayoutGroupsIntoConfigured()    │
│   settings.layout 中定义但 services.yaml 未定义的组/嵌套子组          │
│   会被构造成空组结构注入 configuredServices 树                        │
├────────────────────────────────────────────────────────────────────┤
│ Step 5: ★ 按组名拼接三源服务数组（关键！不是覆盖）                  │
│   对每个 groupName：                                                │
│     dockerSvc = findGroup(dockerServices, groupName)?.services     │
│     k8sSvc    = findGroup(k8sServices, groupName)?.services        │
│     configSvc = findGroup(configServices, groupName)?.services     │
│                                                                    │
│     mergedGroup.services = [                                       │
│       ...dockerSvc,   // 先放 Docker 发现的                         │
│       ...k8sSvc,      // 再接 Kubernetes 发现的                    │
│       ...configSvc,   // 最后放手动配置的（默认 weight=100+）        │
│     ].filter(Boolean)                                              │
│      .sort(compareServices)  // 按 weight → name 排序              │
│                                                                    │
│   ⚠️  同名服务不会覆盖，全部保留！                                  │
│   如果 Docker 和 services.yaml 中都定义了同名 "Nginx"，             │
│   结果是两个 "Nginx" 条目并存，由排序决定先后                       │
│                                                                    │
│   mergedGroup.groups 来自 configuredGroup.groups                    │
│   （配置树 = services.yaml 原生嵌套 + Step 4 layout 注入的嵌套）     │
├────────────────────────────────────────────────────────────────────┤
│ Step 6: 组排序 + 嵌套子组归属处理                                  │
│   - settings.layout 中出现的组按 key 定义顺序排列                   │
│   - 未在 layout 中的组追加到末尾                                   │
│   - 带 parent 属性的子组递归合并到父组（深度遍历）                  │
├────────────────────────────────────────────────────────────────────┤
│ Step 7: pruneEmptyGroups() 裁剪空组                                │
│   group.services.length === 0 && group.groups.length === 0 → 删除  │
└────────────────────────────────────────────────────────────────────┘
```

### 5.2 Step 5 详解：按组拼接而非覆盖

这是整个合并逻辑的核心。代码位于 `src/utils/config/api-response.js` 第 220-252 行：

```js
mergedGroupsNames.forEach((groupName) => {
  const discoveredDockerGroup = findGroupByName(discoveredDockerServices, groupName)
                              || { services: [] };
  const discoveredKubernetesGroup = findGroupByName(discoveredKubernetesServices, groupName)
                                  || { services: [] };
  const configuredGroup = findGroupByName(configuredServices, groupName)
                        || { services: [], groups: [] };

  // ★ 关键：数组展开（拼接），不是对象合并！
  const mergedGroup = {
    name: groupName,
    services: [
      ...discoveredDockerGroup.services,     // 来源 1：Docker
      ...discoveredKubernetesGroup.services, // 来源 2：Kubernetes
      ...configuredGroup.services,           // 来源 3：services.yaml
    ]
      .filter((service) => service)          // 过滤空值
      .sort(compareServices),                // 排序（不是覆盖！）
    groups: [...configuredGroup.groups],     // 嵌套结构来自配置树（services.yaml + layout 注入）
  };
  // ...
});
```

**实际效果示例**：

假设三源均在 "Tools" 组中定义了服务：

| 来源 | services |
|------|----------|
| Docker（weight=0） | `[{name: "Portainer"}, {name: "Nginx"}]` |
| Kubernetes（weight=0） | `[{name: "Traefik"}]` |
| services.yaml（默认 weight=100+） | `[{name: "Nginx", description: "手动配置的Nginx"}, {name: "VSCode"}]` |

**拼接后（未排序）**：
```js
[
  {name: "Portainer", weight: 0},        // Docker
  {name: "Nginx",     weight: 0},        // Docker
  {name: "Traefik",   weight: 0},        // Kubernetes
  {name: "Nginx",     weight: 100, description: "手动配置的Nginx"},  // services.yaml
  {name: "VSCode",    weight: 200},      // services.yaml
]
```

**排序后（按 weight → name 字母序）**：
```js
[
  {name: "Nginx",     weight: 0},                        // Docker（字母序 N 在 P/T 前）
  {name: "Portainer", weight: 0},                        // Docker
  {name: "Traefik",   weight: 0},                        // Kubernetes
  {name: "Nginx",     weight: 100, description: "..."},  // services.yaml
  {name: "VSCode",    weight: 200},                      // services.yaml
]
```

**结论**：
- 两个名为 "Nginx" 的服务 **同时存在**，没有覆盖
- weight 更小的 Docker Nginx 排在前面
- weight 更大的手动配置 Nginx 排在后面
- 若需要调整顺序，可在 Docker 标签或 services.yaml 中显式设置 `weight`

### 5.3 服务排序规则

`compareServices()` —— `src/utils/config/api-response.js` 第 19-25 行：

```js
function compareServices(service1, service2) {
  const comp = service1.weight - service2.weight;
  if (comp !== 0) {
    return comp;  // 第一关键字：weight 升序（值越小越靠前）
  }
  return service1.name.localeCompare(service2.name);  // 第二关键字：name 字母序
}
```

```
排序优先级：
1. weight 升序（数值越小越靠前）
   - Docker / Kubernetes 发现的服务：默认 weight = 0
   - services.yaml 手动配置的服务：默认 weight = (index + 1) * 100
2. weight 相同时按 name 字母顺序升序（localeCompare）
```

### 5.4 组排序规则（layout 驱动）

```js
const definedLayouts = initialSettings.layout ? Object.keys(initialSettings.layout) : null;
```

- 布局中出现的组 → 按 `layout` key **出现顺序** 排列到 `sortedGroups` 对应下标位置
- 布局中 **未出现** 的组 → 追加到 `unsortedGroups`，顺序为三源合并时 **首次出现** 的顺序（即 Docker → K8s → services.yaml 依次遍历 groupName Set 的顺序）

### 5.5 空组裁剪

`pruneEmptyGroups()` —— `src/utils/config/api-response.js` 第 123-134 行

递归遍历：若组的 `services` 和 `groups` 都为空，则从结果中移除。这确保了 `settings.layout` 中仅用于结构声明、实际无服务的组不会显示空壳。

### 5.6 嵌套子组合并

嵌套子组的处理涉及三个关键函数，以及一个核心内部属性 `parent`。

#### 5.6.1 `parent` 属性的本质：仅是内部临时归属标识

**`parent` 不是用户可配置的字段。** 它不出现在任何 YAML 文件、Docker 标签或 Kubernetes 注解中，也无法通过 `homepage.parent` 标签或 `gethomepage.dev/parent` 注解来设置。

`parent` 唯一的赋值来源是 `findGroupByName()` 函数在深度遍历 **合并配置树**（= services.yaml 原生结构 + Step 4 layout 注入的嵌套结构）时的动态标注行为：

```js
// src/utils/config/service-helpers.js 第 710-725 行
export function findGroupByName(groups, name) {
  for (let i = 0; i < groups.length; i += 1) {
    const group = groups[i];
    if (group.name === name) {
      return group;              // 顶层直接匹配，不设 parent
    } else if (group.groups) {
      const foundGroup = findGroupByName(group.groups, name);
      if (foundGroup) {
        foundGroup.parent = group.name;  // ★ 在递归返回时动态标注父组名
        return foundGroup;
      }
    }
  }
  return null;
}
```

**关键语义**：
- 如果目标组是顶层数组的直接成员，`findGroupByName` 返回时 **不会** 设置 `parent`
- 只有当目标组是在某个父组的 `groups` 子树中被递归找到时，才会被动态打上 `parent = 父组名` 标记
- 这个标记在每次调用 `findGroupByName` 时重新计算，是 **临时的运行时状态**，不是持久化配置

#### 5.6.2 `parent` 的消费链路

`parent` 在 `servicesResponse()` 的主循环中被消费（`src/utils/config/api-response.js` 第 237-251 行）：

```js
if (definedLayouts) {
  const layoutIndex = definedLayouts.findIndex((layout) => layout === mergedGroup.name);
  if (layoutIndex > -1) sortedGroups[layoutIndex] = mergedGroup;
  else if (configuredGroup.parent) {
    // ← configuredGroup 来自 findGroupByName(configuredServices, groupName)
    //   如果该组名在 configuredServices 树中是嵌套子组，
    //   则 configuredGroup.parent 已被 findGroupByName 动态标注
    mergeSubgroups(configuredServices, mergedGroup);
    ensureParentGroupExists(sortedGroups, configuredServices, configuredGroup, definedLayouts);
  } else unsortedGroups.push(mergedGroup);
} else if (configuredGroup.parent) {
  mergeSubgroups(configuredServices, mergedGroup);
} else {
  unsortedGroups.push(mergedGroup);
}
```

**判断逻辑**：
- 对每个 `groupName`，通过 `findGroupByName(configuredServices, groupName)` 在 **合并配置树** 中查找
- **合并配置树** = services.yaml 原生结构 **+** Step 4 通过 `mergeLayoutGroupsIntoConfigured()` 注入的 layout 嵌套结构
- 若找到且返回对象带有 `.parent`，说明该组名在合并配置树中是某个父组的嵌套子组（不论该结构源自 services.yaml 还是 layout）
- 此时不将该组当作独立顶层组放入 `sortedGroups` / `unsortedGroups`，而是通过 `mergeSubgroups` 将其服务挂回父组

#### 5.6.3 `mergeSubgroups()` —— 将拼接结果写回父组的嵌套位置

```js
// src/utils/config/api-response.js 第 99-107 行
function mergeSubgroups(configuredGroups, mergedGroup) {
  configuredGroups.forEach((group) => {
    if (group.name === mergedGroup.name) {
      group.services = mergedGroup.services;  // 用三源拼接+排序后的结果替换
    } else if (group.groups) {
      mergeSubgroups(group.groups, mergedGroup);  // 递归深入子树
    }
  });
}
```

在 `configuredServices` 合并配置树中递归查找与 `mergedGroup` 同名的组，找到后用三源拼接+排序后的 services 替换。效果是：Docker/K8s 发现的同名组服务被注入到 services.yaml 或 settings.yaml layout 已声明的嵌套位置。

#### 5.6.4 `ensureParentGroupExists()` —— 确保顶层父组进入排序结果

```js
// src/utils/config/api-response.js 第 109-121 行
function ensureParentGroupExists(sortedGroups, configuredGroups, group, definedLayouts) {
  const parentGroupName = group.parent;
  const parentGroup = findGroupByName(configuredGroups, parentGroupName);
  if (parentGroup && parentGroup.parent) {
    // 父组本身也是嵌套子组 → 继续向上追溯
    ensureParentGroupExists(sortedGroups, configuredGroups, parentGroup, definedLayouts);
  } else {
    // 到达顶层父组 → 放入 sortedGroups 对应 layout 位置
    const parentGroupIndex = definedLayouts.findIndex((layout) => layout === parentGroupName);
    if (parentGroupIndex > -1) {
      sortedGroups[parentGroupIndex] = parentGroup;
    }
  }
}
```

由于嵌套子组的服务被写回父组的 `groups` 子树中，子组自身不会出现在 `sortedGroups` / `unsortedGroups` 中。但子组的 **顶层父组** 必须被放入 `sortedGroups`，否则整个嵌套树都不会出现在最终输出中。此函数就是确保这一点。

对于多层嵌套（如 `Root → Top → Child`），函数会递归向上追溯直到找到没有 `parent` 标记的顶层组。

#### 5.6.5 自动发现服务进入嵌套子组的前提条件

**核心前提**：services.yaml 或 settings.yaml 的 layout 必须已声明相应的嵌套子组结构。

Docker/K8s 自动发现的服务本身 **只有 `group` 字段**（Docker 标签 `homepage.group`，K8s 注解 `gethomepage.dev/group`），该字段仅声明服务所属的组名，无法表达"这个组是某个父组的子组"这一层级关系。嵌套层级关系 **完全依赖** services.yaml 或 layout 中的声明。

具体有两条路径：

**路径 A：通过 services.yaml 声明嵌套结构**

```yaml
# services.yaml
- Top:                    # 父组
    - Child:              # 嵌套子组
        - SvcA:
            href: http://a
```

此时 `configuredServices` 合并配置树中已存在 `Top.groups[0].name = "Child"` 的嵌套结构。当 Docker 发现了 `homepage.group=Child` 的服务时：
1. `mergedGroupsNames` 包含 "Child"
2. `findGroupByName(configuredServices, "Child")` 在递归遍历中找到它，并动态标注 `parent = "Top"`
3. 进入 `configuredGroup.parent` 分支，`mergeSubgroups` 将三源拼接结果写回 `Top.groups[0].services`
4. `ensureParentGroupExists` 确保 "Top" 进入 sortedGroups

**路径 B：通过 settings.yaml layout 声明嵌套结构**

```yaml
# settings.yaml
layout:
  Top:
    Child:
```

此时 layout 先通过 `convertLayoutGroupToGroup()` 转换为空组结构，再通过 `mergeLayoutGroupsIntoConfigured()` 在 Step 4 注入 `configuredServices` 合并配置树中，后续流程与路径 A 相同。

**如果既没有在 services.yaml 也没有在 layout 中声明嵌套结构**，那么即使 Docker/K8s 发现了 `group=Child` 的服务，"Child" 只会被当作一个 **普通顶层组** 处理，不会被挂入任何父组。

#### 5.6.6 `mergeLayoutGroupsIntoConfigured()` —— layout 声明的嵌套结构注入

```js
// src/utils/config/api-response.js 第 136-156 行
function mergeLayoutGroupsIntoConfigured(configuredGroups, layoutGroups) {
  for (const layoutGroup of layoutGroups) {
    const existing = findGroupByName(configuredGroups, layoutGroup.name);
    if (existing) {
      // 组名已存在于 services.yaml → 递归合并子组
      if (layoutGroup.groups?.length) {
        existing.groups ??= [];
        for (const sub of layoutGroup.groups) {
          const existingSub = findGroupByName(existing.groups, sub.name);
          if (!existingSub) {
            existing.groups.push(sub);       // 新增子组
          } else {
            mergeLayoutGroupsIntoConfigured([existingSub], [sub]);  // 递归深合并
          }
        }
      }
    } else {
      configuredGroups.push(layoutGroup);  // 全新组直接追加
    }
  }
}
```

此函数在 `servicesResponse()` 的 Step 4 执行，将 layout 中声明的嵌套组结构注入到 `configuredServices` 合并配置树中。这使得即使 services.yaml 中没有声明某个子组，仅靠 layout 也可以建立起嵌套结构，为后续 Docker/K8s 服务的自动挂入提供"锚点"。

---

## 六、shvl 扁平路径操作库

**实现位置**：`src/utils/config/shvl.js`

基于开源库 robinvdvleuten/shvl 的精简实现，核心作用是将 **点号 + 方括号路径字符串** 转换为对象读写操作，是 Docker 标签扁平结构还原的关键基础设施。

### 6.1 `get(object, path, def)`

```
路径分割正则：/[.[\]]+/
例："widgets[0].version" → ["widgets", "0", "version"]
```

逐键 reduce 深入对象，遇 `undefined` 则返回默认值 `def`。

### 6.2 `set(obj, path, val)`

```js
// 示例
shvl.set({}, "widgets[0].type", "pihole")
// → { widgets: [ { type: "pihole" } ] }
```

**核心逻辑**：
- 路径分割后过滤空串
- 弹出末位键用于最终赋值
- **安全防护**：路径段匹配 `__proto__|constructor|prototype` 时直接返回，阻止原型污染
- 动态创建结构：若下一级键是纯数字（`/^\d+$/`）则初始化为数组 `[]`，否则初始化为对象 `{}`

---

## 七、前端配置消费与验证链路

### 7.1 静态生成（getStaticProps）

**入口**：`src/pages/index.jsx` 第 55-95 行

```js
// 构建时一次性生成 SWR fallback 缓存
props: {
  initialSettings,           // getSettings() 结果
  fallback: {
    "/api/services": services,    // servicesResponse() 结果
    "/api/bookmarks": bookmarks,  // bookmarksResponse() 结果
    "/api/widgets": widgets,      // widgetsResponse() 结果
    "/api/hash": false,
  },
  // 同时注入国际化翻译资源
}
```

### 7.2 运行时 API 端点

| 端点 | 实现文件 | 返回内容 |
|------|----------|----------|
| `GET /api/services` | `src/pages/api/services/index.js` | 合并后（拼接+排序）的服务组树 |
| `GET /api/bookmarks` | `src/pages/api/bookmarks.js` | 书签组数组 |
| `GET /api/widgets` | `src/pages/api/widgets/index.js` | 小部件数组（已脱敏） |
| `GET /api/validate` | `src/pages/api/validate.js` | YAML 语法错误数组 |
| `GET /api/config/custom.css` | `src/pages/api/config/[path].js` | 用户自定义 CSS |
| `GET /api/config/custom.js` | 同上 | 用户自定义 JS |

### 7.3 配置变更热检测

**入口**：`src/pages/index.jsx` 第 102-131 行

```
窗口获得焦点 → 拉取 /api/hash → 与 localStorage 中 hash 比对
         ↓ 不相等
localStorage 更新 + setStale(true)
         ↓
fetch("/api/revalidate") → 触发 Next.js ISR 重建 → window.location.reload()
```

### 7.4 Widget 敏感字段分离设计

为防止密钥泄露到前端，Widgets 采用 **"公共字段下发 + 私有字段后端按需取回"** 模式：

- `cleanWidgetGroups()` —— `src/utils/config/widget-helpers.js` 第 29-54 行
  - 删除 `username / password / key / apiKey`
  - 除 `search` 和 `glances` 外，删除 `url`

- `getPrivateWidgetOptions()` —— `src/utils/config/widget-helpers.js` 第 56-79 行
  - 后端 proxy 处理请求时，通过 `type + index` 从原始配置取回完整私有参数

Services Widgets 也有类似的白名单过滤（`src/utils/config/service-helpers.js` 第 253-439 行），仅将前端渲染必需字段下发。

---

## 八、合并规则优先级总结表

| 维度 | 规则（高 → 低 / 先 → 后） | 说明 |
|------|--------------------------|------|
| **Service 拼接顺序** | Docker 标签 → Kubernetes 注解 → services.yaml | 三个来源的 services 数组按此顺序展开拼接，**不覆盖** |
| **Service 排序优先级** | weight 升序 → name 字母序 | 拼接后的数组统一排序；Docker/K8s 默认 weight=0，services.yaml 默认 weight=100+ |
| **同名服务处理** | **全部保留，不做去重** | 若想去掉重复，需手动删除某个来源中的定义，或通过 weight 控制前后顺序 |
| **组排序** | settings.layout key 出现顺序 → 首次出现顺序 | layout 中定义的组严格按 key 顺序排列，未定义的追加到末尾 |
| **Bookmarks 组排序** | settings.layout key 出现顺序 → YAML 数组顺序 | 同 Services 组排序 |
| **settings 中字段** | YAML 显式值 > 代码默认值 | 如 `headerStyle` 默认为 "underlined"，可被 YAML 覆盖 |
| **环境变量替换** | 不涉及覆盖（值替换） | 在 YAML 解析前执行，占位符找不到匹配值则保留原文 |
| **嵌套子组结构** | services.yaml 或 settings.yaml layout 可声明 | `parent` 是 `findGroupByName` 遍历配置树时的内部临时标注，不是用户可配置字段 |
| **嵌套子组内的服务** | 同样是三源拼接 + 排序 | 进入子组后应用与顶层相同的 "拼接→过滤→排序" 流程 |
| **Widget 单复数** | widgets 数组 + widget 单对象 → 合并为 widgets | widget 追加到数组末尾（`cleanServiceGroups`） |

---

## 九、常见配置陷阱

1. **同名服务不会被覆盖，会重复出现**：
   如果 Docker 标签和 services.yaml 都定义了名为 "Nginx" 的服务且在同一组，最终结果会同时显示两个 Nginx 卡片。解决方法：删除其中一个定义，或显式设置不同的 `weight` 控制顺序。

2. **Docker/K8s 服务 `weight` 默认为 0**：
   会排在配置文件服务（默认 100+）之前。若想让手动配置的服务出现在前面，需在 services.yaml 中显式设 `weight: 0` 或更小。

3. **YAML 解析失败静默降级**：
   `servicesResponse()` 中任何一个来源异常都会被 catch 并降级为空数组，但 `/api/validate` 会返回详细错误，可通过访问该端点排查。

4. **`{{HOMEPAGE_VAR_*}}` 不匹配留原值**：
   如果环境变量未设置，占位符字符串 `{{HOMEPAGE_VAR_XXX}}` 会被保留为字面值，可能引发后续解析问题（如 URL 拼接错误）。

5. **layout-only 组会被最终裁掉**：
   只在 `settings.layout` 中声明、但三源都没有任何服务的组，会被 `pruneEmptyGroups()` 从最终结果中移除（除非其内部嵌套子组有服务）。

6. **原型污染防护**：
   `shvl.set()` 会拒绝 `__proto__`、`constructor`、`prototype` 路径，这是安全设计，不是 Bug。

7. **`parent` 不是用户可配置字段**：
   `parent` 属性是 `findGroupByName()` 在深度遍历 **`configuredServices` 合并配置树**（= services.yaml 原生结构 + Step 4 layout 注入的嵌套结构）时的 **内部临时标注**，仅用于标记"这个组在配置树中是某个父组的嵌套子组"。它不出现在任何 YAML 文件、Docker 标签（无 `homepage.parent`）或 Kubernetes 注解（无 `gethomepage.dev/parent`）中，每次运行时由代码动态计算。

8. **自动发现服务进入嵌套子组的前提条件**：
   Docker/K8s 自动发现的服务只有 `group` 字段（Docker: `homepage.group`，K8s: `gethomepage.dev/group`），该字段仅声明组名，无法表达嵌套层级。要使自动发现的服务进入嵌套子组，**必须** 在 services.yaml 或 settings.yaml layout 中预先声明该子组结构。否则，该组只会被当作普通顶层组处理。

9. **配置文件中非法条目会被静默跳过**：
   `parseServicesToGroups()` 中如果某个服务条目的值为 `null` 或空，会记录一条 warn 日志后跳过该条目（`src/utils/config/service-helpers.js` 第 29-32 行）。
