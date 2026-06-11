# Homepage 主题配置与自定义 CSS 运作机制

本文档逐行追踪源码，说明主题配置来源、初始化生效时序、运行时覆盖逻辑、以及 CSS 加载与层叠覆盖顺序。

---

## 一、整体架构概览

主题系统由两个互相独立的维度组合而成：

| 维度 | 控制什么 | 可选值 | CSS 实现机制 |
|------|---------|--------|-------------|
| **Theme（明暗模式）** | 背景色深浅、滚动条颜色等语义化颜色 | `dark` / `light` | 切换 `<html>` 上的 `.dark` / `.light` 类，改变派生 CSS 变量 |
| **Color（色板）** | 色相系列（红/蓝/灰/琥珀等） | 21 种：slate, gray, zinc, neutral, stone, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose, red, white | 切换 `<html>` 上的 `.theme-{color}` 类，改变基础 CSS 变量 |

两个类组合生效，例如 `<html class="dark theme-slate">` 表示暗色模式 + slate 色板。

---

## 二、主题配置的三个来源

### 2.1 来源一：`settings.yaml`

用户在 `config/settings.yaml` 中固定主题：

```yaml
theme: dark    # 或 light
color: slate   # 21 种色板之一
```

读取路径：

1. `getSettings()`（`src/utils/config/config.js` 第 82-103 行）从磁盘读取 YAML 并解析为对象
2. `getStaticProps()`（`src/pages/index.jsx` 第 55-95 行）在构建/ISR 阶段调用 `getSettings()`，结果作为 `initialSettings` prop 注入页面组件
3. API 端点 `/api/theme`（`src/pages/api/theme.js` 第 1-13 行）也独立读取 settings 返回 `{ color, theme }`

### 2.2 来源二：localStorage

浏览器本地存储保存用户上次手动选择的主题：

| localStorage key | 保存内容 |
|-----------------|---------|
| `theme-mode` | `dark` / `light` |
| `theme-color` | 色板名称，如 `slate` |

### 2.3 来源三：系统偏好

仅对明暗模式有效，读取浏览器的 `prefers-color-scheme: dark` 媒体查询。

### 2.4 默认兜底值

| 维度 | 默认值 |
|------|-------|
| theme | `dark` |
| color | `slate` |

---

## 三、初始化生效时序与优先级

主题不是一次性确定的，而是在 **SSR → Hydration → 多轮 useEffect** 中被多次覆盖。

### 3.1 Provider 设计的关键细节

`src/pages/_app.jsx` 第 87-95 行，Provider 的实例化：

```jsx
<ColorProvider>               {/* 没有传 initialTheme prop */}
  <ThemeProvider>             {/* 没有传 initialTheme prop */}
    <SettingsProvider>        {/* 没有传 initialSettings prop */}
      <TabProvider>
        <Component {...pageProps} />
      </TabProvider>
    </SettingsProvider>
  </ThemeProvider>
</ColorProvider>
```

**三个 Provider 都没有传递初始值 prop。** 这意味着：

- `ColorProvider` 内参数 `initialTheme` 始终为 `undefined`
- `ThemeProvider` 内参数 `initialTheme` 始终为 `undefined`
- `SettingsProvider` 内参数 `initialSettings` 始终为 `undefined`

三个 Provider 内部都有一段监听初始值 prop 的 useEffect（例如 ThemeProvider 第 34-37 行），形式如下：

```jsx
useEffect(() => {
  if (initialTheme !== undefined) setTheme(initialTheme ?? getInitialTheme());
}, [initialTheme]);
```

因为初始值 prop 始终为 `undefined`，这段代码**永远不会执行，是死代码**。settings.yaml 的配置不是通过 Provider 初始值注入的。

### 3.2 组件树嵌套关系

Next.js 的页面默认导出是 `Wrapper`（`src/pages/index.jsx` 第 517 行），内部渲染顺序：

```
Wrapper (第 517 行)
 └── Index (第 97 行)
      └── Home (第 214 行)   ← 最内层子组件
```

React 的 useEffect 按照**后序遍历**（深度优先，子组件先执行）执行。

### 3.3 阶段一：SSR / SSG 渲染阶段

服务端渲染时：

- `window` 对象不存在
- Provider 的 `useState(() => initialTheme ?? getInitialTheme())` 中：
  - `initialTheme` 是 `undefined`
  - `getInitialTheme()` / `getInitialColor()` 内部 `typeof window !== "undefined"` 判断为 false
  - 直接返回默认值 `dark` + `slate`
- **useEffect 在服务端不执行**，所以 SSR 输出的 HTML 中 `<html>` **没有**任何主题类

输出的是不带主题类的"裸"HTML。

### 3.4 阶段二：客户端 Hydration 同步阶段

浏览器接收到 HTML 后 React hydration。此时：

**ThemeProvider state 初始化**（`src/utils/contexts/theme.jsx` 第 3-22 行）：

```
getInitialTheme() 执行顺序：
  ① localStorage["theme-mode"] → 有值则返回
  ② matchMedia("(prefers-color-scheme: dark)") → 匹配则返回 "dark"
  ③ 返回默认 "dark"
```

**ColorProvider state 初始化**（`src/utils/contexts/color.jsx` 第 5-20 行）：

```
getInitialColor() 执行顺序：
  ① localStorage["theme-color"] → 有值则返回
  ② 返回默认 "slate"
```

此时 state 的值由 **localStorage → 系统偏好 → 默认值** 决定，**settings.yaml 的配置尚未参与**。

### 3.5 阶段三：第一轮 useEffect（挂载后执行）

按后序遍历顺序执行（子 → 父）：

#### 3.5.1 Home 组件：注入 settings.yaml 配置

`src/pages/index.jsx` Home 组件第 222-224 行：

```jsx
useEffect(() => {
  setSettings(initialSettings);   // initialSettings 来自 getStaticProps
}, [initialSettings, setSettings]);
```

把来自 `getStaticProps` 的 settings.yaml 配置写入 SettingsContext。此时 SettingsContext 从初始 `{}` 变为实际配置对象。

这一步会触发 SettingsContext 订阅者的重渲染（包含 Home 自身），从而进入第二轮 useEffect。

#### 3.5.2 Index 组件

执行 hashData、windowFocused 等和主题无关的 effect。

#### 3.5.3 Wrapper 组件：同步 `<html>` 类名（第一次）

`src/pages/index.jsx` Wrapper 组件第 540-563 行：

```jsx
useEffect(() => {
  // 明暗模式
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

  // 色板（注意兜底链）
  const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
  // ... 切换类名
}, [backgroundImage, opacity, theme, color, initialSettings.color]);
```

此时：
- `theme` / `color` 来自 Context = localStorage/默认值
- `initialSettings.color` 来自 settings.yaml，但 `color` 已有值，兜底链取 `color`

这一步**第一次**给 `<html>` 加上主题类，页面开始有颜色。

#### 3.5.4 TabProvider / SettingsProvider

SettingsProvider 的初始值监听 effect 判断 `initialSettings === undefined`，不执行。

#### 3.5.5 ThemeProvider：同步 `<html>` 类名（第二次）

`src/utils/contexts/theme.jsx` 第 39-41 行：

```jsx
useEffect(() => {
  rawSetTheme(theme);   // 写入 <html> 类 + localStorage
}, [theme]);
```

`rawSetTheme` 会先 remove 旧类再 add 新类，结果与 Wrapper 写入的相同（都是 localStorage/默认值），同时写入 localStorage。

#### 3.5.6 ColorProvider：同步 `<html>` 类名（第三次）

`src/utils/contexts/color.jsx` 第 38-40 行：

```jsx
useEffect(() => {
  rawSetColor(color);   // 写入 <html> 类 + localStorage
}, [color]);
```

同样写入相同的结果并持久化到 localStorage。

**第一轮结束时**：`<html>` 类名 = localStorage/默认值。settings.yaml 的配置还没覆盖主题。

### 3.6 阶段四：第二轮 useEffect（SettingsContext 变化触发）

Home 组件 3.5.1 中 `setSettings(initialSettings)` 导致 Home 重渲染，依赖 `settings` 的 effect 再次执行。

#### Home 组件：用 settings.yaml 覆盖 Context 值

`src/pages/index.jsx` Home 组件第 234-247 行：

```jsx
useEffect(() => {
  // ... 语言设置省略 ...

  // settings.yaml 覆盖当前主题
  if (settings.theme && theme !== settings.theme) {
    setTheme(settings.theme);
  }
  if (settings.color && color !== settings.color) {
    setColor(settings.color);
  }
}, [i18n, settings, color, setColor, theme, setTheme]);
```

**这是 settings.yaml 配置真正生效的位置。** 判断条件：

- `settings.theme` 为 truthy（即在 settings.yaml 中设置了）
- 当前 Context 的 theme 与配置值不同

满足时调用 `setTheme()` / `setColor()` 修改 Context state。这会触发 ThemeContext/ColorContext 订阅者重渲染，进入第三轮 useEffect。

如果 settings.yaml 未设置（`settings.theme === undefined`），或设置值与 Context 当前值相同，则跳过，流程在此结束。

### 3.7 阶段五：第三轮 useEffect（Context 值变化触发）

如果上一步 setTheme/setColor 被调用：

1. **Wrapper 的 effect**（依赖 `theme`/`color`）：重新同步 `<html>` 类名为最终值
2. **ThemeProvider 的 effect**：`rawSetTheme(最终值)` → 写入 DOM + localStorage
3. **ColorProvider 的 effect**：`rawSetColor(最终值)` → 写入 DOM + localStorage

第三轮结束，主题稳定。

### 3.8 完整时序图

```
  SSR 服务端
    └─ Provider state 初始化为默认 dark+slate，无主题类

  Hydration 客户端同步
    └─ ColorProvider state = localStorage["theme-color"] ?? "slate"
       ThemeProvider state = localStorage["theme-mode"] ?? (prefers-dark ? "dark" : "dark")
       SettingsProvider state = {}

  第一轮 useEffect（挂载）
    ├─ Home effect #1:   setSettings(initialSettings) ← 触发重渲染
    ├─ ...（Index 无关 effect）
    ├─ Wrapper effect:   <html> += .localStorage_theme .theme-localStorage_color
    ├─ ThemeProvider:    <html> 类同步，写入 localStorage
    └─ ColorProvider:    <html> 类同步，写入 localStorage

  第二轮 useEffect（SettingsContext 变化）
    └─ Home effect #2:   if settings.theme != Context.theme → setTheme()
                         if settings.color != Context.color → setColor()
                         ← 触发 Theme/Color Context 变化（若有变化）

  第三轮 useEffect（Theme/Color Context 变化，如有）
    ├─ Wrapper effect:   <html> 类最终同步
    ├─ ThemeProvider:    <html> 类最终同步，写入 localStorage
    └─ ColorProvider:    <html> 类最终同步，写入 localStorage
```

### 3.9 最终优先级结论

```
┌───────────────────────────────────────────────────────┐
│ 优先级 1（最高）：settings.yaml                        │
│   条件：settings.yaml 显式设置了 theme/color           │
│   时机：第二轮 useEffect 的 Home effect #2             │
│   效果：覆盖 localStorage 和系统偏好                    │
│   副作用：写入 localStorage，下次打开仍是这个值         │
├───────────────────────────────────────────────────────┤
│ 优先级 2：localStorage                                │
│   条件：settings.yaml 未设置，但 localStorage 有值      │
│   时机：Hydration 同步阶段                              │
├───────────────────────────────────────────────────────┤
│ 优先级 3：系统偏好（仅 theme）                         │
│   条件：localStorage 无值，系统设置为暗色模式            │
│   时机：Hydration 同步阶段                              │
├───────────────────────────────────────────────────────┤
│ 优先级 4（最低）：默认值                               │
│   theme → dark，color → slate                         │
└───────────────────────────────────────────────────────┘
```

### 3.10 FOUC（主题闪烁）问题

由于初始化分多轮执行，在以下场景会观察到主题闪烁：

- settings.yaml 固定主题为 `light`，但 localStorage 存的是 `dark`
- 用户先看到 1~2 帧深色主题（第三轮前的 localStorage 值），然后被 settings.yaml 覆盖为浅色

根本原因：localStorage 在第一轮 effect 中先写入 DOM，settings.yaml 在第二轮 effect 中才覆盖。

---

## 四、运行时手动切换机制

### 4.1 切换控件的显示条件

页面底部切换按钮（`src/pages/index.jsx` 第 502-506 行）：

```jsx
{!settings?.color && <ColorToggle />}   {/* settings.yaml 未固定 color 时才显示 */}
{!settings.theme && <ThemeToggle />}    {/* settings.yaml 未固定 theme 时才显示 */}
```

settings.yaml 固定了 theme 或 color 后，对应切换按钮自动隐藏。

### 4.2 切换流程

用户点击按钮 → 调用 Context 的 `setTheme`/`setColor` → 触发 Provider effect：

| 操作 | 结果 |
|-----|------|
| 切换明暗模式 | 更新 Context state → `<html>` 切换 `.dark`/`.light` 类 → 写入 `localStorage["theme-mode"]` |
| 切换色板 | 更新 Context state → `<html>` 切换 `.theme-{color}` 类 → 写入 `localStorage["theme-color"]` |

切换值持久化到 localStorage，下次打开时优先使用。

---

## 五、主题 CSS 变量体系

### 5.1 基础层：色板变量（`src/styles/theme.css`）

每个色板定义 9 级颜色梯度 + 2 个 Logo 渐变变量：

```css
.theme-slate {
  --color-50: 248 250 252;      /* 最浅 */
  --color-100: 241 245 249;
  --color-200: 226 232 240;
  --color-300: 203 213 225;
  --color-400: 148 163 184;
  --color-500: 100 116 139;
  --color-600: 71 85 105;
  --color-700: 51 65 85;
  --color-800: 30 41 59;
  --color-900: 15 23 42;        /* 最深 */
  --color-logo-start: 148 163 184;
  --color-logo-stop: 51 65 85;
}
```

颜色值使用 `R G B` 空格分隔格式（无逗号），配合 Tailwind 的 `<alpha-value>` 通道语法使用。

### 5.2 语义层：明暗模式变量（`src/styles/globals.css` 第 64-74 行）

根据明暗模式，从色板变量中挑选合适的级别：

```css
.light {
  --bg-color: var(--color-50);           /* 浅色背景 = 色板最浅色 */
  --scrollbar-thumb: rgb(var(--color-300));
  --scrollbar-track: rgb(var(--color-200));
}
.dark {
  --bg-color: var(--color-800);          /* 深色背景 = 色板第 8 级 */
  --scrollbar-thumb: rgb(var(--color-600));
  --scrollbar-track: rgb(var(--color-700));
}
```

**色板变量是基础，明暗模式变量是对色板变量的语义化挑选**。改变色板会同时影响明/暗两种模式的颜色。

### 5.3 Tailwind 工具类桥接（`tailwind.config.js` 第 19-32 行）

将 CSS 变量注册为 Tailwind 颜色：

```js
colors: {
  theme: {
    50:  "rgb(var(--color-50) / <alpha-value>)",
    100: "rgb(var(--color-100) / <alpha-value>)",
    // ... 直到 900
  },
}
```

组件中可直接使用 `bg-theme-100`、`text-theme-800`、`bg-theme-500/20` 等 Tailwind 类，实际颜色由当前激活的色板动态决定。

### 5.4 特殊处理：`theme-white`

白色色板的 `--color-100` 到 `--color-700` 都是灰色值，直接用 Tailwind 半透明类会导致白底上几乎不可见。因此 `src/styles/theme.css` 第 17-29 行对白色色板有额外的固定值覆盖：

```css
.theme-white .bg-theme-100\/20:not([class^="backdrop-blur"]) {
  background-color: rgb(245, 245, 245);  /* 固定浅灰色，不随变量变 */
}
```

---

## 六、CSS 加载顺序与层叠覆盖

### 6.1 构建时 CSS（`src/pages/_app.jsx` 顶部 import）

```jsx
import "styles/globals.css";   // 顺序 1
import "styles/manrope.css";   // 顺序 2
import "styles/theme.css";     // 顺序 3
```

Next.js 在构建时将这三个文件合并打包（可能分 chunk，但内部 CSS 规则顺序保持不变），**import 顺序决定 CSS 层叠顺序**，后引入者覆盖前者。

### 6.2 运行时 CSS（`src/pages/_document.jsx` 第 9-10 行）

```jsx
<link rel="preload" href="/api/config/custom.css" as="style" />
<link rel="stylesheet" href="/api/config/custom.css" />
```

`custom.css` 通过独立 `<link>` 标签注入 HTML `<head>`。API 端点 `src/pages/api/config/[path].js` 第 13-34 行从 `config/custom.css` 读取文件内容返回；文件不存在则返回空内容。

### 6.3 运行时 JS（`src/pages/index.jsx` 第 439 行）

```jsx
<Script src="/api/config/custom.js" />
```

### 6.4 浏览器解析 `<head>` 的完整顺序

```
HTML <head> 按出现顺序解析：

  │
  ▼
┌───────────────────────────────────────────────────────┐
│ 1. Next.js 构建产物 <link rel="stylesheet">            │
│    内部规则顺序（由 _app.jsx import 顺序决定）：        │
│    ┌─────────────────────────────────────────────┐    │
│    │ a. globals.css                              │    │
│    │    - @import tailwindcss（Tailwind v4 基础） │    │
│    │    - Tailwind 工具类（含 theme-{N} 映射）    │    │
│    │    - @layer base 边框颜色兼容               │    │
│    │    - html/body/#__next 全局样式              │    │
│    │    - .light / .dark 语义化 CSS 变量         │    │
│    │    - 滚动条、图表间距等全局规则              │    │
│    ├─────────────────────────────────────────────┤    │
│    │ b. manrope.css                              │    │
│    │    - Manrope 字体 @font-face                │    │
│    ├─────────────────────────────────────────────┤    │
│    │ c. theme.css                                │    │
│    │    - 21 个 .theme-{color} 基础 CSS 变量     │    │
│    │    - .theme-white 的特殊覆盖规则            │    │
│    └─────────────────────────────────────────────┘    │
├───────────────────────────────────────────────────────┤
│ 2. <link rel="stylesheet" href="/api/config/custom.css"> │
│    用户自定义 CSS，独立 <link>，在所有构建样式之后       │
└───────────────────────────────────────────────────────┘

<body> 渲染完成后：
┌───────────────────────────────────────────────────────┐
│ 3. <Script src="/api/config/custom.js">               │
│    用户自定义 JS，可动态操作 DOM/CSS                    │
└───────────────────────────────────────────────────────┘
```

### 6.5 覆盖关系分析

CSS 层叠优先级由 **来源 → 特异性 → 出现顺序** 共同决定。本项目中：

| 覆盖能力 | 样式来源 | 说明 |
|---------|---------|------|
| 🥇 最强 | `custom.css` | 最后加载的独立 `<link>`，同特异性下必然胜出 |
| 🥈 | `theme.css` | 构建时 CSS 的最后一部分，定义色板变量 |
| 🥉 | `globals.css` | 构建时 CSS 前部，含 Tailwind 基础和语义变量 |
| 最弱 | Tailwind 基础层 | 在 globals.css 中最先 import |

**实际覆盖示例**：

| 需求 | 在 custom.css 中写法 | 为什么有效 |
|-----|-------------------|-----------|
| 修改 slate 色板的主题色 | `.theme-slate { --color-500: 255 0 0; }` | 同特异性，custom.css 后加载 |
| 强制所有卡片不透明 | `.service-card { opacity: 1 !important; }` | `!important` 提升优先级 |
| 修改暗色模式背景 | `.dark { --bg-color: 10 10 30; }` | 同特异性，custom.css 后加载 |
| 精准修改某个服务 | `#myserviceid .card-body { padding: 8px; }` | id 选择器特异性高于类选择器 |

---

## 七、`themes.js` 的独立用途

`src/utils/styles/themes.js` 是与 CSS 变量体系**独立**的 JS 色板映射：

```js
{
  slate: {
    light: "#f8fafc",      // 浅色模式背景色（十六进制）
    dark: "#1e293b",       // 深色模式背景色（十六进制）
    iconStart: "#94a3b8",  // Logo 渐变起始色
    iconEnd: "#334155",    // Logo 渐变结束色
  },
  // ... 其他 20 个色板
}
```

它**不参与 CSS 变量定义**，而是在 JS 代码中需要十六进制颜色值时使用，例如：

- `<meta name="theme-color">` 和 `<meta name="msapplication-TileColor">`（`src/pages/index.jsx` 第 434-435 行）
- PWA manifest 中的 `theme_color` 和 `background_color`

---

## 八、骨架文件与首次运行

`src/skeleton/` 目录存放首次运行时的配置模板：

| 骨架文件 | 默认内容 |
|---------|---------|
| `settings.yaml` | 仅含 `providers` 占位，不设置 theme/color |
| `custom.css` | 空文件 |
| `custom.js` | 空文件 |
| `services.yaml` / `bookmarks.yaml` / `widgets.yaml` | 示例配置 |

当 `config/` 目录下缺少对应文件时，`src/utils/config/config.js` 第 15-50 行的 `checkAndCopyConfig()` 会自动将骨架文件复制过去。

---

## 九、总结速查表

### 主题来源优先级

```
settings.yaml > localStorage > 系统偏好 > 默认值
```

### 初始化时序口诀

```
SSR（无类）→ Hydration（读 localStorage）
  → 第一轮 useEffect：写入 DOM（localStorage 值）
  → 第二轮 useEffect：settings.yaml 覆盖 Context
  → 第三轮 useEffect：最终写入 DOM 和 localStorage
```

### CSS 覆盖优先级

```
custom.css > theme.css > globals.css > Tailwind 基础层
```

### 主题生效位置

| 位置 | 作用 |
|-----|------|
| `<html class="dark">` / `.light` | 切换明暗模式 CSS 变量 |
| `<html class="theme-slate">` | 切换色板 CSS 变量 |
| `localStorage["theme-mode"]` | 持久化用户选择的明暗模式 |
| `localStorage["theme-color"]` | 持久化用户选择的色板 |
| `config/settings.yaml` 的 `theme`/`color` | 固定主题，覆盖 localStorage 和系统偏好 |
| `config/custom.css` | 自定义样式，可覆盖所有内置 CSS |
