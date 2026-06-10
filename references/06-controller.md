# 06. Controller 编写规范

## 6.1 Controller 职责边界

Controller 是**协调者**,只做四件事:

- 注册命令、快捷键、菜单
- 监听服务事件并协调响应
- 连接 UI 组件与服务层
- **不包含**核心业务逻辑(那是 service 的职责)

## 6.2 完整 Controller 模板

**默认基类 `Disposable`**(不是 `RxDisposable`)。订阅 Observable 用 `disposeWithMe(observable$.subscribe(...))`,无需 `takeUntil(dispose$)` 噪音。只有 pipe 内多条流需要同时短路时才用 `RxDisposable`。

构造函数中按业务顺序调用 `_initXxx()` 私有方法:**Commands → Shortcuts → Components → Menus → Views → Listeners**。

```typescript
export class TerminalUIController extends Disposable {
  constructor(
    @Inject(Injector) private readonly _injector: Injector,
    @Inject(ComponentManagerService) private readonly _componentManagerService: ComponentManagerService,
    @ICommandService private readonly _commandService: ICommandService,
    @IMenuManagerService private readonly _menuManagerService: IMenuManagerService,
    @IShortcutService private readonly _shortcutService: IShortcutService,
    @IUIPartsService private readonly _uiPartsService: IUIPartsService,
    @ITerminalViewRegistry private readonly _viewRegistry: ITerminalViewRegistry,
    @ITerminalService private readonly _terminalService: ITerminalService,
  ) {
    super();

    this._initCommands();
    this._initShortcuts();
    this._initComponents();
    this._initMenus();
    this._initViews();
    this._initListeners();
  }

  // ===== 注册命令 =====
  private _initCommands(): void {
    [
      CreateSessionCommand,
      CloseSessionCommand,
      ToggleHostDialogOperation,
    ].forEach((command) => {
      this.disposeWithMe(this._commandService.registerCommand(command));
    });
  }

  // ===== 注册快捷键 =====
  private _initShortcuts(): void {
    this.disposeWithMe(this._shortcutService.registerShortcut({
      id: CreateSessionCommand.id,
      description: 'terminal-ui.shortcuts.new-session',
      binding: KeyCode.N | MetaKeys.CTRL_COMMAND,
    }));
  }

  // ===== 注册 UI 组件 =====
  private _initComponents(): void {
    this.disposeWithMe(this._componentManagerService.register(IconKey, IconComponent));
    this.disposeWithMe(
      this._uiPartsService.registerComponent(BuiltInUIPart.CONTENT,
        () => connectInjector(TerminalContainer, this._injector)),
    );
  }

  // ===== 注册菜单 schema (集中声明,见 6.6) =====
  private _initMenus(): void {
    this._menuManagerService.mergeMenu(menuSchema);
  }

  // ===== 注册视图类型 =====
  private _initViews(): void {
    this.disposeWithMe(this._viewRegistry.registerView('ssh', TerminalView));
    this.disposeWithMe(this._viewRegistry.registerView('local', LocalTerminalView));
  }

  // ===== 订阅服务事件 — disposeWithMe(subscribe(...)) =====
  private _initListeners(): void {
    this.disposeWithMe(
      this._terminalService.sessionCreated$.subscribe((session) => {
        this._handleSessionCreated(session);
      }),
    );

    this.disposeWithMe(
      this._terminalService.activeSession$.subscribe((session) => {
        this._handleActiveSessionChanged(session);
      }),
    );
  }

  private _handleSessionCreated(session: ISession): void {
    // ...
  }

  private _handleActiveSessionChanged(session: Nullable<ISession>): void {
    // ...
  }
}
```

### 何时改用 `RxDisposable`

仅当 pipe 内需要 `takeUntil(this.dispose$)` 同时短路多条流时:

```typescript
export class CFRenderController extends RxDisposable {
  private _initSkeleton(): void {
    this.disposeWithMe(
      merge(this._rule$, this._viewModel.markDirty$).pipe(
        bufferTime(16),
        filter((v) => v.length > 0),
        takeUntil(this.dispose$),
      ).subscribe(() => this._markDirty()),
    );
  }
}
```

## 6.3 命令执行监听 — 核心控制器模式

```typescript
export class StatusBarController extends Disposable {
  constructor(
    @ICommandService private readonly _commandService: ICommandService,
    @Inject(SelectionService) private readonly _selectionService: SelectionService,
    @IStatusBarService private readonly _statusBarService: IStatusBarService,
  ) {
    super();
    this._initListeners();
  }

  private _initListeners(): void {
    // 监听特定命令执行后更新 UI
    this.disposeWithMe(
      this._commandService.onCommandExecuted((commandInfo: ICommandInfo) => {
        if (commandInfo.id === SetRangeValuesMutation.id) {
          this._recalculateStatistics();
        }
      }),
    );

    // 监听选区变化
    this.disposeWithMe(
      this._selectionService.selectionMoveEnd$.subscribe((selections) => {
        if (selections) {
          this._recalculateStatistics();
        }
      }),
    );
  }
}
```

| 场景 | 方式 |
|------|------|
| 响应数据变更 | `onCommandExecuted` 监听 Mutation 执行后触发副作用 |
| 拦截 / 验证 | `beforeCommandExecuted` 命令执行前校验 |
| 响应状态流变化 | 订阅 service 暴露的 Observable |

## 6.4 派生状态模式 (deriveStateFromActiveUnit$)

从活动上下文派生 Observable 状态,UI 状态管理关键模式:

```typescript
function deriveStateFromActiveUnit$<T>(
  instanceService: IInstanceService,
  defaultValue: T,
  callback: (active: { unit: UnitModel }) => Observable<T>,
): Observable<T> {
  return instanceService.getCurrentTypeOfUnit$(UnitType.TERMINAL).pipe(
    switchMap((unit) =>
      unit
        ? callback({ unit })
        : of(defaultValue),
    ),
  );
}

// 用法:菜单项 activated$
function BoldMenuItemFactory(accessor: IAccessor): IMenuButtonItem {
  const commandService = accessor.get(ICommandService);
  const instanceService = accessor.get(IInstanceService);

  return {
    id: SetBoldCommand.id,
    commandId: SetBoldCommand.id, // 渲染层执行 commandId, 必填
    type: MenuItemType.BUTTON,
    icon: 'BoldIcon',
    activated$: deriveStateFromActiveUnit$(
      instanceService,
      false,
      ({ unit }) => new Observable<boolean>((subscriber) => {
        const update = () => {
          subscriber.next(/* compute from unit */);
        };

        const disposable = commandService.onCommandExecuted((c) => {
          if (RELEVANT_COMMANDS.includes(c.id)) {
            update();
          }
        });

        update();
        return () => disposable.dispose();
      }),
    ),
  };
}
```

## 6.5 菜单项 Observable 状态

菜单项通过 Observable 暴露动态状态,UI 层自动订阅:

```typescript
export interface IMenuButtonItem {
  id: string;
  type: MenuItemType;
  disabled$?: Observable<boolean>;
  hidden$?: Observable<boolean>;
  activated$?: Observable<boolean>;
  icon?: string | Observable<string>;
}

export interface IMenuSelectorItem<V = string> {
  id: string;
  type: MenuItemType;
  disabled$?: Observable<boolean>;
  hidden$?: Observable<boolean>;
  value$?: Observable<V>;
  selections?: Array<IValueOption> | Observable<Array<IValueOption>>;
}

// 隐藏状态工具
export function getMenuHiddenObservable(
  accessor: IAccessor,
  targetType: UnitType,
): Observable<boolean> {
  const instanceService = accessor.get(IInstanceService);

  return new Observable((subscriber) => {
    const sub = instanceService.focused$.subscribe((unitId) => {
      if (!unitId) {
        return subscriber.next(true);
      }
      const type = instanceService.getUnitType(unitId);
      subscriber.next(type !== targetType);
    });

    // emit initial value synchronously
    const focused = instanceService.getFocusedUnit();
    subscriber.next(!focused || instanceService.getUnitType(focused.getUnitId()) !== targetType);

    return () => sub.unsubscribe();
  });
}
```

## 6.6 菜单 Schema 模式 — `mergeMenu(menuSchema)`

菜单**不**散落在 controller 里逐条 `addMenuItem`,而是集中在 `controllers/menu.schema.ts` 声明,由 controller 通过 `_menuManagerService.mergeMenu(menuSchema)` 一次注册:

```typescript
// controllers/menu.schema.ts
import type { MenuSchemaType } from '@termlnk/ui';
import { ContextMenuPosition, RibbonStartGroup } from '@termlnk/ui';

export const menuSchema: MenuSchemaType = {
  [RibbonStartGroup.OTHERS]: {
    [ToggleHostDialogOperation.id]: {
      order: 0,
      menuItemFactory: ToggleHostDialogMenuItemFactory,
    },
  },
  [ContextMenuPosition.MAIN_AREA]: {
    [CloseSessionCommand.id]: {
      order: 10,
      menuItemFactory: CloseSessionMenuItemFactory,
    },
  },
};
```

```typescript
// controllers/terminal-ui.controller.ts
private _initMenus(): void {
  this._menuManagerService.mergeMenu(menuSchema);
}
```

集中声明优势:菜单结构一眼可见、order 冲突易发现、不需 disposeWithMe 包装(schema 是一次性注入)。
