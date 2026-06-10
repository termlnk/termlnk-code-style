# 07. Plugin 编写规范

## 7.1 强制规范 — 每个插件必须有 config schema (即使空)

**铁律**：所有 `Plugin` 子类**必须**遵循下面四条，不可省略任何一条，即使逻辑上不需要任何配置：

1. **必须**有 `controllers/config.schema.ts` 定义 `XXX_PLUGIN_CONFIG_KEY` + `IXxxConfig` + `defaultPluginConfig`
2. 构造器**必须**有第一参数 `_config: Partial<IXxxConfig> = defaultPluginConfig`（用 `Partial` 是因为消费方通常只传部分字段，缺省字段由 `merge(defaultPluginConfig, _config)` 补齐）
3. 构造器**必须**注入 `IConfigService` 并在 `super()` 后立刻 `merge` + `setConfig(KEY, merged)`
4. `onStarting()` 注册依赖**必须**走 `mergeOverrideWithDependencies(deps, this._config?.override)`，让消费方有 override 通道

违反任何一条都会在 `pnpm typecheck` 后被 `scripts/check-plugin-conformance.sh` 类的扫描脚本捕获。`packages/extension-ui` 历史上违反过该规范（直接 `_config: IExtensionUIConfig = {}` 且无 setConfig），是反面教材。

## 7.2 标准模板 (desktop / packages/*)

```typescript
// controllers/config.schema.ts
import type { DependencyOverride } from '@termlnk/core';

export const TERMINAL_UI_PLUGIN_CONFIG_KEY = 'terminal-ui.config';

export interface ITerminalUIConfig {
  override?: DependencyOverride;
  // ... 其他业务字段 + 持久化字段(可选)
}

export const defaultPluginConfig: ITerminalUIConfig = {};
```

```typescript
// plugin.ts
import type { Dependency } from '@termlnk/core';
import type { ITerminalUIConfig } from './controllers/config.schema';
import {
  DependentOn,
  IConfigService,
  Inject,
  Injector,
  merge,
  mergeOverrideWithDependencies,
  Plugin,
  registerDependencies,
  touchDependencies,
} from '@termlnk/core';
import { RPCClientPlugin } from '@termlnk/rpc-client';
import { UIPlugin } from '@termlnk/ui';
import { defaultPluginConfig, TERMINAL_UI_PLUGIN_CONFIG_KEY } from './controllers/config.schema';

export const TERMINAL_UI_PLUGIN_NAME = 'TERMINAL_UI_PLUGIN';

@DependentOn(UIPlugin, RPCClientPlugin)
export class TerminalUIPlugin extends Plugin {
  static override pluginName = TERMINAL_UI_PLUGIN_NAME;

  constructor(
    private readonly _config: Partial<ITerminalUIConfig> = defaultPluginConfig,
    @Inject(Injector) protected readonly _injector: Injector,
    @IConfigService private readonly _configService: IConfigService
  ) {
    super();

    // Deep merge — defaults must NOT be overwritten by nested config.
    const config = merge({}, defaultPluginConfig, this._config);
    this._configService.setConfig(TERMINAL_UI_PLUGIN_CONFIG_KEY, config);
  }

  override onStarting(): void {
    const dependencies: Dependency[] = [
      [IHostExplorerService, { useClass: HostExplorerService }],
      [ITerminalUIService, { useClass: TerminalUIService }],
      [HostDialogController],
      [TerminalUIController],
    ];
    registerDependencies(
      this._injector,
      mergeOverrideWithDependencies(dependencies, this._config?.override)
    );
  }

  override onReady(): void { /* register interceptors / subscribe to command execution */ }

  override onRendered(): void {
    touchDependencies(this._injector, [
      [HostDialogController],
      [TerminalUIController],
    ]);
  }

  override onSteady(): void {
    touchDependencies(this._injector, [
      [TerminalPersistenceController],
    ]);
  }
}
```

**禁止写法**：
```typescript
// ❌ 用 undefined 占位绕过 config 规范
constructor(
  _config: undefined = undefined,
  @InjectSelf() injector: Injector
) { ... }

// ❌ 接口内联在 plugin.ts、无 default、无 setConfig
constructor(
  private readonly _config: Partial<IXxxConfig> = {},
  @Inject(Injector) protected readonly _injector: Injector
) { super(); }  // 漏掉 _configService + merge + setConfig

// ❌ 把 _injector / _configService 拆成类体字段 + 构造体内 this._x = x 显式赋值。
//    DI 注入字段必须用 parameter property 形式（见 02-di.md），mobile / desktop 一致。
export class XxxPlugin extends Plugin {
  protected readonly _injector: Injector;
  private readonly _configService: IConfigService;

  constructor(
    private readonly _config: Partial<IXxxConfig> = defaultPluginConfig,
    @InjectSelf() injector: Injector,
    @IConfigService configService: IConfigService
  ) {
    super();
    this._injector = injector;
    this._configService = configService;
    // ...
  }
}
```

## 7.3 Mobile 与 desktop 共用同一份 plugin 模板

`apps/mobile/plugins/*` 的 `Plugin` 子类与 desktop §7.2 模板**完全一致**——同样用 parameter property 形式注入 `_injector` / `_configService`，没有任何变体或例外。

唯一可选差异：注入 child injector 时 mobile 常用 `@InjectSelf()`，desktop 常用 `@Inject(Injector)`，两者等价；mobile 子 injector 语义更明确时用 `@InjectSelf()`。

> Mobile **service / repository / controller** 受 babel 链 bug 限制，**当前**临时拆字段写——属于工具链债，与本节无关，详见 [[mobile-babel-no-param-property-decorator]] memory 与 `02-di.md`。

## 7.4 关键工具与装饰器

| API | 来源 | 用途 |
|-----|------|------|
| `@DependentOn(P1, P2, ...)` | `@termlnk/core` | 声明插件级依赖；启动时若被依赖插件未注册，会自动报错 |
| `@Inject(X)` | `@termlnk/core` | 注入具体类（非接口标识符） |
| `@InjectSelf()` | `@termlnk/core` | 注入当前 child injector（plugin 场景常用） |
| `@Optional(X)` | `@termlnk/core` | 注入可选依赖，未注册时为 `undefined` |
| `merge(...)` | `@termlnk/core`（lodash-es） | **深合并**配置，代替手写 spread |
| `registerDependencies(injector, deps)` | `@termlnk/core` | 批量注册 |
| `touchDependencies(injector, deps)` | `@termlnk/core` | 批量触发实例化 |
| `mergeOverrideWithDependencies(deps, override)` | `@termlnk/core` | 配合 `_config.override` 让上层覆盖默认服务实现 |

### 配置合并 — 必须 `merge()`，不要 spread

```typescript
// ❌ 错误：浅 spread 会让嵌套配置整体覆盖默认值
const cfg = { ...defaultPluginConfig, ...this._config };

// ✅ 正确：lodash merge 深合并
const cfg = merge({}, defaultPluginConfig, this._config);
```

### override 配置 — 让消费方替换默认实现

```typescript
new TerminalUIPlugin({
  override: [
    [IHostExplorerService, { useClass: CustomHostExplorerService }],
  ],
});
```

## 7.5 生命周期钩子选择

| 钩子 | 时机 | 适合做什么 |
|-----|------|----------|
| `onStarting()` | 插件启动前 | 注册 DI 依赖、注册命令 |
| `onReady()` | 核心服务就绪 | 注册拦截器、订阅命令执行、跨插件协作、可选 `touchDependencies(IXxxService)` 强制初始化 |
| `onRendered()` | 首次渲染完成 | `touchDependencies` 激活 UI 相关控制器（让组件第一时间可见） |
| `onSteady()` | 完全稳定 | `touchDependencies` 激活重型 / 非关键控制器（持久化、统计、Agent 等） |

**典型分配**：UI 关键路径（菜单、视图注册、快捷键 controller）→ `onRendered`；后台任务、磁盘 IO、网络订阅 → `onSteady`。

## 7.6 插件配置键规范 — 一插件一 key (强制)

**核心约束**：每个插件**只能拥有一个顶级配置键**。运行时配置（`IConfigService` 内存）与持久化字段（`ConfigRepository` 数据库 `configEntity` 表）**共用该键**；所有持久化字段作为 subKey 声明在 `IXxxConfig` 接口上，由 `ConfigRepository.getField / setField` 按字段读写。

### 7.6.1 命名规则

| 项目 | 规则 | 示例 |
|-----|------|------|
| Key 值 | `<plugin-name>.config` | `'electron.config'` / `'electron-main.config'` / `'storage-mobile.config'` |
| 常量名 | UPPER_SNAKE_CASE + `_PLUGIN_CONFIG_KEY` | `ELECTRON_MAIN_PLUGIN_CONFIG_KEY` |
| 类型接口 | `IXxxConfig` 聚合所有字段 | `IElectronMainConfig` / `IStorageMobileConfig` |
| 文件位置 | `<plugin-pkg>/src/controllers/config.schema.ts` | — |

### 7.6.2 接口定义模板

```typescript
// packages/electron-main/src/controllers/config.schema.ts
import type { DependencyOverride } from '@termlnk/core';
import type { IMainWindowState } from '../services/window-state/type';

export const ELECTRON_MAIN_PLUGIN_CONFIG_KEY = 'electron-main.config';

export interface IElectronMainConfig {
  // ── runtime fields (injected at plugin construction, served by IConfigService) ──
  override?: DependencyOverride;
  url?: string;
  preload?: string;

  // ── persistent fields (read/written via ConfigRepository per subKey) ──
  mainWindowState?: IMainWindowState;
}

export const defaultPluginConfig: IElectronMainConfig = {};
```

运行时字段与持久化字段**同一个接口**声明，仅通过访问路径区分。

### 7.6.3 双重访问方式

```typescript
// ① 内存运行时配置 — 插件构造时设置
this._configService.setConfig(ELECTRON_MAIN_PLUGIN_CONFIG_KEY, runtimeConfig);

// Controller 中读取
const cfg = this._configService.getConfig<IElectronMainConfig>(ELECTRON_MAIN_PLUGIN_CONFIG_KEY);
const url = cfg?.url;

// ② 持久化字段 — 按 subKey 读写数据库
await this._configRepository.setField(
  ELECTRON_MAIN_PLUGIN_CONFIG_KEY,
  'mainWindowState',
  state,
);

const stored = await this._configRepository.getField<IMainWindowState>(
  ELECTRON_MAIN_PLUGIN_CONFIG_KEY,
  'mainWindowState',
);
```

### 7.6.4 监听外部变更

```typescript
this.disposeWithMe(
  this._configRepository.changed$.pipe(
    filter((e) => e.key === ELECTRON_MAIN_PLUGIN_CONFIG_KEY
      && (e.subKey === 'mainWindowState' || e.subKey === undefined)),
  ).subscribe(() => {
    void this._onConfigChanged();
  }),
);
```

`e.subKey === undefined` 覆盖整键 `set()`（导入/重置）。

### 7.6.5 禁止事项

| 禁止 | 原因 |
|-----|------|
| ❌ 为单个持久化字段单独开顶级 key（如 `electron-main.main-window-state`） | 破坏"一插件一键"原则 |
| ❌ 写入未在 `IXxxConfig` 中声明的字段 | 丢失类型安全 |
| ❌ 跨插件直接读写他人 config key | 越过服务边界，必须经 Service API |
| ❌ 持久化字段用 `set(KEY, ...)` 覆盖整键 | 会清掉其他 subKey，必须用 `setField` |
| ❌ `_config: undefined = undefined` 占位 | 违反 §7.1 强制规范 |
| ❌ `IXxxConfig` 内联在 plugin.ts | 必须独立到 `controllers/config.schema.ts` |

### 7.6.6 新增持久化字段流程

1. 在 `IXxxConfig` 接口中加可选字段（类型须独立定义）
2. 提供 `normalizeXxx(value): IXxx` 帮助函数处理 null / 非法值
3. 消费方通过 `ConfigRepository.getField / setField` 读写
4. 需跨进程/跨控制器响应时，订阅 `ConfigRepository.changed$` 并 `filter` 对应 subKey

### 7.6.7 已落地示例

| 插件 | Config Key | 运行时字段 | 持久化字段 |
|-----|-----------|-----------|-----------|
| `@termlnk/electron` | `electron.config` | `override` | `appSettings: IAppSettings` |
| `@termlnk/electron-main` | `electron-main.config` | `url`, `preload`, `override` | `mainWindowState: IMainWindowState` |
| `@termlnk/sync-mobile` | `sync-mobile.config` | `cloudBaseUrl`, `override` | — |
| `@termlnk/storage-mobile` | `storage-mobile.config` | `override` | — |

## 7.7 自查清单 (写完 plugin 必跑)

- [ ] 有 `controllers/config.schema.ts`，导出 `XXX_PLUGIN_CONFIG_KEY` + `IXxxConfig` + `defaultPluginConfig`
- [ ] 构造器第一参数是 `_config: Partial<IXxxConfig> = defaultPluginConfig`（不是 `undefined`，不是 `{}`）
- [ ] 构造器注入了 `IConfigService` 并在 super 后调用了 `merge` + `setConfig`
- [ ] `onStarting` 通过 `mergeOverrideWithDependencies(deps, this._config?.override)` 注册
- [ ] index.ts 导出了 `IXxxConfig` + `XXX_PLUGIN_CONFIG_KEY` + plugin 类 + plugin 名常量
- [ ] **DI 注入字段全部用 parameter property 形式**（`@Decorator() <可见性> readonly _x: T`）—— mobile / desktop / web / packages 一视同仁，**禁止**拆成类体字段 + 体内赋值。详见 `02-di.md`。
- [ ] 用 `grep -E "setConfig|PLUGIN_CONFIG_KEY|defaultPluginConfig|merge\(" plugin.ts` 四个匹配均 > 0
