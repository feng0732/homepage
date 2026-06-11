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
