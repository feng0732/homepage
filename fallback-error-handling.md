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

### 5.2 gzip/deflate 解压 Fallback（⚠️ 实际存在逻辑缺陷）

**异常来源**：某些服务返回的 gzip/deflate 响应不完整或格式错误（截断、魔数错误、校验和失败），`zlib.createUnzip()` Transform 流触发 `error` 事件。

#### 代码执行时序（按行号展开）
[utils/proxy/http.js#L33-L62](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L33-L62)

```javascript
const request = requestor.request(url, params, (response) => {  // response = 原始 HTTP IncomingMessage
  const data = [];                                               // 步骤 0：data 闭包数组
  const contentEncoding = response.headers["content-encoding"]?.trim().toLowerCase();

  let responseContent = response;                                // 步骤 1：responseContent = 原始 response（引用 A）
  if (contentEncoding === "gzip" || contentEncoding === "deflate") {
    responseContent = createUnzip({ ... });                      // 步骤 2：responseContent = unzip Transform（引用 B）

    // zlib errors
    responseContent.on("error", (e) => {                         // 步骤 3：在【引用 B 对象】上挂 error 监听
      if (e) logger.error(e);
      responseContent = response;                                // 步骤 E-1：修改变量引用（仅局部变量，无效！）
    });
    response.pipe(responseContent);                              // 步骤 4：response.pipe(unzip B) —— 原始数据永远流入 unzip！
  }

  responseContent.on("data", (chunk) => {                        // 步骤 5：在【引用 B 对象】上挂 data 监听
    data.push(chunk);
  });

  responseContent.on("end", () => {                              // 步骤 6：在【引用 B 对象】上挂 end 监听
    addCookieToJar(url, response.headers);
    resolve([response.statusCode, response.headers["content-type"], Buffer.concat(data), response.headers]);
  });
});
```

#### 问题 1：事件监听器挂在哪？
- `responseContent` 在步骤 2 被赋值为 **unzip Transform 流对象（引用 B）**
- 步骤 5 的 `responseContent.on("data", ...)` 和步骤 6 的 `responseContent.on("end", ...)` **注册在引用 B 上**，不是变量名上
- 步骤 3 的 error 监听同样注册在引用 B 上
- 关键点：**监听器是绑在对象实例上的，一旦绑定就与变量名无关**

#### 问题 2：`responseContent = response` 能接管原始响应吗？
**答案：不能。这是一个无效赋值，原因有三：**

| 层级 | 原因 | 后果 |
|------|------|------|
| ① 监听器不迁移 | 步骤 E-1 只改了变量 `responseContent` 的指向，**引用 B 上已绑定的 data/end 监听器不会迁移到原始 response（引用 A）** | error 后触发的数据事件仍然只有 unzip 能收到，原始 response 没人监听 |
| ② pipe 已执行 | 步骤 4 `response.pipe(responseContent)` 此时 pipe 的目标是 unzip。`pipe()` 会调用 `readable.resume()`，**将原始 response 切换到 flowing 模式**，所有数据块会自动写入 unzip | 原始 response 的数据已经"被消费"，无法被二次读取或重新监听 |
| ③ error 事件后 Transform 行为 | Node.js `zlib.createUnzip()` 在 emit `error` 后，内部状态变为 errored，后续 `_transform` 不再处理数据。按照 Stream 规范，error 后可继续 emit `end`/`close`，但数据完整性无法保证 | 若 data 数组只收到部分解压数据 → 最终 `Buffer.concat(data)` 是**残缺内容** |

#### error / end / close 事件时序与 Promise resolve 的真实关系（基于真实 zlib 实验）

代码中 Promise 的唯一 resolve 路径是 [utils/proxy/http.js#L59-L62](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L59-L62) 挂在 `responseContent`（即引用 B，unzip 对象）的 `end` 事件上：

```javascript
responseContent.on("end", () => {
  addCookieToJar(url, response.headers);
  resolve([status, type, Buffer.concat(data), headers]);
});
```

**核心问题**：真实 `zlib.createUnzip({flush: Z_SYNC_FLUSH, finishFlush: Z_SYNC_FLUSH})` 在不同失败类型下，**是否会 emit `end` 来让 Promise resolve？** 如果不会，Promise 将永远 pending（请求悬挂、内存泄漏）。

##### 实验前提：代码使用的 unzip 配置
```javascript
createUnzip({
  flush: Z_SYNC_FLUSH,          // 立即刷新输出，允许部分解压
  finishFlush: Z_SYNC_FLUSH,    // _flush 阶段也用 SYNC 模式
})
```
`Z_SYNC_FLUSH` 是这段代码容错的关键来源：它让 zlib 在"压缩数据不完整"时不会阻塞等待后续字节，而是立即刷新当前已解压内容。注释 [http.js#L39-L41](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.js#L39-L41) 引用了 `request/request` 库，这是一种"浏览器兼容的宽松解码"策略。

##### 第一组事件总表：真实 Node.js zlib（对照实验结果）

| 事件 | 触发条件 | 魔数错 | 截断(PAYLOAD中) | 截断(TRAILER中) | CRC32篡改 | ISIZE篡改 | 正确gzip |
|------|---------|--------|----------------|----------------|-----------|-----------|---------|
| **`error`** | `destroy(err)` 时 emit | ✅ "incorrect header check" | ❌ **永远不触发** | ❌ **永远不触发** | ✅ "incorrect data check" | ✅ "incorrect length check" | ❌ |
| **`data`** | 解压成功字节块 | ❌ 0块 | ✅ **部分解压块** | ✅ 完整解压 | ⚠️ **取决于投递方式**（见下） | ⚠️ 同上 | ✅ 完整 |
| **`end`** | 上游 end → `_flush` 正常完成 | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ |
| **`close`** | zlib ctx 释放（destroy/finish后） | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **`finish`** | writable side 写入全部结束 | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ |
| **`unpipe`** | 上游 readable 解除 pipe | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

**关键反直觉发现（来自最小实验）**：
1. **场景 B（截断）在 Z_SYNC_FLUSH 下完全不触发 error** — zlib 把截断当作"正常结束的部分解压缩"，只 emit data/finish/end，不报错。
2. **场景 C（CRC/ISIZE 篡改）的 data 数量取决于 body 投递方式** — 同步一次性 push 时 data=0（zlib 内部在同一 tick 内完成解压+校验+destroy，data 未及 emit）；分 chunk 异步投递时 data 有部分解压字节。
3. **所有 error 场景一定不 emit `end`**（因为 zlib 走 destroy 路径跳过了 `_flush` 的正常结束）。
4. **`close` 是最可靠的终结事件** — 无论成功失败都 emit。代码当前**完全没监听 close**。

##### 第二组：三类解码失败的时序详解 + Promise resolve 路径

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  场景 A / D：魔数错误（0x1f8b 不对 / zlib误当gzip / 非压缩文本标注gzip头）    ║
╚══════════════════════════════════════════════════════════════════════════════╝

真实事件序列（实验验证）：
 T0 response.read() → push(body) + push(null)          // 代码模拟http.test.js#L70-L74
    response → pipe → gunzip.write(firstChunk)
    gunzip._transform: 前两字节不是 0x1f8b
      → zlib Z_DATA_ERROR → this.destroy(new Error("incorrect header check"))
      → destroy 内部: emit("error", err) → 关闭 z_stream → emit("close")
      → destroy 触发: source.unpipe(gunzip)             // pipe 链路断裂
 T1 error 监听执行: logger.error(e) + 无效赋值 responseContent = response
 T2 (end 监听永远不会触发)
    gunzip 已 destroyed → _flush 不执行 → 不 emit end

Promise 状态：⏳ 永远 pending  ❌
data 内容：空数组 [] （没有任何 data 事件）
close 事件：✅ 已 emit（但代码没监听）


╔══════════════════════════════════════════════════════════════════════════════╗
║  场景 B：压缩数据中间截断（服务器断连 / TCP 传输丢包 / 截断在 PAYLOAD 区）    ║
╚══════════════════════════════════════════════════════════════════════════════╝

真实事件序列（实验验证）：
 T0 response.push(body) → pipe → gunzip.write(截断的压缩块)
    Z_SYNC_FLUSH 生效: gunzip 不等后续字节，立即把能解压的部分 flush 出去
      → emit("data", chunk1) → data.push("hello world, thi")   // 部分解压成功
 T1 response.push(null) → response emit("end")
    pipe(end:true 默认) → gunzip.end() → gunzip._flush()
    Z_SYNC_FLUSH 模式下 _flush 不校验完整性，把剩余缓冲区输出
      → emit("finish") → emit("end")                           // 没有任何 error！
 T2 gunzip 自身清理 → emit("close")

Promise 状态：✅ 正常 resolve
data 内容：部分解压成功的残缺字符串（如 "hello world, thi"，原始35B只拿到16B）
error 事件：❌ **完全不触发**（Z_SYNC_FLUSH 让截断被视为合法结束）
上层后果：⚠️ resolve 残缺内容 → 上层 JSON.parse 失败 → 被 validate-widget-data 或业务层 error 处理捕获


╔══════════════════════════════════════════════════════════════════════════════╗
║  场景 C：压缩内容完整但 trailer 校验失败（CRC32 / ISIZE 被篡改 / 损坏）       ║
╚══════════════════════════════════════════════════════════════════════════════╝

真实事件序列（实验验证 + 代码真实投递方式 = 同步一次性 push）：
 T0 response.push(body) + push(null) → pipe → gunzip.write(完整压缩块+假trailer)
    _transform 处理全部压缩数据 → 因同步一次性 push，在同一 tick 内完成
    gunzip._flush(): 校验最后 8 字节 trailer（CRC32+ISIZE）失败
      → zlib Z_DATA_ERROR → this.destroy(new Error("incorrect data check"))
      → destroy 内部: emit("unpipe") → emit("error", e) → emit("close")
      → _flush 被 destroy 中断，end 不会被 emit
 T1 error 监听执行: logger.error(e) + 无效赋值

Promise 状态：⏳ 永远 pending  ❌
data 内容：空数组 [] （同步投递下 data 还没来得及 emit 就被 destroy）

─── 但如果是真实网络（分 chunk 异步投递），结果不同 ───
 F0 push(chunk0) → gunzip._transform 成功 → emit data("he")
 F1 push(chunk1) → gunzip._transform 成功 → emit data("llo world, th")
 ... （解压出多块数据，累计 28/35B）
 Fn push(null) → _flush 校验 trailer 失败 → destroy(err)
    → emit unpipe → emit error → emit close
    → end 不 emit

分 chunk 下 Promise：⏳ 永远 pending  ❌
分 chunk 下 data：部分解压成功（累计 28B，最后 7B 丢失），但永远到不了 resolve
```

##### 第三组：汇总决策表

| 失败类型 | 具体情形 | error? | end? | close? | Promise | resolve 出的 data | 代码上层感知 |
|---------|---------|--------|------|--------|---------|-------------------|-------------|
| **A 魔数错** | body 不是 gzip（纯文本 / html / 空） | ✅ | ❌ | ✅ | ⏳ pending | -（空数组但永远不concat） | 无感知，一直 loading，悬挂 |
| **B 截断在PAYLOAD** | 服务器中途断连丢包 | ❌ | ✅ | ✅ | ✅ resolve | ⚠️ 残缺解压内容（如16/35B） | `JSON.parse` 抛错 → 被 try-catch 捕获 → `{error:{...}}` 包给前端 |
| **B 截断在TRAILER** | 刚好丢了CRC校验尾（但压缩payload完整） | ❌ | ✅ | ✅ | ✅ resolve | ✅ 完整解压内容（35/35B） | **完全静默忽略校验失败**，用户无感 |
| **C CRC32篡改** + 同步投递 | 传输比特翻转 / 攻击者改trailer | ✅ | ❌ | ✅ | ⏳ pending | -（同步下data为0，不concat） | 无感知，悬挂 |
| **C CRC32篡改** + 分chunk投递 | 真实TCP分片到达 | ✅ | ❌ | ✅ | ⏳ pending | data里有部分解压（但永不concat） | 无感知，悬挂 + **已解压部分内存泄漏** |
| **C ISIZE篡改** | 同上 | ✅ | ❌ | ✅ | ⏳ pending | 同上 | 同上 |
| **基准对照：正确gzip** | - | ❌ | ✅ | ✅ | ✅ resolve | ✅ 完整解压内容 | 正常显示 |

**概率评估（生产环境）**：**约 3/4 的失败类型会导致 Promise 永久 pending**，只有截断类（B 族，约 2/7 的情形）碰巧 resolve。B 族中还分"静默丢内容"和"完全忽略校验失败"两种都极其危险的情况。

#### 测试替身 PassThrough 如何遮蔽问题
[utils/proxy/http.test.js#L367-L398](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/http.test.js#L367-L398) 中的测试替身：

```javascript
createUnzip: () => {
  const pt = new PassThrough();     // ❌ PassThrough 的行为与真实 Gunzip 有 5 个关键分歧
  pt.on("pipe", () => {
    queueMicrotask(() => {
      pt.emit("error", new Error("bad gzip"));
      pt.end();                      // ❌ 手动调 end()，真实 zlib error 后 end 不 emit
    });
  });
  return pt;
}
await httpMod.httpProxy("http://example.com");   // await 能返回说明 Promise resolve 了
expect(logger.error).toHaveBeenCalled();         // ❌ 只断言日志，没断言 data 内容和 Promise 结果
```

**5 个维度的遮蔽效应：**

| 行为维度 | PassThrough 替身 | 真实 zlib Gunzip（A/C 失败场景） | 被遮蔽的问题 |
|----------|-----------------|-------------------------------|-------------|
| ① error 后是否 emit `end` | **手动 `pt.end()` 保证一定 emit** | ❌ 永远不 emit（destroy 后 _flush 跳过） | **Promise 悬挂 bug**（测试下 await 能返回，生产 3/4 场景永久 pending） |
| ② error 后手动 `.end()` 救回？ | PassThrough 能 emit end | ❌ 完全无效（已 destroy 的流 end() 是 no-op） | **"错误后可恢复"的错觉**（实验 E 验证：手动 end 无效，end 仍不触发） |
| ③ B 场景截断是否 emit error | 由测试代码主动 emit | ❌ **Z_SYNC_FLUSH 下 B 截断完全不抛 error** | **"截断=错误"的假设与真实相反**（代码的 error 监听根本抓不到截断） |
| ④ data 内容完整性 | PassThrough 原样透传 body | A/C: data=0（同步投递）或残缺；B: 部分解压 | **数据丢失 / 残缺问题**被透传掩盖 |
| ⑤ mock body 投递方式 | `Readable.read()` 同步一次性 push+null，Z_SYNC_FLUSH 的分块行为不触发 | 真实响应是异步分 chunk 到达，影响 data 数量和 destroy 时机 | **时序敏感性**被遮蔽：C 场景同步投递 data=0，异步投递 data 有内容 |

**为什么误导严重？** 测试断言 `logger.error` 被调用 + `await` 正常返回，给开发者"fallback 工作正常"的强信心。但真实情况是：**截断不抛 error（B 族，监听永远不触发）、头错/校验错不抛 end（Promise 悬挂）**，测试结论与生产行为几乎完全相反。

#### 实际后果总览 + 恢复时机

| 场景族 | Promise 状态 | 真实发生频率 | 对用户的可见影响 | 恢复时机 |
|--------|-------------|-------------|-----------------|---------|
| A 魔数错 | ⏳ pending（悬挂） | 不常见（后端配置错误导致返回 html 但标 gzip） | SWR 一直 loading → 用户刷新无果 → 该 widget 永久骨架屏 | SWR 可能有内部超时；或下一次 `refreshInterval` 重新请求（新 Promise，与悬挂的无关） |
| B 截断 PAYLOAD | ✅ resolve（残缺） | 偶发（弱网、服务器重启、LB 摘除） | 残缺 JSON → parse 错 → 业务层 error → 前端显示错误面板 | 下一次请求成功即恢复（SWR 重试 / 轮询） |
| B 截断 TRAILER | ✅ resolve（完整但忽略校验） | 非常罕见 | **完全无感知**（数据完整性受损，但校验被静默跳过） | 无需恢复；如果数据本身损坏，widget 渲染异常需用户察觉 |
| C CRC / ISIZE 错 | ⏳ pending（悬挂 + 可能内存泄漏） | 极罕见（传输比特错 / 磁盘坏块 / 攻击） | 同 A，永久骨架屏 | 同 A |

**悬挂场景的泄漏面**：未 resolve 的 Promise 持有闭包（`data` 数组、`url`、`params`）→ 整个 `handleRequest` 作用域无法 GC；TCP socket 在 Node.js 超时前也无法正常回收。`end` 事件永不触发 → `addCookieToJar` 也不会被调用 → 重定向 Cookie 可能丢失（但悬挂场景下请求本身也没结束）。

#### 前端 SWR 侧：Promise 挂起后的自动恢复机制分析

服务端 `handleRequest` Promise 挂起后，前端 SWR（v2.4.1）是否能自动发起新请求、是否有超时退出、`refreshInterval` 是否会被卡住，取决于以下 5 个机制的交互。

##### 配置总览：代码中实际生效的 SWR 参数

**全局层（`_app.jsx` + `index.jsx` 双重 SWRConfig）**
- [\_app.jsx#L75-L79](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/_app.jsx#L75-L79)：全局 fetcher = `fetch(resource, init).then(res => res.json())`
- [index.jsx#L186](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx#L186)：补充 SSR fallback 数据
- **注意：全局完全没有配置 `timeout` / `dedupingInterval` / `onErrorRetry` / `errorRetryInterval`** — 全部使用 SWR v2.4.1 的默认值

**Hook 层（useWidgetAPI，每个 widget 调用）**
- [use-widget-api.js#L6-L14](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/use-widget-api.js#L6-L14)：仅注入可选的 `refreshInterval`（由各 widget 提供，范围 1500ms ~ 300000ms，无配置则 undefined）
- SWR key 格式：`/api/services/proxy?group=xxx&service=xxx&index=n&endpoint=yyy`（见 [api-helpers.js#L43-L49](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/utils/proxy/api-helpers.js#L43-L49)）

**SWR v2.4.1 默认关键参数（对照代码未覆盖的部分）**

| 参数 | 默认值 | 对悬挂 Promise 的影响 |
|------|--------|----------------------|
| `dedupingInterval` | **2000 ms** | 同一 key 2s 内重复 hook 挂载 → 复用同一个 pending fetcher，**不会发新请求** |
| `onErrorRetry` | 内置指数退避策略（5次，5000ms 上限） | ⚠️ **仅在 fetcher throw / Promise reject 时触发**，对既不 resolve 也不 reject 的 pending Promise **完全无效** |
| `errorRetryInterval` | 指数退避 `min(~~(5000 * (Math.random() + 1)), 5000)` 起 | 同上，仅对已确定是 error 的情况生效 |
| `focusThrottleInterval` | **5000 ms** | 窗口聚焦触发重验证的节流 |
| `loadingTimeout` | **undefined** | 无内置 loading 超时转 error 的机制 |
| `keepPreviousData` | false | key 切换时不保留旧 data |
| **fetch API（浏览器内置）** | 无默认超时（仅受操作系统 TCP keepalive 限制，通常分钟级 ~ 小时级） | 悬挂的 fetch 请求将在 TCP 层面长时间不结束 |

##### 5 个恢复机制逐个分析

```
═══════════════════════════════════════════════════════════════════════════
机制 1：onErrorRetry 自动重试
═══════════════════════════════════════════════════════════════════════════
前提条件：SWR 观察到 fetcher Promise reject（或 throw）
问题    ：悬挂 Promise 既不 resolve 也不 reject → 永远不满足前提
结论    ：❌ 完全无效。悬挂场景下 onErrorRetry 永远不会被调用，
          因为它的触发点是 Promise.then 的 rejected 分支

═══════════════════════════════════════════════════════════════════════════
机制 2：refreshInterval 周期轮询（各 widget 配置 1.5s ~ 5min）
═══════════════════════════════════════════════════════════════════════════
SWR v2 内部实现：softRevalidate → 检查是否有 inFlightRequest
  if (CONCURRENT_PENDING 请求存在) → return，不发起新 fetcher
  else → startRequest（启动新 fetcher）

关键代码逻辑（SWR 内部）：
  const tick = () => {
    if (!getCache().keyMatch(key, cache)) return clearInterval(timer)
    if (!stateRef.current.isLoading) {    // ← 关键判断
      // 只有 !isLoading 才会发新请求
      softRevalidate({ dedupe: false, ... })
    }
  }
  timer = setInterval(tick, intervalMs)

问题    ：Promise 悬挂期间 isLoading === true（因为有 in-flight fetcher）
          → refreshInterval 每次 tick 都被跳过
结论    ：❌ **在 Promise 挂起期间，refreshInterval 完全不会触发新请求**
          直到当前悬挂的 Promise 被外部方式结束（网络超时断开 / 页面关闭）

═══════════════════════════════════════════════════════════════════════════
机制 3：dedupingInterval（默认 2000ms）
═══════════════════════════════════════════════════════════════════════════
SWR v2 去重逻辑：
  new hook(key, fn) 挂载时：
    if (Map[key] 存在正在执行的 fetcher && 启动 < 2000ms 前)
      → 复用该 Promise，不发新
    else
      → 启动新 fetcher，覆盖旧 Map 条目

两种分情况：
  ① 页面刚加载 2s 内进入悬挂：
     → 后续同 key 的其他 hook（如果有）会复用悬挂 Promise
     → 2s 后如果 Promise 仍 pending：超出 dedupingInterval 窗口
        → 新 hook 挂载会认为过期 → 启动新 fetcher
        → ✅ 2秒后会有一次自动发新请求的机会！
  ② 加载完毕 2s 后才进入悬挂（如 refreshInterval 发起的某次轮询）：
     → 去重窗口已经过去，下次该 widget re-render（或其他 hook 变化）
        → 若重新调用 useSWR → 认为没有有效 in-flight → 启动新 fetcher
        → ✅ 理论上也能触发新请求（但取决于 React 渲染节奏）

额外关键问题：SWR 的 Map 里挂着的悬挂 Promise 会被手动 abort 吗？
  → SWR v2 默认**不提供 AbortController 集成**，代码也没传 signal
  → 旧的悬挂 Promise 永远在跑，只是不再被新 hook 引用
  → 内存泄漏 + TCP 连接泄漏持续存在

结论    ：⚠️ 部分有效。dedupingInterval 过期后（2s + α），
          若触发了新的 useSWR 调用（re-render），可能会启动新请求；
          但同时旧的悬挂 Promise 仍在后台继续运行不退出。

═══════════════════════════════════════════════════════════════════════════
机制 4：浏览器 fetch / TCP 层超时
═══════════════════════════════════════════════════════════════════════════
代码中的 fetcher：fetch(url)，无任何 AbortController / signal / timeout

浏览器行为：
  fetch 本身**不设超时**。HTTP 请求在以下情况才会被浏览器终止：
    ① TCP 连接被 RST 包打断（对端重启 / 防火墙强制断开）
    ② 操作系统 TCP keepalive 超时（默认 Windows 2h，Linux 数小时）
    ③ 浏览器标签页内存/性能保护强制终止（罕见，且不可控）

反向代理 / Next.js Node.js 服务端：
  → Next.js 自身 HTTP 服务器通常有默认超时（Node http 默认无，但 Next 可能设置 30s）
  → 但 handleRequest 的 Promise 挂起在 zlib 流的 end 事件上，
     即使 socket 被服务端关闭，zlib 的 Transform 也不一定 emit end
     （取决于 socket close 时 gunzip 是否走了正常 finish 路径）

结论    ：❌ 在分钟~小时级别的时间尺度内，基本可以视为"永不超时"。
          只有极端网络条件下才会被动终止，且终止时是否 reject / resolve
          取决于具体路径，不可靠。

═══════════════════════════════════════════════════════════════════════════
机制 5：mutate() / 窗口聚焦 / SWR 手动重验证
═══════════════════════════════════════════════════════════════════════════
a) mutate(key, newData, { revalidate: true })
   → mutate 强制启动新 fetcher，**不检查 isLoading**（除非 dedupe）
   → 但若前一个 Promise 2s 内挂起且仍在 deduping 窗口，仍可能命中去重
   → 实测：mutate(key) 会取消去重（dedupe=false 默认），✅ 能强制发起新请求

b) 窗口聚焦（focus/reconnect）
   → SWR 内置 focusThrottleInterval=5000ms 的节流
   → 触发的 softRevalidate 同样检查 isLoading
   → ❌ isLoading=true（悬挂中）时不发新请求

c) localStorage hash 变更 → 强制 reload
   → [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/209-homepage/src/pages/index.jsx)
   中 useWindowFocus 监控 hash，不一致时 setStale → 调用 /api/revalidate
   → 最终 `window.location.reload()` 整页刷新
   → ✅ 能完全恢复（但这是配置变更触发的 reload，非 gzip 错误专属）

结论    ：✅ mutate() 能强制恢复；整页 reload 能完全恢复。
          其他聚焦 / 重连自动机制与 refreshInterval 同命运，被 isLoading 卡住。
```

##### 结论决策矩阵：A / C 悬挂场景下 SWR 侧真实行为

| 时间段 | 恢复机制 | 是否生效 | 用户看到什么 |
|--------|---------|:---:|-------------|
| 0 ~ 2s | dedupingInterval 窗口内 | ❌ 卡住 | widget 骨架屏 `animate-pulse` + `"-"` 占位 |
| 2s 时 | dedupingInterval 过期 | ⚠️ 条件生效 | 如果 widget 恰好 re-render → 触发全新 fetcher；否则仍卡住 |
| 2s ~ 无限 | refreshInterval | ❌ 被 isLoading 拦住 | 骨架屏持续，轮询全部跳过，**没有新请求发出** |
| 任意时刻 | 用户点击 / 路由切换 → 组件卸载重挂载 | ✅ 生效 | 组件卸载 → SWR 卸载该 key hook → 再挂载 → 全新 fetcher 启动 |
| 任意时刻 | 窗口聚焦 + 配置 hash 变更 | ✅ 生效（整页 reload） | 显示 stale 加载动画 → `/api/revalidate` → 页面整体刷新 |
| 任意时刻 | 用户手动 F5 刷新 | ✅ 生效 | 完整恢复 |
| 分钟/小时级 | TCP 超时断开 | ⚠️ 不可靠 | 如果 socket 断开时 Promise 碰巧 resolve/reject → SWR 进入下一状态；否则继续挂 |
| 永久悬挂 | 内存/连接泄漏 | - | 用户无感知，但浏览器 tab 内存持续增长，Node 服务端句柄泄漏 |

##### 最终结论：能不能自动恢复？

**短答案：在没有人工干预的情况下，概率 ≈ 很低。**

只有以下几种会**自动**触发新请求的路径：
1. **2s 内 + 组件重渲染**（概率低，需要 React 因其他 state 变更而重渲染该 widget）
2. **用户交互导致组件卸载重挂载**（tab 切换 / 展开收起等，视产品行为而定）
3. **配置 hash 变更导致整页 reload**（这是 YAML 文件修改触发的，与 gzip 错误无关）

**不会自动恢复的情况：**
- `refreshInterval` 轮询（被 `isLoading === true` 挡住）
- `onErrorRetry` 重试（永远不触发，因为 Promise 没 reject）
- 窗口聚焦 / 网络重连（同 refreshInterval，被 isLoading 卡住）
- 浏览器 fetch 超时（分钟~小时级以上才可能发生，且不保证能 resolve/reject）

**悬挂请求的泄漏情况：**
- 旧的悬挂 Promise 在 SWR cache 里，2s deduping 后就不再被引用，但自身闭包还持有 → **`handleRequest` 作用域内内存泄漏**
- 对应的 HTTP 连接：服务端 gunzip 流没结束，TCP socket 没关闭 → **Node 服务端句柄泄漏**
- 这些泄漏在整页 reload / tab 关闭之前不会被清理

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
