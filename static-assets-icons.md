# 静态资源与图标系统代码脉络

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        Next.js 应用层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Pages SSR  │  │ API Routes  │  │     React Components    │  │
│  │ getStaticProps │ /api/theme  │  │ ResolvedIcon / Favicon  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        Context 状态层                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ ThemeContext │  │ ColorContext │  │   SettingsContext    │   │
│  │ (明暗模式)   │  │ (主题色系)   │  │  (用户配置聚合)      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                        样式 & 资源层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  theme.css   │  │ globals.css  │  │  Tailwind CSS v4    │   │
│  │ (CSS变量)    │  │ (全局样式)   │  │  (原子化工具类)      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      外部 CDN / 公共资源                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  jsDelivr    │  │  public/     │  │  config/custom.*    │   │
│  │  图标库CDN   │  │  静态文件    │  │  用户自定义资源     │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、资源定位系统

### 2.1 静态资源目录结构

| 路径 | 用途 | 访问方式 |
|------|------|---------|
| `public/` | Next.js 公共静态资源 | 根路径 `/xxx` 直接访问 |
| `public/locales/` | i18n 多语言 JSON 文件 | 通过 next-i18next 加载 |
| `public/*.png/svg/ico` | 站点元图标 (favicon/PWA manifest) | HTML `<link>` 标签引用 |
| `src/styles/` | 全局 CSS、字体文件 | 构建时打包注入 |
| `src/skeleton/` | 配置文件骨架模板 | 服务端首次启动复制到 `config/` |
| `config/` (运行时) | 用户配置目录 (YAML/custom.*) | 通过 `/api/config/[path]` 动态读取 |

关键配置文件：[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/next.config.js#L1-L19)

```javascript
// 允许的远程图片域名白名单
images: {
  remotePatterns: [
    { protocol: "https", hostname: "cdn.jsdelivr.net" },
  ],
  unoptimized: true, // 禁用Next.js图片优化，使用原始URL
}
```

### 2.2 公共资源访问入口

**PWA manifest 与站点图标**：在 [_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/_document.jsx#L1-L18) 中预注册：

```jsx
<Head>
  <link rel="manifest" href="/site.webmanifest?v=4" />
  <link rel="preload" href="/api/config/custom.css" as="style" />
  <link rel="stylesheet" href="/api/config/custom.css" />
</Head>
```

**用户自定义 CSS/JS**：通过 API Route 动态服务

[pages/api/config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/api/config/%5Bpath%5D.js#L1-L34)

```javascript
// 仅允许 custom.css 和 custom.js
if (!["custom.css", "custom.js"].includes(relativePath)) {
  return res.status(422).end("Unsupported file");
}
// 从 config/ 目录读取文件内容并返回对应 MIME 类型
const filePath = path.join(CONF_DIR, relativePath);
```

**首页 Head 标签资源注入**：在 [pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L407-L439)

```jsx
// 自定义 favicon 优先，否则使用默认 public/ 下的图标
settings.favicon ? (
  <link rel="icon" href={settings.favicon} />
) : (
  <>
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png?v=4" />
    <link rel="shortcut icon" href="/homepage.ico" />
    <link rel="mask-icon" href="/safari-pinned-tab.svg?v=4" />
  </>
)
// 注入 meta theme-color，动态绑定当前主题色
<meta name="theme-color" content={themes[settings.color || "slate"][settings.theme || "dark"]} />
// 注入用户自定义 JS
<Script src="/api/config/custom.js" />
```

### 2.3 字体资源

[src/styles/manrope.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/manrope.css) 定义 Manrope 字体的 `@font-face` 规则，字体文件存放在 `src/styles/font/` 下（`.ttf` 和 `.woff2` 格式），通过构建打包到静态产物中。

---

## 三、图标映射系统

### 3.1 核心解析组件：ResolvedIcon

所有服务卡片、书签、小组件的图标统一通过 [components/resolvedicon.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/resolvedicon.jsx#L1-L153) 解析渲染。

**图标解析优先级（从上到下匹配）：**

| 优先级 | 匹配规则 | 图标源 | 渲染方式 | 示例 |
|--------|----------|--------|----------|------|
| 1 | `http://` 或 `/` 开头 | 直接 URL / 本地路径 | `<Image />` | `https://example.com/logo.png` |
| 2 | `sh-` 前缀 | selfhst/icons (GitHub+jsDelivr) | `<Image />` | `sh-plex.svg`、`sh-jellyfin.webp`、`sh-sonarr` |
| 3 | `mdi-` 前缀 | @mdi/svg (Material Design Icons) | CSS mask + 主题色填充 | `mdi-home`、`mdi-cog#ff0000` |
| 4 | `si-` 前缀 | simple-icons (品牌图标) | CSS mask + 主题色填充 | `si-github`、`si-docker#2496ed` |
| 5 | `.svg` 后缀 | homarr-labs/dashboard-icons (svg/) | `<Image />` | `plex.svg` |
| 6 | `.webp` 后缀 | homarr-labs/dashboard-icons (webp/) | `<Image />` | `plex.webp` |
| 7 | `.png` 后缀 或 默认 | homarr-labs/dashboard-icons (png/) | `<Image />` | `plex.png`、`radarr` |

**图标集 CDN 基础 URL 映射**：

```javascript
const iconSetURLs = {
  mdi: "https://cdn.jsdelivr.net/npm/@mdi/svg@latest/svg/",
  si:  "https://cdn.jsdelivr.net/npm/simple-icons@latest/icons/",
};
// selfhst/icons
`https://cdn.jsdelivr.net/gh/selfhst/icons@main/${extension}/${iconName}.${extension}`
// dashboard-icons (svg/webp/png)
`https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/${type}/${iconName}.${type}`
```

### 3.2 MDI/SI 图标颜色映射机制

**mdi-/si- 前缀图标不直接渲染 `<img>`，而是使用 CSS mask 技术实现动态变色**：

```jsx
// resolvedicon.jsx L66-L96
<div style={{
  background: `${iconColor}`,  // 颜色层
  mask: `url(${iconSource}) no-repeat center / contain`,        // 图标形状遮罩
  WebkitMask: `url(${iconSource}) no-repeat center / contain`,
}} />
```

**颜色决策流程**：

1. **自定义十六进制颜色**：图标名尾部匹配 `#RRGGBB`，如 `mdi-home#ff5733`，直接使用该颜色
2. **settings.iconStyle === "theme"**：根据明暗模式选择色阶
   - 暗色模式：`rgb(var(--color-300) / var(--tw-text-opacity, 1))`
   - 亮色模式：`rgb(var(--color-900) / var(--tw-text-opacity, 1))`
3. **默认 logo 渐变**：使用主题渐变色
   - `linear-gradient(180deg, rgb(var(--color-logo-start)), rgb(var(--color-logo-stop)))`

### 3.3 图标使用入口

**Service 卡片**：[components/services/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/services/item.jsx#L44-L60)
```jsx
<ResolvedIcon icon={service.icon} />  // 默认 32x32
```

**Bookmark 卡片**：[components/bookmarks/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/bookmarks/item.jsx#L28-L45)
```jsx
// iconOnly 模式: 7x7
<div className="w-7 h-7">
  <ResolvedIcon icon={bookmark.icon} alt={bookmark.abbr} />
</div>
// 标准模式: 5x5
<div className="shrink-0 w-5 h-5">
  <ResolvedIcon icon={bookmark.icon} alt={bookmark.abbr} />
</div>
```

### 3.4 react-icons 内联图标库

除了 CDN 远程图标，项目还内置依赖 [react-icons](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/package.json#L39) (`^5.6.0`)，用于内部 UI 元素：

```jsx
// 示例：pages/index.jsx L17 使用错误图标
import { BiError } from "react-icons/bi";
<BiError className="float-right w-6 h-6" />
```

### 3.5 动态 Favicon 生成

[components/favicon.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/favicon.jsx#L1-L114) 在客户端运行时，将 SVG 序列化 → Base64 → 绘制到 Canvas → 导出为 ICO 格式并注入 `<head>`，使其颜色跟随当前主题色系：

```javascript
// 1. 从 ColorContext 获取主题色，渲染 SVG（含渐变）
const { iconStart, iconEnd } = themes[color];
// 2. XMLSerializer 将 SVG DOM 转为字符串
const xml = new XMLSerializer().serializeToString(svg);
// 3. Base64 编码，创建 Image 对象
const svg64 = Buffer.from(xml).toString("base64");
img.src = "data:image/svg+xml;base64," + svg64;
// 4. Canvas 绘制后导出为 data URL，创建 <link rel="shortcut icon">
canvas.getContext("2d").drawImage(img, 0, 0);
link.href = canvas.toDataURL("image/x-icon");
```

---

## 四、主题适配系统

主题系统采用 **双层解耦架构**：明暗模式（Theme）× 主题色系（Color），分别由两个独立 Context 管理。

### 4.1 ThemeContext：明暗模式

[utils/contexts/theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/contexts/theme.jsx#L1-L46)

**初始化优先级**：
1. 父组件传入 `initialTheme` prop
2. `localStorage["theme-mode"]` 持久化值
3. 系统偏好：`window.matchMedia("(prefers-color-scheme: dark)")`
4. 兜底默认：`"dark"`

**切换机制**：
```javascript
const rawSetTheme = (rawTheme) => {
  const root = window.document.documentElement;
  const isDark = rawTheme === "dark";
  root.classList.remove(isDark ? "light" : "dark");
  root.classList.add(rawTheme);       // 给 <html> 添加 .dark / .light 类
  localStorage.setItem("theme-mode", rawTheme);
};
```

**CSS 变量联动**（[globals.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/globals.css#L64-L74)）：
```css
.light {
  --bg-color: var(--color-50);
  --scrollbar-thumb: rgb(var(--color-300));
  --scrollbar-track: rgb(var(--color-200));
}
.dark {
  --bg-color: var(--color-800);
  --scrollbar-thumb: rgb(var(--color-600));
  --scrollbar-track: rgb(var(--color-700));
}
```

### 4.2 ColorContext：主题色系

[utils/contexts/color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/contexts/color.jsx#L1-L45)

**初始化优先级**：
1. 父组件传入 `initialTheme` prop
2. `localStorage["theme-color"]` 持久化值
3. 兜底默认：`"slate"`

**切换机制**：
```javascript
const rawSetColor = (rawColor) => {
  const root = window.document.documentElement;
  root.classList.remove(`theme-${lastColor}`);   // 移除旧色系类
  root.classList.add(`theme-${rawColor}`);       // 添加新色系类 .theme-xxx
  localStorage.setItem("theme-color", rawColor);
  lastColor = rawColor;
};
```

### 4.3 主题色值定义：themes.js 与 theme.css

**色板数据文件**：[utils/styles/themes.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/styles/themes.js#L1-L142)

共定义 **23 个** 主题色系（white/slate/gray/zinc/neutral/stone/red/orange/amber/yellow/lime/green/emerald/teal/cyan/sky/blue/indigo/violet/purple/fuchsia/pink/rose），每个色系定义：

| 字段 | 用途 |
|------|------|
| `light` | 亮色模式下主题色（HTML meta） |
| `dark` | 暗色模式下主题色（HTML meta） |
| `iconStart` | Favicon 渐变起始色 / 图标渐变色起点 |
| `iconEnd` | Favicon 渐变结束色 / 图标渐变色终点 |

**CSS 变量注入文件**：[styles/theme.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/theme.css)

每个 `.theme-{name}` 类定义一套完整的 CSS 自定义属性（共 12 个变量）：

```css
.theme-slate {
  --color-50: 248 250 252;      /* 最浅色 */
  --color-100: 241 245 249;
  --color-200: 226 232 240;
  --color-300: 203 213 225;     /* 暗色模式下的图标色 */
  --color-400: 148 163 184;
  --color-500: 100 116 139;
  --color-600: 71 85 105;
  --color-700: 51 65 85;
  --color-800: 30 41 59;        /* 暗色模式下的背景色 */
  --color-900: 15 23 42;        /* 亮色模式下的图标色 */
  --color-logo-start: 148 163 184;  /* 图标渐变起始 */
  --color-logo-stop: 51 65 85;      /* 图标渐变结束 */
}
```

色值使用空格分隔的 **RGB 分量格式**（而非 `rgb()`），配合 Tailwind 的 alpha-value 语法使用：

### 4.4 Tailwind 主题映射

[tailwind.config.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/tailwind.config.js#L17-L34)

将 CSS 变量映射为 Tailwind 语义色：

```javascript
colors: {
  theme: {
    50:  "rgb(var(--color-50)  / <alpha-value>)",
    100: "rgb(var(--color-100) / <alpha-value>)",
    200: "rgb(var(--color-200) / <alpha-value>)",
    // ... 300 ~ 900
  },
}
darkMode: "class",  // 使用类名策略切换暗色
```

**Tailwind v4 配置**：[globals.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/globals.css#L1-L7)

```css
@import 'tailwindcss';
@config '../../tailwind.config.js';
@theme { --breakpoint-3xl: 112rem; }
```

### 4.5 完整应用流程

在 [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/_app.jsx#L73-L98) 中从外到内的 Provider 嵌套：

```
ColorProvider → ThemeProvider → SettingsProvider → TabProvider → Page
```

外层 Wrapper（[pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L517-L593)）负责同步 `<html>` 类名：

```javascript
// 同步明暗类
html.classList.toggle("dark", theme === "dark");
html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

// 同步主题色系类
const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
html.classList.remove(...themeClassesToRemove);
html.classList.add(desiredThemeClass);
```

**首页 settings → theme 同步**（[pages/index.jsx L234-L247](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L234-L247)）：

```javascript
useEffect(() => {
  if (settings.theme && theme !== settings.theme) setTheme(settings.theme);
  if (settings.color && color !== settings.color) setColor(settings.color);
}, [settings, color, setColor, theme, setTheme]);
```

**服务端 theme API**：[pages/api/theme.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/api/theme.js#L1-L14) 提供 SSR 时的初始主题读取：
```javascript
return res.status(200).json({
  color: settings.color || "slate",
  theme: settings.theme || "dark",
});
```

### 4.6 背景图适配

支持配置化背景（[pages/index.jsx L520-L575](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L520-L575)）：

```javascript
// settings.yaml 中的配置格式
background: string | {
  image: string;
  opacity: number;     // 0-100，转换为叠加层的透明度
  blur?: string;       // backdrop-blur 程度
  saturate?: number;   // backdrop-saturate
  brightness?: number; // backdrop-brightness
}
```

DOM 结构：
```
<div id="background">  /* fixed全屏，z-index:0，放置背景图 + 主题色遮罩 */
<div id="page_wrapper">  /* 内容区容器 */
  <div id="inner_wrapper">  /* 应用 backdrop-filter 滤镜 */
```

---

## 五、加载与缓存策略

### 5.1 数据层缓存：SWR

[SWR](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/package.json#L41) (`stale-while-revalidate`) 是项目统一的数据获取与缓存库。

**全局配置**（[_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/_app.jsx#L75-L79)）：
```jsx
<SWRConfig value={{
  fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
}}>
```

**SSR Fallback 预填充**：`getStaticProps` 构建时预取数据，通过 `fallback` 注入 SWR 缓存，避免客户端首屏二次请求（[pages/index.jsx L55-L77](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L55-L77)）：

```javascript
return {
  props: {
    fallback: {
      "/api/services": services,   // 构建时获取的服务数据
      "/api/bookmarks": bookmarks,
      "/api/widgets": widgets,
      "/api/hash": false,
    },
  },
};
```

首页二次包裹（[L186-L191](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L186-L191)）：
```jsx
<SWRConfig value={{ fallback, fetcher: /* 同样的fetcher */ }}>
  <Home initialSettings={initialSettings} />
</SWRConfig>
```

### 5.2 配置变更检测：Hash 轮询

通过 `/api/hash` 检测配置文件变化，窗口重新获得焦点时触发检查（[pages/index.jsx L100-L131](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L100-L131)）：

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

// 窗口聚焦时重新校验 hash
useEffect(() => {
  if (windowFocused) mutateHash();
}, [windowFocused, mutateHash]);

// hash 变化则触发 ISR 重新验证 + 刷新页面
useEffect(() => {
  const previousHash = localStorage.getItem("hash");
  if (previousHash && previousHash !== hashData.hash) {
    localStorage.setItem("hash", hashData.hash);
    fetch("/api/revalidate").then((res) => {
      if (res.ok) window.location.reload();
    });
  }
}, [hashData]);
```

配合自定义 Hook：[utils/hooks/window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/hooks/window-focus.js#L1-L26) 监听 `window.focus/blur` 事件。

### 5.3 内存缓存：memory-cache

服务端环境变量使用 [`memory-cache`](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/package.json#L29) 包做进程内缓存，避免每次配置读取都遍历 `process.env`：

[utils/config/config.js L52-L62](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/config/config.js#L52-L62)
```javascript
function getCachedEnvironmentVars() {
  let cachedVars = cache.get(cacheKey);
  if (!cachedVars) {
    cachedVars = Object.entries(process.env).filter(
      ([key]) => key.includes("HOMEPAGE_VAR_") || key.includes("HOMEPAGE_FILE_"),
    );
    cache.put(cacheKey, cachedVars);  // 进程内常驻缓存
  }
  return cachedVars;
}
```

### 5.4 组件懒加载：next/dynamic

**Toggle 组件 SSR 禁用**（因为需要 localStorage，服务端无此 API）：
[pages/index.jsx L30-L40](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L30-L40)

```javascript
const ThemeToggle = dynamic(() => import("components/toggles/theme"), { ssr: false });
const ColorToggle = dynamic(() => import("components/toggles/color"), { ssr: false });
const Version     = dynamic(() => import("components/version"),          { ssr: false });
```

**Widget 组件全量动态导入**：[widgets/components.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/widgets/components.js#L1-L167)

将约 150+ 个 widget 组件全部通过 `next/dynamic` 懒加载，避免打包时全部纳入首屏 chunk，仅在实际使用该类型 widget 时才请求对应代码块：

```javascript
const components = {
  adguard: dynamic(() => import("./adguard/component")),
  plex:    dynamic(() => import("./plex/component")),
  sonarr:  dynamic(() => import("./sonarr/component")),
  // ... 约 150 个
};
```

### 5.5 静态资源版本化（Cache Busting）

public/ 下的静态资源通过 query 参数版本号绕过浏览器缓存（[pages/index.jsx L427-L431](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx#L427-L431)）：

```html
/apple-touch-icon.png?v=4
/favicon-32x32.png?v=4
/safari-pinned-tab.svg?v=4
/site.webmanifest?v=4
```

### 5.6 配置文件自动初始化

首次启动时自动从骨架模板复制到 config 目录（[utils/config/config.js L15-L50](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/config/config.js#L15-L50)）：

```javascript
export default function checkAndCopyConfig(config) {
  if (!existsSync(CONF_DIR)) mkdirSync(CONF_DIR, { recursive: true });
  const configYaml = join(CONF_DIR, config);
  if (!existsSync(configYaml)) {
    const configSkeleton = join(process.cwd(), "src", "skeleton", config);
    copyFileSync(configSkeleton, configYaml);
  }
  // 同时验证 yaml 语法有效性
  yaml.load(readFileSync(configYaml, "utf8"));
}
```

配置目录可通过环境变量 `HOMEPAGE_CONFIG_DIR` 覆盖，默认 `<cwd>/config`。

---

## 六、关键数据流

### 6.1 图标渲染数据流

```
services.yaml / bookmarks.yaml
        │
        ▼
  servicesResponse() / bookmarksResponse()
  [api-response.js] ── 合并 Docker/K8s 自动发现 + 手动配置
        │
        ▼
  SWR Fallback (SSR 构建时)
        │
        ▼
  useSWR("/api/services") ── /api/bookmarks
        │
        ▼
  <Item service={...}>
        │
        ▼
  <ResolvedIcon icon={service.icon} />
        │
        ├── http/ 开头 ──► <Image src="直接URL" />
        ├── sh- 前缀 ──► jsDelivr + selfhst/icons
        ├── mdi-/si- 前缀 ──► CSS mask + 主题色
        │     └── ColorContext + ThemeContext ──► iconColor 计算
        └── 默认 ──► dashboard-icons CDN
```

### 6.2 主题切换数据流

```
┌──────────────────────────────────────────────────────────────┐
│ 用户操作: 点击 ThemeToggle / ColorToggle                     │
│ 或 settings.yaml 中预设 theme/color                         │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
           setTheme() / setColor() [Context Provider]
                           │
            ┌──────────────┴──────────────┐
            ▼                             ▼
  localStorage["theme-mode"]     localStorage["theme-color"]
  documentElement.classList     documentElement.classList
  .dark / .light                .theme-{color}
            │                             │
            ▼                             ▼
  globals.css 中 CSS 变量         theme.css 中 .theme-*
  --bg-color / --scrollbar-*     12 级色阶变量 (--color-50 ~ 900)
            │                             │
            └──────────────┬──────────────┘
                           ▼
         Tailwind 原子类 bg-theme-700 / text-theme-200 / dark:...
         resolvedicon.jsx 中 mask 图标的 iconColor
         动态 Favicon (favicon.jsx) 的渐变颜色
```

---

## 七、核心文件索引

| 文件 | 职责 |
|------|------|
| [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/next.config.js) | Next.js 构建配置、远程图片白名单 |
| [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/_app.jsx) | 全局 Provider 嵌套、样式注入入口 |
| [src/pages/_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/_document.jsx) | SSR 文档模板、custom.css 预加载 |
| [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/index.jsx) | 首页主逻辑、SSR fallback、Head 资源注入 |
| [src/pages/api/theme.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/api/theme.js) | 服务端主题配置 API |
| [src/pages/api/config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/pages/api/config/%5Bpath%5D.js) | custom.css/custom.js 动态服务 |
| [src/components/resolvedicon.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/resolvedicon.jsx) | 图标解析与渲染核心组件 |
| [src/components/favicon.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/favicon.jsx) | 动态主题色 Favicon 生成 |
| [src/components/services/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/services/item.jsx) | 服务卡片 (含图标使用) |
| [src/components/bookmarks/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/components/bookmarks/item.jsx) | 书签卡片 (含图标使用) |
| [src/utils/contexts/theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/contexts/theme.jsx) | 明暗模式 Context |
| [src/utils/contexts/color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/contexts/color.jsx) | 主题色系 Context |
| [src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/contexts/settings.jsx) | 用户设置 Context |
| [src/utils/styles/themes.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/styles/themes.js) | 23 套主题色板定义 (JS 对象) |
| [src/styles/theme.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/theme.css) | 23 套主题 CSS 变量注入 |
| [src/styles/globals.css](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/styles/globals.css) | Tailwind 入口 + 全局基础样式 |
| [tailwind.config.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/tailwind.config.js) | Tailwind 自定义主题映射 |
| [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/config/config.js) | 配置文件加载、环境变量替换、骨架复制 |
| [src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/config/api-response.js) | 服务/书签/Widget 数据聚合响应 |
| [src/utils/hooks/window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/utils/hooks/window-focus.js) | 窗口焦点 Hook (配置刷新触发) |
| [src/widgets/components.js](file:///d:/fz/0601/solo-dogfeeding/code/211-homepage/src/widgets/components.js) | 150+ Widget 组件 dynamic import 映射 |
