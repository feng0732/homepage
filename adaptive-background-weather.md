# 首页自适应背景与天气栏代码分析

## 一、整体架构概览

首页的自适应背景和天气栏功能由以下核心模块协作完成：

```
┌─────────────────────────────────────────────────────────────┐
│                     页面渲染层 (React)                       │
│  ┌─────────────┐  ┌──────────────────────────────────────┐  │
│  │  Wrapper    │  │  Home 组件                           │  │
│  │  (背景渲染)  │  │  (Widget 容器 + 信息栏布局)          │  │
│  └─────────────┘  └──────────────────────────────────────┘  │
│         │                           │                       │
│         ▼                           ▼                       │
│  ┌─────────────┐          ┌──────────────────┐              │
│  │  CSS 主题   │          │  Widget 映射器   │              │
│  │  (theme.css)│          │  (widget.jsx)    │              │
│  └─────────────┘          └──────────────────┘              │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────┐              │
│                          │  天气组件        │              │
│                          │  (weather.jsx)   │              │
│                          │  (openmeteo.jsx) │              │
│                          └──────────────────┘              │
│                                     │                       │
└─────────────────────────────────────┼───────────────────────┘
                                      │
┌─────────────────────────────────────┼───────────────────────┐
│               API 层 (Next.js)       │                       │
│                                     ▼                       │
│                          ┌──────────────────┐              │
│                          │  /api/widgets/*  │              │
│                          │  weather.js      │              │
│                          │  openmeteo.js    │              │
│                          └──────────────────┘              │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────┐              │
│                          │  cachedRequest   │              │
│                          │  (http.js)       │              │
│                          └──────────────────┘              │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────┐              │
│                          │  第三方天气 API  │              │
│                          └──────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、背景来源与配置

### 2.1 配置读取流程

背景配置从 `settings.yaml` 读取，通过 `getSettings()` 函数解析：

**配置定义位置**：[config.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/config/config.js#L82-L103)

```javascript
export function getSettings() {
  checkAndCopyConfig("settings.yaml");
  const settingsYaml = join(CONF_DIR, "settings.yaml");
  const rawFileContents = readFileSync(settingsYaml, "utf8");
  const fileContents = substituteEnvironmentVars(rawFileContents);
  const initialSettings = yaml.load(fileContents) ?? {};
  // ... 布局转换逻辑
  return initialSettings;
}
```

### 2.2 背景配置格式

**配置文档参考**：[settings.md](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/docs/configs/settings.md#L32-L74)

支持两种配置格式：

**格式一：简单字符串（仅图片URL）**
```yaml
background: https://example.com/background.jpg
```

**格式二：对象（支持高级滤镜）**
```yaml
background:
  image: /images/background.png
  blur: sm           # 模糊程度: sm, "", md, xl...
  saturate: 50       # 饱和度: 0, 50, 100...
  brightness: 50     # 亮度: 0, 50, 75...
  opacity: 50        # 不透明度: 0-100
```

### 2.3 背景渲染逻辑

**渲染代码位置**：[index.jsx#L517-L592](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/index.jsx#L517-L592)

背景渲染在最外层的 `Wrapper` 函数组件中完成：

```javascript
export default function Wrapper({ initialSettings, fallback }) {
  const { theme } = useContext(ThemeContext);
  const { color } = useContext(ColorContext);
  
  // 解析背景配置
  let backgroundImage = "";
  let opacity = initialSettings?.backgroundOpacity ?? 0;
  let backgroundBlur = false;
  let backgroundSaturate = false;
  let backgroundBrightness = false;
  
  if (initialSettings?.background) {
    const bg = initialSettings.background;
    if (typeof bg === "object") {
      backgroundImage = bg.image || "";
      if (bg.opacity !== undefined) {
        opacity = 1 - bg.opacity / 100;  // 转换为遮罩透明度
      }
      backgroundBlur = bg.blur !== undefined;
      backgroundSaturate = bg.saturate !== undefined;
      backgroundBrightness = bg.brightness !== undefined;
    } else {
      backgroundImage = bg;  // 简单字符串格式
    }
  }

  // 动态切换主题类名
  useEffect(() => {
    const html = document.documentElement;
    html.classList.remove("dark", "scheme-dark", "scheme-light");
    html.classList.toggle("dark", theme === "dark");
    html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");
    
    const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
    // 移除旧主题类，添加新主题类
    // ...
  }, [backgroundImage, opacity, theme, color, initialSettings.color]);

  return (
    <>
      {/* 背景层 */}
      {backgroundImage && (
        <div
          id="background"
          style={{
            backgroundImage: `linear-gradient(rgb(var(--bg-color) / ${opacity}), rgb(var(--bg-color) / ${opacity})), url('${backgroundImage}')`,
          }}
        />
      )}
      {/* 内容层（应用滤镜） */}
      <div id="page_wrapper">
        <div
          id="inner_wrapper"
          className={classNames(
            "w-full h-full overflow-auto",
            backgroundBlur && `backdrop-blur${...}`,
            backgroundSaturate && `backdrop-saturate-${...}`,
            backgroundBrightness && `backdrop-brightness-${...}`,
          )}
        >
          <Index initialSettings={initialSettings} fallback={fallback} />
        </div>
      </div>
    </>
  );
}
```

### 2.4 CSS 样式定义

**背景样式**：[globals.css#L36-L45](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/styles/globals.css#L36-L45)

```css
#background {
  position: fixed;
  inset: 0;
  z-index: 0;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  background-attachment: scroll;
  pointer-events: none;
}
```

**主题颜色变量**：[theme.css](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/styles/theme.css)

每个主题（如 `.theme-slate`, `.theme-blue` 等）定义了一套 CSS 变量：
```css
.theme-slate {
  --color-50: 248 250 252;
  --color-100: 241 245 249;
  --color-800: 30 41 59;  /* 深色背景 */
  /* ... */
}

.light {
  --bg-color: var(--color-50);
}

.dark {
  --bg-color: var(--color-800);
}
```

---

## 三、天气组件与刷新机制

### 3.1 天气组件类型

系统支持三种天气数据源，对应三个组件：

| 组件类型 | 组件文件 | API 接口 |
|---------|---------|----------|
| weatherapi | [weather.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/weather/weather.jsx) | [/api/widgets/weather](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/weather.js) |
| openmeteo | [openmeteo.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx) | [/api/widgets/openmeteo](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openmeteo.js) |
| openweathermap | [weather.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openweathermap/weather.jsx) | [/api/widgets/openweathermap](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openweathermap.js) |

### 3.2 Widget 动态映射

**映射代码**：[widget.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/widget.jsx#L1-L36)

```javascript
const widgetMappings = {
  weatherapi: dynamic(() => import("components/widgets/weather/weather")),
  openweathermap: dynamic(() => import("components/widgets/openweathermap/weather")),
  openmeteo: dynamic(() => import("components/widgets/openmeteo/openmeteo")),
  // ...其他 widget
};

export default function Widget({ widget, style }) {
  const InfoWidget = widgetMappings[widget.type];
  if (InfoWidget) {
    return (
      <ErrorBoundary>
        <InfoWidget options={{ ...widget.options, style }} />
      </ErrorBoundary>
    );
  }
  return <div>Missing <strong>{widget.type}</strong></div>;
}
```

### 3.3 天气组件结构（以 OpenMeteo 为例）

**组件代码**：[openmeteo.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx)

组件采用**外层容器 + 内部数据组件**的结构：

```javascript
// 内部数据组件 - 负责 API 调用和渲染
function Widget({ options }) {
  const { t } = useTranslation();
  
  // SWR 数据获取 - 关键！自动处理缓存和刷新
  const { data, error } = useSWR(
    `/api/widgets/openmeteo?${new URLSearchParams({ ...options }).toString()}`
  );

  if (error || data?.error) return <Error options={options} />;
  if (!data) return <LoadingState />;

  // 计算昼夜状态
  const condition = data.current_weather.weathercode;
  const timeOfDay = 
    data.current_weather.time > data.daily.sunrise[0] && 
    data.current_weather.time < data.daily.sunset[0]
      ? "day" : "night";

  return (
    <Container>
      <PrimaryText>
        {options.label && `${options.label}, `}
        {t("common.number", { value: data.current_weather.temperature, ... })}
      </PrimaryText>
      <SecondaryText>{t(`wmo.${data.current_weather.weathercode}-${timeOfDay}`)}</SecondaryText>
      <WidgetIcon icon={mapIcon(condition, timeOfDay)} size="xl" />
    </Container>
  );
}

// 外层容器 - 负责地理位置获取
export default function OpenMeteo({ options }) {
  const [location, setLocation] = useState(false);
  const [requesting, setRequesting] = useState(false);

  // 静态配置优先
  if (!location && options.latitude && options.longitude) {
    setLocation({ latitude: options.latitude, longitude: options.longitude });
  }

  // 地理位置请求函数
  const requestLocation = useCallback(() => {
    setRequesting(true);
    navigator.geolocation.getCurrentPosition(
      (position) => {
        setLocation({ 
          latitude: position.coords.latitude, 
          longitude: position.coords.longitude 
        });
        setRequesting(false);
      },
      () => setRequesting(false),
      {
        enableHighAccuracy: true,
        maximumAge: 1000 * 60 * 60 * 3,  // 缓存3小时
        timeout: 1000 * 30,
      }
    );
  }, []);

  // 权限检查 - 已授权则自动获取
  useEffect(() => {
    if (!options.latitude && !options.longitude && typeof navigator !== "undefined") {
      navigator.permissions?.query({ name: "geolocation" }).then((result) => {
        if (result.state === "granted") {
          requestLocation();
        }
      });
    }
  }, [options.latitude, options.longitude, requestLocation]);

  // 未获取位置时显示授权按钮
  if (!location) {
    return (
      <ContainerButton callback={requestLocation}>
        <PrimaryText>{t("weather.current")}</PrimaryText>
        <SecondaryText>{t("weather.allow")}</SecondaryText>
        <WidgetIcon icon={requesting ? MdLocationSearching : MdLocationDisabled} />
      </ContainerButton>
    );
  }

  // 有位置后渲染数据组件
  return <Widget options={{ ...location, ...options }} />;
}
```

### 3.4 刷新机制详解

天气数据刷新采用**双层缓存策略**：

#### 第一层：客户端 SWR 缓存（useSWR）

**SWR 配置**：[index.jsx#L186](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/index.jsx#L186)

```javascript
<SWRConfig value={{ 
  fallback, 
  fetcher: (resource, init) => fetch(resource, init).then((res) => res.json()) 
}}>
```

SWR 默认行为：
- 页面聚焦时自动重新验证（`revalidateOnFocus: true`）
- 网络恢复时自动重新验证
- 重复数据请求去重（`dedupingInterval`）
- 组件卸载后缓存保留

**注意**：天气组件直接使用 `useSWR` 但未指定 `refreshInterval`，意味着：
- 不会定时自动刷新
- 仅在页面重新聚焦或组件重新挂载时刷新

#### 第二层：服务器端内存缓存（cachedRequest）

**缓存代码**：[http.js#L85-L109](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/proxy/http.js#L85-L109)

```javascript
import cache from "memory-cache";

export async function cachedRequest(url, duration = 5, ua = "homepage") {
  const cached = cache.get(url);
  
  if (cached) {
    return cached;  // 命中缓存直接返回
  }

  // 未命中，发起请求
  const options = { headers: { "User-Agent": ua, Accept: "application/json" } };
  let [, , data] = await httpProxy(url, options);
  
  // 解析响应数据
  if (Buffer.isBuffer(data)) {
    try {
      data = JSON.parse(Buffer.from(data).toString());
    } catch (e) {
      data = Buffer.from(data).toString();
    }
  }
  
  // 存入缓存，duration 单位为分钟
  cache.put(url, data, duration * 1000 * 60);
  return data;
}
```

**API 调用示例**：[openmeteo.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openmeteo.js#L1-L9)

```javascript
export default async function handler(req, res) {
  const { latitude, longitude, units, cache, timezone } = req.query;
  const degrees = units === "metric" ? "celsius" : "fahrenheit";
  const apiUrl = `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&daily=sunrise,sunset&current_weather=true&temperature_unit=${degrees}&timezone=${timezone}`;
  
  // cache 参数来自配置，控制服务器端缓存时间（分钟）
  return res.send(await cachedRequest(apiUrl, cache));
}
```

### 3.5 天气图标映射

不同天气 API 使用不同的天气代码，需要映射到统一的图标组件：

**WeatherAPI 映射**：[condition-map.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/weather/condition-map.js)

```javascript
const conditions = [
  {
    code: 1000,  // WeatherAPI 代码：晴天
    icon: {
      day: Icons.WiDaySunny,
      night: Icons.WiNightClear,
    },
  },
  {
    code: 1003,  // WeatherAPI 代码：多云
    icon: {
      day: Icons.WiDayCloudy,
      night: Icons.WiNightPartlyCloudy,
    },
  },
  // ... 更多天气代码映射
];

export default function mapIcon(weatherStatusCode, timeOfDay) {
  const mapping = conditions.find((condition) => condition.code === weatherStatusCode);
  if (mapping) {
    return timeOfDay === "day" ? mapping.icon.day : mapping.icon.night;
  }
  return Icons.WiDaySunny;  // 默认图标
}
```

**OpenMeteo 映射**：[openmeteo-condition-map.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/weather/openmeteo-condition-map.js)

使用 WMO（世界气象组织）天气代码：
```javascript
const conditions = [
  { code: 0, icon: { day: Icons.WiDaySunny, night: Icons.WiNightClear } },        // 晴朗
  { code: 1, icon: { day: Icons.WiDayCloudy, night: Icons.WiNightAltCloudy } },  // 大部晴朗
  { code: 2, icon: { day: Icons.WiDayCloudy, night: Icons.WiNightAltCloudy } },  // 阴天
  { code: 3, icon: { day: Icons.WiDayCloudy, night: Icons.WiNightAltCloudy } },  // 阴
  { code: 45, icon: { day: Icons.WiDayFog, night: Icons.WiNightFog } },          // 有雾
  // ...
];
```

---

## 四、地区配置详解

### 4.1 配置方式

**配置文件**：[widgets.yaml](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/skeleton/widgets.yaml)

**配置示例**：[openmeteo.md](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/docs/widgets/info/openmeteo.md)

```yaml
- openmeteo:
    label: Beijing              # 可选：显示标签
    latitude: 39.9042          # 静态纬度
    longitude: 116.4074        # 静态经度
    timezone: Asia/Shanghai    # 可选：时区
    units: metric              # 单位：metric 或 imperial
    cache: 5                   # 服务器缓存时间（分钟）
    format:                    # 可选：数字格式化
      maximumFractionDigits: 1
```

### 4.2 地区获取优先级

地区获取遵循**静态配置优先，动态获取兜底**的策略：

```
                    组件初始化
                         │
                         ▼
          ┌───────────────────────────┐
          │ options.latitude &&       │
          │ options.longitude 存在?   │
          └───────────┬───────────────┘
                      │
              ┌───────┴───────┐
              │ 是            │ 否
              ▼               ▼
      使用静态配置    ┌──────────────────────┐
                      │ 检查浏览器定位权限   │
                      └──────────┬───────────┘
                                 │
                         ┌───────┴───────┐
                         │ 已授权        │ 未授权
                         ▼               ▼
                  自动获取位置     显示授权按钮
                     (静默)        (用户点击后请求)
```

**关键代码**：[openmeteo.jsx#L62-L95](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx#L62-L95)

```javascript
// 1. 静态配置优先（渲染阶段同步判断）
if (!location && options.latitude && options.longitude) {
  setLocation({ latitude: options.latitude, longitude: options.longitude });
}

// 2. 权限检查（useEffect 异步执行）
useEffect(() => {
  if (!options.latitude && !options.longitude && typeof navigator !== "undefined") {
    navigator.permissions?.query({ name: "geolocation" }).then((result) => {
      if (result.state === "granted") {
        requestLocation();  // 已授权，静默获取
      }
    });
  }
}, [options.latitude, options.longitude, requestLocation]);
```

### 4.3 地理位置 API 参数

```javascript
navigator.geolocation.getCurrentPosition(
  successCallback,
  errorCallback,
  {
    enableHighAccuracy: true,        // 高精度模式
    maximumAge: 1000 * 60 * 60 * 3,  // 位置缓存3小时
    timeout: 1000 * 30,              // 30秒超时
  }
);
```

---

## 五、页面渲染流程

### 5.1 整体渲染流程

```
SSR 阶段 (getStaticProps)
        │
        ▼
┌──────────────────────────┐
│ 读取 settings.yaml       │
│ 读取 services.yaml       │
│ 读取 bookmarks.yaml      │
│ 读取 widgets.yaml        │
│ 构建 fallback 数据       │
└───────────┬──────────────┘
            │
            ▼
客户端渲染 (React)
            │
            ▼
┌──────────────────────────┐
│ Wrapper 组件             │
│  ├─ 解析 background 配置 │
│  ├─ 应用主题类名         │
│  ├─ 渲染 #background 层  │
│  └─ 渲染 #inner_wrapper  │
│      └─ Index 组件       │
│          ├─ 验证配置     │
│          ├─ Home 组件    │
│          │   ├─ Head (元数据)
│          │   ├─ 信息栏 (#information-widgets)
│          │   │   ├─ 左对齐 Widget
│          │   │   └─ 右对齐 Widget (天气栏)
│          │   │       └─ Widget 映射
│          │   │           └─ 天气组件
│          │   │               ├─ 位置获取
│          │   │               └─ SWR 数据获取
│          │   ├─ 服务/书签组
│          │   └─ 页脚
│          └─ 错误处理
└──────────────────────────┘
```

### 5.2 信息栏布局逻辑

**布局代码**：[index.jsx#L454-L496](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/index.jsx#L454-L496)

信息栏分为左右两部分，通过 `rightAlignedWidgets` 数组控制哪些组件靠右显示：

```javascript
// 右对齐 Widget 类型 - 天气组件都在这里
const rightAlignedWidgets = ["weatherapi", "openweathermap", "weather", "openmeteo", "search", "datetime"];

// 渲染结构
<div id="information-widgets" className={headerStyles[headerStyle]}>
  <div id="widgets-wrap" className="flex flex-row w-full flex-wrap justify-between gap-x-2">
    {/* 左对齐 Widget */}
    {widgets
      .filter((widget) => !rightAlignedWidgets.includes(widget.type))
      .map((widget, i) => (
        <Widget key={i} widget={widget} style={{...}} />
      ))}

    {/* 右对齐 Widget 容器 */}
    <div id="information-widgets-right" className="flex flex-wrap grow sm:basis-auto justify-between md:justify-end">
      {widgets
        .filter((widget) => rightAlignedWidgets.includes(widget.type))
        .map((widget, i) => (
          <Widget key={i} widget={widget} style={{...}} />
        ))}
    </div>
  </div>
</div>
```

### 5.3 主题与背景的联动

主题切换会同时影响背景和内容：

1. **背景层**：通过 `linear-gradient(rgb(var(--bg-color) / opacity), ...)` 将主题色叠加到背景图片上
2. **内容层**：通过 CSS 变量 `--bg-color`、`--color-*` 控制文本和卡片颜色
3. **滤镜层**：通过 `backdrop-blur`、`backdrop-saturate`、`backdrop-brightness` 对内容层应用滤镜

**关键联动点**：[index.jsx#L540-L563](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/index.jsx#L540-L563)

```javascript
useEffect(() => {
  const html = document.documentElement;
  const body = document.body;

  // 切换明暗模式
  html.classList.remove("dark", "scheme-dark", "scheme-light");
  html.classList.toggle("dark", theme === "dark");
  html.classList.add(theme === "dark" ? "scheme-dark" : "scheme-light");

  // 切换颜色主题
  const desiredThemeClass = `theme-${color || initialSettings.color || "slate"}`;
  const themeClassesToRemove = Array.from(html.classList).filter(
    (cls) => cls.startsWith("theme-") && cls !== desiredThemeClass
  );
  if (themeClassesToRemove.length) {
    html.classList.remove(...themeClassesToRemove);
  }
  if (!html.classList.contains(desiredThemeClass)) {
    html.classList.add(desiredThemeClass);
  }
}, [backgroundImage, opacity, theme, color, initialSettings.color]);
```

---

## 六、协作流程总结

### 6.1 数据流向图

```
┌──────────────┐
│ settings.yaml│  background, theme, color
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│ getSettings()│────▶│ Wrapper      │
└──────────────┘     │  背景渲染    │
                      └──────┬───────┘
                             │
┌──────────────┐            │
│ widgets.yaml │            ▼
│  - openmeteo │     ┌──────────────┐
│  - latitude  │────▶│ Widget.jsx   │
│  - longitude │     │  组件映射    │
└──────┬───────┘     └──────┬───────┘
       │                    │
       │                    ▼
       │             ┌──────────────┐
       │             │ openmeteo.jsx│
       │             │  位置获取    │
       │             └──────┬───────┘
       │                    │
       │                    ▼
       │             ┌──────────────┐
       └────────────▶│ useSWR()     │
                     │  客户端缓存  │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │ /api/widgets/│
                     │ openmeteo    │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │ cachedRequest│
                     │  服务端缓存  │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │ Open-Meteo   │
                     │  API         │
                     └──────────────┘
```

### 6.2 关键协作点

1. **配置驱动渲染**：所有视觉元素（背景、主题、天气）都由 YAML 配置驱动
2. **分层缓存**：SWR（客户端）+ memory-cache（服务端）+ 第三方 API 缓存
3. **优雅降级**：地理位置获取失败时显示授权按钮，API 失败时显示错误组件
4. **响应式主题**：CSS 变量 + 动态类名实现主题与背景的联动
5. **渐进式加载**：地理位置 → 加载状态 → 天气数据显示

### 6.3 可配置项清单

| 配置项 | 文件 | 说明 |
|-------|------|------|
| `background` | settings.yaml | 背景图片 URL 或对象（含滤镜） |
| `backgroundOpacity` | settings.yaml | 背景不透明度（0-100） |
| `cardBlur` | settings.yaml | 卡片模糊效果 |
| `theme` | settings.yaml | `dark` / `light` |
| `color` | settings.yaml | 主题色（slate, blue, red 等） |
| `headerStyle` | settings.yaml | 信息栏样式（boxed, underlined, clean） |
| `providers.weatherapi` | settings.yaml | WeatherAPI API Key |
| `providers.openweathermap` | settings.yaml | OpenWeatherMap API Key |
| `widgets[].openmeteo.latitude` | widgets.yaml | 静态纬度 |
| `widgets[].openmeteo.longitude` | widgets.yaml | 静态经度 |
| `widgets[].openmeteo.cache` | widgets.yaml | 服务器缓存时间（分钟） |
| `widgets[].openmeteo.units` | widgets.yaml | 单位（metric/imperial） |

---

## 七、代码优化建议

### 7.1 天气刷新可配置性

**问题**：天气组件未设置 `refreshInterval`，用户无法配置自动刷新频率。

**建议优化**：在天气组件中添加 `refreshInterval` 支持：

```javascript
// 在 Widget 组件内部
const refreshInterval = options.refreshInterval 
  ? Math.max(60000, options.refreshInterval)  // 最小1分钟
  : undefined;

const { data, error } = useSWR(url, { refreshInterval });
```

### 7.2 背景变化的性能优化

**问题**：`useEffect` 依赖项包含 `backgroundImage` 和 `opacity`，但实际只需要 `theme` 和 `color` 来更新类名。

**建议优化**：拆分 effect，减少不必要的执行：

```javascript
// 只处理主题类名
useEffect(() => {
  const html = document.documentElement;
  html.classList.toggle("dark", theme === "dark");
  // ...
}, [theme, color, initialSettings.color]);
```

### 7.3 地理位置缓存

**问题**：地理位置结果仅存储在组件 state 中，页面刷新后需要重新获取。

**建议优化**：将获取到的位置存入 localStorage：

```javascript
useEffect(() => {
  const savedLocation = localStorage.getItem('weatherLocation');
  if (savedLocation && !options.latitude) {
    setLocation(JSON.parse(savedLocation));
  }
}, [options.latitude]);

// 获取位置后保存
const onPositionSuccess = (position) => {
  const loc = { latitude: position.coords.latitude, longitude: position.coords.longitude };
  setLocation(loc);
  localStorage.setItem('weatherLocation', JSON.stringify(loc));
};
```

---

## 八、三组天气数据源逐行对比分析

本节按代码执行顺序，逐一剖析 WeatherAPI、OpenWeatherMap、OpenMeteo 三组数据源在地区参数获取、API 密钥 / 提供者解析、缓存刷新策略和渲染入口上的异同。

### 8.1 渲染入口：Widget 映射到组件

三个天气组件均通过 [widget.jsx](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/widget.jsx#L4-L17) 的 `widgetMappings` 注册，由 `dynamic()` 懒加载：

```javascript
const widgetMappings = {
  weatherapi:      dynamic(() => import("components/widgets/weather/weather")),
  openweathermap:  dynamic(() => import("components/widgets/openweathermap/weather")),
  openmeteo:       dynamic(() => import("components/widgets/openmeteo/openmeteo")),
};
```

映射关系：`widgets.yaml` 中的键名（如 `openmeteo`）→ `widgetMappings[type]` → 对应组件的 `default export`。

三个组件在 [index.jsx#L42](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/index.jsx#L42) 的 `rightAlignedWidgets` 中均被标记为右对齐，因此天气栏始终出现在信息栏右侧：

```javascript
const rightAlignedWidgets = ["weatherapi", "openweathermap", "weather", "openmeteo", "search", "datetime"];
```

### 8.2 地区参数获取

三个组件的外层容器结构几乎完全一致，均采用相同的地区获取策略：

#### 共同流程

```
组件挂载
   │
   ├─ 1. 检查 options.latitude / options.longitude（来自 widgets.yaml 静态配置）
   │      └─ 有值 → setLocation({ latitude, longitude })  ← 渲染阶段同步执行
   │
   └─ 2. useEffect：无静态坐标时检查浏览器定位权限
          └─ navigator.permissions.query({ name: "geolocation" })
               ├─ "granted" → requestLocation() 静默获取
               └─ 其他     → 显示授权按钮，等用户点击
```

#### 关键差异

| 维度 | WeatherAPI | OpenWeatherMap | OpenMeteo |
|------|-----------|----------------|-----------|
| 外层组件名 | `WeatherApi` | `OpenWeatherMap` | `OpenMeteo` |
| 静态配置字段 | `latitude` / `longitude` | `latitude` / `longitude` | `latitude` / `longitude` |
| 额外地区字段 | 无 | 无 | `timezone`（传给 API） |
| `ContainerButton` 附加类名 | 无 | 无 | `information-widget-openmeteo-location-button` |
| 地理位置参数 | 完全相同 | 完全相同 | 完全相同 |

> **结论**：三个组件的地区获取逻辑是**复制粘贴**的同一段代码，逻辑完全一致。唯一区别是 OpenMeteo 支持 `timezone` 参数。

### 8.3 API 密钥 / 提供者解析

这是三个数据源**差异最大**的部分。

#### WeatherAPI — [weather.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/weather.js#L1-L30)

```javascript
const { latitude, longitude, provider, cache, lang, index } = req.query;

// 第一步：从 widgets.yaml 私有选项中取 apiKey
const privateWidgetOptions = await getPrivateWidgetOptions("weatherapi", index);
let { apiKey } = privateWidgetOptions;

// 第二步：没有 apiKey 也没有 provider → 报错
if (!apiKey && !provider) {
  return res.status(400).json({ error: "Missing API key or provider" });
}

// 第三步：没有 apiKey 但有 provider → 只接受 "weatherapi"
if (!apiKey && provider !== "weatherapi") {
  return res.status(400).json({ error: "Invalid provider for endpoint" });
}

// 第四步：provider === "weatherapi" → 从 settings.yaml 全局 providers 取
if (!apiKey && provider) {
  const settings = getSettings();
  apiKey = settings?.providers?.weatherapi;
}

// 第五步：还是没有 → 报错
if (!apiKey) {
  return res.status(400).json({ error: "Missing API key" });
}
```

**密钥获取链路**：

```
widgets.yaml 中该 widget 的 apiKey 字段（私有选项）
       │
       │ 没找到
       ▼
前端传入 provider 参数 === "weatherapi" ?
       │
       │ 是
       ▼
settings.yaml 中 providers.weatherapi
       │
       │ 还没找到
       ▼
返回 400 错误
```

#### OpenWeatherMap — [openweathermap.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openweathermap.js#L1-L30)

代码结构**与 WeatherAPI 完全相同**，仅字符串替换：

```javascript
const privateWidgetOptions = await getPrivateWidgetOptions("openweathermap", index);
//                                        ^^^^^^^^^^^^^^^^
if (!apiKey && provider !== "openweathermap") {
//                           ^^^^^^^^^^^^^^^
  return res.status(400).json({ error: "Invalid provider for endpoint" });
}

if (!apiKey && provider) {
  const settings = getSettings();
  apiKey = settings?.providers?.openweathermap;
  //                             ^^^^^^^^^^^^^^^
}
```

**密钥获取链路**：

```
widgets.yaml 中该 widget 的 apiKey 字段（私有选项）
       │
       │ 没找到
       ▼
前端传入 provider 参数 === "openweathermap" ?
       │
       │ 是
       ▼
settings.yaml 中 providers.openweathermap
       │
       │ 还没找到
       ▼
返回 400 错误
```

#### OpenMeteo — [openmeteo.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openmeteo.js#L1-L9)

```javascript
const { latitude, longitude, units, cache, timezone } = req.query;
const degrees = units === "metric" ? "celsius" : "fahrenheit";
const timezeone = timezone ?? "auto";
const apiUrl = `https://api.open-meteo.com/v1/forecast?...`;
return res.send(await cachedRequest(apiUrl, cache));
```

**无需 API 密钥**。Open-Meteo 是免费的开放 API，不需要注册和认证。因此整个 handler 不涉及任何密钥解析逻辑，代码极为精简。

#### 三者对比

| 维度 | WeatherAPI | OpenWeatherMap | OpenMeteo |
|------|-----------|----------------|-----------|
| 是否需要 API Key | **是** | **是** | **否** |
| 密钥来源1：widgets.yaml 内联 | `apiKey` 字段 | `apiKey` 字段 | — |
| 密钥来源2：settings.yaml 全局 | `providers.weatherapi` | `providers.openweathermap` | — |
| provider 参数校验 | `"weatherapi"` | `"openweathermap"` | — |
| 无密钥时行为 | 返回 400 | 返回 400 | — |
| `getPrivateWidgetOptions` 的 type 参数 | `"weatherapi"` | `"openweathermap"` | — |

### 8.4 API 请求构造与参数传递

#### WeatherAPI

**客户端 → 服务端请求**：[weather.jsx#L18-L20](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/weather/weather.jsx#L18-L20)

```javascript
useSWR(`/api/widgets/weather?${new URLSearchParams({ lang: i18n.language, ...options }).toString()}`);
```

传递的查询参数：`lang` + `options` 展开的全部字段（`latitude`, `longitude`, `units`, `cache`, `provider`, `index`, `label`, `format` 等）

**服务端 → 第三方 API**：

```
http://api.weatherapi.com/v1/current.json?q={latitude},{longitude}&key={apiKey}&lang={lang}
```

注意 WeatherAPI 的坐标通过 `q` 参数传递，格式为 `lat,lon` 逗号分隔。

#### OpenWeatherMap

**客户端 → 服务端请求**：[openweathermap/weather.jsx#L18-L20](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openweathermap/weather.jsx#L18-L20)

```javascript
useSWR(`/api/widgets/openweathermap?${new URLSearchParams({ lang: i18n.language, ...options }).toString()}`);
```

传递的查询参数与 WeatherAPI 相同。

**服务端 → 第三方 API**：

```
https://api.openweathermap.org/data/2.5/weather?lat={latitude}&lon={longitude}&appid={apiKey}&units={units}&lang={lang}
```

注意 OpenWeatherMap 坐标通过 `lat` / `lon` 两个独立参数传递，且传入了 `units` 参数。

#### OpenMeteo

**客户端 → 服务端请求**：[openmeteo.jsx#L18](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx#L18)

```javascript
useSWR(`/api/widgets/openmeteo?${new URLSearchParams({ ...options }).toString()}`);
```

**注意**：OpenMeteo **不传 `lang` 参数**！这是唯一不传语言参数的天气组件。Open-Meteo API 本身不提供多语言描述，因此不需要语言参数。

**服务端 → 第三方 API**：

```
https://api.open-meteo.com/v1/forecast?latitude={latitude}&longitude={longitude}&daily=sunrise,sunset&current_weather=true&temperature_unit={celsius|fahrenheit}&timezone={timezone|auto}
```

OpenMeteo 额外请求了 `daily=sunrise,sunset` 日出日落数据，用于客户端计算昼夜状态。

#### 三者对比

| 维度 | WeatherAPI | OpenWeatherMap | OpenMeteo |
|------|-----------|----------------|-----------|
| 客户端传 `lang` | **是** (`i18n.language`) | **是** (`i18n.language`) | **否** |
| 坐标参数格式 | `q=lat,lon` | `lat=x&lon=y` | `latitude=x&longitude=y` |
| 传 units 参数 | 否（API 自带公/英制字段） | **是** (`units=metric/imperial`) | **是** (`temperature_unit=celsius/fahrenheit`) |
| 请求日出日落 | 否（API 返回 `is_day` 字段） | 否（API 返回 `sys.sunrise`/`sys.sunset`） | **是** (`daily=sunrise,sunset`) |
| 第三方 API 域名 | `api.weatherapi.com` | `api.openweathermap.org` | `api.open-meteo.com` |

### 8.5 缓存与刷新机制

三个数据源共享**完全相同**的双层缓存架构，但细节有差异。

#### 第一层：客户端 SWR 缓存

三个组件都直接使用 `useSWR(url)` 调用，**均未设置 `refreshInterval`**，因此：

- 不会定时自动刷新天气数据
- 刷新触发条件仅依赖 SWR 默认行为：
  - 窗口重新获得焦点（`revalidateOnFocus: true`，默认开启）
  - 网络恢复（`revalidateOnReconnect: true`，默认开启）
  - 组件重新挂载

**SWR 请求 URL 差异**：

| 组件 | SWR 请求 URL 模式 |
|------|------------------|
| WeatherAPI | `/api/widgets/weather?lang=zh&latitude=xx&longitude=xx&...` |
| OpenWeatherMap | `/api/widgets/openweathermap?lang=zh&latitude=xx&longitude=xx&...` |
| OpenMeteo | `/api/widgets/openmeteo?latitude=xx&longitude=xx&...`（无 lang） |

SWR 的缓存键就是完整 URL，因此不同参数组合会生成不同的缓存条目。

#### 第二层：服务端内存缓存

三个 API handler 都调用同一个 `cachedRequest(url, cache)` 函数。

**WeatherAPI 调用**：[weather.js#L29](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/weather.js#L29)

```javascript
return res.send(await cachedRequest(apiUrl, cache));
```

**OpenWeatherMap 调用**：[openweathermap.js#L29](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openweathermap.js#L29)

```javascript
return res.send(await cachedRequest(apiUrl, cache));
```

**OpenMeteo 调用**：[openmeteo.js#L9](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/openmeteo.js#L9)

```javascript
return res.send(await cachedRequest(apiUrl, cache));
```

`cachedRequest` 的 `cache` 参数来自客户端查询参数 `req.query.cache`，对应 `widgets.yaml` 中的 `cache` 配置项。`duration` 参数的单位是**分钟**，默认值为 `5`（5 分钟）。

缓存键为完整的第三方 API URL，因此：
- 同一地点 + 同一 API 的请求会命中缓存
- 不同 `lang` 参数的 WeatherAPI 请求会生成不同缓存条目（因为 URL 不同）

#### 缓存刷新完整链路

```
用户切换标签页 / 重新聚焦窗口
        │
        ▼
SWR 检测到 focus 事件 → 发起 revalidate
        │
        ▼
fetch("/api/widgets/openmeteo?latitude=50&longitude=30&...")
        │
        ▼
API handler 取出 cache 参数
        │
        ▼
cachedRequest(apiUrl, cache)
        │
        ├─ 内存缓存命中 → 直接返回（不请求第三方）
        │
        └─ 内存缓存未命中 / 过期 → 请求第三方 API → 存入缓存 → 返回
```

### 8.6 数据解析与渲染

三个组件的内部 `Widget` 函数组件负责解析 API 返回数据并渲染，**数据结构差异导致解析逻辑完全不同**。

#### WeatherAPI — [weather.jsx#L15-L54](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/weather/weather.jsx#L15-L54)

```javascript
// 天气状况代码
const condition = data.current.condition.code;

// 昼夜判断：API 直接返回 is_day 布尔值
const timeOfDay = data.current.is_day ? "day" : "night";

// 温度：API 同时返回公制和英制
const value = options.units === "metric" ? data.current.temp_c : data.current.temp_f;

// 天气描述：API 返回多语言 condition.text
<SecondaryText>{data.current.condition.text}</SecondaryText>

// 图标映射：使用 condition-map.js（WeatherAPI 代码体系 1000-1282）
<WidgetIcon icon={mapIcon(condition, timeOfDay)} />
```

**API 返回数据结构（关键字段）**：
```json
{
  "current": {
    "temp_c": 22.5,
    "temp_f": 72.5,
    "is_day": 1,
    "condition": {
      "code": 1003,
      "text": "Partly cloudy"
    }
  }
}
```

#### OpenWeatherMap — [openweathermap/weather.jsx#L15-L50](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openweathermap/weather.jsx#L15-L50)

```javascript
// 天气状况代码
const condition = data.weather[0].id;

// 昼夜判断：比较当前时间戳与日出日落时间戳
const timeOfDay = data.dt > data.sys.sunrise && data.dt < data.sys.sunset ? "day" : "night";

// 温度：API 根据 units 参数返回对应单位的温度
const value = data.main.temp;

// 天气描述：API 返回多语言 weather[0].description
<SecondaryText>{data.weather[0].description}</SecondaryText>

// 图标映射：使用 owm-condition-map.js（OWM 代码体系 200-804）
<WidgetIcon icon={mapIcon(condition, timeOfDay)} />

// 错误检测：检查 cod === 401（认证失败）
if (data?.cod === 401) return <Error />;
```

**API 返回数据结构（关键字段）**：
```json
{
  "main": { "temp": 22.5 },
  "weather": [{ "id": 801, "description": "few clouds" }],
  "dt": 1718000000,
  "sys": { "sunrise": 1717970000, "sunset": 1718020000 }
}
```

#### OpenMeteo — [openmeteo.jsx#L15-L55](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx#L15-L55)

```javascript
// 天气状况代码
const condition = data.current_weather.weathercode;

// 昼夜判断：比较当前时间字符串与日出日落字符串（ISO 格式）
const timeOfDay =
  data.current_weather.time > data.daily.sunrise[0] &&
  data.current_weather.time < data.daily.sunset[0]
    ? "day" : "night";

// 温度：API 根据 temperature_unit 返回对应单位
const value = data.current_weather.temperature;

// 天气描述：使用 i18n 翻译键 wmo.{code}-{day|night}
<SecondaryText>{t(`wmo.${data.current_weather.weathercode}-${timeOfDay}`)}</SecondaryText>

// 图标映射：使用 openmeteo-condition-map.js（WMO 代码体系 0-99）
<WidgetIcon icon={mapIcon(condition, timeOfDay)} />
```

**API 返回数据结构（关键字段）**：
```json
{
  "current_weather": {
    "temperature": 22.5,
    "weathercode": 3,
    "time": "2024-06-10T14:00"
  },
  "daily": {
    "sunrise": ["2024-06-10T05:30"],
    "sunset": ["2024-06-10T21:00"]
  }
}
```

#### 渲染差异对比

| 维度 | WeatherAPI | OpenWeatherMap | OpenMeteo |
|------|-----------|----------------|-----------|
| 温度字段 | `current.temp_c` / `temp_f` | `main.temp` | `current_weather.temperature` |
| 昼夜判断 | `current.is_day`（API 直接返回） | `dt > sys.sunrise && dt < sys.sunset` | `time > daily.sunrise[0] && time < daily.sunset[0]` |
| 天气代码字段 | `current.condition.code` | `weather[0].id` | `current_weather.weathercode` |
| 天气描述来源 | API 返回 `condition.text` | API 返回 `weather[0].description` | **i18n 翻译键** `wmo.{code}-{day\|night}` |
| 图标映射文件 | `condition-map.js` | `owm-condition-map.js` | `openmeteo-condition-map.js` |
| 天气代码体系 | WeatherAPI (1000-1282) | OWM (200-804) | WMO (0-99) |
| 认证失败检测 | `data?.error` | `data?.cod === 401` | `data?.error` |
| 容器 CSS 类名 | `information-widget-weather` | `information-widget-openweathermap` | `information-widget-openmeteo` |

### 8.7 昼夜判断逻辑对比

三种 API 对"当前是白天还是黑夜"的判断方式差异显著：

**WeatherAPI**：最简单，API 直接返回 `is_day: 1 | 0`
```javascript
const timeOfDay = data.current.is_day ? "day" : "night";
```

**OpenWeatherMap**：通过 Unix 时间戳比较
```javascript
const timeOfDay = data.dt > data.sys.sunrise && data.dt < data.sys.sunset ? "day" : "night";
```

**OpenMeteo**：通过 ISO 时间字符串比较
```javascript
const timeOfDay =
  data.current_weather.time > data.daily.sunrise[0] &&
  data.current_weather.time < data.daily.sunset[0]
    ? "day" : "night";
```

> OpenMeteo 的字符串比较在此场景下是可靠的，因为 ISO 格式时间字符串的字典序与时间序一致。

### 8.8 天气描述多语言策略对比

| 数据源 | 描述来源 | 多语言支持方式 |
|--------|---------|---------------|
| WeatherAPI | API 返回 `condition.text` | API 本身支持 `lang` 参数，返回对应语言的描述 |
| OpenWeatherMap | API 返回 `weather[0].description` | API 本身支持 `lang` 参数，返回对应语言的描述 |
| OpenMeteo | **不返回描述文本** | 使用 i18n 翻译键 `wmo.{code}-{day\|night}`，描述文本在前端本地化文件中维护 |

OpenMeteo 的策略意味着天气描述的翻译质量取决于项目自身的本地化文件，而非第三方 API。这种方式的优势是：即使 API 不支持某语言，前端仍可自行提供翻译。

### 8.9 完整数据流对比图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        widgets.yaml 配置                             │
│                                                                      │
│  - weatherapi:          - openweathermap:       - openmeteo:         │
│      latitude: 39.9        latitude: 39.9          latitude: 39.9   │
│      longitude: 116.4      longitude: 116.4        longitude: 116.4 │
│      apiKey: xxx           apiKey: xxx             timezone: ...     │
│      cache: 5              cache: 5                units: metric     │
│                             units: metric            cache: 5        │
│                             provider: ...                            │
└────────┬───────────────────────┬───────────────────────┬─────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐   ┌─────────────────────┐   ┌─────────────────┐
│ WeatherApi 组件  │   │ OpenWeatherMap 组件  │   │ OpenMeteo 组件   │
│ (weather.jsx)   │   │ (weather.jsx)       │   │ (openmeteo.jsx) │
│                 │   │                     │   │                 │
│ 地区获取 ──────  │   │ 地区获取 ──────      │   │ 地区获取 ──────  │
│ (同三份代码)     │   │ (同三份代码)         │   │ (同三份代码)     │
└────────┬────────┘   └─────────┬───────────┘   └────────┬────────┘
         │                      │                        │
         ▼                      ▼                        ▼
  useSWR(               useSWR(                  useSWR(
   /api/widgets/         /api/widgets/            /api/widgets/
    weather?              openweathermap?          openmeteo?
     lang=zh              lang=zh                  (无lang)
     &lat=...             &lat=...                 &lat=...
     &lon=...             &lon=...                 &lon=...
     &cache=5             &units=...               &units=...
     &provider=...        &cache=5                 &timezone=...
     &index=...           &provider=...            &cache=5
                          &index=...               &index=...
  )                     )                        )
         │                      │                        │
         ▼                      ▼                        ▼
┌─────────────────┐   ┌─────────────────────┐   ┌─────────────────┐
│ weather.js      │   │ openweathermap.js    │   │ openmeteo.js     │
│                 │   │                     │   │                 │
│ 1.取 widget     │   │ 1.取 widget          │   │ 无需密钥         │
│   私有 apiKey   │   │   私有 apiKey        │   │                 │
│ 2.无 → 取       │   │ 2.无 → 取            │   │ 直接构造 API URL │
│   providers.    │   │   providers.          │   │                 │
│   weatherapi    │   │   openweathermap      │   │                 │
│ 3.仍无 → 400    │   │ 3.仍无 → 400         │   │                 │
└────────┬────────┘   └─────────┬───────────┘   └────────┬────────┘
         │                      │                        │
         ▼                      ▼                        ▼
  cachedRequest(          cachedRequest(           cachedRequest(
   weatherapi_url,         owm_url,                 openmeteo_url,
   cache_min               cache_min                cache_min
  )                       )                        )
         │                      │                        │
         ▼                      ▼                        ▼
┌─────────────────┐   ┌─────────────────────┐   ┌─────────────────┐
│ WeatherAPI      │   │ OpenWeatherMap API   │   │ Open-Meteo API  │
│ API 响应        │   │ 响应                 │   │ 响应            │
│                 │   │                     │   │                 │
│ current.temp_c  │   │ main.temp           │   │ current_weather │
│ current.temp_f  │   │ weather[0].id       │   │   .temperature  │
│ current.is_day  │   │ weather[0].desc     │   │   .weathercode  │
│ current.cond.   │   │ sys.sunrise/sunset  │   │   .time         │
│   code / text   │   │ dt                  │   │ daily.sunrise   │
│                 │   │                     │   │ daily.sunset    │
└────────┬────────┘   └─────────┬───────────┘   └────────┬────────┘
         │                      │                        │
         ▼                      ▼                        ▼
┌─────────────────┐   ┌─────────────────────┐   ┌─────────────────┐
│ condition-map   │   │ owm-condition-map   │   │ openmeteo-      │
│ .js             │   │ .js                 │   │ condition-map.js│
│ (1000-1282)     │   │ (200-804)           │   │ (0-99 WMO)      │
└─────────────────┘   └─────────────────────┘   └─────────────────┘
```
---

## 九、密钥与缓存边界深度剖析

### 9.1 私有密钥清洗链路

天气组件的 API 密钥不会暴露给前端，而是通过「清洗 → 索引回查」的机制在服务端安全获取。

#### 9.1.1 完整清洗流程

**第一步：读取原始配置 — [widgetsFromConfig()](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/config/widget-helpers.js#L8-L27)**

```javascript
const widgetsArray = widgets.map((group, index) => ({
  type: Object.keys(group)[0],
  options: {
    index,                              // 添加数组索引
    ...group[Object.keys(group)[0]],   // 展开所有配置（含 apiKey 等敏感字段）
  },
}));
```

- 将 YAML 中的键值对转换为 `{ type, options }` 结构
- **`index` 字段由 YAML 数组位置决定**，是后续回查的关键
- 此时 `options` 中包含完整的敏感信息（`apiKey`、`password` 等）

**第二步：清洗敏感字段 — [cleanWidgetGroups()](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/config/widget-helpers.js#L29-L54)**

```javascript
// 要删除的私有字段
["username", "password", "key", "apiKey"].forEach((pO) => {
  if (optionKeys.includes(pO)) {
    delete sanitizedOptions[pO];
  }
});

// 非 search/glances 类型还要删除 url
if (widget.type !== "search" && widget.type !== "glances" && optionKeys.includes("url")) {
  delete sanitizedOptions.url;
}

// 重新添加 index
return {
  type: widget.type,
  options: {
    index,
    ...sanitizedOptions,
  },
};
```

清洗规则：
- **始终删除**：`username`、`password`、`key`、`apiKey`
- **条件删除**：非 `search` / `glances` 类型的 `url`
- **始终保留**：`index`（即使原始配置中没有，也会重新附加）

**第三步：发送给前端 — [widgetsResponse()](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/config/api-response.js#L73-L85)**

```javascript
export async function widgetsResponse() {
  configuredWidgets = cleanWidgetGroups(await widgetsFromConfig());
  return configuredWidgets;
}
```

前端通过 `/api/widgets` 拿到的就是清洗后的数据，**不含任何密钥**。

#### 9.1.2 索引回查机制

当后端 API handler 需要获取密钥时，通过 `type + index` 回查原始配置：

**回查函数 — [getPrivateWidgetOptions()](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/utils/config/widget-helpers.js#L56-L78)**

```javascript
export async function getPrivateWidgetOptions(type, widgetIndex) {
  const widgets = await widgetsFromConfig();  // 重新读取原始配置！

  const privateOptions = widgets.map((widget) => {
    const { index, url, username, password, key, apiKey } = widget.options;
    return {
      type: widget.type,
      options: { index, url, username, password, key, apiKey },  // 只提取私有字段
    };
  }) || {};

  // 通过 type + index 定位具体 widget
  return type !== undefined && widgetIndex !== undefined
    ? privateOptions.find((o) => 
        o.type === type && o.options.index === parseInt(widgetIndex, 10)
      )?.options
    : privateOptions;
}
```

> **注意**：`getPrivateWidgetOptions()` 内部调用了 `widgetsFromConfig()`，这意味着**每次天气 API 请求都会重新读取一次 widgets.yaml 文件**。这是一个潜在的性能瓶颈。

**回查链路**：

```
前端 SWR 请求 /api/widgets/weather?index=0&latitude=...
         │
         ▼
后端 handler 调用 getPrivateWidgetOptions("weatherapi", "0")
         │
         ▼
widgetsFromConfig() 重新读取 widgets.yaml
         │
         ▼
遍历数组，找到 type === "weatherapi" 且 index === 0 的 widget
         │
         ▼
返回其 apiKey 等私有字段
```

**索引的准确性**：
- `index` 是 widget 在 `widgets.yaml` 数组中的位置（0-based）
- `widgetsFromConfig()` 和 `cleanWidgetGroups()` 都会重新生成 index
- 只要 YAML 数组顺序不变，index 就稳定
- 但如果用户在中间插入/删除 widget，后面所有 widget 的 index 都会变化

### 9.2 provider fallback 异常边界分析

WeatherAPI 和 OpenWeatherMap 的 API handler 都有一套 `provider` 参数的 fallback 逻辑，其边界条件较为微妙。

#### 9.2.1 完整决策树

以 WeatherAPI 为例 — [weather.js#L10-L25](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/pages/api/widgets/weather.js#L10-L25)

```
有 apiKey（来自 widget 私有配置）?
   ├─ 是 → 直接使用，跳过所有 provider 校验
   └─ 否 → 继续判断
            │
            ├─ 连 provider 参数都没有 → 400 "Missing API key or provider"
            │
            ├─ 有 provider，但 provider !== "weatherapi" → 400 "Invalid provider for endpoint"
            │
            └─ provider === "weatherapi" → 尝试从 settings.providers.weatherapi 取
                     │
                     ├─ 取到了 → 使用
                     └─ 没取到 → 400 "Missing API key"
```

#### 9.2.2 关键边界点

| 场景 | 行为 | 代码位置 |
|------|------|---------|
| widget 内联 `apiKey` 存在 | **忽略 provider 参数**，直接使用内联 key | 第7-8行 |
| 无内联 key，也无 `provider` 参数 | 返回 400 "Missing API key or provider" | 第10-12行 |
| 无内联 key，`provider` 是错误值 | 返回 400 "Invalid provider for endpoint" | 第14-16行 |
| 无内联 key，`provider` 正确，但 settings 中也没有 | 返回 400 "Missing API key" | 第18-25行 |
| 有内联 key 但同时有错误的 provider | **正常工作**，因为第一步就取到了 key，不会校验 provider | 隐含逻辑 |

**测试验证**：[weather.test.js](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/__tests__/pages/api/widgets/weather.test.js) 覆盖了上述四种主要场景。

#### 9.2.3 provider 参数从哪来

前端组件的 SWR 请求使用的是 `...options` 展开，`options` 来自 `widgets.yaml` 配置。由于 `provider` 不是敏感字段（不在清洗列表中），所以：

- 如果用户在 `widgets.yaml` 中配置了 `provider: weatherapi`，它会被传递到后端
- 如果没有配置，就不会传 `provider` 参数，此时必须依赖 widget 内联的 `apiKey`

> **容易混淆的设计**：`provider` 参数的主要作用是**触发 fallback 到 settings.yaml 的全局密钥**，而不是"选择 provider 类型"。因为每个 API endpoint 只对应一种 provider（如 `/api/widgets/weather` 只处理 weatherapi）。

### 9.3 静态地区与浏览器定位的优先级与覆盖

#### 9.3.1 初始化顺序与状态变化

以 OpenMeteo 组件为例 — [openmeteo.jsx#L58-L95](file:///d:/fz/0601/solo-dogfeeding/code/203-homepage/src/components/widgets/openmeteo/openmeteo.jsx#L58-L95)

```javascript
// 初始 state
const [location, setLocation] = useState(false);  // false = 未获取

// 渲染期同步设置（关键！在函数体中直接执行）
if (!location && options.latitude && options.longitude) {
  setLocation({ latitude: options.latitude, longitude: options.longitude });
}

// useEffect 中检查是否需要动态获取
useEffect(() => {
  if (!options.latitude && !options.longitude && typeof navigator !== "undefined") {
    // 只有静态坐标不存在时，才会尝试浏览器定位
    navigator.permissions?.query(...).then(...);
  }
}, [options.latitude, options.longitude, requestLocation]);
```

#### 9.3.2 执行时序详解

**场景 A：有静态配置（latitude/longitude 在 widgets.yaml 中）**

```
首次渲染
   │
   ├─ useState(false) → location = false
   │
   ├─ 进入 if 判断：!false && options.latitude && options.longitude → true
   │      └─ setLocation({ lat, lng })  ← 同步触发 state 更新
   │         （注意：这是在渲染函数中调用 setState，容易造成一次额外渲染）
   │
   ├─ 此时 location 仍是 false（当次渲染中 state 不会立即更新）
   │      └─ 走 if (!location) 分支 → 显示授权按钮？
   │
   ▼
React 检测到 state 变化 → 触发第二次渲染
   │
   ├─ location = { lat, lng }  ← 有值了
   │
   ├─ 再次进入 if 判断：!location → false，跳过
   │
   ├─ 运行 useEffect：!options.latitude → false，直接跳过
   │      └─ 不会请求浏览器定位
   │
   └─ 渲染内部 Widget 组件，发起 API 请求
```

> **注意**：首次渲染时 `location` 还是 `false`，所以会先渲染一次授权按钮，然后立即因为 state 更新而重渲染为天气数据。这会导致一次额外的初始渲染和可能的闪烁。

**场景 B：无静态配置，依赖浏览器定位**

```
首次渲染
   │
   ├─ useState(false) → location = false
   │
   ├─ 进入 if 判断：!false && options.latitude → false（无配置）
   │      └─ 不设置 location
   │
   ├─ if (!location) → true
   │      └─ 显示授权按钮（ContainerButton）
   │
   ▼
useEffect 执行
   │
   ├─ 条件：!options.latitude && !options.longitude → true
   │
   ├─ navigator.permissions.query()
   │      ├─ "granted" → requestLocation() → 静默获取 → setLocation → 重渲染
   │      └─ 其他状态 → 保持显示授权按钮，等用户点击
   │
   ▼
用户点击授权按钮 → requestLocation() → 获取成功 → setLocation → 重渲染 → 显示天气
```

#### 9.3.3 优先级结论

| 优先级 | 位置来源 | 触发条件 | 备注 |
|-------|---------|---------|------|
| 1（最高） | widgets.yaml 静态配置 | `options.latitude && options.longitude` 存在 | 直接使用，不会尝试浏览器定位 |
| 2 | 浏览器定位（已授权） | 无静态配置 + 权限已授予 | 组件挂载后自动静默获取 |
| 3（最低） | 浏览器定位（用户点击） | 无静态配置 + 权限未授予 | 用户点击授权按钮后触发 |

**不会出现的情况**：静态配置 + 浏览器定位同时生效。因为有静态配置时，`useEffect` 的条件 `!options.latitude && !options.longitude` 为假，根本不会执行定位逻辑。

### 9.4 客户端缓存键 vs 服务端缓存键

#### 9.4.1 客户端 SWR 缓存键

SWR 的缓存键就是请求的完整 URL，由 `useSWR(url)` 中的 `url` 决定。

**WeatherAPI 客户端缓存键示例**：
```
/api/widgets/weather?lang=zh-Hans&latitude=39.9&longitude=116.4&units=metric&cache=5&provider=weatherapi&index=0&label=Beijing&format[maximumFractionDigits]=1
```

**OpenMeteo 客户端缓存键示例**：
```
/api/widgets/openmeteo?latitude=39.9&longitude=116.4&units=metric&cache=5&timezone=Asia%2FShanghai&index=0&label=Beijing&format[maximumFractionDigits]=1
```

**影响客户端缓存键的参数**：

| 参数 | WeatherAPI | OpenWeatherMap | OpenMeteo | 说明 |
|------|-----------|----------------|-----------|------|
| `latitude` | ✅ | ✅ | ✅ | 纬度 |
| `longitude` | ✅ | ✅ | ✅ | 经度 |
| `lang` | ✅ | ✅ | ❌ | 语言（OpenMeteo 不传） |
| `units` | ✅ | ✅ | ✅ | 单位 |
| `cache` | ✅ | ✅ | ✅ | 缓存 TTL（仅传参，不影响键的语义） |
| `provider` | ✅ | ✅ | ❌ | provider 标识 |
| `index` | ✅ | ✅ | ✅ | widget 索引 |
| `label` | ✅ | ✅ | ✅ | 显示标签（纯展示，但在 URL 中） |
| `format` | ✅ | ✅ | ✅ | 数字格式配置 |
| `timezone` | ❌ | ❌ | ✅ | 时区（仅 OpenMeteo） |

> 注意：`label`、`format`、`cache` 这些不影响 API 响应内容的参数也会出现在缓存键中，意味着**不同显示配置会产生不同的缓存条目**，即使返回的天气数据完全一样。

#### 9.4.2 服务端 cachedRequest 缓存键

服务端缓存键是**第三方 API 的完整 URL**，由 API handler 构造。

**WeatherAPI 服务端缓存键**：
```
http://api.weatherapi.com/v1/current.json?q=39.9,116.4&key=from-widget&lang=zh
```

**OpenWeatherMap 服务端缓存键**：
```
https://api.openweathermap.org/data/2.5/weather?lat=39.9&lon=116.4&appid=from-widget&units=metric&lang=zh
```

**OpenMeteo 服务端缓存键**：
```
https://api.open-meteo.com/v1/forecast?latitude=39.9&longitude=116.4&daily=sunrise,sunset&current_weather=true&temperature_unit=celsius&timezone=Asia/Shanghai
```

#### 9.4.3 两层缓存键的差异对比

| 维度 | 客户端 SWR 缓存键 | 服务端 cachedRequest 缓存键 |
|------|------------------|---------------------------|
| **缓存位置** | 浏览器内存 | 服务器内存（memory-cache） |
| **键构成** | 内部 API URL + 所有 options 参数 | 第三方 API 完整 URL |
| **包含密钥** | ❌ 不含 | ✅ **包含**（apiKey/appid 在 URL 中） |
| **包含 index** | ✅ 包含 | ❌ 不包含 |
| **包含 label** | ✅ 包含 | ❌ 不包含 |
| **包含 format** | ✅ 包含 | ❌ 不包含 |
| **包含 cache** | ✅ 包含（作为参数值） | ❌ 不包含（只影响 TTL） |
| **包含 provider** | ✅ 包含 | ❌ 不包含 |
| **缓存失效** | 页面刷新、组件卸载 | 内存缓存过期（TTL）、服务重启 |
| **缓存粒度** | 更细（每个显示配置一份） | 更粗（相同 API 参数共享） |

#### 9.4.4 缓存穿透与重复请求

**完整的请求链路与缓存检查点**：

```
用户刷新页面 / 聚焦窗口
        │
        ▼
  SWR 触发 revalidate
   （检查客户端缓存）
        │
        ├─ 命中且新鲜 → 直接返回（不发请求）
        │
        └─ 不命中 / 需 revalidate → 发起 fetch 请求
                  │
                  ▼
           /api/widgets/openmeteo?...
                  │
                  ▼
           后端 API handler
                  │
                  ▼
           cachedRequest(apiUrl, cache)
            （检查服务端缓存）
                  │
                  ├─ 命中 → 直接返回（不请求第三方）
                  │
                  └─ 不命中 → 请求第三方 API → 存入服务端缓存 → 返回
                           │
                           ▼
                    存入 SWR 客户端缓存
```

**两层缓存的协作**：
- SWR 缓存防止**重复的客户端请求**（如快速切标签页）
- 服务端缓存防止**重复的第三方 API 请求**（如多用户访问同一地点）
- 两层缓存的 TTL 独立：SWR 由 `refreshInterval`（未设置则永不主动过期，只在 revalidate 时更新），服务端由 `cache` 参数控制

**潜在问题**：
- 客户端缓存键包含 `label`、`format` 等展示参数，导致相同天气数据被多份缓存
- `apiKey` 出现在服务端缓存键中，如果不同用户使用不同的 key（如多租户场景），即使查询同一地点也无法共享缓存
- `getPrivateWidgetOptions()` 每次请求都重新读取文件，可能成为高并发下的瓶颈

