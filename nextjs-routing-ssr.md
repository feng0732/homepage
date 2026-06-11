# Next.js 路由与 SSR 深度理解

基于 [homepage](https://github.com/gethomepage/homepage) 项目（Next.js 16 + Pages Router）的源码分析。

---

## 一、路由系统解析机制

### 1.1 Pages Router 文件系统路由

Next.js Pages Router 基于 `src/pages/` 目录结构自动生成路由，无需手动配置路由表。

**项目路由映射表：**

| 文件路径 | 路由地址 | 说明 |
|---------|---------|------|
| `src/pages/index.jsx` | `/` | 首页，主应用入口 |
| `src/pages/_app.jsx` | - | 全局应用包装器，不对应具体路由 |
| `src/pages/_document.jsx` | - | 自定义 HTML 文档结构 |
| `src/pages/api/*` | `/api/*` | API 路由 |
| `src/pages/api/config/[path].js` | `/api/config/:path` | 动态 API 路由 |
| `src/pages/robots.txt.js` | `/robots.txt` | 特殊文件路由 |
| `src/pages/browserconfig.xml.jsx` | `/browserconfig.xml` | 特殊文件路由 |
| `src/pages/site.webmanifest.jsx` | `/site.webmanifest` | PWA 清单路由 |

[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next.config.js#L1-L19) 中的路由相关配置：

```javascript
const { i18n } = require("./next-i18next.config");

const nextConfig = {
  reactStrictMode: true,
  output: "standalone",
  i18n,
};
```

[next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next-i18next.config.js#L112-L117) 中的 i18n 路由配置：

```javascript
module.exports = {
  i18n: {
    defaultLocale: "en",
    locales: ["en"],
  },
};
```

### 1.2 中间件（Middleware）与首页请求的真实关联

[src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/middleware.js#L1-L23) 的完整代码：

```javascript
import { NextResponse } from "next/server";

export function middleware(req) {
  const host = req.headers.get("host");
  const port = process.env.PORT || 3000;
  let allowedHosts = [`localhost:${port}`, `127.0.0.1:${port}`, `[::1]:${port}`];
  const allowAll = process.env.HOMEPAGE_ALLOWED_HOSTS === "*";
  if (process.env.HOMEPAGE_ALLOWED_HOSTS) {
    allowedHosts = allowedHosts.concat(process.env.HOMEPAGE_ALLOWED_HOSTS.split(","));
  }
  if (!allowAll && (!host || !allowedHosts.includes(host))) {
    return NextResponse.json(
      { error: "Host validation failed. See logs for more details." },
      { status: 400 },
    );
  }
  return NextResponse.next();
}

export const config = {
  matcher: "/api/:path*",
};
```

#### 关键修正：中间件不拦截首页 SSR 请求

这是一个容易混淆的点。`config.matcher` 的值是 `"/api/:path*"`，这意味着：

**中间件只拦截 `/api/` 开头的请求，不拦截页面请求。**

具体来说：

| 请求路径 | 是否经过中间件 | 原因 |
|----------|:---:|------|
| `GET /`（首页 SSR） | ❌ | 不匹配 `/api/:path*` |
| `GET /api/services` | ✅ | 匹配 `/api/:path*` |
| `GET /api/config/custom.css` | ✅ | 匹配 `/api/:path*` |
| `GET /api/revalidate` | ✅ | 匹配 `/api/:path*` |
| `GET /api/hash` | ✅ | 匹配 `/api/:path*` |

**所以首页的 SSR 渲染流程实际上是这样的：**

```
用户请求 GET /
    ↓
不经过 middleware（matcher 不匹配）
    ↓
直接进入路由匹配 → src/pages/index.jsx
    ↓
执行 getStaticProps() 获取数据
    ↓
渲染 HTML 返回
```

**中间件的真实角色是 API 网关防护，而非全局请求拦截。** 它保护的是 API 路由（如 `/api/services`、`/api/config/custom.css`），防止通过伪造 Host 头访问这些接口。这是因为项目作为一个仪表盘应用，其 API 端点暴露了 Docker/Kubernetes/Proxmox 等基础设施信息，需要做 Host 白名单校验。

**为什么首页不需要中间件保护？** 因为首页请求 `/` 是一个页面请求，走的是 Next.js 的页面渲染管道（`getStaticProps` → React SSR → HTML），而不是直接暴露 API 数据。即使恶意请求访问 `/`，返回的只是 HTML 页面，不会泄露敏感信息。

**间接关联：** 首页的客户端代码中，`useSWR("/api/services")`、`useSWR("/api/bookmarks")` 等调用会向 `/api/*` 端点发请求，这些请求**会**经过中间件的 Host 校验。如果 Host 不合法，SWR 请求会被中间件拦截返回 400，导致客户端无法获取数据。

### 1.3 动态路由解析

[src/pages/api/config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/api/config/[path].js#L1-L34) 展示了动态路由参数的获取：

```javascript
export default async function handler(req, res) {
  const { path: relativePath } = req.query;
  if (!["custom.css", "custom.js"].includes(relativePath)) {
    return res.status(422).end("Unsupported file");
  }
  // 读取配置文件...
}
```

注意：这个动态路由也受中间件保护，因为 `/api/config/custom.css` 匹配 `/api/:path*`。

---

## 二、服务端取数（getStaticProps）

项目使用 **ISR（Incremental Static Regeneration，增量静态再生）** 模式，通过 `getStaticProps` 在构建时和运行时获取数据。

### 2.1 getStaticProps 执行流程

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L55-L95) 中的核心取数逻辑：

```javascript
export async function getStaticProps() {
  let logger;
  try {
    logger = createLogger("index");
    const { providers, ...settings } = getSettings();

    const services = await servicesResponse();
    const bookmarks = await bookmarksResponse();
    const widgets = await widgetsResponse();
    const language = normalizeLanguage(settings.language);

    return {
      props: {
        initialSettings: settings,
        fallback: {
          "/api/services": services,
          "/api/bookmarks": bookmarks,
          "/api/widgets": widgets,
          "/api/hash": false,
        },
        ...(await serverSideTranslations(language)),
      },
    };
  } catch (e) {
    return {
      props: {
        initialSettings: {},
        fallback: {
          "/api/services": [],
          "/api/bookmarks": [],
          "/api/widgets": [],
          "/api/hash": false,
        },
        ...(await serverSideTranslations("en")),
      },
    };
  }
}
```

**关键特性：**
- 仅在 **Node.js 环境** 执行，不会打包到客户端
- 构建时执行一次，之后可通过 `res.revalidate()` 触发重新生成
- 返回的 `props` 会序列化为 JSON 嵌入 HTML

### 2.2 配置文件读取链

[src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/config/config.js#L82-L103) 中的 `getSettings` 函数：

```javascript
export function getSettings() {
  checkAndCopyConfig("settings.yaml");
  const settingsYaml = join(CONF_DIR, "settings.yaml");
  const rawFileContents = readFileSync(settingsYaml, "utf8");
  const fileContents = substituteEnvironmentVars(rawFileContents);
  const initialSettings = yaml.load(fileContents) ?? {};
  return initialSettings;
}
```

### 2.3 ISR 增量重新验证

[src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/api/revalidate.js#L1-L8)：

```javascript
export default async function handler(req, res) {
  try {
    await res.revalidate("/");
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}
```

**触发时机：** 客户端检测到配置哈希变化时，调用 `/api/revalidate` 触发重新生成（见 [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L119-L127)）。注意这个 API 请求也会经过中间件的 Host 校验。

---

## 三、客户端导航机制——本项目的特殊之处

### 3.1 关键修正：本项目不使用 next/link

在整个 `src/` 目录中搜索 `next/link` 的导入和 `<Link` 标签，结果为 **零匹配**：

```
# 搜索 from "next/link" → 无结果
# 搜索 <Link → 无结果
```

项目唯一使用的 Next.js 路由 API 是 [useRouter](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L14)，且只用于获取 `asPath`（当前 URL 路径）：

```javascript
import { useRouter } from "next/router";
// ...
const { asPath } = useRouter();
```

这个 `asPath` 用于 [第 291-294 行](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L290-L295) 的标签页初始化：

```javascript
useEffect(() => {
  if (!activeTab) {
    const initialTab = asPath.substring(asPath.indexOf("#") + 1);
    setActiveTab(initialTab === "/" ? slugifyAndEncode(tabs["0"]) : initialTab);
  }
});
```

### 3.2 为什么不需要 next/link？

这是由项目的应用特性决定的：

1. **单页应用架构**：Homepage 只有一个页面路由 `/`，不存在页面间导航的需求
2. **标签页切换基于 hash**：用户在不同"页面"间切换是通过 URL hash（如 `/#docker`、`/#media`）实现的，这是纯客户端行为，不触发路由变化
3. **外部链接使用原生 `<a>` 标签**：书签和服务项指向外部 URL，用原生 `<a href="...">` 即可，不需要 Next.js 的客户端路由

因此，本项目 **不存在传统意义上的客户端导航（SPA 路由切换）**。所有的"导航"本质上只有两种：
- **标签页切换**：修改 URL hash，纯客户端行为，不触发 Next.js 路由
- **外部跳转**：原生 `<a>` 标签，浏览器全量导航

### 3.3 与通用 Next.js 项目的对比

| 场景 | 通用 Next.js 项目 | 本项目 |
|------|-------------------|--------|
| 页面间导航 | `next/link` + 客户端路由 | 不适用（只有一个页面路由） |
| 标签/视图切换 | 可能用路由参数 | URL hash + React state |
| 外部链接 | `<a>` 或 `next/link` + `prefetch={false}` | 原生 `<a>` |
| 路由信息获取 | `useRouter()` 获取 params/query | `useRouter()` 仅获取 `asPath` |
| ISR 重新验证 | 可能用 `router.replace(router.asPath)` | `window.location.reload()` 全量刷新 |

---

## 四、服务端数据如何接入状态上下文——完整链路追踪

这是理解本项目中 SSR 与客户端状态衔接最关键的部分。数据从 `getStaticProps` 到 React Context 的传递并不像初看那么简单，存在一个容易被忽视的**断层**。

### 4.1 完整数据流

```
getStaticProps() 返回 props
    ↓
{ initialSettings: {...}, fallback: {...} }
    ↓
Next.js 序列化为 pageProps，嵌入 __NEXT_DATA__
    ↓
_app.jsx 接收 pageProps，展开为 <Component {...pageProps} />
    ↓
index.jsx 的 Wrapper 组件接收 { initialSettings, fallback }
    ↓
Wrapper → Index → Home → useContext(SettingsContext)
```

### 4.2 关键断层：_app.jsx 中的 Provider 不接收 initialSettings

仔细看 [_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_app.jsx#L73-L98)：

```javascript
function MyApp({ Component, pageProps }) {
  return (
    <SWRConfig value={{ fetcher: ... }}>
      <ColorProvider>           {/* ← 没有传 initialTheme */}
        <ThemeProvider>         {/* ← 没有传 initialTheme */}
          <SettingsProvider>    {/* ← 没有传 initialSettings */}
            <TabProvider>       {/* ← 没有传 initialTab */}
              <Component {...pageProps} />
            </TabProvider>
          </SettingsProvider>
        </ThemeProvider>
      </ColorProvider>
    </SWRConfig>
  );
}
```

**所有 Provider 都没有从 `pageProps` 接收初始值！** 这意味着：

- `SettingsProvider` 的 `initialSettings` prop 是 `undefined`
- `ThemeProvider` 的 `initialTheme` prop 是 `undefined`
- `ColorProvider` 的 `initialTheme` prop 是 `undefined`
- `TabProvider` 的 `initialTab` prop 是 `undefined`

### 4.3 各 Provider 在无初始值时的行为

#### SettingsProvider

[src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/settings.jsx#L5-L14)：

```javascript
export function SettingsProvider({ initialSettings, children }) {
  const [settings, setSettings] = useState(() => initialSettings ?? {});
  // initialSettings 是 undefined，所以 settings 初始值为 {}

  useEffect(() => {
    if (initialSettings !== undefined) setSettings(initialSettings ?? {});
    // initialSettings 是 undefined，所以这个 effect 不会执行
  }, [initialSettings]);

  // ...
}
```

**结果：** 在 `_app.jsx` 层级，`SettingsContext` 的初始值是空对象 `{}`，不是服务端获取的配置。

#### ThemeProvider

[src/utils/contexts/theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/theme.jsx#L1-L46)：

```javascript
const getInitialTheme = () => {
  if (typeof window !== "undefined" && window.localStorage) {
    const storedPrefs = window.localStorage.getItem("theme-mode");
    if (typeof storedPrefs === "string") return storedPrefs;
    const userMedia = window.matchMedia("(prefers-color-scheme: dark)");
    if (userMedia.matches) return "dark";
  }
  return "dark";
};

export function ThemeProvider({ initialTheme, children }) {
  const [theme, setTheme] = useState(() => initialTheme ?? getInitialTheme());
  // initialTheme 是 undefined，所以使用 getInitialTheme()
  // 服务端：typeof window === "undefined" → 返回 "dark"
  // 客户端：检查 localStorage 或系统偏好 → 可能返回 "light" 或 "dark"
}
```

**结果：** 主题初始值不来自服务端配置，而是来自客户端的 `localStorage` 或系统偏好。

#### ColorProvider

[src/utils/contexts/color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/color.jsx#L5-L44)：

```javascript
const getInitialColor = () => {
  if (typeof window !== "undefined" && window.localStorage) {
    const storedPrefs = window.localStorage.getItem("theme-color");
    if (typeof storedPrefs === "string") return storedPrefs;
  }
  return "slate";
};

export function ColorProvider({ initialTheme, children }) {
  const [color, setColor] = useState(() => initialTheme ?? getInitialColor());
  // initialTheme 是 undefined，所以使用 getInitialColor()
  // 服务端：typeof window === "undefined" → 返回 "slate"
  // 客户端：检查 localStorage → 可能返回用户自定义颜色
}
```

### 4.4 数据如何真正接入上下文——页面组件中的手动注入

服务端数据并非通过 Provider 的 props 传入，而是在页面组件内部通过 **`useEffect` + `setSettings()`** 手动注入到 Context 中。

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L214-L224) 中的 `Home` 组件：

```javascript
function Home({ initialSettings }) {
  const { settings, setSettings } = useContext(SettingsContext);
  // ...

  useEffect(() => {
    setSettings(initialSettings);  // 手动将 props 注入 Context
  }, [initialSettings, setSettings]);
```

同样，主题和颜色也是手动注入的（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L234-L247)）：

```javascript
useEffect(() => {
  const language = normalizeLanguage(settings.language);
  if (language) {
    i18n.changeLanguage(language);
  }

  if (settings.theme && theme !== settings.theme) {
    setTheme(settings.theme);  // 从 settings 中取 theme，手动注入 ThemeContext
  }

  if (settings.color && color !== settings.color) {
    setColor(settings.color);  // 从 settings 中取 color，手动注入 ColorContext
  }
}, [i18n, settings, color, setColor, theme, setTheme]);
```

### 4.5 完整数据流时序图

```
[服务端] getStaticProps() 返回 { initialSettings, fallback }
    ↓
[服务端] _app.jsx 渲染：
    ↓  SettingsProvider(initialSettings=undefined) → settings = {}
    ↓  ThemeProvider(initialTheme=undefined) → theme = "dark"（默认值）
    ↓  ColorProvider(initialTheme=undefined) → color = "slate"（默认值）
    ↓
[服务端] Wrapper 组件渲染：
    ↓  使用 initialSettings.background（直接从 props）
    ↓  Index 组件渲染（SWRConfig 包裹 fallback 数据）
    ↓  Home 组件渲染（此时 settings 还是 {}，初始渲染用 props）
    ↓
[服务端] HTML 生成，包含初始渲染结果
    ↓
[客户端] Hydration 开始
    ↓
[客户端] React 恢复组件树，SettingsContext.settings = {}
    ↓  ThemeProvider 检查 localStorage → 可能更新 theme
    ↓  ColorProvider 检查 localStorage → 可能更新 color
    ↓
[客户端] useEffect 执行（首次挂载后）
    ↓  Home 中的 useEffect: setSettings(initialSettings)
    ↓  → SettingsContext.settings 更新为完整配置
    ↓
[客户端] settings 变化触发第二个 useEffect
    ↓  setTheme(settings.theme) → ThemeContext 更新
    ↓  setColor(settings.color) → ColorContext 更新
    ↓
[客户端] 所有 Context 就绪，页面完全交互
```

### 4.6 这种设计的影响——短暂的不一致窗口

由于服务端数据不是在 Provider 初始化时就传入，而是通过 `useEffect` 延迟注入，存在一个短暂的**不一致窗口**：

1. **SSR 阶段**：`settings = {}`，所以服务端渲染时 `settings.theme`、`settings.color` 都是 `undefined`
2. **Hydration 后、useEffect 执行前**：`settings` 仍然是 `{}`
3. **useEffect 执行后**：`settings` 才被更新为服务端获取的完整配置

这意味着如果配置中指定了 `theme: "light"`，服务端渲染的 HTML 仍然使用 `theme = "dark"`（默认值），客户端 Hydration 后也暂时是 `"dark"`，直到 `useEffect` 触发才切换为 `"light"`。

这可能导致 **FOUC（Flash of Unstyled Content）**，即用户短暂看到暗色主题后突然切换到亮色主题。

[Wrapper 组件](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L517-L593) 中的主题相关代码也体现了这一点：

```javascript
useEffect(() => {
  const html = document.documentElement;
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");
  // ... 这段代码在 useEffect 中，意味着首次渲染后才会修正主题
}, [backgroundImage, opacity, theme, color, initialSettings.color]);
```

### 4.7 另一种可行方案：Provider 直接接收 pageProps

如果想让服务端数据在 Provider 初始化时就生效，可以修改 `_app.jsx`：

```javascript
// 假设这样修改（本项目实际没有这样做）
function MyApp({ Component, pageProps }) {
  return (
    <SettingsProvider initialSettings={pageProps.initialSettings}>
      <ThemeProvider initialTheme={pageProps.initialSettings?.theme}>
        <ColorProvider initialTheme={pageProps.initialSettings?.color}>
          <TabProvider>
            <Component {...pageProps} />
          </TabProvider>
        </ColorProvider>
      </ThemeProvider>
    </SettingsProvider>
  );
}
```

这样做的好处是 Provider 初始化时就拥有正确的值，服务端渲染和客户端 Hydration 的结果一致，避免 FOUC。但本项目选择了在页面组件中通过 `useEffect` 手动注入，可能是因为：

- `SettingsProvider` 定义在 `_app.jsx` 中，而 `initialSettings` 是页面级数据，不同页面可能有不同的 props 结构
- 保持 Provider 的通用性，不与特定页面的数据结构耦合
- 配置变更后，客户端可以独立于 SSR 更新 Context（通过 `setSettings()`）

### 4.8 background 数据的特殊路径

值得注意的是，并非所有服务端数据都走 Context 中转。[Wrapper 组件](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L517-L538) 中的 `background` 配置就是**直接从 props 读取**，不经过 Context：

```javascript
export default function Wrapper({ initialSettings, fallback }) {
  let backgroundImage = "";
  let opacity = initialSettings?.backgroundOpacity ?? 0;
  // ...

  if (initialSettings?.background) {
    const bg = initialSettings.background;
    if (typeof bg === "object") {
      backgroundImage = bg.image || "";
      // ...
    }
  }
  // backgroundImage 直接从 initialSettings 读取，不经 Context
}
```

这形成了一个**双轨数据流**：

| 数据 | 传递方式 | 是否经过 Context |
|------|---------|:---:|
| background 配置 | props 直接传递 | ❌ |
| title、description | props 直接传递（`<Head>` 中） | ❌ |
| 主题、颜色 | props → useEffect → Context | ✅ |
| 服务/书签/小部件 | props → SWR fallback → useSWR | ❌（走 SWR 缓存） |
| 语言 | props → useEffect → i18n | ❌（走 i18next） |

---

## 五、页面拼装与 HTML 生成过程

### 5.1 渲染层级结构

```
请求 GET / 到达
    ↓
（不经过 middleware，因为 matcher 只匹配 /api/:path*）
    ↓
路由匹配到 src/pages/index.jsx
    ↓
getStaticProps() 服务端取数 → 返回 pageProps
    ↓
_document.jsx（生成 HTML 骨架）
    ├─ <Html>
    ├─ <Head>（全局 meta、manifest、custom.css）
    └─ <body>
        ├─ <Main />（注入 _app + 页面内容）
        └─ <NextScript />（注入 Next.js runtime 脚本）
            ↓
_app.jsx（全局包装）
    ├─ SWRConfig（全局 fetcher 配置）
    ├─ ColorProvider（初始值：undefined → 降级为 "slate"）
    ├─ ThemeProvider（初始值：undefined → 降级为 "dark"）
    ├─ SettingsProvider（初始值：undefined → 降级为 {}）
    ├─ TabProvider（初始值：undefined → 降级为 false）
    └─ <Component {...pageProps} />（实际页面组件）
        ↓
index.jsx 的 Wrapper → Index → Home（业务内容）
```

### 5.2 _document.jsx - HTML 骨架

[src/pages/_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_document.jsx#L1-L18)：

```javascript
import { Head, Html, Main, NextScript } from "next/document";

export default function Document() {
  return (
    <Html>
      <Head>
        <meta name="mobile-web-app-capable" content="yes" />
        <link rel="manifest" href="/site.webmanifest?v=4" crossOrigin="use-credentials" />
        <link rel="preload" href="/api/config/custom.css" as="style" />
        <link rel="stylesheet" href="/api/config/custom.css" />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  );
}
```

**关键点：**
- 仅在服务端渲染执行，客户端不执行
- `Main` 和 `NextScript` 是必须的占位符
- 注意 `<link rel="preload" href="/api/config/custom.css">`——这是一个 API 路由，由 [src/pages/api/config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/api/config/[path].js#L1-L34) 提供，会经过中间件 Host 校验

### 5.3 _app.jsx - 全局应用包装

[src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_app.jsx#L73-L98)：

```javascript
function MyApp({ Component, pageProps }) {
  return (
    <SWRConfig value={{ fetcher: (resource, init) => fetch(resource, init).then(res => res.json()) }}>
      <Head>
        <meta name="viewport" content="width=device-width, initial-scale=1.0, ..." />
      </Head>
      <ColorProvider>
        <ThemeProvider>
          <SettingsProvider>
            <TabProvider>
              <Component {...pageProps} />
            </TabProvider>
          </SettingsProvider>
        </ThemeProvider>
      </ColorProvider>
    </SWRConfig>
  );
}

export default appWithTranslation(MyApp, nextI18nextConfig);
```

**关键点：**
- `appWithTranslation` HOC 来自 `next-i18next`，为 `_app.jsx` 添加 i18n 能力
- `pageProps` 被展开传入 `<Component />`，**但没有被拆分传入各 Provider**
- SWRConfig 在最外层提供全局 fetcher，但页面级的 `SWRConfig`（在 `Index` 组件中）会覆盖这里的配置

---

## 六、客户端接续渲染（Hydration）

### 6.1 Hydration 执行流程

```
浏览器加载 HTML
    ↓
1. 解析 HTML，显示静态内容（用户已可看到页面）
    ↓
2. 下载并执行 Next.js runtime
    ↓
3. 读取 <script id="__NEXT_DATA__"> 中的序列化数据
    ↓
4. React 客户端初始化，开始 Hydration
    ↓
5. 逐节点对比服务端 HTML 和客户端渲染结果
    ↓
6. 绑定事件监听器，页面变为可交互
    ↓
7. useEffect 执行：
   - setSettings(initialSettings) → Context 更新
   - setTheme(settings.theme) → 主题切换（可能触发重渲染）
   - setColor(settings.color) → 颜色切换（可能触发重渲染）
    ↓
8. SWR 从 fallback 恢复缓存，后续 useSWR 使用缓存数据
```

### 6.2 数据无缝接续：SWR fallback 机制

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L185-L191)：

```javascript
<SWRConfig value={{
  fallback,
  fetcher: (resource, init) => fetch(resource, init).then(res => res.json())
}}>
```

当 `Home` 组件中调用 `useSWR("/api/services")` 时，SWR 发现 `fallback` 中已有该 key 的数据，不会立即发送网络请求，而是直接返回缓存值。后续的 revalidate（如窗口获得焦点时）才会触发真正的网络请求。

注意这里的两层 SWRConfig：
- `_app.jsx` 中的全局 SWRConfig（只提供 fetcher）
- `Index` 组件中的页面级 SWRConfig（提供 fallback + fetcher，覆盖全局配置）

### 6.3 SSR 与 CSR 分界：dynamic 组件

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L30-L40)：

```javascript
const ThemeToggle = dynamic(() => import("components/toggles/theme"), { ssr: false });
const ColorToggle = dynamic(() => import("components/toggles/color"), { ssr: false });
const Version = dynamic(() => import("components/version"), { ssr: false });
```

这些组件只在客户端渲染。服务端渲染时，它们的位置为空，Hydration 后才加载并渲染。

### 6.4 客户端配置更新检测

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L104-L131)：

```javascript
// 窗口获得焦点时，重新获取配置哈希
useEffect(() => {
  if (windowFocused) {
    mutateHash();
  }
}, [windowFocused, mutateHash]);

// 哈希变化 → 触发 ISR 重新验证 → 全量刷新
useEffect(() => {
  if (hashData) {
    if (typeof window !== "undefined") {
      const previousHash = localStorage.getItem("hash");
      if (!previousHash) {
        localStorage.setItem("hash", hashData.hash);
      }
      if (previousHash && previousHash !== hashData.hash) {
        setStale(true);
        localStorage.setItem("hash", hashData.hash);
        fetch("/api/revalidate").then((res) => {
          if (res.ok) {
            window.location.reload();
          }
        });
      }
    }
  }
}, [hashData]);
```

注意：`fetch("/api/revalidate")` 请求**会经过中间件**的 Host 校验。如果 Host 不合法，revalidate 请求会失败，配置更新无法生效。

---

## 七、完整请求生命周期总结

### 7.1 首次访问（服务端渲染完整流程）

```
用户请求 GET /
    ↓
[不经过 middleware]（matcher 只匹配 /api/:path*，首页不匹配）
    ↓
路由匹配到 src/pages/index.jsx
    ↓
[服务端取数] getStaticProps()
    ├─ getSettings() 读取 settings.yaml
    ├─ servicesResponse() 聚合 Docker/K8s/YAML 数据
    ├─ bookmarksResponse() 读取书签
    ├─ widgetsResponse() 读取小部件
    └─ serverSideTranslations() 加载翻译
    ↓
返回 pageProps: { initialSettings, fallback, _nextI18Next }
    ↓
[页面渲染]
    ├─ _document.jsx → HTML 骨架 + custom.css
    ├─ _app.jsx → Providers（初始值为 undefined/默认值）
    ├─ Wrapper → 背景图（直接从 props 读取）
    ├─ Index → SWRConfig fallback 数据桥
    └─ Home → 业务内容（settings 此时为 {}）
    ↓
[数据注入] pageProps 序列化为 __NEXT_DATA__
    ↓
HTTP 响应返回完整 HTML
    ↓
[客户端]
    ├─ 浏览器解析 HTML，显示静态内容
    ├─ 下载 Next.js runtime
    ├─ 读取 __NEXT_DATA__ 恢复数据
    ├─ React Hydration，绑定事件
    ├─ useEffect: setSettings(initialSettings) → Context 更新
    ├─ useEffect: setTheme(settings.theme) → 主题修正
    ├─ useEffect: setColor(settings.color) → 颜色修正
    ├─ SWR 从 fallback 恢复缓存
    └─ 页面完全可交互
```

### 7.2 客户端数据请求（需要经过中间件）

```
客户端 useSWR("/api/services") 发起请求
    ↓
[经过 middleware] Host 校验
    ├─ 校验通过 → 正常返回数据
    └─ 校验失败 → 返回 400 → SWR 报错
    ↓
SWR 更新缓存，触发重渲染
```

### 7.3 配置变更检测流程

```
窗口获得焦点
    ↓
mutateHash() → GET /api/hash（经过中间件）
    ↓
比较 localStorage 中的 hash
    ↓
hash 不同 → fetch("/api/revalidate")（经过中间件）
    ↓
res.revalidate("/") 触发 ISR 重新生成
    ↓
window.location.reload() 全量刷新页面
```

---

## 八、关键设计模式总结

| 模式 | 用途 | 本项目实现 |
|------|------|---------|
| **ISR 增量静态再生** | 结合 SSG 性能与 SSR 动态性 | `getStaticProps` + `res.revalidate` |
| **SWR Fallback 数据桥** | 服务端数据传递到客户端缓存 | 页面级 `SWRConfig` + `fallback` |
| **useEffect 手动注入 Context** | 页面级 props → 全局 Context | `setSettings(initialSettings)` |
| **Dynamic Import 禁用 SSR** | 避免浏览器 API 报错 | `dynamic(..., { ssr: false })` |
| **中间件 API 网关防护** | 保护 API 端点安全 | `middleware.js` + `matcher: "/api/:path*"` |
| **双轨数据流** | 不同数据走不同路径 | props 直读 vs Context 中转 vs SWR 缓存 |
| **Hash 变更检测** | 配置变更感知 | `useSWR("/api/hash")` + localStorage 对比 |

---

## 九、核心文件速查

| 文件 | 职责 | 执行环境 |
|------|------|---------|
| [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next.config.js) | Next.js 配置、i18n、输出模式 | 构建时 + 运行时（Node） |
| [src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/middleware.js) | API 路由 Host 校验（不拦截页面请求） | Edge Runtime |
| [src/pages/_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_document.jsx) | HTML 文档结构 | 仅服务端 |
| [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_app.jsx) | 全局 Providers（不传 initialSettings） | 服务端 + 客户端 |
| [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx) | 首页组件 + getStaticProps + useEffect 注入 Context | 混合 |
| [src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/settings.jsx) | SettingsContext（支持 initialSettings prop） | 服务端 + 客户端 |
| [src/utils/contexts/theme.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/theme.jsx) | ThemeContext（降级到 localStorage/系统偏好） | 服务端 + 客户端 |
| [src/utils/contexts/color.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/color.jsx) | ColorContext（降级到 localStorage/默认 "slate"） | 服务端 + 客户端 |
| [src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/config/api-response.js) | 服务端数据聚合逻辑 | 仅服务端 |
