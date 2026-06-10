# 08. 命令系统规范

## 8.1 三种命令类型

| 类型 | 用途 | 持久化 | Undo/Redo | 异步 |
|-----|------|--------|-----------|-----|
| **COMMAND** | 业务逻辑编排 | 否 | 编排 Mutation 的 undo | 支持 |
| **MUTATION** | 持久化数据变更 | 是 | 是 | 仅同步 |
| **OPERATION** | UI 状态变更 | 否 | 否 | 支持 |

## 8.2 Operation 示例

```typescript
export const ToggleHostDialogOperation: IOperation = {
  id: 'terminal-ui.operation.toggle-host-dialog',
  type: CommandType.OPERATION,
  handler: (accessor) => {
    const dialogService = accessor.get(IDialogService);
    dialogService.toggle(HOST_DIALOG_ID);
    return true;
  },
};
```

## 8.3 Command 示例

```typescript
export interface ICreateSessionCommandParams {
  hostId: string;
  shellType?: string;
}

export const CreateSessionCommand: ICommand<ICreateSessionCommandParams> = {
  id: 'terminal-ui.command.create-session',
  type: CommandType.COMMAND,
  handler: async (accessor, params) => {
    if (!params) {
      return false;
    }

    const sessionService = accessor.get(ITerminalSessionService);
    const commandService = accessor.get(ICommandService);

    const session = sessionService.createSession({
      hostId: params.hostId,
      shellType: params.shellType,
    });

    if (!session) {
      return false;
    }

    // persist via mutation
    const result = commandService.syncExecuteCommand(
      SaveSessionMutation.id,
      { sessionId: session.id, config: session.config },
    );

    return result;
  },
};
```

## 8.4 命令 ID 命名

格式:`<包名>.<类型>.<kebab-case-动作>`

```typescript
// ✅ 正确
'terminal-ui.command.create-session'
'terminal-ui.operation.toggle-host-dialog'
'terminal.mutation.save-session'

// ❌ 错误
'createSession'                       // 缺前缀和类型
'terminal-ui.command.CreateSession'   // 不是 kebab-case
```

## 8.5 选择命令类型的决策

- 要写到磁盘/数据库 → **Mutation** (同步、可 undo)
- 改 UI 状态 (开关 dialog、切 tab) → **Operation**
- 调用其他命令、做业务编排 → **Command** (可异步)

## 8.6 命令组合

Command 通常通过 `syncExecuteCommand` / `executeCommand` 调用一个或多个 Mutation,Mutation 才真正改数据。Controller 通过 `onCommandExecuted` 监听 Mutation 执行后更新 UI (见 [06-controller.md](./06-controller.md))。
