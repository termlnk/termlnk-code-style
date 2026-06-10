# Base46 主题系统 (颜色 / 主题切换 / xterm 适配)

> **所有颜色必须走 Base46 语义变量,严禁硬编码色值、严禁 `tm:dark:` 分支切色。**
> 主题系统实现在 `@termlnk/themes`,状态在 `@termlnk/core` 的 `IThemeService`,DOM 注入在 `@termlnk/ui` 的 `ThemeSwitcherService`。

## 硬规则 (违反必改)

- **禁止硬编码颜色**:不写 `bg-gray-800` / `#1e222a` / `rgb(...)`,只用 Base46 语义类 `tm:bg-one-bg`、`tm:text-light-grey`。
- **Tailwind 前缀是 `tm:`(冒号),不是 `tm-`(连字符)**。色名里的下划线转连字符:`one_bg` → `tm:bg-one-bg`,`grey_fg2` → `tm:text-grey-fg2`。
- **禁止 `tm:dark:` 切色**。颜色随主题自动切换(CSS 变量运行时替换),写 `tm:dark:bg-xxx` 是反模式。`tm:dark` 变体仅用于「明暗下结构/资源不同」的极少数场景。
- **新建主题颜色不全靠公式**。`base_30` 是 30 个 UI 色,`base_16` 是 16 个语法色;梯度百分比是**设计指南**,移植主题保留上游原值(见下文)。
- **`base_30` 必填全 30 个键,`base_16` 是 `Partial`**(可缺,适配器有 `base_30` 兜底)。
- **改终端配色走 `base_30` 语义色**:xterm 的 16 个 ANSI 色取自 `base_30`(`red`/`green`/`baby_pink`...),不是 `base_16`。

## 主题数据结构 (`@termlnk/themes` → `types/theme.ts`)

```typescript
export type ThemeType = 'dark' | 'light';

export interface ITheme {
  name: string;                    // 稳定 id, kebab-case: 'one-dark'
  displayName: string;             // i18n key: 'theme.one-dark'
  type: ThemeType;
  base_30: IBase30Colors;          // 30 个 UI 色, 必填全
  base_16: Partial<IBase16Colors>; // 16 个语法色, 可缺省
}

// 自定义主题 (用户/编辑器创建)
export interface ICustomTheme extends ITheme {
  isCustom: true;
  extendsFrom?: string;            // 派生自哪个内置主题的 name
}
export function isCustomTheme(theme: ITheme): theme is ICustomTheme;
```

### `base_30` — UI 颜色(30 个)

以 `black`(主背景)为 0% 基准,梯度百分比是**新建主题的设计指南**:

| 分组 | 键 | 梯度(指南) | 用途 |
|-----|----|----------|------|
| 中性背景 | `black` | 0% 基准 | 页面/窗口主背景 |
| | `darker_black` | -6% | 更深嵌套背景、禁用底 |
| | `black2` | +6% | 次级背景(侧栏 vs 主区) |
| | `one_bg` | +10% | **最常用**:卡片/面板/下拉背景 |
| | `one_bg2` | +16% | hover 背景 |
| | `one_bg3` | +22% | active/focus 背景 |
| 前景文本 | `white` | 接近白 | 主文本(`base46ToXterm` 的 fg 兜底) |
| | `light_grey` | grey+28% | 主要文本 |
| | `grey_fg2` | grey+20% | 普通文本 |
| | `grey_fg` | grey+10% | 次要文本/描述 |
| | `grey` | +40% | 禁用/占位/滚动条 |
| UI 专用 | `line` | +15% | 边框、分隔线 |
| | `statusline_bg` | +4% | 状态栏背景 |
| | `lightbg` | statusline+13% | 浅背景强调 |
| | `pmenu_bg` | 强调色 | 弹出菜单高亮 |
| | `folder_bg` | 通常 blue | 文件夹图标 |
| 语义色 | `red` `baby_pink` `pink` | — | 错误/危险/删除 (`baby_pink` ≈ red 变浅) |
| | `green` `vibrant_green` | — | 成功/新增 |
| | `blue` `nord_blue` | — | 主操作/链接 (`nord_blue` ≈ blue 变深) |
| | `yellow` `sun` | — | 警告/修改 (`sun` ≈ yellow 变亮) |
| | `purple` `dark_purple` | — | 特殊/高级 |
| | `teal` `orange` `cyan` | — | 信息/待处理/装饰 |

> **梯度只是参考**。内置主题从 [NvChad/base46](https://github.com/NvChad/base46) 移植,保留上游手调原值(如 `one-dark` 的 `one_bg3` = `#373b43`,并非 `black` +22% 的算法值)。**新建/自定义**主题才用百分比起步,再手调。

### `base_16` — 语法高亮颜色(16 个,遵循 Base16 标准)

| 键 | 语义 | 键 | 语义 |
|----|------|----|------|
| `base00` | 默认背景 | `base08` | 红 - 变量/标签 |
| `base01` | 浅背景(行号) | `base09` | 橙 - 常量/数字 |
| `base02` | 选区背景 | `base0A` | 黄 - 类名 |
| `base03` | 注释 | `base0B` | 绿 - 字符串 |
| `base04` | 深前景 | `base0C` | 青 - 转义/正则 |
| `base05` | **默认前景**(正文) | `base0D` | 蓝 - 函数 |
| `base06` | 浅前景 | `base0E` | 紫 - 关键字 |
| `base07` | 最浅前景 | `base0F` | 棕 - 废弃 |

> ⚠️ `base0A`–`base0F` 的字母**大写**,CSS 变量同样大写:`--tm-base0A`(见下文生成规则)。

## 主题定义模板 (`themes/dark/<name>.ts`)

```typescript
import type { ITheme } from '../../types';

export const oneDark: ITheme = {
  name: 'one-dark',          // kebab-case, 全局唯一
  displayName: 'theme.one-dark', // i18n key
  type: 'dark',
  base_30: {
    white: '#abb2bf',
    black: '#1e222a',
    darker_black: '#1b1f27',
    // ... 必须写全 30 个键
    blue: '#61afef',
    pmenu_bg: '#61afef',
    folder_bg: '#61afef',
  },
  base_16: {
    base00: '#1e222a',       // 通常等于 base_30.black
    base05: '#abb2bf',       // 默认前景
    base08: '#e06c75',
    // ... 16 个可按需补全
  },
};
```

新主题落地三步:① 建 `themes/{dark,light}/<name>.ts`;② 在同目录 `index.ts` 加 `export { xxx } from './<name>'`;③ 在 `src/index.ts` facade 补导出。`utils/registry.ts` 用 `Object.values(dark/light)` 自动汇总成 `ALL_THEMES` / `THEME_MAP`,**无需手动注册**。

## CSS 变量生成规则 (`utils/css-generator.ts`)

`generateCSSVariables(theme, prefix = '--tm')` 把主题转成 CSS 变量,注入 `:root`:

- `base_30`:key 下划线 → 连字符。`one_bg` → `--tm-one-bg`,`grey_fg2` → `--tm-grey-fg2`。
- `base_16`:key **原样保留**(含大写)。`base00` → `--tm-base00`,`base0A` → `--tm-base0A`。
- 注入由 `injectThemeToDOM` 写入 `<style id="tm-theme-variables">`;`removeThemeFromDOM` 移除。
- **无静态 fallback**:`:root` 里没有任何 `--tm-*` 默认值,**主题未注入前所有色为空**。首屏必须先 `setTheme`。

## Tailwind 集成 (`tm:` 前缀)

共享配置 `@termlnk/shared/tailwind/`:

```css
/* tailwind.css */
@import 'tailwindcss' prefix(tm);   /* 所有工具类前缀 tm: */
@import './theme.css';

/* theme.css —— 重置默认调色板, @theme inline 绑定运行时变量 */
@theme { --color-*: initial; }
@theme inline {
  --color-one-bg: var(--tm-one-bg);     /* tm:bg-one-bg 生效 */
  --color-light-grey: var(--tm-light-grey);
  --color-base0A: var(--tm-base0A);
  /* ... base_30 + base_16 全量映射 */
}
@custom-variant dark (&:where(.dark, .dark *));
```

各 UI 包 `src/global.css` 复用:

```css
@import 'tailwindcss/utilities' prefix(tm);
@import '@termlnk/shared/tailwind/theme.css';
```

组件里的正确写法:

```tsx
// ✅ 正确:Base46 语义类, 颜色随主题自动切换
<button className="tm:bg-one-bg hover:tm:bg-one-bg2 active:tm:bg-one-bg3 tm:text-light-grey tm:border-line">

// ❌ 错误:硬编码
<button className="bg-gray-800 hover:bg-gray-700 text-white">

// ❌ 错误:dark: 分支(颜色已自动切换, 多此一举且会失配)
<button className="tm:dark:bg-one-bg tm:bg-white">
```

> `cn()` 条件类名用对象语法、静态类名放第一个参数(见 SKILL.md「风格硬约束」)。`tm:hover:...` 整体作为一个静态类,不要把 `tm:` 与 `hover:` 拆开。

### 组件配色速查

| 组件 | 背景 | 文字 | 边框 | hover | active |
|-----|------|------|------|-------|--------|
| 页面 | `tm:bg-black` | `tm:text-light-grey` | — | — | — |
| 卡片/面板 | `tm:bg-one-bg` | `tm:text-light-grey` | `tm:border-line` | `tm:bg-one-bg2` | — |
| 按钮 primary | `tm:bg-blue` | `tm:text-white` | — | `blue/90` | `blue/80` |
| 按钮 ghost | `tm:bg-transparent` | `tm:text-light-grey` | — | `tm:bg-one-bg2` | `tm:bg-one-bg3` |
| 下拉菜单 | `tm:bg-one-bg` | `tm:text-light-grey` | `tm:border-line` | `tm:bg-one-bg2` | `tm:bg-one-bg3` |
| 状态栏 | `tm:bg-statusline-bg` | `tm:text-grey-fg` | — | — | — |
| 分隔线 | `tm:bg-line` | — | — | — | — |

语义状态:`error` 用 `tm:text-red`、`success` 用 `tm:text-green`、`warning` 用 `tm:text-yellow`、`info` 用 `tm:text-blue`(配 `/10` 背景、`/30` 边框)。

## 主题切换接线 (DI + RxJS)

`IThemeService` 是**纯状态持有者**(不碰 DOM):

```typescript
// @termlnk/core — services/theme/theme.service.ts
export interface IThemeService {
  readonly currentTheme$: Observable<ITheme | null>;
  currentTheme: ITheme | null;            // 同步 getter
  readonly themeType$: Observable<ThemeType>; // 派生自 currentTheme$
  themeType: ThemeType;                    // 默认 'dark'
  setTheme(theme: ITheme | null): void;    // null = 清空
}
export const IThemeService = createIdentifier<IThemeService>('core.theme-service');
```

DOM 注入由 `@termlnk/ui` 的 `ThemeSwitcherService` 承担,在 `Workbench` 里订阅接线:

```tsx
// 颜色注入:currentTheme$ → <style> 变量
useLayoutEffect(() => {
  const sub = themeService.currentTheme$.subscribe((theme) => {
    themeSwitcherService.injectThemeToHead(theme); // null → removeThemeFromDOM
  });
  return () => sub.unsubscribe();
}, []);

// 明暗类:themeType$ → 切 documentElement 上的 tm:dark
useLayoutEffect(() => {
  const sub = themeService.themeType$.subscribe((themeType) => {
    document.documentElement.classList.toggle('tm:dark', themeType === 'dark');
  });
  return () => sub.unsubscribe();
}, []);
```

> 业务侧**只调 `IThemeService.setTheme(theme)`**,注入是框架职责,不要在业务组件里直接调 `injectThemeToDOM`。

## 终端配色 — base46ToXterm 适配器

`adapters/base46-to-xterm.ts` 把 `ITheme` 转成 xterm.js 的 `IXtermTheme`。**关键:16 个 ANSI 色取自 `base_30` 语义色,只有背景/前景/选区取自 `base_16`**:

```typescript
base46ToXterm(theme, backgroundOpacity?) → {
  background: base_16.base00,   // 透明开启时为 '#00000000', 交给容器 color-mix
  foreground: base_16.base05,
  cursor: base_16.base05,
  selectionBackground: base_16.base02,
  // 滚动条: base_30.grey / grey_fg / light_grey
  // ANSI normal: black=base00, red/green/yellow/blue/cyan=同名 base_30, magenta=base_30.purple, white=base05
  // ANSI bright: brightBlack=base03, brightRed=baby_pink, brightGreen=vibrant_green,
  //              brightYellow=sun, brightBlue=nord_blue, brightMagenta=pink, brightCyan=teal, brightWhite=base07
}
```

注意:`magenta` ← `base_30.purple`(xterm 用 `magenta`,本系统用 `purple`);所有 `base_16` 取值都有 `base_30` 兜底,故 `base_16` 可 `Partial`。

## 窗口透明度 (`injectTransparencyToDOM`)

桌面端毛玻璃:仅**背景色**叠加 alpha,前景(文字/边框)保持不透明。

```typescript
injectTransparencyToDOM(theme, opacity); // opacity ∈ (0,1); >=1 自动移除覆盖
```

- 只覆盖背景类键:`black` `darker_black` `black2` `one_bg` `one_bg2` `one_bg3` `statusline_bg` `lightbg` `pmenu_bg`。
- 用 `color-mix(in srgb, <color> <percent>%, transparent)`,写入独立 `<style id="tm-transparency-override">`,并置 `--tm-bg-opacity` + `html { background: transparent !important }`。
- 终端侧:`base46ToXterm` 在 `opacity < 1` 时把 `background` 设为 `#00000000`,避免和容器 `color-mix` 双重叠加。

## 扩展贡献主题 (`themes` contribution point)

扩展通过 `themes` 贡献点提供 Base46 兼容的 JSON 主题:

- 加载进 `IExtensionThemeRegistry`(`extension.theme-registry`),设置 UI 的主题选择器订阅 `themes$`。
- 扩展停用时从 registry 移除其条目,但**不**重置 `IThemeService.currentTheme`——用户选中的主题跨扩展重载保持粘性。
- JSON 解析后补默认:`name ?? theme.id`、`displayName ?? theme.label`。

## 移动端范围边界 (apps/mobile)

**移动端不消费运行时 Base46 主题系统**。它用 NativeWind 的 `VariableContextProvider` + 一套**精简语义 token**(`--color-surface` / `--color-content` / `--color-accent` / `--color-divider` ...),`DARK_VARS` 取值镜像 Base46 onedark(如 `surface=#1e222a`、`accent=#61afef`)。

- 桌面/Web 渲染端 = 完整 Base46 + `tm:` 工具类 + 运行时换肤。
- 移动端 = 固定明/暗两套语义调色板,经 `useColorScheme` 跟随系统;非 className 场景(lucide 图标 `color`)用 `useThemeColors()` 取 hex。
- **不要**把 `tm:bg-one-bg` 这类类名写进 `apps/mobile`,也不要让移动端依赖 `injectThemeToDOM`。

## 对比度 (WCAG 2.0)

| 等级 | 比率 | 场景 |
|-----|------|------|
| AA | ≥ 4.5:1 | 普通文本(必需) |
| AA Large | ≥ 3:1 | 大文本 18px+ |
| AAA | ≥ 7:1 | 增强(推荐) |

- `light_grey` / `grey_fg2` 在 `black` 上必须 ≥ 4.5:1。
- `grey` 可低于 4.5:1(仅用于禁用/占位)。
- 自定义主题在编辑器保存前应过对比度检查。

## Checklist

- [ ] 无硬编码色值;所有颜色走 `tm:` 语义类或 `--tm-*` 变量
- [ ] 前缀是 `tm:`(冒号)、下划线转连字符;无 `tm:dark:` 切色分支
- [ ] 新主题:`name` kebab-case 唯一、`displayName` 是 i18n key、`base_30` 写全 30 键
- [ ] 新主题已在 `themes/{dark,light}/index.ts` + `src/index.ts` 双导出
- [ ] 改终端色改的是 `base_30` 语义色(不是 `base_16`)
- [ ] 业务只调 `IThemeService.setTheme`,不直接 `injectThemeToDOM`
- [ ] 移动端用语义 token,未引入 `tm:` 类或 `@termlnk/themes` 运行时注入
- [ ] 自定义主题过 WCAG AA(正文 ≥ 4.5:1)

## 落地参考

| 场景 | 文件 |
|-----|------|
| 类型/接口 | `@termlnk/themes` `src/types/theme.ts` |
| 内置主题(57 暗 / 14 亮) | `@termlnk/themes` `src/themes/{dark,light}/*.ts` |
| CSS 变量生成 + 注入 + 透明度 | `@termlnk/themes` `src/utils/css-generator.ts` |
| 注册表汇总 | `@termlnk/themes` `src/utils/registry.ts`(`ALL_THEMES` / `THEME_MAP`) |
| xterm 适配 | `@termlnk/themes` `src/adapters/base46-to-xterm.ts` |
| Tailwind 前缀 + 变量绑定 | `internal/shared/tailwind/{tailwind,theme}.css` |
| 主题状态服务 | `@termlnk/core` `src/services/theme/theme.service.ts`(`IThemeService`) |
| DOM 注入 + 订阅接线 | `@termlnk/ui` `theme-switcher.service.ts` + `views/workbench/Workbench.tsx` |
| 扩展主题贡献点 | `@termlnk/extension-ui` `contributions/points/themes.point.ts` |
| 主题选择/编辑 UI | `@termlnk/themes-ui` `views/theme-picker` / `theme-editor` |
| 移动端语义调色板 | `apps/mobile` `src/theme/theme-provider.tsx` |
```
