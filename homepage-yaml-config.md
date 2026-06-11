# Homepage YAML 配置体系与合并逻辑深度分析

## 一、整体架构概览

Homepage 的 YAML 配置系统采用 **"多文件分治 + 多源合并 + 分层覆盖"** 的设计模式。配置体系分为三个主要层次：

1. **配置文件层**：8 个独立 YAML 文件各司其职
2. **来源发现层**：Services 支持 3 种来源（配置文件 / Docker 标签 / Kubernetes 注解）
3. **运行时层**：通过 Context + SWR 实现前端消费与热更新

核心配置处理链路分布在以下关键模块：

| 模块 | 职责 | 文件位置 |
|------|------|----------|
| 配置基础能力 | 目录管理、文件校验、环境变量替换 | [config.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/config.js) |
| 扁平路径操作 | 点号/数组路径读写（shvl 库） | [shvl.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/shvl.js) |
| API 响应构造 | 多源合并、排序、空组裁剪 | [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js) |
| Services 解析 | 3 种来源的数据提取与标准化 | [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js) |
| Widgets 解析 | 配置读取与敏感字段清洗 | [widget-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/widget-helpers.js) |

---

## 二、配置文件清单与职责

### 2.1 八大配置文件

所有配置文件均位于 `CONF_DIR` 目录，通过环境变量 `HOMEPAGE_CONFIG_DIR` 指定，默认值为 `<cwd>/config`。首次启动时若文件不存在，会从 `src/skeleton/` 复制模板文件。

| 文件名 | 职责 | 是否参与合并 | 读取入口 |
|--------|------|-------------|----------|
| `settings.yaml` | 全局设置（主题、布局、语言、背景、provider 密钥等） | 不合并（单一来源） | `getSettings()` |
| `services.yaml` | 静态服务组定义（手动配置） | 与 Docker/K8s 合并 | `servicesFromConfig()` |
| `bookmarks.yaml` | 书签组定义 | 不合并（单一来源） | `bookmarksResponse()` |
| `widgets.yaml` | 顶部信息小部件定义 | 不合并（单一来源） | `widgetsFromConfig()` |
| `docker.yaml` | Docker 服务器连接参数 | 本身不合并，但驱动服务发现 | `getDockerArguments()` |
| `kubernetes.yaml` | Kubernetes 集群模式配置 | 本身不合并，驱动服务发现 | `getKubernetes()` / `getKubeConfig()` |
| `proxmox.yaml` | Proxmox 连接参数 | 不合并 | `getProxmoxConfig()` |
| `custom.css` / `custom.js` | 用户自定义样式与脚本（非 YAML） | 不合并 | `/api/config/[path]` |

### 2.2 配置文件自动初始化机制

`checkAndCopyConfig(config)` 函数负责配置文件的生命周期管理：

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

**实现位置**：[config.js:64-80](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/config.js#L64-L80)

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

[config.js:52-62](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/config.js#L52-L62) 使用 `memory-cache` 模块缓存环境变量过滤结果：

- **缓存键**：`homepageEnvironmentVariables`
- **缓存内容**：`Object.entries(process.env)` 过滤出前缀为 `HOMEPAGE_VAR_` 或 `HOMEPAGE_FILE_` 的条目
- **触发时机**：首次调用 `substituteEnvironmentVars()` 时构建，后续复用

---

## 四、核心配置读取流程

### 4.1 Settings 读取（含 layout 兼容转换）

**入口**：`getSettings()` —— [config.js:82-103](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/config.js#L82-L103)

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

### 4.2 Services 多源读取详解

Services 是唯一涉及 **多源合并** 的配置类型，三个来源分别由三个独立函数读取：

#### 来源 A：配置文件 `services.yaml`

**入口**：`servicesFromConfig()` —— [service-helpers.js:53-61](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L53-L61)

解析核心在 `parseServicesToGroups()` —— [service-helpers.js:17-51](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L17-L51)

**嵌套组递归解析规则**：
- 遍历 YAML 数组，每个条目是 `{ 组名: [ 条目数组 ] }`
- 条目如果是 **对象 + 值为数组**，则识别为 **子组**，递归解析
- 条目如果是 **对象 + 值为对象**，则识别为 **服务项**
- 服务项的 `weight` 默认分配：`(当前服务在组内索引 + 1) * 100`，用户显式指定的 `weight` 优先级更高

```yaml
# 嵌套示例
- Main:                # 主组
    - Child:           # 子组（因为值是数组）
        - SvcA: { ... }
        - SvcB: { ... }
    - SvcRoot: { ... } # 直接服务
```

解析结果结构：
```js
{
  name: "Main",
  type: "group",
  services: [ { name: "SvcRoot", weight: 100, type: "service", ... } ],
  groups: [
    {
      name: "Child",
      type: "group",
      services: [ { name: "SvcA", weight: 100 }, { name: "SvcB", weight: 200 } ],
      groups: []
    }
  ]
}
```

#### 来源 B：Docker 容器/服务标签发现

**入口**：`servicesFromDocker()` —— [service-helpers.js:63-170](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L63-L170)

**执行流程**：
1. 读取 `docker.yaml` 获取服务器列表
2. 并行连接所有 Docker 服务器（`Promise.all`），单台失败不影响整体
3. Swarm 模式调 `listServices`，普通模式调 `listContainers`
4. 过滤标签前缀为 `homepage.` 的容器
5. 使用 **shvl.set()** 将扁平标签路径还原为嵌套对象

**标签解析关键代码**（[service-helpers.js:98-121](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L98-L121)）：

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

#### 来源 C：Kubernetes 资源注解发现

**入口**：`servicesFromKubernetes()` —— [service-helpers.js:172-229](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L172-L229)

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

### 4.3 Bookmarks 读取

**入口**：`bookmarksResponse()` —— [api-response.js:27-71](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js#L27-L71)

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

**入口**：`widgetsFromConfig()` —— [widget-helpers.js:8-27](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/widget-helpers.js#L8-L27)

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

Services 的合并是整个配置体系最复杂的部分，逻辑集中在 `servicesResponse()` 函数。

**入口**：`servicesResponse()` —— [api-response.js:158-256](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js#L158-L256)

### 5.1 合并流程总览（7 个步骤）

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: 并行加载三源数据                                    │
│   servicesFromDocker() / servicesFromKubernetes() /         │
│   servicesFromConfig()，任一失败降级为空数组                │
├─────────────────────────────────────────────────────────────┤
│ Step 2: 合并前清洗 cleanServiceGroups()                     │
│   - 规范化 weight 为数字（字符串→parseInt，对象→0）         │
│   - widget 单对象追加到 widgets 数组末尾                    │
│   - 字段白名单过滤 + 按类型解析布尔/JSON/数字                │
├─────────────────────────────────────────────────────────────┤
│ Step 3: 构建合并组名集合 mergedGroupsNames                  │
│   Set([...dockerGroups, ...k8sGroups, ...configGroups])     │
├─────────────────────────────────────────────────────────────┤
│ Step 4: 注入 layout-only 组                                 │
│   mergeLayoutGroupsIntoConfigured()                         │
│   settings.layout 中定义但 services.yaml 未定义的组          │
│   会被构造成空组插入 configuredServices                     │
├─────────────────────────────────────────────────────────────┤
│ Step 5: 按组名合并三源服务                                  │
│   mergedGroup.services = [                                  │
│     ...docker.services,  // 优先级最低                      │
│     ...k8s.services,     // 优先级中等                      │
│     ...config.services   // 优先级最高（追加在最后）         │
│   ].sort(compareServices)                                   │
│                                                             │
│   mergedGroup.groups 仅来自 configuredGroup.groups          │
├─────────────────────────────────────────────────────────────┤
│ Step 6: 排序 + 子组归属处理                                 │
│   - settings.layout 中出现的组按定义顺序排列                │
│   - 未在 layout 中的组追加到末尾                            │
│   - 带 parent 属性的子组递归合并到父组                      │
├─────────────────────────────────────────────────────────────┤
│ Step 7: pruneEmptyGroups() 裁剪空组                         │
│   services.length === 0 && groups.length === 0 → 删除       │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 服务排序规则

`compareServices()` —— [api-response.js:19-25](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js#L19-L25)

```
排序优先级：
1. weight 升序（数值越小越靠前，默认：config=100+, docker/k8s=0）
2. weight 相同时按 name 字母顺序升序
```

**关键含义**：Docker/K8s 自动发现的服务 `weight=0`，配置文件服务 `weight=100+`，故 **自动发现服务默认排在手动配置服务之前**。若想调整，需在 Docker 标签或 YAML 中显式设 `weight`。

### 5.3 组排序规则（layout 驱动）

`definedLayouts = Object.keys(settings.layout)`

- 布局中出现的组 → 按 `layout` key 出现顺序排列到 `sortedGroups` 对应下标位置
- 布局中未出现的组 → 追加到 `unsortedGroups`，顺序为三源合并时首次出现顺序

### 5.4 空组裁剪

`pruneEmptyGroups()` —— [api-response.js:123-134](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js#L123-L134)

递归遍历：若组的 `services` 和 `groups` 都为空，则从结果中移除。这确保了 `settings.layout` 中仅用于结构声明、实际无服务的组不会显示空壳。

### 5.5 嵌套子组合并

嵌套子组的处理涉及两个辅助函数：

**`findGroupByName()`** —— [service-helpers.js:710-725](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L710-L725)

深度优先搜索，找到子组时会动态附加 `parent` 属性，供后续归属判断使用。

**`mergeSubgroups()`** —— [api-response.js:99-107](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/api-response.js#L99-L107)

当服务组来自 Docker/K8s 自动发现但 `parent` 属性指示它是嵌套子组时：
1. 递归在 configuredGroups 树中查找同名组
2. 找到后用合并后的 services 替换原 services
3. `ensureParentGroupExists()` 确保顶层父组在 sortedGroups 中有对应位置

---

## 六、shvl 扁平路径操作库

**实现位置**：[shvl.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/shvl.js)

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

**入口**：[index.jsx:55-95](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/index.jsx#L55-L95)

```js
// 构建时一次性生成 SWR fallback 缓存
props: {
  initialSettings,           // getSettings() 结果
  fallback: {
    "/api/services": services,    // servicesResponse()
    "/api/bookmarks": bookmarks,  // bookmarksResponse()
    "/api/widgets": widgets,      // widgetsResponse()
    "/api/hash": false,
  },
  // 同时注入国际化翻译资源
}
```

### 7.2 运行时 API 端点

| 端点 | 实现 | 返回内容 |
|------|------|----------|
| `GET /api/services` | [services/index.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/api/services/index.js) | 合并后的服务组树 |
| `GET /api/bookmarks` | [bookmarks.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/api/bookmarks.js) | 书签组数组 |
| `GET /api/widgets` | [widgets/index.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/api/widgets/index.js) | 小部件数组（已脱敏） |
| `GET /api/validate` | [validate.js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/api/validate.js) | YAML 语法错误数组 |
| `GET /api/config/custom.css` | [config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/api/config/[path].js) | 用户自定义 CSS |
| `GET /api/config/custom.js` | 同上 | 用户自定义 JS |

### 7.3 配置变更热检测

**入口**：[index.jsx:102-131](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/pages/index.jsx#L102-L131)

```
窗口获得焦点 → 拉取 /api/hash → 与 localStorage 中 hash 比对
         ↓ 不相等
localStorage 更新 + setStale(true)
         ↓
fetch("/api/revalidate") → 触发 Next.js ISR 重建 → window.location.reload()
```

### 7.4 Widget 敏感字段分离设计

为防止密钥泄露到前端，Widgets 采用 **"公共字段下发 + 私有字段后端按需取回"** 模式：

- `cleanWidgetGroups()` —— [widget-helpers.js:29-54](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/widget-helpers.js#L29-L54)
  - 删除 `username / password / key / apiKey`
  - 除 `search` 和 `glances` 外，删除 `url`

- `getPrivateWidgetOptions()` —— [widget-helpers.js:56-79](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/widget-helpers.js#L56-L79)
  - 后端 proxy 处理请求时，通过 `type + index` 从原始配置取回完整私有参数

Services Widgets 也有类似的白名单过滤（[service-helpers.js:253-439](file:///d:/fz/0601/solo-dogfeeding/code/195-homepage/src/utils/config/service-helpers.js#L253-L439)），仅将前端渲染必需字段下发。

---

## 八、合并规则优先级总结表

| 维度 | 优先级（高 → 低） | 说明 |
|------|------------------|------|
| **Service 项来源** | services.yaml > Kubernetes 注解 > Docker 标签 | 体现在数组合并顺序（后追加的排后面），但受 weight 影响 |
| **Service 项排序** | weight 数值（升序）> name 字母序（升序） | weight 相同时按字母序；Docker/K8s 默认 weight=0，config 默认 100+ |
| **组排序** | settings.layout 键出现顺序 > 首次出现顺序 | layout 中定义的组严格按 key 顺序排列，未定义的追加到末尾 |
| **Bookmarks 组排序** | settings.layout 键出现顺序 > YAML 数组顺序 | 同 Services 组排序 |
| **settings 中字段** | YAML 显式值 > 代码默认值 | 如 `headerStyle` 默认为 "underlined"，可被 YAML 覆盖 |
| **环境变量替换** | 无冲突（是值替换，不是覆盖） | 在 YAML 解析前执行，占位符找不到匹配值则保留原文 |
| **嵌套子组** | 仅 services.yaml 可声明嵌套结构 | Docker/K8s 发现的子组通过 `parent` 标注，合并进配置文件已声明的嵌套树中 |
| **Widget 单复数** | widgets 数组 + widget 单对象 → 合并为 widgets | widget 追加到数组末尾（`cleanServiceGroups`） |

---

## 九、常见配置陷阱

1. **Docker 服务与 YAML 服务同名不同组**：两个来源的服务会被按 `group` 分组（各自在不同组），不会冲突。
2. **Docker 服务 `weight` 默认为 0**：会排在配置文件服务之前，若想调整顺序需显式在标签中设置 `homepage.weight`。
3. **YAML 解析失败静默降级**：`servicesResponse()` 中任何一个来源异常都会被 catch 并降级为空数组，但 `/api/validate` 会返回详细错误。
4. **`{{HOMEPAGE_VAR_*}}` 不匹配留原值**：如果环境变量未设置，占位符字符串会被保留为字面值，可能引发后续解析问题。
5. **layout-only 组**：只在 `settings.layout` 中声明但无任何服务的组，会被 `pruneEmptyGroups()` 最终裁掉（除非其内部嵌套子组有服务）。
6. **原型污染防护**：`shvl.set()` 会拒绝 `__proto__`、`constructor`、`prototype` 路径，避免安全漏洞。
