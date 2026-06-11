# layout 空组合并/清理代码分析（整理版）

本文档聚焦 servicesResponse 中「只在 settings.layout 声明、没有实际服务」的三类场景，对比其代码执行路径与最终结果，所有结论均附有行号依据。

---

## 一、执行顺序核心依据

代码位置：[api-response.js L158-L256](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L158-L256)

**关键事实：`mergedGroupsNames` 在 `mergeLayoutGroupsIntoConfigured` 之前生成**

| 代码行 | 执行内容 | 影响 |
|--------|---------|------|
| L164-L189 | 加载三路服务数据（docker、k8s、services.yaml） | 得到 `discoveredDockerServices`、`discoveredKubernetesServices`、`configuredServices` |
| L191-L197 | 加载 `initialSettings` | 得到 settings.layout |
| **L199-L207** | **生成 `mergedGroupsNames`**<br>合并三路来源的组名去重<br>`[docker组名, k8s组名, configuredServices组名].flat()` | **此时 configuredServices 还未包含 layout 空组**<br>纯 layout 空组名不在此列表中 |
| L211 | `definedLayouts = Object.keys(initialSettings.layout)` | 取 layout 中声明的所有组名 |
| **L212-L218** | **`convertLayoutGroupToGroup` + `mergeLayoutGroupsIntoConfigured`**<br>把 layout 每个 key 转为空组结构 `{name, services:[], groups:[...]}`<br>然后 push 进 configuredServices | 此时 configuredServices 才包含 layout 空组<br>但 mergedGroupsNames 已生成，不再更新 |
| **L220-L252** | **主循环 `mergedGroupsNames.forEach`**<br>对每个组名做三路合并 + layout 排序 | 纯 layout 空组名不在此列表，不会进入循环 |
| L254 | `allGroups = [...sortedGroups.filter(g=>g), ...unsortedGroups]` | 合并排序/未排序组 |
| L255 | `pruneEmptyGroups(allGroups)` | 递归清理空组 |

**结论**：`mergeLayoutGroupsIntoConfigured`（L217）确实把 layout 空组推入 `configuredServices`，但 `mergedGroupsNames`（L199-L207）已生成完毕，纯 layout 空组名**不在主循环名单内**。

---

## 二、`findGroupByName` 的 parent 标记机制

代码位置：[service-helpers.js L710-L725](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/service-helpers.js#L710-L725)

```js
export function findGroupByName(groups, name) {
  for (let i = 0; i < groups.length; i += 1) {
    const group = groups[i];
    if (group.name === name) {
      return group;                  // 顶层命中，不设 parent
    } else if (group.groups) {
      const foundGroup = findGroupByName(group.groups, name);
      if (foundGroup) {
        foundGroup.parent = group.name;  // 在返回前把父组名打到子组对象上
        return foundGroup;
      }
    }
  }
  return null;
}
```

- `parent` 字段**直接写在** `configuredServices` 内的对象上
- 只有当子组在 `configuredServices` 的嵌套结构中被找到时才会打上
- 决定了主循环中子组走 `configuredGroup.parent` 分支

---

## 三、三类场景逐行对比

### 场景 A：纯单层 layout 空组

**配置**：
```yaml
# services.yaml（空或只有不相关组）
# settings.yaml
layout:
  OnlyInLayout: {}
```

**逐行追踪**：

| 代码行 | 关键变量状态 |
|--------|------------|
| L184 | `configuredServices = []` |
| L199-L207 | `mergedGroupsNames = []`（不含 OnlyInLayout） |
| L211 | `definedLayouts = ["OnlyInLayout"]` |
| L214-L216 | `layoutGroups = [{ name: "OnlyInLayout", services: [], groups: [] }]` |
| L217 | `mergeLayoutGroupsIntoConfigured` 执行后<br>`configuredServices = [{name:"OnlyInLayout", services:[], groups:[]}]` |
| L220 | `mergedGroupsNames.forEach` → **循环 0 次**（OnlyInLayout 不在名单中） |
| L254 | `allGroups = []`（sortedGroups 和 unsortedGroups 均为空） |
| L255 | `pruneEmptyGroups([])` → `[]` |
| 最终结果 | `[]`，连空壳都不留下 |

**结论**：纯 layout 空组**不会进入主循环，也不会出现在最终结果里**。没有服务来源牵引时完全消失。

---

### 场景 B：父子全空嵌套 layout 组

**配置**：
```yaml
# services.yaml（空）
# settings.yaml
layout:
  Parent:
    Child: {}   # Parent 和 Child 均无实际服务
```

**逐行追踪**：

| 代码行 | 关键变量状态 |
|--------|------------|
| L184 | `configuredServices = []` |
| L199-L207 | `mergedGroupsNames = []` |
| L214-L216 | `layoutGroups = [{ name: "Parent", services:[], groups:[{name:"Child", services:[], groups:[]}] }]` |
| L217 | `configuredServices = [Parent（含 Child 空组嵌套）]` |
| L220 | `mergedGroupsNames.forEach` → **循环 0 次** |
| L254 | `allGroups = []` |
| L255 | `pruneEmptyGroups([])` → `[]` |
| 最终结果 | `[]` |

**结论**：和单层空组一样，完全消失。没有任何服务牵引时，所有嵌套空结构都不会进入主循环。

---

### 场景 C：被实际服务牵引的嵌套 layout 组

子组有实际服务（来自 docker/k8s/services.yaml），父组可能只在 layout 中声明空壳。有三种子场景，核心路径一致。

#### 场景 C-1：子组被 Docker 发现，父组在 services.yaml 中定义

对应测试用例：[api-response.test.js L166-L193](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.test.js#L166-L193)

**配置**：
```yaml
# services.yaml
- Root: []    # 空组
# docker 容器 label: homepage.group=Child  # Child 有一个服务
# settings.yaml
layout:
  Root:
    Top:      # layout 定义 Root → Top → Child 三层嵌套
      Child: {}
```

**逐行追踪**：

| 代码行 | 关键变量状态 |
|--------|------------|
| L165 | `discoveredDockerServices = [{ name:"Child", services:[{name:"svc"}], groups:[] }]` |
| L184 | `configuredServices = [{name:"Root", services:[], groups:[]}]` |
| L199-L207 | `mergedGroupsNames = ["Child", "Root"]` |
| L214-L216 | `layoutGroups = [{ name:"Root", services:[], groups:[{name:"Top", services:[], groups:[{name:"Child", services:[], groups:[]}]}] }]` |
| L217 | merge 后 `configuredServices[0] = {name:"Root", services:[], groups:[{name:"Top", services:[], groups:[{name:"Child", services:[], groups:[]}]}]` |
| L220 | 第一次遍历 "Child" |
| L227 | `findGroupByName(configuredServices, "Child")` → 深度搜索找到 Child，设置 `Child.parent = "Top"` |
| L229-L235 | `mergedGroup = {name:"Child", services:[docker 的 svc], groups:[]}` |
| L238 | `definedLayouts.findIndex("Child") = -1`（Child 不在顶层 layout key） |
| L240 | `configuredGroup.parent = "Top"`（非空）→ 走嵌套分支 |
| L242 | `mergeSubgroups(configuredServices, mergedGroup)` → 找到 Child，设置其 services = [svc] |
| L244 | `ensureParentGroupExists` 递归回溯：<br>1. parentGroupName = "Top"<br>2. findGroupByName 找到 Top，设置 `Top.parent = "Root"`<br>3. 递归 ensureParentGroupExists<br>4. parentGroupName = "Root"<br>5. findGroupByName 找到 Root（无父）<br>6. `definedLayouts.findIndex("Root") = 0`<br>7. `sortedGroups[0] = Root`（完整嵌套结构） |
| L220 | 第二次遍历 "Root" |
| L227 | `configuredGroup = findGroupByName(configuredServices, "Root")`（顶层命中，无 parent） |
| L229-L235 | `mergedGroup = { name: "Root", services:[], groups:[Top（已含 Child 有服务）] }` |
| L238 | `layoutIndex = 0` → `sortedGroups[0] = mergedGroup`（覆盖上面写入的 Root） |
| L254 | `allGroups = [Root（完整嵌套结构）]` |
| L255 | `pruneEmptyGroups`：Root.groups=[Top].length>0 → 保留；Top.groups=[Child].length>0 → 保留；Child.services 非空 → 保留 |
| 最终结果 | `[{name:"Root", groups:[{name:"Top", groups:[{name:"Child", services:[svc]}]}] }]` ✅ |

测试用例断言 `expect(groups.map((g) => g.name)).toEqual(["Root"])` 和 `expect(groups[0].groups[0].groups[0].name).toBe("Child")` 完美命中。

#### 场景 C-2：子组被 Docker 发现，父组只在 layout 中声明空壳

**配置**（与 C-1 唯一区别：services.yaml 为空）：
```yaml
# services.yaml（空）
# docker 容器 label: homepage.group=Child
# settings.yaml
layout:
  Root:
    Top:
      Child: {}
```

**逐行追踪差异点**：

| 代码行 | 关键变量状态 |
|--------|------------|
| L184 | `configuredServices = []` |
| L199-L207 | `mergedGroupsNames = ["Child"]`（只有 docker 的 Child） |
| L217 | merge 后 `configuredServices = [Root（完整嵌套空壳）]` |
| L220 | 遍历 "Child"，流程同 C-1，`configuredGroup.parent = "Top"` |
| L244 | `ensureParentGroupExists` → `sortedGroups[0] = Root`（来自 configuredServices[0]） |
| L220 | 无第二次遍历（Root 不在 mergedGroupsNames） |
| L254 | `allGroups = [Root]` → 同 C-1 一样保留完整嵌套 ✅ |

**结论**：父组即使完全只在 layout 中定义空壳，只要内层有服务牵引，**整个嵌套结构都能保留**。

#### 场景 C-3：子组在 services.yaml 中嵌套定义（与 layout 结构一致）

对应测试用例：[api-response.test.js L195-L235](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.test.js#L195-L235)

**配置**：
```yaml
# services.yaml
- Top:
    - Child: []   # services.yaml 已有嵌套空壳
# docker 发现 name:Child 有服务
# settings.yaml
layout:
  Top:
    Child: {}
```

与 C-1 类似，区别是 services.yaml 中 Top 已有 Child 空壳。`mergeLayoutGroupsIntoConfigured` 找到同名 Top 已存在 → 递归 merge 子组，Child 已存在则不再重复 push。最终流程和 C-1 一致，测试验证完整保留。

---

## 四、三类场景汇总表

| 场景 | 能否进入主循环 | 最终结果 | 关键原因 |
|------|---------------|---------|----------|
| A. 单层纯空组 | ❌ 不能 | ❌ 完全消失 | `mergedGroupsNames` 在 merge 前生成，纯 layout 空组名不在其中 |
| B. 多层全空嵌套 | ❌ 不能 | ❌ 完全消失 | 和 A 同样原因 |
| C-1. 内层有服务 + 父在 services.yaml 中 | ✅ 子组名在 docker/k8s/configured 中 | ✅ 完整嵌套保留 | `findGroupByName` 找到带子组并打 `parent` → `mergeSubgroups` 填服务 → `ensureParentGroupExists` 回溯挂父组进 sortedGroups |
| C-2. 内层有服务 + 父组只在 layout 中 | ✅ 子组名进入 | ✅ 完整嵌套保留 | `ensureParentGroupExists` 从 configuredServices 中找到 merge 进来的 layout 空壳，推入 sortedGroups |
| C-3. 内层有服务 + services.yaml 有嵌套 | ✅ 子组名进入 | ✅ 完整嵌套保留 | 和 C-1 相同路径 |

---

## 五、书签路径的空组行为

代码位置：[bookmarksResponse()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L27-L71)

**书签响应完全不具备空组构造能力**：
- 没有 `convertLayoutGroupToGroup`
- 没有 `mergeLayoutGroupsIntoConfigured`
- 没有 `pruneEmptyGroups`

书签只遍历 `bookmarksArray`（来自 bookmarks.yaml 的真实内容）。如果某个组名只在 layout 中声明、YAML 中不存在：
1. 书签响应里根本不会出现该组
2. 前端 [index.jsx L301-L303](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L301-L303) 的 `layoutGroups` 中，`services.find(name)` 找不到，`bookmarks.find(name)` 也找不到 → 返回 undefined
3. 被 `tabGroupFilter` 的 `g && ...` 条件过滤掉 → 不渲染

**结论**：书签 + 纯 layout 空组，无论服务端还是客户端都彻底不存在。

---

## 六、`pruneEmptyGroups` 的潜在缺陷

代码位置：[api-response.js L123-L134](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L123-L134)

```js
function pruneEmptyGroups(groups) {
  return groups.filter((group) => {
    if (group.services.length === 0 && group.groups.length === 0) return false;  // 入口判空
    if (group.groups.length > 0) {
      group.groups = pruneEmptyGroups(group.groups);  // 递归，可能把 groups 清空
    }
    return true;  // ⚠️ 递归后不再重新判空
  });
}
```

**缺陷逻辑**：父组 `services=[]`、`groups=[Child]` 进入过滤 → 入口判空通过（groups.length>0）→ 递归清理 Child（假设 Child 全空被过滤）→ `group.groups = []` → 直接 return true → **父组成为 services 和 groups 都空的幽灵组**。

**触发条件**：外层父组已进入 allGroups（有服务牵引进入主循环），但内层嵌套子组在递归中被全部清空。

**纯空组场景不会触发**：场景 A、B 根本进不了 allGroups，pruneEmptyGroups 没有输入。只有在「外层有服务牵引进入 allGroups + 内层嵌套全空」的构造场景下才可能出现。
