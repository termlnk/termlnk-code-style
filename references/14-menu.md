# 菜单系统 (下拉菜单 / 右键菜单)

> **所有动作菜单——右键上下文菜单与锚定式按钮下拉——必须走这一套菜单系统 + 命令系统**。

## 硬规则 (违反必改)

- **禁止**在业务组件里手写 `DropdownMenu` / `DropdownMenuItem` + 内联 `onSelect` 来做动作菜单。
- 唯一的菜单渲染层是 `DesktopContextMenu` (`@termlnk/ui`)；业务代码只声明「菜单 schema + 命令」并用 `IContextMenuService.triggerContextMenu` 触发。
- 菜单项执行的是 **`commandId`**（不是 `id`）——`DesktopContextMenu` 调 `executeCommand(item.commandId, item.params)`。`commandId` 缺失 = 点击无效。
- 命令**从服务读目标，不靠菜单参数**（`focused/target` 服务范式）；只有「同一命令的多个数据选项」才用 `params` / SELECTOR 的 `selections$` 携带数据。

## 菜单项类型 (`@termlnk/ui` → `services/menu/menu.ts`)

```typescript
enum MenuItemType { BUTTON, SELECTOR }

// 固定动作项
interface IMenuButtonItem {
  id: string;            // schema 标识
  commandId: string;     // 点击执行的命令 (必填)
  params?: object;        // 传给命令的参数 (可选)
  type: MenuItemType.BUTTON;
  title?: string;        // locale key, 经 localeService.t() 渲染
  tooltip?: string;
  icon?: string | Observable<string>;
  hidden$?: Observable<boolean>;
  disabled$?: Observable<boolean>;
  activated$?: Observable<boolean>;
}

// 变长 / 数据驱动选项 (如运行时计算的建议列表)
interface IMenuSelectorItem {
  id: string;
  type: MenuItemType.SELECTOR;
  selections?: IValueOption[] | Observable<IValueOption[]>; // 静态或流
  selectionsCommandId?: string;   // 选中某项时执行的命令
  value$?: Observable<string | number>; // 当前值, 渲染 ✓
}

interface IValueOption {
  value?: string | number;
  label: string;        // 展示文本 (动态项用运行时字符串; 静态项工厂里 t() 后填入)
  params?: object;        // 传给命令的参数 (携带该选项的数据)
  commandId?: string;    // 覆盖 selectionsCommandId
  disabled?: boolean;
}
```

`BUTTON` 用于固定菜单项；`SELECTOR` 用于**变长**或**数据驱动**的选项列表（选项个数运行时才知道）。

## 编写步骤

### 1. 命令 (`commands/.../*.command.ts`)

命令在 DI 层执行，从服务读目标 / 从 params 取数据：

```typescript
export const DeleteHostCommand: ICommand = {
  id: 'terminal-ui.command.delete-host',
  handler: (accessor: IAccessor) => {
    const focused = accessor.get(IHostExplorerService).getFocusedHost(); // 读目标
    if (!focused) return false;
    accessor.get(IHostManagerService).delete(focused.id);
    return true;
  },
};
```

### 2. 菜单工厂 + position (`controllers/.../*.menu.ts`)

```typescript
export const HOST_ITEM_MENU = 'terminal-ui.context-menu.hosts-explorer.item'; // <包名>.context-menu.<scope>

export function DeleteHostMenuItemFactory(accessor: IAccessor): IMenuButtonItem {
  return {
    id: DeleteHostCommand.id,
    commandId: DeleteHostCommand.id,   // 必填
    type: MenuItemType.BUTTON,
    title: 'terminal-ui.hosts-explorer.context-menu.delete',
    disabled$: accessor.get(IHostExplorerService).focusedHost$.pipe(map((h) => !h)),
  };
}
```

### 3. 菜单 schema (`controllers/.../menu.schema.ts`)

```typescript
export const menuSchema: MenuSchemaType = {
  [HOST_ITEM_MENU]: {
    [DeleteHostCommand.id]: { order: 1, menuItemFactory: DeleteHostMenuItemFactory },
  },
};
```

### 4. Controller 注册

```typescript
private _initCommands(): void {
  this.disposeWithMe(this._commandService.registerCommand(DeleteHostCommand));
}
private _initMenus(): void {
  this._menuManagerService.mergeMenu(menuSchema);
}
```

### 5. 视图触发 (右键 / 左键按钮统一)

```tsx
// 右键上下文菜单
onContextMenu={(e) => contextMenuService.triggerContextMenu(e.nativeEvent, HOST_ITEM_MENU)}

// 左键锚定下拉 (工具栏 "+" 按钮等)
onClick={(e) => contextMenuService.triggerContextMenu(e.nativeEvent, KEYCHAIN_ADD_MENU)}
```

两者机制相同：菜单按鼠标坐标弹出。触发前若菜单依赖目标，先 `setTarget(...)`（见下）。

## 命令读目标范式 (focused/target service)

菜单项不接收「点了哪一行」作参数。视图在右键/聚焦时把目标写进服务，命令从服务读：

```typescript
// 服务持目标
class FileContextService extends Disposable {
  private readonly _target$ = new BehaviorSubject<IFileContextTarget | null>(null);
  readonly target$ = this._target$.asObservable();
  get target() { return this._target$.getValue(); }
  setTarget(t: IFileContextTarget) { this._target$.next(t); }
  clear() { this._target$.next(null); }
}

// 视图右键时设目标再触发
const handleContextMenu = (e, entry) => {
  fileContextService.setTarget({ entry, actions: { /* 绑定该 entry 的动作 */ } });
  contextMenuService.triggerContextMenu(e.nativeEvent, SFTP_FILE_MENU);
};

// 命令读目标 (单命令 + params.action 复用)
export const FileActionCommand: ICommand<{ action: FileAction }> = {
  id: 'sftp-ui.command.file-action',
  handler: (accessor, params) => {
    const svc = accessor.get(FileContextService);
    svc.target?.actions[params.action]?.();
    svc.clear();
    return true;
  },
};
```

## SELECTOR 变长 / 数据驱动菜单

当选项个数运行时才确定（如逐请求生成的建议列表），用 `SELECTOR` + `selections$`，每个选项用 `params` 携带数据，命令从 params 取：

```typescript
// 服务把运行时数据映射成选项流
readonly selections$: Observable<IValueOption[]> = this._target$.pipe(
  map((t) => (t?.suggestions ?? []).map((s, i) => ({ value: i, label: s.label, params: { suggestion: s } }))),
);

// 工厂返回 SELECTOR
export function allowAlwaysMenuFactory(accessor: IAccessor): IMenuSelectorItem {
  return {
    id: 'agent-ui.approval.allow-always',
    type: MenuItemType.SELECTOR,
    selections: accessor.get(ApprovalMenuService).selections$,
    selectionsCommandId: AllowAlwaysCommand.id,
  };
}

// 命令从 params 取选项数据 + 从服务取上下文
export const AllowAlwaysCommand: ICommand<{ suggestion: ISuggestedRule }> = {
  id: 'agent-ui.command.approval.allow-always',
  handler: (accessor, params) => {
    const target = accessor.get(ApprovalMenuService).target;
    if (!target) return false;
    accessor.get(IAgentToolPermissionService).respond(buildRule(target.request, params.suggestion));
    return true;
  },
};
```

## Checklist

- [ ] 没有手写 `DropdownMenu` 做动作菜单（唯一渲染层是 `DesktopContextMenu`）
- [ ] 每个菜单项设了 `commandId`（不是只有 `id`）
- [ ] `title` 是 locale key（5 语言齐全）；position 命名 `<包名>.context-menu.<scope>`
- [ ] 命令从服务读目标，不靠菜单参数；变长/数据项才用 `params` / `selections$`
- [ ] Controller 里 `registerCommand` + `mergeMenu`；服务在 plugin 注册
- [ ] 触发统一用 `IContextMenuService.triggerContextMenu(e.nativeEvent, MENU)`

## 落地参考

| 场景 | 文件 |
|-----|------|
| 右键菜单 (读 focused 目标) | `terminal-ui` HostExplorer / TreeItem + `commands/delete-host.command.ts` |
| 左键锚定 BUTTON 下拉 | `terminal-ui` KeychainExplorer「+」+ `controllers/keychain/add-menu.ts` |
| 右键菜单 + 单命令 params | `sftp-ui` RemoteFilePane + `commands/file/file-action.command.ts` |
| SELECTOR 变长数据下拉 | `agent-ui` PendingApprovalBar + `services/approval/approval-menu.service.ts` |
| 渲染层 | `@termlnk/ui` `views/components/context-menu/DesktopContextMenu.tsx` |
