# 13. 文件组织与目录结构

## 13.1 插件包形态总览

Termlnk 仓库中所有插件包遵循**完全一致**的目录契约（只是 workspace 位置不同）。四种形态：

| 形态 | 位置 | 包名约定 | 典型场景 |
|-----|------|---------|---------|
| **共享契约/实现包** | `packages/<name>/` | `@termlnk/<name>` | 跨 desktop/mobile/web 复用的服务、双进程契约 |
| **Desktop 私有插件包** | `apps/desktop/plugins/<name>/` | `@termlnk/<name>` | Electron 主进程 / 渲染端 app 级模块（island-core、electron-main 等） |
| **Web 私有插件包** | `apps/web/plugins/<name>/` | `@termlnk/<name>` | Web app 级模块（web-server、web-renderer 等） |
| **Mobile 私有插件包** | `apps/mobile/plugins/<name>-mobile/` | `@termlnk/<name>-mobile` | Expo app 级模块（auth-mobile、storage-mobile 等） |

**workspace 声明**（`pnpm-workspace.yaml`）已覆盖四类插件形态（下为与插件相关的 glob 节选）：
```yaml
packages:
  - packages/*               # 共享契约/实现包
  - apps/desktop/plugins/*   # desktop 私有插件
  - apps/web/plugins/*       # web 私有插件
  - apps/mobile/plugins/*    # mobile 私有插件
  - apps/mobile/packages/*   # mobile 原生模块 (react-native-russh 等)
```
（真实文件还含 `internal/*`、`extensions/*`、`apps/desktop/main`、`apps/web/server` 等 app / 工具包条目，非插件形态，此处从略。）

**命名硬规则**：
- Desktop app 私有插件包名与共享契约包**对称**：`@termlnk/island-core`（实现）对应 `@termlnk/island`（契约）
- Mobile app 私有插件包必须用 `-mobile` 后缀：`@termlnk/auth-mobile` 对应 `@termlnk/auth` 契约。**禁止** `mobile-auth`、`mobile-platform` 这类前缀形式或大杂烩命名
- 包目录名 = 包名去掉 `@termlnk/` 前缀：`@termlnk/storage-mobile` → `apps/mobile/plugins/storage-mobile/`

## 13.2 单个插件包的标准目录

**所有形态共享同一份标准结构**（含或不含某些子目录视实际需要）：

```
<plugin-package>/
├── package.json              # 必有 — scripts: typecheck/lint/lint:fix; exports: { ".": "./src/index.ts" }
├── tsconfig.json             # extends @termlnk/shared/tsconfig/base (mobile 包覆盖 jsx: 'react-native')
├── tsconfig.node.json        # 可选 — desktop 包 vitest 需要 (mobile 包不需要)
├── vitest.config.ts          # 可选 — desktop 包跑 vitest (mobile 包暂不跑)
├── nativewind-env.d.ts       # mobile 含 React 组件的包才需要 (/// <reference types="react-native-css/types" />)
└── src/
    ├── index.ts              # ❗facade — 公开导出，必有
    ├── plugin.ts             # ❗插件入口，必有，文件名固定为 plugin.ts (不是 xxx.plugin.ts)
    ├── controllers/
    │   ├── config.schema.ts  # ❗必有 — 即使空 config 也要定义 KEY + 接口 + default (见 07-plugin.md §7.1)
    │   ├── <name>.controller.ts
    │   └── menu/             # 菜单 schema (见 14-menu.md)
    │       └── <scope>.menu.ts
    ├── services/             # DI 服务实现
    │   └── <domain>/         # 大包按 domain 分子目录 (rpc-client 范例)
    │       └── <name>.service.ts
    ├── commands/             # 命令定义
    │   ├── commands/         # COMMAND 类型
    │   ├── mutations/        # MUTATION 类型
    │   └── operations/       # OPERATION 类型
    ├── models/               # 数据模型 (interface + 工厂)
    ├── views/                # React 组件 (UI 包)
    │   ├── components/
    │   └── parts/
    ├── locale/               # i18n (zh-CN.ts / en-US.ts / ja-JP.ts / ko-KR.ts / zh-TW.ts)
    ├── common/               # 共享工具、常量 (不依赖其他目录)
    └── contributions/        # 贡献点 (extension-ui 等)
```

### 必有 / 可选清单

| 文件 / 目录 | 必须 | 说明 |
|------------|------|------|
| `package.json` | ✅ | exports 指向 `./src/index.ts`；scripts 至少含 `typecheck` / `lint` / `lint:fix` |
| `tsconfig.json` | ✅ | extends `@termlnk/shared/tsconfig/base` |
| `src/index.ts` | ✅ | facade — 唯一对外入口 |
| `src/plugin.ts` | ✅ | 插件类（文件名固定，**不要写成** `xxx.plugin.ts`） |
| `src/controllers/config.schema.ts` | ✅ | 每个插件 1 个，含 `XXX_PLUGIN_CONFIG_KEY` + `IXxxConfig` + `defaultPluginConfig` |
| `src/services/` | 视需要 | 没有 service 的纯配置插件可省 |
| `src/controllers/` | 视需要 | 没有 controller 的服务型插件可省 |
| `src/views/` | 视需要 | UI 包才有 |
| `src/locale/` | 视需要 | 含 UI 字符串才有 |
| `tsconfig.node.json` / `vitest.config.ts` | desktop 包可选；mobile 包通常无 | mobile 包目前不跑 vitest |

### 反面教材

```
❌ apps/mobile/src/platform/mobile-platform.plugin.ts   # 不该把 plugin.ts 跟普通 service 平铺混放
❌ apps/mobile/src/sync/mobile-sync.plugin.ts           # 同上
❌ packages/foo/src/foo.plugin.ts                       # 文件名错 — 应为 plugin.ts
❌ apps/mobile/plugins/mobile-platform/                  # 命名错 — 应该按对应契约拆细 (auth-mobile / storage-mobile)
```

## 13.3 文件命名

| 类型 | 格式 | 示例 |
|------|------|------|
| 插件入口 | 固定 `plugin.ts` | `plugin.ts` ✅ ／ ❌ `xxx.plugin.ts` |
| 包入口 (facade) | 固定 `index.ts` | `index.ts` |
| 配置 schema | 固定 `config.schema.ts` | `controllers/config.schema.ts` |
| 服务文件 | `kebab-case.service.ts` | `session.service.ts` |
| 控制器文件 | `kebab-case.controller.ts` | `terminal-ui.controller.ts` |
| 命令文件 | `kebab-case.command.ts` | `create-session.command.ts` |
| Mutation 文件 | `kebab-case.mutation.ts` | `save-session.mutation.ts` |
| Operation 文件 | `kebab-case.operation.ts` | `toggle-dialog.operation.ts` |
| 模型文件 | `kebab-case.model.ts` | `session.model.ts` |
| 工厂文件 | `kebab-case.factory.ts` | `mobile-sftp-client.factory.ts` |
| 菜单 schema | `<scope>.menu.ts` | `host-tree.menu.ts` |
| React 组件 | `PascalCase.tsx` | `SessionList.tsx`, `HostDialog.tsx` |
| Hook 文件 | `use-<name>.ts` 或 `use-<name>.tsx` | `use-host-tree.ts` |
| 类型文件 | `kebab-case.ts` 或 `types.ts` | `types.ts`, `i-session.ts` |
| 自动生成产物 | `<name>.generated.ts`（必须列入 `.gitignore`） | `xterm-bundle.generated.ts` |

## 13.4 包名命名

| 类别 | 规则 | 示例 |
|-----|------|------|
| 共享契约包 | `@termlnk/<domain>` | `@termlnk/auth`, `@termlnk/sync` |
| 共享实现包 | `@termlnk/<domain>-core`（主进程实现）/ `@termlnk/<domain>-ui`（渲染端 UI） | `@termlnk/auth-core`, `@termlnk/sftp-ui` |
| Desktop app 私有插件 | `@termlnk/<name>` 同上，放 `apps/desktop/plugins/` | `@termlnk/island-core`, `@termlnk/electron-main` |
| Web app 私有插件 | `@termlnk/<name>`（无后缀，同 desktop），放 `apps/web/plugins/` | `@termlnk/web-server`, `@termlnk/web-renderer` |
| Mobile app 私有插件 | `@termlnk/<contract>-mobile` 后缀 | `@termlnk/auth-mobile`, `@termlnk/storage-mobile` |
| Mobile 原生模块 | `@termlnk/<native-name>` 不加 `-mobile` | `@termlnk/react-native-russh` |

**禁止形式**：
- ❌ `@termlnk/mobile-<name>` — 前缀错位
- ❌ `@termlnk/<name>Mobile`、`@termlnk/<name>_mobile` — 非 kebab-case
- ❌ `@termlnk/<role>` 模糊后缀 — `@termlnk/foo-manager`、`@termlnk/foo-router` 等（见 SKILL.md 命名规则）

## 13.5 导入硬约束 (lint 卡)

- **禁止桶导入** — 不从 `index.ts` 导入，直接导入具体文件
- **禁止穿透导入** — 不跨包导入内部文件 (`packages/foo/src/internal/...`)
- **禁止自包导入** — 包内不要用包名导入自己（用相对路径）
- **Facade 文件**（通常是 `index.ts`）必须显式声明返回类型；facade 外**禁止**导入 facade，facade 中**禁止**导入外部

## 13.6 ESLint 自定义规则速查

| 规则 | 用途 |
|-----|------|
| `no-barrel-import` | 禁止从主入口导入（防循环依赖） |
| `no-penetrating-import` | 禁止跨界导入（保持层级边界） |
| `no-external-imports-in-facade` | 禁止 facade 中的外部导入 |
| `no-facade-imports-outside-facade` | 禁止 facade 外导入 facade |
| `no-self-package-imports` | 禁止自引用包导入 |

## 13.7 Mobile 端独有约束

- **`jsx: "react-native"`** — mobile plugin 包的 tsconfig 必须覆盖（`@termlnk/shared/tsconfig/base` 默认是 `react-jsx`）
- **`types: ["node", "react", "react-native", "react-native-css/types"]`** — `react-native-css/types` 是 NativeWind 的 className 类型补丁，所有 mobile plugin tsconfig 都加上（即使包内无 React 组件，否则跨包 typecheck 看到含 className 的 tsx 会报错）
- **`allowImportingTsExtensions: true`** — RN 监听器需要
- **xterm-bundle 等自动生成产物**放在 `<package>/src/services/` 但加入 `.gitignore`，由 postinstall 脚本（如 `scripts/generate-xterm-bundle.cjs`）生成
- **babel 链 bug 只影响 mobile 的 service / repository / controller**（构造器装饰器 + TS 参数属性不能并存，须临时拆字段）；**mobile plugin 不受影响，照常用参数属性**。详见 [02-di.md](./02-di.md) 「Mobile babel 工具链债」与 [07-plugin.md §7.3](./07-plugin.md#73-mobile-与-desktop-共用同一份-plugin-模板)

## 13.8 自查清单 (新建插件包必跑)

- [ ] 包名符合 §13.4 命名（共享 `@termlnk/<name>` / mobile `@termlnk/<name>-mobile`）
- [ ] 包目录位置与 `pnpm-workspace.yaml` glob 匹配
- [ ] `package.json` 含 `exports: { ".": "./src/index.ts" }` + scripts (typecheck/lint/lint:fix)
- [ ] `tsconfig.json` extends `@termlnk/shared/tsconfig/base`，mobile 包加 jsx/types 覆盖
- [ ] `src/plugin.ts` 文件名严格是 `plugin.ts`，**不**是 `xxx.plugin.ts`
- [ ] `src/controllers/config.schema.ts` 存在且导出 KEY + 接口 + default
- [ ] `src/index.ts` 至少导出 plugin 类、plugin 名常量、KEY、IXxxConfig
- [ ] `pnpm install` 后能被 workspace 识别（`pnpm ls @termlnk/<name>` 有结果）
- [ ] `pnpm typecheck` 通过
