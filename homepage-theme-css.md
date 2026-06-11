# Homepage 主题配置与自定义 CSS 运作机制

本文档通过追踪源码，详细说明 Homepage 项目的主题配置加载、样式注入和覆盖顺序。

---

## 一、整体架构概览

Homepage 的主题系统由两个维度组成：

| 维度 | 控制什么 | 可选值 | CSS 机制 |
|------|---------|--------|---------|
| **Theme（明暗模式）** | 背景色深浅、滚动条颜色 | `dark` / `light` | `.dark` / `.light` 类切换 CSS 变量 |
| **Color（色板）** | 前景色的色相系列 | `slate`, `gray`, `zinc`, `red`, `blue` 等 21 种 | `.theme-{color}` 类切换 CSS 变量 |

两者组合决定最终页面的颜色外观。例如 `.dark.theme-slate` 表示暗色模式 + slate 色板。

---

## 二、主题配置的来源与加载

### 2.1 配置文件：`settings.yaml`

用户在 `config/settings.yaml` 中指定 `theme` 和 `color`：

```yaml
theme: dark    # 或 light
color: slate   # 21 种色板之一
```

### 2.2 配置读取流程

1. **服务端**：`getSettings()` 函数（[config.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/config/config.js#L82-L103)）从 `config/settings.yaml` 读取并解析 YAML，同时支持环境变量替换。
2. **构建时注入**：`getStaticProps()`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L55-L95)）在 Next.js 构建阶段调用 `getSettings()`，将配置作为 `initialSettings` 传递给页面组件。
3. **API 端点**：`/api/theme`（[theme.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/api/theme.js#L1-L13)）也读取 settings 返回 `{ color, theme }`，供运行时查询使用。

### 2.3 默认值

- `theme` 默认：`dark`
- `color` 默认：`slate`

如果用户未在 `settings.yaml` 中设置这些字段，则使用默认值。

---

## 三、主题 CSS 变量体系

### 3.1 色板变量：`theme.css`

文件 [theme.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/theme.css) 定义了每个色板的 CSS 变量集合。每个 `.theme-{color}` 类定义了：

- `--color-50` 到 `--color-900`：9 级颜色梯度，使用 `R G B` 空格分隔格式（供 Tailwind `<alpha-value>` 通道使用）
- `--color-logo-start` / `--color-logo-stop`：Logo 渐变色

例如 `.theme-slate`：

```css
.theme-slate {
  --color-50: 248 250 252;
  --color-100: 241 245 249;
  --color-200: 226 232 240;
  --color-300: 203 213 225;
  --color-400: 148 163 184;
  --color-500: 100 116 139;
  --color-600: 71 85 105;
  --color-700: 51 65 85;
  --color-800: 30 41 59;
  --color-900: 15 23 42;
  --color-logo-start: 148 163 184;
  --color-logo-stop: 51 65 85;
}
```

### 3.2 明暗模式变量：`globals.css`

文件 [globals.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/globals.css) 中的 `.light` 和 `.dark` 类根据明暗模式定义派生变量：

```css
.light {
  --bg-color: var(--color-50);       /* 浅色背景 */
  --scrollbar-thumb: rgb(var(--color-300));
  --scrollbar-track: rgb(var(--color-200));
}

.dark {
  --bg-color: var(--color-800);      /* 深色背景 */
  --scrollbar-thumb: rgb(var(--color-600));
  --scrollbar-track: rgb(var(--color-700));
}
```

这里 `--bg-color` 引用的是 `--color-50` / `--color-800`，而这些变量由当前激活的 `.theme-{color}` 类提供。因此 **色板变量是基础，明暗模式变量是对色板变量的语义化组合**。

### 3.3 Tailwind 的 `theme` 色映射

在 [tailwind.config.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/tailwind.config.js#L19-L32) 中，将 CSS 变量桥接到 Tailwind 工具类：

```js
colors: {
  theme: {
    50:  "rgb(var(--color-50) / <alpha-value>)",
    100: "rgb(var(--color-100) / <alpha-value>)",
    // ... 直到 900
  },
}
```

这使得组件中可以直接使用 `bg-theme-100`、`text-theme-800` 等 Tailwind 类名，实际颜色由当前激活的色板决定。

---

## 四、运行时主题切换机制

### 4.1 Context 层

两个 React Context 管理运行时的主题状态：

| Context | 文件 | 状态 | 修改 DOM |
|---------|------|------|---------|
| `ThemeContext` | [theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/theme.jsx) | `theme`（`dark`/`light`） | 切换 `<html>` 的 `.dark`/`.light` 类 |
| `ColorContext` | [color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/color.jsx) | `color`（如 `slate`） | 切换 `<html>` 的 `.theme-{color}` 类 |

两者都在 `_app.jsx` 中通过 Provider 嵌套包裹整个应用：

```jsx
<ColorProvider>
  <ThemeProvider>
    <SettingsProvider>
      <TabProvider>
        <Component {...pageProps} />
      </TabProvider>
    </SettingsProvider>
  </ThemeProvider>
</ColorProvider>
```

### 4.2 初始值优先级

**ThemeContext**（[theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/theme.jsx#L3-L17)）：

1. `initialTheme` prop（来自 settings.yaml 的 `theme` 字段）
2. `localStorage` 中的 `theme-mode`
3. 系统偏好 `prefers-color-scheme: dark`
4. 兜底默认值：`dark`

**ColorContext**（[color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/color.jsx#L5-L15)）：

1. `initialTheme` prop（来自 settings.yaml 的 `color` 字段）
2. `localStorage` 中的 `theme-color`
3. 兜底默认值：`slate`

### 4.3 DOM 操作

**ThemeContext** 的 `rawSetTheme`（[theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/theme.jsx#L24-L31)）：
- 移除旧的 `.dark`/`.light` 类
- 添加新的类
- 同步写入 `localStorage`

**ColorContext** 的 `rawSetColor`（[color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/contexts/color.jsx#L22-L30)）：
- 使用 `lastColor` 变量追踪上一次的色板，移除 `theme-{lastColor}` 类
- 添加 `theme-{newColor}` 类
- 同步写入 `localStorage`

### 4.4 页面级主题应用

在 [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx) 的 `Wrapper` 组件中（第 517-593 行），有一个额外的 `useEffect` 直接操作 `<html>` 元素的类名：

```jsx
useEffect(() => {
  // 明暗模式
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

  // 色板
  const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
  const themeClassesToRemove = Array.from(html.classList).filter(
    (cls) => cls.startsWith("theme-") && cls !== desiredThemeClass,
  );
  html.classList.remove(...themeClassesToRemove);
  html.classList.add(desiredThemeClass);
}, [backgroundImage, opacity, theme, color, initialSettings.color]);
```

这段代码是对 Context 层操作的补充，确保在背景图片等依赖项变化时也能正确同步 `<html>` 类名。

### 4.5 用户手动切换

页面底部的切换控件（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L502-L506)）：

- `ColorToggle`：仅在 `settings.color` 未设置时显示，允许用户在 21 种色板间切换
- `ThemeToggle`：仅在 `settings.theme` 未设置时显示，允许用户在明暗间切换

也就是说，**在 settings.yaml 中固定了 theme 或 color 后，对应的切换按钮会自动隐藏**。

---

## 五、CSS 加载与注入顺序

### 5.1 构建时 CSS（在 `_app.jsx` 中引入）

```
1. styles/globals.css     ← Tailwind 基础 + 全局样式 + .light/.dark 变量
2. styles/manrope.css     ← Manrope 字体定义
3. styles/theme.css       ← 各色板的 CSS 变量定义
```

这三者由 Next.js 在构建时打包，按 import 顺序决定 CSS 层叠优先级（后引入者覆盖前者）。

### 5.2 运行时 CSS（在 `_document.jsx` 中注入）

```
4. /api/config/custom.css ← 用户自定义 CSS（通过 <link> 标签）
```

文件 [_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/_document.jsx#L1-L18) 中：

```jsx
<link rel="preload" href="/api/config/custom.css" as="style" />
<link rel="stylesheet" href="/api/config/custom.css" />
```

该 API 端点（[[path].js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/api/config/[path].js#L1-L34)）从 `config/custom.css` 读取文件内容并以 `text/css` Content-Type 返回。如果文件不存在则返回空内容。

### 5.3 运行时 JS

```
5. /api/config/custom.js ← 用户自定义 JS（通过 <Script> 标签）
```

在 [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L439) 中：

```jsx
<Script src="/api/config/custom.js" />
```

### 5.4 完整加载与覆盖顺序

```
┌─────────────────────────────────────────────────────────┐
│ 1. globals.css (构建时打包)                               │
│    - @import tailwindcss (Tailwind v4 基础层)             │
│    - Tailwind 工具类 (包括 theme-{N} 映射)                │
│    - @layer base 边框颜色兼容性                            │
│    - html/body 全局样式                                    │
│    - .light / .dark CSS 变量                              │
│    - 滚动条、布局等全局规则                                │
├─────────────────────────────────────────────────────────┤
│ 2. manrope.css (构建时打包)                               │
│    - Manrope 字体 @font-face                             │
├─────────────────────────────────────────────────────────┤
│ 3. theme.css (构建时打包)                                 │
│    - .theme-white / .theme-slate / ... 的 CSS 变量        │
│    - .theme-white 的特殊覆盖规则                          │
├─────────────────────────────────────────────────────────┤
│ 4. custom.css (运行时 <link>)                             │
│    - 用户自定义 CSS，优先级最高                            │
│    - 可覆盖上述所有构建时样式                              │
├─────────────────────────────────────────────────────────┤
│ 5. custom.js (运行时 <Script>)                            │
│    - 用户自定义 JS，可动态修改 DOM/CSS                     │
└─────────────────────────────────────────────────────────┘
```

**关键结论**：`custom.css` 作为运行时注入的 `<link>` 样式表，其样式在级联顺序上位于所有构建时 CSS 之后，因此可以覆盖任何默认样式。

---

## 六、`themes.js` 的用途

[themes.js](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/styles/themes.js) 提供了色板的**十六进制颜色值**映射，每个色板包含 `light`、`dark`、`iconStart`、`iconEnd` 四个属性。

此文件**不参与 CSS 变量定义**，它的用途是在 JS 中获取特定色板的固定颜色值，例如：

- 设置 `<meta name="theme-color">` 和 `<meta name="msapplication-TileColor">`（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/pages/index.jsx#L434-L435)）
- 在 `site.webmanifest` 中设置 PWA 的 `theme_color` 和 `background_color`

---

## 七、特殊处理：`theme-white`

[theme.css](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/styles/theme.css#L1-L29) 中 `.theme-white` 有额外的覆盖规则：

```css
.theme-white .bg-theme-100\/20:not([class^="backdrop-blur"]),
.theme-white .dark\:bg-white\/5:not([class^="backdrop-blur"]) {
  background-color: rgb(245, 245, 245);
}
```

因为白色色板的 `--color-100` 和 `--color-800` 都是 120 120 120 / 255 255 255，直接使用 `bg-theme-100/20` 会导致白底上几乎不可见，所以需要用固定颜色值覆盖。

---

## 八、骨架文件与首次运行

项目在 [src/skeleton/](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/skeleton/) 目录下提供了默认配置文件模板：

- `settings.yaml` — 最小化配置，仅含 providers 占位
- `custom.css` — 空文件

当 `config/` 目录下不存在对应配置文件时，[checkAndCopyConfig()](file:///d:/fz/0601/solo-dogfeeding/code/202-homepage/src/utils/config/config.js#L15-L50) 会将骨架文件复制到 config 目录，确保首次运行不会出错。

---

## 九、自定义 CSS 实践要点

1. **覆盖色板变量**：可以在 `custom.css` 中重新定义 `.theme-slate { --color-500: ... }` 等变量
2. **覆盖全局变量**：可以覆盖 `.dark { --bg-color: ... }` 等
3. **直接覆盖样式**：可以针对任何组件的类名/id 编写覆盖规则
4. **服务/书签 id 定位**：在 services.yaml 中设置 `id`，即可用 `#myserviceid` 选择器精准定位
5. **custom.css 优先级最高**：因为它最后加载，同样的特异性下必然胜出；如需更强覆盖可使用 `!important`
