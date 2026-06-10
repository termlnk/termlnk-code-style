# 05. Service 编写规范

## 5.1 完整服务模板 (照抄即可)

```typescript
import { createIdentifier } from '@termlnk/core/common/di';
import { Disposable, toDisposable } from '@termlnk/core/shared/lifecycle';
import type { Observable } from 'rxjs';
import { BehaviorSubject, Subject } from 'rxjs';

// ===== 1. 定义接口 =====
export interface ISessionService {
  /** 当前所有会话列表 */
  readonly sessions$: Observable<ISession[]>;
  /** 会话创建事件 */
  readonly sessionCreated$: Observable<ISession>;
  /** 当前活动会话 */
  readonly activeSession$: Observable<Nullable<ISession>>;

  /** 同步获取当前活动会话 */
  get activeSession(): Nullable<ISession>;

  createSession(config: ISessionConfig): ISession;
  closeSession(sessionId: string): boolean;
  setActiveSession(sessionId: string): void;
}

// ===== 2. 创建 DI 标识符 =====
export const ISessionService = createIdentifier<ISessionService>('terminal.session.service');

// ===== 3. 实现服务 =====
export class SessionService extends Disposable implements ISessionService {
  // ----- 3a. 私有 Subject (_ 前缀 + $ 后缀) -----
  private readonly _sessions$ = new BehaviorSubject<ISession[]>([]);
  private readonly _sessionCreated$ = new Subject<ISession>();
  private readonly _activeSession$ = new BehaviorSubject<Nullable<ISession>>(null);

  // ----- 3b. 公开 Observable ($ 后缀, asObservable) -----
  readonly sessions$: Observable<ISession[]> = this._sessions$.asObservable();
  readonly sessionCreated$: Observable<ISession> = this._sessionCreated$.asObservable();
  readonly activeSession$: Observable<Nullable<ISession>> = this._activeSession$.asObservable();

  // ----- 3c. 私有状态 -----
  private readonly _sessionMap = new Map<string, ISession>();

  // ----- 3d. 同步 getter (BehaviorSubject 必备) -----
  get activeSession(): Nullable<ISession> {
    return this._activeSession$.getValue();
  }

  // ----- 3e. 构造函数注入 -----
  constructor(
    @ILogService private readonly _logService: ILogService,
    @ICommandService private readonly _commandService: ICommandService,
  ) {
    super();
  }

  // ----- 3f. 公开方法 -----
  createSession(config: ISessionConfig): ISession {
    this.ensureNotDisposed();

    const session = new TerminalSession(config);
    this._sessionMap.set(session.id, session);
    this._sessions$.next([...this._sessionMap.values()]);
    this._sessionCreated$.next(session);

    this._logService.info('[SessionService]', `Session created: ${session.id}`);
    return session;
  }

  closeSession(sessionId: string): boolean {
    const session = this._sessionMap.get(sessionId);
    if (!session) {
      return false;
    }

    session.dispose();
    this._sessionMap.delete(sessionId);
    this._sessions$.next([...this._sessionMap.values()]);

    return true;
  }

  setActiveSession(sessionId: string): void {
    const session = this._sessionMap.get(sessionId);
    this._activeSession$.next(session ?? null);
  }

  // ----- 3g. dispose:清理所有资源 -----
  override dispose(): void {
    super.dispose();

    // complete 所有 Subject
    this._sessions$.complete();
    this._sessionCreated$.complete();
    this._activeSession$.complete();

    // 清理业务资源
    this._sessionMap.forEach((session) => session.dispose());
    this._sessionMap.clear();
  }
}
```

## 5.2 服务编写检查清单

写完任何 service 都过一遍:

- [ ] 接口、实现类、文件名三者都以 `Service` 结尾 (即使职责是 router/bus/manager,也不能用这些名词裸结尾;反例 `IDeepLinkRouter` → `IDeepLinkRouterService`)
- [ ] 接口与标识符同名,接口以 `I` 开头
- [ ] 继承 `Disposable` 或 `RxDisposable`
- [ ] 私有 Subject 使用 `_` 前缀 + `$` 后缀
- [ ] 公开 Observable 用 `.asObservable()` 转换
- [ ] 接口里 Observable 类型是 `Observable<T>`,不是 `Subject<T>`
- [ ] BehaviorSubject 提供同步 getter
- [ ] `dispose()` 中 `complete()` 所有 Subject
- [ ] `dispose()` 中清理所有 Map/Set/数组等集合
- [ ] 修改操作前调用 `ensureNotDisposed()`
- [ ] 构造函数参数 `private readonly` + `_` 前缀
- [ ] 日志通过 `ILogService` (禁止 `console.*`)

## 反模式

```typescript
// ❌ 直接暴露 Subject
readonly sessions$ = new BehaviorSubject<ISession[]>([]);

// ❌ 接口缺少 readonly / 类型错
export interface ISessionService {
  sessions$: BehaviorSubject<ISession[]>;
}

// ❌ dispose 忘记 super.dispose() 或忘记清理
override dispose(): void {
  this._sessions$.complete();
}

// ❌ 在 service 里用 console.log
this._logService // 应该用这个
```
