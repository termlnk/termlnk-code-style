# 10. React 组件与 Observable 集成

## 10.1 useObservable Hook — 首选

将 Observable 值绑定到 React 状态,自动管理订阅:

```typescript
import { useObservable } from '@wendellhu/redi/react-bindings';

function ThemeToggle() {
  const themeService = useDependency(IThemeService);
  const darkMode = useObservable(themeService.darkMode$, false);

  return (
    <button onClick={() => themeService.setDarkMode(!darkMode)}>
      {darkMode ? 'Light' : 'Dark'}
    </button>
  );
}
```

## 10.2 手动订阅管理 — useObservable 不够时

```typescript
function SessionList() {
  const sessionService = useDependency(ISessionService);
  const [sessions, setSessions] = useState<ISession[]>([]);
  const [active, setActive] = useState<Nullable<ISession>>(null);

  useEffect(() => {
    const subscriptions: Subscription[] = [];

    subscriptions.push(sessionService.sessions$.subscribe(setSessions));
    subscriptions.push(sessionService.activeSession$.subscribe(setActive));

    // ✅ 必须清理所有订阅
    return () => subscriptions.forEach((s) => s.unsubscribe());
  }, [sessionService]);

  return (/* ... */);
}
```

## 10.3 useObservableRef — 不触发重渲染

只需要读取最新值、不需要驱动重渲染时:

```typescript
function PerformanceMonitor() {
  const metricsService = useDependency(IMetricsService);
  const latestMetrics = useObservableRef(metricsService.metrics$);

  const handleExport = useCallback(() => {
    exportMetrics(latestMetrics.current);
  }, [latestMetrics]);

  return <button onClick={handleExport}>Export</button>;
}
```

## 10.4 菜单状态订阅 Hook

聚合菜单项多个 Observable 状态 (`disabled$` / `hidden$` / `activated$` / `value$`):

```typescript
function useToolbarItemStatus(menuItem: IDisplayMenuItem<IMenuItem>): IToolbarItemStatus {
  const { disabled$, hidden$, activated$, value$ } = menuItem;

  const [disabled, setDisabled] = useState(false);
  const [hidden, setHidden] = useState(false);
  const [activated, setActivated] = useState(false);
  const [value, setValue] = useState<any>();

  useEffect(() => {
    const subscriptions: Subscription[] = [];

    if (disabled$) {
      subscriptions.push(disabled$.subscribe(setDisabled));
    }
    if (hidden$) {
      subscriptions.push(hidden$.subscribe(setHidden));
    }
    if (activated$) {
      subscriptions.push(activated$.subscribe(setActivated));
    }
    if (value$) {
      subscriptions.push(value$.subscribe(setValue));
    }

    return () => subscriptions.forEach((s) => s.unsubscribe());
  }, [disabled$, hidden$, activated$, value$]);

  return { disabled, hidden, activated, value };
}
```

## React 组件硬约束

- `cn()` 条件类名**必须**用对象语法 `{ 'class': cond }`,**禁止** `cond && 'class'` 短路
- 静态类名通过模板字符串放进 `cn()` 第一个参数 (含 `tm:hover:*` 等伪类),不要拆到第二个参数
- 颜色用 base46 语义类 (`tm:bg-black` / `tm:text-light-grey` / `tm:border-line`),**禁止**硬编码 (`bg-gray-800`) 和 `dark:` 前缀
- React 组件文件用 PascalCase (`SessionList.tsx`)
