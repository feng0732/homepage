# 容器镜像与运行环境 — 代码链路全解析

本文档从代码层面逐层拆解 Homepage 项目中「容器镜像来源 → 环境变量注入 → 容器展示信息 → 部署差异」这四条链路如何协作，帮助读者建立完整的认知地图。

---

## 一、镜像来源：从源码到可运行镜像

### 1.1 生产镜像 — Dockerfile（多阶段构建）

[Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/Dockerfile) 采用经典的 **builder / runner 两阶段构建**：

```
builder 阶段 (node:22-slim)
  ├─ 接收 ARG: CI, BUILDTIME, VERSION, REVISION
  ├─ 若 CI != "true" → 安装 pnpm → 执行 pnpm install + pnpm build
  │   └─ 构建时通过 NEXT_PUBLIC_BUILDTIME / VERSION / REVISION 注入前端变量
  └─ 若 CI == "true" → 跳过构建（CI 上下文已预构建产物）

runner 阶段 (node:22-alpine)
  ├─ OCI LABEL 元数据（title / description / url / source / licenses）
  ├─ COPY --from=builder .next/standalone → 产出精简的 Next.js standalone 输出
  ├─ COPY --from=builder .next/static     → 静态资源
  ├─ COPY docker-entrypoint.sh            → 入口脚本
  ├─ apk add su-exec iputils-ping shadow  → 运行时工具
  ├─ ENV NODE_ENV=production HOSTNAME=:: PORT=3000
  ├─ HEALTHCHECK → wget http://127.0.0.1:$PORT/api/healthcheck
  ├─ ENTRYPOINT ["docker-entrypoint.sh"]
  └─ CMD ["node", "server.js"]
```

**关键点：**
- [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/next.config.js#L6) 中 `output: "standalone"` 是 runner 阶段能够仅复制 `.next/standalone` 就可运行的前提。
- 构建阶段通过 `ARG` + `ENV` 把 `CI` 传递进去，用条件判断决定是否执行完整构建。CI 环境下构建已在 workflow 中完成，Dockerfile 内跳过，只做 `COPY --from=builder`。

### 1.2 CI 镜像构建 — docker-publish.yml

[.github/workflows/docker-publish.yml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/.github/workflows/docker-publish.yml) 是正式镜像发布的唯一入口：

```
1. Checkout → 提取 Docker metadata（标签策略）
   └─ 标签规则：nightly / 分支名 / semver(vX.Y.Z) / latest

2. 本地 pnpm install + pnpm build
   └─ 在 workflow 中先构建前端产物，注入：
      NEXT_PUBLIC_BUILDTIME ← org.opencontainers.image.created
      NEXT_PUBLIC_VERSION   ← org.opencontainers.image.version
      NEXT_PUBLIC_REVISION  ← org.opencontainers.image.revision

3. docker/build-push-action
   └─ build-args: CI=true, BUILDTIME, VERSION, REVISION
      → Dockerfile 内 CI=true，跳过第二次构建，直接使用已有产物
   └─ 推送到 Docker Hub + ghcr.io
   └─ 多平台: linux/amd64, linux/arm64
```

**协作要点：** CI workflow 在 **workflow 层**先完成 `pnpm build`（此时 `NEXT_PUBLIC_*` 变量被 Next.js 内联到客户端 JS 中），然后通过 `CI=true` 构建参数告诉 Dockerfile 不再重复构建，直接打包已有产物。这意味着 `NEXT_PUBLIC_*` 的值在 workflow 执行时就已经烧录进了 `.next` 目录，之后 runner 阶段只是搬运。

### 1.3 开发镜像 — Dockerfile-tilt

[Dockerfile-tilt](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/Dockerfile-tilt) 面向 K8s 本地开发（Tilt 工具链），与生产镜像差异显著：

| 特性 | 生产 Dockerfile | Dockerfile-tilt |
|---|---|---|
| 基础镜像 | node:22-slim → node:22-alpine | node:18-alpine |
| 构建方式 | 多阶段，standalone 输出 | 单阶段，完整源码 |
| 启动命令 | `node server.js` | `npx next dev` |
| 构建时变量 | BUILDTIME/VERSION/REVISION | node_env=development |
| 目标 | 精简运行时 | 热重载开发 |

[k3d/Tiltfile](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/Tiltfile) 中 `docker_build('k3d-registry.localhost:55000/homepage:local', '..', dockerfile="../Dockerfile-tilt")` 使用本地 k3d registry 构建并推送，配合 `live_update` 实现文件同步热更新。

---

## 二、环境变量注入：三层注入机制

Homepage 的环境变量注入不是简单的 `docker run -e`，而是横跨 **Dockerfile 构建期 → entrypoint 运行期 → 应用配置期** 三个阶段，每层职责不同。

### 2.1 构建期注入（烧录到前端 JS）

在 Dockerfile 构建阶段和 CI workflow 中：

```
ARG BUILDTIME / VERSION / REVISION
  ↓
NEXT_PUBLIC_BUILDTIME=$BUILDTIME
NEXT_PUBLIC_VERSION=$VERSION
NEXT_PUBLIC_REVISION=$REVISION
  ↓
pnpm run build（Next.js 将 NEXT_PUBLIC_* 内联到客户端 bundle）
```

消费端：[src/components/version.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/version.jsx#L12-L16)

```js
const buildTime = process.env.NEXT_PUBLIC_BUILDTIME?.length
  ? process.env.NEXT_PUBLIC_BUILDTIME
  : new Date().toISOString();
const revision = process.env.NEXT_PUBLIC_REVISION?.length ? process.env.NEXT_PUBLIC_REVISION : "dev";
const version = process.env.NEXT_PUBLIC_VERSION?.length ? process.env.NEXT_PUBLIC_VERSION : "dev";
```

- 如果构建时未提供（如 `pnpm dev` 本地开发），fallback 到 `"dev"` 和当前时间。
- 前端页面底部 [Version](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/version.jsx#L41) 组件根据 version 值判断显示格式：`main`/`dev`/`nightly` 纯文本，正式版则链接到 GitHub Release。
- Version 组件还会通过 `/api/releases` 接口（[src/pages/api/releases.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/releases.js)）查询 GitHub 最新 Release，对比版本号显示「Update Available」提示。

### 2.2 运行期注入（entrypoint 脚本处理）

[docker-entrypoint.sh](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/docker-entrypoint.sh) 在容器启动时处理以下环境变量：

| 变量 | 默认值 | 作用 |
|---|---|---|
| `PUID` / `PGID` | `0` (root) | 非 root 运行：chown `/app/config` 和 `/app/.next`，然后 `su-exec` 降权 |
| `HOSTNAME` | `::` (IPv6 双栈) | 探测 IPv6 绑定是否可用，失败则回退 `0.0.0.0` |
| `HOMEPAGE_BUILDTIME` | `date +%s` | **容器启动级的「虚拟构建时间」，与配置哈希和页面刷新强相关**（详见 2.3 节） |
| `HOMEPAGE_CONFIG_DIR` | `/app/config` | 配置文件目录，见下文 |

entrypoint 的完整执行流：

```
1. 设置 PUID/PGID 默认值
2. 若 /app/config 不存在 → ln -s /config /app/config（兼容 lscr.io 路径）
3. export HOMEPAGE_BUILDTIME=$(date +%s)   ← 每次容器启动时重新生成
4. IPv6 探测 → 可能回退 HOSTNAME=0.0.0.0
5. 根据 PUID 调整 /app/config 和 /app/config/logs 的 ownership
6. 调整 /app/.next 的 ownership
7. 若 PUID != 0 且当前为 root → exec su-exec PUID:PGID 降权
8. 否则直接 exec "$@"（即 node server.js）
```

**`HOMEPAGE_BUILDTIME` 的关键作用：**
这个变量是 entrypoint 在每次容器启动时用 `date +%s` 动态生成的 Unix 时间戳。它并非传给前端展示（展示用的是 `NEXT_PUBLIC_BUILDTIME`，在 CI 构建时已烧录），而是与配置文件内容一起参与 `/api/hash` 接口的哈希计算，从而实现「容器重启/重新创建 → 即使配置文件未改动 → 前端也能感知并触发全页刷新」的机制（详见 2.3 节完整协作链路）。

### 2.3 配置刷新与前端重新校验的协作链路（HOMEPAGE_BUILDTIME → /api/hash → 前端刷新）

这是运行时环境变量影响页面状态最核心的协作机制，横跨入口脚本、后端 API 和前端 React 组件三层。

#### 2.3.1 入口脚本：`HOMEPAGE_BUILDTIME` 的生成

[docker-entrypoint.sh 第 13 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/docker-entrypoint.sh#L13)：

```sh
export HOMEPAGE_BUILDTIME=$(date +%s)
```

- 每次容器启动（`docker start` / `docker restart` / 容器 recreate）都会生成一个**新的**时间戳。
- `docker exec` 进入容器不会触发，因为不会重新执行 entrypoint。
- 该变量通过 `export` 注入到 node server.js 的运行时环境中。

#### 2.3.2 `/api/hash` 接口：哈希的计算与返回

[src/pages/api/hash.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/hash.js)：

```js
const configs = [
  "docker.yaml", "settings.yaml", "services.yaml",
  "bookmarks.yaml", "widgets.yaml", "custom.css", "custom.js",
];

export default async function handler(req, res) {
  // 1. 对每个配置文件计算 sha256
  const hashes = configs.map((config) => {
    checkAndCopyConfig(config);
    const configYaml = join(CONF_DIR, config);
    return hash(readFileSync(configYaml, "utf8"));
  });

  // 2. 拼接所有文件哈希 + HOMEPAGE_BUILDTIME，再计算一次哈希
  const buildTime = process.env.HOMEPAGE_BUILDTIME?.length ? process.env.HOMEPAGE_BUILDTIME : "";
  const combinedHash = hash(hashes.join("") + buildTime);

  res.send({ hash: combinedHash });
}
```

关键逻辑在第 31-33 行：
- `hashes.join("")` 是所有配置文件内容的哈希拼接，代表「配置内容是否改变」。
- 加上 `buildTime`（即 `HOMEPAGE_BUILDTIME`）后再哈希，代表「即使配置文件没变，只要容器重启过，也视为整体状态变化」。
- 源码直接运行时（无 entrypoint），`HOMEPAGE_BUILDTIME` 为空，哈希仅反映配置文件变化。

这意味着 **两种场景下 `/api/hash` 的返回值都会变化**：
1. 用户编辑了 docker.yaml / services.yaml 等配置文件 → 配置文件哈希变化。
2. 容器被重启或重新创建 → `HOMEPAGE_BUILDTIME` 变化。

#### 2.3.3 组件层级与 SWR 缓存边界：hash 请求为何不读 fallback

这是理解配置刷新机制最关键但最易混淆的一点。需要从组件树的层级关系说起。

**三层 SWRConfig 嵌套结构**（由外到内）：

```
_app.jsx: <SWRConfig value={{ fetcher }}>          ← 第 1 层：注册全局 fetcher
  └─ <Wrapper>
       └─ <Index>                                   ← ⚠️ hash/validate 请求在这里
            ├─ useSWR("/api/hash")
            ├─ useSWR("/api/validate")
            └─ <SWRConfig value={{ fallback }}>     ← 第 2 层：注入 fallback 缓存
                 └─ <Home>
                      ├─ useSWR("/api/services")     ← ✅ 能读到 fallback
                      ├─ useSWR("/api/bookmarks")    ← ✅ 能读到 fallback
                      └─ useSWR("/api/widgets")      ← ✅ 能读到 fallback
```

[_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/_app.jsx#L73-L97) 在最外层注册全局 `fetcher`，所有 `useSWR` 调用共享此 fetcher。

[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx) 中：
- `Index` 组件（[第 97 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx#L97)）在 `SWRConfig fallback` 的**外层**，直接调用 `useSWR("/api/hash")` 和 `useSWR("/api/validate")`。
- `SWRConfig fallback`（[第 186 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx#L186)）包裹的是 `Home` 组件及其子树。

**SWR fallback 的工作原理：** `SWRConfig` 的 `fallback` 值只对其**子组件树**中的 `useSWR` 调用生效。`Index` 组件在 `SWRConfig fallback` 的**外层**，所以 `useSWR("/api/hash")` 和 `useSWR("/api/validate")` **完全无法读取** fallback 缓存，它们永远会发起真实的网络请求。

**`"/api/hash": false` 的真正作用：**

```js
fallback: {
  "/api/services": services,    // Home 子树：直接使用缓存数据，不发起网络请求
  "/api/bookmarks": bookmarks,
  "/api/widgets": widgets,
  "/api/hash": false,           // ← 这行的作用对象不是 Index 中的 hash 请求
}
```

这个 `false` 不是给 `Index` 组件中的 `useSWR("/api/hash")` 用的（它根本读不到这个 fallback），而是给 `Home` 子树中可能存在的其他 `useSWR("/api/hash")` 调用的。设为 `false` 意味着：如果在 `Home` 子树中有人调用 `useSWR("/api/hash")`，SWR 会把初始数据设为 `false`（falsy），从而**阻止**子树从 SWR 缓存中拿到一个「看起来有效」的旧哈希值，强制子树也必须等待真实请求返回。

总结：
- `Index` 中的 hash/validate 请求**始终发起真实网络请求**，因为它们在 `SWRConfig fallback` 之外，读不到 fallback。
- `Home` 中的 services/bookmarks/widgets 请求**首屏先用 fallback 数据渲染**，组件挂载后 SWR 还会在后台自动重新请求（`revalidateOnMount` 默认 `true`），实现「即显即更」。
- `"/api/hash": false` 是一个**防御性配置**，确保 Home 子树内部不会意外使用哈希缓存。

#### 2.3.4 前端：Window Focus 触发的哈希校验与全页刷新

理解了组件层级后，`Index` 组件中的完整检测-刷新链路如下（[index.jsx 第 97-131 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx#L97-L131)）：

```js
function Index({ initialSettings, fallback }) {
  const windowFocused = useWindowFocus();
  const [stale, setStale] = useState(false);

  // ① 两个 SWR 请求都在 SWRConfig fallback 外层，必定发起真实网络请求
  const { data: errorsData } = useSWR("/api/validate");
  const { error: validateError } = errorsData || {};
  const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

  // ② 窗口焦点变化 → 强制重新请求 /api/hash
  useEffect(() => {
    if (windowFocused) {
      mutateHash();   // bypass SWR 缓存，发起新的 fetch
    }
  }, [windowFocused, mutateHash]);

  // ③ hashData 变化 → 比对 localStorage，决定是否刷新
  useEffect(() => {
    if (hashData) {
      const previousHash = localStorage.getItem("hash");
      if (!previousHash) {
        localStorage.setItem("hash", hashData.hash);  // 首次：记录基准
      }
      if (previousHash && previousHash !== hashData.hash) {
        setStale(true);                                // 显示旋转加载动画
        localStorage.setItem("hash", hashData.hash);   // 更新基准
        fetch("/api/revalidate").then((res) => {
          if (res.ok) {
            window.location.reload();                   // 全页硬刷新
          }
        });
      }
    }
  }, [hashData]);
```

`useWindowFocus` 钩子（[src/utils/hooks/window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/hooks/window-focus.js)）监听 `window.focus` 和 `window.blur` 事件。用户切换标签页再切回来时，自动触发 `mutateHash()` 重新请求。

注意 `mutateHash()` 与普通 SWR revalidation 的区别：SWR 默认的 `useSWR` 也会在窗口 focus 时重新校验（如果开启了 `revalidateOnFocus`，默认为 true），但 `mutateHash()` 是**显式调用**，**无条件**触发一次新的 fetch，不依赖 SWR 内置的 focus revalidation 机制。这确保即使 SWR 因为某种策略跳过了自动 revalidation，哈希校验也一定会执行。

#### 2.3.5 配置刷新的完整请求与状态依赖链

一次完整的配置刷新涉及 **3 个 API 请求** 和 **3 个状态变化**：

**请求清单：**

| 请求 | 发起位置 | 触发条件 | 是否读 fallback |
|---|---|---|---|
| `GET /api/validate` | `Index` → `useSWR` | 页面加载时（SWR 挂载） | ❌ 在 fallback 外层 |
| `GET /api/hash` | `Index` → `useSWR` | 页面加载时 + window focus 时 | ❌ 在 fallback 外层 |
| `GET /api/revalidate` | `Index` → `fetch` | 仅当 hash 变化时 | N/A（裸 fetch，不走 SWR） |

**状态变化依赖链：**

```
window focus 事件
  │
  ├─ useWindowFocus() 返回 true
  │
  ├─ useEffect([windowFocused]) → mutateHash()
  │     │
  │     └─ 触发 GET /api/hash → 后端返回 { hash: "新值" }
  │           │
  │           └─ hashData 更新（SWR 内部状态）
  │                 │
  │                 └─ useEffect([hashData]) 执行
  │                       │
  │                       ├─ previousHash = localStorage.getItem("hash")
  │                       │
  │                       ├─ 若 previousHash 不存在 → 仅写入 localStorage（首次访问）
  │                       │
  │                       └─ 若 previousHash !== hashData.hash
  │                             │
  │                             ├─ ① setStale(true)     → React 重渲染，显示旋转加载动画
  │                             │                         Index 返回 <div className="animate-spin">
  │                             │                         而非 <SWRConfig><Home/></SWRConfig>
  │                             │
  │                             ├─ ② localStorage.setItem("hash", 新值)
  │                             │
  │                             └─ ③ fetch("/api/revalidate")
  │                                   │
  │                                   └─ 后端 res.revalidate("/") → ISR 重新执行 getStaticProps
  │                                         │
  │                                         └─ res.ok → window.location.reload()
  │                                               │
  │                                               └─ 浏览器硬刷新，重新加载整个页面
  │                                                   新页面包含新的 getStaticProps 结果
```

**关键细节：`setStale(true)` 会卸载整个 `Home` 组件树，但不会清空 SWR 缓存。**

当 `stale` 为 true 时，[index.jsx 第 151-157 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx#L151-L157) 返回旋转加载动画，**不再渲染** `<SWRConfig><Home/></SWRConfig>`。这意味着：
- Home 及其所有子组件（ServicesGroup、BookmarksGroup、Widget 等）全部**卸载**（React 组件树中移除）。
- 但 **SWR 全局缓存不会被清空**。SWR 的缓存储存在全局 cache 中，`SWRConfig` 组件的卸载**不会**触发缓存清除。`/api/services`、`/api/bookmarks`、`/api/widgets` 等数据仍然保留在内存缓存中。
- 然而，这些缓存只是暂时不可见，等待 `window.location.reload()` 后整个页面重新加载时，所有 JS 状态（包括 SWR 缓存）都会被彻底重置。

这个「先卸载再硬刷新」的设计保证了配置变更后，**不会出现旧数据和新配置混合渲染的中间状态**。同时也避免了在 revalidate 与 stale 数据共存的闪烁问题。

#### 2.3.5.1 四个环节的精确区分：页面隐藏、ISR 再生成、浏览器刷新、缓存清空

配置刷新流程中涉及四个容易混淆的环节，它们分别发生在不同层面，效果各异：

| 环节 | 发生位置 | 触发方式 | 实际效果 | 缓存是否被清空 |
|---|---|---|---|---|
| 页面内容隐藏 | 客户端 React | `setStale(true)` | Home 组件树卸载，显示旋转动画 | ❌ SWR 缓存保留，仅 UI 隐藏 |
| 静态页面重新生成 | 服务端 Next.js | `fetch("/api/revalidate")` → `res.revalidate("/")` | 服务端重新执行 `getStaticProps()`，生成新的静态 HTML 和 JSON | ❌ 与客户端缓存无关 |
| 浏览器刷新 | 浏览器 | `window.location.reload()` | 整页重载，所有 JS 运行时重建 | ✅ 所有 JS 状态（含 SWR 缓存）全部销毁重建 |
| 缓存显式清空 | — | 无 | — | ❌ 代码中没有显式调用 cache.clear() |

**逐环节详解：**

**① 页面内容隐藏（`setStale(true)`）**
- 属于客户端行为。
- React 组件层面上，`SWRConfig` 和 `Home` 从 DOM 中移除，但 SWR 的全局缓存仍然存在于内存中。
- 如果 `stale` 状态被撤销（实际流程中紧接着就是 reload，不会发生），组件重新挂载时会立即显示之前缓存的数据。
- 目的：在 revalidate 过程中不让用户看到新旧混合的数据，避免视觉闪烁。

**② 静态页面重新生成（`res.revalidate("/")`）**
- 纯服务端行为。
- Next.js ISR 机制重新运行 `getStaticProps()`，读取最新的配置文件，生成新的静态 HTML 和 JSON 数据。
- 与客户端浏览器状态完全无关，客户端只是发起一个 fetch 请求触发它。
- revalidate 完成后，**服务端的静态缓存会被替换为新内容**。
- 客户端如果不刷新页面，看到的仍然是旧的客户端渲染结果。

**③ 浏览器刷新（`window.location.reload()`）**
- 浏览器行为，销毁当前 document，重新请求 HTML 页面。
- 所有 React 组件状态、SWR 缓存、localStorage 之外的所有 JS 内存状态全部被销毁。
- 新页面加载后，SWR 重新从 `fallback` 初始化。
- 这是确保页面数据全新的最终手段。

**④ 缓存显式清空**
- 代码中**没有**任何地方显式调用 SWR 的 `cache.clear()` 或 `mutate(key, undefined)` 来主动清空缓存。
- 缓存的清空完全依赖 `window.location.reload()` 的副作用。

#### 2.3.5.2 `"/api/hash": false` 的实际影响

fallback 中 `"/api/hash": false` 的精确含义和作用经常被误解。

**SWR fallback 的工作原理：**
- `fallback` 是 `SWRConfig` 提供给其子组件中 `useSWR` 的**初始数据**。
- 当子组件中调用 `useSWR(key)` 时，如果 SWR 缓存中没有该 key 的数据，就使用 `fallback[key]` 作为初始值。
- `fallback` 只在 `useSWR` 首次渲染时生效，不会改变全局缓存状态。
- Index 组件在 `SWRConfig` **外部**，所以它的 `useSWR("/api/hash")` 完全不受 fallback 影响。

**`"/api/hash": false` 的精确作用：**

1. **对 Index 组件的 `useSWR("/api/hash")`**：**没有任何影响**。因为 Index 在 SWRConfig 外面，读不到这个 fallback。

2. **对 Home 子树内的潜在 hash 请求**：如果 Home 子树中有人调用 `useSWR("/api/hash")`，初始数据会是 `false`（falsy 值），而不是 `undefined`。这是一个**防御性配置**，确保子树不会因为拿不到数据而显示异常。

3. **语义上的作用**：明确地在配置中标注「hash 数据不使用 fallback，必须从网络请求获取」。

4. **测试断言**：测试用例（[index.test.jsx 第 195 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/__tests__/pages/index.test.jsx#L195)）中 `expect(result.props.fallback["/api/hash"]).toBe(false)` 验证了这个设计意图。

**实际效果**：Home 子树内**没有任何组件调用** `useSWR("/api/hash")`，所以这个 fallback 条目在运行时实际效果有限，更多是**设计意图的声明**和**防御性编程**。

#### 2.3.5.3 SWR 缓存的完整生命周期

结合两层 SWRConfig 的嵌套关系，SWR 缓存的生命周期如下：

```
页面加载
  │
  ├─ _app.jsx <SWRConfig value={{ fetcher }}>    ← 外层：注册全局 fetcher
  │   │
  │   └─ Index 组件挂载
  │        ├─ useSWR("/api/validate") → 无 fallback → 发起请求 ①
  │        ├─ useSWR("/api/hash")     → 无 fallback → 发起请求 ②
  │        │
  │        └─ 校验通过 → 渲染 <SWRConfig value={{ fallback }}>
  │             │
  │             └─ Home 组件挂载
  │                  ├─ 渲染阶段：立即使用 fallback 数据（不等待网络）
  │                  │   ├─ useSWR("/api/services")  → data = services (from fallback)
  │                  │   ├─ useSWR("/api/bookmarks") → data = bookmarks (from fallback)
  │                  │   └─ useSWR("/api/widgets")   → data = widgets (from fallback)
  │                  │
  │                  └─ 挂载后（revalidateOnMount 默认 true）：后台重新请求
  │                      ├─ 发起 GET /api/services   ③
  │                      ├─ 发起 GET /api/bookmarks  ④
  │                      └─ 发起 GET /api/widgets    ⑤
  │
  ▼
请求陆续返回
  │
  ├─ ③ services 返回 → 更新缓存 → data 更新 → 组件重渲染（静默刷新）
  ├─ ④ bookmarks 返回 → 更新缓存 → data 更新 → 组件重渲染（静默刷新）
  └─ ⑤ widgets 返回 → 更新缓存 → data 更新 → 组件重渲染（静默刷新）
  │
  ▼
window focus → mutateHash() → /api/hash 返回新值
  │
  ├─ hashData 变化 → 与 localStorage 对比 → 不一致
  │
  ├─ setStale(true)
  │   │
  │   └─ <SWRConfig><Home/></SWRConfig> 从 DOM 卸载
  │        │
  │        └─ ⚠️ 注意：SWR 全局缓存中的数据仍在内存中，不会被清空
  │
  ├─ fetch("/api/revalidate")
  │   └─ 服务端 ISR 重新生成静态页面
  │
  └─ window.location.reload()
       │
       └─ 浏览器卸载整个 document
            └─ 所有 JS 内存状态全部销毁（包括 SWR 缓存、React state、事件监听器等）
```

**核心结论：**
- SWR 缓存的生命周期 = 页面的生命周期。
- SWRConfig 的挂载/卸载不会清空缓存。
- 只有 `window.location.reload()` 或用户手动刷新页面，SWR 缓存才会被彻底重置。

#### 2.3.6 首屏加载时的请求顺序与缓存命中

用户首次访问或刷新页面时，请求的完整时序。关键要区分「首屏渲染用什么数据」和「挂载后是否重新请求」这两个独立的问题。

**SWR 默认策略（项目未显式覆盖，使用 SWR v2 默认值）：**

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `revalidateOnMount` | `true` | 组件挂载时，即使缓存中有数据，也会重新请求 |
| `revalidateOnFocus` | `true` | 窗口获得焦点时重新验证 |
| `revalidateOnReconnect` | `true` | 网络重连时重新验证 |
| `dedupingInterval` | `2000` ms | 2 秒内相同 key 的请求自动去重 |
| `refreshInterval` | `0`（禁用） | 不自动定时刷新 |

项目中两层 `SWRConfig` 都只设置了 `fetcher` 和 `fallback`，没有覆盖上述选项，所以全部使用 SWR v2 默认值。

**首屏完整时序：**

```
浏览器请求 GET /
  │
  ├─ Next.js SSR 渲染
  │   ├─ getStaticProps() 返回 { initialSettings, fallback }
  │   └─ HTML 包含内联的 fallback 数据（services/bookmarks/widgets 的 JSON）
  │
  ▼
浏览器接收 HTML → React Hydration 开始
  │
  ├─ 阶段 1：Index 组件挂载
  │   │
  │   ├─ useSWR("/api/validate")
  │   │   └─ 无 fallback（在 SWRConfig 外）→ 发起真实请求 ①
  │   │
  │   └─ useSWR("/api/hash")
  │       └─ 无 fallback（在 SWRConfig 外）→ 发起真实请求 ②
  │
  ├─ validateError 检查
  │   └─ 若 validate 请求返回错误 → 显示红色 Error 页面，不渲染 Home
  │
  ├─ errorsData 检查
  │   └─ 若 validate 返回 YAML 语法错误数组 → 显示琥珀色警告页面
  │
  ├─ 阶段 2：校验通过 → 渲染 <SWRConfig fallback={...}><Home/></SWRConfig>
  │   │
  │   └─ Home 组件挂载
  │        │
  │        ├─ 渲染阶段（同步，立即）：
  │        │   ├─ useSWR("/api/services")  → 命中 fallback → 初始 data = services  ✅ 立即有数据
  │        │   ├─ useSWR("/api/bookmarks") → 命中 fallback → 初始 data = bookmarks ✅ 立即有数据
  │        │   └─ useSWR("/api/widgets")   → 命中 fallback → 初始 data = widgets   ✅ 立即有数据
  │        │
  │        └─ 挂载后（异步，useEffect 中）：
  │            ├─ revalidateOnMount = true（默认）→ 发起 GET /api/services   ③
  │            ├─ revalidateOnMount = true（默认）→ 发起 GET /api/bookmarks  ④
  │            └─ revalidateOnMount = true（默认）→ 发起 GET /api/widgets    ⑤
  │
  ├─ 请求陆续返回
  │   ├─ validate ① 返回 → 更新 errorsData，如有错误切换到错误页面
  │   ├─ hash ② 返回 → localStorage 写入基准 hash
  │   ├─ services ③ 返回 → 更新缓存 → 重新渲染（静默刷新）
  │   ├─ bookmarks ④ 返回 → 更新缓存 → 重新渲染（静默刷新）
  │   └─ widgets ⑤ 返回 → 更新缓存 → 重新渲染（静默刷新）
  │
  └─ 所有请求完成 → 页面数据为最新
```

**关键结论：**

- **首屏渲染**：services/bookmarks/widgets **立即用 fallback 数据显示**，没有加载状态。
- **后台刷新**：组件挂载后，SWR 依据 `revalidateOnMount: true` 的默认策略，**在后台静默发起 3 个请求**刷新数据。
- **首屏总共 5 个真实网络请求**：`/api/validate` ①、`/api/hash` ②、`/api/services` ③、`/api/bookmarks` ④、`/api/widgets` ⑤。
- 其中 ③④⑤ 是「后台请求」，不影响首屏显示速度，但保证数据最终是最新的。

**为什么要设计成这样？**

fallback 的作用是**消除首屏加载白屏**，让用户一打开页面就能看到内容。而 `revalidateOnMount` 的作用是**保证数据新鲜度**——毕竟 getStaticProps 的数据是构建时（或上次 revalidate 时）生成的，可能已经过时了。两者配合实现「Stale-While-Revalidate」模式：先显示旧数据，后台刷新，有新数据就静默更新。

#### 2.3.7 `/api/revalidate` 接口：Next.js ISR 增量静态再生成

[src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/revalidate.js)：

```js
export default async function handler(req, res) {
  try {
    await res.revalidate("/");
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}
```

这个接口调用 Next.js 的 Incremental Static Regeneration (ISR) 机制，让服务端重新执行 `getStaticProps()`，读取最新的 services.yaml / bookmarks.yaml / widgets.yaml 并生成新的静态 HTML。然后前端 `window.location.reload()` 拉取重新生成的页面。

此外，[Revalidate 组件](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/toggles/revalidate.jsx) 在页面底部提供了一个手动刷新按钮，直接调用同一个 `/api/revalidate` 接口，允许用户主动触发 ISR 再生成。

#### 2.3.8 `/api/validate` 接口：配置文件语法校验

[src/pages/api/validate.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/validate.js) 在页面加载时被调用，检查所有 YAML 配置文件的语法：

```js
const configs = ["docker.yaml", "settings.yaml", "services.yaml", "bookmarks.yaml", "kubernetes.yaml", "proxmox.yaml"];

export default async function handler(req, res) {
  let errors = configs.map((config) => checkAndCopyConfig(config)).filter((status) => status !== true);
  // ...
  res.send(errors);
}
```

如果存在 YAML 解析错误（[checkAndCopyConfig](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L44-L49) 返回的是 yaml 解析异常对象而非 true），前端会在 [index.jsx 第 133-183 行](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx#L133-L183) 显示错误页面而非仪表盘。

注意 `/api/validate` 的校验列表与 `/api/hash` 的哈希列表**不完全相同**：
- `/api/hash` 包含：docker.yaml, settings.yaml, services.yaml, bookmarks.yaml, widgets.yaml, **custom.css**, **custom.js**
- `/api/validate` 包含：docker.yaml, settings.yaml, services.yaml, bookmarks.yaml, **kubernetes.yaml**, **proxmox.yaml**

差异原因是 `validate` 负责 YAML 语法校验（kubernetes.yaml / proxmox.yaml 是 YAML 文件，容易出错），而 `hash` 负责检测配置变更（custom.css / custom.js 是用户自定义资源，变更后需要刷新页面）。

#### 2.3.9 完整协作时序

```
容器启动
  │
  ├─ docker-entrypoint.sh
  │   └─ export HOMEPAGE_BUILDTIME=$(date +%s)    ← 每启动一次都是新值
  │
  ▼
Next.js server 启动
  │
  ├─ getStaticProps() 首次执行
  │   ├─ getSettings() / servicesResponse() / ...
  │   └─ 生成静态页面 + SWR fallback 数据
  │
  ▼
用户浏览器访问首页
  │
  ├─ 1. 服务端渲染 HTML 返回（内含 initialSettings + fallback JSON）
  │
  ├─ 2. React Hydration
  │     ├─ Index 挂载 → 发起 GET /api/validate + GET /api/hash
  │     │   （均在 SWRConfig fallback 外层，不读缓存）
  │     │
  │     ├─ 校验通过 → 渲染 SWRConfig(fallback) → Home 挂载
  │     │   ├─ 渲染阶段：立即用 fallback 数据显示（无加载状态）
  │     │   └─ 挂载后（revalidateOnMount 默认 true）：后台发起 3 个请求刷新
  │     │       ├─ GET /api/services   （静默刷新）
  │     │       ├─ GET /api/bookmarks  （静默刷新）
  │     │       └─ GET /api/widgets    （静默刷新）
  │     │
  │     └─ GET /api/hash 返回 → localStorage 写入基准 hash
  │
  ▼
用户切换走标签页 → 切回来（触发 window focus）
  │
  ├─ useWindowFocus() → true
  ├─ useEffect → mutateHash()    ← 显式强制 SWR 重新 fetch /api/hash
  │
  ├─ 后端 hash.js 重新计算
  │   ├─ configs 内容哈希（如果配置文件没变则相同）
  │   └─ HOMEPAGE_BUILDTIME（如果容器没重启则相同）
  │
  ▼
有两种情况：

情况 A：配置未修改，容器未重启
  └─ 哈希相同 → mutateHash() 返回相同数据 → 无动作

情况 B：配置已修改 或 容器已重启
  └─ 哈希不同
      ├─ setStale(true)     ← Home 整棵子树卸载，显示旋转加载动画
      ├─ localStorage 更新 hash
      ├─ fetch("/api/revalidate") → 服务端 ISR 重新 getStaticProps()
      └─ window.location.reload() → 全页硬刷新，新页面使用新的 fallback 数据
```

#### 2.3.10 三种「配置变化」场景的实际效果

| 场景 | HOMEPAGE_BUILDTIME | 配置文件哈希 | combinedHash 变化 | 前端行为 |
|---|---|---|---|---|
| 用户编辑了 services.yaml | 不变 | 变化 | ✅ 变化 | Window focus 后：Home 卸载 → ISR → 硬刷新 |
| `docker restart homepage` | 变化 | 不变 | ✅ 变化 | Window focus 后：Home 卸载 → ISR → 硬刷新 |
| `docker-compose up -d --force-recreate` | 变化 | 不变 | ✅ 变化 | Window focus 后：Home 卸载 → ISR → 硬刷新 |
| 无任何变化 | 不变 | 不变 | ❌ 不变 | 正常显示 |

这个设计的好处是：**容器重启后，即使配置文件在 volume 中完全未变，前端也会在用户回到页面时自动重新拉取最新的服务列表**。这对于 Docker/K8s 服务发现（自动发现新容器/新 Ingress）尤其重要——比如新增了一个带 homepage 标签的容器，只要重启过 Homepage 容器（或等待下次触发），前端就能感知并展示新服务。

### 2.4 应用配置期注入（YAML 模板变量替换）

这是 Homepage 最有特色的一层。[src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L8-L9) 定义了两个前缀：

```js
const homepageVarPrefix = "HOMEPAGE_VAR_";
const homepageFilePrefix = "HOMEPAGE_FILE_";
```

[substituteEnvironmentVars](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L64-L80) 函数在读取任何 YAML 配置文件时，自动替换 `{{HOMEPAGE_VAR_XXX}}` 和 `{{HOMEPAGE_FILE_XXX}}` 占位符：

- `HOMEPAGE_VAR_XXX` → 替换为环境变量 `XXX` 的值
- `HOMEPAGE_FILE_XXX` → 读取环境变量值作为文件路径，替换为文件内容

此函数被以下配置加载路径统一调用：

| 配置文件 | 调用位置 |
|---|---|
| `docker.yaml` | [src/utils/config/docker.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/docker.js#L21) |
| `kubernetes.yaml` | [src/utils/config/kubernetes.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/kubernetes.js#L11) |
| `services.yaml` | [src/utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/service-helpers.js#L58) |
| `bookmarks.yaml` | [src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/api-response.js#L32) |
| `settings.yaml` | [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L87) |
| Docker label 值 | [src/utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/service-helpers.js#L115) |

### 2.5 安全相关环境变量

**`HOMEPAGE_ALLOWED_HOSTS`**：[src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/middleware.js) 在所有 `/api/*` 请求上校验 Host 头，防止 Host Header Injection 攻击。

```
默认允许: localhost:3000, 127.0.0.1:3000, [::1]:3000
+ HOMEPAGE_ALLOWED_HOSTS 中逗号分隔的域名
若设为 "*" → 允许所有
```

**`HOMEPAGE_CONFIG_DIR`**：[src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L11-L13) 确定配置目录位置：

```js
export const CONF_DIR = process.env.HOMEPAGE_CONFIG_DIR
  ? process.env.HOMEPAGE_CONFIG_DIR
  : join(process.cwd(), "config");
```

K8s Helm 部署中通过 `env` 字段设置（见 [k3d/k3d-helm-values.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/k3d-helm-values.yaml#L49-L51)）：

```yaml
env:
  - name: HOMEPAGE_ALLOWED_HOSTS
    value: "homepage.k3d.localhost:8080"
```

---

## 三、容器展示信息：Docker 与 Kubernetes 双通道

Homepage 作为服务仪表盘，既能监控 Docker 容器状态，也能监控 Kubernetes 工作负载。两者在代码层面采用了对称但独立的通道。

### 3.1 Docker 通道

#### 服务发现（标签自动发现）

[servicesFromDocker](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/service-helpers.js#L63-L170) 读取 `docker.yaml` 中配置的 Docker 连接信息，通过 dockerode 连接 Docker daemon，遍历容器/服务，筛选带 `homepage.` 前缀标签的容器：

```
容器标签 homepage.name → 服务名称
容器标签 homepage.group → 服务分组
容器标签 homepage.widget.type → widget 类型
容器标签 homepage.instance.XXX → 多实例过滤
```

#### 连接配置

[getDockerArguments](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/docker.js#L16-L63) 解析 `docker.yaml`（经过环境变量替换后），支持：

- `socket` → `{ socketPath: "/var/run/docker.sock" }`
- `host` + 可选 `port` → `{ host, port }`
- `tls` → 读取 caFile/certFile/keyFile
- `swarm` → 启用 Docker Swarm 模式
- `headers` → 自定义请求头

#### 状态 API

[api/docker/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/docker/status/[...service].js)：

```
请求: /api/docker/status/{containerName}/{server}
流程:
  1. getDockerArguments(server) → 获取连接参数
  2. docker.listContainers({all: true}) → 查找容器
  3. container.inspect() → 获取 State.Status + State.Health.Status
  4. 若启用 swarm → 检查 service 和 task 状态
返回: { status, health }
```

#### 统计 API

[api/docker/stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/docker/stats/[...service].js)：

```
请求: /api/docker/stats/{containerName}/{server}
流程:
  1. 同上获取连接参数和容器
  2. container.stats({stream: false}) → 获取资源使用数据
  3. 若 swarm → 找到本地 task 容器获取 stats
返回: { stats } (包含 cpu, memory, network 等原始数据)
```

#### 前端展示

Docker 服务的展示链路：

```
service.container 存在
  → [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/item.jsx#L106-L114) 渲染 <Status> 组件
    → [status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/status.jsx) 调用 /api/docker/status/
    → 展示: running / healthy / unhealthy / starting / exited / not found / partial
  → 点击展开 <Docker> 组件
    → [widgets/docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/docker/component.jsx) 同时调用 /api/docker/status/ + /api/docker/stats/
    → 展示: CPU%, 内存, 网络RX/TX
```

Status 组件的颜色映射（[status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/status.jsx#L17-L43)）：

| 状态 | 颜色 |
|---|---|
| running | emerald-500 (绿) |
| healthy | emerald-500 (绿) |
| starting | blue-500 (蓝) |
| unhealthy | orange-400 (橙) |
| not found / exited / partial | orange-400 (橙) |
| error | rose-500 (红) |
| unknown | 黑/白低对比度 |

### 3.2 Kubernetes 通道

#### 服务发现（Ingress / HTTPRoute 注解发现）

[servicesFromKubernetes](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/service-helpers.js#L172-L229) 通过 KubeConfig 连接集群，遍历 Ingress、Traefik IngressRoute、HTTPRoute 资源，筛选带有 `gethomepage.dev/` 前缀注解的资源：

```yaml
annotations:
  gethomepage.dev/enabled: "true"
  gethomepage.dev/name: "MyApp"
  gethomepage.dev/group: "MyGroup"
  gethomepage.dev/icon: "app.png"
```

注解基础常量定义在 [kubernetes.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/kubernetes.js#L58-L60)：

```js
export const ANNOTATION_BASE = "gethomepage.dev";
export const ANNOTATION_WIDGET_BASE = `${ANNOTATION_BASE}/widget.`;
```

#### 连接配置

[getKubeConfig](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/kubernetes.js#L17-L34) 读取 `kubernetes.yaml`，根据 mode 选择：

- `cluster` → `kc.loadFromCluster()` (in-cluster 方式，K8s Pod 内)
- `default` → `kc.loadFromDefault()` (kubeconfig 文件)
- `disabled` → 返回 null

#### 状态 API

[api/kubernetes/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/kubernetes/status/[...service].js)：

```
请求: /api/kubernetes/status/{namespace}/{appName}?podSelector=xxx
流程:
  1. getKubeConfig() → 获取 K8s 客户端
  2. labelSelector = podSelector ?? "app.kubernetes.io/name=appName"
  3. coreApi.listNamespacedPod({namespace, labelSelector})
  4. 判断 Pod 状态: allReady→running, someReady→partial, else→down
返回: { status }
```

#### 前端展示

Kubernetes 服务的展示链路：

```
service.app 存在（且 service.external 不存在）
  → [item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/item.jsx#L116-L125) 渲染 <KubernetesStatus> 组件
    → [kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/kubernetes-status.jsx) 调用 /api/kubernetes/status/
    → 展示: running / down / partial / not found
  → 点击展开 <Kubernetes> 组件
    → [widgets/kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/kubernetes/component.jsx) 同时调用 /api/kubernetes/status/ + /api/kubernetes/stats/
    → 展示: CPU%, 内存
```

### 3.3 双通道在 servicesResponse 中的合并

[api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/api-response.js#L158-L256) 中 `servicesResponse()` 是三种来源的汇聚点：

```
servicesFromDocker()       → discoveredDockerServices
servicesFromKubernetes()   → discoveredKubernetesServices
servicesFromConfig()       → configuredServices（来自 services.yaml）

三者的 group name 合并去重
同 group 内 services 数组合并 + 按 weight 排序
最终按 settings.yaml 的 layout 定义排序
```

这意味着一个服务可以同时被 Docker 发现和 K8s 发现，只要 group name 一致就会合并在同一个分组下展示。

### 3.4 服务发现的配置骨架

首次运行时，[checkAndCopyConfig](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js#L15-L50) 会将 `src/skeleton/` 下的模板文件复制到配置目录：

| 骨架文件 | 内容 |
|---|---|
| [docker.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/skeleton/docker.yaml) | 注释示例，展示 socket 和 host 两种连接方式 |
| [kubernetes.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/skeleton/kubernetes.yaml) | 空 sample |
| [services.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/skeleton/services.yaml) | 服务分组示例 |
| [settings.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/skeleton/settings.yaml) | provider API key 占位 |

---

## 四、部署差异：Docker / Kubernetes / Tilt / Source 四种模式对比

### 4.1 Docker 部署

```
镜像: ghcr.io/gethomepage/homepage:latest
端口: 3000
卷挂载:
  - /path/to/config:/app/config        → 配置文件
  - /var/run/docker.sock:ro (可选)      → Docker 集成
环境变量:
  - HOMEPAGE_ALLOWED_HOSTS (必须)
  - PUID / PGID (可选，非 root 运行)
入口: docker-entrypoint.sh → node server.js
```

Docker 部署下，容器通过 socket 直接与本机 Docker daemon 通信，`docker.yaml` 通常配置 `socket: /var/run/docker.sock`。

### 4.2 Kubernetes 部署

```
镜像: 同上，通过 Helm Chart 部署
特殊配置:
  - kubernetes.yaml mode: cluster → 使用 in-cluster ServiceAccount
  - ServiceAccount + RBAC 权限（读取 Ingress/Pod/CRD）
  - HOMEPAGE_ALLOWED_HOSTS 通过 Helm values 注入
  - Ingress 注解自动发现服务
```

K8s 部署的核心差异：
1. 不依赖 Docker socket，而是通过 K8s API 发现服务
2. [kubernetes.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/skeleton/kubernetes.yaml) 的 `mode: cluster` 让应用使用 Pod 内 ServiceAccount
3. 服务信息来自 Ingress/HTTPRoute 的 `gethomepage.dev/` 注解，而非 Docker label
4. Helm Chart 中 `serviceAccount.create: true` + `enableRbac: true` 确保有足够权限

### 4.3 Tilt 开发部署

```
镜像: k3d-registry.localhost:55000/homepage:local
Dockerfile: Dockerfile-tilt (node:18-alpine, npx next dev)
集群: k3d (Docker 内 K3s)
热更新: Tilt live_update 同步文件 + pnpm install
```

[k3d/k3d.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/k3d.yaml) 定义了 1 server + 2 agent 的 k3d 集群，端口映射 8080→80。

[k3d/k3d-helm-values.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/k3d-helm-values.yaml) 中：
- `image.repository: k3d-registry.localhost:55000/homepage` 指向本地 registry
- `kubernetes.mode: cluster` 启用集群内发现
- `persistence.dotnext` 默认关闭（Tilt 中可开启加速重启）
- `ingress` 注入了 `gethomepage.dev/` 注解，用于自发现测试

### 4.4 源码直接运行

```
pnpm install → pnpm dev
无 Docker / 无 K8s 集成（或仅通过远程 host 连接）
NEXT_PUBLIC_* 变量为空 → Version 组件显示 "dev"
```

### 4.5 四种模式差异汇总

| 维度 | Docker | Kubernetes | Tilt 开发 | 源码运行 |
|---|---|---|---|---|
| 镜像基础 | node:22-alpine | 同 Docker | node:18-alpine | 无 |
| 构建方式 | standalone | standalone | next dev | next dev |
| 服务发现 | Docker label | K8s 注解 | K8s 注解 | services.yaml |
| 容器连接 | docker.sock | K8s API | K8s API | 远程 host |
| 环境变量 | docker run -e | Helm values.env | Helm values.env | shell env |
| 版本信息 | 正式 semver | 正式 semver | dev | dev |
| 热更新 | 无 | 无 | live_update | HMR |
| 非root支持 | PUID/PGID | SecurityContext | - | - |

---

## 五、协作全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CI / 构建阶段                                 │
│  docker-publish.yml                                                 │
│  ├─ pnpm build (NEXT_PUBLIC_BUILDTIME/VERSION/REVISION → 前端JS)    │
│  └─ docker build (CI=true, ARG 传递 → 跳过二次构建)                  │
│      └─ 产物: ghcr.io/gethomepage/homepage:vX.Y.Z                  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        容器启动阶段                                  │
│  docker-entrypoint.sh                                               │
│  ├─ PUID/PGID → chown + su-exec 降权                                │
│  ├─ HOSTNAME → IPv6 探测 / IPv4 回退                                │
│  ├─ /app/config → 符号链接兼容                                      │
│  ├─ export HOMEPAGE_BUILDTIME=$(date +%s)   ← 每次启动都是新值       │
│  └─ exec node server.js                                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        应用运行阶段                                  │
│  Next.js standalone server                                          │
│  ├─ middleware.js: HOMEPAGE_ALLOWED_HOSTS 校验                      │
│  ├─ config.js: HOMEPAGE_VAR_*/FILE_* → YAML 模板替换               │
│  ├─ CONF_DIR = HOMEPAGE_CONFIG_DIR || /app/config                   │
│  ├─ checkAndCopyConfig: 骨架文件自动初始化                           │
│  │                                                                   │
│  ├─ 配置刷新通道:                                                     │
│  │   /api/hash → sha256(configs 内容 + HOMEPAGE_BUILDTIME)          │
│  │   /api/revalidate → res.revalidate("/") → 重新执行 getStaticProps│
│  │   /api/validate → YAML 语法错误检测                              │
│  │                                                                   │
│  ├─ Docker 通道:                                                     │
│  │   docker.yaml → dockerode → listContainers/inspect/stats         │
│  │   → /api/docker/status + /api/docker/stats                       │
│  │   → <Status> + <Docker> 组件                                      │
│  │                                                                   │
│  ├─ Kubernetes 通道:                                                 │
│  │   kubernetes.yaml → KubeConfig → Ingress/Pod/CRD 查询            │
│  │   → /api/kubernetes/status + /api/kubernetes/stats               │
│  │   → <KubernetesStatus> + <Kubernetes> 组件                        │
│  │                                                                   │
│  └─ 合并输出: servicesResponse() → 三源合并 → 前端展示               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        前端展示与刷新阶段                             │
│  index.jsx → Index → Home                                           │
│  ├─ 首屏渲染: getStaticProps → initialSettings + SWR fallback       │
│  │   ├─ Index 层（SWRConfig 外层）: hash/validate 始终真实请求       │
│  │   └─ Home 层（SWRConfig 内层）: 先用 fallback 渲染 + 后台重请求    │
│  │                                                                   │
│  ├─ 配置刷新检测:                                                    │
│  │   useWindowFocus() → window focus 事件                           │
│  │     → mutateHash() → 重新请求 /api/hash                          │
│  │     → 与 localStorage.hash 对比                                  │
│  │     → 若不同: setStale(卸载Home) → /api/revalidate → reload     │
│  │                                                                   │
│  ├─ Version 组件: NEXT_PUBLIC_VERSION/REVISION/BUILDTIME            │
│  │   └─ /api/releases → 更新检测                                    │
│  │                                                                   │
│  ├─ ServicesGroup → Item                                            │
│  │   ├─ service.container → <Status> + <Docker>                     │
│  │   └─ service.app → <KubernetesStatus> + <Kubernetes>             │
│  │                                                                   │
│  └─ Widget 组件: 各服务自定义 widget                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键文件索引

| 职责 | 文件 |
|---|---|
| 生产 Dockerfile | [Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/Dockerfile) |
| 开发 Dockerfile | [Dockerfile-tilt](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/Dockerfile-tilt) |
| 容器入口脚本 | [docker-entrypoint.sh](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/docker-entrypoint.sh) |
| CI 镜像发布 | [.github/workflows/docker-publish.yml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/.github/workflows/docker-publish.yml) |
| Next.js 配置 | [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/next.config.js) |
| 环境变量替换 | [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/config.js) |
| Docker 连接配置 | [src/utils/config/docker.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/docker.js) |
| K8s 连接配置 | [src/utils/config/kubernetes.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/kubernetes.js) |
| 服务发现与合并 | [src/utils/config/service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/service-helpers.js) |
| API 响应聚合 | [src/utils/config/api-response.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/config/api-response.js) |
| Docker 状态 API | [src/pages/api/docker/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/docker/status/[...service].js) |
| Docker 统计 API | [src/pages/api/docker/stats/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/docker/stats/[...service].js) |
| K8s 状态 API | [src/pages/api/kubernetes/status/[...service].js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/kubernetes/status/[...service].js) |
| 配置哈希 API | [src/pages/api/hash.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/hash.js) |
| ISR 再生成 API | [src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/revalidate.js) |
| 配置校验 API | [src/pages/api/validate.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/api/validate.js) |
| 首页入口（含刷新逻辑） | [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/pages/index.jsx) |
| 安全中间件 | [src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/middleware.js) |
| 版本展示组件 | [src/components/version.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/version.jsx) |
| Docker 状态展示 | [src/components/services/status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/status.jsx) |
| K8s 状态展示 | [src/components/services/kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/kubernetes-status.jsx) |
| Docker 统计展示 | [src/widgets/docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/docker/component.jsx) |
| K8s 统计展示 | [src/widgets/kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/kubernetes/component.jsx) |
| 服务卡片入口 | [src/components/services/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/item.jsx) |
| 窗口焦点钩子 | [src/utils/hooks/window-focus.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/utils/hooks/window-focus.js) |
| 手动刷新按钮 | [src/components/toggles/revalidate.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/toggles/revalidate.jsx) |
| K8s 开发环境 | [k3d/](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d) 目录 |
| Helm Values | [k3d/k3d-helm-values.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/k3d-helm-values.yaml) |
