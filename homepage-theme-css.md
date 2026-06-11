# Homepage 主题配置与自定义 CSS 运作机制

本文档通过逐行追踪源码，详细说明 Homepage 项目的主题配置来源、初始化生效时序、运行时覆盖逻辑、以及 CSS 加载和层叠覆盖顺序。

---

## 一、整体架构概览

Homepage 的主题系统由两个互相独立的维度组合而成：

| 维度 | 控制什么 | 可选值 | CSS 实现机制 |
|------|---------|--------|-------------|
| **Theme（明暗模式）** | 背景色深浅、滚动条颜色等语义化颜色 | `dark` / `light` | 切换 `<html>` 上的 `.dark` / `.light` 类，改变派生 CSS 变量 |
| **Color（色板）** | 色相系列（红/蓝/灰/琥珀等） | 21 种：slate, gray, zinc, neutral, stone, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose, red, white | 切换 `<html>` 上的 `.theme-{color}` 类，改变基础 CSS 变量 |

两个类组合生效，例如 `<html class="dark theme-slate">` 表示暗色模式 + slate 色板。

---

## 二、主题配置的三个来源

主题值有三个来源，它们在不同阶段被读取，最终合并成页面生效的主题。

### 2.1 来源一：配置文件 `settings.yaml`

用户在 `config/settings.yaml` 中固定主题：

```yaml
theme: dark    # 或 light
color: slate   # 21 种色板之一
```

**读取路径**：

1. `getSettings()`（[config.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/config/config.js#L82-L103)）从磁盘读取 `config/settings.yaml`，解析为对象
2. `getStaticProps()`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L55-L95)）在 Next.js 构建/ISR 阶段调用 `getSettings()`，将其作为 `initialSettings` prop 传入页面组件
3. API 端点 `/api/theme`（[theme.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/api/theme.js#L1-L13)）也会独立读取 settings，返回 `{ color, theme }`

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

这是理解主题系统最关键的部分。**主题不是一次性确定的，而是在 SSR → Hydration → 多轮 useEffect 中被多次覆盖**。

### 3.1 Provider 初始化设计的关键细节

`_app.jsx` 中 Provider 的实例化（[\_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/_app.jsx#L87-L95)）：

```jsx
<ColorProvider>              {/* 没有传 initialTheme prop */}
  <ThemeProvider>            {/* 没有传 initialTheme prop */}
    <SettingsProvider>
      <TabProvider>
        <Component {...pageProps} />
      </TabProvider>
    </SettingsProvider>
  </ThemeProvider>
</ColorProvider>
```

**这里没有传递 `initialTheme`！** 也就是说 Provider 内部的 `initialTheme` 参数始终是 `undefined`。

### 3.2 阶段一：SSR / SSG 渲染阶段

服务端渲染时：

- `window` 对象不存在
- Provider 的 `useState(() => initialTheme ?? getInitialTheme())` 中：
  - `initialTheme` 是 `undefined`
  - `getInitialTheme()` 和 `getInitialColor()` 内部 `typeof window !== "undefined"` 判断为 false
  - 直接返回默认值 `dark` + `slate`
- **但 useEffect 在服务端不执行**，所以 SSR 输出的 HTML 中 `<html>` **不会有** `.dark` / `.theme-slate` 类

也就是说，SSR 输出的 HTML 是不带主题类的"裸"HTML。

### 3.3 阶段二：客户端 Hydration 同步阶段

浏览器接收到 HTML 后，React 进行 hydration。此时：

**ThemeProvider 初始化**（[theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/theme.jsx#L3-L22)）：

```
getInitialTheme() 执行顺序：
  ① 读 localStorage["theme-mode"] → 有值则返回
  ② 读 matchMedia("(prefers-color-scheme: dark)") → 匹配则返回 "dark"
  ③ 返回默认 "dark"
```

**ColorProvider 初始化**（[color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/color.jsx#L5-L20)）：

```
getInitialColor() 执行顺序：
  ① 读 localStorage["theme-color"] → 有值则返回
  ② 返回默认 "slate"
```

此时 state 的值由 **localStorage → 系统偏好 → 默认值** 决定，**settings.yaml 的配置还没有参与**。

Provider 内部还有一个监听 `initialTheme` 的 useEffect：

```jsx
useEffect(() => {
  if (initialTheme !== undefined) setTheme(initialTheme ?? getInitialTheme());
}, [initialTheme]);
```

但因为 `initialTheme` 始终是 `undefined`（见 3.1 节），这个 useEffect **永远不会执行**，是一段死代码。

### 3.4 阶段三：useEffect 第一轮执行

Hydration 完成后，所有组件的 useEffect 按树的深度优先顺序执行。

#### 3.4.1 Provider 的 DOM 同步 effect

ThemeProvider（[theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/theme.jsx#L39-L41)）：

```jsx
useEffect(() => {
  rawSetTheme(theme);  // 把 localStorage/默认值写入 <html> 类和 localStorage
}, [theme]);
```

ColorProvider（[color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/color.jsx#L38-L40)）：

```jsx
useEffect(() => {
  rawSetColor(color);  // 把 localStorage/默认值写入 <html> 类和 localStorage
}, [color]);
```

此时 `<html>` 第一次被加上主题类，页面开始显示颜色。

#### 3.4.2 Home 组件的 settings 注入

Home 组件（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L222-L224)）：

```jsx
useEffect(() => {
  setSettings(initialSettings);  // 把 settings.yaml 的配置写入 SettingsContext
}, [initialSettings, setSettings]);
```

这一步把来自 `getStaticProps` 的 settings.yaml 配置注入到全局 SettingsContext 中。

#### 3.4.3 Wrapper 组件的 DOM 同步

Wrapper 组件（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L540-L563)）：

```jsx
useEffect(() => {
  // 明暗模式
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

  // 色板（注意这里的兜底链）
  const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
  // ... 执行类切换
}, [backgroundImage, opacity, theme, color, initialSettings.color]);
```

注意这里色板用了 `color || initialSettings.color || "slate"` 的兜底。此时 `color` 来自 Context（localStorage/默认值），`initialSettings.color` 来自 settings.yaml。如果 Context 中的 color 是空值，会直接用 settings.yaml 的值。

### 3.5 阶段四：useEffect 第二轮执行（settings 变化触发）

Home 组件 3.4.2 中调用了 `setSettings(initialSettings)`，导致 SettingsContext 变化，触发 Home 组件的另一个 useEffect（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L234-L247)）：

```jsx
useEffect(() => {
  // ... 语言设置省略 ...

  // settings.yaml 配置覆盖当前主题
  if (settings.theme && theme !== settings.theme) {
    setTheme(settings.theme);
  }

  if (settings.color && color !== settings.color) {
    setColor(settings.color);
  }
}, [i18n, settings, color, setColor, theme, setTheme]);
```

**这一步是 settings.yaml 配置生效的真正位置。** 如果 settings.yaml 中设置了 `theme` 或 `color`，且与当前 Context 中的值（来自 localStorage/默认值）不同，就会调用 `setTheme` / `setColor` 覆盖。

### 3.6 阶段五：useEffect 第三轮执行（theme/color state 变化触发）

如果上一步 `setTheme` / `setColor` 被调用，会再次触发：

1. Provider 的 `rawSetTheme` / `rawSetColor` effect → 把最终值写入 `<html>` 类和 localStorage
2. Wrapper 的 DOM 同步 effect → 再次同步 `<html>` 类名

### 3.7 最终优先级结论

```
┌───────────────────────────────────────────────────────┐
│ 优先级 1（最高）：settings.yaml                        │
│   条件：在 settings.yaml 中显式设置了 theme/color       │
│   时机：useEffect 第二轮（Home 组件）                   │
│   效果：覆盖 localStorage 和系统偏好                    │
│   副作用：写入 localStorage，下次打开时仍是这个值       │
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

### 3.8 FOUC（主题闪烁）问题

由于初始化分多轮执行，在以下场景可能观察到主题闪烁：

- settings.yaml 固定主题为 `light`，但 localStorage 中存的是 `dark`
- 用户会先看到 1~2 帧的深色主题，然后被 settings.yaml 覆盖为浅色

原因是 localStorage 在 hydration 阶段先生效，settings.yaml 在后续 useEffect 中才覆盖。

---

## 四、运行时手动切换机制

### 4.1 切换控件的显示条件

页面底部的切换按钮（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L502-L506)）：

```jsx
{!settings?.color && <ColorToggle />}   {/* 仅当 settings.yaml 未固定 color 时显示 */}
{!settings.theme && <ThemeToggle />}    {/* 仅当 settings.yaml 未固定 theme 时显示 */}
```

即 **在 settings.yaml 中固定了 theme 或 color 后，对应的切换按钮会自动隐藏**。

### 4.2 切换流程

用户点击切换按钮 → 调用 Context 的 `setTheme` / `setColor` → 触发 Provider 的 effect：

| 操作 | 结果 |
|-----|------|
| 切换明暗模式 | 更新 Context state → `<html>` 切换 `.dark`/`.light` 类 → 写入 `localStorage["theme-mode"]` |
| 切换色板 | 更新 Context state → `<html>` 切换 `.theme-{color}` 类 → 写入 `localStorage["theme-color"]` |

切换值会被持久化到 localStorage，下次打开时优先使用。

---

## 五、主题 CSS 变量体系

### 5.1 基础层：色板变量（`theme.css`）

[theme.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/theme.css) 为每个色板定义 9 级颜色梯度 + 2 个 Logo 渐变变量：

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

注意颜色值使用 `R G B` 空格分隔格式（没有逗号），这是为了配合 Tailwind 的 `<alpha-value>` 通道语法。

### 5.2 语义层：明暗模式变量（`globals.css`）

[globals.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/globals.css#L64-L74) 根据明暗模式，从色板变量中挑选合适的级别：

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

**色板变量是基础，明暗模式变量是对色板变量的语义化选择**。改变色板会同时影响明/暗两种模式的颜色。

### 5.3 Tailwind 工具类桥接（`tailwind.config.js`）

[tailwind.config.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/tailwind.config.js#L19-L32) 将 CSS 变量注册为 Tailwind 颜色：

```js
colors: {
  theme: {
    50:  "rgb(var(--color-50) / <alpha-value>)",
    100: "rgb(var(--color-100) / <alpha-value>)",
    200: "rgb(var(--color-200) / <alpha-value>)",
    // ... 直到 900
  },
}
```

组件中可直接使用 `bg-theme-100`、`text-theme-800`、`bg-theme-500/20` 等 Tailwind 类，实际颜色由当前激活的色板动态决定。

### 5.4 特殊处理：`theme-white`

白色色板比较特殊——它的 `--color-100` 到 `--color-700` 都是灰色值，直接用 Tailwind 的半透明类会导致白底上几乎不可见。因此 [theme.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/theme.css#L17-L29) 对白色色板写了额外的固定值覆盖：

```css
.theme-white .bg-theme-100\/20:not([class^="backdrop-blur"]) {
  background-color: rgb(245, 245, 245);  /* 固定浅灰色，不随变量变 */
}
```

---

## 六、CSS 加载顺序与层叠覆盖

### 6.1 构建时 CSS（`_app.jsx` import）

```jsx
import "styles/globals.css";   // 第 1 个
import "styles/manrope.css";   // 第 2 个
import "styles/theme.css";     // 第 3 个
```

Next.js 在构建时将这三个文件合并打包，**import 顺序决定 CSS 在产物中的顺序**，后引入者覆盖前者。

### 6.2 运行时 CSS（`_document.jsx` `<link>`）

[_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/_document.jsx#L9-L10)：

```jsx
<link rel="preload" href="/api/config/custom.css" as="style" />
<link rel="stylesheet" href="/api/config/custom.css" />
```

`custom.css` 通过独立的 `<link>` 标签注入 HTML `<head>`。API 端点 [[path].js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/api/config/[path].js#L13-L34) 从 `config/custom.css` 读取文件内容返回；文件不存在则返回空内容。

### 6.3 运行时 JS（`index.jsx` `<Script>`）

[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L439)：

```jsx
<Script src="/api/config/custom.js" />
```

### 6.4 完整加载顺序图

```
HTML <head> 中样式资源加载顺序（浏览器从上到下解析）：

  │
  ▼
┌──────────────────────────────────────────────────────┐
│ 1. Next.js 构建产物 CSS（包含 3 个 import 合并）        │
│    ┌──────────────────────────────────────────────┐  │
│    │ a. globals.css                               │  │
│    │    - @import tailwindcss（Tailwind v4 基础） │  │
│    │    - Tailwind 工具类（含 theme-{N} 颜色映射） │  │
│    │    - @layer base 边框颜色兼容                │  │
│    │    - html/body/#__next 全局样式               │  │
│    │    - .light / .dark 语义化 CSS 变量          │  │
│    │    - 滚动条、图表间距等全局规则               │  │
│    ├──────────────────────────────────────────────┤  │
│    │ b. manrope.css                               │  │
│    │    - Manrope 字体 @font-face                 │  │
│    ├──────────────────────────────────────────────┤  │
│    │ c. theme.css                                 │  │
│    │    - 21 个 .theme-{color} 的基础 CSS 变量    │  │
│    │    - .theme-white 的特殊覆盖规则             │  │
│    └──────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────┤
│ 2. <link rel="stylesheet" href="/api/config/custom.css"> │
│    用户自定义 CSS，优先级最高                          │
└──────────────────────────────────────────────────────┘
  │
  ▼
<body> 渲染后加载：
┌──────────────────────────────────────────────────────┐
│ 3. <Script src="/api/config/custom.js">              │
│    用户自定义 JS，可动态操作 DOM/CSS                   │
└──────────────────────────────────────────────────────┘
```

### 6.5 覆盖关系分析

CSS 层叠优先级由 **来源 → 特异性 → 出现顺序** 共同决定。在此项目中：

| 覆盖能力 | 样式来源 | 说明 |
|---------|---------|------|
| 🥇 最强 | `custom.css` | 最后加载的 `<link>` 样式表，相同特异性下必然胜出 |
| 🥈 | `theme.css` | 构建时 CSS 的最后一部分，定义色板变量 |
| 🥉 | `globals.css` | 构建时 CSS 的中间部分，含 Tailwind 基础和语义变量 |
| 最弱 | Tailwind 基础层 | 在 `globals.css` 中最先 import |

**实际覆盖示例**：

| 需求 | 在 custom.css 中写法 | 为什么有效 |
|-----|-------------------|-----------|
| 改 slate 色板的主题色 | `.theme-slate { --color-500: 255 0 0; }` | 特异性相同（单类选择器），custom.css 后加载 |
| 强制所有卡片不透明 | `.service-card { opacity: 1 !important; }` | `!important` 提升优先级 |
| 改暗色模式背景 | `.dark { --bg-color: 10 10 30; }` | 同上 |
| 改特定服务卡片 | `#myserviceid .card-body { padding: 8px; }` | id 选择器特异性高于类选择器 |

---

## 七、`themes.js` 的独立用途

[themes.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/styles/themes.js) 是一个与 CSS 变量体系**独立**的 JS 色板映射：

```js
{
  slate: {
    light: "#f8fafc",      // 浅色模式的背景色（十六进制）
    dark: "#1e293b",       // 深色模式的背景色（十六进制）
    iconStart: "#94a3b8",  // Logo 渐变起始色
    iconEnd: "#334155",    // Logo 渐变结束色
  },
  // ... 其他 20 个色板
}
```

它**不参与 CSS 变量定义**，而是在 JS 代码中需要十六进制颜色值时使用，例如：

- `<meta name="theme-color">` 和 `<meta name="msapplication-TileColor">`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L434-L435)）
- PWA manifest 中的 `theme_color` 和 `background_color`

---

## 八、骨架文件与首次运行

[src/skeleton/](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/skeleton/) 目录存放首次运行时的配置模板：

| 骨架文件 | 默认内容 |
|---------|---------|
| `settings.yaml` | 仅含 `providers` 占位，不设置 theme/color |
| `custom.css` | 空文件 |
| `custom.js` | 空文件 |
| `services.yaml` / `bookmarks.yaml` / `widgets.yaml` | 示例配置 |

当 `config/` 目录下缺少对应文件时，[checkAndCopyConfig()](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/config/config.js#L15-L50) 会自动将骨架文件复制过去。

---

## 九、总结速查表

### 主题来源优先级

```
settings.yaml > localStorage > 系统偏好 > 默认值
```

### 初始化时序

```
SSR → Hydration(读取localStorage) → 第一轮useEffect(写入DOM)
  → 第二轮useEffect(settings.yaml覆盖) → 第三轮useEffect(最终写入DOM)
```

### CSS 覆盖优先级

```
custom.css > theme.css > globals.css > Tailwind 基础层
```

### 主题生效位置

| 位置 | 作用 |
|-----|------|
| `<html class="dark">` | 切换明暗模式 CSS 变量 |
| `<html class="theme-slate">` | 切换色板 CSS 变量 |
| `localStorage.theme-mode` | 持久化用户选择的明暗模式 |
| `localStorage.theme-color` | 持久化用户选择的色板 |
