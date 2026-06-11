# Homepage i18n 多语言体系代码链路分析

## 一、技术栈概览

| 库 | 版本 | 作用 |
|---|---|---|
| `next-i18next` | ^16.0.7 | Next.js 的 i18n 集成库，封装 i18next/react-i18next |
| `react-i18next` | ^15.5.3 | React 绑定层，提供 `useTranslation` Hook |
| `i18next` | ^26.3.0 | 核心国际化引擎 |

依赖关系：[package.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/package.json#L24-L38)

---

## 二、语言包加载机制

### 2.1 语言包目录结构

语言包以 JSON 格式存放于 `public/locales/{语言代码}/common.json`，例如：

- [public/locales/en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/public/locales/en/common.json) — 英文
- `public/locales/zh-Hans/common.json` — 简体中文
- `public/locales/fr/common.json` — 法语
- ... 共 40+ 种语言（含 pt-BR、nb-NO 等带连字符的 code）

每个 JSON 文件结构按命名空间分组：

```json
{
  "common":    { "bytes": "{{value, bytes}}", "number": "{{value, number}}", ... },
  "widget":    { "missing_type": "Missing Widget Type: {{type}}", ... },
  "resources": { "cpu": "CPU", "mem": "MEM", ... },
  "unifi":     { "uptime": "Uptime", ... },
  ...
}
```

### 2.2 配置文件（静态配置）

[next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js) 定义了 next-i18next 的初始化参数：

```javascript
module.exports = {
  i18n: {
    defaultLocale: "en",
    locales: ["en"],          // 注意：这里仅声明 en，实际语言由运行时动态加载
  },
  serializeConfig: false,
  use: [
    {
      init: (i18next) => {
        // 注册自定义 formatter：bytes / rate / percent / date / relativeDate / duration
        i18next.services.formatter.add("bytes", (value, lng, options) => ...);
        i18next.services.formatter.add("rate",  (value, lng, options) => ...);
        i18next.services.formatter.add("percent", ...);
        i18next.services.formatter.add("date", ...);
        i18next.services.formatter.add("relativeDate", ...);
        i18next.services.formatter.add("duration", ...);
      },
      type: "3rdParty",
    },
  ],
};
```

> 关键点：`locales: ["en"]` 仅作为 Next.js 构建期的最小声明；**运行时并不会被此数组限制**，实际加载何种语言由 `serverSideTranslations(language)` 传入的参数决定。

[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next.config.js#L1-L19) 将同一 i18n 配置注入 Next.js：

```javascript
const { i18n } = require("./next-i18next.config");
const nextConfig = { ..., i18n };
module.exports = nextConfig;
```

### 2.3 服务端加载（SSG 阶段）

首页的 `getStaticProps` 负责在**构建/重生成时**把翻译 JSON 注入 pageProps。

核心代码在 [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L55-L95)：

```javascript
// 语言别名映射，兼容旧配置
const LANGUAGE_ALIASES = { "zh-cn": "zh-Hans" };
const normalizeLanguage = (language) => {
  if (!language) return "en";
  const alias = LANGUAGE_ALIASES[language.toLowerCase()];
  return alias || language;
};

export async function getStaticProps() {
  const { providers, ...settings } = getSettings();   // 读取 settings.yaml
  const language = normalizeLanguage(settings.language);

  return {
    props: {
      initialSettings: settings,
      fallback: { /* services / bookmarks / widgets SWR fallback */ },
      ...(await serverSideTranslations(language)),   // ★ 加载指定语言包
    },
  };
}
```

`serverSideTranslations(language)` 来自 `next-i18next/pages/serverSideTranslations`，它会：
1. 读取 `public/locales/{language}/common.json`
2. 序列化为 `_nextI18Next` 注入 `pageProps`
3. 由客户端水合后供 `useTranslation` 使用

异常分支（catch 块）使用兜底语言 `"en"`。

### 2.4 客户端初始化

应用根组件 [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/_app.jsx#L73-L100) 使用 `appWithTranslation` HOC 包装：

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
1. 创建全局 `I18nextProvider`，把服务端序列化的 `_nextI18Next` 恢复为 i18next 实例
2. 把配置（含自定义 formatter）应用到实例
3. 使所有子组件可通过 `useTranslation()` 访问 t / i18n

---

## 三、文案选择逻辑

### 3.1 基础用法：`useTranslation()` Hook

所有需要翻译的组件都从 `next-i18next/pages` 导入 Hook：

```javascript
import { useTranslation } from "next-i18next/pages";

function Component() {
  const { t, i18n } = useTranslation();
  return <div>{t("resources.cpu")}</div>;
}
```

实际案例：
- [src/components/widgets/resources/cpu.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/widgets/resources/cpu.jsx#L8-L51) — 基本标签翻译
- [src/widgets/unifi/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/widgets/unifi/component.jsx#L45-L49) — 带插值和 formatter 的翻译

### 3.2 插值变量

```javascript
t("widget.missing_type", { type: "weatherapi" })
// → "Missing Widget Type: weatherapi"
```

对应语言包中的模板：`"missing_type": "Missing Widget Type: {{type}}"`

### 3.3 自定义 Formatter（数值/时间格式化）

这是该项目最具特色的部分。在 [next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js#L118-L161) 注册的 formatter 允许在翻译 key 中直接格式化数值：

| Formatter | 语言包 Key 示例 | 组件调用 | 输出 |
|---|---|---|---|
| `bytes` | `common.bytes: "{{value, bytes}}"` | `t("common.bytes", { value: 1024 })` | `1.024 kB`（按 locale） |
| `rate` | `common.byterate: "{{value, rate(bits: false)}}"` | `t("common.byterate", { value: 5000 })` | `5 kB/s` |
| `percent` | `common.percent: "{{value, percent}}"` | `t("common.percent", { value: 42 })` | `42%`（Intl.NumberFormat） |
| `number` | `common.number: "{{value, number}}"` | `t("common.number", { value: 1234.5, maximumFractionDigits: 1 })` | `1,234.5` |
| `date` | `common.date: "{{value, date}}"` | `t("common.date", { value: timestamp })` | 本地化日期 |
| `relativeDate` | `common.relativeDate: "{{value, relativeDate}}"` | `t("common.relativeDate", { value: isoDate })` | `in 3 days` / `2 hours ago` |
| `duration` | `common.duration: "{{value, duration}}"` | `t("common.duration", { value: 3661 })` | `1h1m1s`（拼接 `common.hours` 等 key） |

实际使用示例（[uptime.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/widgets/resources/uptime.jsx#L25-L32)）：

```javascript
<Resource
  value={t("common.duration", { value: data.uptime })}   // 秒数 → "2d5h"
  label={t("resources.uptime")}
  percentage={percent}
/>
```

### 3.4 隐式翻译：Block 组件的 `label` prop

通用展示组件 [src/components/services/widget/block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/services/widget/block.jsx#L9-L55) 会**自动把 label 当作 key 翻译**：

```javascript
export default function Block({ value, label }) {
  const { t } = useTranslation();
  return (
    <div>
      <div>{value === undefined ? "-" : value}</div>
      <div>{t(label)}</div>   {/* label 直接传 key 字符串 */}
    </div>
  );
}
```

调用方（[unifi/component.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/widgets/unifi/component.jsx#L27-L34)）只需传 key：

```jsx
<Block label="unifi.uptime" />
<Block label="unifi.wan" value={wan.up ? t("unifi.up") : t("unifi.down")} />
```

---

## 四、语言切换与刷新机制

### 4.1 语言来源：`settings.yaml`

语言配置存储在 `config/settings.yaml` 中（默认 skeleton 未写该字段，默认英文）：

```yaml
language: fr   # ca / de / en / es / fr / he / hr / hu / it / nb-NO / nl / pt / ru / sv / vi / zh-Hans / zh-Hant
```

读取函数在 [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/utils/config/config.js#L82-L103)：

```javascript
export function getSettings() {
  checkAndCopyConfig("settings.yaml");
  const settingsYaml = join(CONF_DIR, "settings.yaml");
  const rawFileContents = readFileSync(settingsYaml, "utf8");
  const fileContents = substituteEnvironmentVars(rawFileContents);
  return yaml.load(fileContents) ?? {};
}
```

### 4.2 客户端动态切换

页面组件的 `Home` 子组件监听 `settings.language` 变化，调用 `i18n.changeLanguage()`：

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L234-L247)：

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
      i18n.changeLanguage(language);   // ★ 动态切换，触发重渲染
    }
    // 主题/颜色同理
  }, [i18n, settings, color, setColor, theme, setTheme]);
}
```

`i18n.changeLanguage()` 是 i18next 原生 API，会：
1. 异步加载目标语言的 JSON（若未缓存）
2. 改变 `i18n.language`
3. 通知所有使用 `useTranslation` 的组件重渲染（通过 react-i18next 的 context）

### 4.3 配置文件变更 → 页面整体刷新

`settings.yaml` 是服务器文件，浏览器无法监听。项目采用 **hash 轮询** 方案：

[src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx#L100-L131)：

```javascript
const { data: hashData, mutate: mutateHash } = useSWR("/api/hash");

// 窗口获得焦点时重新拉取 hash
useEffect(() => {
  if (windowFocused) mutateHash();
}, [windowFocused, mutateHash]);

// hash 变化时触发 ISR 并 reload
useEffect(() => {
  if (hashData && typeof window !== "undefined") {
    const previousHash = localStorage.getItem("hash");
    if (!previousHash) {
      localStorage.setItem("hash", hashData.hash);
    }
    if (previousHash && previousHash !== hashData.hash) {
      localStorage.setItem("hash", hashData.hash);
      fetch("/api/revalidate").then((res) => {   // 调用 Next.js ISR revalidate
        if (res.ok) window.location.reload();    // 刷新整页
      });
    }
  }
}, [hashData]);
```

流程：
1. `/api/hash` 返回当前所有配置文件的内容 hash
2. 客户端 `localStorage` 保存上次 hash
3. 检测到差异 → 调用 `/api/revalidate` 触发 `getStaticProps` 重跑（加载新语言包） → `location.reload()` 全量刷新

### 4.4 ISR Revalidate API

对应接口 [src/pages/api/revalidate.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/api/revalidate.js)，内部调用 `res.revalidate("/")` 重新执行首页的 `getStaticProps`。

---

## 五、完整链路时序图

```
           ┌─────────────┐   读 YAML   ┌──────────────────┐
           │ settings.yaml│◄───────────┤ getSettings()     │
           └─────────────┘             └────────┬─────────┘
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
            ┌───────────────────────────────────┼───────────────────────────────────┐
            │                                   ▼                                   │
            │   useTranslation() ───► t("key") / i18n.changeLanguage(lng)           │
            │                                                                       │
            │   ┌───────────────┐      ┌─────────────────┐      ┌───────────────┐   │
            │   │  Widget CPU   │      │  Widget Unifi   │      │  Block 组件   │   │
            │   │ t("resources.cpu")   │ t("unifi.uptime")│      │ t(label)      │   │
            │   └───────────────┘      └─────────────────┘      └───────────────┘   │
            └───────────────────────────────────────────────────────────────────────┘

配置变更触发刷新：
  用户编辑 settings.yaml → /api/hash 变化 → /api/revalidate → getStaticProps() 重跑
                                                                     │
                                                          window.location.reload() ◄─┘
```

---

## 六、单元测试中的 Mock

[vitest.setup.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/vitest.setup.js#L11-L32) Mock 了 `next-i18next/pages`：

```javascript
vi.mock("next-i18next/pages", () => ({
  appWithTranslation: (Component) => Component,
  useTranslation: () => ({
    i18n: { language: "en" },
    t: (key, opts) => {
      // 简化：对 common.* 的 formatter key 返回 opts.value，其余返回 key 本身
      if (key === "common.number") return String(opts?.value ?? "");
      // ...
      return key;
    },
  }),
}));
```

这样组件单测无需加载真实翻译文件即可运行。

---

## 七、关键代码文件索引

| 文件 | 作用 |
|---|---|
| [next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next-i18next.config.js) | i18n 主配置 + 自定义 formatter |
| [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/next.config.js) | 注入 i18n 到 Next.js |
| [src/pages/_app.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/_app.jsx) | appWithTranslation 包装应用 |
| [src/pages/index.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/pages/index.jsx) | getStaticProps 加载语言 + changeLanguage 切换 + hash/revalidate 刷新 |
| [src/utils/config/config.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/utils/config/config.js) | 读取 settings.yaml（含 language 字段） |
| [src/utils/contexts/settings.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/utils/contexts/settings.jsx) | SettingsContext 存储 settings |
| [src/components/services/widget/block.jsx](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/src/components/services/widget/block.jsx) | Block 组件隐式翻译 label |
| [public/locales/en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/public/locales/en/common.json) | 英文翻译（基准） |
| [vitest.setup.js](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/vitest.setup.js) | 测试环境 Mock |
| [docs/configs/settings.md](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/docs/configs/settings.md#L443-L456) | 语言配置用户文档 |
| [docs/more/translations.md](file:///d:/fz/0601/solo-dogfeeding/code/201-homepage/docs/more/translations.md) | 翻译贡献指南 |
