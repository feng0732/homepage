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

**核心代码解析：**

[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next.config.js#L1-L19) 中的路由相关配置：

```javascript
const { i18n } = require("./next-i18next.config");

const nextConfig = {
  reactStrictMode: true,
  output: "standalone",  // 独立输出模式，用于 Docker 部署
  i18n,  // 国际化路由配置
};
```

[next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next-i18next.config.js#L112-L117) 中的 i18n 路由配置：

```javascript
module.exports = {
  i18n: {
    defaultLocale: "en",
    locales: ["en"],
  },
  // ...
};
```

### 1.2 中间件（Middleware）

[src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/middleware.js#L1-L23) 在路由匹配前执行，用于请求预处理：

```javascript
import { NextResponse } from "next/server";

export function middleware(req) {
  // 检查 Host 头，防止 Host 头攻击
  const host = req.headers.get("host");
  const port = process.env.PORT || 3000;
  let allowedHosts = [`localhost:${port}`, `127.0.0.1:${port}`, `[::1]:${port}`];
  
  // 环境变量配置允许的主机
  if (process.env.HOMEPAGE_ALLOWED_HOSTS) {
    allowedHosts = allowedHosts.concat(process.env.HOMEPAGE_ALLOWED_HOSTS.split(","));
  }
  
  // 验证失败返回 400
  if (!allowAll && (!host || !allowedHosts.includes(host))) {
    return NextResponse.json(
      { error: "Host validation failed" }, 
      { status: 400 }
    );
  }
  
  return NextResponse.next();  // 继续路由处理
}

// 只对 /api/* 路由应用中间件
export const config = {
  matcher: "/api/:path*",
};
```

**执行时机：** 请求到达 → 中间件 → 路由匹配 → 页面/API 处理

### 1.3 动态路由解析

[src/pages/api/config/[path].js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/api/config/[path].js#L1-L34) 展示了动态路由参数的获取：

```javascript
export default async function handler(req, res) {
  // 从 req.query 获取动态路由参数
  const { path: relativePath } = req.query;
  
  // 只允许访问 custom.css 和 custom.js
  if (!["custom.css", "custom.js"].includes(relativePath)) {
    return res.status(422).end("Unsupported file");
  }
  
  // 根据路径读取配置文件...
}
```

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
    
    // 1. 读取设置配置
    const { providers, ...settings } = getSettings();
    
    // 2. 并行获取三类数据
    const services = await servicesResponse();   // 服务数据
    const bookmarks = await bookmarksResponse(); // 书签数据
    const widgets = await widgetsResponse();     // 小部件数据
    
    const language = normalizeLanguage(settings.language);
    
    // 3. 返回 props 给页面组件
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
    // 错误处理，返回降级数据
    return {
      props: {
        initialSettings: {},
        fallback: { /* 空数据 */ },
        ...(await serverSideTranslations("en")),
      },
    };
  }
}
```

**关键特性：**
- 仅在 **Node.js 环境** 执行，不会打包到客户端
- 构建时执行一次，之后可通过 `revalidate` 触发重新生成
- 返回的 `props` 会序列化为 JSON 嵌入 HTML

### 2.2 配置文件读取链

[src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/config/config.js#L82-L103) 中的 `getSettings` 函数：

```javascript
export function getSettings() {
  // 检查并复制默认配置（如果不存在）
  checkAndCopyConfig("settings.yaml");
  
  const settingsYaml = join(CONF_DIR, "settings.yaml");
  const rawFileContents = readFileSync(settingsYaml, "utf8");
  
  // 环境变量替换：{{HOMEPAGE_VAR_XXX}}
  const fileContents = substituteEnvironmentVars(rawFileContents);
  
  // YAML 解析
  const initialSettings = yaml.load(fileContents) ?? {};
  
  // 兼容性处理...
  return initialSettings;
}
```

### 2.3 多源数据聚合

[src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/config/api-response.js#L158-L256) 中的 `servicesResponse` 展示了复杂的数据聚合：

```javascript
export async function servicesResponse() {
  // 1. 从 Docker 自动发现服务
  const discoveredDockerServices = cleanServiceGroups(await servicesFromDocker());
  
  // 2. 从 Kubernetes 自动发现服务
  const discoveredKubernetesServices = cleanServiceGroups(await servicesFromKubernetes());
  
  // 3. 从 services.yaml 读取手动配置
  const configuredServices = cleanServiceGroups(await servicesFromConfig());
  
  // 4. 合并三类来源，按权重排序
  const mergedGroup = {
    name: groupName,
    services: [
      ...discoveredDockerGroup.services,
      ...discoveredKubernetesGroup.services,
      ...configuredGroup.services
    ].sort(compareServices),
    groups: [...configuredGroup.groups],
  };
  
  // 5. 按 settings.yaml 中的 layout 排序
  // ...
  
  return pruneEmptyGroups(allGroups);
}
```

### 2.4 ISR 增量重新验证

[src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/api/revalidate.js#L1-L8) 实现手动触发重新生成：

```javascript
export default async function handler(req, res) {
  try {
    await res.revalidate("/");  // 触发首页重新生成
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}
```

**触发时机（见 index.jsx 第 119-127 行）：**
```javascript
// 客户端检测到配置哈希变化时，触发重新验证
if (previousHash && previousHash !== hashData.hash) {
  fetch("/api/revalidate").then((res) => {
    if (res.ok) {
      window.location.reload();
    }
  });
}
```

---

## 三、页面拼装与 HTML 生成过程

Next.js 的页面渲染是一个分层组装的过程，涉及三个关键文件的协作。

### 3.1 渲染层级结构

```
请求到达
    ↓
middleware（API 路由前置检查）
    ↓
getStaticProps（服务端取数，获取 pageProps）
    ↓
_document.jsx（生成 HTML 骨架）
    ├─ <Html>
    ├─ <Head>（全局 meta、link、script）
    └─ <body>
        ├─ <Main />（占位符，注入 _app + 页面内容）
        └─ <NextScript />（注入 Next.js runtime 脚本）
            ↓
_app.jsx（全局包装）
    ├─ SWRConfig（全局数据获取配置）
    ├─ ColorProvider（颜色主题 Context）
    ├─ ThemeProvider（明暗主题 Context）
    ├─ SettingsProvider（设置 Context）
    ├─ TabProvider（标签页 Context）
    └─ <Component {...pageProps} />（实际页面组件）
        ↓
index.jsx（页面组件）
    └─ 业务内容
```

### 3.2 _document.jsx - HTML 骨架

[src/pages/_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_document.jsx#L1-L18) 定义整个 HTML 文档结构：

```javascript
import { Head, Html, Main, NextScript } from "next/document";

export default function Document() {
  return (
    <Html>
      <Head>
        {/* PWA 相关 meta */}
        <meta name="mobile-web-app-capable" content="yes" />
        <link rel="manifest" href="/site.webmanifest?v=4" />
        
        {/* 预加载自定义样式 */}
        <link rel="preload" href="/api/config/custom.css" as="style" />
        <link rel="stylesheet" href="/api/config/custom.css" />
      </Head>
      <body>
        {/* 服务端渲染的 React 内容将注入到这里 */}
        <Main />
        {/* Next.js 客户端 runtime 脚本将注入到这里 */}
        <NextScript />
      </body>
    </Html>
  );
}
```

**关键点：**
- 仅在服务端渲染执行，客户端不执行
- `Main` 和 `NextScript` 是必须的，不能省略
- 适合放置需要在 `<head>` 中的静态资源

### 3.3 _app.jsx - 全局应用包装

[src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_app.jsx#L73-L98) 提供全局上下文和配置：

```javascript
function MyApp({ Component, pageProps }) {
  return (
    <SWRConfig value={{ fetcher: (resource, init) => fetch(resource, init).then(res => res.json()) }}>
      <Head>
        {/* 全局 viewport 设置 */}
        <meta name="viewport" content="width=device-width, initial-scale=1.0, ..." />
      </Head>
      
      {/* 嵌套的 Context Providers */}
      <ColorProvider>
        <ThemeProvider>
          <SettingsProvider>
            <TabProvider>
              {/* pageProps 包含 getStaticProps 返回的数据 */}
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
- 接收 `Component`（当前页面组件）和 `pageProps`（服务端获取的数据）
- 服务端和客户端都会执行
- 适合放置全局 Providers、全局样式、全局布局

### 3.4 页面组件接收 props

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L97-L192) 中的 `Index` 组件接收服务端数据：

```javascript
function Index({ initialSettings, fallback }) {
  // 使用 SWRConfig 将服务端数据注入客户端缓存
  return (
    <SWRConfig value={{ 
      fallback,  // 服务端预取的数据
      fetcher: (resource, init) => fetch(resource, init).then(res => res.json()) 
    }}>
      <ErrorBoundary>
        <Home initialSettings={initialSettings} />
      </ErrorBoundary>
    </SWRConfig>
  );
}
```

### 3.5 服务端渲染出的 HTML 结构

最终生成的 HTML 大致结构：

```html
<!DOCTYPE html>
<html>
  <head>
    <!-- _document.jsx 中的 Head 内容 -->
    <meta name="mobile-web-app-capable" content="yes">
    <link rel="manifest" href="/site.webmanifest?v=4">
    <link rel="stylesheet" href="/api/config/custom.css">
    
    <!-- _app.jsx 中的 Head 内容 -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    
    <!-- 页面组件中的 Head 内容 -->
    <title>Homepage</title>
    <meta name="description" content="...">
    
    <!-- Next.js 自动注入的样式 -->
    <style data-next-hide-fouc>...</style>
  </head>
  <body>
    <!-- 服务端渲染的 React 应用内容 -->
    <div class="container m-auto flex flex-col...">
      <!-- 完整的页面 HTML... -->
    </div>
    
    <!-- 序列化的 pageProps（供客户端 hydration 使用） -->
    <script id="__NEXT_DATA__" type="application/json">
      {
        "props": {
          "pageProps": {
            "initialSettings": {...},
            "fallback": {...},
            "_nextI18Next": {...}
          }
        },
        "page": "/",
        "query": {},
        "buildId": "abc123",
        "isFallback": false,
        "gsp": true,  // 标记使用 getStaticProps
        "scriptLoader": []
      }
    </script>
    
    <!-- Next.js runtime 脚本 -->
    <script src="/_next/static/chunks/main.js" async></script>
    <script src="/_next/static/chunks/pages/index.js" async></script>
  </body>
</html>
```

---

## 四、客户端接续渲染（Hydration）

Hydration 是将服务端渲染的静态 HTML "激活" 为交互式 React 应用的过程。

### 4.1 Hydration 执行流程

```
浏览器加载 HTML
    ↓
1. 解析 HTML，显示静态内容（用户已可看到页面）
    ↓
2. 下载并执行 Next.js runtime（main.js）
    ↓
3. 读取 <script id="__NEXT_DATA__"> 中的序列化数据
    ↓
4. React 客户端初始化，开始 Hydration
    ↓
5. 逐节点对比服务端 HTML 和客户端渲染结果
    ↓
6. 绑定事件监听器，页面变为可交互
    ↓
7. SWR 从 fallback 恢复缓存，后续数据请求使用客户端逻辑
```

### 4.2 数据无缝接续：SWR fallback 机制

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L185-L191) 中的 SWR 配置：

```javascript
<SWRConfig value={{ 
  fallback,  // 来自 getStaticProps 的预取数据
  fetcher: (resource, init) => fetch(resource, init).then(res => res.json()) 
}}>
  <ErrorBoundary>
    <Home initialSettings={initialSettings} />
  </ErrorBoundary>
</SWRConfig>
```

**效果：** 当客户端组件调用 `useSWR("/api/services")` 时，不会立即发送请求，而是直接使用 `fallback` 中已有的数据。

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L226-L228) 中的客户端数据获取：

```javascript
// 由于 fallback 中已有数据，这三个 useSWR 不会立即发送网络请求
const { data: services } = useSWR("/api/services");
const { data: bookmarks } = useSWR("/api/bookmarks");
const { data: widgets } = useSWR("/api/widgets");
```

### 4.3 SSR 与 CSR 分界：dynamic 组件

对于依赖浏览器 API 的组件，使用 `next/dynamic` 禁用 SSR：

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L30-L40)：

```javascript
// 这些组件只在客户端渲染，服务端渲染时显示空内容
const ThemeToggle = dynamic(() => import("components/toggles/theme"), {
  ssr: false,  // 禁用服务端渲染
});

const ColorToggle = dynamic(() => import("components/toggles/color"), {
  ssr: false,
});

const Version = dynamic(() => import("components/version"), {
  ssr: false,
});
```

**原因：** 这些组件可能使用 `window`、`localStorage` 等浏览器 API，在服务端渲染时会报错。

### 4.4 Context 状态恢复

[src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/settings.jsx#L1-L15) 展示了如何从服务端 props 恢复状态：

```javascript
export function SettingsProvider({ initialSettings, children }) {
  // 使用服务端传入的 initialSettings 作为初始状态
  const [settings, setSettings] = useState(() => initialSettings ?? {});
  
  // 当 initialSettings 变化时更新状态（如 ISR 重新验证后）
  useEffect(() => {
    if (initialSettings !== undefined) {
      setSettings(initialSettings ?? {});
    }
  }, [initialSettings]);
  
  const value = useMemo(() => ({ settings, setSettings }), [settings]);
  
  return <SettingsContext.Provider value={value}>{children}</SettingsContext.Provider>;
}
```

**数据流：**
`getStaticProps` → `pageProps.initialSettings` → `SettingsProvider` → `useContext(SettingsContext)`

### 4.5 Hydration 不匹配问题

测试文件 [src/__tests__/pages/_app.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/__tests__/pages/_app.test.jsx#L1-L37) 揭示了 Next.js 内部依赖：

```javascript
// Next's Head implementation relies on internal Next contexts; 
// stub it for unit tests.
vi.mock("next/head", () => ({
  default: ({ children }) => <>{children}</>,
}));
```

**避免 Hydration Mismatch 的原则：**
1. 服务端和客户端渲染的初始 HTML 必须完全一致
2. 依赖浏览器 API 的代码放在 `useEffect` 或 `dynamic({ ssr: false })` 中
3. 随机值、时间戳等在服务端和客户端可能不同的值，统一在客户端生成
4. 使用 `typeof window !== "undefined"` 进行环境判断

### 4.6 客户端更新机制

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx#L104-L131) 中的配置更新检测：

```javascript
// 窗口获得焦点时，重新获取配置哈希
useEffect(() => {
  if (windowFocused) {
    mutateHash();
  }
}, [windowFocused, mutateHash]);

// 检测到配置变化时，触发 ISR 重新验证并刷新
useEffect(() => {
  if (hashData) {
    if (typeof window !== "undefined") {
      const previousHash = localStorage.getItem("hash");
      
      if (!previousHash) {
        localStorage.setItem("hash", hashData.hash);
      }
      
      // 哈希变化说明配置已更新
      if (previousHash && previousHash !== hashData.hash) {
        setStale(true);
        localStorage.setItem("hash", hashData.hash);
        
        // 调用 revalidate API 触发重新生成
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

---

## 五、完整请求生命周期总结

### 5.1 首次访问（服务端渲染完整流程）

```
用户输入 URL 并回车
    ↓
DNS 解析 → TCP 连接 → TLS 握手
    ↓
HTTP 请求到达 Next.js 服务器
    ↓
[中间件层] src/middleware.js
  ├─ 检查 Host 头是否在白名单
  └─ 验证通过，继续处理
    ↓
[路由匹配]
  ├─ 匹配到 / 路由
  └─ 对应 src/pages/index.jsx
    ↓
[服务端取数]
  ├─ 调用 getStaticProps()
  │   ├─ getSettings() 读取 settings.yaml
  │   ├─ servicesResponse() 聚合服务数据
  │   ├─ bookmarksResponse() 读取书签
  │   ├─ widgetsResponse() 读取小部件
  │   └─ serverSideTranslations() 加载翻译
  └─ 返回 { props: { initialSettings, fallback, ... } }
    ↓
[页面渲染]
  ├─ _document.jsx 生成 HTML 骨架
  ├─ _app.jsx 包装全局 Providers
  ├─ index.jsx 渲染页面内容
  └─ 序列化为静态 HTML
    ↓
[数据注入]
  ├─ 将 pageProps 序列化为 JSON
  └─ 注入到 <script id="__NEXT_DATA__">
    ↓
HTTP 响应返回完整 HTML
    ↓
[客户端]
  ├─ 浏览器解析 HTML，显示内容
  ├─ 下载 Next.js runtime 脚本
  ├─ 读取 __NEXT_DATA__ 恢复数据
  ├─ React Hydration，绑定事件
  ├─ SWR 从 fallback 恢复缓存
  └─ 页面完全可交互
```

### 5.2 客户端导航（SPA 模式）

当用户点击页面内链接（使用 `next/link`）时：

```
用户点击链接
    ↓
[客户端路由]
  ├─ 阻止默认页面跳转
  ├─ 使用 History API 修改 URL
  └─ 不触发完整页面刷新
    ↓
[代码分割加载]
  ├─ 按需加载目标页面的 JS chunk
  └─ 不重新加载公共 chunk
    ↓
[客户端取数]
  ├─ 调用目标页面的 getStaticProps/getServerSideProps
  │  （通过 AJAX 请求 Next.js 服务器）
  └─ 获取新的 pageProps
    ↓
[客户端渲染]
  ├─ 卸载旧页面组件
  ├─ 使用新的 pageProps 渲染新页面
  └─ 更新 DOM，保持应用状态
```

---

## 六、关键设计模式总结

| 模式 | 用途 | 代码位置 |
|------|------|---------|
| **ISR 增量静态再生** | 结合 SSG 的性能和 SSR 的动态性 | `getStaticProps` + `res.revalidate` |
| **SWR Fallback** | 服务端数据无缝传递到客户端缓存 | `SWRConfig` + `fallback` 属性 |
| **Dynamic Import** | 按条件禁用 SSR，避免浏览器 API 报错 | `dynamic(() => import(...), { ssr: false })` |
| **Context 分层** | 全局状态跨服务端/客户端传递 | `_app.jsx` 中的嵌套 Providers |
| **中间件防护** | 路由级安全检查，不侵入业务代码 | `middleware.js` + `matcher` 配置 |
| **__NEXT_DATA__** | 序列化数据桥，连接服务端与客户端 | Next.js 自动注入的 script 标签 |

---

## 七、核心文件速查

| 文件 | 职责 | 执行环境 |
|------|------|---------|
| [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/next.config.js) | Next.js 配置、i18n、输出模式 | 构建时 + 运行时（Node） |
| [src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/middleware.js) | 路由前置中间件 | Edge Runtime |
| [src/pages/_document.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_document.jsx) | HTML 文档结构 | 仅服务端 |
| [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/_app.jsx) | 全局包装、Providers | 服务端 + 客户端 |
| [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/pages/index.jsx) | 首页组件 + getStaticProps | getStaticProps: 服务端<br/>组件: 服务端 + 客户端 |
| [src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/config/api-response.js) | 服务端数据聚合逻辑 | 仅服务端 |
| [src/utils/contexts/*.jsx](file:///d:/fz/0601/solo-dogfeeding/code/212-homepage/src/utils/contexts/settings.jsx) | 状态管理 Context | 服务端 + 客户端 |
