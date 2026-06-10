# 12. 测试规范

## 12.1 Test Bed 工厂模式

测试**不走** `Core` / `registerPlugin` / `Plugin`。每个 spec 在本地写一个 `createTestBed()`,用裸 `Injector` 注册被测服务 + 假依赖,返回 `{ injector, ...被测对象 }`:

```typescript
// packages/terminal/src/services/session/__tests__/session.service.spec.ts
import { ICommandService, ILogService, Injector } from '@termlnk/core';

interface ITestBed {
  injector: Injector;
  service: ISessionService;
}

function createTestBed(): ITestBed {
  const injector = new Injector();
  injector.add([ILogService, { useClass: NoopLogService }]);
  injector.add([ICommandService, { useClass: CommandService }]);
  injector.add([ISessionService, { useClass: SessionService }]);
  return { injector, service: injector.get(ISessionService) };
}
```

> **禁止**用 `Plugin` 子类 + `_config: undefined` 搭 test bed —— 既无必要,也违反 [07-plugin.md §7.1](./07-plugin.md) 的 config 规范。`createTestBed()` 只是本地工厂,直接 `new Injector()` 按需注册依赖,`afterEach` 用 `injector.dispose()` 整体释放。

## 12.2 服务测试 — 最小化 mock

```typescript
describe('SessionService', () => {
  let injector: Injector;
  let service: ISessionService;

  beforeEach(() => {
    ({ injector, service } = createTestBed());
  });

  afterEach(() => {
    injector.dispose();
  });

  it('should emit on sessions$ when session created', () => {
    const sessions: ISession[][] = [];
    service.sessions$.subscribe((s) => sessions.push(s));

    service.createSession({ hostId: 'h1' });
    expect(sessions).toHaveLength(2); // initial empty + after create
    expect(sessions[1]).toHaveLength(1);
  });
});
```

## 12.3 Observable 测试 — 真实 Subject,不用 marble

```typescript
it('should update when observable emits', () => {
  const subject = new BehaviorSubject<string>('initial');
  const values: string[] = [];

  subject.subscribe((v) => values.push(v));
  subject.next('updated');

  expect(values).toEqual(['initial', 'updated']);
  subject.complete();
});

// React Hook
it('should sync observable to state', () => {
  const { result } = renderHook(() => useObservable(observable$, 'default'));

  expect(result.current).toBe('default');

  act(() => subject$.next('new value'));
  expect(result.current).toBe('new value');
});
```

## 12.4 命令测试

```typescript
describe('CreateSessionCommand', () => {
  let injector: Injector;
  let commandService: ICommandService;
  let sessionService: ISessionService;

  beforeEach(() => {
    ({ injector } = createTestBed());
    commandService = injector.get(ICommandService);
    sessionService = injector.get(ISessionService);

    // register command in beforeEach
    commandService.registerCommand(CreateSessionCommand);
  });

  afterEach(() => {
    injector.dispose();
  });

  it('should create session via command', async () => {
    const spy = vi.spyOn(sessionService, 'createSession');

    await commandService.executeCommand(CreateSessionCommand.id, { hostId: 'h1' });
    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ hostId: 'h1' }));
  });
});
```

## 12.5 测试文件组织

```
packages/xxx/src/
├── services/
│   ├── session.service.ts
│   └── __tests__/
│       └── session.service.spec.ts
├── commands/
│   └── __tests__/
│       └── create-session.command.spec.ts
└── __testing__/
    └── fakes.ts                   # 共享假实现/工具 (createTestBed 在各 spec 内本地定义)
```
