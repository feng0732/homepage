# Homepage 错误兜底与 Fallback 逻辑梳理

本文档从代码层面梳理 Homepage 项目中的错误来源、降级分支、占位展示和恢复时机。整体架构采用**分层防御**策略，从 SSR 构建阶段到前端组件渲染，共 10+ 层容错机制。

---

## 一、整体架构概览

错误处理体系按请求流向可分为以下 10 层：

| 层级 | 位置 | 防御目标 |
|------|------|----------|
| L1 | SSR 构建层 | `getStaticProps` 配置读取异常 |
| L2 | 配置校验层 | `/api/validate` YAML 格式/语法错误 |
| L3 | ErrorBoundary 组件树 | 渲染期 JS 异常（崩溃保护） |
| L4 | 服务端 HTTP 代理层 | DNS 解析失败 / 网络错误 / gzip 解压错误 |
| L5 | 代理路由层 | 未知 widget 类型 / 未授权端点 / 数据校验失败 |
| L6 | `useWidgetAPI` Hook 层 | SWR 请求错误 + 业务数据 error 归一化 |
| L7 | Widget 组件内部 | 多接口并行的错误与 loading 分支 |
| L8 | Container/Block 渲染层 | 错误面板 / 骨架屏占位 |
| L9 | Widget 映射缺失 | 找不到 widget 组件时的占位 UI |
| L10 | 状态指示器 | Ping/Docker Status/SiteMonitor 的降级渲染 |

---

## 二、L1：SSR 构建层 — `getStaticProps`

### 异常来源
`getStaticProps` 在构建时（或 ISR 重验证时）读取配置文件：
- `getSettings()` 读取 `settings.yaml`
- `servicesResponse()` 合并 Docker/K8s/Config 三类服务
- `bookmarksResponse()` 读取 `bookmarks.yaml`
- `widgetsResponse()` 读取 `widgets.yaml`
- `serverSideTranslations()` 加载国际化语言包

### 降级分支
[pages/index.jsx#L55-L94](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L55-L94)

```javascript
export async function getStaticProps() {
  try {
    // ... 正常读取配置 ...
    return { props: { initialSettings, fallback: {...}, ...translations } };
  } catch (e) {
    // 降级：空配置 + 空数组 + 英文
    return {
      props: {
        initialSettings: {},
        fallback: {
          "/api/services": [],   // 空服务列表
          "/api/bookmarks": [],  // 空书签列表
          "/api/widgets": [],    // 空 widget 列表
          "/api/hash": false,    // hash 校验失效
        },
        ...(await serverSideTranslations("en")), // 语言降级为 en
      },
    };
  }
}
```

**关键逻辑：**
1. `initialSettings` 置空 → 页面使用默认主题/颜色/布局
2. SWR `fallback` 置空 → 前端 `useSWR("/api/services")` 等初始返回空数组
3. 语言强制为 `en` → 避免 i18n 崩溃

### 恢复时机
- 下一次 ISR 触发（手动调用 `/api/revalidate` 或 `revalidate` 周期）
- 重新构建/部署

---

## 三、L2：配置校验层 — `/api/validate`

### 异常来源
[pages/api/validate.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/api/validate.js)

在 6 个 YAML 配置文件上执行 `checkAndCopyConfig()`：
```
docker.yaml, settings.yaml, services.yaml, bookmarks.yaml, kubernetes.yaml, proxmox.yaml
```
常见错误：
- YAML 语法错误（缩进、冒号）
- 无法解析的字段
- 示例条目未删除（`example.com` 等）

### 降级分支
[pages/index.jsx#L133-L183](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L133-L183)

前端有两种错误界面：

**分支 A：单一严重错误（`validateError`，通常是 settings 本身格式错）**
```jsx
if (validateError) {
  // 全屏红色错误面板：显示 BiError 图标 + 原始错误文本
  <div className="w-full h-screen ... bg-rose-200 dark:bg-rose-800">
    <pre>{validateError}</pre>
  </div>
}
```

**分支 B：多个非致命错误（`errorsData.length > 0`，如某个配置有警告）**
```jsx
if (errorsData && errorsData.length > 0) {
  // 全屏琥珀色面板：按配置文件分块显示 name/config/reason/行号
  errorsData.map((error, i) => (
    <div className="bg-amber-200 dark:bg-amber-800">
      {error.name} - {error.config}
      Reason: "{error.reason}" at line {error.mark?.line}
    </div>
  ))
}
```

### 恢复时机
- 用户修复 YAML 后，`stale` 机制（localStorage hash 对比）检测到配置变化 → 触发 `/api/revalidate` → `window.location.reload()`
- 手动刷新页面

---

## 四、L3：React ErrorBoundary — 崩溃保护

### 异常来源
React 子组件树在渲染期间抛出的**同步异常**（异步错误不捕获）：
- JS 运行时错误（访问 undefined 属性、类型错误）
- 渲染期异常（如 map 非数组、JSON.parse 失败）
- 第三方库渲染崩溃

**注意：** ErrorBoundary 无法捕获事件处理器中的异步错误、SSR 错误和自身内部错误。

### 部署位置（5 处）
| 位置 | 代码位置 | 保护范围 |
|------|----------|----------|
| 全局 | [pages/index.jsx#L185-L191](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L185-L191) | 整个 `<Home />` 组件 |
| Info Widget | [components/widgets/widget.jsx#L25-L28](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/widgets/widget.jsx#L25-L28) | 单个信息 widget（weather/glances 等） |
| Service Widget | [components/services/widget.jsx#L14-L17](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/widget.jsx#L14-L17) | 单个服务 widget（sonarr/portainer 等） |
| Bookmarks List | [components/bookmarks/group.jsx#L75-L77](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/bookmarks/group.jsx#L75-L77) | 单个书签组的列表 |

### 降级分支
[components/errorboundry.jsx#L23-L36](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/errorboundry.jsx#L23-L36)

```jsx
if (errorInfo) {
  return (
    <div className="bg-rose-100 dark:bg-rose-900 ... rounded-md p-2 m-1">
      <div className="font-medium">Something went wrong.</div>
      <details className="text-xs font-mono whitespace-pre">
        <summary>{error.toString()}</summary>
        {errorInfo.componentStack}
      </details>
    </div>
  );
}
```

**展示效果：**
- 玫瑰色（rose）错误卡片，不破坏整体布局
- `<details>` 默认折叠，展开可见组件堆栈
- 同时 `console.error` 输出到浏览器控制台

### 恢复时机
**ErrorBoundary 一旦进入错误状态就无法自动恢复**，只能通过：
- 组件卸载并重新挂载（key 变化、切换 tab、路由切换）
- 整页刷新
- ErrorBoundary 的 props.children 引用变化（实际上 `getDerivedStateFromError` 后的 `componentDidCatch` 不自动重置，需父组件提供 `key` prop 或实现 `resetErrorBoundary` 机制，当前实现未提供）

---

## 五、L4：服务端 HTTP 代理层

### 5.1 DNS 解析 Fallback（Alpine/musl 兼容）

**异常来源**：在 Kubernetes Alpine 容器中，`dns.lookup`（使用系统 getaddrinfo + musl libc）有时返回 `ENOTFOUND` / `EAI_NONAME`，但 `dns.resolve*`（c-ares 解析器）能正常解析。

**降级分支**：[utils/proxy/http.js#L111-L225](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L111-L225)

```
1. hostname 已是 IP → 直接返回
2. dns.lookup() → 成功：返回
3. dns.lookup() 失败且 code ∈ {ENOTFOUND, EAI_NONAME}
   ├─ family=6 → dns.resolve6()
   ├─ family=4 → dns.resolve4()
   └─ 未指定  → dns.resolve4() → 失败再 dns.resolve6()
4. 所有 resolver 均失败 → 输出 debug 日志后回调原始错误
```

**恢复时机**：每次 HTTP 请求都会重新执行整个解析链路，无需手动恢复。

### 5.2 gzip 解压 Fallback

**异常来源**：某些服务返回的 gzip 响应不完整或格式错误，`zlib.createUnzip()` 触发 `error` 事件。

**降级分支**：[utils/proxy/http.js#L47-L52](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L47-L52)

```javascript
responseContent.on("error", (e) => {
  if (e) logger.error(e);
  responseContent = response; // fallback：切回原始未解压流
});
response.pipe(responseContent);
```

**效果**：解压失败 → 直接使用原始响应体（可能是压缩的二进制，上层 JSON.parse 会再失败，但不会崩溃）。

### 5.3 HTTP 请求异常兜底

**异常来源**：
- 连接超时 / ECONNREFUSED / ECONNRESET
- TLS 证书错误
- 目标服务不可达

**降级分支**：[utils/proxy/http.js#L268-L293](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L268-L293)

```javascript
try {
  const [status, contentType, data, responseHeaders] = await request;
  return [status, contentType, data, responseHeaders, params];
} catch (err) {
  logger.error("Error calling ...");
  return [
    500,
    "application/json",
    {
      error: {
        message: rawError?.message ?? "Unknown error",
        url: sanitizeErrorURL(url),     // 脱敏：apikey/token → ***
        rawError,
      },
    },
    null,
  ];
}
```

**关键点：** `httpProxy` 从不 throw，异常被 `catch` 后**包装成 `{ error: {...} }` 的 200 OK 响应返回**（但 status code 仍为 500）。上层 `genericProxyHandler` 会透传这个 `{error}` 结构。

### 恢复时机
- SWR 自动重试（默认指数退避）
- `refreshInterval` 周期刷新
- 用户刷新页面

---

## 六、L5：代理路由层 — `/api/services/proxy`

### 异常来源 & 降级分支
[pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/api/services/proxy.js)

整个 handler 被 try-catch 包裹（最后防线）：
```javascript
try {
  // 业务逻辑
} catch (e) {
  if (e) logger.error(e);
  return res.status(500).send({ error: "Unexpected error" });
}
```

**各检查点（403 降级）：**
| 检查项 | 条件 | 返回 |
|--------|------|------|
| widget 类型 | `!widgets[type]` | `{ error: "Unknown proxy service type" }` |
| endpoint 缺失+非 calendar | `!req.query.endpoint` 且非 calendar | 直接调用 handler（可能返回 4xx） |
| method 不匹配 | `mapping.method !== req.method` | `{ error: "Unsupported method" }` |
| endpoint 未映射 | `!mapping.endpoint` | `{ error: "Unsupported service endpoint" }` |
| segment 非法 | `segments[key]` 含 `/` `\` `..` | `{ error: "Unsupported segment" }` |
| unmapped 请求 | 无 mapping + 不匹配 allowedEndpoints | `{ error: "Unmapped proxy request." }` |

### 数据校验降级
[utils/proxy/validate-widget-data.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/validate-widget-data.js)

在 `genericProxyHandler` 中调用：
```javascript
if (status === 200) {
  if (!validateWidgetData(widget, endpoint, resultData)) {
    return res.status(status).json({
      error: { message: "Invalid data", url, data: resultData }
    });
  }
}
```

校验内容：
1. Buffer → JSON 解析（失败返回 `{error: {message: "Invalid data"}}`）
2. `mapping.validate` 指定的必填字段存在性检查

### HTTP 错误状态降级
[utils/proxy/handlers/generic.js#L75-L91](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/handlers/generic.js#L75-L91)

```javascript
if (status >= 400) {
  logger.debug("HTTP Error %d calling ...", status, ...);
  return res.status(status).json({
    error: {
      message: "HTTP Error",
      url: sanitizeErrorURL(url),
      data: Buffer.isBuffer(resultData) ? ...toString() : resultData,
    },
  });
}
```

**关键：** 所有 4xx/5xx 响应统一包装为 `{ error: {message, url, data} }` 的 JSON 结构，前端可一致处理。

### 恢复时机
- SWR 自动重试 / 轮询
- 目标服务恢复后，下一次请求即成功

---

## 七、L6：`useWidgetAPI` Hook — 错误归一化

### 异常来源
1. SWR 网络层错误（fetch 失败、500、CORS 等）→ `error`
2. 业务数据中的 `data.error`（L4/L5 返回的 `{error: {...}}` 结构）

### 降级分支
[utils/proxy/use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/use-widget-api.js)

```javascript
export default function useWidgetAPI(widget, ...options) {
  // 空 URL 跳过请求（endpoint === "" 或 formatProxyUrl 失败）
  let url = formatProxyUrl(widget, ...options);
  if (options[0] === "") { url = null; }

  const { data, error, mutate } = useSWR(url, config);

  // 关键：将 data.error 上浮为顶层 error
  return { data, error: data?.error ?? error, mutate };
}
```

**设计要点：**
- `data?.error ?? error`：业务层 error 优先于网络层 error
- 返回的 `error` 可能是：字符串、对象 `{message,url,rawError,data}`、Error 实例、数组 `[500, error]`（L4 原始格式）

### 恢复时机
- `mutate()` 手动触发重验证
- SWR 自动重试（`onErrorRetry`）
- `refreshInterval` 定时刷新
- `useWindowFocus` → 窗口聚焦时重验证 hash（首页级触发重加载）

---

## 八、L7：Widget 组件内部 — 多接口分支模式

典型 widget（如 sonarr、portainer）并行调用多个 endpoint，采用**错误优先、加载其次**的分支模式。

### 示例 1：Sonarr（4 个接口）
[widgets/sonarr/component.jsx#L28-L57](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/widgets/sonarr/component.jsx#L28-L57)

```javascript
const { data: wantedData, error: wantedError }   = useWidgetAPI(widget, "wanted/missing");
const { data: queuedData, error: queuedError }   = useWidgetAPI(widget, "queue");
const { data: seriesData, error: seriesError }    = useWidgetAPI(widget, "series");
const { data: queueDetailsData, error: queueDetailsError } = useWidgetAPI(widget, "queue/details");

// 分支 1：任一错误 → 错误面板（短路）
if (wantedError || queuedError || seriesError || queueDetailsError) {
  return <Container service={service} error={wantedError ?? queuedError ?? seriesError ?? queueDetailsError} />;
}

// 分支 2：任一数据未到 → 骨架屏（Block 无 value 触发 animate-pulse）
if (!wantedData || !queuedData || !seriesData || !queueDetailsData) {
  return (
    <Container service={service}>
      <Block label="sonarr.wanted" />   {/* 无 value → 占位 "-" + animate-pulse */}
      <Block label="sonarr.queued" />
      <Block label="sonarr.series" />
    </Container>
  );
}

// 分支 3：正常渲染
```

### 示例 2：Portainer（双重 error 检测）
[widgets/portainer/component.jsx#L76-L79](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/widgets/portainer/component.jsx#L76-L79)

```javascript
if (containersCount.error || containersCount.message) {
  // 数据本身可能是 error 对象（如环境变量未配置导致的业务错误）
  return <Container service={service} error={containersCount?.error ?? containersCount} />;
}
```

### 恢复时机
- SWR 自动重试机制
- 配置修复后 hash 变化触发 reload
- `refreshInterval` 轮询

---

## 九、L8：Container / Block / Error 渲染层

### 9.1 Container — 全局错误隐藏开关
[components/services/widget/container.jsx#L24-L30](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/widget/container.jsx#L24-L30)

```javascript
if (error) {
  if (settings.hideErrors || service.widget.hide_errors) {
    return null;  // 完全不渲染（隐藏所有错误 UI）
  }
  return <Error service={service} error={error} />;
}
```

### 9.2 Block — 骨架屏占位
[components/services/widget/block.jsx#L38-L48](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/widget/block.jsx#L38-L48)

```jsx
<div
  className={classNames(
    "...",
    value === undefined ? "animate-pulse" : "",   // 无值 → 脉冲动画（骨架屏）
  )}
>
  <div className="font-thin text-sm">
    {value === undefined || value === null ? "-" : value}   {/* 占位 "-" */}
  </div>
  <div className="font-bold text-xs uppercase">{t(label)}</div>
</div>
```

**展示策略：**
| 状态 | value | 文本 | CSS |
|------|-------|------|-----|
| 加载中 | `undefined` | `-` | `animate-pulse` 脉冲动画 |
| 正常 | 数字/字符串 | 值本身 | 无额外动画 |
| 无数据 | `null` | `-` | 无动画（静态占位） |

### 9.3 Error 组件 — 可折叠错误详情面板
[components/services/widget/error.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/widget/error.jsx)

**错误归一化管道：**
```
原始 error（可能是 string/number/{data:{error}}/array/Error）
  │
  ├─ string → { message: error }
  ├─ number → { message: `Error ${number}` }
  └─ error.data.error → 上浮为 error（脱壳 axios 包装）
  ↓
最终渲染字段：
  ├─ message  → widget.api_error
  ├─ url      → widget.url（脱敏后）
  ├─ rawError → widget.raw_error（JSON 展开）
  └─ data     → widget.response_data（Buffer 转字符串）
```

**展示效果：**
- 玫瑰色 `<summary>` 摘要栏，带 `IoAlertCircle` 图标和 `widget.api_error` 文案
- 点击展开：白底黑字的 `<details>` 面板，字段化展示 message / url / rawError / responseData
- `sanitizeErrorURL` 已在 L4/L5 层将 `apikey/token/auth` 等替换为 `***`

### 恢复时机
- 数据恢复后，error prop 清除 → 自动重新渲染正常内容
- 无需手动干预（SWR mutuate 触发）

---

## 十、L9：Widget 类型缺失兜底

### 10.1 Info Widget 缺失
[components/widgets/widget.jsx#L31-L35](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/widgets/widget.jsx#L31-L35)

```jsx
if (InfoWidget) return <ErrorBoundary><InfoWidget /></ErrorBoundary>;

// 兜底：显示 "Missing <widget.type>"
return (
  <div className="flex-none flex flex-row items-center justify-center">
    Missing <strong>{widget.type}</strong>
  </div>
);
```

### 10.2 Service Widget 缺失
[components/services/widget.jsx#L20-L24](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/widget.jsx#L20-L24)

```jsx
if (ServiceWidget) return <ErrorBoundary><ServiceWidget /></ErrorBoundary>;

// 兜底：带国际化的 "Missing widget type: xxx"
return (
  <div className="... service-missing">
    <div className="font-thin text-sm">
      {t("widget.missing_type", { type: widget.type })}
    </div>
  </div>
);
```

### 恢复时机
- 添加缺失的 widget 实现文件到 `widgets/components.js` 映射表
- 修复 YAML 中错误的 widget 类型名

---

## 十一、L10：状态指示器降级渲染

### 11.1 Ping 状态
[components/services/ping.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/ping.jsx)

| 状态 | 颜色 | 文本 |
|------|------|------|
| 初始加载（!data && !error） | `opacity-20` | `ping` + 不可用 |
| error | `text-rose-500` | `ping.error` |
| !data.alive | `text-rose-500/80` | `ping.down` |
| alive=true | `text-emerald-500/80` | 具体延迟值 (ms) |

### 11.2 Docker Status
[components/services/status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/components/services/status.jsx)

| 状态 | 颜色 | 说明 |
|------|------|------|
| error | `text-rose-500/80` | `docker.error` |
| running+healthy | `text-emerald-500/80` | `docker.healthy` |
| running+starting | `text-blue-500/80` | `docker.starting` |
| running+unhealthy | `text-orange-400/*` | `docker.unhealthy` |
| exited/not_found | `text-orange-400/*` | 对应 i18n key |
| 未知（初始） | `opacity` 降低 | `docker.unknown` |

---

## 十二、完整数据流与错误传播路径

```
┌─────────────────────────────────────────────────────────────────────┐
│ 构建阶段 (getStaticProps)                                            │
│  ┌─ getSettings() / servicesResponse() / bookmarksResponse()        │
│  │     ↓ try-catch                                                   │
│  └─ 失败 → initialSettings={} + fallback=[] + language=en           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│ 页面初始化                                                           │
│  ┌─ useSWR("/api/validate")                                          │
│  │     ↓                                                             │
│  ├─ validateError? → 全屏玫瑰错误面板 (L2 分支 A)                    │
│  ├─ errorsData.length>0? → 琥珀配置错误列表 (L2 分支 B)              │
│  └─ 通过 → 进入 <Home> 并被全局 ErrorBoundary 包裹 (L3)              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
        Info Widgets               Services/Bookmarks
          (L9 缺失检测)               (L3 局部保护)
               │                           │
               ▼                           ▼
        <ErrorBoundary>              <ErrorBoundary>
          (widget.jsx)                 (service widget)
               │                           │
               ▼                           ▼
        Widget Component           <Container> (L8 hideErrors 检查)
          (L7 多分支)                       │
               │                           ▼
        ┌──────┴──────┐            Widget Component
        ▼             ▼              (L7 error/loading 分支)
   has error?    !data?                  │
        │           │                ┌────┴─────┐
        ▼           ▼                ▼          ▼
  <Container error>  <Block 无值>  error?    !data?
  → <Error L9>      → animate-pulse   │          │
        │                            ▼          ▼
        │                     <Error L8>   <Block 无值>
        │                              animate-pulse
        │                                   │
        │                                   ▼
        │                           正常渲染 Block
        │                                with value
        │                                   │
        └───────────────────┬───────────────┘
                            │
                   ┌────────▼─────────┐
                   │  数据请求链路      │
                   │  useWidgetAPI    │
                   │  (L6 归一化)      │
                   └────────┬─────────┘
                            │
                   ┌────────▼─────────┐
                   │  /api/services/  │
                   │  proxy (L5)      │
                   │  · 类型/端点校验  │
                   │  · 数据格式校验  │
                   │  · HTTP 4xx 包装 │
                   └────────┬─────────┘
                            │
                   ┌────────▼─────────┐
                   │  httpProxy (L4)  │
                   │  · DNS 双解析    │
                   │  · gzip fallback │
                   │  · catch 网络错  │
                   │  → {error:{...}} │
                   └──────────────────┘
```

---

## 十三、恢复时机汇总

| 触发方式 | 影响层级 | 说明 |
|----------|----------|------|
| **SWR 自动重试** | L6-L8 | 默认指数退避，`onErrorRetry` 控制 |
| **refreshInterval** | L6-L8 | 各 widget 自定义轮询周期（ms） |
| **窗口聚焦** | L2 | `useWindowFocus` → `mutateHash` → hash 比对 |
| **hash 变化** | L1-L8 | localStorage.hash ≠ `/api/hash` → `setStale(true)` → 加载动画 → 调用 `/api/revalidate` → `window.location.reload()` |
| **手动刷新** | L1-L10 | F5 / reload 按钮 |
| **ISR 重验证** | L1 | `/api/revalidate` + ISR 周期触发重建 |
| **ErrorBoundary 重挂载** | L3 | 父组件 key 变化 / 路由切换（当前实现未提供 reset API） |
| **mutate() 手动** | L6-L8 | `useWidgetAPI` 返回的 mutate 函数（各 widget 内部使用） |

---

## 十四、关键设计模式总结

### 1. 错误归一化（三级包装）
- **L4 httpProxy**：`[500, error]` → `{ error: {message, url, rawError} }`
- **L5 genericProxyHandler**：HTTP 4xx → `{ error: {message: "HTTP Error", ...} }`
- **L6 useWidgetAPI**：`error = data?.error ?? error`（业务 error 与网络 error 统一）

### 2. 渐进式降级
```
正常 → skeleton (Block animate-pulse) → error 面板 → ErrorBoundary 玫瑰卡片 → 全屏错误页
```
每一级都尽量缩小影响范围：单 Block → 单 Widget → 单组 → 整页。

### 3. 错误脱敏
`sanitizeErrorURL` 在 L4/L5 层将 URL 中的 `apikey`、`token`、`auth`、`access_token` 等敏感参数替换为 `***`，确保日志和错误 UI 中不泄露凭证。

### 4. 双重错误源检测
典型 widget 同时检查：
- `useWidgetAPI` 返回的顶层 `error`（网络/代理层）
- `data.error` / `data.message`（业务层，如 portainer 环境变量缺失）

### 5. SWR fallback 作为 SSR-客户端桥梁
`getStaticProps` 构建时预取的数据 → 注入 `SWRConfig.fallback` → 首屏避免 4 个 `/api/*` 请求，同时作为构建失败时的空数组兜底。
