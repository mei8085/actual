# Actual Budget 主题系统完整解析

## 一、概述

Actual Budget 的主题系统采用「CSS 变量 + Redux 状态管理 + 应用级偏好持久化」的五层架构。主题是**应用级别的配置，而非预算文件粒度的持久化**，所有预算文件共享同一套主题设置。

```
┌─────────────────────────────────────────────────────────┐
│  持久化层 (global-store.json)                          │
│  ┌──────────────────────────────────────────────────┐ │
│  │ theme: 'light' | 'dark' | 'auto'                │ │
│  │ preferredDarkTheme: 'dark' | 'midnight'         │ │
│  │ installedCustomLightTheme: JSON string          │ │
│  │ installedCustomDarkTheme: JSON string           │ │
│  │ customCssOverride: CSS string                   │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  状态管理层 (Redux prefsSlice)                         │
│  useGlobalPref('theme') → [value, setter]             │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  主题注入层 (ThemeStyle + CustomThemeStyle)            │
│  <style> 标签注入 CSS 变量                            │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  组件库映射层 (@actual-app/components/theme)            │
│  theme.pageBackground → 'var(--color-pageBackground)' │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  视图组件消费层 (View, Text, Button 等)                 │
│  style={{ backgroundColor: theme.pageBackground }}     │
└─────────────────────────────────────────────────────────┘
```

---

## 二、持久化层：用户偏好存储

### 2.1 存储位置

主题偏好存储在**应用级全局偏好文件**中，而非单个预算文件。

| 偏好类型 | 存储位置 | 说明 |
|---------|---------|------|
| GlobalPrefs | `global-store.json` | 应用级别，所有预算文件共享 |
| MetadataPrefs | 每个预算目录下的 `metadata.json` | 预算文件级别，但**不包含主题设置** |
| SyncedPrefs | 预算数据库的 `preferences` 表 | 跨设备同步的预算级偏好，**不包含主题设置** |

### 2.2 主题相关的 GlobalPrefs 字段

定义在 `packages/loot-core/src/types/prefs.ts:99-124`

```typescript
export type Theme = 'light' | 'dark' | 'auto' | 'midnight' | string;
export type DarkTheme = 'dark' | 'midnight';

export type GlobalPrefs = Partial<{
  theme: Theme;                          // 主题模式
  preferredDarkTheme: DarkTheme;            // auto 模式下的暗色主题偏好
  installedCustomLightTheme?: string;        // 亮色自定义主题（JSON 序列化的 InstalledTheme）
  installedCustomDarkTheme?: string;         // 暗色自定义主题（JSON 序列化的 InstalledTheme）
  customCssOverride?: string;               // 用户自定义 CSS 覆盖
}>;
```

**InstalledTheme 结构**（`packages/desktop-client/src/style/customThemes.ts:15-21`）：
```typescript
export type InstalledTheme = {
  id: string;
  name: string;
  repo: string;
  cssContent: string;    // 主题 CSS 内容
  baseTheme?: BaseTheme; // 基础主题：'light' | 'dark' | 'midnight'
};
```

### 2.3 后端存储实现

**存储后端**：`packages/loot-core/src/server/preferences/app.ts:66-191`

- **保存** (`saveGlobalPrefs`): 通过 `asyncStorage.setItem()` 写入 `global-store.json`
- **加载** (`loadGlobalPrefs`): 通过 `asyncStorage.multiGet()` 读取

```typescript
// 保存主题
if (prefs.theme !== undefined) {
  await asyncStorage.setItem('theme', prefs.theme);
}

// 加载主题
const { theme } = await asyncStorage.multiGet(['theme']);
```

### 2.4 预算文件粒度的主题？

**重要说明**：Actual Budget 目前**没有实现**预算文件粒度的主题持久化。主题是**应用级别的**，所有预算文件共享同一套主题设置。

如果需要实现预算文件粒度的主题，需要将主题相关字段从 `GlobalPrefs` 移动到 `MetadataPrefs`，并在加载预算时从该预算的 `metadata.json` 中读取。详见本文第八节的可落地改造方案。

---

## 三、状态管理层：Redux + useGlobalPref

### 3.1 Redux Slice

`packages/desktop-client/src/prefs/prefsSlice.ts`

```typescript
type PrefsState = {
  local: MetadataPrefs;    // 预算文件元数据（从 metadata.json 加载）
  global: GlobalPrefs;  // 应用级全局偏好（从 global-store.json 加载）
  synced: SyncedPrefs; // 跨设备同步偏好（从预算数据库加载）
};
```

**加载流程**（`App.tsx:91`）：
```typescript
// 应用启动时加载全局偏好
await dispatch(loadGlobalPrefs());
```

### 3.2 useGlobalPref Hook

`packages/desktop-client/src/hooks/useGlobalPref.ts:12-34`

```typescript
export function useGlobalPref<K extends keyof GlobalPrefs>(
  prefName: K,
  onSaveGlobalPrefs?: () => void,
): [GlobalPrefs[K], SetGlobalPrefAction<K>] {
  const dispatch = useDispatch();
  
  // 创建 setter 函数
  const setGlobalPref = useCallback<SetGlobalPrefAction<K>>(
    value => {
      void dispatch(
        saveGlobalPrefs({
          prefs: { [prefName]: value },
          onSaveGlobalPrefs,
        }),
      );
    },
    [prefName, dispatch, onSaveGlobalPrefs],
  );
  
  // 从 Redux state 读取值
  const globalPref = useSelector(
    state => state.prefs.global?.[prefName] as GlobalPrefs[K],
  );
  
  return [globalPref, setGlobalPref];
}
```

**使用方式**：
```typescript
// 读取主题，默认值为 'auto'
const [theme, setTheme] = useGlobalPref('theme');

// 切换主题：自动触发 saveGlobalPrefs → 写入 global-store.json
setTheme('dark');
```

### 3.3 主题相关 Hooks

`packages/desktop-client/src/style/theme.tsx:36-45`

```typescript
export function useTheme() {
  const [theme = 'auto', setThemePref] = useGlobalPref('theme');
  return [theme, setThemePref] as const;
}

export function usePreferredDarkTheme() {
  const [darkTheme = 'dark', setDarkTheme] = useGlobalPref('preferredDarkTheme');
  return [darkTheme, setDarkTheme] as const;
}
```

---

## 四、主题注入层：CSS 变量注入（深入解析）

### 4.1 主题 CSS 文件结构

`packages/component-library/src/themes/`

| 文件 | 作用 |
|------|------|
| `palette.css` | 定义基础调色板变量 (`--palette-*`) |
| `light.css` | 亮色主题语义化颜色 |
| `dark.css` | 暗色主题语义化颜色 |
| `midnight.css` | 午夜主题语义化颜色 |

**调色板层** (`palette.css`)：
```css
:root {
  --palette-navy100: #e8ecf0;
  --palette-blue500: #2b8fed;
  --palette-green700: #147d64;
  /* ... 约 50+ 个调色板变量 */
}
```

**主题层** (`light.css`)：
```css
:root {
  --color-pageBackground: var(--palette-navy100);
  --color-pageText: #272630;
  --color-cardBackground: var(--palette-white);
  /* ... 约 255 个语义化颜色变量 */
}
```

### 4.2 状态层到样式注入的衔接（核心链路）

这是最容易让人困惑的部分，让我们从**Redux state 变化 → 组件重新渲染 → CSS 变量注入**的完整链路拆解：

```
Redux state.prefs.global.theme 变化
           ↓
useSelector 触发订阅组件重新渲染
           ↓
useGlobalPref('theme') 返回新值
           ↓
useTheme() 返回新的 activeTheme
           ↓
ThemeStyle 组件重新执行
           ↓
useEffect 检测到 activeTheme 依赖变化
           ↓
执行 effect 逻辑，计算新的 themeColors
           ↓
setThemeColors(newColors) 更新本地 state
           ↓
组件重新渲染，<style>{themeColors}</style> 注入新 CSS
           ↓
浏览器重新解析 --color-* 变量
           ↓
所有使用 theme.* 的组件样式自动更新
```

**关键代码**（`theme.tsx:106-164`）：
```typescript
export function ThemeStyle() {
  // 1. 从 Redux 读取状态
  const [activeTheme] = useTheme();
  const [darkThemePreference] = usePreferredDarkTheme();
  const [installedCustomLightThemeJson] = useGlobalPref('installedCustomLightTheme');
  const [installedCustomDarkThemeJson] = useGlobalPref('installedCustomDarkTheme');
  
  // 2. 本地 state 存储当前注入的 CSS
  const [themeColors, setThemeColors] = useState<string | undefined>(undefined);

  // 3. 依赖数组：任何一个变化都会重新执行 effect
  useEffect(() => {
    // ... 计算 themeColors 的逻辑 ...
  }, [activeTheme, darkThemePreference, installedCustomLightThemeJson, installedCustomDarkThemeJson]);

  // 4. 注入 CSS
  if (!themeColors) return null;
  return (
    <>
      <style>{paletteCss}</style>      {/* 调色板：永远不变 */}
      <style>{themeColors}</style>    {/* 主题颜色：随状态变化 */}
    </>
  );
}
```

**关键点理解**：
- `paletteCss` 是静态的，永远不会变化，只注入一次
- `themeColors` 是动态的，随主题状态变化而变化
- `useEffect` 的依赖数组确保了**任何相关状态变化都会触发重新计算**
- 不需要使用 React Context，因为 CSS 变量是全局的，一旦注入到 `<style>` 中，整个文档都能访问

### 4.3 ThemeStyle 组件：auto 模式明暗选择深入解析

`packages/desktop-client/src/style/theme.tsx:95-174`

这是整个主题系统最复杂的部分，让我们逐行拆解：

```typescript
export function ThemeStyle() {
  const [activeTheme] = useTheme();
  const [darkThemePreference] = usePreferredDarkTheme();
  const [installedCustomLightThemeJson] = useGlobalPref('installedCustomLightTheme');
  const [installedCustomDarkThemeJson] = useGlobalPref('installedCustomDarkTheme');
  const [themeColors, setThemeColors] = useState<string | undefined>(undefined);

  useEffect(() => {
    if (activeTheme === 'auto') {
      // ─────────────────────────────────────────────────────
      // AUTO 模式：跟随系统主题
      // ─────────────────────────────────────────────────────
      
      // 1. 解析已安装的自定义主题
      const installedLight = parseInstalledTheme(installedCustomLightThemeJson);
      const installedDark = parseInstalledTheme(installedCustomDarkThemeJson);

      // 2. 确定亮色主题 CSS
      // 优先级：自定义主题的 baseTheme → 默认 light
      const lightColors =
        (installedLight?.baseTheme && getBaseThemeColors(installedLight.baseTheme)) ||
        themes['light'].colors;

      // 3. 确定暗色主题 CSS
      // 优先级：自定义主题的 baseTheme → 用户偏好的暗色主题（dark/midnight）
      const darkColors =
        (installedDark?.baseTheme && getBaseThemeColors(installedDark.baseTheme)) ||
        themes[darkThemePreference].colors;

      // 4. 创建媒体查询监听器
      function darkThemeMediaQueryListener(event: MediaQueryListEvent) {
        if (event.matches) {
          setThemeColors(darkColors);  // 系统切到暗色
        } else {
          setThemeColors(lightColors); // 系统切到亮色
        }
      }
      
      const darkThemeMediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
      darkThemeMediaQuery.addEventListener('change', darkThemeMediaQueryListener);

      // 5. 初始设置：检查当前系统主题
      if (darkThemeMediaQuery.matches) {
        setThemeColors(darkColors);
      } else {
        setThemeColors(lightColors);
      }

      // 6. 清理：移除监听器避免内存泄漏
      return () => {
        darkThemeMediaQuery.removeEventListener('change', darkThemeMediaQueryListener);
      };
    } else {
      // ─────────────────────────────────────────────────────
      // 固定主题模式：light / dark / midnight
      // ─────────────────────────────────────────────────────
      
      const installedTheme = parseInstalledTheme(installedCustomLightThemeJson);
      
      if (installedTheme?.baseTheme) {
        // 如果自定义主题指定了基础主题，使用基础主题的颜色
        setThemeColors(
          getBaseThemeColors(installedTheme.baseTheme) ??
            themes[activeTheme as ThemeKey]?.colors,
        );
      } else {
        // 直接使用选中的主题
        setThemeColors(themes[activeTheme as ThemeKey]?.colors);
      }
    }
  }, [activeTheme, darkThemePreference, installedCustomLightThemeJson, installedCustomDarkThemeJson]);

  if (!themeColors) return null;

  return (
    <>
      <style>{paletteCss}</style>
      <style>{themeColors}</style>
    </>
  );
}
```

**auto 模式工作原理详解**：

1. **双主题准备**：auto 模式下会同时准备 `lightColors` 和 `darkColors` 两套 CSS
2. **系统主题监听**：使用 `window.matchMedia('(prefers-color-scheme: dark)')` 创建媒体查询
3. **动态切换**：系统主题变化时触发 `change` 事件，调用 `setThemeColors` 切换注入的 CSS
4. **用户偏好参与**：暗色主题使用 `preferredDarkTheme`（用户可以选择 'dark' 或 'midnight'）
5. **自定义主题集成**：如果安装了自定义主题，会优先使用自定义主题指定的 `baseTheme`

### 4.4 CustomThemeStyle 组件：自定义主题 + 迁移逻辑

`packages/desktop-client/src/style/theme.tsx:183-247`

这个组件负责两件事：
1. 注入自定义主题 CSS
2. 执行一次性的 legacy override 迁移

```typescript
export function CustomThemeStyle() {
  // 1. 执行迁移（只在需要时运行）
  useMigrateLegacyOverride();
  
  // 2. 读取状态
  const [activeTheme] = useTheme();
  const [installedCustomLightThemeJson] = useGlobalPref('installedCustomLightTheme');
  const [installedCustomDarkThemeJson] = useGlobalPref('installedCustomDarkTheme');
  const [customCssOverride] = useGlobalPref('customCssOverride');

  // 3. 计算要注入的 CSS（useMemo 缓存）
  const validatedCss = useMemo(() => {
    const safeValidate = (css: string | undefined, errorLabel: string) => {
      if (!css?.trim()) return '';
      try {
        return validateThemeCss(css);  // 安全验证，防止恶意 CSS
      } catch (error) {
        console.error(errorLabel, { error });
        return '';
      }
    };

    let baseCss = '';
    if (activeTheme === 'auto') {
      // auto 模式：用 @media 包装，让浏览器根据系统主题选择
      const lightCss = safeValidate(
        parseInstalledTheme(installedCustomLightThemeJson)?.cssContent,
        'Invalid custom light theme CSS',
      );
      if (lightCss) {
        baseCss += `@media (prefers-color-scheme: light) { ${lightCss} }\n`;
      }
      const darkCss = safeValidate(
        parseInstalledTheme(installedCustomDarkThemeJson)?.cssContent,
        'Invalid custom dark theme CSS',
      );
      if (darkCss) {
        baseCss += `@media (prefers-color-scheme: dark) { ${darkCss} }\n`;
      }
    } else {
      // 固定主题：直接注入
      baseCss = safeValidate(
        parseInstalledTheme(installedCustomLightThemeJson)?.cssContent,
        'Invalid custom theme CSS',
      );
    }

    // 用户自定义 CSS 覆盖（优先级最高）
    const overrideLayer = safeValidate(
      customCssOverride,
      'Invalid custom CSS override',
    );

    const combined = [baseCss, overrideLayer].filter(Boolean).join('\n');
    return combined || null;
  }, [activeTheme, installedCustomLightThemeJson, installedCustomDarkThemeJson, customCssOverride]);

  if (!validatedCss) return null;

  return <style id="custom-theme-active">{validatedCss}</style>;
}
```

**auto 模式下自定义主题的巧妙设计**：
- 不使用 JavaScript 监听和切换，而是直接用 CSS `@media (prefers-color-scheme)` 查询
- 浏览器会自动根据系统主题应用对应的 CSS 块
- 这样即使 JavaScript 执行有延迟，主题切换也能立即响应

### 4.5 Legacy Override 迁移逻辑深入解析

`packages/desktop-client/src/style/theme.tsx:53-89` 和 `customThemes.ts:738-772`

**背景**：旧版本中，用户的自定义 CSS 覆盖（`overrideCss`）是存储在 `InstalledTheme` 对象内部的。新版本将其提取为独立的 `customCssOverride` 全局偏好，需要做一次性迁移。

**迁移 Hook**：
```typescript
function useMigrateLegacyOverride() {
  // 读取相关偏好
  const [customCssOverride, setCustomCssOverride] = useGlobalPref('customCssOverride');
  const [installedCustomLightThemeJson, setInstalledCustomLightThemeJson] = useGlobalPref('installedCustomLightTheme');
  const [installedCustomDarkThemeJson, setInstalledCustomDarkThemeJson] = useGlobalPref('installedCustomDarkTheme');

  useEffect(() => {
    // 调用迁移函数
    const result = migrateLegacyOverride({
      existingOverride: customCssOverride,
      lightJson: installedCustomLightThemeJson,
      darkJson: installedCustomDarkThemeJson,
    });

    // 如果需要迁移
    if (!result) return;

    // 写入新的 customCssOverride
    setCustomCssOverride(result.override);
    
    // 清理旧数据：从 InstalledTheme JSON 中移除 overrideCss 字段
    if (result.newLightJson !== installedCustomLightThemeJson) {
      setInstalledCustomLightThemeJson(result.newLightJson);
    }
    if (result.newDarkJson !== installedCustomDarkThemeJson) {
      setInstalledCustomDarkThemeJson(result.newDarkJson);
    }
  }, [customCssOverride, installedCustomLightThemeJson, installedCustomDarkThemeJson, setCustomCssOverride, setInstalledCustomLightThemeJson, setInstalledCustomDarkThemeJson]);
}
```

**迁移核心逻辑**（`customThemes.ts:738-772`）：
```typescript
export function migrateLegacyOverride(params: {
  existingOverride: string | undefined;
  lightJson: string | undefined;
  darkJson: string | undefined;
}): { override: string; newLightJson: string | undefined; newDarkJson: string | undefined } | null {
  const { existingOverride, lightJson, darkJson } = params;

  // 1. 如果已经有 customCssOverride 了，说明已经迁移过，直接返回
  if (existingOverride?.trim()) {
    return null;
  }

  // 2. 尝试从旧的 InstalledTheme JSON 中提取 overrideCss
  const lightLegacy = extractLegacyOverride(lightJson);
  const darkLegacy = extractLegacyOverride(darkJson);
  
  // 3. 冲突处理：如果两个主题都有 override，亮色优先（因为 UI 只显示一个）
  const legacy = lightLegacy ?? darkLegacy;
  if (!legacy) {
    return null;  // 没有需要迁移的数据
  }

  // 4. 清理：重新序列化 InstalledTheme，自动丢弃 overrideCss 字段
  // 原理：parseInstalledTheme 只会提取已知字段，未知字段（如 overrideCss）会被丢弃
  const stripOverride = (json: string | undefined): string | undefined => {
    const parsed = parseInstalledTheme(json);
    return parsed ? serializeInstalledTheme(parsed) : json;
  };

  // 5. 返回迁移结果
  return {
    override: legacy,
    newLightJson: lightLegacy ? stripOverride(lightJson) : lightJson,
    newDarkJson: darkLegacy ? stripOverride(darkJson) : darkJson,
  };
}
```

**迁移的幂等性保证**：
- 第一次运行：提取 override，写入 customCssOverride，清理旧数据
- 第二次运行：existingOverride 已经有值，直接返回 null
- 即使组件多次重渲染，也不会重复执行迁移

**字段清理的巧妙设计**：
`parseInstalledTheme` 函数只会提取明确声明的字段（id, name, repo, cssContent, baseTheme），任何额外字段（如旧的 `overrideCss`）都会被自动丢弃。这样重新序列化后，旧字段就被清理了。

### 4.6 应用入口注入

`packages/desktop-client/src/components/App.tsx:227-228`

```tsx
<ThemeStyle />
<CustomThemeStyle />
```

**注入顺序很重要**：
1. `ThemeStyle` 先注入基础主题 CSS 变量
2. `CustomThemeStyle` 后注入自定义主题，可以覆盖基础主题的变量
3. 后者的 CSS 选择器优先级相同，但由于顺序在后，会覆盖前者

---

## 五、组件库主题映射层

### 5.1 theme 对象

`packages/component-library/src/theme.ts:1-222`

将语义化颜色名映射到 CSS 变量：

```typescript
export const theme = {
  pageBackground: 'var(--color-pageBackground)',
  pageText: 'var(--color-pageText)',
  cardBackground: 'var(--color-cardBackground)',
  buttonPrimaryBackground: 'var(--color-buttonPrimaryBackground)',
  // ... 约 200+ 个映射
};
```

### 5.2 组件消费方式

组件通过 `theme` 对象引用颜色：

```tsx
import { theme } from '@actual-app/components/theme';

function MyComponent() {
  return (
    <View
      style={{
        backgroundColor: theme.pageBackground,
        color: theme.pageText,
      }}
    >
      <Text style={{ color: theme.pageText }}>Hello</Text>
    </View>
  );
}
```

Emotion CSS 在运行时将 `theme.pageBackground` 解析为 `var(--color-pageBackground)`，浏览器再解析为实际的颜色值。

---

## 六、完整数据流

### 6.1 主题切换完整流程

```
用户点击设置主题（UI）
    ↓
setTheme('dark') 调用 useGlobalPref 返回的 setter
    ↓
dispatch(saveGlobalPrefs({ prefs: { theme: 'dark' } }))
    ↓
┌──────────────────────────────────────────────────┐
│ 1. 发送 'save-global-prefs' 消息到后端         │
│ 2. asyncStorage.setItem('theme', 'dark')        │
│ 3. 写入 global-store.json                       │
└──────────────────────────────────────────────────┘
    ↓
mergeGlobalPrefs({ theme: 'dark' }) 更新 Redux state
    ↓
useSelector 触发所有订阅组件重新渲染
    ↓
useTheme() 返回新的 theme 值
    ↓
ThemeStyle 组件 useEffect 检测到 activeTheme 依赖变化
    ↓
执行 effect 逻辑，计算新的 themeColors
    ↓
setThemeColors(themes.dark.colors) 更新本地 state
    ↓
组件重新渲染，<style> 注入新的 dark.css 内容
    ↓
浏览器重新解析 --color-* 变量
    ↓
所有使用 theme.* 的组件样式自动更新
```

### 6.2 系统主题跟随（auto 模式）

```
系统主题从亮色切换到暗色
    ↓
window.matchMedia('(prefers-color-scheme: dark)') 触发 change 事件
    ↓
darkThemeMediaQueryListener 被调用
    ↓
event.matches === true
    ↓
setThemeColors(darkColors)
    ↓
<style> 重新注入 dark.css 内容
    ↓
浏览器重新解析 --color-* 变量
    ↓
所有组件样式自动更新
```

### 6.3 应用启动加载流程

```
应用启动
    ↓
App.tsx 执行 init() 函数
    ↓
await dispatch(loadGlobalPrefs())
    ↓
发送 'load-global-prefs' 消息到后端
    ↓
asyncStorage.multiGet() 读取 global-store.json
    ↓
setPrefs() 更新 Redux state
    ↓
ThemeStyle 和 CustomThemeStyle 首次渲染
    ↓
useEffect 执行，注入初始主题 CSS
    ↓
应用显示对应主题
```

---

## 七、主题状态变更的实际影响范围（精确统计）

### 7.1 订阅点分类统计

首先明确概念：
- **Hook 定义**：只是定义函数，本身不订阅状态（不计入）
- **组件/Hook 调用**：实际调用 Hook，会订阅状态变化（计入）

**Hook 定义（不计入）**：
| Hook | 定义位置 | 说明 |
|------|---------|------|
| `useTheme()` | `theme.tsx:36-39` | 内部调用 `useGlobalPref('theme')` |
| `usePreferredDarkTheme()` | `theme.tsx:41-45` | 内部调用 `useGlobalPref('preferredDarkTheme')` |

**实际订阅组件/Hook（计入，去重后共 10 个）**：

| 序号 | 组件/Hook | 文件位置 | 订阅内容 | 场景 |
|------|-----------|----------|----------|------|
| 1 | `ThemeStyle` | `theme.tsx:95-174` | `useTheme()` + `usePreferredDarkTheme()` + `installedCustomLightTheme` + `installedCustomDarkTheme` | 预算打开/未打开 |
| 2 | `useMigrateLegacyOverride` | `theme.tsx:53-89` | `customCssOverride` + `installedCustomLightTheme` + `installedCustomDarkTheme` | 预算打开/未打开 |
| 3 | `CustomThemeStyle` | `theme.tsx:183-247` | `useTheme()` + `installedCustomLightTheme` + `installedCustomDarkTheme` + `customCssOverride` | 预算打开/未打开 |
| 4 | `useMetaThemeColor` | `hooks/useMetaThemeColor.ts:13-27` | `useTheme()` + `usePreferredDarkTheme()` | 预算打开/未打开 |
| 5 | `useTagCSS` | `hooks/useTagCSS.ts:11-44` | `useTheme()` | 预算内 |
| 6 | `ThemeSettings` | `settings/Themes.tsx:42-200+` | `useTheme()` + `usePreferredDarkTheme()` + `customCssOverride` | 预算内 |
| 7 | `ThemeInstaller` | `settings/ThemeInstaller.tsx` | `customCssOverride` | 预算内 |
| 8 | `FormulaEditor` | `formula/FormulaEditor.tsx:37` | `useTheme()` | 预算内 |
| 9 | `ThemeSelector` | `ThemeSelector.tsx:22-75` | `useTheme()` | 预算内 |
| 10 | `App.tsx` | `App.tsx:195` | `useTheme()` | 预算打开/未打开 |

### 7.2 按场景分类

| 场景 | 会重新渲染的组件/Hook | 数量 |
|------|-----------------------|------|
| 预算未打开（管理页） | `ThemeStyle`, `useMigrateLegacyOverride`, `CustomThemeStyle`, `useMetaThemeColor`, `App.tsx` | 5 |
| 预算已打开（预算页） | 全部 10 个 | 10 |

### 7.3 不会重新渲染但样式会自动更新的组件

所有使用 `theme` 对象引用 CSS 变量的组件（数量很多）：
- 浏览器自动更新 CSS 变量值
- 不需要 React 重渲染
- 这是 CSS 变量方案的一大优势

### 7.4 性能影响

主题变化的性能影响很小：
- 最多 10 个组件/Hook 重渲染
- 浏览器重新解析 CSS 变量
- 大部分组件不需要重渲染
- 实际感知不到性能影响

---

## 八、预算文件粒度主题持久化改造方案（可落地步骤）

### 8.1 改造目标

将主题从**应用级别**改为**预算文件级别**，每个预算文件可以有独立的主题设置。切换预算文件时，主题自动切换。

### 8.2 预算开关场景的代码流分析（基于真实代码）

在设计改造方案前，先理解当前预算加载/卸载的完整代码流：

#### 8.2.1 预算加载流程

```
用户选择预算文件
    ↓
dispatch(loadBudget({ id: budgetId }))
    ↓
send('load-budget', { id }) 加载预算数据库
    ↓
成功后 dispatch(loadPrefs())  [budgetfilesSlice.ts:91]
    ↓
send('load-prefs') 读取该预算的 metadata.json
    ↓
setPrefs({ local: prefs, global, synced })  [prefsSlice.ts:53-55]
    ↓
Redux state.prefs.local 从 {} → 包含 { id: 'xxx', ... }
    ↓
useMetadataPref('id') 返回 budgetId
    ↓
AppInner 组件检测到 budgetId，渲染 <FinancesApp />  [App.tsx:155]
```

**关键状态变化**：`state.prefs.local` 从空对象变为包含预算元数据的对象。

#### 8.2.2 预算关闭流程

```
用户点击关闭预算
    ↓
dispatch(closeBudget())
    ↓
dispatch(resetApp())  [budgetfilesSlice.ts:103]
    ↓
extraReducers 处理 resetApp，重置 state  [prefsSlice.ts:186-190]
    ↓
state.prefs.local 重置为 {}（但 global 保留）
    ↓
useMetadataPref('id') 返回 undefined
    ↓
AppInner 组件检测到没有 budgetId，渲染 <ManagementApp />  [App.tsx:155]
```

**关键状态变化**：`state.prefs.local` 从包含预算元数据的对象变为空对象。

#### 8.2.3 预算是否已加载的准确判断

**之前的错误**：直接用 `useMetadataPref('id')` 判断预算是否加载。

**正确的判断**：`useMetadataPref('id')` 的返回值有三种状态：

| 状态 | 返回值 | 说明 |
|------|--------|------|
| 预算未加载 | `undefined` | `state.prefs.local` 是空对象 `{}` |
| 预算已加载 | `'xxx'`（字符串） | `state.prefs.local` 包含 `{ id: 'xxx' }` |
| 加载中 | 可能是 `undefined` | 异步加载过程中 |

**正确的判断代码**：
```typescript
const [budgetId] = useMetadataPref('id');
const isBudgetLoaded = typeof budgetId === 'string' && budgetId.length > 0;
```

### 8.3 共用主题解析策略（修正版）

采用**混合模式**，同时支持全局默认主题和预算独立主题。根据预算是否加载，自动切换解析逻辑。

#### 8.3.1 解析优先级

```
预算已加载场景（isBudgetLoaded = true）：
  优先级从高到低：
  1. 当前预算的 MetadataPrefs.theme（如果有值）
  2. GlobalPrefs.theme（全局默认主题）
  3. 硬编码默认值 'auto'

预算未加载场景（isBudgetLoaded = false）：
  优先级从高到低：
  1. GlobalPrefs.theme（全局默认主题）
  2. 硬编码默认值 'auto'
```

#### 8.3.2 读写分流策略

| 场景 | 读取来源 | 写入目标 |
|------|---------|---------|
| 预算已加载 | 预算主题 → 全局主题 → 默认 | 用户可选：写入预算 或 写入全局 |
| 预算未加载 | 全局主题 → 默认 | 只能写入全局 |

#### 8.3.3 完整实现代码

```typescript
// packages/desktop-client/src/hooks/useBudgetTheme.ts
import { useMemo } from 'react';

import type { Theme, DarkTheme } from '@actual-app/core/types/prefs';

import { useGlobalPref } from './useGlobalPref';
import { useMetadataPref } from './useMetadataPref';

/**
 * 预算级主题 Hook：智能检测预算是否加载，自动切换解析策略
 * 
 * - 预算已加载：优先使用预算主题，回退到全局主题
 * - 预算未加载：只使用全局主题
 */
export function useBudgetTheme() {
  // 读取预算主题（从 metadata.json）
  const [budgetTheme, setBudgetTheme] = useMetadataPref('theme');
  // 读取全局主题（从 global-store.json）
  const [globalTheme, setGlobalTheme] = useGlobalPref('theme');
  // 检测预算是否已加载（关键：判断 budgetId 是否为有效字符串）
  const [budgetId] = useMetadataPref('id');
  
  // 预算是否已加载
  const isBudgetLoaded = typeof budgetId === 'string' && budgetId.length > 0;
  
  // 计算当前激活的主题（useMemo 缓存，避免不必要的重计算）
  const activeTheme = useMemo(() => {
    if (isBudgetLoaded) {
      // 预算已加载：预算主题 > 全局主题 > 默认 'auto'
      return budgetTheme ?? globalTheme ?? 'auto';
    } else {
      // 预算未加载：全局主题 > 默认 'auto'
      return globalTheme ?? 'auto';
    }
  }, [isBudgetLoaded, budgetTheme, globalTheme]);
  
  // 写入主题的智能函数
  const setTheme = useCallback(
    (newTheme: Theme, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded) {
        // 预算未加载时，强制写入全局
        setGlobalTheme(newTheme);
        return;
      }
      
      // 预算已加载时，根据 scope 决定写入哪里
      if (scope === 'budget') {
        setBudgetTheme(newTheme);
      } else {
        setGlobalTheme(newTheme);
      }
    },
    [isBudgetLoaded, setBudgetTheme, setGlobalTheme],
  );
  
  return [activeTheme, setTheme, isBudgetLoaded] as const;
}

/**
 * 预算级暗色主题偏好 Hook
 */
export function useBudgetPreferredDarkTheme() {
  const [budgetDarkTheme, setBudgetDarkTheme] = useMetadataPref('preferredDarkTheme');
  const [globalDarkTheme, setGlobalDarkTheme] = useGlobalPref('preferredDarkTheme');
  const [budgetId] = useMetadataPref('id');
  
  const isBudgetLoaded = typeof budgetId === 'string' && budgetId.length > 0;
  
  const activeDarkTheme = useMemo(() => {
    if (isBudgetLoaded) {
      return budgetDarkTheme ?? globalDarkTheme ?? 'dark';
    } else {
      return globalDarkTheme ?? 'dark';
    }
  }, [isBudgetLoaded, budgetDarkTheme, globalDarkTheme]);
  
  const setPreferredDarkTheme = useCallback(
    (newTheme: DarkTheme, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded) {
        setGlobalDarkTheme(newTheme);
        return;
      }
      if (scope === 'budget') {
        setBudgetDarkTheme(newTheme);
      } else {
        setGlobalDarkTheme(newTheme);
      }
    },
    [isBudgetLoaded, setBudgetDarkTheme, setGlobalDarkTheme],
  );
  
  return [activeDarkTheme, setPreferredDarkTheme, isBudgetLoaded] as const;
}

/**
 * 预算级自定义主题 Hook
 */
export function useBudgetCustomTheme() {
  const [budgetLight, setBudgetLight] = useMetadataPref('installedCustomLightTheme');
  const [budgetDark, setBudgetDark] = useMetadataPref('installedCustomDarkTheme');
  const [budgetOverride, setBudgetOverride] = useMetadataPref('customCssOverride');
  
  const [globalLight, setGlobalLight] = useGlobalPref('installedCustomLightTheme');
  const [globalDark, setGlobalDark] = useGlobalPref('installedCustomDarkTheme');
  const [globalOverride, setGlobalOverride] = useGlobalPref('customCssOverride');
  
  const [budgetId] = useMetadataPref('id');
  const isBudgetLoaded = typeof budgetId === 'string' && budgetId.length > 0;
  
  const activeLightTheme = useMemo(() => {
    return isBudgetLoaded ? budgetLight ?? globalLight : globalLight;
  }, [isBudgetLoaded, budgetLight, globalLight]);
  
  const activeDarkTheme = useMemo(() => {
    return isBudgetLoaded ? budgetDark ?? globalDark : globalDark;
  }, [isBudgetLoaded, budgetDark, globalDark]);
  
  const activeOverride = useMemo(() => {
    return isBudgetLoaded ? budgetOverride ?? globalOverride : globalOverride;
  }, [isBudgetLoaded, budgetOverride, globalOverride]);
  
  return {
    // 读取
    installedCustomLightTheme: activeLightTheme,
    installedCustomDarkTheme: activeDarkTheme,
    customCssOverride: activeOverride,
    isBudgetLoaded,
    // 写入
    setInstalledCustomLightTheme: (value: string | undefined, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded || scope === 'global') {
        setGlobalLight(value);
      } else {
        setBudgetLight(value);
      }
    },
    setInstalledCustomDarkTheme: (value: string | undefined, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded || scope === 'global') {
        setGlobalDark(value);
      } else {
        setBudgetDark(value);
      }
    },
    setCustomCssOverride: (value: string | undefined, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded || scope === 'global') {
        setGlobalOverride(value);
      } else {
        setBudgetOverride(value);
      }
    },
  };
}
```

### 8.4 改造步骤（共 15 步，详细可落地）

#### 步骤 1：扩展 MetadataPrefs 类型

**文件**：`packages/loot-core/src/types/prefs.ts`

```typescript
export type MetadataPrefs = Partial<{
  budgetName: string;
  id: string;
  lastUploaded: string;
  cloudFileId: string;
  groupId: string;
  encryptKeyId: string;
  lastSyncedTimestamp: string;
  resetClock: boolean;
  lastScheduleRun: string;
  userId: string;
  
  // ── 新增：预算文件级主题设置 ──
  theme: Theme;
  preferredDarkTheme: DarkTheme;
  installedCustomLightTheme?: string;
  installedCustomDarkTheme?: string;
  customCssOverride?: string;
}>;
```

#### 步骤 2：创建预算级主题 Hook

**文件**：`packages/desktop-client/src/hooks/useBudgetTheme.ts`（新建）

实现上文中的 `useBudgetTheme`、`useBudgetPreferredDarkTheme`、`useBudgetCustomTheme`。

#### 步骤 3：修改 theme.tsx 中的基础 Hook

**文件**：`packages/desktop-client/src/style/theme.tsx`

```typescript
// 替换 useGlobalPref 为 useBudgetTheme
export function useTheme() {
  // 只返回 [theme, setTheme]，丢弃 isBudgetLoaded（保持向后兼容）
  const [theme, setThemePref] = useBudgetTheme();
  return [theme, setThemePref] as const;
}

export function usePreferredDarkTheme() {
  const [darkTheme, setDarkTheme] = useBudgetPreferredDarkTheme();
  return [darkTheme, setDarkTheme] as const;
}
```

#### 步骤 4：修改 ThemeStyle 组件

**文件**：`packages/desktop-client/src/style/theme.tsx`

将所有 `useGlobalPref('installedCustomLightTheme')` 等调用替换为 `useBudgetCustomTheme`：

```typescript
export function ThemeStyle() {
  const [activeTheme] = useTheme();
  const [darkThemePreference] = usePreferredDarkTheme();
  
  // 使用预算级自定义主题 Hook
  const {
    installedCustomLightTheme: installedCustomLightThemeJson,
    installedCustomDarkTheme: installedCustomDarkThemeJson,
  } = useBudgetCustomTheme();
  
  const [themeColors, setThemeColors] = useState<string | undefined>(undefined);
  
  // ... 其余逻辑不变 ...
}
```

#### 步骤 5：修改 CustomThemeStyle 组件

**文件**：`packages/desktop-client/src/style/theme.tsx`

```typescript
export function CustomThemeStyle() {
  useMigrateLegacyOverride();
  
  const [activeTheme] = useTheme();
  
  // 使用预算级自定义主题 Hook
  const {
    installedCustomLightTheme: installedCustomLightThemeJson,
    installedCustomDarkTheme: installedCustomDarkThemeJson,
    customCssOverride,
  } = useBudgetCustomTheme();
  
  // ... 其余逻辑不变 ...
}
```

#### 步骤 6：修改 useMigrateLegacyOverride Hook

**文件**：`packages/desktop-client/src/style/theme.tsx`

支持迁移到预算级主题：

```typescript
function useMigrateLegacyOverride() {
  const {
    customCssOverride,
    installedCustomLightTheme: installedCustomLightThemeJson,
    installedCustomDarkTheme: installedCustomDarkThemeJson,
    isBudgetLoaded,
    setCustomCssOverride,
    setInstalledCustomLightTheme,
    setInstalledCustomDarkTheme,
  } = useBudgetCustomTheme();
  
  useEffect(() => {
    // 如果预算已加载，先尝试迁移到预算级；否则迁移到全局
    const scope = isBudgetLoaded ? 'budget' : 'global';
    
    const result = migrateLegacyOverride({
      existingOverride: customCssOverride,
      lightJson: installedCustomLightThemeJson,
      darkJson: installedCustomDarkThemeJson,
    });
    
    if (!result) return;
    
    setCustomCssOverride(result.override, scope);
    if (result.newLightJson !== installedCustomLightThemeJson) {
      setInstalledCustomLightTheme(result.newLightJson, scope);
    }
    if (result.newDarkJson !== installedCustomDarkThemeJson) {
      setInstalledCustomDarkTheme(result.newDarkJson, scope);
    }
  }, [
    customCssOverride,
    installedCustomLightThemeJson,
    installedCustomDarkThemeJson,
    isBudgetLoaded,
    setCustomCssOverride,
    setInstalledCustomLightTheme,
    setInstalledCustomDarkTheme,
  ]);
}
```

#### 步骤 7：修改 useMetaThemeColor Hook

**文件**：`packages/desktop-client/src/hooks/useMetaThemeColor.ts`

```typescript
// 将 useTheme() 和 usePreferredDarkTheme() 直接使用（因为我们已经在 theme.tsx 中修改了这两个 Hook）
```

不需要额外修改，因为 `useTheme()` 已经改为使用 `useBudgetTheme()`。

#### 步骤 8：修改 useTagCSS Hook

**文件**：`packages/desktop-client/src/hooks/useTagCSS.ts`

```typescript
// 不需要修改，useTheme() 已经改为使用预算级 Hook
```

#### 步骤 9：修改 ThemeSettings 组件

**文件**：`packages/desktop-client/src/components/settings/Themes.tsx`

增加范围选择器，让用户选择写入预算还是全局：

```typescript
export function ThemeSettings() {
  const [theme, switchTheme] = useTheme();
  const [darkTheme, switchDarkTheme] = usePreferredDarkTheme();
  
  // 获取预算加载状态
  const [, , isBudgetLoaded] = useBudgetTheme();
  
  // 新增：当前写入范围
  const [scope, setScope] = useState<'budget' | 'global'>('budget');
  
  // 切换主题时带上 scope
  const handleSwitchTheme = (newTheme: Theme) => {
    switchTheme(newTheme, scope);
  };
  
  // 切换暗色主题偏好时带上 scope
  const handleSwitchDarkTheme = (newTheme: DarkTheme) => {
    switchDarkTheme(newTheme, scope);
  };
  
  return (
    <Setting>
      {/* 新增：范围选择器（仅预算已加载时显示） */}
      {isBudgetLoaded && (
        <View style={{ flexDirection: 'row', gap: 10, marginBottom: 15 }}>
          <Button
            variant={scope === 'budget' ? 'primary' : 'normal'}
            onPress={() => setScope('budget')}
          >
            <Trans>仅当前预算</Trans>
          </Button>
          <Button
            variant={scope === 'global' ? 'primary' : 'normal'}
            onPress={() => setScope('global')}
          >
            <Trans>全局默认</Trans>
          </Button>
        </View>
      )}
      
      {/* 原有主题选择逻辑，使用 handleSwitchTheme 替代 switchTheme */}
    </Setting>
  );
}
```

#### 步骤 10：修改 ThemeInstaller 组件

**文件**：`packages/desktop-client/src/components/settings/ThemeInstaller.tsx`

使用预算级自定义主题 Hook。

#### 步骤 11：修改 ThemeSelector 组件

**文件**：`packages/desktop-client/src/components/ThemeSelector.tsx`

```typescript
// 不需要修改，useTheme() 已经改为使用预算级 Hook
```

#### 步骤 12：修改 FormulaEditor 组件

**文件**：`packages/desktop-client/src/components/formula/FormulaEditor.tsx`

```typescript
// 不需要修改，useTheme() 已经改为使用预算级 Hook
```

#### 步骤 13：修改 App.tsx 中的主题使用

**文件**：`packages/desktop-client/src/components/App.tsx:195`

```typescript
// 不需要修改，useTheme() 已经改为使用预算级 Hook
```

#### 步骤 14：添加数据迁移逻辑

**文件**：`packages/desktop-client/src/prefs/prefsSlice.ts`

在 `loadPrefs` 成功后执行一次性迁移，将全局主题复制到预算：

```typescript
export const loadPrefs = createAppAsyncThunk(
  `${sliceName}/loadPrefs`,
  async (_, { dispatch, getState }) => {
    const prefs = await send('load-prefs');
    
    // ... 原有逻辑 ...
    
    // 新增：迁移全局主题到预算（如果预算没有主题设置）
    if (prefs && prefs.id) {
      const globalPrefs = getState().prefs.global;
      const needsMigration = !prefs.theme && globalPrefs.theme;
      
      if (needsMigration) {
        await send('save-prefs', {
          id: prefs.id,
          theme: globalPrefs.theme,
          preferredDarkTheme: globalPrefs.preferredDarkTheme ?? 'dark',
          installedCustomLightTheme: globalPrefs.installedCustomLightTheme,
          installedCustomDarkTheme: globalPrefs.installedCustomDarkTheme,
          customCssOverride: globalPrefs.customCssOverride,
        });
        
        // 重新加载 prefs 以获取最新状态
        const updatedPrefs = await send('load-prefs');
        dispatch(setPrefs({
          local: updatedPrefs,
          global: globalPrefs,
          synced: getState().prefs.synced,
        }));
      }
    }
    
    return prefs;
  },
);
```

#### 步骤 15：测试验证

测试以下场景：

| 测试场景 | 预期结果 |
|---------|---------|
| 未打开预算时，修改主题 | 写入 global-store.json，管理页主题变化 |
| 打开预算 A，选择"仅当前预算"修改主题 | 写入预算 A 的 metadata.json，预算页主题变化 |
| 打开预算 A，选择"全局默认"修改主题 | 写入 global-store.json，所有预算的主题都变化（除非预算有独立设置） |
| 关闭预算 A，回到管理页 | 主题自动切回全局设置 |
| 打开预算 B（新预算，没有主题设置） | 使用全局主题设置 |
| 打开预算 B，设置独立主题 | 写入预算 B 的 metadata.json |
| 在预算 A 和预算 B 之间切换 | 主题自动切换到对应预算的设置 |
| 系统主题变化（auto 模式） | 预算有独立设置时使用预算的 auto，否则使用全局的 auto |

### 8.5 改造注意事项

1. **向后兼容**：旧的 `metadata.json` 没有主题字段，`useBudgetTheme` 会自动回退到全局主题
2. **全局主题的保留**：`GlobalPrefs.theme` 作为默认值和管理页主题，不删除
3. **设置 UI 提示**：明确告诉用户当前设置的是"当前预算"还是"全局默认"
4. **性能**：每次切换预算都会重新注入主题 CSS，这是正常的，不会有性能问题
5. **自定义主题文件大小**：如果用户安装了包含内嵌字体的自定义主题，`metadata.json` 会变大，但这是可接受的
6. **迁移的幂等性**：数据迁移只在预算没有主题设置时执行，不会重复迁移

### 8.6 改造骨架代码总结

以下是改造的核心骨架，可直接参考实现：

```typescript
// ─────────────────────────────────────────────────────
// 1. 类型扩展 (packages/loot-core/src/types/prefs.ts)
// ─────────────────────────────────────────────────────
export type MetadataPrefs = Partial<{
  // ... 原有字段 ...
  theme: Theme;
  preferredDarkTheme: DarkTheme;
  installedCustomLightTheme?: string;
  installedCustomDarkTheme?: string;
  customCssOverride?: string;
}>;

// ─────────────────────────────────────────────────────
// 2. 预算级主题 Hook (packages/desktop-client/src/hooks/useBudgetTheme.ts)
// ─────────────────────────────────────────────────────
export function useBudgetTheme() {
  const [budgetTheme, setBudgetTheme] = useMetadataPref('theme');
  const [globalTheme, setGlobalTheme] = useGlobalPref('theme');
  const [budgetId] = useMetadataPref('id');
  
  // 关键：准确判断预算是否已加载
  const isBudgetLoaded = typeof budgetId === 'string' && budgetId.length > 0;
  
  const activeTheme = useMemo(() => {
    return isBudgetLoaded
      ? budgetTheme ?? globalTheme ?? 'auto'
      : globalTheme ?? 'auto';
  }, [isBudgetLoaded, budgetTheme, globalTheme]);
  
  const setTheme = useCallback(
    (newTheme: Theme, scope: 'budget' | 'global' = 'budget') => {
      if (!isBudgetLoaded) {
        setGlobalTheme(newTheme);
      } else if (scope === 'budget') {
        setBudgetTheme(newTheme);
      } else {
        setGlobalTheme(newTheme);
      }
    },
    [isBudgetLoaded, setBudgetTheme, setGlobalTheme],
  );
  
  return [activeTheme, setTheme, isBudgetLoaded] as const;
}

// ─────────────────────────────────────────────────────
// 3. 修改 theme.tsx 中的 useTheme (packages/desktop-client/src/style/theme.tsx)
// ─────────────────────────────────────────────────────
export function useTheme() {
  const [theme, setTheme] = useBudgetTheme();
  return [theme, setTheme] as const;
}

// ─────────────────────────────────────────────────────
// 4. 修改 ThemeSettings 增加范围选择器
// ─────────────────────────────────────────────────────
// 见步骤 9 的代码

// ─────────────────────────────────────────────────────
// 5. 在 loadPrefs 中添加数据迁移逻辑
// ─────────────────────────────────────────────────────
// 见步骤 14 的代码
```

---

## 九、关键边界场景深入分析

### 9.1 预算未打开时管理页主题来源与实际行为

**问题**：没有打开预算文件时，管理页（ManagementApp）的主题从哪里来？

**代码证据链**：

1. **主题注入时机**（`App.tsx:227-228`）：
   ```tsx
   <ThemeStyle />
   <CustomThemeStyle />
   ```
   这两个组件在应用根组件中渲染，**无论是否打开预算都会渲染**。

2. **全局偏好加载时机**（`App.tsx:91`）：
   ```typescript
   await dispatch(loadGlobalPrefs());
   ```
   全局偏好（包括主题）在应用启动时就加载了，在加载预算之前。

3. **管理页也使用主题**（`ManagementApp.tsx:7, 45, 69`）：
   ```tsx
   <View style={{ height: '100%', color: theme.pageText }}>
   // 版本号颜色使用 theme.pageTextSubdued
   useMetaThemeColor(isNarrowWidth ? theme.mobileConfigServerViewTheme : undefined);
   ```
   管理页也使用 `theme` 对象引用 CSS 变量。

**实际行为结论**：

| 场景 | 主题来源 | 能否修改主题 |
|------|---------|-------------|
| 未打开预算（管理页） | `global-store.json` 中的全局主题设置 | ❌ 不能（设置页面只在预算内可见） |
| 已打开预算（预算页） | `global-store.json` 中的全局主题设置 | ✅ 可以（通过设置页面） |

**重要发现**：
- 即使没有预算，主题 CSS 变量也会被注入并生效
- 但用户无法在管理页修改主题（设置页面 `ThemeSettings` 只在 `FinancesApp` 内的 `Settings` 组件中渲染（`settings/index.tsx:240`））
- `ThemeSelector` 快速切换按钮也只在预算内的标题栏中

### 9.2 预算文件粒度方案的补充改动点

之前的 8 步方案遗漏了以下关键改动点，已补充到本章的第 6-13 步中。

---

## 十、常见问题解答

### Q: 为什么不使用 React Context 传递主题？
A: 因为主题是通过 CSS 变量实现的，一旦注入到 `<style>` 标签中就是全局的。组件只需要通过 `theme` 对象引用 CSS 变量名，不需要通过 Context 获取当前主题值。这种方式比 Context 更简单、性能更好。

### Q: auto 模式下，JavaScript 监听和 CSS @media 有什么区别？
A: `ThemeStyle` 使用 JavaScript 监听是因为需要切换基础主题 CSS（light.css / dark.css），而 `CustomThemeStyle` 使用 CSS @media 是因为自定义主题的 CSS 可以直接用媒体查询包装，让浏览器自动切换。两者配合使用。

### Q: 为什么主题不存储在预算数据库里？
A: 主题是 UI 层面的设置，不属于业务数据。存储在 `metadata.json` 中更合适，因为它是预算文件的元数据，不需要同步到其他设备（当然如果需要也可以同步）。

### Q: 切换主题时所有组件都会重新渲染吗？
A: 不会全部重新渲染。**会重新渲染的组件和 Hook**（直接订阅主题状态的）有 10 个：
1. `ThemeStyle` - 核心主题注入组件
2. `useMigrateLegacyOverride` - 迁移逻辑 Hook
3. `CustomThemeStyle` - 自定义主题注入组件
4. `useMetaThemeColor` - 更新浏览器主题色 meta 标签
5. `useTagCSS` - 生成标签 CSS 样式
6. `ThemeSettings` - 设置页面（预算内可见）
7. `ThemeInstaller` - 自定义主题安装器
8. `FormulaEditor` - 公式编辑器
9. `ThemeSelector` - 快速主题切换按钮
10. `App.tsx` - 根组件

**不会重新渲染但样式会自动更新**：
- 所有使用 `theme` 对象引用 CSS 变量的组件（通过浏览器自动更新 CSS 变量值）
- 这是 CSS 变量方案的一大优势，大部分组件不需要 React 重渲染

---

## 十一、关键文件索引

| 文件路径 | 作用 |
|---------|------|
| `packages/loot-core/src/types/prefs.ts` | 定义 GlobalPrefs、Theme、DarkTheme 类型 |
| `packages/loot-core/src/server/preferences/app.ts` | 后端持久化逻辑（load/save global prefs） |
| `packages/loot-core/src/server/prefs.ts` | 后端 metadata prefs 逻辑（load/save budget prefs） |
| `packages/desktop-client/src/prefs/prefsSlice.ts` | Redux 状态管理（load/save actions） |
| `packages/desktop-client/src/hooks/useGlobalPref.ts` | 全局偏好 Hook（读 + 写） |
| `packages/desktop-client/src/hooks/useMetadataPref.ts` | 预算元数据 Hook |
| `packages/desktop-client/src/style/theme.tsx` | ThemeStyle + CustomThemeStyle + 迁移逻辑 |
| `packages/desktop-client/src/style/customThemes.ts` | 自定义主题工具函数（验证、解析、迁移） |
| `packages/desktop-client/src/hooks/useMetaThemeColor.ts` | 浏览器主题色 meta 标签 Hook |
| `packages/desktop-client/src/hooks/useTagCSS.ts` | 标签样式 Hook |
| `packages/component-library/src/theme.ts` | 组件库主题映射（theme 对象） |
| `packages/component-library/src/themes/*.css` | CSS 变量定义（palette + 三套主题） |
| `packages/desktop-client/src/components/App.tsx` | 应用入口，主题组件注入位置 |
| `packages/desktop-client/src/components/FinancesApp.tsx` | 预算页根组件 |
| `packages/desktop-client/src/components/manager/ManagementApp.tsx` | 管理页根组件 |
| `packages/desktop-client/src/components/settings/index.tsx` | 设置页面入口 |
| `packages/desktop-client/src/components/settings/Themes.tsx` | 主题设置页面 |
| `packages/desktop-client/src/components/settings/ThemeInstaller.tsx` | 自定义主题安装器 |
| `packages/desktop-client/src/components/ThemeSelector.tsx` | 快速主题切换按钮 |
| `packages/desktop-client/src/components/formula/FormulaEditor.tsx` | 公式编辑器 |
