# Actual Budget 主题系统完整解析

## 一、概述

Actual Budget 的主题系统采用「CSS 变量 + Redux 状态管理 + 应用级偏好持久化」的三层架构。主题是**应用级别的配置，而非预算文件粒度的持久化，所有预算文件共享同一套主题设置。

```
┌─────────────────────────────────────────────────────────┐
│  用户偏好持久化 (global-store.json)              │
│  ┌─────────────────────────────────────────┐    │
│  │ theme: 'light' | 'dark' | 'auto'   │    │
│  │ preferredDarkTheme: 'dark' | 'midnight'     │    │
│  │ installedCustomLightTheme: JSON string       │    │
│  │ installedCustomDarkTheme: JSON string        │    │
│  │ customCssOverride: CSS string               │    │
│  └─────────────────────────────────────────┘    │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  Redux 状态管理 (prefsSlice)                │
│  useGlobalPref('theme')                      │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  主题样式注入 (ThemeStyle + CustomThemeStyle │
│  <style> 标签注入 CSS 变量               │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  组件库主题映射 (@actual-app/components/theme │
│  theme.pageBackground → var(--color-*)       │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  视图组件消费 (View, Text, Button 等)        │
│  style={{ backgroundColor: theme.pageBackground }} │
└─────────────────────────────────────────────────┘
```

---

## 二、持久化层：用户偏好存储

### 2.1 存储位置

主题偏好存储在**应用级全局偏好文件**中，而非单个预算文件。

| 偏好类型 | 存储位置 | 说明 |
|---------|---------|------|
| GlobalPrefs | `global-store.json | 应用级别，所有预算文件共享 |
| MetadataPrefs | 每个预算目录下的 `metadata.json` | 预算文件级别，但**不包含主题设置** |
| SyncedPrefs | 预算数据库的 `preferences` 表 | 跨设备同步的预算级偏好，**不包含主题设置** |

### 2.2 主题相关的 GlobalPrefs 字段

定义在 `packages/loot-core/src/types/prefs.ts:99-124

```typescript
export type GlobalPrefs = Partial<{
  theme: Theme;                          // 'light' | 'dark' | 'auto' | 'midnight'
  preferredDarkTheme: DarkTheme;            // 'dark' | 'midnight'
  installedCustomLightTheme?: string;          // JSON string of InstalledTheme
  installedCustomDarkTheme?: string;          // JSON string of InstalledTheme
  customCssOverride?: string;                 // 用户自定义 CSS 覆盖
}>;
```

### 2.3 后端存储实现

**存储后端：`packages/loot-core/src/server/preferences/app.ts:66-191

- **保存** (`saveGlobalPrefs`): 通过 `asyncStorage.setItem()` 写入 `global-store.json
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

**重要说明**：Actual Budget 目前**没有实现预算文件粒度的主题持久化。主题是**所有预算文件共享同一套主题设置。

如果需要实现预算文件粒度的主题，需要将主题相关字段从 GlobalPrefs 移动到 MetadataPrefs，并在加载预算时从该预算的 metadata.json 中读取。

---

## 三、状态管理层：Redux + useGlobalPref

### 3.1 Redux Slice

`packages/desktop-client/src/prefs/prefsSlice.ts

```typescript
type PrefsState = {
  local: MetadataPrefs;    // 预算文件元数据
  global: GlobalPrefs;  // 应用级全局偏好
  synced: SyncedPrefs; // 跨设备同步偏好
};
```

### 3.2 useGlobalPref Hook

`packages/desktop-client/src/hooks/useGlobalPref.ts:12-34

```typescript
export function useGlobalPref<K extends keyof GlobalPrefs>(
  prefName: K,
  onSaveGlobalPrefs?: () => void,
): [GlobalPrefs[K], SetGlobalPrefAction<K>] {
  const dispatch = useDispatch();
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
  const globalPref = useSelector(
    state => state.prefs.global?.[prefName] as GlobalPrefs[K],
  );
  return [globalPref, setGlobalPref];
}
```

**使用方式：

```typescript
// 读取主题
const [theme, setTheme] = useGlobalPref('theme');
// 切换主题
setTheme('dark'); // 自动触发 saveGlobalPrefs → 写入 global-store.json
```

### 3.3 主题相关 Hooks

`packages/desktop-client/src/style/theme.tsx:36-45

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

## 四、主题注入层：CSS 变量注入

### 4.1 主题 CSS 文件结构

`packages/component-library/src/themes/

| 文件 | 作用 |
|------|------|
| `palette.css` | 定义基础调色板变量 (`--palette-*`) |
| `light.css` | 亮色主题语义化颜色 |
| `dark.css` | 暗色主题语义化颜色 |
| `midnight.css` | 午夜主题语义化颜色 |

**调色板层** (`palette.css`):
```css
:root {
  --palette-navy100: #e8ecf0;
  --palette-blue500: #2b8fed;
  --palette-green700: #147d64;
  /* ... 约 50+ 个调色板变量 */
}
```

**主题层** (`light.css`):
```css
:root {
  --color-pageBackground: var(--palette-navy100);
  --color-pageText: #272630;
  --color-cardBackground: var(--palette-white);
  /* ... 约 255 个语义化颜色变量 */
}
```

### 4.2 ThemeStyle 组件

`packages/desktop-client/src/style/theme.tsx:95-174

**核心逻辑：

```typescript
export function ThemeStyle() {
  const [activeTheme] = useTheme();
  const [darkThemePreference] = usePreferredDarkTheme();
  const [themeColors, setThemeColors] = useState<string | undefined>(undefined);

  useEffect(() => {
    if (activeTheme === 'auto') {
      // 跟随系统：监听 prefers-color-scheme 变化
      const darkThemeMediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
      
      function darkThemeMediaQueryListener(event: MediaQueryListEvent) {
        if (event.matches) {
          setThemeColors(darkColors);
        } else {
          setThemeColors(lightColors);
        }
      }
      
      darkThemeMediaQuery.addEventListener('change', darkThemeMediaQueryListener);
      
      // 初始设置
      if (darkThemeMediaQuery.matches) {
        setThemeColors(darkColors);
      } else {
        setThemeColors(lightColors);
      }
      
      return () => {
        darkThemeMediaQuery.removeEventListener('change', darkThemeMediaQueryListener);
      };
    } else {
      // 固定主题
      setThemeColors(themes[activeTheme as ThemeKey]?.colors);
    }
  }, [activeTheme, darkThemePreference, ...]);

  if (!themeColors) return null;

  return (
    <>
      <style>{paletteCss}</style>
      <style>{themeColors}</style>
    </>
  );
}
```

**auto 模式工作原理：
1. 使用 `window.matchMedia('(prefers-color-scheme: dark)')` 监听系统主题变化
2. 系统切换时动态切换注入的 CSS 变量
3. 清理时移除监听器避免内存泄漏

### 4.3 CustomThemeStyle 组件

`packages/desktop-client/src/style/theme.tsx:183-247

支持自定义主题覆盖：

```typescript
export function CustomThemeStyle() {
  const [activeTheme] = useTheme();
  const [installedCustomLightThemeJson] = useGlobalPref('installedCustomLightTheme');
  const [installedCustomDarkThemeJson] = useGlobalPref('installedCustomDarkTheme');
  const [customCssOverride] = useGlobalPref('customCssOverride');

  const validatedCss = useMemo(() => {
    if (activeTheme === 'auto') {
      // auto 模式下，使用 @media 分别为亮色和暗色模式应用不同自定义主题
      return `
        @media (prefers-color-scheme: light) { ${lightCss} }
        @media (prefers-color-scheme: dark) { ${darkCss} }
      `;
    } else {
      // 固定主题模式下，直接应用自定义主题
      return lightCss;
    }
  }, [...]);

  if (!validatedCss) return null;

  return <style id="custom-theme-active">{validatedCss}</style>;
}
```

### 4.4 应用入口注入

`packages/desktop-client/src/components/App.tsx:227-228

```tsx
<ThemeStyle />
<CustomThemeStyle />
```

这两个组件在应用根组件中渲染，确保 CSS 变量在整个应用中生效。

---

## 五、组件库主题映射层

### 5.1 theme 对象

`packages/component-library/src/theme.ts:1-222

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
用户点击设置主题
    ↓
setTheme('dark') 调用 useGlobalPref 返回的 setter
    ↓
dispatch(saveGlobalPrefs({ prefs: { theme: 'dark' } }))
    ↓
┌─────────────────────────────────────────┐
│ 1. 发送 'save-global-prefs 消息到后端 │
│ 2. asyncStorage.setItem('theme', 'dark')    │
│ 3. 写入 global-store.json               │
└─────────────────────────────────────────┘
    ↓
mergeGlobalPrefs({ theme: 'dark' }) 更新 Redux state
    ↓
useTheme() 重新计算，返回新的 theme 值
    ↓
ThemeStyle 组件 useEffect 依赖 activeTheme 变化
    ↓
setThemeColors(themes.dark.colors)
    ↓
<style> 重新注入 dark.css 内容
    ↓
浏览器重新解析 --color-* 变量
    ↓
所有使用 theme.* 的组件自动更新样式
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
所有组件样式自动更新
```

---

## 七、关键文件索引

| 文件路径 | 作用 |
|---------|------|
| `packages/loot-core/src/types/prefs.ts` | 定义 GlobalPrefs 类型 |
| `packages/loot-core/src/server/preferences/app.ts` | 后端持久化逻辑 |
| `packages/desktop-client/src/prefs/prefsSlice.ts` | Redux 状态管理 |
| `packages/desktop-client/src/hooks/useGlobalPref.ts` | 全局偏好 Hook |
| `packages/desktop-client/src/style/theme.tsx` | ThemeStyle + CustomThemeStyle |
| `packages/component-library/src/theme.ts` | 组件库主题映射 |
| `packages/component-library/src/themes/*.css` | CSS 变量定义 |
| `packages/desktop-client/src/components/App.tsx` | 主题注入入口 |

---

## 八、如何实现预算文件粒度的主题持久化（如果需要

当前主题是应用级别的，如果需要实现预算文件粒度的主题，需要做以下修改：

1. 将 `theme` 等字段从 `GlobalPrefs` 移动到 `MetadataPrefs`
2. 修改 `loadPrefs`/`savePrefs` 支持主题字段
3. 修改 `useTheme` 从 `useMetadataPref` 而非 `useGlobalPref`
4. 在加载预算时触发主题重新应用
