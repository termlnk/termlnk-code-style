# 04. 生命周期与资源释放规范

## 4.1 Disposable 体系

`@termlnk/core` 提供四层 Disposable 基类:

| 基类 | 用途 | 特性 |
|-----|------|------|
| `Disposable` | **首选**通用基类 | `disposeWithMe()` 管理子资源 |
| `RxDisposable` | pipe 内多流 `takeUntil(dispose$)` 短路时 | 继承 Disposable,多一个 `protected dispose$ = new Subject<void>()` |
| `RCDisposable` | 引用计数共享资源 | `inc()/dec()` 管理引用,归零自动释放根资源 |
| `DisposableCollection` | 批量管理多个 disposable | 集合式管理 |

**默认选 `Disposable`**。绝大多数 Service / Controller 用 `Disposable` + `disposeWithMe(observable$.subscribe(...))` 就够了。`RxDisposable` 只在 pipe 中需要 `takeUntil(this.dispose$)` 同时短路多条流时才有价值。

## 4.2 Disposable 基类

```typescript
export class Disposable implements IDisposable {
  protected _disposed = false;
  private readonly _collection = new DisposableCollection();

  /** 注册子资源,随本对象一起销毁 */
  disposeWithMe(disposable: DisposableLike): IDisposable {
    return this._collection.add(disposable);
  }

  /** 确保对象未销毁,已销毁则抛异常 */
  protected ensureNotDisposed(): void {
    if (this._disposed) {
      throw new Error('[Disposable]: object is disposed!');
    }
  }

  dispose(): void {
    if (this._disposed) return;
    this._disposed = true;
    this._collection.dispose();
  }
}
```

## 4.3 RxDisposable 基类

用于通过 `takeUntil(this.dispose$)` 自动取消订阅的场景:

```typescript
export class RxDisposable extends Disposable {
  protected readonly dispose$ = new Subject<void>();

  override dispose(): void {
    super.dispose();
    this.dispose$.next();
    this.dispose$.complete();
  }
}
```

## 4.4 DisposableLike 与 toDisposable

`disposeWithMe()` 接受 `DisposableLike`,**包括函数、RxJS Subscription 和 IDisposable**,**无需 `toDisposable()` 包装**:

```typescript
// ✅ RxJS Subscription 直接传入
this.disposeWithMe(someObservable$.subscribe(handler));

// ✅ 回调函数直接传入
this.disposeWithMe(() => {
  element.removeEventListener('click', handler);
});

// ✅ 已有 IDisposable
this.disposeWithMe(someService.registerCommand(command));
```

仅在需要**显式转换**为 `IDisposable`(函数返回值类型要求)时用 `toDisposable()`:

```typescript
function registerHandler(observable$: Observable<unknown>): IDisposable {
  return toDisposable(observable$.subscribe(handler));
}
```

> 历史代码中 `disposeWithMe(toDisposable(() => ...))` 仍然合法;新代码倾向于直接 `disposeWithMe(() => ...)`。

## 4.5 RCDisposable — 引用计数

多个消费方共享同一底层资源,归零时统一释放:

```typescript
import { RCDisposable } from '@termlnk/core';

const shared = new RCDisposable(rootResource);
shared.inc(); // 引用 +1
shared.inc(); // 引用 +1
shared.dec(); // 引用 -1
shared.dec(); // 引用 -1 → 归零,自动 dispose 内部 rootResource
```

适用:终端会话池、共享 WebSocket、共享 PTY。

## 4.6 订阅清理(必须遵守)

### 默认模式:`disposeWithMe(observable$.subscribe(...))`

绝大多数场景用这个,Service / Controller / Plugin 都适用:

```typescript
export class MyService extends Disposable {
  constructor(@IThemeService private readonly _themeService: IThemeService) {
    super();

    this.disposeWithMe(
      this._themeService.currentTheme$.subscribe((theme) => {
        this._applyTheme(theme);
      }),
    );
  }
}
```

### 进阶模式:`RxDisposable + takeUntil(this.dispose$)`

**仅在 pipe 内需要同时短路多条流时使用**。例如 `merge()` / `switchMap` 组合需要在 dispose 时整体停下,用 `takeUntil` 表达更清晰:

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

> 单一 Observable 订阅不要用 `takeUntil`,直接 `disposeWithMe(...subscribe(...))` 更直接。

## 4.7 Subject complete 规范

**`dispose()` 中必须 `complete()` 所有 Subject**:

```typescript
export class UserService extends Disposable {
  private readonly _currentUser$ = new BehaviorSubject<IUser>(defaultUser);
  private readonly _userChanged$ = new Subject<{ type: string; user: IUser }>();

  override dispose(): void {
    super.dispose();
    this._currentUser$.complete();
    this._userChanged$.complete();
  }
}
```

或者通过 `disposeWithMe` 统一管理:

```typescript
export class ThemeService extends Disposable {
  private readonly _currentTheme$ = new BehaviorSubject<Theme>(defaultTheme);
  readonly currentTheme$ = this._currentTheme$.asObservable();

  constructor() {
    super();
    this.disposeWithMe(toDisposable(() => {
      this._currentTheme$.complete();
    }));
  }
}
```

## 反模式

```typescript
// ❌ 裸 unsubscribe,容易漏
const sub = obs$.subscribe(...);
// 没有清理路径

// ❌ dispose() 忘记 complete subject
override dispose(): void {
  super.dispose();
  // 漏掉 this._xxx$.complete()
}

// ❌ 同时混用两种清理方式(没必要,挑一种)
this.disposeWithMe(
  obs$.pipe(takeUntil(this.dispose$)).subscribe(...),
);
```
