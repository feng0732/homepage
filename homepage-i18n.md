# Homepage i18n 多语言体系代码链路分析

## 一、技术栈概览

| 库 | 版本 | 作用 |
|---|---|---|
| `next-i18next` | ^16.0.7 | Next.js 的 i18n 集成库，封装 i18next / react-i18next |
| `react-i18next` | ^15.5.3 | React 绑定层，提供 `useTranslation` Hook 与 Context |
| `i18next` | ^26.3.0 | 核心国际化引擎，负责语言加载、插值、formatter 等 |

依赖声明：[package.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/package.json#L24-L38)

---

## 二、语言包加载机制

### 2.1 语言包目录结构

语言包以 JSON 格式存放于 `public/locales/{语言代码}/common.json`，共 40+ 种语言（含 `pt-BR`、`nb-NO` 等带连字符的 code）：

- [public/locales/en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/public/locales/en/common.json) — 英文基准
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

[next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js) 定义初始化参数与自定义 formatter：

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

[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next.config.js#L1-L19) 将同一 i18n 配置注入 Next.js：

```javascript
const { i18n } = require("./next-i18next.config");
const nextConfig = { ..., i18n };
module.exports = nextConfig;
```

### 2.3 服务端加载（SSG / ISR 阶段）

首页的 `getStaticProps` 在**构建或重生成**时读取 `settings.yaml` 中的 `language` 字段，调用 `serverSideTranslations(language)` 加载对应语言包并注入 `pageProps`。

核心代码：[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L45-L95)

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
    // 兜底：加载失败时用英文
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

`serverSideTranslations(language)` 的内部行为：
1. 读取 `public/locales/{language}/common.json`
2. 序列化为 `_nextI18Next` 对象注入 `pageProps`
3. 客户端水合时还原为 i18next 实例

### 2.4 客户端初始化（_app.jsx）

应用根组件用 `appWithTranslation` HOC 包装，创建全局 `I18nextProvider`：

[src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/_app.jsx#L73-L100)

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
- [src/components/widgets/resources/cpu.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/widgets/resources/cpu.jsx#L8-L51) — 标签 + 数值格式化
- [src/widgets/unifi/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/widgets/unifi/component.jsx#L45-L49) — 插值 + 复合拼接

### 3.2 插值变量

```javascript
t("widget.missing_type", { type: "weatherapi" })
// → "Missing Widget Type: weatherapi"
```

对应语言包模板：`"missing_type": "Missing Widget Type: {{type}}"`

### 3.3 自定义 Formatter（数值 / 时间格式化）

这是项目最具特色的部分。在 [next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js#L118-L161) 注册了 6 个 formatter，允许在翻译 key 中直接格式化数值。全部基于 `Intl.NumberFormat` / `Intl.DateTimeFormat` / `Intl.RelativeTimeFormat`，天然支持 locale 适配。

| Formatter | 语言包 Key 示例 | 组件调用示例 | 输出 |
|---|---|---|---|
| `bytes` | `common.bytes: "{{value, bytes}}"` | `t("common.bytes", { value: 1024 })` | `1.024 kB`（按 locale 千分位） |
| `rate` | `common.byterate: "{{value, rate(bits: false)}}"` | `t("common.byterate", { value: 5000 })` | `5 kB/s` |
| `percent` | `common.percent: "{{value, percent}}"` | `t("common.percent", { value: 42 })` | `42%` |
| `number` | `common.number: "{{value, number}}"` | `t("common.number", { value: 1234.5, maximumFractionDigits: 1 })` | `1,234.5` |
| `date` | `common.date: "{{value, date}}"` | `t("common.date", { value: ts })` | 本地化日期字符串 |
| `relativeDate` | `common.relativeDate: "{{value, relativeDate}}"` | `t("common.relativeDate", { value: iso })` | `in 3 days` / `2 hours ago` |
| `duration` | `common.duration: "{{value, duration}}"` | `t("common.duration", { value: 3661 })` | `1h1m1s`（递归调用 `t("common.hours")` 等） |

实际使用示例（[src/components/widgets/resources/uptime.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/widgets/resources/uptime.jsx#L25-L32)）：

```javascript
<Resource
  value={t("common.duration", { value: data.uptime })}  // 秒数 → "2d5h"
  label={t("resources.uptime")}
  percentage={percent}
/>
```

### 3.4 隐式翻译：Block 组件的 `label` prop

通用展示组件 [src/components/services/widget/block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/services/widget/block.jsx#L9-L55) 内部自动把 `label` 当作翻译 key 调用 `t()`：

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

调用方（[src/widgets/unifi/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/widgets/unifi/component.jsx#L27-L34)）只需传 key：

```jsx
<Block label="unifi.uptime" />
<Block label="unifi.wan" value={wan.up ? t("unifi.up") : t("unifi.down")} />
```

---

## 四、无效 / 缺失语言的错误恢复机制

项目在**四个层级**设置了兜底策略，确保任何异常情况下都不会显示裸 key 或白屏。

### 4.1 第一层：配置读取前的空值兜底（normalizeLanguage）

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L49-L53)

```javascript
const normalizeLanguage = (language) => {
  if (!language) return "en";          // 空值 → 英文
  const alias = LANGUAGE_ALIASES[language.toLowerCase()];
  return alias || language;
};
```

兜底场景：
- `settings.yaml` 中未配置 `language` 字段
- `language` 设为 `null` / 空字符串
- 旧版配置值如 `zh-CN` → 自动映射为 `zh-Hans`

### 4.2 第二层：getStaticProps 异常兜底

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L78-L94)

```javascript
try {
  const { providers, ...settings } = getSettings();
  const language = normalizeLanguage(settings.language);
  return { props: { ..., ...(await serverSideTranslations(language)) } };
} catch (e) {
  return {
    props: {
      initialSettings: {},
      fallback: { /* 空数据 */ },
      ...(await serverSideTranslations("en")),   // 任何异常都回退到英文
    },
  };
}
```

兜底场景：
- `getSettings()` 抛错（配置文件损坏、权限不足等）
- `servicesResponse()` / `bookmarksResponse()` / `widgetsResponse()` 抛错
- `serverSideTranslations(language)` 抛错（语言目录不存在、JSON 损坏等）

> 注意：catch 块用 `"en"` 硬编码调用 `serverSideTranslations("en")`，依赖 `public/locales/en/common.json` **必须存在且有效**（英文作为基准语言始终完整）。

### 4.3 第三层：i18next 运行时 fallback

虽然项目未显式配置 `fallbackLng`，但 i18next 默认行为 + next-i18next 的包装提供了隐性兜底：

- `defaultLocale: "en"` 作为初始化语言
- 当 `i18n.changeLanguage(lng)` 目标语言包加载失败时，i18next 会保留当前语言，不会报错崩溃
- 缺失的翻译 key 会显示 key 本身（如 `some.missing.key`），这是 i18next 默认行为，不会导致组件渲染异常

### 4.4 第四层：客户端运行时语言切换的防御

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L234-L247)

```javascript
useEffect(() => {
  const language = normalizeLanguage(settings.language);
  if (language) {
    i18n.changeLanguage(language);   // 有值才切换，空值跳过
  }
  // ...
}, [i18n, settings, ...]);
```

- `normalizeLanguage` 确保传入 `changeLanguage` 的一定是非空字符串
- `i18n.changeLanguage()` 本身是幂等且安全的：语言不存在时只会异步加载失败，不会抛出同步异常

### 兜底链路总览

```
settings.language 为空？
    │ 是
    ▼
normalizeLanguage() → "en" ────────────────────┐
    │ 否                                       │
    ▼                                          │
serverSideTranslations(language) 正常？        │
    │ 否                                       │
    ▼                                          │
catch 块 → serverSideTranslations("en")        │
    │ 是                                       │
    ▼                                          │
客户端 i18n.changeLanguage(language) 失败？    │
    │ 是（网络/路径问题）                      │
    ▼                                          │
i18next 保留当前语言，不报错                    │
                                               │
          最终兜底：英文（en） ◄────────────────┘
```

---

## 五、语言切换与刷新机制（职责边界）

项目中存在**两套**改变语言生效的机制，分别解决不同层级的问题。两者职责明确、互不替代，共同组成完整的语言更新链路。

### 5.1 机制 A：客户端运行时切换（i18n.changeLanguage）

**触发源**：`SettingsContext` 中的 `settings.language` 变化。

**代码位置**：[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L214-L247)

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
| 谁能触发 | 同一次页面会话内的 settings 状态变化 |

**适用场景**：同一次页面加载中，settings 状态变化时的无感切换。例如未来若在前端 UI 上加一个语言切换下拉框，直接改 `settings.language` 就能实时生效。

### 5.2 机制 B：配置文件变更 → ISR 重生成 → 整页刷新

**触发源**：服务器上的配置文件（`settings.yaml` 等）内容变化。

**完整链路**（三段式）：

**第一段：hash 检测** — [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L100-L131)

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

**第二段：hash API** — [src/pages/api/hash.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/api/hash.js)

计算所有配置文件的 SHA256 合并 hash：

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
  const combinedHash = hash(hashes.join("") + buildTime);
  res.send({ hash: combinedHash });
}
```

**第三段：revalidate API** — [src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/api/revalidate.js)

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

**特点**：
| 维度 | 说明 |
|---|---|
| 触发方式 | 文件系统变化 → hash 变更 → ISR → 整页 reload |
| 是否刷新页面 | 是，`window.location.reload()` 全量刷新 |
| 速度 | 慢（秒级，涉及服务端重生成 + 页面重载） |
| 影响范围 | 服务端页面静态资源 + 客户端整页 |
| 谁能触发 | 服务器配置文件变更，对所有访问者生效 |

**适用场景**：用户在服务器上编辑了 `settings.yaml` 修改语言（或其他任何配置），页面在用户下次聚焦时自动检测并全量刷新。

### 5.3 两者的职责边界对比

| 维度 | 运行时切换（changeLanguage） | 配置变更刷新（hash + ISR + reload） |
|---|---|---|
| **解决的问题** | 客户端状态变化时的实时翻译切换 | 服务器配置文件变化时的全量同步 |
| **数据来源** | `SettingsContext`（内存中的 settings 对象） | `config/settings.yaml` 文件系统 |
| **语言包加载位置** | 客户端异步加载 `public/locales/*.json` | 服务端 `getStaticProps` 注入 `_nextI18Next` |
| **是否经过 SSG 重跑** | 否 | 是（ISR 重新执行 `getStaticProps`） |
| **对 SSR / SEO 生效** | 否（仅客户端 DOM 更新） | 是（服务端 HTML 已使用新语言渲染） |
| **刷新成本** | 低，仅 React 重渲染 | 高，整页重载 + 服务端重生成 |
| **用户感知** | 无感 | 有加载动画（spinner） |

### 5.4 两者如何配合

对于"修改 `settings.yaml` 中的 `language` 字段"这一场景，完整链路是：

```
  用户编辑 settings.yaml
        │
        ▼
  /api/hash 返回新 hash
        │
        ▼
  客户端检测到 hash 变化
        │
        ├── 调用 /api/revalidate → ISR 重跑 getStaticProps
        │       （服务端加载新语言包，生成新 HTML）
        │
        ▼
  window.location.reload()  ←────── 全量刷新
        │
        ▼
  新页面加载 → 新语言已在服务端注入
        │
        ▼
  appWithTranslation 水合新语言包  →  正常显示
```

> **为什么有了 ISR + reload 还需要 `changeLanguage`？**
>
> 因为 `changeLanguage` 解决的是**同一次页面会话内** settings 状态变化时的实时响应。如果未来有前端 UI 直接修改 `settings`（不涉及服务器文件），就走 changeLanguage 路径。而配置文件修改是跨会话、跨用户的全局变更，必须走 ISR + reload 才能保证服务端输出与客户端一致。
>
> 另外，`initialSettings` 在 `Home` 组件挂载后通过 `setSettings(initialSettings)` 写入 context，这也会触发一次 `changeLanguage`，确保服务端注入的语言与客户端 i18n 实例同步。

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
  /api/hash 返回新 hash  ──  对比 localStorage 中旧 hash
        │
        ▼
  调用 /api/revalidate  →  res.revalidate("/") → getStaticProps() 重跑（加载新语言包）
        │
        ▼
  window.location.reload()  ←  整页刷新
```

---

## 七、单元测试中的 Mock

[vitest.setup.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/vitest.setup.js#L11-L32) Mock 了 `next-i18next/pages`，让组件单测无需加载真实翻译文件：

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

针对 index 页面的测试（[src/__tests__/pages/index.test.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/__tests__/pages/index.test.jsx#L486-L517)）还单独 mock 了 `changeLanguage` 以验证调用：

```javascript
const i18n = { language: "en", changeLanguage: vi.fn() };

// ... 设置新 settings 后
expect(i18n.changeLanguage).toHaveBeenCalledWith("en");
```

---

## 八、关键代码文件索引

| 文件 | 作用 |
|---|---|
| [next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js) | i18n 主配置 + 6 个自定义 formatter |
| [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next.config.js) | 注入 i18n 到 Next.js |
| [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/_app.jsx) | `appWithTranslation` 包装应用根组件 |
| [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx) | `getStaticProps` 加载语言 + `changeLanguage` 切换 + hash/revalidate 刷新 |
| [src/pages/api/hash.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/api/hash.js) | 配置文件 hash 计算接口 |
| [src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/api/revalidate.js) | ISR 重生成接口 |
| [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/utils/config/config.js) | 读取 `settings.yaml`（含 `language` 字段） |
| [src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/utils/contexts/settings.jsx) | `SettingsContext` 存储 settings 状态 |
| [src/components/services/widget/block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/services/widget/block.jsx) | Block 组件隐式翻译 label |
| [public/locales/en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/public/locales/en/common.json) | 英文翻译基准 |
| [vitest.setup.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/vitest.setup.js) | 测试环境 i18n Mock |
| [docs/configs/settings.md](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/docs/configs/settings.md#L443-L456) | 语言配置用户文档 |
| [docs/more/translations.md](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/docs/more/translations.md) | 翻译贡献指南 |
