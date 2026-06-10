# 01. 核心架构原则

## 分层架构

项目采用严格的分层架构,每层只能依赖其下层:

```
common/        → 不依赖其他层(纯工具/常量/类型)
models/        → 仅依赖 common
services/      → 依赖 common, models
commands/      → 依赖 common, models, services
controllers/   → 依赖以上所有层
views/         → 依赖以上所有层(React 组件)
```

## 设计原则

| 原则 | 说明 |
|------|------|
| **DI 优先** | 所有跨模块通信必须通过依赖注入,禁止直接实例化或全局变量 |
| **响应式数据流** | 状态变化通过 RxJS Observable 传播,禁止手动回调链 |
| **命令驱动** | 所有用户交互和状态变更通过命令系统执行 |
| **接口隔离** | 对外暴露接口(`I` 前缀),隐藏实现细节 |
| **自动资源回收** | 所有资源必须通过 Disposable 机制管理,确保无泄漏 |
| **对称双进程契约** | 主进程 / 渲染进程共享同一个接口类型,分别实现,禁止为渲染端单开 `*Client*` 接口 |

## 对称双进程契约 (必须遵守)

Termlnk 是 Electron 双进程架构:

- **主进程** (`rpc-server` / `shared-terminal-core` / `electron-main` 等) 承载业务逻辑
- **渲染进程** (`rpc-client` / `*-ui` 等) 承载 UI 与 facade
- 二者通过 tRPC over IPC 通信

**契约 (接口 + DI 标识符) 放在与运行时无关的契约包**: `@termlnk/shared-terminal`, `@termlnk/terminal`, `@termlnk/auth` 等。

```typescript
// 契约层: packages/shared-terminal/src/services/shared-terminal.service.ts
export interface ISharedTerminalService {
  readonly sessions$: Observable<readonly ISharedSession[]>;
  listSessions(): Promise<readonly ISharedSession[]>;
  setDriver(sessionId: string, clientId: string | null): Promise<void>;
}
export const ISharedTerminalService = createIdentifier<ISharedTerminalService>(
  'shared-terminal.service'
);

// 主进程实现: packages/shared-terminal-core/src/services/shared-terminal.service.ts
export class SharedTerminalService extends Disposable implements ISharedTerminalService { /* 调本地服务 */ }

// 渲染端实现: packages/rpc-client/src/services/shared-terminal/shared-terminal.service.ts
export class SharedTerminalService extends Disposable implements ISharedTerminalService { /* 转 tRPC */ }
```

两端各在自家插件里注册 `[ISharedTerminalService, { useClass: SharedTerminalService }]`。**同名接口、同一标识符、两份实现**。React 组件只看到 `ISharedTerminalService`,不知道也不关心运行在哪个进程。

### 签名规范

- 所有跨进程方法签名一律返回 `Promise<T>` 或 `Observable<T>`,即使主进程内部同步
- 同步 getter 仅给 BehaviorSubject 当前值用 (`get sessions(): readonly ISharedSession[]`),且只在主进程独立 API 提供,**不进契约接口**

### 反模式 (禁止)

- ❌ 接口后缀 `*ClientService` (如 `IExtensionClientService`, `ISharedTerminalClientService`) —— 泄漏运行时位置
- ❌ 只在 `rpc-client` 定义接口而没有对应主进程实现
- ❌ 主进程与渲染端各自定义"功能相同但 shape 略不同"的两个接口 —— 必然漂移
- ❌ 渲染端直接注入主进程服务标识符 (`@IPtyMultiplexerService`) —— 这是主进程独享服务,渲染端应通过契约接口访问

### 合理例外

- **纯协议门面**: `IRPCClientService` 只暴露 `getClient(): TRPCClient`,保留 `Client` 后缀表明这是 tRPC 协议门面,不是业务接口
- **真正只在渲染端存在的状态**: React 路由、UI 布局、Workbench 状态 —— 命名 `IXxxService` (不带 `Client`),放 UI 包

## 实战决策树

- 跨模块通信 → DI 服务 + Observable,**禁止** EventEmitter / 全局事件总线
- 状态变更 → 走命令系统 (`commandService.executeCommand`),**禁止**在 controller 里直接改 service 内部状态
- 数据共享 → 服务暴露 `xxx$` Observable,消费方订阅,**禁止**轮询或手动通知
- 资源释放 → 全部走 Disposable 体系,**禁止**裸 `subscription.unsubscribe()` 散落各处
- 业务逻辑 → 放 service,**不**放 controller (controller 只协调)
- 跨进程接口 → 契约层定义,主进程 + 渲染端各自实现同名服务,**禁止** `*Client*` 接口后缀
