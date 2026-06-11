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
| `HOMEPAGE_BUILDTIME` | `date +%s` | 运行时生成的构建时间戳（仅 entrypoint 内部使用，非前端展示用） |
| `HOMEPAGE_CONFIG_DIR` | `/app/config` | 配置文件目录，见下文 |

entrypoint 的完整执行流：

```
1. 设置 PUID/PGID 默认值
2. 若 /app/config 不存在 → ln -s /config /app/config（兼容 lscr.io 路径）
3. 设置 HOMEPAGE_BUILDTIME
4. IPv6 探测 → 可能回退 HOSTNAME=0.0.0.0
5. 根据 PUID 调整 /app/config 和 /app/config/logs 的 ownership
6. 调整 /app/.next 的 ownership
7. 若 PUID != 0 且当前为 root → exec su-exec PUID:PGID 降权
8. 否则直接 exec "$@"（即 node server.js）
```

### 2.3 应用配置期注入（YAML 模板变量替换）

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

### 2.4 安全相关环境变量

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
│                        前端展示阶段                                  │
│  index.jsx → Home                                                   │
│  ├─ Version 组件: NEXT_PUBLIC_VERSION/REVISION/BUILDTIME            │
│  │   └─ /api/releases → 更新检测                                    │
│  ├─ ServicesGroup → Item                                            │
│  │   ├─ service.container → <Status> + <Docker>                     │
│  │   └─ service.app → <KubernetesStatus> + <Kubernetes>             │
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
| 安全中间件 | [src/middleware.js](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/middleware.js) |
| 版本展示组件 | [src/components/version.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/version.jsx) |
| Docker 状态展示 | [src/components/services/status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/status.jsx) |
| K8s 状态展示 | [src/components/services/kubernetes-status.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/kubernetes-status.jsx) |
| Docker 统计展示 | [src/widgets/docker/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/docker/component.jsx) |
| K8s 统计展示 | [src/widgets/kubernetes/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/widgets/kubernetes/component.jsx) |
| 服务卡片入口 | [src/components/services/item.jsx](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/src/components/services/item.jsx) |
| K8s 开发环境 | [k3d/](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d) 目录 |
| Helm Values | [k3d/k3d-helm-values.yaml](file:///d:/fz/0601/solo-dogfeeding/code/207-homepage/k3d/k3d-helm-values.yaml) |
