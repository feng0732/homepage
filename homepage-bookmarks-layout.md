# 书签与服务布局代码分析

本文档按代码执行顺序，分析 Homepage 项目中书签（Bookmarks）和服务（Services）的分组配置、排序规则以及页面布局生效过程。

---

## 一、配置文件结构（YAML 输入）

### 1.1 书签配置 bookmarks.yaml

文件位置：[bookmarks.yaml](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/skeleton/bookmarks.yaml)

```yaml
- Developer:           # 分组名（第一层 key）
    - Github:          # 书签名（第二层 key）
        - abbr: GH     # 书签属性在第三层，以数组包裹
          href: https://github.com/
```

**注意**：书签的属性值被包裹在一个数组元素内（`- abbr: GH`），这与 services 不同。

### 1.2 服务配置 services.yaml

文件位置：[services.yaml](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/skeleton/services.yaml)

```yaml
- My First Group:          # 分组名（第一层 key）
    - My First Service:    # 服务名（第二层 key）
        href: http://localhost/    # 服务属性直接在第二层对象上
        description: Homepage is awesome
```

**注意**：服务的属性直接挂在服务名对应的对象上，没有额外的数组包裹。

---

## 二、服务端配置加载与处理

### 2.1 配置文件检查与环境变量替换

文件位置：[config.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/config.js)

**`checkAndCopyConfig(config)`**（L15-L50）：
- 确认 `config/` 目录存在，不存在则创建
- 若目标 yaml 不存在，从 `src/skeleton/` 复制骨架文件
- 尝试 yaml.load 解析，失败则返回错误对象

**`substituteEnvironmentVars(str)`**（L64-L80）：
- 扫描字符串中的 `{{HOMEPAGE_VAR_XXX}}` 和 `{{HOMEPAGE_FILE_XXX}}`
- 分别替换为环境变量值或文件内容
- 使用 `memory-cache` 缓存环境变量列表

**`getSettings()`**（L82-L103）：
- 读取 `settings.yaml`
- **关键**：对 `layout` 字段做兼容处理——如果是数组，转成对象格式：
  ```js
  // 数组格式：[{ "Group A": {...} }, { "Group B": {...} }]
  // 转成对象格式：{ "Group A": {...}, "Group B": {...} }
  ```
  这个对象的 key 顺序就是分组的显示顺序。

---

### 2.2 服务分组解析 parseServicesToGroups

文件位置：[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/service-helpers.js)

**`parseServicesToGroups(services)`**（L17-L51）：

将 YAML 嵌套对象转换为易用的 JS 数组结构：

```
输入 YAML 结构:
[
  { "Group Name": [
    { "Service 1": { href, description, ... } },
    { "Sub Group": [ ... 嵌套子组 ... ] }   // 当值是数组时递归
  ]}
]

输出 JS 结构:
[
  {
    name: "Group Name",
    type: "group",
    services: [
      { name: "Service 1", href, description, weight: N*100, type: "service" }
    ],
    groups: [ ... 递归嵌套子组 ... ]
  }
]
```

**weight 默认值逻辑**（L39）：
```js
weight: entries[entryName].weight ?? (serviceGroupServices.length + 1) * 100
```
- 用户未指定时，按在 YAML 中出现的顺序，依次分配 100、200、300...
- 这保证了原始书写顺序的稳定性

服务来源有三个：
| 来源 | 函数 | 说明 |
|------|------|------|
| 配置文件 | `servicesFromConfig()` | 解析 services.yaml |
| Docker 发现 | `servicesFromDocker()` | 读取容器 label `homepage.*` |
| Kubernetes 发现 | `servicesFromKubernetes()` | 扫描 Ingress/HTTPRoute annotation |

---

### 2.3 书签响应 bookmarksResponse

文件位置：[api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js)

**`bookmarksResponse()`**（L27-L71）：

```
流程:
1. 读取 + 环境变量替换 + yaml.load 解析 bookmarks.yaml
2. 数据结构转换（L48-L54）:
   bookmarks.map(group => ({
     name: Object.keys(group)[0],                          // 分组名
     bookmarks: group[Object.keys(group)[0]].map(entries => ({
       name: Object.keys(entries)[0],                      // 书签名
       ...entries[Object.keys(entries)[0]][0],             // 取数组[0]展开属性
     })),
   }))
3. 按 settings.layout 排序（L56-L70）:
   - definedLayouts = Object.keys(initialSettings.layout)
   - 遍历每个书签组：
     - 若组名在 definedLayouts 中，放入 sortedGroups[layoutIndex]
     - 否则追加到 unsortedGroups
   - 最终结果：[...sortedGroups.filter(g=>g), ...unsortedGroups]
```

**书签无内部排序**：每个分组内的书签保持 YAML 原始顺序，没有 weight 或 name 排序。

---

### 2.4 服务响应 servicesResponse（核心）

文件位置：[api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js)

**`servicesResponse()`**（L158-L256）——整体流程：

```
Step 1: 加载三路服务数据
  ├─ discoveredDockerServices     = cleanServiceGroups(servicesFromDocker())
  ├─ discoveredKubernetesServices = cleanServiceGroups(servicesFromKubernetes())
  └─ configuredServices           = cleanServiceGroups(servicesFromConfig())
  同时加载 initialSettings = getSettings()

Step 2: 收集所有分组名（去重）
  mergedGroupsNames = [...new Set([docker组名, k8s组名, 配置组名].flat())]

Step 3: 如果 settings.layout 存在，处理"仅在 layout 中定义但不在 services.yaml 中的空组"
  layoutGroups = Object.entries(initialSettings.layout)
    .map(([key, value]) => convertLayoutGroupToGroup(key, value))
  mergeLayoutGroupsIntoConfigured(configuredServices, layoutGroups)

Step 4: 遍历每个分组名，合并 + 排序（L220-L252）
  对每个 groupName:
    a) 分别从 docker/k8s/config 三路找该组（找不到就给空数组）
    b) mergedGroup = {
         name: groupName,
         services: [...docker服务, ...k8s服务, ...配置服务]
                     .filter(Boolean)
                     .sort(compareServices),    // ★ 服务内排序
         groups: [...configuredGroup.groups],
       }
    c) 按 settings.layout 决定分组排序位置：
         layoutIndex = definedLayouts.findIndex(名匹配)
         if (layoutIndex > -1) → sortedGroups[layoutIndex] = mergedGroup
         else if (有 parent 嵌套组) → mergeSubgroups + ensureParentGroupExists
         else → unsortedGroups.push(mergedGroup)

Step 5: 拼接结果 + 清理空组
  allGroups = [...sortedGroups.filter(g=>g), ...unsortedGroups]
  return pruneEmptyGroups(allGroups)
```

**服务内排序 compareServices**（L19-L25）：
```js
function compareServices(service1, service2) {
  const comp = service1.weight - service2.weight;
  if (comp !== 0) return comp;                    // 先按 weight 升序
  return service1.name.localeCompare(service2.name);  // weight 相同按名称字典序
}
```

**空组清理 pruneEmptyGroups**（L123-L134）：递归移除没有 services 且没有 groups 的空分组。

---

### 2.5 API 路由层

书签接口：[pages/api/bookmarks.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/api/bookmarks.js)
```js
export default async function handler(req, res) {
  res.send(await bookmarksResponse());
}
```

服务接口：[pages/api/services/index.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/api/services/index.js)
```js
export default async function handler(req, res) {
  res.send(await servicesResponse());
}
```

---

## 三、前端页面渲染流程（index.jsx）

文件位置：[pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx)

### 3.1 服务端预取 getStaticProps（L55-L95）

```js
export async function getStaticProps() {
  const settings = getSettings();
  const services = await servicesResponse();   // 构建时就取好
  const bookmarks = await bookmarksResponse();
  const widgets = await widgetsResponse();
  return {
    props: {
      initialSettings: settings,
      fallback: {                   // 传给 SWR 的 fallback，避免首屏再请求
        "/api/services": services,
        "/api/bookmarks": bookmarks,
        "/api/widgets": widgets,
      },
      ...serverSideTranslations(language),
    },
  };
}
```

### 3.2 客户端数据获取（L97-L192）

```js
function Index({ initialSettings, fallback }) {
  return (
    <SWRConfig value={{ fallback, fetcher: (url) => fetch(url).then(r => r.json()) }}>
      <ErrorBoundary>
        <Home initialSettings={initialSettings} />
      </ErrorBoundary>
    </SWRConfig>
  );
}
```

在 Home 组件中用 useSWR 获取（首屏直接用 fallback，不发请求）：
```js
const { data: services } = useSWR("/api/services");
const { data: bookmarks } = useSWR("/api/bookmarks");
const { data: widgets } = useSWR("/api/widgets");
```

### 3.3 Tab 分组逻辑（L279-L311）

```js
const tabs = useMemo(() => [
  ...new Set(
    Object.keys(settings.layout ?? {})
      .map((groupName) => settings.layout[groupName]?.tab?.toString())
      .filter(Boolean),
  ),
], [settings.layout]);
```

- Tab 的来源：遍历 `settings.layout` 中每个分组配置的 `tab` 字段
- 相同 tab 值的分组会被归到同一个 Tab 下
- Tab 过滤函数：
  ```js
  const tabGroupFilter = (g) => g && [activeTab, ""].includes(slugifyAndEncode(settings.layout?.[g.name]?.tab));
  ```
  - 组的 tab 等于当前激活 tab → 显示
  - 组没有配置 tab（空字符串）→ 所有 Tab 下都显示

### 3.4 三段式分组渲染（L297-L405）

**`servicesAndBookmarksGroups`** 是核心渲染输出，分三部分渲染：

| 区块 | 数据源 | 过滤条件 |
|------|--------|----------|
| `#layout-groups` | 遍历 `settings.layout` 的 key，去 services/bookmarks 中查找对应组 | 通过 `tabGroupFilter` 按 Tab 过滤 |
| `#services` | `services` 数组中未在 `settings.layout` 中定义的组 | `undefinedGroupFilter`：`settings.layout?.[g.name] === undefined` |
| `#bookmarks` | `bookmarks` 数组中未在 `settings.layout` 中定义的组 | 同上 |

也就是说：
- **在 settings.layout 中有定义的组** → 在 `#layout-groups` 区块按 layout 的 key 顺序渲染
- **没有在 settings.layout 中定义的组** → 分别在 `#services` 和 `#bookmarks` 区块按服务端返回的顺序渲染

渲染 ServicesGroup / BookmarksGroup 时，传入 `settings.layout?.[group.name]` 作为该组的专属布局配置。

---

## 四、书签组件渲染链

组件层级：`BookmarksGroup` → `List` → `Item`

### 4.1 BookmarksGroup（分组容器）

文件：[components/bookmarks/group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/bookmarks/group.jsx)

**分组外层宽度（响应式）**（L26-L32）：
```js
layout?.style === "row"
  ? "basis-full"                                            // row 模式：独占一行
  : "basis-full md:basis-1/4 lg:basis-1/5 xl:basis-1/6"    // 列模式：响应式分栏
```
加上自定义最大列数（仅 >6 时生效）：
```js
layout?.style !== "row" && maxGroupColumns && parseInt(maxGroupColumns, 10) > 6
  ? `3xl:basis-1/${maxGroupColumns}`
  : ""
```

**折叠功能**：使用 `@headlessui/react` 的 `Disclosure` + `Transition`，支持：
- `layout.initiallyCollapsed` 或全局 `groupsInitiallyCollapsed` → 默认折叠
- `layout.header === false` → 隐藏标题栏
- `layout.icon` → 分组标题前的图标
- `disableCollapse` → 禁用折叠按钮

### 4.2 List（书签列表容器）

文件：[components/bookmarks/list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/bookmarks/list.jsx)

**三种列表样式**：
```js
// 1) iconsOnly / bookmarksStyle === "icons" → 图标网格
classes = "grid gap-2 bookmark-list";
style.gridTemplateColumns = "repeat(auto-fill, minmax(60px, 1fr))";

// 2) layout.style === "row" → 按 columns 配置的栅格
classes = `grid ${columnMap[layout?.columns]} gap-x-2`;

// 3) 默认 → 垂直列表
classes = "flex flex-col bookmark-list";
```

### 4.3 Item（单个书签卡片）

文件：[components/bookmarks/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/bookmarks/item.jsx)

- `iconOnly=true` → 60×60 方形图标/缩写
- 默认模式 → 左侧图标(abbr) + 中间名称 + 右侧描述(hostname)
- `target` 优先级：`bookmark.target` > `settings.target` > `"_blank"`

---

## 五、服务组件渲染链

组件层级：`ServicesGroup` → `List` → `Item`（并可嵌套 `ServicesGroup`）

### 5.1 ServicesGroup（分组容器，支持嵌套）

文件：[components/services/group.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/services/group.jsx)

**分组外层宽度（响应式）**（L31-L37）：
```js
layout?.style === "row"
  ? "basis-full"
  : "basis-full md:basis-1/2 lg:basis-1/3 xl:basis-1/4"   // 比书签更宽
```
加上自定义最大列数：
```js
layout?.style !== "row" && maxGroupColumns ? `3xl:basis-1/${maxGroupColumns}` : ""
```

**嵌套子组**（L89-L108）：
```jsx
{group.groups?.length > 0 && (
  <div className={`grid ${...}`}>
    {group.groups.map((subgroup) => (
      <ServicesGroup
        key={subgroup.name}
        group={subgroup}
        layout={layout?.[subgroup.name]}   // 子组从父组 layout 中按名取配置
        isSubgroup
      />
    ))}
  </div>
)}
```
子组通过 `isSubgroup` 标记，去掉 padding 并加上 `subgroup` CSS class。

### 5.2 List（服务列表容器）

文件：[components/services/list.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/services/list.jsx)

```js
layout?.style === "row"
  ? `grid ${columnMap[layout?.columns]} gap-x-2`
  : "flex flex-col"
```
逻辑同书签 List，但多了 `useEqualHeights` 等高参数传递给 Item。

### 5.3 Item（单个服务卡片）

文件：[components/services/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/services/item.jsx)

功能比书签丰富得多：
- **链接**：`service.href` 存在且非 `#` 才渲染 `<a>`
- **状态指示**（右上角）：
  - `service.ping` → [Ping](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/services/ping.jsx) 组件
  - `service.siteMonitor` → SiteMonitor
  - `service.container` + server → Docker Status
  - `service.app` → Kubernetes Status
  - `service.proxmoxNode` + proxmoxVMID → Proxmox Status
- **展开统计**：点击状态指示可展开 Docker/K8s/Proxmox 统计面板
- **等高布局**：`useEqualHeights` → `h-[calc(100%-0.5rem)]`
- **Widgets**：`service.widgets.map(w => <Widget widget={w} />)`

---

## 六、列布局映射 columnMap

文件：[utils/layout/columns.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/layout/columns.js)

```js
export const columnMap = [
  "grid-cols-1 md:grid-cols-1 lg:grid-cols-1",   // index 0/1: 1列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-2",   // index 2: 2列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-3",   // index 3: 3列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-4",   // index 4: 4列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-5",   // index 5: 5列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-6",   // index 6: 6列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-7",   // index 7: 7列
  "grid-cols-1 md:grid-cols-2 lg:grid-cols-8",   // index 8: 8列
];
```

使用方式：`columnMap[layout?.columns]`，即 `layout.columns` 直接作为数组索引取 Tailwind class。

注意 index 0 和 1 都是 1 列，所以传入 0/1/不传效果相同。

---

## 七、完整流程图（端到端）

```
┌─────────────────────────────────────────────────────────────────────┐
│                          构建时 (getStaticProps)                     │
│                                                                     │
│  settings.yaml ──► getSettings()                                    │
│                       ├─ layout 数组转对象（确定分组顺序）            │
│                       └─ 其他全局设置                                │
│                                                                     │
│  services.yaml ──► servicesFromConfig() ──┐                         │
│  docker.yaml   ──► servicesFromDocker()  ├─► servicesResponse()     │
│  k8s           ──► servicesFromK8s()    ─┘    ├─ 三路合并           │
│                                                ├─ 组内按 weight→name │
│                                                ├─ 组间按 layout 排序 │
│                                                └─ 空组清理           │
│                                                                     │
│  bookmarks.yaml ──► bookmarksResponse()                             │
│                       ├─ YAML → JS 结构转换                         │
│                       └─ 按 settings.layout 排序分组                 │
│                                                                     │
│  结果通过 SWR fallback 注入页面                                       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          客户端渲染 (Home)                           │
│                                                                     │
│  useSWR("/api/services")  ──► services 数组                         │
│  useSWR("/api/bookmarks") ──► bookmarks 数组                        │
│                                                                     │
│  settings.layout 中配置了 tab?                                      │
│    YES → 生成 Tab 栏，按 activeTab 过滤分组                          │
│    NO  → 所有组平铺显示                                              │
│                                                                     │
│  渲染三个区块：                                                      │
│  ├─ #layout-groups: settings.layout 中定义的组（按 key 顺序）        │
│  ├─ #services:      未在 layout 中定义的服务组                       │
│  └─ #bookmarks:     未在 layout 中定义的书签组                       │
│                                                                     │
│  每个组 → ServicesGroup / BookmarksGroup                             │
│             ├─ 外层 flex-basis 响应式分栏（1/2 / 1/3 / 1/4 / ...）  │
│             ├─ Disclosure 折叠头                                     │
│             └─ List → Item（row/column/icons 三种列表样式）         │
│                    ServicesGroup 还可递归嵌套子组                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键配置项速查

| 配置路径 | 作用 | 生效位置 |
|---------|------|---------|
| `settings.layout` 对象的 **key 顺序** | 决定分组显示顺序 | `api-response.js` L211/238，`index.jsx` L301 |
| `settings.layout["组名"].tab` | 分组所属 Tab | `index.jsx` L279-L288, L298 |
| `settings.layout["组名"].style` | `"row"` 横向栅格 / 默认纵向 | `group.jsx` L28, `list.jsx` L7/L10 |
| `settings.layout["组名"].columns` | row 模式下列数（1-8） | `list.jsx` → `columnMap[columns]` |
| `settings.layout["组名"].iconsOnly` | 书签仅显示图标 | `bookmarks/list.jsx` L9 |
| `settings.layout["组名"].header` | `false` 隐藏分组标题 | `group.jsx` L38/L42 |
| `settings.layout["组名"].icon` | 分组标题图标 | `group.jsx` L40/L44 |
| `settings.layout["组名"].initiallyCollapsed` | 分组默认折叠 | `group.jsx` L20/L35 |
| `settings.maxGroupColumns` | 3xl 断点下的最大列数（>6 书签生效，任意服务生效） | `group.jsx` L29-L31 / L34 |
| `settings.bookmarksStyle` | `"icons"` 全局书签图标模式 | `index.jsx` L383, `bookmarks/list.jsx` L9 |
| `settings.useEqualHeights` | 服务卡片等高 | `services/item.jsx` L40 |
| `settings.groupsInitiallyCollapsed` | 全部分组默认折叠 | `group.jsx` L20/L35 |
| `settings.disableCollapse` | 禁用分组折叠 | `group.jsx` L39/L43 |
| `services.yaml` 中服务的 `weight` | 组内服务排序（数字越小越前） | `api-response.js` L19-L25 |
