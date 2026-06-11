# Home Assistant 集成代码分析

## 一、整体架构概览

Homepage 项目的 Home Assistant 集成采用 **三层架构** 设计：

```
┌─────────────────────────────────────────────────────────┐
│  前端组件层 (React)                                      │
│  component.jsx - UI 渲染 + useWidgetAPI hook            │
└─────────────────────┬───────────────────────────────────┘
                      │ SWR (HTTP 请求)
                      ▼
┌─────────────────────────────────────────────────────────┐
│  后端代理层 (Next.js API Route)                          │
│  proxy.js - 服务端代理、认证、数据转换                   │
└─────────────────────┬───────────────────────────────────┘
                      │ httpProxy (HTTP 请求)
                      ▼
┌─────────────────────────────────────────────────────────┐
│  Home Assistant 实例 (外部服务)                           │
│  REST API (/api/states, /api/template)                  │
└─────────────────────────────────────────────────────────┘
```

---

## 二、核心文件与职责

### 2.1 文件清单

| 文件 | 层级 | 职责 |
|------|------|------|
| [widget.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/widget.js) | 配置层 | 导出 widget 配置，指定 proxyHandler |
| [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js) | 后端层 | 处理 API 代理请求、数据转换 |
| [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx) | 前端层 | React 组件，渲染数据 |
| [use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/proxy/use-widget-api.js) | 通用 hook | 基于 SWR 的数据获取 hook |
| [proxy.js (API 路由)](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/api/services/proxy.js) | 路由层 | 通用代理路由，分发到各 widget 的 proxyHandler |

### 2.2 Widget 注册机制

Home Assistant widget 通过两个入口注册：

**后端注册** - [widgets.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/widgets.js)：
```javascript
import homeassistant from "./homeassistant/widget";
const widgets = { homeassistant, /* ... */ };
```

**前端注册** - [components.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/components.js)：
```javascript
const components = {
  homeassistant: dynamic(() => import("./homeassistant/component")),
  // ...
};
```

---

## 三、状态获取流程详解

### 3.1 完整数据流图

```
1. 组件挂载
   component.jsx
       │
       ▼
2. useWidgetAPI hook (SWR)
   use-widget-api.js
       │
       ├─ 构建代理 URL: /api/services/proxy?group=...&service=...&index=...
       └─ SWR 发起 fetch 请求
       │
       ▼
3. 通用代理路由
   pages/api/services/proxy.js
       │
       ├─ 调用 getServiceWidget() 获取 widget 配置
       ├─ 根据 type 查找对应 widget
       └─ 调用 widget.proxyHandler(req, res)
       │
       ▼
4. Home Assistant 代理处理器
   widgets/homeassistant/proxy.js
       │
       ├─ 解析查询配置 (defaultQueries 或 custom)
       ├─ 并行发起多个 HA API 请求
       │   ├─ /api/template (POST) - Jinja2 模板
       │   └─ /api/states/<entity> (GET) - 实体状态
       └─ 格式化输出为 { label, value } 数组
       │
       ▼
5. 前端渲染
   component.jsx
       │
       └─ 遍历数据，渲染 Block 组件
```

### 3.2 前端数据获取：useWidgetAPI

[use-widget-api.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/proxy/use-widget-api.js#L1-L17) 是所有 widget 通用的数据获取 hook：

```javascript
export default function useWidgetAPI(widget, ...options) {
  const config = {};
  if (options && options[1]?.refreshInterval) {
    config.refreshInterval = options[1].refreshInterval;
  }
  let url = formatProxyUrl(widget, ...options);
  // ...
  const { data, error, mutate } = useSWR(url, config);
  return { data, error: data?.error ?? error, mutate };
}
```

**关键特性：**
- 基于 [SWR](https://swr.vercel.app/) (stale-while-revalidate) 库
- 支持自定义 `refreshInterval` 刷新间隔
- 自动错误处理（优先使用 data.error）
- 提供 `mutate` 方法用于手动刷新

### 3.3 代理 URL 构建

[api-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/proxy/api-helpers.js#L43-L49) 中的 `formatProxyUrl` 函数：

```javascript
export function formatProxyUrl(widget, endpoint, queryParams) {
  const params = getURLSearchParams(widget, endpoint);
  if (queryParams) {
    params.append("query", JSON.stringify(queryParams));
  }
  return `/api/services/proxy?${params.toString()}`;
}
```

构建出的 URL 示例：
```
/api/services/proxy?group=home&service=homeassistant&index=0
```

### 3.4 服务端代理分发

[pages/api/services/proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/api/services/proxy.js#L10-L115) 是所有 widget 代理请求的统一入口：

```javascript
export default async function handler(req, res) {
  const { service, group, index } = req.query;
  const serviceWidget = await getServiceWidget(group, service, index);
  const type = serviceWidget?.type;
  const widget = widgets[type];
  
  const serviceProxyHandler = widget.proxyHandler || genericProxyHandler;
  
  if (!req.query.endpoint) {
    return await serviceProxyHandler(req, res);
  }
  // ... mappings 处理
}
```

---

## 四、实体转换逻辑

### 4.1 两种查询模式

Home Assistant widget 支持 **两种查询模式**，由 [proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L33-L62) 中的 `getQuery` 函数处理：

#### 模式一：Template 查询（默认）

使用 Home Assistant 的 Jinja2 模板引擎：

```javascript
{
  template: "{{ states.person|selectattr('state','equalto','home')|list|length }} / {{ states.person|list|length }}",
  label: "homeassistant.people_home",
}
```

- **请求方式**：`POST /api/template`
- **请求体**：`{ template: "..." }`
- **输出格式**：`{ label, value: data.toString() }`

#### 模式二：State 查询

直接获取指定实体的状态：

```javascript
{
  state: "sensor.temperature",
  label: "{attributes.friendly_name}",
  value: "{state} {attributes.unit_of_measurement}",
}
```

- **请求方式**：`GET /api/states/<entity_id>`
- **输出格式**：通过 `formatOutput` 函数解析模板字符串

### 4.2 模板格式化：formatOutput

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L22-L31) 中的 `formatOutput` 函数实现了简单的模板替换：

```javascript
function formatOutput(output, data) {
  return output.replace(
    /\{.*?\}/g,
    (match) =>
      match
        .replace(/\{|\}/g, "")
        .split(".")
        .reduce((o, p) => (o ? o[p] : ""), data) ?? "",
  );
}
```

**工作原理：**
1. 使用正则 `/\{.*?\}/g` 匹配所有 `{...}` 占位符
2. 去掉花括号，按 `.` 分割成路径数组
3. 使用 `reduce` 深度访问对象属性
4. 找不到则返回空字符串

**示例：**
```
输入模板: "{state} {attributes.unit_of_measurement}"
数据对象: { state: "23.5", attributes: { unit_of_measurement: "°C" } }
输出结果: "23.5 °C"
```

### 4.3 默认查询配置

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L7-L20) 中定义了三个默认查询：

```javascript
const defaultQueries = [
  {
    template: "{{ states.person|selectattr('state','equalto','home')|list|length }} / {{ states.person|list|length }}",
    label: "homeassistant.people_home",
  },
  {
    template: "{{ states.light|selectattr('state','equalto','on')|list|length }} / {{ states.light|list|length }}",
    label: "homeassistant.lights_on",
  },
  {
    template: "{{ states.switch|selectattr('state','equalto','on')|list|length }} / {{ states.switch|list|length }}",
    label: "homeassistant.switches_on",
  },
];
```

### 4.4 自定义查询

用户可以通过 `custom` 字段自定义查询（最多 4 个）：

```javascript
if (!widget.fields && widget.custom) {
  if (typeof widget.custom === "string") {
    widget.custom = JSON.parse(widget.custom);
  }
  queries = widget.custom.slice(0, 4);
}
```

### 4.5 并行请求与结果处理

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L91-L104) 使用 `Promise.all` 并行执行所有查询：

```javascript
const results = await Promise.all(queries.map((q) => getQuery(q, widget)));

const err = results.find((r) => r.result[2]?.error);
if (err) {
  const [status, , data] = err.result;
  return res.status(status).send(data);
}

return res.status(200).send(
  results.map((r) => {
    const [status, , data] = r.result;
    return status === 200 ? r.output(data) : { label: status, value: data.toString() };
  }),
);
```

**错误处理策略：**
- 任一查询出错 → 立即返回错误
- 成功的查询 → 通过 `output()` 函数转换为 `{ label, value }` 格式
- 非 200 状态码 → label 设为状态码，value 设为响应数据

---

## 五、界面刷新机制

### 5.1 SWR 驱动的自动刷新

Home Assistant 组件在 [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L9-L10) 中设置了 60 秒刷新间隔：

```javascript
const { data, error } = useWidgetAPI(widget, null, { refreshInterval: 60000 });
```

### 5.2 SWR 的刷新策略

SWR 提供了多种刷新机制：

| 刷新方式 | 说明 |
|---------|------|
| `refreshInterval` | 定时刷新（Home Assistant 使用 60s） |
| `revalidateOnFocus` | 窗口聚焦时重新验证（默认开启） |
| `revalidateOnReconnect` | 网络重连时重新验证（默认开启） |
| `mutate()` | 手动触发刷新 |

### 5.3 服务端配置获取

在页面初始化时，[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/index.jsx#L55-L95) 的 `getStaticProps` 会预取服务数据：

```javascript
export async function getStaticProps() {
  const services = await servicesResponse();
  // ...
  return {
    props: {
      fallback: {
        "/api/services": services,
        // ...
      },
    },
  };
}
```

这意味着页面首次加载时，服务列表数据是预渲染的，而 widget 的具体数据是在客户端通过 SWR 动态获取的。

### 5.4 配置变更检测

[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/index.jsx#L102-L131) 中还有一个配置变更检测机制：

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

useEffect(() => {
  if (hashData) {
    const previousHash = localStorage.getItem("hash");
    if (previousHash && previousHash !== hashData.hash) {
      setStale(true);
      fetch("/api/revalidate").then((res) => {
        if (res.ok) {
          window.location.reload();
        }
      });
    }
  }
}, [hashData]);
```

当配置文件变更时，页面会自动刷新。

---

## 六、组件渲染流程

### 6.1 服务卡片渲染链路

```
ServicesGroup
    │
    ▼
Item [item.jsx]
    │
    ├─ 服务标题、图标
    ├─ 状态标签 (ping, site monitor, container 等)
    └─ widgets.map → Widget
           │
           ▼
        Widget [services/widget.jsx]
           │
           ├─ 根据 widget.type 查找组件
           └─ 渲染 ServiceWidget 组件
                  │
                  ▼
               Component [homeassistant/component.jsx]
                  │
                  ├─ useWidgetAPI 获取数据
                  └─ data.map → Block
                         │
                         ▼
                      Block [services/widget/block.jsx]
                         ├─ label (翻译)
                         └─ value
```

### 6.2 Home Assistant 组件

[component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L1-L21) 非常简洁：

```javascript
export default function Component({ service }) {
  const { widget } = service;
  const { data, error } = useWidgetAPI(widget, null, { refreshInterval: 60000 });
  
  if (error) {
    return <Container service={service} error={error} />;
  }

  return (
    <Container service={service}>
      {data?.map((d) => (
        <Block label={d.label} value={d.value} key={d.label} />
      ))}
    </Container>
  );
}
```

### 6.3 Container 容器组件

[Container](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/container.jsx) 负责：
- 错误显示/隐藏（根据 `hideErrors` 配置）
- `fields` 字段过滤（只显示指定的字段）
- 高亮配置上下文
- Widget 别名映射（如 `pialert` → `netalertx`）

### 6.4 Block 块组件

[Block](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/block.jsx) 负责：
- 显示 label（经过 `t()` 翻译）
- 显示 value
- 高亮效果（根据 highlight 配置）
- 加载状态（value 为 undefined 时显示脉冲动画）

---

## 七、关键技术点

### 7.1 服务端安全代理

所有对 Home Assistant 的请求都通过服务端代理，有以下安全优势：

1. **API Key 保护**：密钥只存在服务端，不暴露给前端
2. **CORS 绕过**：服务端请求不受跨域限制
3. **请求白名单**：只允许预定义的 API 端点
4. **错误脱敏**：[sanitizeErrorURL](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/proxy/api-helpers.js#L71-L79) 会移除敏感参数

### 7.2 配置白名单机制

[service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L256-L439) 中的 `cleanServiceGroups` 函数采用白名单机制：

- 只有明确列出的配置字段才会传递到前端
- 敏感字段（如 `url`, `key` 等）只在服务端使用
- 防止配置信息泄露

### 7.3 HTTP 客户端

[http.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/proxy/http.js) 提供了统一的 HTTP 客户端：

- 支持 HTTP/HTTPS
- 支持 gzip/deflate 解压
- Cookie 管理
- DNS 回退机制（解决 Alpine/musl 的 DNS 问题）
- 连接池（keep-alive agent）
- 错误日志

---

## 八、总结：状态-转换-刷新联动

### 8.1 完整闭环

```
┌──────────────────────────────────────────────────────────┐
│                     页面初始化阶段                          │
│  getStaticProps → 预取服务配置 → SWR fallback            │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     组件挂载阶段                          │
│  Component 挂载 → useWidgetAPI → SWR 首次请求            │
│  → /api/services/proxy → homeassistantProxyHandler      │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     数据获取阶段                          │
│  getQuery (state/template) → HA REST API                │
│  → formatOutput 格式化 → { label, value } 数组          │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     界面渲染阶段                          │
│  SWR 更新 data → Component re-render → Block 渲染       │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     定时刷新阶段                          │
│  refreshInterval (60s) → SWR revalidate → 重复数据获取  │
│  → 界面更新                                              │
└──────────────────────────────────────────────────────────┘
```

### 8.2 核心联动点

| 联动点 | 涉及模块 | 说明 |
|--------|---------|------|
| 配置 → 代理 | `getServiceWidget` + `proxy.js` | 服务端根据配置找到对应 HA 实例地址和密钥 |
| 原始数据 → 展示数据 | `formatOutput` + `output()` | HA API 原始数据 → `{ label, value }` 格式 |
| 数据 → 界面 | `useWidgetAPI` + `Block` | SWR 响应式更新 React 组件 |
| 定时 → 刷新 | `refreshInterval` + SWR | 60 秒自动刷新数据 |
| 窗口聚焦 → 刷新 | SWR `revalidateOnFocus` | 切回页面时自动刷新 |

### 8.3 扩展点

Home Assistant widget 的设计支持以下扩展：

1. **自定义查询**：通过 `custom` 配置添加更多 state 或 template 查询
2. **字段过滤**：通过 `fields` 配置控制显示哪些数据块
3. **高亮规则**：通过 `highlight` 配置根据值设置颜色高亮
4. **刷新间隔**：通过 `refreshInterval` 调整刷新频率（需修改代码支持配置）

---

## 九、代码优化建议

### 9.1 错误处理可改进

当前实现中，任一查询失败就返回整个错误。可以考虑部分失败降级：

```javascript
// 替代当前的全部失败逻辑
return res.status(200).send(
  results.map((r) => {
    const [status, , data] = r.result;
    if (r.result[2]?.error) {
      return { label: "Error", value: status.toString(), error: true };
    }
    return status === 200 ? r.output(data) : { label: status, value: data.toString() };
  }),
);
```

### 9.2 刷新间隔可配置化

目前 60 秒是硬编码的。要真正让 refreshInterval 生效，需要改动**两处**（详细分析见第 12.1 节）：

**第一处** — [service-helpers.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js) 的 `cleanServiceGroups` 中，在 `type === "homeassistant"` 分支（或所有 widget 的通用段）把白名单解构出的 `refreshInterval` 赋到前端 widget 对象：

```javascript
// service-helpers.js cleanServiceGroups 中新增
if (type === "homeassistant") {
  if (refreshInterval) widget.refreshInterval = refreshInterval;
}
```

**第二处** — [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L9) 从 widget 读取而非使用字面量：

```javascript
// component.jsx 第 9 行
const { data, error } = useWidgetAPI(
  widget,
  null,
  { refreshInterval: widget.refreshInterval ?? 60000 },
);
```

只改其中任一处都无效：
- 只改第一处 → 前端 widget 有了值，但组件不用
- 只改第二处 → 组件想读，但 widget.refreshInterval 为 undefined

### 9.3 WebSocket 实时更新

对于需要实时更新的场景，可以考虑使用 Home Assistant 的 WebSocket API 替代轮询，减少不必要的请求。

---

## 十、自定义查询与字段过滤的交互关系

### 10.1 两条配置通道的本质区别

Home Assistant widget 存在两条影响"卡片最终展示内容"的配置通道，但它们**分处不同层级**，作用时机和影响范围完全不同：

| 维度 | `custom` 自定义查询 | `fields` 字段过滤 |
|------|---------------------|-------------------|
| **作用层级** | 服务端（proxy.js） | 前端（Container） |
| **影响对象** | 决定向 HA 发什么请求、返回哪些 `{label, value}` | 决定已返回的数据中哪些 Block 被渲染 |
| **数据流位置** | 请求阶段（上游） | 渲染阶段（下游） |
| **配置来源** | 原始 YAML（不经白名单过滤） | 经 `cleanServiceGroups` 白名单传递到前端 |
| **默认值** | 3 个 template 查询（people_home / lights_on / switches_on） | `null`（不过滤，全部显示） |

### 10.2 取舍机制：`fields` 优先于 `custom`

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L78-L89) 中的关键判断逻辑：

```javascript
let queries = defaultQueries;
if (!widget.fields && widget.custom) {
  // ...
  queries = widget.custom.slice(0, 4);
}
```

**条件分支解读：**

```
                widget.fields 存在？
                     │
          ┌──── 是 ──┴── 否 ────┐
          │                      │
   使用 defaultQueries    widget.custom 存在？
   （3个默认模板查询）          │
                          ┌─ 是 ─┴── 否 ──┐
                          │                │
                   使用 custom       使用 defaultQueries
                   （最多4个自定义）   （3个默认模板查询）
```

**核心规则：`fields` 的存在会阻断 `custom` 的生效。**

这是一个互斥设计：
- 配了 `fields` → 服务端始终使用默认查询，前端 Container 负责过滤显示
- 没配 `fields` 但配了 `custom` → 服务端使用自定义查询替代默认查询
- 两个都没配 → 使用默认查询，前端显示全部 3 个 Block

### 10.3 为什么 `fields` 会阻断 `custom`

原因在于两条通道的数据格式匹配关系：

1. **默认查询返回的 label** 是 i18n 键，如 `homeassistant.people_home`、`homeassistant.lights_on`
2. **`fields` 过滤** 使用 `child.props.label` 与 `homeassistant.xxx` 进行匹配
3. **`custom` 查询返回的 label** 由用户自定义，如 `sensor.temperature`、`Living Room`

如果 `fields` 和 `custom` 同时生效，`fields` 中的值（如 `homeassistant.people_home`）将无法匹配 `custom` 查询返回的 label，导致所有 Block 被过滤掉，卡片内容为空。因此代码用 `!widget.fields` 做了互斥保护。

### 10.4 四种配置组合的实际效果

| 组合 | `fields` | `custom` | 服务端查询 | 前端展示 |
|------|----------|----------|-----------|---------|
| A（默认） | 无 | 无 | 3 个默认 template | 显示全部 3 个 Block |
| B | 有 | 无 | 3 个默认 template | 只显示 fields 中列出的 Block |
| C | 无 | 有 | 自定义查询（最多 4 个） | 显示全部自定义 Block |
| D | 有 | 有 | 3 个默认 template | 只显示 fields 中列出的 Block（custom 被忽略！） |

> ⚠️ **注意**：组合 D 是一个容易踩坑的场景——用户同时配了 `fields` 和 `custom`，期望自定义查询生效，但实际 `custom` 被完全忽略，只有默认查询 + fields 过滤在工作。

### 10.5 两类配置归属与影响侧别逐字段核查

这是本分析中最关键的发现——每一类配置在代码中通过**不同的路径**传递，影响着不同的执行层面：

#### A. `custom` 自定义查询（纯服务端配置）

| 核查项 | 证据 | 结论 |
|--------|------|------|
| 是否在 `cleanServiceGroups` 白名单中解构 | [service-helpers.js#L256-L261](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L256-L261)：白名单里只有 `fields, hideErrors, highlight, type`，**没有 `custom`** | 前端 widget 对象中 **不存在** `custom` 字段 |
| 是否在前端组件中被引用 | `component.jsx`、`Container`、`Block` 中均无 `custom` 引用 | 前端完全不可见 |
| 服务端读取路径 | `getServiceWidget` → `getServiceItem` → `servicesFromConfig()` → `parseServicesToGroups()`，这是**原始 YAML 解析结果**，没有经过 `cleanServiceGroups`，所以包含完整的 `custom` | 服务端 **可读** |
| 服务端引用节点 | [proxy.js#L79](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L79)：`if (!widget.fields && widget.custom)` | 只在服务端影响查询取舍 |
| 定时刷新能否感知 `custom` 变更 | 每次代理请求都重新 `getServiceWidget()`，即重读原始 YAML | ✅ 定时刷新 **可以** 感知 |
| 配置重载（hash）是否必须 | 不需要——改完 `custom`，最多等 60 秒下一次定时刷新即生效 | ❌ 不依赖配置重载 |

**结论：`custom` 是 100% 的服务端配置，前端完全感知不到它的存在。**

#### B. `fields` 字段过滤（前后端两端存在，影响两个不同层面）

| 核查项 | 证据 | 结论 |
|--------|------|------|
| 是否在 `cleanServiceGroups` 白名单中解构 | [service-helpers.js#L258](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L258)：`fields` 在白名单中；[#L453](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L453)：`fields: fieldsList \|\| null` | 前端 widget 对象中 **存在** `fields` 字段 |
| 服务端读取路径 | `getServiceWidget` 返回原始配置，包含完整的 `fields` | 服务端 **可读** |
| 服务端引用节点（影响查询取舍） | [proxy.js#L79](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L79)：`if (!widget.fields && widget.custom)` → `widget.fields` 只要不是 `null/false/undefined/""` 就会阻断 `custom` | 服务端层面：`fields` 决定用 defaultQueries 还是 custom |
| 前端引用节点（影响展示过滤） | [container.jsx#L35-L63](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/container.jsx#L35-L63)：`if (fields && type) { visibleChildren = childrenArray.filter(...) }` | 前端层面：`fields` 决定哪些 Block 被渲染 |
| 定时刷新能否感知 `fields` 变更（服务端侧） | 代理请求每次重读原始 YAML，`fields` 变更（从有到无/从无到有）会影响 `!widget.fields` 分支判断 | ✅ 服务端的查询取舍 **可以** 在定时刷新中感知 |
| 定时刷新能否感知 `fields` 变更（前端侧） | 前端 widget 对象来自 `/api/services`（`cleanServiceGroups` 处理结果），`useWidgetAPI` 只发代理请求，**不** 重新获取服务列表 | ❌ 前端的过滤规则 **不会** 在定时刷新中同步，必须等配置重载（整页刷新） |
| 配置重载（hash）是否必须 | 服务端查询取舍：不需要；前端过滤规则：必须 | **不对称** |

**结论：`fields` 是少数"跨两侧"的配置项——同一个字段，服务端和前端各读一份，做不同的事情。这是不对称行为的根源。**

#### C. `refreshInterval` 刷新间隔（代码核查：完全不可配置）

| 核查项 | 证据 | 结论 |
|--------|------|------|
| HA 组件是否从 `widget` 读取 | [component.jsx#L9](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L9)：`useWidgetAPI(widget, null, { refreshInterval: 60000 })` —— 字面量 60000，**没有** 读取 `widget.refreshInterval` | 组件侧硬编码 |
| `cleanServiceGroups` 是否为 HA 传递 `refreshInterval` | [service-helpers.js#L337-L338](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L337-L338) 白名单声明行注释：`// glances, customapi, iframe, prometheusmetric`；实际设置：仅 [#L532](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L532)（iframe）、[#L603](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L603)（glances）、[#L620](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L620)（customapi）、[#L679](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L679)（prometheusmetric）——**`homeassistant` 不在其中** | 即使 YAML 写了 `refreshInterval`，也不会传到前端 widget 对象 |
| YAML 配了 `refreshInterval` 是否生效 | 两条通路都堵了：前端传不到、组件也不读 | ❌ **完全不可配置，永远是 60000ms（60 秒）** |
| 配置重载能否改变 refreshInterval | 不能——重载了也还是硬编码 60000 | ❌ |
| 其他 widget 如何做（参考） | glances：`const { ..., refreshInterval = defaultInterval } = widget;` 然后传入 useWidgetAPI；customapi：`const { ..., refreshInterval = 10000 } = widget;`；iframe：自己实现 `setInterval` 而非 SWR | HA 组件缺少这一层读取逻辑 |

#### D. 三类配置横向对比汇总

下表的“HA 代理实际使用”指 Home Assistant 代理处理器是否读取并影响请求逻辑；不是指原始 YAML 在服务端是否能被解析。

| 配置项 | HA 代理实际使用 | 前端可见 | 服务端影响 | 前端影响 | 定时刷新可感知变更 | 需配置重载生效 | YAML 可配置 |
|--------|:---:|:---:|---------|---------|:---:|:---:|:---:|
| `custom` | ✅ | ❌ | 用自定义查询替代默认查询 | 无 | ✅ | ❌ | ✅ |
| `fields` | ✅ | ✅ | 阻断 custom 生效（用默认查询） | 过滤 Block 展示 | 服务端✅/前端❌ | **仅前端侧需要** | ✅ |
| `url` | ✅ | ❌ | HA 实例地址 | 无 | ✅ | ❌ | ✅ |
| `key` | ✅ | ❌ | Bearer Token 鉴权 | 无 | ✅ | ❌ | ✅ |
| `hideErrors` / `hide_errors` | ❌ | ✅ | 无 | 是否隐藏错误面板 | ❌ | ✅ | ✅ |
| `highlight` | ❌ | ✅ | 无 | Block 颜色高亮 | ❌ | ✅ | ✅ |
| `refreshInterval` | — | — | — | — | — | — | ❌（硬编码 60s） |

### 10.6 返回数据到卡片内容的完整转化链

```
HA API 原始响应
     │
     ▼
┌─────────────────────────────────────────────────┐
│ proxy.js: output() 函数                         │
│                                                 │
│ Template 查询:                                  │
│   HA 返回纯文本 → { label, value: text }        │
│   例: { label: "homeassistant.people_home",     │
│         value: "2 / 4" }                        │
│                                                 │
│ State 查询:                                     │
│   HA 返回 JSON → formatOutput() 解析占位符      │
│   例: { label: "Living Room",                   │
│         value: "23.5 °C" }                      │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ HTTP 响应: res.status(200).send([...])          │
│ 返回 [{ label, value }, ...] 数组               │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ useWidgetAPI: SWR 接收 JSON                     │
│ data = [{ label, value }, ...]                  │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ component.jsx: data?.map(d =>                   │
│   <Block label={d.label} value={d.value} />     │
│ )                                               │
│ 每条数据生成一个 Block 子元素                     │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ Container: fields 过滤                           │
│                                                 │
│ if (fields && type) {                           │
│   visibleChildren = children.filter(child =>    │
│     fields.some(field => {                      │
│       fullField = field.includes(".")           │
│         ? field                                 │
│         : `${type}.${field}`                    │
│       return fullField === child.props.label    │
│     })                                          │
│   )                                             │
│ }                                               │
│                                                 │
│ 对于 HA widget:                                 │
│   fields: ["people_home"]                       │
│   → 匹配 "homeassistant.people_home"            │
│   → 只保留匹配的 Block                          │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ Block: 最终渲染                                  │
│ <div> value (翻译后的 label) </div>              │
│ label 通过 t() 翻译函数显示                      │
└─────────────────────────────────────────────────┘
```

### 10.6 Container 字段过滤的匹配规则

[Container](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/container.jsx#L34-L63) 的过滤逻辑核心：

```javascript
visibleChildren = childrenArray?.filter((child) =>
  fields.some((field) => {
    let fullField = field;
    if (!field.includes(".")) {
      fullField = `${type}.${field}`;
    }
    let matches = fullField === (child?.props?.field || child?.props?.label);
    // ...
    return matches;
  }),
);
```

**对于 Home Assistant widget 的匹配规则：**

| `fields` 中的值 | 自动补全为 | 匹配 `child.props.label` |
|-----------------|-----------|------------------------|
| `"people_home"` | `"homeassistant.people_home"` | `homeassistant.people_home` ✅ |
| `"homeassistant.people_home"` | `"homeassistant.people_home"` | `homeassistant.people_home` ✅ |
| `"lights_on"` | `"homeassistant.lights_on"` | `homeassistant.lights_on` ✅ |

**关键细节：**
- HA 的 `component.jsx` 创建 `<Block label={d.label} />` 时没有传 `field` prop
- 因此 Container 匹配时使用 `child.props.label`，即 proxy.js 返回的 `label` 值
- 如果 custom 查询的 label 是 `"Living Room"`，fields 中必须写完整的 `"homeassistant.Living Room"` 才能匹配——但这基本不可行，这就是互斥设计的根本原因

---

## 十一、错误与空值的传递路径

### 11.1 错误传递的完整链路

错误可以在多个层级产生，每层的传递方式不同：

```
┌──────────────────────────────────────────────────────────────┐
│ 1. 服务端: proxy.js                                          │
│                                                              │
│ 场景 A: group/service 缺失                                   │
│   → res.status(400).json({ error: "Invalid proxy service" })│
│                                                              │
│ 场景 B: widget 配置不存在                                    │
│   → res.status(400).json({ error: "Invalid proxy service" })│
│                                                              │
│ 场景 C: custom JSON 解析失败                                 │
│   → res.status(400).json({ error: "Error parsing widget..." })│
│                                                              │
│ 场景 D: HA API 请求失败 (httpProxy 返回 error)               │
│   → res.status(status).send(data)                           │
│   其中 data = { error: { message, url, rawError } }         │
│                                                              │
│ 场景 E: HA API 返回非 200 状态码                             │
│   → results.map 中: { label: status, value: data.toString() }│
│   注意: 这不会触发前端错误! 而是作为"正常数据"返回            │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. SWR + useWidgetAPI: 错误提取                              │
│                                                              │
│ return { data, error: data?.error ?? error }                 │
│                                                              │
│ 场景 A/B/C/D: HTTP 状态码非 200                              │
│   → SWR 的 fetcher 抛出错误                                  │
│   → error 非 null                                            │
│                                                              │
│ 场景 E: HTTP 200 但数据含非 200 状态码                       │
│   → data = [{ label: 404, value: "..." }]                   │
│   → error = null (SWR 认为请求成功)                          │
│   → 卡片会正常渲染，但显示异常的 label/value                  │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. component.jsx: 错误分支                                   │
│                                                              │
│ if (error) {                                                 │
│   return <Container service={service} error={error} />;      │
│ }                                                            │
│ → 有错误时不渲染任何数据 Block，只渲染 Error 组件             │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. Container: 错误展示决策                                   │
│                                                              │
│ if (error) {                                                 │
│   if (settings.hideErrors || service.widget.hide_errors) {   │
│     return null;  ← 全局或 widget 级别隐藏错误               │
│   }                                                          │
│   return <Error service={service} error={error} />;          │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘
```

### 11.2 错误的界面表现

[Error 组件](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/error.jsx) 的渲染逻辑：

```javascript
// 错误对象归一化
if (typeof error === "string") error = { message: error };
else if (typeof error === "number") error = { message: `Error ${error}` };
if (error?.data?.error) error = error.data.error;
```

最终在界面上显示一个可展开的红色详情面板，包含：
- API 错误消息（`error.message`）
- 请求 URL（`error.url`，已脱敏）
- 原始错误（`error.rawError`）
- 响应数据（`error.data`）

### 11.3 空值与加载状态

| data 状态 | Component / Block 行为 | 视觉效果 |
|-----------|------------------------|---------|
| `undefined`（SWR 首次加载中） | `data?.map` 不执行，Container 收到空 children | 容器存在但没有 Block，不会出现单个 Block 的脉冲动画 |
| `null` | `data?.map` 不执行 | 容器存在但没有 Block |
| `[]`（空数组） | map 无迭代 | 容器存在但没有 Block |
| `[{ label, value: undefined }]` | 会创建 value 为 `undefined` 的 Block | value 区域显示 `"-"`，整个 Block 有 `animate-pulse` 脉冲动画 |
| `[{ label, value: "" }]` | value 为空字符串 | value 区域显示空字符串（非 `"-"`） |

**注意**：[Block](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/components/services/widget/block.jsx#L41-L48) 的加载脉冲动画只在单个 Block 的 `value === undefined` 时触发；SWR 首次加载阶段通常还没有 `data` 数组，因此不会先渲染出占位 Block。`null` 值会显示 `"-"` 但没有脉冲效果：

```javascript
className={classNames(
  "bg-theme-200/50 ...",
  value === undefined ? "animate-pulse" : "",  // 脉冲动画
)}
// ...
<div className="font-thin text-sm">
  {value === undefined || value === null ? "-" : value}  // 占位符
</div>
```

### 11.4 服务端部分成功的特殊行为

[proxy.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/proxy.js#L91-L104) 的结果处理：

```javascript
const err = results.find((r) => r.result[2]?.error);
if (err) {
  return res.status(status).send(data);  // 全部失败
}

return res.status(200).send(
  results.map((r) => {
    const [status, , data] = r.result;
    return status === 200
      ? r.output(data)                         // 成功：正常 { label, value }
      : { label: status, value: data.toString() };  // 非200：label=状态码
  }),
);
```

**这意味着：**
- 3 个默认查询中，如果 1 个返回 404（实体不存在），但 `result[2]` 无 `error` 属性
- 整体请求仍返回 200
- 该查询在卡片上显示为 `label: 404, value: "Not Found"` 之类的文本
- 用户看到的是一个显示异常的 Block，而不是错误提示

---

## 十二、定时刷新、配置重载与 refreshInterval 的真实关系

### 12.1 核心澄清：refreshInterval 对 HA 组件是硬编码，不可配置

先给出明确结论，再分析两者的更新行为。

**代码核查结论：Home Assistant 卡片的刷新间隔当前版本**（代码在 [component.jsx#L9](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L9)）**中是硬编码的字面量 `60000`（60 秒），无法通过 YAML 配置改变。**

两条通路都堵死了：

1. **`cleanServiceGroups` 不为 HA 传递 `refreshInterval` 到前端**：
   - 白名单解构行 [service-helpers.js#L337-L338](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/utils/config/service-helpers.js#L337-L338) 注释写的是 `// glances, customapi, iframe, prometheusmetric`
   - 实际执行赋值的位置：`iframe`（#L532）、`glances`（#L603）、`customapi`（#L620）、`prometheusmetric`（#L679）——没有 `homeassistant` 分支
   - 所以即便在 YAML 里写了 `refreshInterval: 10000`，前端 widget 对象中也**没有** 这个属性

2. **HA 组件自身也不读取 `widget.refreshInterval`**：
   ```javascript
   // homeassistant/component.jsx 第 9 行
   useWidgetAPI(widget, null, { refreshInterval: 60000 })
   //                                        ^^^^^^^
   //                                   字面量 60000，不是 widget.refreshInterval
   ```

**与其他 widget 的对比**（说明缺失了什么逻辑）：

```
customapi 组件（可配置 refreshInterval）:
  const { mappings = [], refreshInterval = 10000, display = "block" } = widget;
  const { data } = useWidgetAPI(widget, null, {
    refreshInterval: Math.max(1000, refreshInterval),
  });

glances 组件（可配置 refreshInterval）:
  const { ..., refreshInterval = defaultInterval } = widget;
  useWidgetAPI(widget, "info", { refreshInterval, ... });

homeassistant 组件（不可配置）:
  useWidgetAPI(widget, null, { refreshInterval: 60000 });
  //                                        ↑ 硬编码
```

> 💡 **要让 refreshInterval 可配置**，需同时改动两处：
> 1. 在 `service-helpers.js` 的 `cleanServiceGroups` 中新增 `if (type === "homeassistant")` 分支，将 `refreshInterval` 赋到 widget 对象
> 2. 在 `homeassistant/component.jsx` 第 9 行改为 `useWidgetAPI(widget, null, { refreshInterval: widget.refreshInterval ?? 60000 })`

### 12.2 两种更新机制对比

| 维度 | 定时刷新 (SWR refreshInterval) | 配置重载 (hash + revalidate) |
|------|-------------------------------|------------------------------|
| **触发条件** | 每 60 秒自动触发 | 配置文件内容变更时触发 |
| **影响范围** | 单个 widget 的数据 | 整个页面 |
| **更新内容** | 重新请求 `/api/services/proxy` | 重新请求 `/api/services`（服务列表）+ 页面硬刷新 |
| **组件生命周期** | 不卸载，仅 data 更新 | 组件完全卸载并重新挂载 |
| **SWR 缓存** | 保留（stale-while-revalidate） | 清空（页面刷新后 SWR 缓存丢失） |
| **视觉反馈** | 无闪烁，数据静默替换 | 页面显示加载中 → 完整重新渲染 |
| **触发文件** | [component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/widgets/homeassistant/component.jsx#L9) | [index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/index.jsx#L110-L131) + [hash.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/api/hash.js) + [revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/api/revalidate.js) |

### 12.2 定时刷新的精确行为

```
SWR refreshInterval: 60000 (60秒)
     │
     ├─ 每 60 秒 SWR 自动发起 revalidate
     │   → fetch("/api/services/proxy?group=...&service=...&index=0")
     │
     ├─ 请求期间：
     │   ├─ 旧数据仍然显示（stale-while-revalidate）
     │   └─ 无加载指示器
     │
     ├─ 请求成功：
     │   ├─ data 更新 → Component re-render
     │   ├─ Block 的 value 更新
     │   └─ 无视觉闪烁（React diff 更新）
     │
     └─ 请求失败：
         ├─ error 变为非 null
         ├─ Container 切换到 Error 组件
         └─ 数据 Block 消失，显示错误面板
         （下次 60 秒后如果恢复，error 变 null，数据恢复显示）
```

**关键点：**
- SWR 的 `refreshInterval` 只在组件挂载后生效，触发的是同一个代理 URL 的重新验证
- 代理接口每次执行都会通过服务端配置重新定位 widget，所以 `custom`、`url`、`key` 以及服务端用于取舍查询的 `fields` 可能在下一次代理请求中被感知
- 前端已经拿到的 widget 对象不会因定时刷新而更新，因此 `fields` 的界面过滤、`hide_errors` 和高亮配置等前端行为仍要等服务列表重新获取或整页重载后才同步

### 12.3 配置重载的精确行为

```
配置文件修改 (services.yaml, settings.yaml 等)
     │
     ▼
hash.js 检测到文件内容变化
→ 计算新的 SHA256 hash
     │
     ▼
index.jsx 中的 useEffect 检测 hash 变化
→ localStorage 中旧 hash !== 新 hash
→ setStale(true) → 显示加载动画
→ fetch("/api/revalidate") → Next.js ISR 重新生成页面
→ window.location.reload() → 整页硬刷新
     │
     ▼
页面重新加载
→ getStaticProps 重新执行
→ servicesResponse() 重新获取（含 cleanServiceGroups）
→ 新的 SWR fallback 数据
→ 组件完全重新挂载
→ useWidgetAPI 重新发起首次请求
```

**关键点：**
- 配置重载是"全有或全无"的——无法只重载某个 widget
- `hash.js` 监控的文件列表：`docker.yaml`、`settings.yaml`、`services.yaml`、`bookmarks.yaml`、`widgets.yaml`、`custom.css`、`custom.js`
- 页面挂载和 SWR 默认重新验证会读取 `/api/hash`，窗口聚焦时还会显式调用 `mutateHash()` 再查一次
- 因此配置变更通常在页面初次加载、窗口重新聚焦或 SWR 重新验证 hash 时被发现，而不是 Home Assistant widget 的 60 秒数据刷新直接触发整页重载

### 12.4 两种刷新分别触发哪些更新

| 更新内容 | 定时刷新 (60s) | 配置重载 |
|---------|:---:|:---:|
| HA 实体状态值（如温度、开关数） | ✅ | ✅ |
| 服务端查询配置（`custom` / `url` / `key` 变更） | ✅ | ✅ |
| 服务端用于取舍 custom 的 `fields` 是否存在 | ✅ | ✅ |
| 前端 `fields` 过滤配置变更 | ❌ | ✅ |
| 前端 `hide_errors` 配置变更 | ❌ | ✅ |
| 前端 `highlight` 高亮配置变更 | ❌ | ✅ |
| 服务增减/重排 | ❌ | ✅ |
| 新增/删除 widget | ❌ | ✅ |
| **`refreshInterval` 变更** | ❌ | ❌（**完全不可配置**，见 12.1） |

**解释：**
- 定时刷新只重新调用 `/api/services/proxy`，服务端会重新读取原始服务配置，所以 `custom`、`url`、`key` 以及服务端判断 `fields` 是否存在的分支会在代理请求中更新
- 但前端的 widget 对象来自 `/api/services`，定时刷新不会重新获取服务列表；因此 `fields` 真正过滤哪些 Block、错误是否隐藏、高亮规则等前端行为只能通过服务列表刷新或整页重载同步
- 这造成了一个不对称：改 `custom` 可能在下一次代理刷新后生效；改 `fields` 可能先影响服务端查询取舍，但前端过滤规则要等配置重载后才和服务端一致
- **`refreshInterval` 特殊说明**：当前版本硬编码为 60 秒（第 12.1 节详述），改 YAML 的 `refreshInterval` 对 HA widget 没有任何效果，无论定时刷新还是配置重载都不能改变它

### 12.5 窗口聚焦触发的双重效果

当用户切回浏览器标签页时，两件事同时发生：

1. **SWR 的 `revalidateOnFocus`（默认开启）**：
   - 对 `/api/services/proxy` 发起 revalidate
   - 相当于一次定时刷新，只更新数据值

2. **`mutateHash()`**（[index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/199-homepage/src/pages/index.jsx#L104-L108) 手动调用）：
   - 重新请求 `/api/hash`
   - 如果 hash 变了，触发整页重载
   - 如果 hash 没变，无额外效果

所以窗口聚焦 = 数据刷新 + 配置变更检测，两者叠加确保了用户切回页面时总能看到最新状态。
