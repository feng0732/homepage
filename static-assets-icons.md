# 静态资源与图标系统代码脉络

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        Next.js 应用层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Pages SSR  │  │ API Routes  │  │     React Components    │  │
│  │getStaticProps│ custom.css/js │  │ ResolvedIcon(核心)       │  │
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

关键配置文件：`next.config.js` L1-L19

```javascript
images: {
  remotePatterns: [
    { protocol: "https", hostname: "cdn.jsdelivr.net" },
  ],
  unoptimized: true,
}
```

### 2.2 公共资源访问入口

**PWA manifest 与站点图标**：在 `src/pages/_document.jsx` L1-L18 中预注册：

```jsx
<Head>
  <meta name="mobile-web-app-capable" content="yes" />
  <link rel="manifest" href="/site.webmanifest?v=4" crossOrigin="use-credentials" />
  <link rel="preload" href="/api/config/custom.css" as="style" />
  <link rel="stylesheet" href="/api/config/custom.css" />
</Head>
```

**manifest 链接跨域说明**：`<link rel="manifest">` 显式设置了 `crossOrigin="use-credentials"`，指示浏览器在请求 manifest.json 时携带凭证（cookies / HTTP auth），这对部署在需要鉴权的反向代理后的场景很重要。

**用户自定义 CSS/JS**：通过 API Route 动态服务

`src/pages/api/config/[path].js` L1-L34

```javascript
if (!["custom.css", "custom.js"].includes(relativePath)) {
  return res.status(422).end("Unsupported file");
}
const filePath = path.join(CONF_DIR, relativePath);
```

**首页 Head 标签资源注入**：在 `src/pages/index.jsx` L409-L439

```jsx
<title>{initialSettings.title || "Homepage"}</title>
<meta name="description" content={initialSettings.description || "..."} />
{settings.disableIndexing && <meta name="robots" content="noindex, nofollow" />}
{settings.base && <base href={settings.base} />}
{/* favicon 处理：settings.favicon 优先，否则使用 public/ 下的静态文件 */}
<meta name="msapplication-TileColor" content={themes[settings.color || "slate"][settings.theme || "dark"]} />
<meta name="theme-color" content={themes[settings.color || "slate"][settings.theme || "dark"]} />
<meta name="color-scheme" content="dark light" />
```

viewport meta 不在 `_document.jsx`，而是在 `src/pages/_app.jsx` L80-L86 的 `<Head>` 中：
```jsx
<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no" />
```

用户自定义 JS 通过 `next/script` 注入：`<Script src="/api/config/custom.js" />`

**PWA manifest 动态生成**：`src/pages/site.webmanifest.jsx` L1-L46 通过 `getServerSideProps` 在服务端动态生成 manifest JSON，`theme_color` 和 `background_color` 取自 themes.js 对应当前色系+明暗模式的值。

**browserconfig.xml 动态生成**：`src/pages/browserconfig.xml.jsx` L1-L28 同理动态生成，`<TileColor>` 取自 `themes[color][theme]`。

### 2.3 字体资源

`src/styles/manrope.css` 定义 Manrope 字体的 `@font-face` 规则，字体文件存放在 `src/styles/font/` 下（`.ttf` 和 `.woff2` 格式），通过构建打包到静态产物中。

---

## 三、图标映射系统

### 3.1 核心解析组件：ResolvedIcon

`src/components/resolvedicon.jsx` L1-L153 是整个图标系统的统一入口，负责将字符串形式的图标标识解析为具体的渲染结果。

**图标解析优先级（从上到下依次匹配，先命中先返回）：**

| 优先级 | 匹配规则 | 图标源 | 渲染方式 | 示例 |
|--------|----------|--------|----------|------|
| 1 | `http` 开头 或 `/` 开头 | 直接 URL / 本地路径 | Next `<Image />` | `https://example.com/logo.png`、`/icons/x.png` |
| 2 | `sh-` 前缀 | selfhst/icons (GitHub+jsDelivr) | Next `<Image />` | `sh-plex.svg`、`sh-jellyfin.webp`、`sh-sonarr`（无后缀默认 png） |
| 3 | `mdi-` / `si-` 前缀 | @mdi/svg 或 simple-icons (npm+jsDelivr) | CSS mask + 主题色填充 | `mdi-home`、`si-github`、`mdi-home-#ff00ff` |
| 4 | `.svg` 后缀 | homarr-labs/dashboard-icons (svg/) | Next `<Image />` | `plex.svg` |
| 5 | `.webp` 后缀 | homarr-labs/dashboard-icons (webp/) | Next `<Image />` | `plex.webp` |
| 6 | 其他（含 `.png` 或无后缀） | homarr-labs/dashboard-icons (png/) | Next `<Image />` | `plex.png`、`radarr` |

**图标集 CDN 基础 URL 映射**：

```javascript
const iconSetURLs = {
  mdi: "https://cdn.jsdelivr.net/npm/@mdi/svg@latest/svg/",
  si:  "https://cdn.jsdelivr.net/npm/simple-icons@latest/icons/",
};
// selfhst/icons
`https://cdn.jsdelivr.net/gh/selfhst/icons@main/${extension}/${iconName}.${extension}`
// dashboard-icons (svg/webp/png 三种后缀对应不同子目录)
`https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/${type}/${iconName}.${type}`
```

**前缀判断细节**：prefix 提取逻辑为 `icon.split("-")[0]`（resolvedicon.jsx L35），因此只要图标字符串以 `mdi-` 或 `si-` 开头就会进入 mask 渲染分支，即使后续部分含有点号。而 `.svg` / `.webp` 后缀判断只在 `prefix in iconSetURLs` 不匹配时才走到。

### 3.2 MDI/SI 图标颜色映射机制

**mdi-/si- 前缀图标不渲染 `<img>`，而是使用 CSS mask 技术实现动态着色**：

```jsx
// resolvedicon.jsx L84-L95
<div style={{
  width, height,
  maxWidth: "100%", maxHeight: "100%",
  background: `${iconColor}`,
  mask: `url(${iconSource}) no-repeat center / contain`,
  WebkitMask: `url(${iconSource}) no-repeat center / contain`,
}} />
```

**颜色决策流程（三级优先）**：

1. **自定义十六进制颜色**：图标名尾部匹配 `#RRGGBB` 后缀
   - 正则：`/[#][a-f0-9][a-f0-9][a-f0-9][a-f0-9][a-f0-9][a-f0-9]$/i`（resolvedicon.jsx L75）
   - **仅支持 6 位十六进制**，不支持 3 位简写（如 `#f00`），不支持 alpha 通道（如 `#ff000080`）
   - 格式为图标名后追加连字符+hex：如 `mdi-home-#ff00ff`、`si-github-#2496ed`
   - 匹配后从 iconName 中去除 `-{colorMatches[0]}` 部分，iconColor 直接使用 hex 字符串
2. **settings.iconStyle === "theme"**：根据明暗模式选择色阶
   - 暗色模式：`rgb(var(--color-300) / var(--tw-text-opacity, 1))`
   - 亮色模式：`rgb(var(--color-900) / var(--tw-text-opacity, 1))`
3. **默认（其他 iconStyle 值或未设置）**：使用主题渐变色
   - `linear-gradient(180deg, rgb(var(--color-logo-start)), rgb(var(--color-logo-stop)))`

### 3.3 ResolvedIcon 的全部实际使用入口

经代码全量 grep，ResolvedIcon 在以下 **8 个业务组件文件**中被 import 和使用（不含测试文件 `src/components/resolvedicon.test.jsx`）：

| 序号 | 文件 | 使用方式 | 图标尺寸 |
|------|------|---------|---------|
| 1 | `src/components/services/item.jsx` L54-L58 | `<ResolvedIcon icon={service.icon} />` | 默认 32×32（卡片主图和 compact 模式两处渲染） |
| 2 | `src/components/services/group.jsx` L46 | `<ResolvedIcon icon={layout.icon} />` | 默认 32×32（外层容器 w-7 h-7） |
| 3 | `src/components/bookmarks/item.jsx` L32-L42 | `<ResolvedIcon icon={bookmark.icon} alt={bookmark.abbr} />` | iconOnly 时 w-7 h-7；标准时 w-5 h-5（两处渲染） |
| 4 | `src/components/bookmarks/group.jsx` L42 | `<ResolvedIcon icon={layout.icon} />` | 默认 32×32（外层容器 w-7 h-7） |
| 5 | `src/components/quicklaunch.jsx` L307 | `{r.icon && <ResolvedIcon icon={r.icon} />}` | 默认 32×32（外层容器 w-5） |
| 6 | `src/components/widgets/logo/logo.jsx` L15 | `<ResolvedIcon icon={options.icon} width={48} height={48} />` | 48×48（唯一自定义非默认尺寸） |
| 7 | `src/widgets/glances/metrics/process.jsx` L11-L17 | 进程状态映射：`mdi-circle`/`mdi-circle-outline`/`mdi-circle-double`/`mdi-circle-opacity`/`mdi-decagram-outline`/`mdi-hexagon-outline`/`mdi-rhombus-outline`，均 `width={32} height={32}` | 32×32（外层容器 w-3 h-3） |
| 8 | `src/widgets/glances/metrics/containers.jsx` L11-L14 | 容器状态映射：`running`/`healthy`→`mdi-circle`，`paused`→`mdi-circle-outline`，`stopped`→`mdi-circle-double`，均 `width={32} height={32}` | 32×32（外层容器 w-3 h-3） |

**图标数据来源说明**：
- `service.icon` / `bookmark.icon`：来自用户 YAML 配置（services.yaml / bookmarks.yaml）
- `layout.icon`：来自 settings.yaml 中 layout 节点下分组级别的 `icon` 字段
- `options.icon`：来自 widgets.yaml 中 logo 类型 widget 的 `icon` 字段
- `r.icon`（quicklaunch）：来自搜索结果中的各条目标签（service、bookmark、widget 聚合）
- Glances 状态图标：代码内硬编码的 mdi- 前缀常量字符串

### 3.4 Logo Widget 的内联 SVG 降级

`src/components/widgets/logo/logo.jsx` L1-L75 中，当 `options.icon` 未配置时，降级渲染内联 SVG 作为 Homepage 默认 logo。该 SVG 使用 CSS 变量 `rgba(var(--color-logo-start))` / `rgba(var(--color-logo-stop))` 作为填充色，**随主题色系自动变化**。

### 3.5 react-icons 内联图标库

项目依赖 `package.json` 中的 `react-icons` (`^5.6.0`)，用于 UI 内部元素图标（与 ResolvedIcon 无关，是打包时的内联 SVG）：

```jsx
// services/group.jsx / bookmarks/group.jsx 中的折叠箭头
import { MdKeyboardArrowDown } from "react-icons/md";
<MdKeyboardArrowDown className="...text-theme-800 dark:text-theme-300..." />

// pages/index.jsx 中的错误图标
import { BiError } from "react-icons/bi";
<BiError className="float-right w-6 h-6" />
```

### 3.6 动态 Favicon 组件（未接入页面）

`src/components/favicon.jsx` L1-L114 定义了一个可跟随主题色系动态生成 favicon 的组件，其工作原理为：

```
ColorContext.color → themes[color].iconStart/iconEnd
        ↓
渲染带渐变色的 <Svg> 组件
        ↓
XMLSerializer 序列化 SVG DOM → Base64 编码
        ↓
创建 <img> 加载 data:image/svg+xml;base64,...
        ↓
img.onload → Canvas drawImage → canvas.toDataURL("image/x-icon")
        ↓
创建 <link rel="shortcut icon"> 注入 document.head
```

**事实核查：该组件未被任何页面或布局实际 import 和渲染。** 全项目搜索仅测试文件 `src/components/favicon.test.jsx` L8 引用了它。首页实际 favicon 处理逻辑在 `src/pages/index.jsx` L420-L432，使用的是：
- 用户配置 `settings.favicon` 时的静态链接，或
- `public/` 目录下的静态图片文件

**因此，当前页面的 favicon 不会随主题切换而更新。** favicon.jsx 是一个已实现但未接入生产页面的组件。

---

## 四、主题适配系统

主题系统采用 **双层解耦架构**：明暗模式（Theme）× 主题色系（Color），分别由两个独立 Context 管理。

### 4.1 ThemeContext：明暗模式

`src/utils/contexts/theme.jsx` L1-L46

**初始化优先级**（Provider 挂载时执行）：
1. 父组件传入 `initialTheme` prop（当前 `_app.jsx` 未传）
2. `localStorage["theme-mode"]` 持久化值
3. 系统偏好：`window.matchMedia("(prefers-color-scheme: dark)")`
4. 兜底默认：`"dark"`

**切换机制**：
```javascript
const rawSetTheme = (rawTheme) => {
  const root = window.document.documentElement;
  const isDark = rawTheme === "dark";
  root.classList.remove(isDark ? "light" : "dark");
  root.classList.add(rawTheme);
  localStorage.setItem("theme-mode", rawTheme);
};
```

**CSS 变量联动**（`src/styles/globals.css` L64-L74）：
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

`src/utils/contexts/color.jsx` L1-L45

**初始化优先级**（Provider 挂载时执行）：
1. 父组件传入 `initialTheme` prop（当前 `_app.jsx` 未传）
2. `localStorage["theme-color"]` 持久化值
3. 兜底默认：`"slate"`

**切换机制**：
```javascript
let lastColor = false;

const rawSetColor = (rawColor) => {
  const root = window.document.documentElement;
  root.classList.remove(`theme-${lastColor}`);
  root.classList.add(`theme-${rawColor}`);
  localStorage.setItem("theme-color", rawColor);
  lastColor = rawColor;
};
```

### 4.3 主题色值定义：themes.js 与 theme.css

**色板数据文件**：`src/utils/styles/themes.js` L1-L142

共定义 **23 个** 主题色系（white/slate/gray/zinc/neutral/stone/red/orange/amber/yellow/lime/green/emerald/teal/cyan/sky/blue/indigo/violet/purple/fuchsia/pink/rose），每个色系定义 4 个字段：

| 字段 | 用途 | 使用位置 |
|------|------|---------|
| `light` | 亮色模式下的主题色值 | HTML `<meta name="theme-color">`/`msapplication-TileColor`、PWA manifest、browserconfig.xml |
| `dark` | 暗色模式下的主题色值 | 同上 |
| `iconStart` | 渐变起始色 | favicon.jsx 的 SVG 渐变（未接入页面） |
| `iconEnd` | 渐变结束色 | 同上 |

**CSS 变量注入文件**：`src/styles/theme.css`

每个 `.theme-{name}` 类定义一套完整的 CSS 自定义属性（共 12 个变量）：

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

色值使用空格分隔的 **RGB 分量格式**（而非 `rgb()` 包裹），这是为了配合 Tailwind 的 `<alpha-value>` 语法使用。

**white 主题的特殊处理**：`src/styles/theme.css` L1-L29 中 `.theme-white` 定义了额外覆盖规则，对 `bg-theme-100/20` 和 `dark:bg-white/5` 等类做了硬编码颜色重写。

### 4.4 Tailwind 主题映射

`tailwind.config.js` L17-L34

```javascript
colors: {
  theme: {
    50:  "rgb(var(--color-50)  / <alpha-value>)",
    100: "rgb(var(--color-100) / <alpha-value>)",
    200: "rgb(var(--color-200) / <alpha-value>)",
    // ... 300 ~ 900
  },
}
darkMode: "class",
```

**Tailwind v4 配置**：`src/styles/globals.css` L1-L7

```css
@import 'tailwindcss';
@config '../../tailwind.config.js';
@theme { --breakpoint-3xl: 112rem; }
```

### 4.5 完整应用流程

**主题配置的数据来源**：
- 配置文件：`config/settings.yaml` 中的 `theme` 和 `color` 字段
- SSR 提取：`getStaticProps` → `getSettings()`（`src/utils/config/config.js`）直接在服务端读取 YAML → 得到 `initialSettings.color` / `initialSettings.theme`
- ⚠️ `src/pages/api/theme.js` **未被任何页面调用**，仅在测试文件 `src/__tests__/pages/api/theme.test.js` 中被引用，不参与页面渲染流程

**Provider 初始化（`src/pages/_app.jsx` L87-L96）**：

```
ColorProvider（无 initialTheme prop，走 localStorage → 默认 slate）
  └─► ThemeProvider（无 initialTheme prop，走 localStorage → 系统偏好 → 默认 dark）
       └─► SettingsProvider（接收 initialSettings prop）
            └─► TabProvider
                 └─► Wrapper（index.jsx 外层组件）
                      └─► Home（index.jsx 内层组件）
```

**两层 `<html>` 类名同步**（确保首屏无闪烁 + 后续变更响应）：

**① Wrapper 组件（`src/pages/index.jsx` L517-L563）**：挂载时和主题变化时直接操作 `document.documentElement.classList`

```javascript
useEffect(() => {
  const html = document.documentElement;
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

  const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
  const themeClassesToRemove = Array.from(html.classList).filter(
    (cls) => cls.startsWith("theme-") && cls !== desiredThemeClass,
  );
  if (themeClassesToRemove.length) html.classList.remove(...themeClassesToRemove);
  if (!html.classList.contains(desiredThemeClass)) html.classList.add(desiredThemeClass);
}, [theme, color, initialSettings.color /* 还有背景相关 */]);
```

Wrapper 中 theme 和 color 来自 Context（`useContext(ThemeContext/ColorContext)`），但 `initialSettings.color` 作为 Context 还未同步时的兜底值。

**② Context 内部 rawSetTheme/rawSetColor**：每当 setTheme/setColor 被调用，useEffect 触发后也会同步 `<html>` 类名（见 4.1/4.2 节）。两处同步是冗余但安全的设计。

**Home 组件 settings → Context 同步**（`src/pages/index.jsx` L234-L247）：

```javascript
useEffect(() => {
  if (settings.theme && theme !== settings.theme) {
    setTheme(settings.theme);
  }
  if (settings.color && color !== settings.color) {
    setColor(settings.color);
  }
}, [settings, color, setColor, theme, setTheme]);
```

这一步把来自 YAML 配置的主题设置"注入"到 Context，覆盖用户在 localStorage 中可能残留的旧值。

**用户交互切换**：
- 当 `settings.theme` 未设置时，页脚渲染 `ThemeToggle` 组件（`src/pages/index.jsx` L505）
- 当 `settings.color` 未设置时，页脚渲染 `ColorToggle` 组件（`src/pages/index.jsx` L503）
- Toggle 组件调用 Context 的 setTheme/setColor → 写入 localStorage + 同步 `<html>` 类名

### 4.6 背景图适配

支持配置化背景（`src/pages/index.jsx` L520-L575）：

```javascript
// settings.yaml 中的配置格式
background: string | {
  image: string;
  opacity: number;     // 0-100，最终转换为 1 - opacity/100
  blur?: string;
  saturate?: number;
  brightness?: number;
}
```

DOM 结构：
```
<div id="background">       /* fixed 全屏，z-index:0，背景图 + 主题色叠加层 */
<div id="page_wrapper">     /* 内容区容器 */
  <div id="inner_wrapper">  /* 应用 backdrop-filter 滤镜 */
```

背景叠加使用 `linear-gradient(rgb(var(--bg-color) / ${opacity}), ...)` 在背景图之上覆盖一层半透明主题色。

---

## 五、加载与缓存策略

### 5.1 客户端数据缓存：SWR

`package.json` 中 `swr` (`stale-while-revalidate`) 是项目统一的客户端数据获取与缓存库。

**全局配置**（`src/pages/_app.jsx` L75-L79）：
```jsx
<SWRConfig value={{
  fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()),
}}>
```

全局未设置 `dedupingInterval`、`revalidateOnFocus`、`revalidateIfStale`、`revalidateOnReconnect`，均使用 SWR 默认值（`dedupingInterval: 2000ms`、`revalidateOnFocus: true`、`revalidateIfStale: true`、`revalidateOnReconnect: true`），即：
- 相同 URL 的并发请求在 2 秒内自动去重
- 浏览器标签页重新获得焦点时自动重新验证
- 挂载组件时如果缓存已过期则自动重新验证
- 网络恢复连接时自动重新验证

**SSR Fallback 预填充**：`getStaticProps` 构建时预取数据，通过 `fallback` 注入 SWR 缓存，避免客户端首屏二次请求（`src/pages/index.jsx` L55-L77）：

```javascript
return {
  props: {
    fallback: {
      "/api/services": services,
      "/api/bookmarks": bookmarks,
      "/api/widgets": widgets,
      "/api/hash": false,
    },
  },
};
```

首页二次包裹（`src/pages/index.jsx` L186-L191）：
```jsx
<SWRConfig value={{ fallback, fetcher: /* 同样的fetcher */ }}>
  <Home initialSettings={initialSettings} />
</SWRConfig>
```

### 5.2 Widget 定时刷新：SWR refreshInterval

Widget 通过 `useWidgetAPI`（`src/utils/proxy/use-widget-api.js` L1-L16）统一请求数据，该 Hook 封装了 SWR 并支持传入 `refreshInterval` 实现定时轮询：

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  if (options[0] === "") url = null;
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

`refreshInterval` 由各 Widget 组件自行设定，典型值如下：

| 场景 | Widget | refreshInterval | 说明 |
|------|--------|-----------------|------|
| 实时播放状态 | plex / tautulli / tracearr | 5000ms (5s) | 正在播放时持续轮询 |
| 实时播放状态 | jellyfin / emby | 5000ms (5s) | enableNowPlaying 时；否则 60000ms |
| 系统监控 | glances | 1500ms (1.5s) | 默认值，用户可通过 YAML 覆盖 |
| 容器编排 | kubernetes / longhorn | 1500ms (1.5s) | |
| 下载器 | jdownloader | 30000ms (30s) | |
| 智能家居 | homeassistant | 60000ms (60s) | |
| 日历 | calendar (ical) | 300000ms (5min) | |
| 站点监控 | site-monitor / ping | 30000ms (30s) | |
| 自定义 API | customapi | 10000ms (10s)，最小 1000ms | 用户可通过 YAML 配置 |
| Prometheus | prometheusmetric | 10000ms (10s)，最小 1000ms | 用户可通过 YAML 配置 |
| iframe | iframe | 用户配置，最小 1000ms | 使用 setInterval 自实现，不走 SWR |
| 系统资源 | resources (cpu/memory/disk/network/uptime/cputemp) | `settings.refresh` 配置 | 来自 settings.yaml 全局设置 |

**Glances 的用户可配置刷新**：`src/widgets/glances/metrics/*.jsx` 中，`refreshInterval` 从 widget 配置读取，并用 `Math.max(defaultInterval, refreshInterval)` 确保不低于默认值（通常 1500ms 或 3000ms）。

### 5.3 配置变更检测：Hash 轮询

通过 `/api/hash` 检测配置文件变化，窗口重新获得焦点时触发检查（`src/pages/index.jsx` L100-L131）：

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

useEffect(() => {
  if (windowFocused) mutateHash();
}, [windowFocused, mutateHash]);

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

配合自定义 Hook：`src/utils/hooks/window-focus.js` L1-L26 监听 `window.focus/blur` 事件。

### 5.4 服务端进程内缓存：memory-cache

`memory-cache` 包提供 Node.js 进程内键值对缓存，支持 TTL 过期。项目中有 **3 类独立用途**：

#### 5.4.1 环境变量缓存

`src/utils/config/config.js` L52-L62：缓存 `process.env` 中 `HOMEPAGE_VAR_*` / `HOMEPAGE_FILE_*` 变量的过滤结果，无 TTL（进程生命周期内有效）：

```javascript
function getCachedEnvironmentVars() {
  let cachedVars = cache.get(cacheKey);
  if (!cachedVars) {
    cachedVars = Object.entries(process.env).filter(
      ([key]) => key.includes("HOMEPAGE_VAR_") || key.includes("HOMEPAGE_FILE_"),
    );
    cache.put(cacheKey, cachedVars);
  }
  return cachedVars;
}
```

#### 5.4.2 HTTP 代理通用响应缓存

`src/utils/proxy/http.js` L85-L109：`cachedRequest()` 函数对外部 API 响应做短期缓存，TTL 为 `duration * 60 * 1000` 毫秒（`duration` 默认 5 分钟）：

```javascript
export async function cachedRequest(url, duration = 5, ua = "homepage") {
  const cached = cache.get(url);
  if (cached) return cached;
  let [, , data] = await httpProxy(url, { headers: { "User-Agent": ua, Accept: "application/json" } });
  if (Buffer.isBuffer(data)) data = JSON.parse(Buffer.from(data).toString());
  cache.put(url, data, duration * 1000 * 60);
  return data;
}
```

#### 5.4.3 Widget 代理 Token/Session 缓存

**这是服务端缓存的主线**：约 15 个 Widget 的代理处理器（proxy.js）使用 `memory-cache` 缓存认证令牌/会话，避免每次 API 请求都重新登录。

**统一缓存模式**：

```
1. cache.get(key) → 命中则直接使用
2. 未命中 → 调用外部 login API → cache.put(key, token/session, TTL)
3. 后续请求携带缓存凭证 → 若返回 401/403 → cache.del(key) → 重新登录
```

**缓存键命名规范**：`{proxyName}__{tokenType}.{service}`，如 `omadaProxyHandler__session.g.svc.0`

**各 Widget 的 Token/Session 缓存详情**：

| Widget | 缓存键 | 缓存内容 | TTL | 失效策略 |
|--------|--------|---------|-----|---------|
| omada | `__session.group.svc.idx` | `{ token, cookieHeader }` | 55 分钟 | 401/403/`errorCode>0` 时 `cache.del()` 重登录 |
| npm | `__token.svc` | JWT token | `expiration - 5分钟` | 403 时 `cache.del()` 重登录 |
| pihole | `__sessionSID.svc` | SID 字符串 | `validity * 1000`（服务端返回的有效期） | 登录失败时 `cache.del()` |
| kavita | `__sessionToken.svc` | access token | 无 TTL（进程内永久） | 无 token 时触发登录 |
| homebridge | `__sessionToken.svc` | access token | `expiresIn * 1000 - 5分钟` | 无 token 时触发登录 |
| homebox | `__sessionToken.svc` | token 字符串 | `expiresAtDate - Date.now()` | 无 token 时触发登录 |
| freshrss | `__sessionToken.svc` | auth token | 无 TTL | 无 token 时触发登录 |
| dispatcharr | `__token.svc` | access token | 无 TTL | 401/认证失败时 `cache.del()` |
| crowdsec | `__sessionToken.svc` | JWT token | 服务端返回的 `ttl` 毫秒 | 401 时 `cache.del()` |
| plex | `__libraries/libraries` | 媒体库列表 | 6 小时 | — |
| plex | `__albums/movies/tv` | 专辑/电影/剧集 | 10 分钟 | — |
| qnap | `__sessionToken.svc` | SID token | 无 TTL | 无 token 时触发登录 |
| transmission | `__headers.svc` | CSRF token (X-Transmission-Session-Id) | 无 TTL | 409 响应时更新 |
| pyload | `__sessionId.svc` | session cookie | 23 小时 | 认证失败时 `cache.del()` |
| pyload | `__isNg.svc` | 是否 ng 版本 | 无 TTL | — |
| beszel | `__token.svc` | JWT token | `expiration - 5分钟` | 认证失败时 `cache.del()` |
| booklore | `__token.svc` | JWT token | `expiration - 5分钟` | 认证失败时 `cache.del()` |
| synology | （handler 内部管理） | SID | 服务端返回的 TTL | 401/404 时重新登录 |
| unifi | （handler 内部管理） | CSRF token + cookie | 900 秒 | — |

### 5.5 Cookie Jar：跨请求 Cookie 持久化

`src/utils/proxy/cookie-jar.js` L1-L40 使用 `tough-cookie` 库维护一个全局 `CookieJar` 实例，在服务端代理请求中自动管理 Cookie：

- **`setCookieHeader(url, params)`**：请求前从 Jar 中提取匹配 URL 的 Cookie 写入 `headers.Cookie`
- **`addCookieToJar(url, headers)`**：响应后将 `Set-Cookie` 头存入 Jar，设置 `maxAge: 3600s`
- **重定向处理**：`http.js` 中 `beforeRedirect` 钩子在每次重定向时同步更新 Cookie

这个 Cookie Jar 与 `memory-cache` 的 Token 缓存互补：Cookie Jar 管理 HTTP 层面的 Cookie 传递，Token 缓存管理应用层面的认证令牌。

### 5.6 HTTP Agent 连接复用

`src/utils/proxy/http.js` L228-L250 使用 `Map` 缓存 HTTP Agent 实例，启用 `keepAlive: true`，避免每次代理请求都建立新 TCP 连接：

```javascript
const agentCache = new Map();

function getAgent(protocol, disableIpv6) {
  const cacheKey = `${protocol}:${disableIpv6 ? "ipv4" : "auto"}`;
  const cachedAgent = agentCache.get(cacheKey);
  if (cachedAgent) return cachedAgent;
  const agent = protocol === "https:"
    ? new https.Agent({ keepAlive: true, ...agentOptions, rejectUnauthorized: false })
    : new http.Agent({ keepAlive: true, ...agentOptions });
  agentCache.set(cacheKey, agent);
  return agent;
}
```

支持 `HOMEPAGE_PROXY_DISABLE_IPV6=true` 环境变量禁用 IPv6。同时包含 DNS 回退机制：`dns.lookup` 失败时自动尝试 `dns.resolve`（c-ares），解决 Alpine/musl 环境 DNS 解析问题。

### 5.7 代理请求链路总览

```
浏览器 Widget 组件
    │
    ▼
useWidgetAPI(widget, endpoint, { refreshInterval })  [客户端]
    │  → useSWR("/api/services/proxy?group=&service=&endpoint=", config)
    │     → SWR 缓存 + refreshInterval 定时轮询
    ▼
/api/services/proxy  [服务端 API Route]
    │  → getServiceWidget() 获取 widget 配置
    │  → 查找 widgets[type].proxyHandler
    ▼
┌────────────────────────────────────────────────────────────────┐
│ ProxyHandler (3 类):                                           │
│                                                                │
│ ① genericProxyHandler                                         │
│    → 直接 httpProxy(url, { headers })                          │
│    → 无额外缓存                                                │
│                                                                │
│ ② credentialedProxyHandler                                    │
│    → 根据 widget.type 注入不同认证头 (Bearer/Basic/X-API-Key)  │
│    → httpProxy(url, { headers, withCredentials: true })        │
│    → 无额外缓存，凭证从 YAML 配置直接读取                      │
│                                                                │
│ ③ 自定义 ProxyHandler (如 omada/npm/pihole/plex 等)            │
│    → cache.get(tokenKey) → 命中则使用缓存 token/session        │
│    → 未命中 → login() → cache.put(tokenKey, token, TTL)        │
│    → httpProxy(url, { headers: { Authorization: token } })     │
│    → 401/403 → cache.del(tokenKey) → 重新 login()             │
│    → 部分还使用 cachedRequest() 做响应级短期缓存               │
└────────────────────────────────────────────────────────────────┘
    │
    ▼
httpProxy(url, params)  [底层 HTTP 客户端]
    │  → getAgent() 复用 keep-alive Agent
    │  → addCookieHandler() 自动携带/存储 Cookie
    │  → DNS 回退机制
    ▼
外部服务 API
```

### 5.8 组件懒加载：next/dynamic

**Toggle 组件 SSR 禁用**（因为需要 localStorage，服务端无此 API）：
`src/pages/index.jsx` L30-L40

```javascript
const ThemeToggle = dynamic(() => import("components/toggles/theme"), { ssr: false });
const ColorToggle = dynamic(() => import("components/toggles/color"), { ssr: false });
const Version     = dynamic(() => import("components/version"),          { ssr: false });
```

**Widget 组件全量动态导入**：`src/widgets/components.js` L1-L167

将约 150+ 个 widget 组件全部通过 `next/dynamic` 懒加载：

```javascript
const components = {
  adguard: dynamic(() => import("./adguard/component")),
  plex:    dynamic(() => import("./plex/component")),
  // ... 约 150 个
};
```

### 5.9 静态资源版本化（Cache Busting）

`public/` 下的静态资源通过 query 参数版本号绕过浏览器缓存：

```
/apple-touch-icon.png?v=4
/favicon-32x32.png?v=4
/safari-pinned-tab.svg?v=4
/site.webmanifest?v=4
/android-chrome-192x192.png?v=2
/mstile-150x150.png?v=2
```

### 5.10 配置文件自动初始化

首次启动时自动从骨架模板复制到 config 目录（`src/utils/config/config.js` L15-L50）：

```javascript
export default function checkAndCopyConfig(config) {
  if (!existsSync(CONF_DIR)) mkdirSync(CONF_DIR, { recursive: true });
  const configYaml = join(CONF_DIR, config);
  if (!existsSync(configYaml)) {
    const configSkeleton = join(process.cwd(), "src", "skeleton", config);
    copyFileSync(configSkeleton, configYaml);
  }
  yaml.load(readFileSync(configYaml, "utf8"));
}
```

配置目录可通过环境变量 `HOMEPAGE_CONFIG_DIR` 覆盖，默认 `<cwd>/config`。

---

## 六、关键数据流

### 6.1 图标渲染数据流

```
YAML 配置 / Docker K8s 自动发现
        │
        ▼
  servicesResponse() / bookmarksResponse() / widgetsResponse()
  [src/utils/config/api-response.js]
        │
        ▼
  getStaticProps() → SWR fallback (SSR 构建时预填)
        │
        ▼
  useSWR("/api/services") / useSWR("/api/bookmarks") / useSWR("/api/widgets")
        │
        ├─► src/components/services/item.jsx  ──► <ResolvedIcon icon={service.icon} />
        ├─► src/components/services/group.jsx ──► <ResolvedIcon icon={layout.icon} />
        ├─► src/components/bookmarks/item.jsx ──► <ResolvedIcon icon={bookmark.icon} />
        ├─► src/components/bookmarks/group.jsx──► <ResolvedIcon icon={layout.icon} />
        ├─► src/components/quicklaunch.jsx    ──► <ResolvedIcon icon={r.icon} />
        ├─► src/components/widgets/logo/logo.jsx ──► <ResolvedIcon icon={options.icon} 48x48 />
        ├─► src/widgets/glances/metrics/process.jsx ──► 7 种状态 mdi-* 图标
        └─► src/widgets/glances/metrics/containers.jsx ──► 4 种状态 mdi-* 图标
               │
               ▼
        ResolvedIcon 解析逻辑 (src/components/resolvedicon.jsx)
               │
               ├── http / 开头 ──► <Image src="直接URL" />
               ├── sh- 前缀 ──► jsDelivr + selfhst/icons ──► <Image />
               ├── mdi-/si- 前缀 ──► CSS mask + 主题色
               │     ├── 尾部 -#RRGGBB ──► 自定义hex色
               │     ├── iconStyle=theme ──► --color-300(dark) / --color-900(light)
               │     └── 默认 ──► logo 渐变
               └── 其他 ──► dashboard-icons CDN ──► <Image />
```

### 6.2 主题切换数据流

```
┌─────────────────────────────────────────────────────────────────┐
│ SSR 构建阶段:                                                    │
│   getStaticProps() → getSettings() → initialSettings.color/theme│
│   (直接读 config/settings.yaml，不经过任何 API)                  │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Provider 初始化 (_app.jsx):                                      │
│   ColorProvider  → localStorage["theme-color"] 或 默认 "slate"  │
│   ThemeProvider  → localStorage["theme-mode"]/系统偏好/默认 "dark"│
│   SettingsProvider → 用 initialSettings 初始化 state            │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Wrapper 挂载 (index.jsx L517):                                   │
│   useEffect → 同步 <html> 类名: .dark/.light + .theme-{color}   │
│   (用 Context 值 + initialSettings.color 兜底)                   │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Home 组件同步 (index.jsx L234):                                  │
│   useEffect: settings.theme/color → setTheme()/setColor()       │
│   → Provider 内部 rawSet 函数再次同步 <html> 类名 + localStorage │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 用户交互切换 (仅当 settings 未预设 theme/color 时):               │
│   ColorToggle / ThemeToggle → setColor/setTheme                 │
│   → localStorage + <html> 类名同步                               │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
               theme.css / globals.css 中 CSS 变量生效
                             │
                             ▼
         Tailwind 原子类 bg-theme-700 / text-theme-200 / dark:...
         resolvedicon.jsx 中 mask 图标的 iconColor
         Logo Widget 降级 SVG 的 rgba(var(--color-logo-start/stop))
         ⚠️ favicon.jsx 未接入页面，不参与主题联动
         ⚠️ /api/theme 未被任何页面调用，仅测试文件使用
```

---

## 七、核心文件索引

| 文件 | 职责 |
|------|------|
| `next.config.js` | Next.js 构建配置、远程图片白名单 |
| `src/pages/_app.jsx` | 全局 Provider 嵌套、viewport meta、SWR 全局配置、样式入口 |
| `src/pages/_document.jsx` | SSR 文档模板、manifest（含 crossOrigin）、custom.css 预加载 |
| `src/pages/index.jsx` | 首页主逻辑、Wrapper 层类名同步、getStaticProps、Head 资源注入 |
| `src/pages/site.webmanifest.jsx` | PWA manifest 动态生成（含 theme_color） |
| `src/pages/browserconfig.xml.jsx` | Windows 磁贴配置动态生成（含 TileColor） |
| `src/pages/api/theme.js` | 主题配置 API（未接入页面，仅测试文件使用） |
| `src/pages/api/config/[path].js` | custom.css/custom.js 动态服务 |
| `src/components/resolvedicon.jsx` | 图标解析与渲染核心组件 |
| `src/components/favicon.jsx` | 动态主题色 Favicon 生成（未接入页面） |
| `src/components/services/item.jsx` | 服务卡片（ResolvedIcon 入口 1/8） |
| `src/components/services/group.jsx` | 服务分组标题（ResolvedIcon 入口 2/8） |
| `src/components/bookmarks/item.jsx` | 书签卡片（ResolvedIcon 入口 3/8） |
| `src/components/bookmarks/group.jsx` | 书签分组标题（ResolvedIcon 入口 4/8） |
| `src/components/quicklaunch.jsx` | 快速启动搜索（ResolvedIcon 入口 5/8） |
| `src/components/widgets/logo/logo.jsx` | Logo 信息 Widget（ResolvedIcon 入口 6/8，48×48） |
| `src/widgets/glances/metrics/process.jsx` | Glances 进程状态图标（ResolvedIcon 入口 7/8） |
| `src/widgets/glances/metrics/containers.jsx` | Glances 容器状态图标（ResolvedIcon 入口 8/8） |
| `src/utils/contexts/theme.jsx` | 明暗模式 Context |
| `src/utils/contexts/color.jsx` | 主题色系 Context |
| `src/utils/contexts/settings.jsx` | 用户设置 Context |
| `src/utils/styles/themes.js` | 23 套主题色板定义（JS 对象） |
| `src/styles/theme.css` | 23 套主题 CSS 变量注入 |
| `src/styles/globals.css` | Tailwind 入口 + 全局基础样式 |
| `src/styles/manrope.css` | Manrope 字体 @font-face 定义 |
| `tailwind.config.js` | Tailwind 自定义主题映射 |
| `src/utils/config/config.js` | 配置文件加载、环境变量替换、骨架复制 |
| `src/utils/config/api-response.js` | 服务/书签/Widget 数据聚合响应 |
| `src/utils/hooks/window-focus.js` | 窗口焦点 Hook（配置刷新触发） |
| `src/utils/proxy/use-widget-api.js` | Widget API 请求 Hook（封装 SWR + refreshInterval） |
| `src/utils/proxy/api-helpers.js` | 代理 URL 格式化、参数解析、错误 URL 脱敏 |
| `src/utils/proxy/http.js` | 底层 HTTP 客户端（keep-alive Agent、cachedRequest、Cookie 管理、DNS 回退） |
| `src/utils/proxy/cookie-jar.js` | 跨请求 Cookie 持久化（tough-cookie 全局 Jar） |
| `src/utils/proxy/handlers/generic.js` | 通用代理处理器（Basic Auth + 直接转发） |
| `src/utils/proxy/handlers/credentialed.js` | 凭证代理处理器（按 widget.type 注入不同认证头） |
| `src/pages/api/services/proxy.js` | 服务代理 API Route（映射 endpoint → proxyHandler） |
| `src/widgets/components.js` | 150+ Widget 组件 dynamic import 映射 |
