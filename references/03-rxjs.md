# 03. RxJS 响应式编程规范

## 3.1 Observable 命名 — `$` 后缀 (强制)

**所有 Observable 属性必须以 `$` 后缀结尾**,这是本项目最核心的命名规则:

```typescript
// ✅ 正确
private readonly _sessions$ = new BehaviorSubject<ISession[]>([]);
readonly sessions$ = this._sessions$.asObservable();

private readonly _activeSession$ = new BehaviorSubject<Nullable<ISession>>(null);
readonly activeSession$ = this._activeSession$.asObservable();

private readonly _sessionCreated$ = new Subject<ISession>();
readonly sessionCreated$ = this._sessionCreated$.asObservable();

// ❌ 错误:缺少 $ 后缀
private readonly _sessions = new BehaviorSubject<ISession[]>([]);
```

## 3.2 Subject 选择

| Subject 类型 | 场景 | 示例 |
|-------------|------|------|
| `BehaviorSubject<T>` | **有状态**数据流,需要当前值和初始值 | 主题、当前用户、会话列表 |
| `Subject<T>` | **无状态**事件流,只关注未来事件 | 用户操作、命令执行通知 |
| `ReplaySubject<T>` | 重放历史值给新订阅者 | 罕用,特殊场景 |

```typescript
// ✅ BehaviorSubject:有当前值的状态
private readonly _darkMode$ = new BehaviorSubject<boolean>(false);
readonly darkMode$ = this._darkMode$.asObservable();
get darkMode(): boolean { return this._darkMode$.getValue(); }

// ✅ Subject:纯事件流
private readonly _sessionCreated$ = new Subject<ISession>();
readonly sessionCreated$ = this._sessionCreated$.asObservable();
```

## 3.3 Subject 封装 — 私有 Subject + 公开 Observable

```typescript
export class ThemeService extends Disposable {
  // ✅ 正确
  private readonly _currentTheme$ = new BehaviorSubject<Theme>(defaultTheme);
  readonly currentTheme$: Observable<Theme> = this._currentTheme$.asObservable();

  setTheme(theme: Theme): void {
    this._currentTheme$.next(theme);
  }
}

// ❌ 错误:外部可以 currentTheme$.next(xxx),破坏封装
export class ThemeService extends Disposable {
  readonly currentTheme$ = new BehaviorSubject<Theme>(defaultTheme);
}
```

## 3.4 接口中声明 Observable

接口里类型必须是 `Observable<T>`,不是 `Subject<T>` / `BehaviorSubject<T>`:

```typescript
// ✅ 正确
export interface IThemeService {
  readonly currentTheme$: Observable<Theme>;
  readonly darkMode$: Observable<boolean>;
  setTheme(theme: Theme): void;
}

// ❌ 错误
export interface IThemeService {
  readonly currentTheme$: BehaviorSubject<Theme>;
}
```

## 3.5 动态 Observable 创建 (按 key 订阅)

```typescript
export class ConfigService extends Disposable implements IConfigService {
  private readonly _configChanged$ = new Subject<{ [key: string]: unknown }>();
  readonly configChanged$ = this._configChanged$.asObservable();
  private readonly _config = new Map<string, any>();

  subscribeConfigValue$<T = unknown>(key: string): Observable<T> {
    return new Observable<T>((observer) => {
      if (this._config.has(key)) {
        observer.next(this._config.get(key) as T);
      }
      const sub = this._configChanged$.pipe(
        filter((c) => key in c),
      ).subscribe((c) => observer.next(c[key] as T));
      return () => sub.unsubscribe();
    });
  }
}
```

## 3.6 常用操作符

### combineLatest — 任一源发射时重算

```typescript
this.disposeWithMe(
  combineLatest([
    this._menuService.menuChanged$.pipe(startWith(undefined)),
    this._instanceService.focused$.pipe(startWith(undefined)),
  ]).subscribe(() => {
    this._updateRibbon();
  }),
);
```

### switchMap — 切换数据源(自动取消上一个)

```typescript
this.selectionChanged$ = this._currentWorkbook$.pipe(
  switchMap((workbook) =>
    !workbook ? of([]) : this._getWorkbookSelection(workbook.id).selectionChanged$,
  ),
  distinctUntilChanged((prev, curr) => isEqual(prev, curr)),
  skip(1),
).pipe(takeUntil(this.dispose$));
```

### merge — 合并多个同类型事件流

```typescript
this.selectionChanged$ = merge(this._selectionMoveEnd$, this._selectionSet$);
```

### 防抖 / 节流

```typescript
this._searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
).subscribe((keyword) => this._performSearch(keyword));

this._cellUpdates$.pipe(
  bufferDebounceTime(100),
).subscribe((updates) => this._batchUpdate(updates));
```

## 3.7 自定义工具

```typescript
// 回调 → Observable,自动管理 Disposable
export function fromCallback<T extends readonly unknown[]>(
  callback: (handler: (...args: T) => void) => IDisposable | undefined,
): Observable<T> {
  return new Observable((subscriber) => {
    const disposable = callback((...args: T) => subscriber.next(args));
    return () => disposable?.dispose();
  });
}

// 满足条件后完成(包含满足条件的那个值)
export function takeAfter<T>(predicate: (value: T) => boolean) {
  return (source: Observable<T>) =>
    new Observable<T>((subscriber) => {
      source.subscribe({
        next: (v) => {
          subscriber.next(v);
          if (predicate(v)) {
            subscriber.complete();
          }
        },
        complete: () => subscriber.complete(),
        error: (err) => subscriber.error(err),
      });
    });
}

// 缓冲到 debounce 触发后批量发射
export function bufferDebounceTime<T>(time: number) {
  return (source: Observable<T>) =>
    source.pipe(buffer(source.pipe(debounceTime(time))));
}
```

## 3.8 高级模式

### shareReplay — 多播并缓存最新值

多处订阅同一个数据源时,用 `shareReplay(1)` 避免重复计算:

```typescript
const currentWorkbook$ = this._instanceService
  .getCurrentTypeOfUnit$(UnitType.TERMINAL)
  .pipe(shareReplay(1), takeUntil(this.dispose$));

this.selectionMoveStart$ = currentWorkbook$.pipe(
  switchMap((wb) => !wb ? of(null) : this._getSelection(wb.id).moveStart$),
);
this.selectionMoveEnd$ = currentWorkbook$.pipe(
  switchMap((wb) => !wb ? of(null) : this._getSelection(wb.id).moveEnd$),
);
```

`share()` vs `shareReplay(1)`:
- `shareReplay(1)`:新订阅者**立刻收到**最后一个值(适合状态流)
- `share()`:新订阅者**不会收到**历史值(适合事件流,如 WebSocket 消息)

### 嵌套 switchMap + merge — 动态数据源聚合

```typescript
protected _init(): void {
  const allWorkbooks$ = this._getAliveWorkbooks$().pipe(takeUntil(this.dispose$));

  this.selectionMoveStart$ = allWorkbooks$.pipe(
    switchMap((workbooks) => merge(...workbooks.map((wb) => wb.selectionMoveStart$))),
  );
}

private _getAliveWorkbooks$(): Observable<WorkbookSelectionModel[]> {
  const workbooks$ = new BehaviorSubject(this._getInitialWorkbooks());

  this.disposeWithMe(
    this._instanceService.getTypeOfUnitAdded$(UnitType.TERMINAL)
      .subscribe((wb) => workbooks$.next([...workbooks$.getValue(), wb])),
  );
  this.disposeWithMe(
    this._instanceService.getTypeOfUnitDisposed$(UnitType.TERMINAL)
      .subscribe((wb) => workbooks$.next(workbooks$.getValue().filter((w) => w !== wb))),
  );

  return workbooks$.pipe(
    map((wbs) => wbs.map((wb) => this._ensureSelection(wb.getUnitId()))),
  );
}
```

### distinctUntilChanged 深度比较

对复杂对象数组**必须**提供自定义比较函数:

```typescript
// ✅ 正确
this.selectionChanged$ = source$.pipe(
  distinctUntilChanged((prev, curr) => {
    if (prev.length !== curr.length) return false;
    if (prev.length === 0 && curr.length === 0) return true;
    return prev.every((item, index) =>
      JSON.stringify(item) === JSON.stringify(curr[index]),
    );
  }),
  skip(1),
).pipe(takeUntil(this.dispose$));

// ❌ 错误:默认 === 比较对引用类型无效
this.selectionChanged$ = source$.pipe(distinctUntilChanged());
```

### firstValueFrom — Observable → Promise

```typescript
onStage(stage: LifecycleStages): Promise<void> {
  return firstValueFrom(this.lifecycle$.pipe(
    filter((s) => s >= stage),
    takeAfter((s) => s === stage),
    map(() => void 0),
  )).catch((err) => {
    if (err.name === 'EmptyError') {
      return Promise.reject(new LifecycleUnreachableError(stage));
    }
    return Promise.reject(err);
  });
}

private _whenReady(): Promise<boolean> {
  return firstValueFrom(
    this._initialized$.pipe(filter((v) => v), take(1)),
  );
}
```

### throttleTime — leading/trailing 控制

```typescript
// 搜索:立即响应 + 500ms 后再发最新值
this._searchString$.pipe(
  throttleTime(500, undefined, { leading: true, trailing: true }),
  startWith(void 0),
);

// 渲染限制 60fps
this._engine.onTransformChange$.pipe(
  throttleTime(16),
).subscribe(() => this._recalculate());
```

## 3.9 EventSubject — 带优先级的事件系统

需要优先级控制和事件拦截时使用:

```typescript
export class EventSubject<T> extends Subject<[T, EventState]> {
  private _sortedObservers: IEventObserver<T>[] = [];

  /** 数字越小优先级越高,先执行 */
  subscribeEvent(observer: IEventObserver<T>): Subscription { /* ... */ }

  emitEvent(event: T): INotifyObserversReturn {
    const state = new EventState();
    for (const observer of this._sortedObservers) {
      observer.next?.([event, state]);
      if (state.skipNextObservers) {
        return { handled: true, lastReturnValue: state.lastReturnValue, stopPropagation: true };
      }
    }
    return { handled: this._sortedObservers.length > 0, lastReturnValue: state.lastReturnValue };
  }
}

export function fromEventSubject<T>(subject$: EventSubject<T>): Observable<T> {
  return new Observable((subscriber) => {
    const sub = subject$.subscribeEvent((evt) => subscriber.next(evt));
    return () => sub.unsubscribe();
  });
}
```

## 3.10 操作符选择速查

| 场景 | 操作符 | 备注 |
|------|--------|------|
| 切换数据源(取消前一个) | `switchMap` | 用户切 tab/workbook |
| 顺序执行(排队) | `concatMap` | HTTP 请求保序 |
| 忽略新值直到前一个完成 | `exhaustMap` | 防重复提交 |
| 合并多个状态源 | `combineLatest` | 所有源都发射后开始 |
| 合并多个事件流 | `merge` | 任一源发射即发射 |
| 提供初始值 | `startWith` | 配 combineLatest 防阻塞 |
| 防抖 | `debounceTime` | 用户输入、搜索 |
| 节流 | `throttleTime` | 渲染、滚动 |
| 去重 | `distinctUntilChanged` | 避免重渲染 |
| 跳过初始值 | `skip(1)` | BehaviorSubject 不要初值时 |
| 缓存并共享 | `shareReplay(1)` | 多处订阅同一源 |
| Observable → Promise | `firstValueFrom` | 等待单次值 |
| 回调 → Observable | `fromCallback` | 适配旧 API |
