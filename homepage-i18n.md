# Homepage i18n 多语言体系代码链路分析

## 一、技术栈概览

| 库 | 版本 | 作用 |
|---|---|---|
| `next-i18next` | ^16.0.7 | Next.js 的 i18n 集成库，封装 i18next / react-i18next |
| `react-i18next` | ^15.5.3 | React 绑定层，提供 `useTranslation` Hook 与 Context |
| `i18next` | ^26.3.0 | 核心国际化引擎，负责语言加载、插值、formatter 等 |

依赖声明：[package.json](package.json#L24-L38)

---

## 二、语言包加载机制

### 2.1 语言包目录结构

语言包以 JSON 格式存放于 `public/locales/{语言代码}/common.json`，共 40+ 种语言（含 `pt-BR`、`nb-NO` 等带连字符的 code）：

- [public/locales/en/common.json](public/locales/en/common.json) — 英文基准
- `public/locales/zh-Hans/common.json` — 简体中文
- `public/locales/fr/common.json` — 法语

每个 JSON 文件按命名空间（namespace）分组，全部使用 `common` 这一个 namespace：

```json
{
  "common":    { "bytes": "{{value, bytes}}", "number": "{{value, number}}", ... },
  "widget":    { "missing_type": "Missing Widget Type: {{type}}", ... },
  "resources": { "cpu": "CPU", "mem": "MEM", ... },
  "unifi":     { "uptime": "Uptime", ... },
  ...
}
```

### 2.2 静态配置（next-i18next.config.js）

[next-i18next.config.js](next-i18next.config.js) 定义初始化参数与自定义 formatter：

```javascript
module.exports = {
  i18n: {
    defaultLocale: "en",     // 默认语言（兜底用）
    locales: ["en"],         // ⚠️ 仅构建期最小声明，运行时不受此数组限制
  },
  serializeConfig: false,
  use: [
    {
      init: (i18next) => {
        // 注册 6 个自定义 formatter：bytes / rate / percent / date / relativeDate / duration
        i18next.services.formatter.add("bytes", (value, lng, options) => ...);
        // ...
      },
      type: "3rdParty",
    },
  ],
};
```

> 注意：`locales: ["en"]` 只给 Next.js 构建用，**运行时支持的语言完全取决于 `public/locales/` 下有哪些目录**，由 `serverSideTranslations(language)` 动态加载。

> **关于回退语言的重要澄清（分三层）**：
> 1. **i18next 原生默认值**：`fallbackLng: 'dev'` —— 这是 i18next 库的出厂默认，面向开发者调试
> 2. **next-i18next 包装层**：如果用户没在 config 中显式写 `fallbackLng`，next-i18next 会**自动**把 `config.i18n.defaultLocale` 的值作为 `fallbackLng` 传给 i18next（官方文档明确说明）
> 3. **homepage 项目运行时实际值**：`fallbackLng: "en"` —— 因为 `defaultLocale: "en"`，next-i18next 自动补上了
>
> 所以**项目实际有英文回退**，不是 `'dev'` 也不是"没配置"。单个 key 缺失时会自动去英文包里找。

[next.config.js](next.config.js#L1-L19) 将同一 i18n 配置注入 Next.js：

```javascript
const { i18n } = require("./next-i18next.config");
const nextConfig = { ..., i18n };
module.exports = nextConfig;
```

### 2.3 服务端加载（SSG / ISR 阶段）

首页的 `getStaticProps` 在**构建或重生成**时读取 `settings.yaml` 中的 `language` 字段，调用 `serverSideTranslations(language)` 加载对应语言包并注入 `pageProps`。

核心代码：[src/pages/index.jsx](src/pages/index.jsx#L45-L95)

```javascript
// 语言别名映射（向后兼容）
const LANGUAGE_ALIASES = { "zh-cn": "zh-Hans" };

const normalizeLanguage = (language) => {
  if (!language) return "en";
  const alias = LANGUAGE_ALIASES[language.toLowerCase()];
  return alias || language;
};

export async function getStaticProps() {
  try {
    const { providers, ...settings } = getSettings();
    const language = normalizeLanguage(settings.language);

    return {
      props: {
        initialSettings: settings,
        fallback: { /* SWR fallback data */ },
        ...(await serverSideTranslations(language)),  // 加载指定语言
      },
    };
  } catch (e) {
    // 兜底：任何异常（含语言包加载失败）都回退到英文
    return {
      props: {
        initialSettings: {},
        fallback: { /* 空数据 */ },
        ...(await serverSideTranslations("en")),     // 异常兜底语言
      },
    };
  }
}
```

`serverSideTranslations(language)` 的内部行为（来自 next-i18next 库）：
1. 读取 `public/locales/{language}/common.json`
2. 序列化为 `_nextI18Next` 对象注入 `pageProps`
3. 客户端水合时还原为 i18next 实例

### 2.4 客户端初始化（_app.jsx）

应用根组件用 `appWithTranslation` HOC 包装，创建全局 `I18nextProvider`：

[src/pages/_app.jsx](src/pages/_app.jsx#L73-L100)

```javascript
import { appWithTranslation } from "next-i18next/pages";
import nextI18nextConfig from "../../next-i18next.config";

function MyApp({ Component, pageProps }) {
  return (
    <SWRConfig ...>
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

`appWithTranslation` 的职责：
1. 创建全局 `I18nextProvider`，把服务端序列化的 `_nextI18Next` 还原为 i18next 实例
2. 应用配置中的自定义 formatter
3. 所有子组件通过 `useTranslation()` 访问 `t` / `i18n`

---

## 三、文案选择逻辑

### 3.1 基础用法：`useTranslation()` Hook

所有需要翻译的组件从 `next-i18next/pages` 导入 Hook：

```javascript
import { useTranslation } from "next-i18next/pages";

function Component() {
  const { t, i18n } = useTranslation();
  return <div>{t("resources.cpu")}</div>;
}
```

实际案例：
- [src/components/widgets/resources/cpu.jsx](src/components/widgets/resources/cpu.jsx#L8-L51) — 标签 + 数值格式化
- [src/widgets/unifi/component.jsx](src/widgets/unifi/component.jsx#L45-L49) — 插值 + 复合拼接

### 3.2 插值变量

```javascript
t("widget.missing_type", { type: "weatherapi" })
// → "Missing Widget Type: weatherapi"
```

对应语言包模板：`"missing_type": "Missing Widget Type: {{type}}"`

### 3.3 自定义 Formatter（数值 / 时间格式化）

这是项目最具特色的部分。在 [next-i18next.config.js](next-i18next.config.js#L118-L161) 注册了 6 个 formatter，允许在翻译 key 中直接格式化数值。全部基于 `Intl.NumberFormat` / `Intl.DateTimeFormat` / `Intl.RelativeTimeFormat`，天然支持 locale 适配。

| Formatter | 语言包 Key 示例 | 组件调用示例 | 输出 |
|---|---|---|---|
| `bytes` | `common.bytes: "{{value, bytes}}"` | `t("common.bytes", { value: 1024 })` | `1.024 kB`（按 locale 千分位） |
| `rate` | `common.byterate: "{{value, rate(bits: false)}}"` | `t("common.byterate", { value: 5000 })` | `5 kB/s` |
| `percent` | `common.percent: "{{value, percent}}"` | `t("common.percent", { value: 42 })` | `42%` |
| `number` | `common.number: "{{value, number}}"` | `t("common.number", { value: 1234.5, maximumFractionDigits: 1 })` | `1,234.5` |
| `date` | `common.date: "{{value, date}}"` | `t("common.date", { value: ts })` | 本地化日期字符串 |
| `relativeDate` | `common.relativeDate: "{{value, relativeDate}}"` | `t("common.relativeDate", { value: iso })` | `in 3 days` / `2 hours ago` |
| `duration` | `common.duration: "{{value, duration}}"` | `t("common.duration", { value: 3661 })` | `1h1m1s`（递归调用 `t("common.hours")` 等） |

实际使用示例（[src/components/widgets/resources/uptime.jsx](src/components/widgets/resources/uptime.jsx#L25-L32)）：

```javascript
<Resource
  value={t("common.duration", { value: data.uptime })}  // 秒数 → "2d5h"
  label={t("resources.uptime")}
  percentage={percent}
/>
```

### 3.4 隐式翻译：Block 组件的 `label` prop

通用展示组件 [src/components/services/widget/block.jsx](src/components/services/widget/block.jsx#L9-L55) 内部自动把 `label` 当作翻译 key 调用 `t()`：

```javascript
export default function Block({ value, label }) {
  const { t } = useTranslation();
  return (
    <div>
      <div>{value === undefined ? "-" : value}</div>
      <div>{t(label)}</div>  {/* label 直接传 key 字符串 */}
    </div>
  );
}
```

调用方（[src/widgets/unifi/component.jsx](src/widgets/unifi/component.jsx#L27-L34)）只需传 key：

```jsx
<Block label="unifi.uptime" />
<Block label="unifi.wan" value={wan.up ? t("unifi.up") : t("unifi.down")} />
```

---

## 四、无效语言 / 缺失 Key / 加载失败的真实表现

i18next 本身内置了回退链，配合 homepage 项目的代码层兜底，构成完整的容错策略。以下是每层的真实行为，**区分 i18next 原生默认值、next-i18next 包装层的修正、以及 homepage 项目运行时实际值**。

### 先理清：三层配置的叠加关系

| 配置项 | i18next 原生默认 | next-i18next 包装层处理 | homepage 运行时实际值 | 作用 |
|---|---|---|---|---|
| `fallbackLng` | `'dev'` | **自动覆盖为 `defaultLocale`**（如果用户没显式写） | **`"en"`** ✅ | 所有语言变体都找不到 key 时的终极回退语言 |
| `load` | `'all'` | 不修改 | `'all'` | 加载策略：`'all'` 触发 variant resolving，同时加载变体+纯语言+fallbackLng |
| `returnEmptyString` | `true` | 不修改 | `true` | key 找不到且所有回退都落空时，返回 key 本身（非空字符串） |

> **关键纠正**：之前的分析错误地将 i18next 原生默认的 `'dev'` 当成了 homepage 的实际运行值。实际上 next-i18next 作为包装层，会在创建 i18next 实例时，**自动把 `config.i18n.defaultLocale` 赋给 `fallbackLng`**（除非用户显式覆盖）。这是 next-i18next 的设计行为，官方文档明确说明。
>
> 对 homepage 而言，`defaultLocale: "en"` → 运行时 `fallbackLng: "en"` → **单个 key 缺失时会自动回退到英文**。

对 `load: 'all'` + `fallbackLng: "en"` 组合的具体查找顺序（来自 i18next 官方文档）：
- 设置 `lng: "zh-Hans"` → 依次查找 `["zh-Hans", "zh", "en"]`
- 设置 `lng: "pt-BR"` → 依次查找 `["pt-BR", "pt", "en"]`
- 设置 `lng: "en"`（纯语言无变体） → 查找 `["en"]`
- 设置 `lng: "xx-YY"`（完全不存在） → 尝试 `["xx-YY", "xx", "en"]`，最后落在 `en`

其中 **variant resolving（变体解析）**是 i18next 的隐式回退：如果 `zh-Hans/common.json` 缺少某 key 但 `zh/common.json` 里有，自动补位；`zh` 里也没有，再去 `en` 里找。homepage 项目如果存在 `zh`、`pt` 等纯语言目录，这就是**"变体 → 纯语言 → 英文"**的三级回退。

---

### 4.1 场景一：settings.yaml 中 language 为空或未配置

**代码位置**：[src/pages/index.jsx](src/pages/index.jsx#L49-L53)

```javascript
const normalizeLanguage = (language) => {
  if (!language) return "en";          // 空值 → 英文
  const alias = LANGUAGE_ALIASES[language.toLowerCase()];
  return alias || language;
};
```

**真实表现**（代码层兜底，第一层防线）：
- `language` 未写、为 `null`、为空字符串 → `normalizeLanguage` 返回 `"en"`，`serverSideTranslations("en")` 加载英文语言包
- `language: "zh-CN"`（旧写法）→ 自动映射为 `"zh-Hans"`，加载简体中文
- 其他任何非空字符串 → 原样传递给 `serverSideTranslations`，进入 i18next 内置查找链

### 4.2 场景二：指定的语言目录不存在（如 `language: "xx-YY"`，完全无效语言）

**代码路径**：`getStaticProps` → `normalizeLanguage` 返回 `"xx-YY"` → `serverSideTranslations("xx-YY")` → next-i18next 在服务端读文件系统

**真实表现**（两条分支，最终都能显示英文）：

| 分支 | 触发条件 | 结果 |
|---|---|---|
| **分支 A（抛异常）** | `serverSideTranslations` 因文件不存在、JSON 语法错误、权限不足等**抛出同步/Promise 异常** | 被 `getStaticProps` 的 catch 块（[src/pages/index.jsx](src/pages/index.jsx#L101-L110)）捕获 → `initialSettings` 置空 → 调用 `serverSideTranslations("en")` → 页面正常加载英文 |
| **分支 B（不抛异常）** | `serverSideTranslations` 内部吞掉文件缺失、只在控制台告警不抛错（next-i18next 常见实际行为） | `_nextI18Next` 注入到客户端，`xx-YY` 和 `xx` 资源为空，但 `fallbackLng: "en"` 资源被加载 → 所有 `t(key)` 先查 `xx-YY` → 落空 → 查 `xx` → 落空 → **查 `en` 命中** → 显示英文翻译 |

> **注意修正**：之前的分析错误地认为分支 B 会显示裸 key。实际上由于 `fallbackLng: "en"` 会被自动加载（`load: 'all'` 策略），即使目标语言完全不存在，最终也会落在英文翻译上，**不会显示裸 key**。

**测试验证**：[src/__tests__/pages/index.test.jsx](src/__tests__/pages/index.test.jsx#L208-L221) 中 `throwIn = "services"` 模拟异常后，验证 `serverSideTranslations` 最终被调用时传入 `"en"`。

由于英文是项目基准语言且 `public/locales/en/common.json` 一定存在，`serverSideTranslations("en")` 本身不会失败。

### 4.3 场景三：语言包正常加载，但某个 key 在该语言中缺失（最常见情况）

**i18next 查找链**（以 `language: "zh-Hans"`，调用 `t("some.new.key")` 为例）：

```
  t("some.new.key")
       │
       ▼
 ① 查 zh-Hans 资源 → 有？→ 返回翻译值（结束）
       │ 没有
       ▼
 ② variant resolving：查 zh 资源（纯语言）→ 有？→ 返回翻译值（结束）
       │ 没有（或 public/locales/zh/ 目录根本不存在）
       ▼
 ③ 查 fallbackLng = "en" 资源 → 有？→ 返回英文翻译（结束）✅
       │ 没有（英文基准里也缺这个 key，极罕见）
       ▼
 ④ returnEmptyString: true → 返回 key 本身 "some.new.key"（界面显示裸 key）
```

**真实表现**：
- **不报错、不抛异常、不崩溃**，组件正常渲染
- **会自动回退到英文**（因为 `fallbackLng: "en"`，next-i18next 自动设置）
- 变体级也有隐式回退（如 `zh-Hans` 缺 key → `zh` 补），但要求纯语言目录 `zh/` 同时存在
- 只有当**英文基准里也缺这个 key**时（极罕见，通常是新增功能还没来得及写英文文案），才会显示裸 key

> 项目中所有组件都没有给 `t()` 传 `defaultValue` 第二个参数，也没有使用 `t([key, fallbackKey])` 数组语法提供 fallback key，因此缺失 key 的表现严格遵循上述查找链。

### 4.4 场景四：客户端 `i18n.changeLanguage(lng)` 目标语言加载失败

**代码位置**：[src/pages/index.jsx](src/pages/index.jsx#L234-L247)

```javascript
useEffect(() => {
  const language = normalizeLanguage(settings.language);
  if (language) {
    i18n.changeLanguage(language);
  }
}, [i18n, settings, ...]);
```

**真实表现**：
- `i18n.changeLanguage()` 是异步操作，**不会抛出同步异常**
- 若目标语言 JSON 拉取失败（HTTP 404、网络错误等），i18next 内部触发 `failedLoading` 事件，应用层面不会感知到异常
- **当前语言保持不变**，所有 `t()` 调用继续使用切换前的语言翻译，用户不会看到断档或空白
- 组件不会白屏，也不会报错崩溃

### 4.5 兜底链路总览（按执行顺序 + 四层回退）

```
  settings.language 未配置 / 为空？
        │ 是
        ▼
  normalizeLanguage() 返回 "en"  ───────────────────┐
        │ 否                                        │ （代码层兜底）
        ▼                                           ▼
  serverSideTranslations(language)          serverSideTranslations("en")
        │
        ├── 抛异常？
        │     是
        │     ▼
        │   catch 捕获 ──────────────────────────► serverSideTranslations("en")
        │                                               │
        │     否（正常返回，目标语言资源可能为空）       │ 英文基准，一定成功
        ▼                                               ▼
  客户端水合 i18n 实例                         客户端水合 i18n 实例
 （自动加载 fallbackLng: "en" 资源）                  │
        │                                               ▼
        ▼                                       t(key) 全部命中英文
  运行时 t(key)
        │
        ▼
  i18next 内置四层查找链：
    ① 当前变体（如 zh-Hans） → 命中？→ 返回翻译
    ② 纯语言变体（如 zh）   → 命中？→ 返回翻译  （variant resolving，隐式生效）
    ③ fallbackLng = "en"   → 命中？→ 返回英文  （next-i18next 自动设置 ✅）
    ④ 全部落空 → 返回 key 本身（裸 key，仅当英文基准也缺时）
```

**哪些异常最终会显示英文？**
- ✅ language 为空/未配置 → `normalizeLanguage` → 英文
- ✅ `serverSideTranslations` 抛出异常 → catch 块 → 英文
- ✅ 单个翻译 key 在所选语言中缺失 → `fallbackLng: "en"` 自动回退 → 英文 **（之前错误，已修正）**
- ✅ 语言目录不存在但 next-i18next 不抛错 → `fallbackLng: "en"` 自动回退 → 英文 **（之前错误，已修正）**
- ❌ `changeLanguage` 目标语言加载失败 → 保持切换前语言，不显示英文（也不崩溃）
- ❌ 英文基准本身也缺这个 key → 显示裸 key（极罕见）

---

## 五、语言切换与刷新机制（职责边界）

项目中存在**两套**改变语言生效的机制，分别解决不同层级的问题。两者职责明确、互不替代，共同组成完整的语言更新链路。

### 5.1 机制 A：客户端运行时切换（`i18n.changeLanguage`）

**触发源**：`SettingsContext` 中的 `settings.language` 状态变化。

**代码位置**：[src/pages/index.jsx](src/pages/index.jsx#L214-L247)

```javascript
function Home({ initialSettings }) {
  const { i18n } = useTranslation();
  const { settings, setSettings } = useContext(SettingsContext);

  useEffect(() => {
    setSettings(initialSettings);
  }, [initialSettings, setSettings]);

  useEffect(() => {
    const language = normalizeLanguage(settings.language);
    if (language) {
      i18n.changeLanguage(language);   // 客户端热切换
    }
  }, [i18n, settings, ...]);
}
```

**特点**：
| 维度 | 说明 |
|---|---|
| 触发方式 | React state 变化 → useEffect 回调 |
| 是否刷新页面 | 否，纯客户端重渲染 |
| 速度 | 快（毫秒级，仅重新渲染用了 `t()` 的组件） |
| 影响范围 | 仅浏览器端 i18n 实例 + React 组件树 |
| 语言包来源 | 客户端异步加载 `public/locales/*.json`（HTTP 请求） |
| 对 SSR / SEO 生效 | 否（仅客户端 DOM 更新，服务端 HTML 不变） |
| 谁能触发 | 同一次页面会话内的 settings 状态变化 |

**适用场景**：
1. 页面首次加载时，`Home` 组件把 `initialSettings` 写入 `SettingsContext`，触发一次 `changeLanguage`，确保服务端注入的语言与客户端 i18n 实例一致
2. 同一次页面会话内，如果前端 UI（如语言下拉框）直接修改了内存中的 `settings.language`，无需刷新页面即可切换

**注意**：这套机制**不会**修改服务端 HTML。如果爬虫或首屏访问，看到的语言取决于最近一次 ISR 生成的静态页面。

### 5.2 机制 B：配置文件变更 → ISR 重生成 → 整页刷新

**触发源**：服务器上的配置文件（`settings.yaml` 等 7 个文件）内容变化。

**完整链路**（三段式）：

**第一段：hash 检测（客户端）** — [src/pages/index.jsx](src/pages/index.jsx#L100-L131)

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

// 窗口获得焦点时重新拉取
useEffect(() => {
  if (windowFocused) mutateHash();
}, [windowFocused, mutateHash]);

// hash 变化 → 触发 revalidate + reload
useEffect(() => {
  if (hashData && typeof window !== "undefined") {
    const previousHash = localStorage.getItem("hash");
    if (!previousHash) {
      localStorage.setItem("hash", hashData.hash);
    }
    if (previousHash && previousHash !== hashData.hash) {
      localStorage.setItem("hash", hashData.hash);
      fetch("/api/revalidate").then((res) => {
        if (res.ok) window.location.reload();
      });
    }
  }
}, [hashData]);
```

**第二段：hash API（服务端）** — [src/pages/api/hash.js](src/pages/api/hash.js)

计算所有配置文件的 SHA256 合并 hash，外加构建时间戳：

```javascript
const configs = [
  "docker.yaml", "settings.yaml", "services.yaml",
  "bookmarks.yaml", "widgets.yaml", "custom.css", "custom.js",
];

export default async function handler(req, res) {
  const hashes = configs.map((config) => {
    checkAndCopyConfig(config);
    return hash(readFileSync(join(CONF_DIR, config), "utf8"));
  });
  const buildTime = process.env.HOMEPAGE_BUILDTIME || "";
  const combinedHash = hash(hashes.join("") + buildTime);
  res.send({ hash: combinedHash });
}
```

**第三段：revalidate API（服务端）** — [src/pages/api/revalidate.js](src/pages/api/revalidate.js)

```javascript
export default async function handler(req, res) {
  try {
    await res.revalidate("/");     // Next.js ISR：重新执行 getStaticProps
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}
```

`res.revalidate("/")` 会在后台重新调用 `getStaticProps()`，从而以最新 `settings.yaml` 的 `language` 重新调用 `serverSideTranslations(language)`，生成新的静态 HTML 与 pageProps。

**特点**：
| 维度 | 说明 |
|---|---|
| 触发方式 | 文件系统变化 → hash 变更 → ISR → 整页 reload |
| 是否刷新页面 | 是，`window.location.reload()` 全量刷新 |
| 速度 | 慢（秒级，涉及服务端重生成 + 页面重载） |
| 影响范围 | 服务端页面静态资源 + 客户端整页 |
| 语言包来源 | 服务端 `getStaticProps` 注入 `_nextI18Next` |
| 对 SSR / SEO 生效 | 是（服务端 HTML 已使用新语言渲染） |
| 谁能触发 | 服务器配置文件变更，对所有访问者生效 |

**适用场景**：用户在服务器上编辑了 `settings.yaml` 修改 `language`（或其他任何配置），页面在用户下次聚焦窗口时自动检测到 hash 变化，触发服务端重生成并全量刷新。

### 5.3 两者的职责边界对比

| 维度 | 运行时切换（`changeLanguage`） | 配置变更刷新（hash + ISR + reload） |
|---|---|---|
| **解决的问题** | 同一会话内 settings 状态变化时的实时翻译切换 | 服务器配置文件变化时的全局同步 |
| **数据来源** | `SettingsContext`（内存中的 settings 对象） | `config/settings.yaml`（文件系统） |
| **语言包加载位置** | 客户端异步加载 `public/locales/*.json` | 服务端 `getStaticProps` 注入 `_nextI18Next` |
| **是否经过 SSG 重跑** | 否 | 是（ISR 重新执行 `getStaticProps`） |
| **对 SSR / SEO 生效** | 否（仅客户端 DOM 更新） | 是（服务端 HTML 已用新语言渲染） |
| **刷新成本** | 低，仅 React 重渲染 | 高，整页重载 + 服务端重生成 |
| **用户感知** | 无感 | 有加载 spinner |
| **触发频率** | 每次 `settings.language` 引用变化（含首次挂载） | 仅在配置文件内容 hash 变化时 |

### 5.4 两者如何配合

对于"修改 `settings.yaml` 中的 `language` 字段"这一场景，完整链路是：

```
  用户编辑 settings.yaml（服务器文件）
        │
        ▼
  下次窗口聚焦 → /api/hash 返回新 hash
        │
        ▼
  客户端对比 localStorage 中旧 hash → 不一致
        │
        ├── ① 调用 /api/revalidate
        │      │
        │      ▼
        │   res.revalidate("/") → ISR 重跑 getStaticProps
        │      │
        │      └─ 读取新 settings.language → serverSideTranslations(newLang)
        │         生成新的静态 HTML 和 pageProps
        │
        ▼
  ② window.location.reload()  ←──────  全量刷新
        │
        ▼
  新页面加载 → 新语言已在服务端注入 _nextI18Next
        │
        ▼
  appWithTranslation 水合 → i18n 实例使用新语言
        │
        ▼
  Home 组件挂载 → setSettings(initialSettings)
        │
        ▼
  useEffect 触发 changeLanguage(newLang)
        │
        ▼
  客户端 i18n 与服务端语言确认一致
```

> **为什么有了 ISR + reload 还需要 `changeLanguage`？**
>
> 1. **会话内状态同步**：`changeLanguage` 解决的是**同一次页面会话内** settings 状态变化时的实时响应。如果未来有前端 UI（如下拉框）直接修改内存中的 `settings`，不涉及服务器文件，就走 `changeLanguage` 路径直接生效。
> 2. **水合后二次确认**：页面首次加载时，`Home` 组件把服务端注入的 `initialSettings` 写入 `SettingsContext`，这会触发一次 `changeLanguage`，确保客户端 i18n 实例的语言与服务端注入的 pageProps 语言严格一致，消除水合期间可能产生的偏差。
> 3. **职责分层**：配置文件修改是**跨会话、跨用户**的全局变更，必须走 ISR + reload 才能保证服务端输出与客户端一致；而内存状态变化是**单次会话内**的局部变更，用 `changeLanguage` 即可低成本完成。

---

## 六、完整链路时序图

```
          ┌──────────────┐   读 YAML   ┌──────────────────┐
          │ settings.yaml│◄───────────┤ getSettings()     │
          └──────────────┘             └────────┬─────────┘
                                                 │ language = "fr"
                                                 ▼
                             ┌──────────────────────────────────┐
                             │ getStaticProps()                 │
                             │   serverSideTranslations("fr")   │──► 读 public/locales/fr/common.json
                             └─────────────────┬────────────────┘
                                                 │ pageProps._nextI18Next
                                                 ▼
                             ┌──────────────────────────────────┐
                             │ appWithTranslation(MyApp)        │
                             │   → I18nextProvider              │
                             └─────────────────┬────────────────┘
                                                 │ i18n 实例
            ┌────────────────────────────────────┼────────────────────────────────────┐
            │                                    ▼                                    │
            │    useTranslation() ───► t("key") / i18n.changeLanguage(lng)            │
            │                                                                         │
            │   ┌───────────────┐       ┌─────────────────┐       ┌───────────────┐  │
            │   │  Widget CPU   │       │  Widget Unifi   │       │  Block 组件   │  │
            │   │ t("resources.cpu")    │ t("unifi.uptime")│       │ t(label)      │  │
            │   └───────────────┘       └─────────────────┘       └───────────────┘  │
            └─────────────────────────────────────────────────────────────────────────┘

配置变更触发刷新（全局）：
  用户编辑 settings.yaml
        │
        ▼
  下次窗口聚焦 → /api/hash 返回新 hash
        │
        ▼
  对比 localStorage 中旧 hash → 不一致
        │
        ▼
  调用 /api/revalidate  →  res.revalidate("/") → getStaticProps() 重跑（加载新语言包）
        │
        ▼
  window.location.reload()  ←  整页刷新
```

---

## 七、单元测试中的 Mock

[vitest.setup.js](vitest.setup.js#L11-L32) Mock 了 `next-i18next/pages`，让组件单测无需加载真实翻译文件：

```javascript
vi.mock("next-i18next/pages", () => ({
  appWithTranslation: (Component) => Component,
  useTranslation: () => ({
    i18n: { language: "en" },
    t: (key, opts) => {
      // 对 formatter key 返回 opts.value，其余返回 key 本身
      if (key === "common.number") return String(opts?.value ?? "");
      if (key === "common.percent") return String(opts?.value ?? "");
      if (key === "common.bytes") return String(opts?.value ?? "");
      // ...
      return key;
    },
  }),
}));
```

针对 index 页面的测试（[src/__tests__/pages/index.test.jsx](src/__tests__/pages/index.test.jsx#L486-L517)）单独 mock 了 `changeLanguage` 以验证调用：

```javascript
const i18n = { language: "en", changeLanguage: vi.fn() };

// ... 设置新 settings 后
expect(i18n.changeLanguage).toHaveBeenCalledWith("en");
```

异常回退的测试（[src/__tests__/pages/index.test.jsx](src/__tests__/pages/index.test.jsx#L208-L221)）模拟 `servicesResponse` 抛错，验证：

```javascript
expect(serverSideTranslations).toHaveBeenCalledWith("en");
expect(logger.error).toHaveBeenCalled();
```

---

## 八、关键代码文件索引

| 文件 | 作用 |
|---|---|
| [next-i18next.config.js](next-i18next.config.js) | i18n 主配置 + 6 个自定义 formatter |
| [next.config.js](next.config.js) | 注入 i18n 到 Next.js |
| [src/pages/_app.jsx](src/pages/_app.jsx) | `appWithTranslation` 包装应用根组件 |
| [src/pages/index.jsx](src/pages/index.jsx) | `getStaticProps` 加载语言 + `normalizeLanguage` 兜底 + `changeLanguage` 切换 + hash/revalidate 刷新 |
| [src/pages/api/hash.js](src/pages/api/hash.js) | 7 个配置文件的 SHA256 合并 hash 计算接口 |
| [src/pages/api/revalidate.js](src/pages/api/revalidate.js) | ISR 重生成接口（触发 `getStaticProps` 重跑） |
| [src/utils/config/config.js](src/utils/config/config.js) | 读取 `settings.yaml`（含 `language` 字段） |
| [src/utils/contexts/settings.jsx](src/utils/contexts/settings.jsx) | `SettingsContext` 存储 settings 状态 |
| [src/components/services/widget/block.jsx](src/components/services/widget/block.jsx) | Block 组件隐式翻译 `label` prop |
| [public/locales/en/common.json](public/locales/en/common.json) | 英文翻译基准（所有兜底的最终锚点） |
| [vitest.setup.js](vitest.setup.js) | 测试环境 i18n Mock |
| [src/__tests__/pages/index.test.jsx](src/__tests__/pages/index.test.jsx) | `getStaticProps` 异常回退测试 + `changeLanguage` 调用测试 |
| [docs/configs/settings.md](docs/configs/settings.md#L443-L456) | 语言配置用户文档 |
| [docs/more/translations.md](docs/more/translations.md) | 翻译贡献指南 |
