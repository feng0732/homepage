# 搜索栏与建议处理路径代码解析

本项目存在两套并行的搜索机制：**顶部信息栏 Search Widget** 和 **全局 QuickLaunch 模态框**。两者共享后端 API 路由 `/api/search/searchSuggestion` 以及核心的 `searchProviders` 配置对象，但在状态管理、交互方式和候选来源上存在显著差异。

---

## 一、核心文件一览

| 角色 | 文件路径 |
|------|----------|
| 搜索栏组件（Widget） | [search.jsx](src/components/widgets/search/search.jsx) |
| 全局快速启动（QuickLaunch） | [quicklaunch.jsx](src/components/quicklaunch.jsx) |
| 后端建议 API | [searchSuggestion.js](src/pages/api/search/searchSuggestion.js) |
| 首页入口（状态提升/事件绑定） | [index.jsx](src/pages/index.jsx) |
| 带缓存的 HTTP 请求 | [http.js](src/utils/proxy/http.js) |
| Widget 动态加载入口 | [widget.jsx](src/components/widgets/widget.jsx) |

---

## 二、搜索提供者（searchProviders）配置中心

所有搜索引擎元信息统一收敛于 [search.jsx#L22-L58](src/components/widgets/search/search.jsx#L22-L58) 的 `searchProviders` 对象，这是两套搜索机制的共同根基：

```javascript
export const searchProviders = {
  google:      { name: "Google",      url: "https://www.google.com/search?q=",           suggestionUrl: "https://www.google.com/complete/search?client=chrome&q=", icon: SiGoogle },
  duckduckgo:  { name: "DuckDuckGo",  url: "https://duckduckgo.com/?q=",                 suggestionUrl: "https://duckduckgo.com/ac/?type=list&q=",                 icon: SiDuckduckgo },
  bing:        { name: "Bing",        url: "https://www.bing.com/search?q=",             suggestionUrl: "https://api.bing.com/osjson.aspx?query=",                 icon: BiLogoBing },
  baidu:       { name: "Baidu",       url: "https://www.baidu.com/s?wd=",                suggestionUrl: "http://suggestion.baidu.com/su?&action=opensearch&ie=utf-8&wd=", icon: SiBaidu },
  brave:       { name: "Brave",       url: "https://search.brave.com/search?q=",         suggestionUrl: "https://search.brave.com/api/suggest?&rich=false&q=",     icon: SiBrave },
  custom:      { name: "Custom",      url: false,                                        suggestionUrl: undefined,                                                   icon: FiSearch },
};
```

**关键点**：
- `custom` 提供者的 `url` 为 `false`、`suggestionUrl` 为 `undefined`，需要在运行时从配置中动态注入
- `getAvailableProviderIds(options)` [search.jsx#L60-L68](src/components/widgets/search/search.jsx#L60-L68) 负责过滤用户配置的合法 provider

---

## 三、输入状态管理机制

### 3.1 全局按键唤醒的隐含时序：首个字符如何进入输入框

这是一个**非显式实现**的隐式行为。代码中**没有任何地方**手动提取 `e.key` 并将其拼接到 `searchString` 中，完全依赖浏览器事件的默认传播与执行时序：

**Step 1：全局监听捕获按键（仅唤醒，不获取字符）** [index.jsx#L253-L277](src/pages/index.jsx#L253-L277)
```javascript
document.addEventListener("keydown", function handleKeyDown(e) {
  if (e.target.tagName === "BODY" || e.target.id === "inner_wrapper") {
    if (/* 匹配可输入字符、粘贴快捷键等 */) {
      setSearching(true);  // 仅做一件事：标记打开
      // ❗ 注意：此处既没有调用 e.preventDefault()，也没有读取 e.key
    }
  }
});
```

**Step 2：React 状态更新触发重渲染**
- `searching` 状态从 `false` → `true`，`Home` 组件同步重渲染
- `QuickLaunch` 组件收到新的 prop `isOpen={true}`

**Step 3：QuickLaunch 打开 effect 中主动聚焦输入框** [quicklaunch.jsx#L223-L239](src/components/quicklaunch.jsx#L223-L239)
```javascript
useEffect(() => {
  if (isOpen) {
    searchField.current.focus();  // 🔑 关键：浏览器默认行为执行之前，输入框已获得焦点
    setHidden(false);
  }
}, [isOpen]);
```

**Step 4：浏览器默认行为被动完成字符注入**
- 因为没有调用 `preventDefault()`，浏览器继续执行 `keydown` 的原生默认行为
- 原生默认行为的逻辑是：**「向当前获得焦点的可输入元素插入按键字符」**
- 此时输入框正好获得了焦点，字符被自动插入
- 输入框内容变化触发 `onChange` → `handleSearchChange` → `setSearchString(...)`

**时序依赖图（隐含条件，代码未显式保证）**：
```
用户按下 'h' 键
    │
    ▼
keydown 事件同步分发
    │
    ├─→ ① 全局 handleKeyDown(e) 同步执行
    │     └─ setSearching(true)   [仅标记，不 preventDefault]
    │
    ├─→ ② React 批处理微任务（依赖事件循环模型保证在默认行为之前）
    │     ├─ state 更新：searching = true
    │     ├─ 重渲染：<QuickLaunch isOpen={true} />
    │     └─ useEffect 执行：searchField.current.focus()  ✓ 焦点已转移
    │
    └─→ ③ 浏览器默认行为（最后执行）
          └─ 向当前焦点元素（输入框）插入 'h'
              └─ onChange → handleSearchChange → setSearchString("h")
```

> **隐含风险说明**：这套机制不是代码显式传递字符，而是利用「在同一个 keydown 事件中，React 微任务批处理先于浏览器默认行为执行」的时序特性。如果该特性在未来浏览器/React 版本中发生变化，或者 React 渲染卡顿导致 focus() 延迟，**第一个按键字符将丢失**——这是一个对运行时环境的隐式依赖。

**全局键盘事件触发条件详解** [index.jsx#L253-L277](src/pages/index.jsx#L253-L277)：
只有当焦点位于 `BODY` 或 `#inner_wrapper`（即用户没有聚焦任何输入框/按钮）时，全局监听才生效：
- **普通字符**：`e.key.length === 1` 且匹配字母/空格/Unicode 字母范围（包括西欧重音、西里尔字母等），且**没有**按修饰键（Alt/Ctrl/Cmd/Shift）时 → 唤醒
- **特殊重音与感叹号**：由于某些键盘布局输入 `à-ü` / `!` 需要按住 Shift 等修饰键，因此额外作为例外匹配 → 唤醒
- **粘贴快捷键**：`e.key === "v"` 且 `Ctrl/Cmd` 按下 → 唤醒（支持用户直接粘贴，剪贴板内容由浏览器默认行为填入输入框）
- **Escape 键**：反向操作——`setSearchString("")` 清空输入内容 + `setSearching(false)` 关闭模态框

### 3.2 Search Widget（独立搜索栏）

核心状态位于 [search.jsx#L87-L89](src/components/widgets/search/search.jsx#L87-L89)：

| State 变量 | 类型 | 作用 |
|-----------|------|------|
| `query` | `string` | 当前输入框的原始文本，由 `ComboboxInput` 的 `onChange` 更新 [search.jsx#L180-L182](src/components/widgets/search/search.jsx#L180-L182) |
| `selectedProvider` | `object` | 当前选中的搜索引擎对象，支持 localStorage 持久化 |
| `searchSuggestions` | `array` | 后端返回的建议数组，格式为 `[queryString, [sug1, sug2, ...]]` |

**持久化**：`localStorageKey = "search-name"`，通过 `getStoredProvider()` [search.jsx#L72-L80](src/components/widgets/search/search.jsx#L72-L80) 在组件初始化时读取，通过 `onChangeProvider()` [search.jsx#L158-L161](src/components/widgets/search/search.jsx#L158-L161) 在切换时写入。

### 3.3 QuickLaunch（全局模态框）

状态分为两层：

**提升层（Home 组件）**—— [index.jsx#L249-L250](src/pages/index.jsx#L249-L250)：
- `searching: boolean` — 控制模态框显隐
- `searchString: string` — 输入内容（受控模式）

**组件内部层**—— [quicklaunch.jsx#L26-L29](src/components/quicklaunch.jsx#L26-L29)：
| State 变量 | 作用 |
|-----------|------|
| `results` | 最终候选列表（合并了本地服务/书签 + 搜索建议 + URL 直达） |
| `currentItemIndex` | 键盘/悬停选中项的索引 |
| `url` | 如果输入内容检测为合法 URL，则保存为 `URL` 对象 |
| `searchSuggestions` | 独立保存的搜索建议数组 |

### 3.4 普通查询小写化 vs URL 原值保留

这是 `handleSearchChange` 中的核心分支逻辑 [quicklaunch.jsx#L82-L95](src/components/quicklaunch.jsx#L82-L95)：

```javascript
function handleSearchChange(event) {
  const rawSearchString = event.target.value;  // 浏览器填入的原始值（未处理）
  try {
    if (!/.+[.:].+/g.test(rawSearchString)) throw new Error();  // URL 预检测正则
    let urlString = rawSearchString;
    if (urlString.toLowerCase().indexOf("http") !== 0) urlString = `https://${rawSearchString}`;
    setUrl(new URL(urlString));          // ✅ 保存规范化后的 URL 对象（补了 https 前缀）
    setSearchString(rawSearchString);    // ✅ searchString：保留用户输入的原始大小写
    return;
  } catch (e) {
    setUrl(null);                        // ❌ 非 URL，清空 url 状态
  }
  setSearchString(rawSearchString.toLowerCase());  // ✅ 普通查询：searchString 被强制小写化
}
```

**两条路径的对比**：

| 输入内容 | 路径 | searchString | `url` state | 说明 |
|---------|------|--------------|-------------|------|
| `GitHub.com` | URL 路径 | `"GitHub.com"`（原值） | `URL("https://GitHub.com")` | searchString 保留原值，用于显示 |
| `GitHub` | 普通查询 | `"github"`（小写） | `null` | searchString 转为小写 |
| `docs.example.com/Guide` | URL 路径 | `"docs.example.com/Guide"`（原值） | `URL("https://docs.example.com/Guide")` | 路径大小写完整保留 |
| `hello world` | 普通查询 | `"hello world"`（小写） | `null` | 含空格的查询也整体小写 |

---

### 3.5 URL 原值保留、`url` state、`hideVisitURL` 与 URL 直达候选的完整关系链

这四个概念形成一条**逐层递进的依赖链**，在候选列表构造 `useEffect` 中交汇 [quicklaunch.jsx#L140-L220](src/components/quicklaunch.jsx#L140-L220)：

```
handleSearchChange（输入变化时）
    │
    ├─→「URL 路径」分支执行时
    │     ├─ setSearchString(rawSearchString)     ← 用户可见的显示值（保留原始大小写）
    │     └─ setUrl(new URL(urlString))           ← 规范化后的 URL 对象（已补 https）
    │
    └─→「普通查询」分支执行时
          ├─ setSearchString(rawSearchString.toLowerCase())  ← 小写化用于搜索
          └─ setUrl(null)                                     ← 明确清空 url state
                                                        ↓
                                              候选列表构造 useEffect
                                                        │
                    ┌───────────────────────────────────┼───────────────────────────────────┐
                    ▼                                   ▼                                   ▼
        ① searchString（小写后）             ② url state（URL 对象/null）          ③ hideVisitURL（设置开关）
        用于：本地服务/书签过滤              决定："Visit URL" 候选是否可见         控制：即使 url 存在，用户是否想看到
        用于：Web 搜索 href 编码 URL         提供：候选真正跳转的 href（toString）    可完全屏蔽 URL 直达候选
        用于：建议接口 query 参数
                    │                                   │                                   │
                    │                                   └──────────────┬────────────────────┘
                    │                                                  ▼
                    │                                    if (!hideVisitURL && url) {
                    │                                      newResults.unshift({
                    │                                        href: url.toString(),   // 👈 用 url state，不用 searchString
                    │                                        name: "Visit URL",       //    确保跳转的是带协议的有效 URL
                    │                                        type: "url"
                    │                                      });
                    │                                    }
                    │
                    ▼
            Web 搜索候选：href = searchProvider.url + encodeURIComponent(searchString)
            建议请求 fetch：query = encodeURIComponent(searchString)
```

**关键交叉验证点**：

1. **Web 搜索和建议请求**——[quicklaunch.jsx#L161](src/components/quicklaunch.jsx#L161) 与 [L168-L170](src/components/quicklaunch.jsx#L168-L170)：
   ```javascript
   // Web 搜索候选的 href 使用小写后的 searchString
   href: searchProvider.url + encodeURIComponent(searchString)
   
   // 建议接口的 query 参数也使用小写后的 searchString
   `/api/search/searchSuggestion?query=${encodeURIComponent(searchString)}&providerName=...`
   ```
   ⚠️ **重要**：之前的理解有误——**普通查询小写化后，Web 搜索和建议请求实际也发送小写值**，并非用户输入的原始值。两者共享同一个 `searchString` 变量，这在配置了区分大小写的自定义搜索引擎时需要注意。

2. **URL 直达候选的 href 来源**——[quicklaunch.jsx#L201-L207](src/components/quicklaunch.jsx#L201-L207)：
   ```javascript
   if (!hideVisitURL && url) {              // 双条件判断：开关 && 有有效 URL
     newResults.unshift({
       href: url.toString(),                // 👈 用 url state.toString()
       name: `${t("quicklaunch.visit")} URL`,
       type: "url",
     });
   }
   ```
   为什么不直接用 `searchString`？因为 `searchString` 是用户输入的 `GitHub.com`（无协议），而 `url.toString()` 返回的是 `new URL("https://GitHub.com").toString()` = `"https://GitHub.com/"`（带协议、带规范化尾斜杠），才能直接作为 `window.open` 的有效 href。

3. **hideVisitURL 的作用**：在 [quicklaunch.jsx#L22](src/components/quicklaunch.jsx#L22) 从 settings 中解构：
   - `hideVisitURL = false`（默认）：正常显示 URL 直达候选
   - `hideVisitURL = true`：即使 `url` state 存在（输入是合法 URL），也**不渲染**「Visit URL」候选项，用户只能走 Web 搜索或其他路径

4. **为什么 URL 路径要保留原值？**
   - **searchString 保留原值**：用于输入框显示，用户看到的就是自己输入的内容，不会出现"输入 `GitHub.com`，输入框突然变成 `github.com`"的认知不一致
   - **url state 存规范化对象**：用于实际跳转，保证一定带有 `https://` 协议，才能被浏览器正确打开
   - 两者各司其职，互不干扰

---

## 四、候选建议来源逻辑

### 4.1 数据流总览

```
用户输入字符
    │
    ▼
┌─────────────────────────────────────┐
│ 前端防抖/触发条件检查                │
│   • query.trim().length > 0         │
│   • query !== searchSuggestions[0]  │  ← 避免重复请求相同内容
│   • showSearchSuggestions === true  │
└──────────────────┬──────────────────┘
                   │ fetch
                   ▼
┌──────────────────────────────────────────────────────┐
│  /api/search/searchSuggestion                        │
│  ?query=xxx&providerName=Google                      │
└──────────────────┬───────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         ▼                    ▼
   内置 provider          Custom provider
   (读 searchProviders)   (查 widgets → 查 quicklaunch settings)
         │                    │
         └─────────┬──────────┘
                   ▼
    cachedRequest(url, 5, "Mozilla/5.0")
                   │  memory-cache，5 分钟 TTL
                   ▼
         返回 [query, [sug1, sug2, ..., sug4]]
                   │  截断至前 4 条
                   ▼
         前端合并入候选列表 / 渲染下拉框
```

### 4.2 Search Widget 中的触发逻辑

位于 [search.jsx#L100-L130](src/components/widgets/search/search.jsx#L100-L130) 的 `useEffect`，依赖 `[selectedProvider, options, query, searchSuggestions]`：

- 使用 `AbortController` 处理并发竞态：组件卸载或依赖变化时自动 `abort()` 上一个请求
- 最大建议数：后端返回后在 [search.jsx#L116-L118](src/components/widgets/search/search.jsx#L116-L118) 通过 `splice(0, 4)` 截断为 4 条

### 4.3 QuickLaunch 中的多源候选合并

位于 [quicklaunch.jsx#L140-L220](src/components/quicklaunch.jsx#L140-L220) 的 `useEffect`，按以下优先级**构造最终 `results` 数组**：

| 优先级 | 来源 | 类型标识 | 条件 |
|-------|------|----------|------|
| 1（最高） | 直接 URL 访问 | `"url"` | `/.+[.:].+/` 正则命中 + `new URL()` 校验成功，通过 `unshift` 插入最前面 |
| 2 | 本地服务 + 书签匹配 | 无（默认 bookmark/service） | 对 `servicesAndBookmarks` 进行 `filter`：匹配 `name`（必选）或 `description`（需 `searchDescriptions` 开启） |
| 3 | 搜索引擎 Web 搜索 | `"search"` | 始终有搜索 provider 时追加 |
| 4 | 搜索建议 | `"searchSuggestion"` | 开启 `showSearchSuggestions` 且 provider 有 `suggestionUrl` |

**描述匹配排序**：当 `searchDescriptions=true` 时，为每条结果打 `priority` 分：
- 名称命中 → priority = 2
- 仅描述命中 → priority = 1
然后 `sort((a, b) => b.priority - a.priority)`，确保名称匹配排在前面。

### 4.4 后端 API 路由深入解析

[searchSuggestion.js#L7-L36](src/pages/api/search/searchSuggestion.js#L7-L36) 完整处理流程：

**步骤 1：根据 providerName 定位 provider** [L8-L14]
```javascript
const provider = Object.values(searchProviders).find(({ name }) => name === providerName);
if (!provider) return res.json([query, []]);  // 未找到，返回空数组
```

**步骤 2：Custom 提供者的 URL 动态注入** [L16-L30]
优先级链：**widgets.yaml 的 search widget 配置** → **settings.yaml 的 quicklaunch.custom 配置**
```javascript
if (provider.name === "Custom") {
  const searchWidget = (await widgetsFromConfig()).find(w => w.type === "search");
  if (searchWidget) {
    provider.url = searchWidget.options.url;
    provider.suggestionUrl = searchWidget.options.suggestionUrl;
  } else {
    const settings = getSettings();
    if (settings.quicklaunch?.provider === "custom") {
      provider.url = settings.quicklaunch.url;
      provider.suggestionUrl = settings.quicklaunch.suggestionUrl;
    }
  }
}
```
⚠️ **注意**：此处直接修改了从 `searchProviders` 拿到的对象引用，属于**可变副作用**。测试用例中通过 `beforeEach` 重置 `providers.custom.url` 来规避交叉污染。

**步骤 3：缓存请求** [L32-L36]
调用 `cachedRequest(url, 5, "Mozilla/5.0")`，5 分钟内存缓存，User-Agent 设置为浏览器标识以绕过部分搜索引擎的爬虫拦截。

### 4.5 cachedRequest 缓存层

[http.js#L85-L109](src/utils/proxy/http.js#L85-L109)：
- 使用 `memory-cache` 包，key 为完整 URL
- TTL = `duration * 1000 * 60`，即传入的 `5` 实际代表 **5 分钟**
- 自动处理 `Buffer → JSON.parse` 的数据转换，解析失败时 fallback 为原始字符串

---

## 五、快捷跳转与键盘交互

### 5.1 QuickLaunch 键盘映射表（功能最完整）

在 [quicklaunch.jsx#L97-L119](src/components/quicklaunch.jsx#L97-L119) 的 `handleSearchKeyDown`：

| 按键 | 条件 | 行为 |
|------|------|------|
| `Escape` | 始终 | `closeAndReset()` → 延迟 200ms 清空搜索串/索引/建议 |
| `Enter` | `results.length > 0` | 关闭 + `openCurrentItem(event.metaKey)` → `metaKey`=Cmd/Ctrl 时用 `_blank` |
| `ArrowDown` | `results[currentItemIndex + 1]` 存在 | `setCurrentItemIndex(+1)`，`preventDefault()` 防光标跳动 |
| `ArrowUp` | `currentItemIndex > 0` | `setCurrentItemIndex(-1)` |
| `ArrowRight` | 当前项为 `searchSuggestion` 类型 | **自动补全**到输入框：`setSearchString(results[currentItemIndex].name)` |

**打开链接的 target 优先级链** [quicklaunch.jsx#L64-L71](src/components/quicklaunch.jsx#L64-L71)：
```
metaKey(Cmd/Ctrl) 按下 → "_blank"
    → 否则使用 result.target（书签/服务自带）
        → 否则 searchProvider.target
            → 否则 settings.target
                → 最终 fallback "_blank"
```

### 5.2 Search Widget 的键盘交互

在 [search.jsx#L147-L152](src/components/widgets/search/search.jsx#L147-L152)：

| 按键 | 行为 |
|------|------|
| `Enter` | 优先使用 `currentSuggestion`（当前高亮的建议项），否则用输入框原始值 → `doSearch()` |

`currentSuggestion` 是**模块级闭包变量**（非 React state），在渲染 `ComboboxOption` 时通过 `active` 回调更新 [search.jsx#L261-L262](src/components/widgets/search/search.jsx#L261-L262)：
```javascript
{({ active }) => {
  if (active) currentSuggestion = suggestion;  // ← 追踪当前高亮项
  return ...
}}
```

### 5.3 鼠标交互

- **Search Widget**：`ComboboxOption` 使用 `onMouseDown`（而非 `onClick`）触发 `doSearch()` [search.jsx#L256-L258](src/components/widgets/search/search.jsx#L256-L258)，避免 `onBlur` 先执行导致下拉关闭丢失目标
- **QuickLaunch**：`onMouseEnter` → `handleItemHover` 更新 `currentItemIndex` [quicklaunch.jsx#L121-L123](src/components/quicklaunch.jsx#L121-L123)；`onClick` → `handleItemClick` 打开链接

---

## 六、界面反馈与渲染流程

### 6.1 Search Widget（基于 @headlessui/react）

#### 组件嵌套结构
```
ContainerForm
  └─ Raw
      └─ Combobox
          ├─ ComboboxInput          ← 文本输入框
          ├─ Listbox                ← 搜索引擎选择下拉（右侧图标按钮）
          │    ├─ ListboxButton
          │    └─ Transition + ListboxOptions
          │         └─ ListboxOption × N
          └─ ComboboxOptions        ← 搜索建议下拉（条件渲染）
               └─ ComboboxOption × (1 + 建议数)
                    └─ 前缀(query) + 后缀(剩余) 分色显示
```

#### 视觉高亮细节
搜索建议的匹配前缀与剩余部分拆分渲染 [search.jsx#L270-L273](src/components/widgets/search/search.jsx#L270-L273)：
```javascript
<span className="whitespace-pre">{suggestion.indexOf(query) === 0 ? query : ""}</span>
<span className="mr-4 whitespace-pre opacity-50">
  {suggestion.indexOf(query) === 0 ? suggestion.substring(query.length) : suggestion}
</span>
```
- 匹配前缀：完全不透明度
- 剩余后缀：`opacity-50` 半透明

#### Provider 切换动画
`Transition` 组件定义的淡入缩放入场/出场动画 [search.jsx#L211-L219](src/components/widgets/search/search.jsx#L211-L219)：
- enter: `duration-100`，从 `opacity-0 scale-95` 到 `opacity-100 scale-100`
- leave: `duration-75`，反向动画

### 6.2 QuickLaunch 模态框

#### 模态框显隐动画
[quicklaunch.jsx#L262-L267](src/components/quicklaunch.jsx#L262-L267) 通过三层状态控制：
- `isOpen` → 控制 opacity 过渡（300ms）
- `hidden` → 延迟 300ms 后加 `hidden` class，**确保过渡动画完成后再从 DOM 隐藏**
- 点击遮罩层（`.fixed.inset-0.bg-gray-500`）时通过判断 `event.target.tagName === "DIV"` 触发关闭 [quicklaunch.jsx#L224-L226](src/components/quicklaunch.jsx#L224-L226)

#### 当前项视觉反馈
`currentItemIndex` 匹配时追加 `bg-theme-300/50 dark:bg-theme-700/50` 高亮背景 [quicklaunch.jsx#L299-L302](src/components/quicklaunch.jsx#L299-L302)。

#### 描述文本高亮（仅描述匹配时）
`highlightText` 函数 [quicklaunch.jsx#L241-L257](src/components/quicklaunch.jsx#L241-L257) 通过 `RegExp` 动态分割文本，为匹配部分套上 `bg-theme-300/10` 背景：
```javascript
const parts = text.split(new RegExp(`(${searchString})`, "gi"));
// 遍历 parts，匹配段使用高亮 span
```

#### 移动端适配
`MOBILE_BUTTON_POSITIONS` 映射 [quicklaunch.jsx#L11-L16](src/components/quicklaunch.jsx#L11-L16) 支持 4 个角落配置浮动搜索按钮，仅在 `sm:hidden`（移动端）显示。

---

## 七、两套搜索机制的差异对比

| 维度 | Search Widget | QuickLaunch |
|------|--------------|-------------|
| 触发方式 | 直接点击/聚焦输入框 | 全局任意位置键入字符 / 点击移动按钮 |
| 候选范围 | 仅搜索建议 | 本地服务/书签 + URL直达 + Web搜索 + 搜索建议 |
| 状态管理 | 组件内部 useState | 部分提升到 Home 组件 |
| 键盘导航 | 依赖 HeadlessUI 内部机制 + Enter | 完整方向键 + Enter + Escape + ArrowRight补全 |
| Provider 切换 | 下拉选择（Listbox）+ localStorage | 无 UI，仅读取配置 |
| UI 组件库 | @headlessui/react Combobox + Listbox | 原生 input + ul/li + button |
| 搜索建议显示位置 | 输入框下方弹出层 | 候选列表中的独立条目（带类型标识） |
| 输入字符处理 | 无额外处理（直接使用） | URL 保留原值，普通查询小写化 |

---

## 八、潜在注意事项

1. **Custom Provider 可变修改**：`searchSuggestion.js` 中直接修改 `searchProviders.custom` 的字段，属于模块级副作用，并发请求可能互相影响
2. **currentSuggestion 闭包变量**：Search Widget 中用非 state 变量追踪选中项，在 React 严格模式或并发渲染下可能出现不一致
3. **重复请求保护**：前端通过 `query !== searchSuggestions[0]` 判断避免重复请求，这依赖后端返回的第一个元素（原始 query）完全一致
4. **关闭延迟**：QuickLaunch 关闭时使用 `setTimeout(200ms)` + `setTimeout(300ms)` 两段延迟，需在测试中通过 `act + setTimeout` 等待
5. **全局首个字符丢失风险**：详见 3.1 节。唤醒的第一个按键不经过代码显式传递，完全依赖浏览器/React 事件微任务时序，渲染卡顿可能导致字符丢失
6. **URL 检测正则的局限性**：`/.+[.:].+/` 会将 `ip:port`、`a.b` 等误认为 URL，在 catch 块才被纠正，有微小的性能开销
7. **普通查询小写化会透传到搜索引擎**：QuickLaunch 中用户输入 `GitHub`，searchString 被转换为 `github`，后续 Web 搜索候选的 href 和建议接口的 query 参数**都会使用小写值**，自定义搜索引擎若区分大小写需留意
