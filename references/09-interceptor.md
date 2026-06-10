# 09. 拦截器 (Interceptor) 模式

拦截器是**责任链模式**,允许多个模块以优先级顺序拦截和变换数据。

## 9.1 核心接口

```typescript
// 同步
export type InterceptorHandler<M = unknown, C = unknown> = (
  value: Nullable<M>,
  context: C,
  next: (value: Nullable<M>) => Nullable<M>,
) => Nullable<M>;

export interface IInterceptor<M, C> {
  id?: string;
  priority?: number;  // 数字越大越先执行
  handler: InterceptorHandler<M, C>;
}

// 异步
export type AsyncInterceptorHandler<M = unknown, C = unknown> = (
  value: Nullable<M>,
  context: C,
  next: (value: Nullable<M>) => Promise<Nullable<M>>,
) => Promise<Nullable<M>>;
```

## 9.2 拦截器注册

```typescript
export class DataInterceptorService extends Disposable {
  private readonly _interceptorManager = new InterceptorManager({ CELL_CONTENT, ROW_FILTERED });

  constructor() {
    super();
    // 注册默认透传拦截器 (最低优先级)
    this.disposeWithMe(
      this._interceptorManager.intercept(CELL_CONTENT, {
        priority: -1,
        handler: (value, _context, _next) => value,
      }),
    );
  }

  intercept<T extends IInterceptor<any, any>>(name: T, interceptor: T): IDisposable {
    return this.disposeWithMe(
      this._interceptorManager.intercept(name, interceptor),
    );
  }

  fetchThroughInterceptors<T, C>(name: IInterceptor<T, C>): (value: Nullable<T>, context: C) => Nullable<T> {
    return this._interceptorManager.fetchThroughInterceptors(name);
  }
}
```

## 9.3 实际使用

```typescript
export class ThemePlugin extends Plugin {
  override onReady(): void {
    const interceptorService = this._injector.get(DataInterceptorService);

    this.disposeWithMe(
      interceptorService.intercept(CELL_CONTENT, {
        id: 'theme-style-interceptor',
        priority: 8,
        handler: (cell, context, next) => {
          const style = this._getThemeStyle(context.row, context.col);
          if (style) {
            const newCell = { ...cell, themeStyle: style };
            return next(newCell);  // 传递给下一个拦截器
          }
          return next(cell);       // 无修改也要 next
        },
      }),
    );
  }
}
```

## 9.4 拦截器铁律

- 调用 `next(value)` 将值传递给链中下一个拦截器
- **不调用 `next()`** 则**短路**,中断后续拦截器
- `priority` 越大越先执行
- **禁止**在同一个 handler 中多次调用 `next()`
