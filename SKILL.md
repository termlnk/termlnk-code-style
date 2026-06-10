---
name: termlnk-code-style
description: Termlnk 仓库 (DI + RxJS + 命令模式架构) 的强制编码规范。编写、审查或重构 packages/*、apps/{desktop,web,mobile}/plugins/* 下任何 .ts/.tsx 时使用,尤其涉及 DI 注入 (createIdentifier/@Inject/@Optional/@DependentOn)、Observable ($ 后缀/Subject 封装)、Disposable 生命周期、Service/Controller/Plugin 模板、命令系统 (Command/Mutation/Operation)、菜单系统 (下拉/右键/mergeMenu/triggerContextMenu)、Base46 主题与颜色 (tm: 前缀类/base_30/base_16/IThemeService/xterm 配色/换肤透明度)、拦截器、React 集成、插件配置键 (config.schema.ts/PLUGIN_CONFIG_KEY)、目录与命名规范 (plugin.ts/包名/*-mobile) 时;新建插件包或 plugin.ts 必读。
---

# Termlnk 代码规范 (DI + RxJS + 命令模式)

## 概述

本 skill 是 Termlnk 仓库的强制编码规范,建立在三大支柱上:**DI 注入**(跨模块通信)、**RxJS 响应式**(状态传播)、**命令驱动**(状态变更)。编写任何 service / controller / plugin / command / React 组件**之前**,先查 [章节索引](#章节索引--按需打开) 对应 `references/` 文件确认模板与检查清单;违反下方「关键规则速查」必须就地修正。

## 何时启用

- 在 `packages/*`、`apps/desktop/plugins/*`、`apps/web/plugins/*` 或 `apps/mobile/plugins/*` 下新增或修改 `.ts` / `.tsx` 文件
- **新建插件包**（决定目录位置、包名、文件名时 — 见 `references/13-structure.md`）
- **编写或修改 plugin.ts**（必有 config schema、必走 merge+setConfig — 见 `references/07-plugin.md`）
- 评审涉及 DI 注入、Observable、Disposable、命令系统、配置键、插件结构的代码
- 用户提到"服务/控制器/插件/命令/拦截器"相关需求
- 写测试 (`*.spec.ts`) 或搭建 test bed
- **不适用**:非 Termlnk 仓库的代码,或纯文档 / 配置改动且不触及上述任何模式时

## 关键规则速查 (违反必改)

### Observable 命名

```typescript
private readonly _xxx$ = new BehaviorSubject<T>(init);
readonly xxx$: Observable<T> = this._xxx$.asObservable();
```

接口里只暴露 `Observable<T>`,**绝不暴露 Subject**。

### Subject 选择

- 有当前值 / 状态 → `BehaviorSubject`(**必须**提供同步 getter)
- 纯事件流 → `Subject`

### 订阅清理

默认 `Disposable` + `disposeWithMe(...subscribe(...))`:

- 首选:`this.disposeWithMe(observable$.subscribe(fn))`
- 多流 pipe 短路才用 `RxDisposable` + `takeUntil(this.dispose$)`
- `disposeWithMe()` 接受 `DisposableLike`(含函数),**无需** `toDisposable`

### dispose() 清理顺序

```typescript
super.dispose();
this._xxx$.complete();   // 所有 Subject
this._map.clear();       // 所有集合
```

### DI 标识符与接口同名

```typescript
export interface IXxxService { ... }
export const IXxxService = createIdentifier<IXxxService>('<包名>.<服务名>.service');
```

### 构造函数注入 — 区分 Plugin 和其它类

```typescript
// Service / Controller (全部 private readonly)
@ICommandService private readonly _commandService: ICommandService,
@Inject(Injector) private readonly _injector: Injector,

// Plugin:_injector 用 protected readonly,_configService 用 private readonly
@Inject(Injector) protected readonly _injector: Injector,
@IConfigService private readonly _configService: IConfigService,
```

可选依赖用 `@Optional(X)`。

### 命令 ID 格式

```text
'<包名>.<command|mutation|operation>.<kebab-case-动作>'
e.g. 'terminal-ui.command.create-session'
```

### 菜单系统 — 所有动作菜单走菜单系统 + 命令系统

- ❌ 禁止业务里手写 `DropdownMenu` + 内联 `onSelect`
- 菜单项:`menuItemFactory` → `IMenuButtonItem` / `IMenuSelectorItem`,绑 `commandId`(必填,渲染层执行 `item.commandId` 非 `id`)
- schema:`mergeMenu({ '<包名>.context-menu.<scope>': {...} })`
- 触发(右键 / 左键统一):`contextMenuService.triggerContextMenu(e.nativeEvent, MENU)`
- 命令从服务读目标(focused/target service),不靠菜单参数
- 变长 / 数据驱动项 → `SELECTOR` + `selections$` + 每项 `params`
- 详见 `references/14-menu.md`

### Base46 颜色 — 禁硬编码、禁 `tm:dark:` 切色

- 颜色只用 Base46 语义类:`tm:bg-one-bg` / `tm:text-light-grey` / `tm:border-line`(前缀 `tm:` 冒号,下划线转连字符)
- ❌ 禁硬编码 `bg-gray-800` / `#1e222a`;❌ 禁 `tm:dark:bg-xxx`(颜色随主题自动切换)
- 背景层级:`black`(页面) → `one_bg`(卡片) → `one_bg2`(hover) → `one_bg3`(active)
- 新主题:`base_30` 必填 30 键、`base_16` 可 `Partial`;改终端配色改 `base_30` 语义色(非 `base_16`)
- 业务只调 `IThemeService.setTheme(theme)`,注入是框架职责;移动端用语义 token 不用 `tm:` 类
- 详见 `references/15-base46-theme.md`

### 插件配置键 — 一插件一 key (强制,即使空 config 也必须有 schema)

`<plugin-name>.config` 一个顶级 key:

- ❗每个 plugin 必有 `controllers/config.schema.ts`(KEY + 接口 + default)
- ❗构造器第一参数:`_config: Partial<IXxxConfig> = defaultPluginConfig`
- ❗`super()` 后立刻 `merge` + `setConfig(KEY, merged)`
- ❗`onStarting` 必须 `mergeOverrideWithDependencies(deps, _config.override)`
- 运行时字段 → `IConfigService.setConfig/getConfig`
- 持久化字段 → `ConfigRepository.setField/getField`(按 subKey)
- 严禁:`_config: undefined = undefined` / `IXxxConfig` 内联在 `plugin.ts`

### 插件包目录结构 (强制)

```text
<plugin-package>/
├── package.json     exports: { ".": "./src/index.ts" }
├── tsconfig.json    extends @termlnk/shared/tsconfig/base
└── src/
    ├── index.ts          facade — 唯一对外入口
    ├── plugin.ts         ❗文件名固定 (不是 xxx.plugin.ts)
    ├── controllers/
    │   └── config.schema.ts  ❗必有
    ├── services/         DI 服务实现
    ├── views/            React 组件 (UI 包)
    └── locale/           i18n
```

- 位置:`packages/*` | `apps/{desktop,web,mobile}/plugins/*`
- 命名:共享 `@termlnk/<name>` ／ mobile `@termlnk/<name>-mobile`
- 详见 `references/13-structure.md`

### 插件级依赖与工具

```typescript
@DependentOn(P1, P2) class XxxPlugin extends Plugin { ... }
```

- 配置合并:`merge({}, default, this._config)` ← 深合并
- 依赖注册:`registerDependencies(injector, deps)`
- 依赖激活:`touchDependencies(injector, deps)`
- override:`mergeOverrideWithDependencies(deps, _config.override)`

### 对称双进程契约 — 主进程 / 渲染端共享同一个接口

- 契约(interface + identifier)放契约包(`@termlnk/shared-*` 等)
- 主进程实现(`rpc-server` / `*-core`)implements 该接口
- 渲染端实现(`rpc-client`)转 tRPC implements 同一接口
- 两端各注册 `[IXxxService, { useClass: XxxService }]`
- ❌ 禁止接口后缀 `*ClientService`(泄漏运行时位置)
- 方法签名一律 `Promise<T>` / `Observable<T>`
- 纯协议门面(`IRPCClientService`)才可保留 `Client` 后缀

## 必背命名约定

| 类别 | 规则 | 示例 |
|-----|------|------|
| 接口 | `I` 前缀 | `ICommandService` |
| 接口标识符 | 与接口同名 (常量) | `export const ICommandService = createIdentifier<ICommandService>(...)` |
| 私有成员 | `_` 前缀 | `private _configService`, `private readonly _logService` |
| Observable 属性 | `$` 后缀 | `sessions$`, `_currentTheme$` |
| 服务 (接口/类/文件) | 三者都以 `Service` 结尾 | `ICommandService` / `CommandService` / `command.service.ts` |
| 控制器类 | `XxxController` | `TerminalUIController` |
| 插件类 | `XxxPlugin` | `TerminalUIPlugin` |
| 插件名常量 | `XXX_PLUGIN_NAME` | `TERMINAL_UI_PLUGIN_NAME` |
| 配置键常量 | `XXX_PLUGIN_CONFIG_KEY` | `ELECTRON_MAIN_PLUGIN_CONFIG_KEY` |
| 命令 ID | `<包名>.command.动作` | `terminal-ui.command.toggle-host-dialog` |
| React 组件文件 | PascalCase | `SessionList.tsx` |
| 其他 TS 文件 | kebab-case | `session.service.ts` |

> **`Service` 后缀是硬性的,与角色名词无关。** 任何通过 `createIdentifier` + `useClass` 注册的 DI 服务,即使其职责是「路由 / 总线 / 管理器 / 分发器 / 注册表」,**接口、实现类、文件名三者都必须以 `Service` 结尾**;不要用 `Router` / `Bus` / `Manager` / `Dispatcher` / `Registry` 之类的名词作为裸结尾。
>
> ```text
> ❌ IDeepLinkRouter / DeepLinkRouter / deep-link.router.ts
> ✅ IDeepLinkRouterService / DeepLinkRouterService / deep-link-router.service.ts
> ```
>
> 例外:`*Controller`(控制器) 与 `*Plugin`(插件) 不是服务,保留各自后缀;纯协议门面可保留 `Client` 中缀但仍以 `Service` 结尾 (`IRPCClientService`)。

## 风格硬约束 (lint 会卡)

- 缩进 2 空格、单引号 `'`、必须分号
- 箭头函数参数始终带括号:`(x) => x`
- **`if` 必须用大括号**,禁止 `if (!v) return;` 这种单行写法
- `cn()` 条件类名**必须用对象语法** `{ 'class': cond }`,禁止 `cond && 'class'`
- 静态类名通过模板字符串放进 `cn()` 第一个参数,不要把 `tm:hover:...` 拆到第二个参数
- 禁止桶导入 (从 `index.ts` 导入)
- 禁止跨包穿透导入 (导入别包内部文件)
- 禁止自包导入 (包内用包名导入自己)
- Facade 文件必须显式声明返回类型,且与外部互相隔离
- 代码内注释一律英文,只解释 *why* 不解释 *what*,禁止 emoji / 装饰边框 / 多段落

## 章节索引 — 按需打开

| # | 主题 | 文件 | 何时查阅 |
|---|------|------|---------|
| 1 | 核心架构原则 | `references/01-architecture.md` | 设计新模块前、判断职责归属时 |
| 2 | 依赖注入 (DI) | `references/02-di.md` | 创建 service、写构造函数注入时 |
| 3 | RxJS 响应式编程 | `references/03-rxjs.md` | 用到 Observable、shareReplay、switchMap、firstValueFrom 等 |
| 4 | 生命周期与资源释放 | `references/04-lifecycle.md` | 继承 Disposable/RxDisposable、写 dispose() 时 |
| 5 | Service 编写规范 | `references/05-service.md` | 新建 `*.service.ts` 时 (附完整模板 + checklist) |
| 6 | Controller 编写规范 | `references/06-controller.md` | 新建 `*.controller.ts`、菜单 activated$/disabled$、派生状态时 |
| 7 | Plugin 编写规范 | `references/07-plugin.md` | **新建 plugin**（必读 — 强制 config schema + mobile babel 限制）、配置键设计、`onStarting/onReady/onRendered/onSteady` 选择 |
| 8 | 命令系统 | `references/08-command.md` | 新建 command/mutation/operation 时 |
| 9 | 拦截器模式 | `references/09-interceptor.md` | 注册拦截器、责任链场景 |
| 10 | React 与 Observable 集成 | `references/10-react.md` | 写 React 组件,使用 `useObservable` / `useObservableRef` 时 |
| 11 | TypeScript 类型规范 | `references/11-typescript.md` | `Nullable<T>`、`readonly`、interface vs type 选择 |
| 12 | 测试规范 | `references/12-testing.md` | 写 `*.spec.ts`、搭 test bed 时 |
| 13 | 文件组织与目录 | `references/13-structure.md` | **新建插件包**（必读 — 包形态/包名/目录契约）、放新文件、决定命名时 |
| 14 | 菜单系统 (下拉/右键菜单) | `references/14-menu.md` | 写下拉菜单、右键菜单、工具栏按钮菜单时 (必读，禁手写 DropdownMenu) |
| 15 | Base46 主题系统 | `references/15-base46-theme.md` | 写带颜色的 UI、新建/编辑主题、改终端配色、做透明度/换肤时 (必读，禁硬编码色值与 `tm:dark:` 切色) |

## 使用流程

1. **写之前** — 查 [章节索引](#章节索引--按需打开) 对应文件,确认模板和检查清单
2. **写之中** — 严格按速查卡命名规则 + 风格硬约束执行,违反必须就地修正
3. **写之后** — 对照对应章节 checklist 自查 (尤其 service 章节最后的清单)
