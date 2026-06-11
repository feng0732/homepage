# 书签与服务布局边界分析（补充文档）

本文档补充分析三个配置驱动分组的边界场景：同名分组取舍、layout 空组合并/清理、tab 过滤对未配置分组的影响。

---

## 一、服务组与书签组同名时的页面取舍

### 1.1 冲突点：layout-groups 区块中的 `??` 优先级

核心代码位置：[index.jsx L301-L303](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L301-L303)

```js
const layoutGroups = Object.keys(settings.layout ?? {})
  .map((groupName) => services?.find((g) => g.name === groupName) ?? bookmarks?.find((b) => b.name === groupName))
  .filter(tabGroupFilter);
```

查找顺序是 **services 先查，找不到再查 bookmarks**（`??` 空值合并）。如果同名组在 services 响应中存在（即使是一个没有任何服务的空壳），书签中的对应组就被完全跳过。

组类型判别代码位置：[index.jsx L335-L355](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L335-L355)

```js
{layoutGroups.map((group) =>
  group.services ? (         // 数组 [] 也是 truthy
    <ServicesGroup ... />
  ) : (
    <BookmarksGroup ... />
  ),
)}
```

检查条件是 `group.services` 是否 falsy。由于服务组结构恒带 `services: []`（空数组），**空服务组也走 ServicesGroup 分支**。

### 1.2 同名书签组是否会落到 #bookmarks 区块？

**不会。**

`#services` / `#bookmarks` 区块使用的过滤器 [index.jsx L299-L311](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L299-L311)：

```js
const undefinedGroupFilter = (g) => settings.layout?.[g.name] === undefined;

const serviceGroups = services?.filter(tabGroupFilter).filter(undefinedGroupFilter);
const bookmarkGroups = bookmarks.filter(tabGroupFilter).filter(undefinedGroupFilter);
```

同名组 X 已在 `settings.layout[X]` 中声明 → `undefinedGroupFilter(X) === false` → **不会进入两个兜底区块**。书签组被丢进 `layoutGroups` 查找，但被 `services.find` 抢先命中，于是书签组的书签就再也没有渲染出口——**实际丢失了。**

### 1.3 场景汇总表

| 场景（services.yaml 和 bookmarks.yaml 都有同名组 Media，layout 也声明了 `Media`） | 页面结果 |
|-------------------------------------------------------------------------|---------|
| services 有服务，bookmarks 有书签 | 只显示 ServicesGroup（含服务），**书签丢失** |
| services 无服务（空组），bookmarks 有书签 | 显示空 ServicesGroup，**书签丢失** |
| services.yaml 中删去 Media，bookmarks 有书签 | 走 `??` 后半段，正常显示 BookmarksGroup |
| settings.layout 中不声明 Media，两边都有 | services 的 Media 显示在 #services，bookmarks 的 Media 显示在 #bookmarks，**各显示各的，不合并** |

**根本原因**：[bookmarksResponse()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L27-L71) 与 [servicesResponse()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L158-L256) 是两条完全独立的数据流，后端没有任何同名合并逻辑，冲突全部推给前端的 `??` 做单边取舍。

---

## 二、layout 只声明空组时的合并与清理

「layout 只声明空组」指 settings.layout 中定义了分组名，但 services.yaml / bookmarks.yaml / docker / k8s 中都没有对应实际内容的情况。书签和服务路径处理完全不同。

### 2.1 服务路径：servicesResponse() 的五阶段处理

涉及函数（都在 [api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js)）：

| 阶段 | 函数 | 作用 |
|------|------|------|
| 1 | [convertLayoutGroupToGroup()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L87-L97) | 把 settings.layout 每个 key 转成空组结构 `{name, services:[], groups:[嵌套空组]}` |
| 2 | [mergeLayoutGroupsIntoConfigured()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L136-L156) | 将空组 push 进 configuredServices；若同名已存在则只合并子组结构 |
| 3 | mergedGroupsNames 收集 | configuredServices 已含空组，因此 **空组名一定出现在合并循环中** |
| 4 | merge + sort | mergedGroup 中 services 是空数组，groups 只有嵌套空组 |
| 5 | [pruneEmptyGroups()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L123-L134) | 递归清理空组 |

#### convertLayoutGroupToGroup 的构造逻辑（L87-L97）

```js
function convertLayoutGroupToGroup(name, layoutGroup) {
  const group = { name, services: [], groups: [] };
  if (layoutGroup) {
    Object.entries(layoutGroup).forEach(([key, value]) => {
      if (typeof value === "object") {
        group.groups.push(convertLayoutGroupToGroup(key, value));  // 递归
      }
    });
  }
  return group;
}
```

只遍历 `typeof value === "object"` 的 key（即子组），**忽略 tab/style/columns/iconsOnly 等非对象属性**。

#### pruneEmptyGroups 的行为与潜在缺陷（L123-L134）

```js
function pruneEmptyGroups(groups) {
  return groups.filter((group) => {
    // 入口判空（用递归清理子组之前的 groups.length）
    if (group.services.length === 0 && group.groups.length === 0) return false;
    if (group.groups.length > 0) {
      group.groups = pruneEmptyGroups(group.groups);   // 递归，可能把 groups 变空
    }
    return true;   // ⚠️ 递归后不再重新判空
  });
}
```

**缺陷**：父组初始 services=[]，groups=[Child]，进入递归后 Child 被清理（groups=[]），此时父组 groups 也变 []，但 filter 已越过判空分支，直接 return true → **父组成为一个 services 空、groups 也空的幽灵壳组被保留下来**。只要嵌套深度 ≥ 2 就可能触发。

#### 测试用例佐证

[api-response.test.js L106-L133](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.test.js#L106-L133)：
```js
// layout: { GroupA: {}, GroupB: {} }
// services.yaml 含 Empty: {services:[], groups:[]}
expect(groups.map((g) => g.name)).toEqual(["GroupA"]);
// GroupB 和 Empty 都被 pruneEmptyGroups 正确移除了（无嵌套 → 未触发缺陷）
```

### 2.2 书签路径：bookmarksResponse() 对空组的处理

[bookmarksResponse()](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L27-L71) 没有 `convertLayoutGroupToGroup`、没有 `mergeLayoutGroupsIntoConfigured`、没有 `pruneEmptyGroups`。

它只遍历 `bookmarksArray`（来自 bookmarks.yaml 真实内容），如果某个组名只在 layout 中声明、YAML 中不存在 → 永远不会进入 sortedGroups/unsortedGroups，最终 [L70](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/utils/config/api-response.js#L70) 的 `sortedGroups.filter((g) => g)` 会把空位置（undefined）过滤掉。

**结论**：书签路径下 layout 只声明空组 → **书签响应里根本不会出现该组**，在前端 #layout-groups 中 `services.find` 找不到，`bookmarks.find` 也找不到，map 返回 undefined，被 tabGroupFilter 的 `g && ...` 条件过滤掉。

### 2.3 清理行为汇总表

| 场景 | services 路径结果 | bookmarks 路径结果 |
|------|-----------------|-------------------|
| layout 声明单层空组 {Empty:{}}，无任何服务 | ✂️ 被 pruneEmptyGroups 清除 | 🚫 根本不在响应中 |
| layout 声明嵌套空组 {Parent:{Child:{}}}，无服务 | ⚠️ Parent 可能因 pruneEmptyGroups 缺陷保留为壳组 | 🚫 根本不在响应中 |
| layout 声明 Parent:{Child:{}}，Child 有 Docker 发现的服务 | ✅ 整个链路保留（Child.services 非空） | 🚫 根本不在响应中 |
| 只在 services.yaml 写空组，不在 layout 声明 | ✂️ 被 pruneEmptyGroups 清除 | — |

---

## 三、tab 过滤对未配置分组的影响

### 3.1 slugifyAndEncode 的 undefined 兜底

[components/tab.jsx L9-L11](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/tab.jsx#L9-L11)：

```js
export function slugifyAndEncode(tabName) {
  return tabName !== undefined ? encodeURIComponent(slugify(tabName)) : "";
}
```

**tabName 为 undefined 时返回空字符串 `""`。** 测试用例 [tab.test.jsx L13](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/components/tab.test.jsx#L13) 也验证了 `expect(slugifyAndEncode(undefined)).toBe("")`。

### 3.2 tabGroupFilter 真值表

[index.jsx L298](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L298)：

```js
const tabGroupFilter = (g) => g && [activeTab, ""].includes(slugifyAndEncode(settings.layout?.[g.name]?.tab));
```

白名单数组是 `[activeTab, ""]`，命中条件为「等于当前激活 tab」或「等于空字符串」。

| 分组类型 | `settings.layout?.[g.name]?.tab` | `slugifyAndEncode` 返回 | 是否命中 `[activeTab, ""]` | 显示情况 |
|---------|----------------------------------|------------------------|--------------------------|---------|
| 在 layout 中声明 tab:"Tab1" | `"Tab1"` | `"tab1"` | 仅当 `activeTab === "tab1"` | 只在 Tab1 下可见 |
| 在 layout 中声明但**无 tab 配置** | `undefined` | `""` | **永远命中**（""恒在数组中） | **所有 Tab 下都可见** |
| **未配置分组**（不在 layout 中声明） | `undefined`（layout 无此 key） | `""` | **永远命中** | **所有 Tab 下都可见** |

### 3.3 三层过滤链解读

[index.jsx L297-L405](file:///d:/fz/0601/solo-dogfeeding/code/200-homepage/src/pages/index.jsx#L297-L405) 的三个区块各有独立的过滤链：

```
#layout-groups 区块
  Object.keys(settings.layout)
    → map: services.find(name) ?? bookmarks.find(name)   // 布局定义的每个 key 找一次数据
    → filter: tabGroupFilter                             // 按 tab 过滤

#services 区块
  services[]
    → filter: tabGroupFilter          // 第一步：按 tab 过滤（未配置的永远通过）
    → filter: undefinedGroupFilter    // 第二步：只保留 layout 中没声明的

#bookmarks 区块
  bookmarks[]
    → filter: tabGroupFilter          // 同上
    → filter: undefinedGroupFilter    // 同上
```

**结论**：tabGroupFilter 对未配置分组来说相当于"透明的"——它永远放行。真正决定未配置分组显示位置的是 `undefinedGroupFilter`，它确保未配置组不会出现在 #layout-groups，而只出现在各自的 #services / #bookmarks 兜底区块。

### 3.4 一个容易踩坑的配置场景

配置如下：

```yaml
# settings.yaml
layout:
  Media:          # 在 layout 中声明了，但没有写 tab:
    style: row
  Download:
    tab: Tab2

# services.yaml 中还有 3 个组：Media, Tools, Download（和 layout 对应）
# bookmarks.yaml 中还有：Dev-Links（未在 layout 声明）
```

用户本意可能是 Media 属于「无 Tab 分类的默认区」。实际行为：

| 组 | 配置 | tab 归属 | 切到 Tab2 时是否显示 | 切到 # 时（即空 hash）是否显示 |
|----|------|---------|-------------------|------------------------|
| Media | layout 中声明，无 tab | `""`（永远显示） | ✅ 显示在 #layout-groups | ✅ 显示 |
| Download | layout 中声明 tab:Tab2 | `"tab2"` | ✅ 显示在 #layout-groups | ❌ 不显示 |
| Tools | 未在 layout 声明 | `""`（永远显示） | ✅ 显示在 #services | ✅ 显示在 #services |
| Dev-Links | 书签，未在 layout 声明 | `""`（永远显示） | ✅ 显示在 #bookmarks | ✅ 显示在 #bookmarks |

用户激活 Tab2 Tab 后会看到：#layout-groups 里 Media + Download，#services 里 Tools，#bookmarks 里 Dev-Links。**Media 和 Download 在视觉上混在一起，没有分区边界提示 Media 实际是「所有 Tab 都显示」的默认组。**

如果本意是让 Media 只在默认（无 Tab）下显示，目前代码中没有对应配置——**只要没有配置 tab，就被视为「所有 Tab 下都显示」**。要实现默认 Tab 独占只能给其他所有分组都配 tab，然后让默认 hash（空字符串或 `/`）时显示这些 tab 为 "" 的分组。
