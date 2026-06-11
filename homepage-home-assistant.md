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

目前 60 秒是硬编码的，可以从 widget 配置中读取：

```javascript
// component.jsx
const refreshInterval = widget.refreshInterval || 60000;
const { data, error } = useWidgetAPI(widget, null, { refreshInterval });
```

### 9.3 WebSocket 实时更新

对于需要实时更新的场景，可以考虑使用 Home Assistant 的 WebSocket API 替代轮询，减少不必要的请求。
