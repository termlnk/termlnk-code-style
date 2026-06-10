# 11. TypeScript 类型规范

## 11.1 interface vs type

| 场景 | 选择 | 示例 |
|-----|------|------|
| 对象结构 | `interface` | `interface ISessionConfig { ... }` |
| 联合类型 | `type` | `type SessionStatus = 'active' \| 'idle'` |
| 函数签名 | `type` | `type SessionCallback = (s: ISession) => void` |
| 服务接口 | `interface` | `interface ISessionService { ... }` |
| 扩展/继承 | `interface` | `interface IAdvancedConfig extends IBaseConfig` |

## 11.2 Nullable 类型

使用项目提供的 `Nullable<T>`,而非 `T | null | undefined`:

```typescript
// ✅ 正确
private _activeSession: Nullable<ISession> = null;
readonly activeSession$: Observable<Nullable<ISession>>;

// ❌ 错误
private _activeSession: ISession | null | undefined;
```

## 11.3 泛型参数命名

```
T    主类型
U, V 次要类型
K    Key
P    Params
R    Return
```

## 11.4 readonly 用法

```typescript
// ✅ 构造函数注入参数
constructor(
  @ILogService private readonly _logService: ILogService,
) {}

// ✅ Observable 属性
readonly sessions$: Observable<ISession[]>;

// ✅ 接口里的 Observable
export interface IService {
  readonly state$: Observable<IState>;
}

// ✅ 函数参数中的数组(防止意外修改)
function processRanges(ranges: readonly IRange[]): void {}
```

## 11.5 访问修饰符与命名

| 修饰符 | 用途 | 命名 |
|--------|------|------|
| `private` | 内部实现细节 | `_` 前缀:`_sessionMap` |
| `protected` | 子类可访问 | `_` 前缀:`protected readonly _injector` |
| `public` / 无修饰符 | 对外 API | 无前缀:`sessions$`, `createSession()` |
