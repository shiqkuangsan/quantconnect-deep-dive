# 第六章：统一的回测与实盘抽象

[上一篇: 订单执行与真实性建模](./05-order-execution.md) | [下一篇: 开源生态与插件架构](./07-open-source-ecosystem.md)

## 概述

LEAN 引擎最优雅的设计之一就是**回测和实盘之间的统一抽象**。同一份算法代码，既可以在回测环境中运行，也可以在真实市场中运行，只需通过配置文件切换。这在量化交易框架中非常罕见。

对于有前端开发经验的工程师，这个设计理念就像：
- **React 组件** 能够同时在 SSR (服务端渲染) 和 CSR (客户端渲染) 中工作
- **Node.js 代码** 既能在服务器上运行，也能在浏览器中运行
- **数据库适配层** 支持 SQLite（开发）和 PostgreSQL（生产）无缝切换

这正是**依赖注入 (DI)** 和**接口抽象** 强大威力的体现。

---

## 1. 统一抽象的设计哲学

### 为什么这个设计如此优雅？

回测和实盘本质上是两个完全不同的环境：

| 维度 | 回测 | 实盘 |
|------|------|------|
| **时间流控制** | 虚拟时间，由数据驱动 | 真实的墙钟时间 |
| **数据来源** | 本地历史文件 | 实时交易所 WebSocket |
| **订单执行** | 模拟填充 (FillModel) | 真实经纪商 API |
| **延迟** | 零延迟（离线） | 网络延迟 + 经纪商处理 |
| **错误类型** | 无（理论环境） | 连接中断、订单拒绝、部分成交 |

看起来完全不兼容。但 LEAN 通过定义一套**统一的接口集合**，使得上层算法代码对这些差异完全无感。

### 类比：前端工程师能理解的比喻

```
同一个 React 组件
    ↓
SSR 渲染：fetch 数据、生成 HTML → 发送给浏览器
CSR 渲染：fetch 数据、生成 DOM → 挂载到页面
    ↓
从组件代码的角度：完全相同，只是运行时环境不同
```

LEAN 的设计完全类似：

```
同一个交易算法 OnData() / OnOrderEvent()
    ↓
回测模式：加载历史数据、模拟填充、收集统计
实盘模式：监听 WebSocket、调用 API、处理异常
    ↓
从算法代码的角度：完全相同，只是运行时环境不同
```

---

## 2. 接口抽象层：7 个关键接口

LEAN 定义了 7 个核心接口，通过不同的实现在回测和实盘间切换。

### 2.1 IDataFeed - 数据馈送

**接口定义：**

```csharp
public interface IDataFeed
{
    /// <summary>
    /// 获取下一个数据包
    /// </summary>
    BaseDataCollection GetNextTick();

    /// <summary>
    /// 订阅某个数据
    /// </summary>
    void Subscribe(SubscriptionDataConfig config);

    /// <summary>
    /// 是否还有数据
    /// </summary>
    bool IsConnected { get; }
}
```

**回测实现 - FileSystemDataFeed：**
- 从本地 `/Data/` 文件夹加载 CSV 历史数据
- 按时间戳排序，顺序返回
- 时间完全由数据驱动（下一行数据的时间戳就是"当前时间"）
- 速度快，可以一秒钟处理年年的数据

**实盘实现 - LiveTradingDataFeed：**
- 连接到 Interactive Brokers 等经纪商的 WebSocket
- 接收实时行情更新
- 时间由真实的系统时钟控制
- 可能存在数据丢失、网络延迟等问题

**实际代码示例：**

```csharp
// 回测模式
public class FileSystemDataFeed : IDataFeed
{
    private Queue<BaseDataCollection> _data;

    public BaseDataCollection GetNextTick()
    {
        if (_data.Count == 0) return null;
        return _data.Dequeue();  // 简单：直接取出下一个
    }
}

// 实盘模式
public class LiveTradingDataFeed : IDataFeed
{
    private IBrokerage _brokerage;

    public BaseDataCollection GetNextTick()
    {
        // 从 WebSocket 或其他实时源获取数据
        var tick = _brokerage.GetLatestTick(symbol);
        return new BaseDataCollection { tick };
    }
}
```

### 2.2 ITransactionHandler - 交易处理

**接口定义：**

```csharp
public interface ITransactionHandler
{
    /// <summary>
    /// 提交订单
    /// </summary>
    void AddOrder(Order order);

    /// <summary>
    /// 取消订单
    /// </summary>
    void CancelOrder(int orderId);

    /// <summary>
    /// 处理订单事件
    /// </summary>
    void ProcessOrderEvent(OrderEvent @event);
}
```

**回测实现 - BacktestingTransactionHandler：**
- 直接调用 FillModel 生成填充
- 立即返回 OrderFilled 事件
- 所有操作是同步的，确定性的

**实盘实现 - BrokerageTransactionHandler：**
- 通过 IBrokerage 接口发送订单到真实经纪商
- 等待经纪商的 OrderFilled / OrderPartiallyFilled / OrderCanceled 回调
- 订单可能被拒绝、被修改、部分成交等

**对比：**

```csharp
// 回测：同步、即时
public void AddOrder(Order order)
{
    var fill = _fillModel.GetFill(order, lastPrice);
    _orderEvents.Add(new OrderFilled { ... });  // 立即
}

// 实盘：异步、需要等待
public void AddOrder(Order order)
{
    _brokerage.PlaceOrder(order);  // 发送
    // 稍后从经纪商回调接收填充
    // _brokerageMessageHandler.OnOrderEvent(OrderFilled { ... })
}
```

### 2.3 IResultHandler - 结果处理

**接口定义：**

```csharp
public interface IResultHandler
{
    /// <summary>
    /// 保存性能统计
    /// </summary>
    void UpdateStats(Dictionary<string, decimal> stats);

    /// <summary>
    /// 发送结果
    /// </summary>
    void SendFinalResult(AlgorithmResults results);
}
```

**回测实现 - BacktestingResultHandler：**
- 在内存中累积所有交易
- 计算夏普比率、最大回撤、胜率等统计
- 回测完成后，返回完整的性能报告

**实盘实现 - LiveTradingResultHandler：**
- 实时上报到 QuantConnect 云平台
- 显示在 Web 控制台上
- 持久化到数据库
- 支持实时监控

### 2.4 ISetupHandler - 初始化处理

**接口定义：**

```csharp
public interface ISetupHandler
{
    /// <summary>
    /// 初始化算法
    /// </summary>
    AlgorithmSetupResult Setup(IAlgorithm algorithm);

    /// <summary>
    /// 检查初始化是否成功
    /// </summary>
    bool SetupErrorOccurred { get; }
}
```

**回测实现 - BacktestingSetupHandler：**
- 检查必要的历史数据是否存在
- 设置开始和结束日期
- 初始化虚拟现金

**实盘实现 - BrokerageSetupHandler：**
- 连接到经纪商 API
- 验证账户凭证
- 从经纪商同步现有持仓和开放订单
- 验证足够的购买力

### 2.5 IRealTimeHandler - 实时处理

**接口定义：**

```csharp
public interface IRealTimeHandler
{
    /// <summary>
    /// 添加事件到调度队列
    /// </summary>
    void Add(ScheduledEvent @event);

    /// <summary>
    /// 获取下一个应该触发的事件
    /// </summary>
    bool TryGetNextScheduledEvent(out ScheduledEvent @event);
}
```

**回测实现 - BacktestingRealTimeHandler：**
- 按数据时间戳管理调度事件
- 完全确定性

**实盘实现 - LiveTradingRealTimeHandler：**
- 用真实系统时钟（Timer）管理调度事件
- 支持市场开盘/收盘事件
- 支持自定义时间事件

### 2.6 IBrokerage - 经纪商接口

**接口定义：**

```csharp
public interface IBrokerage
{
    /// <summary>
    /// 下达订单
    /// </summary>
    void PlaceOrder(Order order);

    /// <summary>
    /// 取消订单
    /// </summary>
    void CancelOrder(Order order);

    /// <summary>
    /// 获取账户持仓
    /// </summary>
    List<Holding> GetAccountHoldings();

    /// <summary>
    /// 连接状态
    /// </summary>
    bool IsConnected { get; }
}
```

**回测实现 - BacktestingBrokerage：**
- 纯粹模拟，不连接任何真实经纪商
- 订单立即通过 FillModel 生成填充
- 持仓通过内存字典管理

**实盘实现 - InteractiveBrokersBrokerage, AlpacaBrokerage, etc：**
- 真实的网络连接
- HTTP REST API / WebSocket 通信
- 实时订单确认、部分成交、拒绝等

**对比表：**

| 接口 | 回测实现 | 实盘实现 |
|------|---------|---------|
| **IDataFeed** | FileSystemDataFeed | LiveTradingDataFeed |
| **ITransactionHandler** | BacktestingTransactionHandler | BrokerageTransactionHandler |
| **IResultHandler** | BacktestingResultHandler | LiveTradingResultHandler |
| **ISetupHandler** | BacktestingSetupHandler | BrokerageSetupHandler |
| **IRealTimeHandler** | BacktestingRealTimeHandler | LiveTradingRealTimeHandler |
| **IBrokerage** | BacktestingBrokerage | InteractiveBrokersBrokerage |

### 2.7 IAlgorithmHandlers - 处理器包

```csharp
public interface IAlgorithmHandlers
{
    IDataFeed DataFeed { get; set; }
    ITransactionHandler TransactionHandler { get; set; }
    IResultHandler ResultHandler { get; set; }
    ISetupHandler SetupHandler { get; set; }
    IRealTimeHandler RealTimeHandler { get; set; }
    IBrokerage Brokerage { get; set; }
}
```

这个接口将所有 6 个接口打包在一起，方便传递和管理。

---

## 3. 配置切换机制：Composer 类

### 3.1 回测模式配置

**config.json (Backtesting):**

```json
{
  "environment": "backtesting",
  "data-folder": "./Data",
  "cache-folder": "./Cache",
  "version": 2,
  "live-mode": false,

  "algorithm-type-name": "MyAlgorithm",
  "algorithm-language": "CSharp",
  "parameters": {},

  "composer": {
    "data-feed": "FileSystemDataFeed",
    "transaction-handler": "BacktestingTransactionHandler",
    "result-handler": "BacktestingResultHandler",
    "setup-handler": "BacktestingSetupHandler",
    "real-time-handler": "BacktestingRealTimeHandler",
    "brokerage": "BacktestingBrokerage"
  }
}
```

### 3.2 实盘模式配置

**config.json (Live Trading with Interactive Brokers):**

```json
{
  "environment": "live-interactive-brokers",
  "data-folder": "./Data",
  "cache-folder": "./Cache",
  "version": 2,
  "live-mode": true,

  "algorithm-type-name": "MyAlgorithm",
  "algorithm-language": "CSharp",
  "parameters": {},

  "composer": {
    "data-feed": "LiveTradingDataFeed",
    "transaction-handler": "BrokerageTransactionHandler",
    "result-handler": "LiveTradingResultHandler",
    "setup-handler": "BrokerageSetupHandler",
    "real-time-handler": "LiveTradingRealTimeHandler",
    "brokerage": "InteractiveBrokersBrokerage"
  },

  "interactive-brokers": {
    "host": "127.0.0.1",
    "port": 7496,
    "account": "DU123456",
    "clientId": 1
  }
}
```

### 3.3 Composer 类：依赖注入容器

```csharp
public class Composer
{
    /// <summary>
    /// 根据配置，解析接口 → 实现的映射
    /// </summary>
    public static IAlgorithmHandlers CreateHandlers(
        Config config,
        IAlgorithm algorithm)
    {
        var handlers = new AlgorithmHandlers();

        // 读取 config.json 中的 composer 部分
        var composerConfig = config["composer"];

        // 动态加载实现类
        handlers.DataFeed = InstantiateType(
            composerConfig["data-feed"],
            config
        ) as IDataFeed;

        handlers.TransactionHandler = InstantiateType(
            composerConfig["transaction-handler"],
            config
        ) as ITransactionHandler;

        handlers.ResultHandler = InstantiateType(
            composerConfig["result-handler"],
            config
        ) as IResultHandler;

        // ... 其他接口

        return handlers;
    }

    private static object InstantiateType(string typeName, Config config)
    {
        // 使用反射动态创建实例
        var type = Type.GetType(typeName);
        return Activator.CreateInstance(type, config);
    }
}
```

### 3.4 对比前端框架的 DI

**Angular (TypeScript):**
```typescript
providers: [
  {provide: DataService, useClass: MockDataService},  // 测试
  {provide: DataService, useClass: HttpDataService},  // 生产
]
```

**React Context:**
```jsx
<Provider value={backendService}>
  <App />
</Provider>
```

**Webpack resolve.alias:**
```js
resolve: {
  alias: {
    '@database': process.env.NODE_ENV === 'production'
      ? './postgres.js'
      : './sqlite.js'
  }
}
```

**LEAN Composer:**
```json
{
  "composer": {
    "brokerage": "BacktestingBrokerage"  // 测试
    // 改成 "InteractiveBrokersBrokerage"  // 生产
  }
}
```

本质完全相同：通过配置驱动，在运行时切换实现。

---

## 4. 回测模式深度解析

### 4.1 时间是虚拟的

在回测中，"当前时间" 由数据的时间戳完全控制：

```csharp
public class FileSystemDataFeed : IDataFeed
{
    private DateTime _currentTime;

    public BaseDataCollection GetNextTick()
    {
        var tick = _nextDataPoint;  // 从文件读取

        // 时间完全由数据驱动
        _currentTime = tick.Time;

        _algorithm.SetCurrentTime(_currentTime);

        return tick;
    }
}
```

因此：
- 从 2020 年 1 月 1 日到 2023 年 12 月 31 日的 4 年数据，可能在几秒钟内完成
- 时间完全可控，没有随机性
- 可以精确重现每个决策的历史环境

### 4.2 数据被快速回放

```
加载历史数据 CSV:
AAPL,2023-01-01 09:30:00,150.00,151.00,149.50,150.50,1000000
AAPL,2023-01-01 09:31:00,150.50,151.50,150.00,151.00,900000
AAPL,2023-01-01 09:32:00,151.00,151.80,150.80,151.50,850000
...

回放流程：
for each row in csv {
    tick = parse(row)
    algorithm.OnData(tick)  // 调用算法的 OnData
    if (tick.isFill) algorithm.OnOrderEvent(fill)
    // 检查是否超时
    // 检查是否需要强制平仓
}

结果：~200万根 K线 在 5-10 秒内处理完
```

### 4.3 所有回调都是同步的

```csharp
// 算法代码
public override void OnData(Slice data)
{
    var price = data.Bars[_symbol].Close;

    if (price > _ma.Current.Value * 1.05)
    {
        SetHoldings(_symbol, 1.0);  // 买入
    }
}

public override void OnOrderEvent(OrderEvent fill)
{
    Debug($"Order filled: {fill.Quantity} @ {fill.FillPrice}");
}

// 回测执行流程
while (dataFeed.IsConnected)
{
    var slice = dataFeed.GetNextTick();

    // 同步调用 OnData
    algorithm.OnData(slice);

    // 同步处理订单填充
    var orderEvent = transactionHandler.ProcessOrder(lastOrder);
    algorithm.OnOrderEvent(orderEvent);  // 立即执行
}
```

所有操作都是**同步执行**，因此：
- 不需要处理竞态条件
- 不需要考虑异步等待
- 调试非常简单：可以单步执行

### 4.4 填充由 FillModel 模拟

```csharp
public class FillModel : IFillModel
{
    public OrderEvent GetFill(Order order, Slice slice)
    {
        var symbolData = slice.Bars[order.Symbol];

        // 市价单：用当前 close 作为填充价
        if (order is MarketOrder)
        {
            var fillPrice = symbolData.Close;
            return new OrderEvent
            {
                FillPrice = fillPrice,
                FillQuantity = order.Quantity,
                Status = OrderStatus.Filled
            };
        }

        // 限价单：检查价格是否在范围内
        if (order is LimitOrder limitOrder)
        {
            if (symbolData.High >= limitOrder.LimitPrice &&
                symbolData.Low <= limitOrder.LimitPrice)
            {
                return new OrderEvent
                {
                    FillPrice = limitOrder.LimitPrice,
                    FillQuantity = order.Quantity,
                    Status = OrderStatus.Filled
                };
            }
            return new OrderEvent { Status = OrderStatus.None };
        }

        return null;
    }
}
```

可以通过重写 FillModel 来实现：
- 滑点 (slippage)
- 部分成交 (partial fills)
- 订单拒绝 (rejections)
- 延迟成交 (delay)

### 4.5 零网络延迟

由于一切都是本地文件操作，没有网络调用，因此：
- 订单立即提交和填充
- 没有网络故障的可能
- 完全确定性

### 4.6 统计由 BacktestingResultHandler 收集

```csharp
public class BacktestingResultHandler : IResultHandler
{
    private List<Trade> _trades;
    private decimal _initialCash;
    private decimal _finalCash;

    public void UpdateStats()
    {
        var returns = (_finalCash - _initialCash) / _initialCash;
        var sharpeRatio = CalculateSharpe(_trades);
        var maxDrawdown = CalculateMaxDD(_trades);

        // 保存到结果中
        _results.TotalReturn = returns;
        _results.SharpeRatio = sharpeRatio;
        _results.Drawdown = maxDrawdown;
    }
}
```

### 4.7 陷阱：过拟合和前向偏差

**过拟合 (Overfitting):**
- 在回测数据上反复调整参数，直到性能完美
- 但在从未见过的未来数据上表现糟糕
- 原因：参数被"调整"到特定的历史模式

**前向偏差 (Look-Ahead Bias):**
```csharp
public override void OnData(Slice data)
{
    // 错误：使用了当前 bar 的 close 来做决策
    var close = data.Bars[_symbol].Close;

    // 但在实盘中，当前 bar 的 close 直到市场收盘才会知道！
    if (close > _threshold)  // 这是不可能实现的信息
    {
        SetHoldings(_symbol, 1.0);
    }
}
```

---

## 5. 实盘模式深度解析

### 5.1 时间是真实的墙钟时间

```csharp
public class LiveTradingRealTimeHandler : IRealTimeHandler
{
    private DateTime _realWallClock;

    public void UpdateTime()
    {
        _realWallClock = DateTime.UtcNow;  // 系统时钟
    }
}
```

时间不再由数据驱动，而是由真实系统时钟控制。这意味着：
- 1 小时的实盘交易花 1 小时的时间
- 无法"快进"
- 需要在市场开放时间运行

### 5.2 数据从实时来源到达

```csharp
public class LiveTradingDataFeed : IDataFeed
{
    private IBrokerage _brokerage;
    private Queue<Tick> _incomingTicks;

    public void OnBrokerageData(Tick tick)
    {
        // 从 WebSocket / REST API 异步接收数据
        _incomingTicks.Enqueue(tick);
    }

    public BaseDataCollection GetNextTick()
    {
        // 返回队列中最新的数据
        return _incomingTicks.Dequeue();
    }
}
```

数据到达可能不规则：
- 市场繁忙时，可能每毫秒多个数据点
- 市场清淡时，可能几秒没有新数据
- 网络故障时，可能出现数据缺口

### 5.3 大多数回调仍是同步的，但订单事件是异步的

```csharp
// 算法代码（主线程）
public override void OnData(Slice data)
{
    // 这仍然是同步的
    if (data.Bars[_symbol].Close > _threshold)
    {
        SetHoldings(_symbol, 1.0);  // 立即下单
    }
}

// 订单事件处理（来自经纪商线程）
public override void OnOrderEvent(OrderEvent fill)
{
    // 这可能来自另一个线程（经纪商回调）
    Debug($"Order filled at {fill.FillPrice}");
}

// 执行流程
MainThread:
  1. 接收数据
  2. 同步调用 OnData()
  3. 订单被添加到队列
  4. 发送到经纪商 API

BrokerageThread:
  ... 等待经纪商回复 ...
  收到成交消息
  回调 OnOrderEvent()  // 可能在任何时刻
```

这需要考虑**线程安全**：
- 持仓字典需要加锁
- 订单列表需要同步
- 现金余额更新需要原子操作

### 5.4 订单发往真实经纪商

```csharp
public class BrokerageTransactionHandler : ITransactionHandler
{
    private IBrokerage _brokerage;

    public void AddOrder(Order order)
    {
        // 不是模拟，而是真实地发送到经纪商
        var orderId = _brokerage.PlaceOrder(order);

        // 保存订单
        _openOrders[orderId] = order;
    }
}

public class InteractiveBrokersBrokerage : IBrokerage
{
    private TwsSocket _socket;

    public void PlaceOrder(Order order)
    {
        // 转换为 IB 的订单格式
        var ibOrder = new Contract { ... };

        // 发送给 IB API
        _socket.PlaceOrder(ibOrder);

        // 等待回调 OnExecution()
    }
}
```

### 5.5 需要处理多种异常情况

**连接丢失：**
```csharp
public override void OnBrokerageDisconnect()
{
    Debug("Connection lost!");
    // 暂停交易？
    // 尝试重新连接？
    // 发送告警？
}
```

**订单拒绝：**
```csharp
public override void OnOrderEvent(OrderEvent fill)
{
    if (fill.Status == OrderStatus.Rejected)
    {
        Debug($"Order rejected: {fill.Message}");
        // 原因可能：
        // - 账户购买力不足
        // - 交易品种不支持
        // - 订单参数无效
        // - 市场已关闭
    }
}
```

**部分成交：**
```csharp
public override void OnOrderEvent(OrderEvent fill)
{
    if (fill.Status == OrderStatus.PartiallyFilled)
    {
        Debug($"Partial fill: {fill.FillQuantity} of {order.Quantity}");
        // 需要继续等待剩余部分
        // 或者取消未成交部分
    }
}
```

**数据延迟：**
```csharp
// 行情数据可能有几百毫秒的延迟
// 这会影响策略的决策时机
// 需要在回测中模拟这种延迟
```

### 5.6 状态同步：从经纪商加载现有头寸

在启动实盘时，需要从经纪商同步：

```csharp
public class BrokerageSetupHandler : ISetupHandler
{
    public AlgorithmSetupResult Setup(IAlgorithm algorithm)
    {
        // 连接到经纪商
        var brokerage = ConnectToBrokerage();

        // 获取现有持仓
        var holdings = brokerage.GetAccountHoldings();
        foreach (var holding in holdings)
        {
            algorithm.SetHolding(holding.Symbol, holding.Quantity);
        }

        // 获取开放订单
        var openOrders = brokerage.GetOpenOrders();
        foreach (var order in openOrders)
        {
            algorithm.AddOpenOrder(order);
        }

        // 获取账户信息
        var account = brokerage.GetAccountInfo();
        algorithm.SetCash(account.BuyingPower);

        return new AlgorithmSetupResult { Success = true };
    }
}
```

这确保了算法的内部状态与经纪商账户的真实状态保持一致。

### 5.7 Warm-up 期：预热指标

在开始交易之前，需要加载足够的历史数据来初始化指标：

```csharp
public class MyAlgorithm : QCAlgorithm
{
    private SimpleMovingAverage _sma;

    public override void Initialize()
    {
        // 告诉 LEAN：需要 50 天的历史数据才能开始
        SetWarmupPeriod(50);

        _sma = SMA(_symbol, 50);
    }

    public override void OnData(Slice data)
    {
        // 只有当预热完成后，才会调用这个方法
        // 此时 _sma 已经有 50 个数据点，可以使用

        if (_sma.IsReady)
        {
            // 开始交易
        }
    }
}
```

预热过程：
1. 在实盘前，自动从历史数据源加载 50 天的数据
2. 按时间顺序回放，更新所有指标
3. 不产生任何交易信号（只是初始化）
4. 初始化完成后，才开始接收实时数据

### 5.8 错误处理和恢复机制

```csharp
public class LiveTradingAlgorithm : QCAlgorithm
{
    public override void OnData(Slice data)
    {
        try
        {
            // 核心交易逻辑
            var signal = CalculateSignal(data);
            if (signal > 0)
            {
                SetHoldings(_symbol, 1.0);
            }
        }
        catch (Exception ex)
        {
            Error($"Error in OnData: {ex.Message}");

            // 可能的恢复策略：
            // 1. 记录错误，继续运行
            // 2. 发送告警给用户
            // 3. 暂停交易直到手动干预
            // 4. 执行紧急平仓
        }
    }

    public override void OnBrokerageDisconnect()
    {
        // 经纪商连接中断
        Error("Brokerage disconnected!");

        // 等待重新连接
        var maxRetries = 5;
        for (int i = 0; i < maxRetries; i++)
        {
            System.Threading.Thread.Sleep(5000);
            if (Brokerage.IsConnected)
            {
                Log("Reconnected!");
                break;
            }
        }

        if (!Brokerage.IsConnected)
        {
            Error("Failed to reconnect. Liquidating positions...");
            Liquidate();  // 强制平仓
        }
    }
}
```

---

## 6. 模拟盘（Paper Trading）：最好的中间地带

模拟盘是回测和实盘的完美混合体：

| 特性 | 回测 | 模拟盘 | 实盘 |
|------|------|--------|------|
| **时间** | 虚拟 | 实时 | 实时 |
| **数据** | 历史文件 | 实时 | 实时 |
| **执行** | 模拟 | 模拟 | 真实 |
| **风险** | 无 | 无 | 有 |

### 6.1 模拟盘的配置

```json
{
  "environment": "paper-trading",
  "live-mode": true,

  "composer": {
    "data-feed": "LiveTradingDataFeed",        // 实时数据
    "transaction-handler": "BacktestingTransactionHandler",  // 模拟执行
    "result-handler": "LiveTradingResultHandler",
    "setup-handler": "BrokerageSetupHandler",
    "real-time-handler": "LiveTradingRealTimeHandler",
    "brokerage": "BacktestingBrokerage"        // 模拟经纪商
  }
}
```

关键点：
- 使用 **LiveTradingDataFeed** 获取实时行情
- 使用 **BacktestingBrokerage** 进行虚拟成交（无真实风险）
- 在实际市场条件下测试算法

### 6.2 模拟盘的好处

1. **真实的数据和时间**：用实时数据测试，避免回测偏差
2. **真实的延迟**：感受网络延迟的影响
3. **无金融风险**：订单不真正执行
4. **完整的测试**：可以测试错误处理、连接管理等
5. **QuantConnect 支持**：平台有完全的模拟盘功能

### 6.3 从模拟盘到实盘的迁移

只需修改一行配置：

```json
{
  "brokerage": "BacktestingBrokerage"  // 改成
  "brokerage": "InteractiveBrokersBrokerage"  // 这样
}
```

算法代码完全不变！

---

## 7. 差异点与注意事项：抽象何时"漏水"

虽然统一抽象设计很优雅，但实际上还是有一些细微的差异需要开发者注意。

### 7.1 滑点 (Slippage)

**回测中：**
```csharp
// FillModel 可以模拟滑点
fillPrice = limitPrice * (1 + 0.001);  // 模拟 0.1% 的滑点
```

**实盘中：**
```csharp
// 真实滑点取决于：
// - 订单大小
// - 市场流动性
// - 市场波动性
// - 相对于 bid-ask spread 的位置

// 典型：0.01% 到 0.5% 不等
```

**风险：** 回测的滑点模型可能太乐观，导致实盘表现不如预期。

### 7.2 数据质量差异

**回测中：**
- 数据是"干净的"：来自官方来源，已检查
- 没有数据缺口、没有异常价格

**实盘中：**
- 可能有数据缺胜（网络中断）
- 可能有异常成交（极端市场条件）
- 可能有延迟（行情推送延迟）

**处理：**
```csharp
public override void OnData(Slice data)
{
    if (data.ContainsKey(_symbol))  // 检查数据是否存在
    {
        var bar = data[_symbol];

        // 检查数据的时间戳，避免处理过期数据
        if ((DateTime.UtcNow - bar.EndTime).TotalSeconds < 60)
        {
            // 这是最近的数据，可以使用
        }
    }
}
```

### 7.3 时间差异：收盘价 vs 下一开盘价

**回测中，市价单在收盘时提交：**
```csharp
public override void OnData(Slice data)
{
    if (isClosingTime)  // 例如 15:59
    {
        SetHoldings(_symbol, 1.0);  // 市价单

        // FillModel 使用当前 bar 的 close 作为成交价
        // 假设在 15:59 的 close 价成交
    }
}
```

**实盘中，情况可能不同：**
```csharp
// 如果在 15:59 提交市价单
// 但市场要到 16:00 才真正成交
// 成交价可能是 16:00 bar 的 open，而不是 15:59 的 close

// 这会导致成交价的 1-2% 差异
```

### 7.4 订单拒绝处理

**回测中：** 订单几乎不会被拒绝

**实盘中：** 订单可能因以下原因被拒绝：
- 购买力不足
- 市场已关闭
- 品种不支持（例如某些代码在 Interactive Brokers 不可交易）
- 订单参数无效

**代码：**
```csharp
public override void OnOrderEvent(OrderEvent orderEvent)
{
    if (orderEvent.Status == OrderStatus.Rejected)
    {
        Error($"Order rejected: {orderEvent.Message}");

        // 回测中不会到这里
        // 实盘中需要处理

        // 可能的恢复：
        // 1. 重试
        // 2. 调整订单参数
        // 3. 跳过这个交易
    }
}
```

### 7.5 连接管理

**回测中：** 不需要考虑连接

**实盘中：** 需要处理连接问题
```csharp
public override void OnBrokerageConnect()
{
    Log("Connected to broker");
    // 重新初始化状态
}

public override void OnBrokerageDisconnect()
{
    Error("Disconnected from broker!");
    // 尝试重新连接？
    // 平仓？
    // 发送告警？
}
```

### 7.6 应急平仓

**回测中：** 虽然有强制平仓逻辑，但通常不会触发

**实盘中：** 如果以下情况，可能触发强制平仓：
- 亏损超过限额
- 连接中断超过时间限制
- 账户风险警告

```csharp
public override void OnData(Slice data)
{
    var currentValue = Portfolio.TotalPortfolioValue;
    var initialValue = Portfolio.GetInvestedAmount();
    var unrealizedLoss = (currentValue - initialValue) / initialValue;

    if (unrealizedLoss < -0.1)  // 亏损超过 10%
    {
        Error("Loss exceeds limit. Liquidating...");
        Liquidate();
    }
}
```

### 7.7 完整的差异表

| 特性 | 回测 | 实盘 | 备注 |
|------|------|------|------|
| **滑点** | 可配置，通常乐观 | 取决于市场 | 需要更大的缓冲 |
| **数据缺胜** | 无 | 可能发生 | 需要数据验证 |
| **数据延迟** | 零 | 通常 <100ms | 需要考虑在决策中 |
| **订单拒绝** | 罕见 | 可能发生 | 需要异常处理 |
| **连接故障** | 无 | 可能发生 | 需要重连逻辑 |
| **部分成交** | 可通过 FillModel | 常见 | 需要管理未成交部分 |
| **撤单延迟** | 立即 | 可能有延迟 | 已撤单但仍被成交 |
| **市场时间** | 由数据驱动 | 真实时间 | 可能需要市场开盘检查 |

---

## 8. 从回测到实盘的迁移清单

在将策略从回测移到实盘前，检查以下清单：

### 前期准备（2-3 周）

- [ ] **用模拟盘验证**：运行至少 2 周的模拟盘，观察交易频率和夏普比
- [ ] **检查经纪商兼容性**：确认所有交易品种在目标经纪商可交易
- [ ] **设置 API 连接**：成功连接到实际经纪商（模拟账户）
- [ ] **验证数据源**：确保实时数据源（价格、基本面）可靠

### 策略调整（1-2 周）

- [ ] **调整真实性设置**：增加滑点、费用、借贷利率到逼真水平
- [ ] **压力测试**：在历史波动率高的时期进行回测
- [ ] **检查前向偏差**：确保没有使用未来信息
- [ ] **检查过拟合**：在不同时期进行回测（样本外数据）
- [ ] **资本需求分析**：计算最大回撤，确保资本充足

### 代码审查（1 周）

- [ ] **异常处理**：添加 try-catch 块，处理订单拒绝、连接丢失等
- [ ] **日志记录**：设置详细的日志，便于监控和调试
- [ ] **限制器**：实现最大亏损止损、每日交易限制等
- [ ] **监控告警**：设置关键指标告警（例如连接状态、异常成交）
- [ ] **平仓机制**：确保在紧急情况下可以快速平仓

### 账户和风险管理（1 周）

- [ ] **初始资本**：确定第一次投入的资本量（建议较小）
- [ ] **杠杆**：确认是否使用杠杆及比例
- [ ] **交易所费用**：了解交易所费用、佣金、借贷利率
- [ ] **账户类型**：选择合适的账户类型（现金 vs 保证金）
- [ ] **保险**：了解经纪商的破产保护政策

### 实盘前一天

- [ ] **干运行**：在模拟账户上运行完整流程
- [ ] **检查日志**：确保没有警告或错误信息
- [ ] **团队通知**：通知团队/管理层即将启动实盘
- [ ] **备份代码**：备份所有代码版本
- [ ] **紧急联系**：确保能快速联系到技术支持

### 实盘第一天

- [ ] **早期到位**：比市场开盘至少提前 30 分钟启动
- [ ] **监控面板**：打开实时监控面板，时刻关注
- [ ] **小额交易**：第一笔交易用极小的头寸（例如总资本的 1%）
- [ ] **交易日志**：记录每笔交易及理由
- [ ] **连接检查**：定期检查与经纪商的连接状态

### 实盘第一周

- [ ] **日报告**：每天分析交易结果
- [ ] **异常监控**：监控是否出现回测中未见的行为
- [ ] **参数微调**：如果发现问题，记录但不急于改改
- [ ] **风险监控**：时刻关注最大回撤和杠杆水平
- [ ] **通知机制**：确保告警能及时通知到相关人员

### 稳定期（第二周及以后）

- [ ] **性能对标**：将实盘结果与回测结果对标
- [ ] **持续监控**：每周分析一次交易绩效
- [ ] **市场变化**：观察是否出现新的市场模式需要适应
- [ ] **优化迭代**：基于实盘反馈优化策略参数
- [ ] **文档更新**：记录实盘过程中发现的问题和解决方案

---

## 9. 代码示例对比：同一个算法的三种模式

让我们用一个完整的简单策略（均线交叉）来展示回测、模拟盘、实盘的差异。

### 9.1 算法代码（完全相同）

```csharp
public class UnifiedAlgorithm : QCAlgorithm
{
    private string _symbol = "AAPL";
    private SimpleMovingAverage _sma20;
    private SimpleMovingAverage _sma50;
    private bool _isLong = false;

    public override void Initialize()
    {
        // 基础设置（所有模式通用）
        SetStartDate(2023, 1, 1);
        SetEndDate(2023, 12, 31);
        SetCash(100000);

        AddEquity(_symbol);

        // 注册指标
        _sma20 = SMA(_symbol, 20);
        _sma50 = SMA(_symbol, 50);

        // 预热期：需要 50 天的历史数据
        SetWarmupPeriod(50);
    }

    public override void OnData(Slice data)
    {
        // 异常处理：在实盘中很重要
        try
        {
            // 检查数据是否存在
            if (!data.ContainsKey(_symbol))
                return;

            // 检查指标是否就绪
            if (!_sma20.IsReady || !_sma50.IsReady)
                return;

            var close = data[_symbol].Close;
            var sma20 = _sma20.Current.Value;
            var sma50 = _sma50.Current.Value;

            // 黄金交叉：买入
            if (sma20 > sma50 && !_isLong)
            {
                SetHoldings(_symbol, 1.0);
                _isLong = true;
                Log($"BUY: SMA20 {sma20:F2} > SMA50 {sma50:F2}");
            }

            // 死亡交叉：卖出
            if (sma20 < sma50 && _isLong)
            {
                SetHoldings(_symbol, 0);
                _isLong = false;
                Log($"SELL: SMA20 {sma20:F2} < SMA50 {sma50:F2}");
            }
        }
        catch (Exception ex)
        {
            Error($"Error in OnData: {ex.Message}");
        }
    }

    public override void OnOrderEvent(OrderEvent orderEvent)
    {
        // 订单事件回调
        if (orderEvent.Status == OrderStatus.Filled)
        {
            Log($"Order filled: {orderEvent.Direction} {orderEvent.FillQuantity} @ {orderEvent.FillPrice:F2}");
        }
        else if (orderEvent.Status == OrderStatus.Rejected)
        {
            Error($"Order rejected: {orderEvent.Message}");
        }
    }

    public override void OnBrokerageDisconnect()
    {
        // 这在回测中永远不会被调用
        // 在实盘中可能会被调用
        Error("Brokerage connection lost!");
    }
}
```

这个算法代码对**所有三种模式都相同**。

### 9.2 回测模式执行

**配置文件 (config-backtest.json):**

```json
{
  "environment": "backtesting",
  "data-folder": "./Data",
  "live-mode": false,

  "algorithm-type-name": "UnifiedAlgorithm",
  "algorithm-language": "CSharp",

  "composer": {
    "data-feed": "FileSystemDataFeed",
    "transaction-handler": "BacktestingTransactionHandler",
    "result-handler": "BacktestingResultHandler",
    "setup-handler": "BacktestingSetupHandler",
    "real-time-handler": "BacktestingRealTimeHandler",
    "brokerage": "BacktestingBrokerage"
  }
}
```

**执行流程和输出：**

```
启动回测...

[2023-01-01 09:30:00] Warming up with 50 days of historical data...
[2023-02-20 09:30:00] Warm-up complete. Starting strategy...

[2023-03-15 15:59:00] BUY: SMA20 152.34 > SMA50 150.12
[2023-03-15 16:00:00] Order filled: BUY 930 @ 152.45

[2023-05-22 15:59:00] SELL: SMA20 148.56 < SMA50 149.87
[2023-05-22 16:00:00] Order filled: SELL 930 @ 148.34

...（继续处理 2023 年的所有数据）...

[2023-12-29 15:59:00] Backtest complete!

=============== RESULTS ===============
Total Return:        12.34%
Annual Return:       12.34%
Sharpe Ratio:        1.24
Max Drawdown:        -8.56%
Win Rate:            56%
Total Trades:        24
Avg Trade:           0.51%
=======================================

耗时: 3.2 秒
处理的数据点: ~252,000
```

**特点：**
- 速度快：一年的数据在几秒内完成
- 完整的统计信息：所有性能指标一应俱全
- 可再现性：每次运行结果完全相同

### 9.3 模拟盘模式执行

**配置文件 (config-paper.json):**

```json
{
  "environment": "paper-trading",
  "data-folder": "./Data",
  "live-mode": true,

  "algorithm-type-name": "UnifiedAlgorithm",
  "algorithm-language": "CSharp",

  "composer": {
    "data-feed": "LiveTradingDataFeed",
    "transaction-handler": "BacktestingTransactionHandler",
    "result-handler": "LiveTradingResultHandler",
    "setup-handler": "BrokerageSetupHandler",
    "real-time-handler": "LiveTradingRealTimeHandler",
    "brokerage": "BacktestingBrokerage"
  },

  "interactive-brokers": {
    "host": "127.0.0.1",
    "port": 7496,
    "account": "PAPER_ACCOUNT"
  }
}
```

**执行流程和输出（实时）：**

```
2024-03-15 09:29:45 [INFO] Starting paper trading session...
2024-03-15 09:29:50 [INFO] Connected to Interactive Brokers (paper account)
2024-03-15 09:29:55 [INFO] Loading 50 days of historical data for warm-up...
2024-03-15 09:30:12 [INFO] Warm-up complete. Portfolio initialized.

2024-03-15 09:30:15 [DEBUG] AAPL quote: bid=190.23 ask=190.24
2024-03-15 09:31:00 [DEBUG] AAPL quote: bid=190.25 ask=190.26

... （整个交易日，实时数据流入）...

2024-03-15 13:45:30 [INFO] BUY: SMA20 191.34 > SMA50 190.12
2024-03-15 13:45:32 [INFO] Order placed: BUY 500 AAPL @ market
2024-03-15 13:45:33 [INFO] Order filled: BUY 500 @ 191.27

... （继续交易）...

2024-03-15 16:00:00 [INFO] Market closed. Paper trading session ended.

=============== Daily Results ===============
Date: 2024-03-15
P&L: $450.23 (+0.45%)
Trades: 2
Connected Time: 6.5 hours
=======================================
```

**特点：**
- 实时运行：按真实时间运行（1 小时需要 1 小时）
- 真实数据：使用经纪商的实时行情
- 模拟执行：订单不真正发送
- 无风险：即使有亏损也不影响真实账户
- 持续监控：需要定期检查连接状态

### 9.4 实盘模式执行

**配置文件 (config-live.json):**

```json
{
  "environment": "live-interactive-brokers",
  "data-folder": "./Data",
  "live-mode": true,

  "algorithm-type-name": "UnifiedAlgorithm",
  "algorithm-language": "CSharp",

  "composer": {
    "data-feed": "LiveTradingDataFeed",
    "transaction-handler": "BrokerageTransactionHandler",
    "result-handler": "LiveTradingResultHandler",
    "setup-handler": "BrokerageSetupHandler",
    "real-time-handler": "LiveTradingRealTimeHandler",
    "brokerage": "InteractiveBrokersBrokerage"
  },

  "interactive-brokers": {
    "host": "127.0.0.1",
    "port": 7496,
    "account": "LIVE_ACCOUNT"
  },

  "reality-modeler": {
    "slippage": 0.001,
    "commissions": 0.0005,
    "emergency-liquidation-threshold": -0.10
  }
}
```

**执行流程和输出（实时，有真实金钱）：**

```
2024-03-15 09:25:00 [WARN] LIVE TRADING MODE - REAL MONEY AT RISK
2024-03-15 09:25:05 [INFO] Connecting to Interactive Brokers account: LIVE_ACCOUNT
2024-03-15 09:25:10 [INFO] Connected. Account value: $100,000
2024-03-15 09:25:15 [INFO] Loading existing positions...
2024-03-15 09:25:20 [INFO] Existing holdings: none
2024-03-15 09:25:25 [INFO] Loading 50 days of historical data for warm-up...
2024-03-15 09:29:45 [INFO] Warm-up complete. Ready to trade.

2024-03-15 09:30:15 [DEBUG] AAPL quote: bid=190.23 ask=190.24
2024-03-15 09:31:00 [DEBUG] AAPL quote: bid=190.25 ask=190.26

... （监听实时数据）...

2024-03-15 13:45:30 [INFO] BUY: SMA20 191.34 > SMA50 190.12
2024-03-15 13:45:32 [WARN] ORDER SENT TO BROKER: BUY 500 AAPL @ market
2024-03-15 13:45:33 [INFO] Order confirmed by broker: Order ID #123456
2024-03-15 13:45:34 [INFO] Order filled: BUY 500 @ 191.28 (with 0.1% slippage)
2024-03-15 13:45:34 [INFO] Commission: $5.00. Net entry price: 191.33

2024-03-15 13:45:35 [INFO] Portfolio update:
                Position: 500 AAPL @ 191.33
                Unrealized P&L: -$25.00 (-0.05%)
                Cash: $4,433.50

... （继续监听）...

2024-03-15 14:20:00 [DEBUG] Heartbeat - Connected. Position: 500 AAPL. Market price: 191.45

2024-03-15 15:30:45 [WARN] ALERT: Unrealized loss exceeded 5% threshold!
2024-03-15 15:30:45 [WARN] Current position loss: -5.2%

2024-03-15 16:00:00 [INFO] SELL: SMA20 188.56 < SMA50 189.87
2024-03-15 16:00:02 [WARN] ORDER SENT TO BROKER: SELL 500 AAPL @ market
2024-03-15 16:00:03 [INFO] Order confirmed by broker: Order ID #123457
2024-03-15 16:00:04 [INFO] Order filled: SELL 500 @ 188.55 (with 0.1% slippage)
2024-03-15 16:00:04 [INFO] Commission: $5.00. Net exit price: 188.50

2024-03-15 16:00:05 [INFO] Trade completed:
                Entry: 191.33 (500 shares)
                Exit: 188.50
                P&L: -$1,415.00 (-1.41%)
                Time held: 2:15:00

2024-03-15 16:00:30 [INFO] Market closed. All positions: 0. Cash: $98,585.00

=============== Daily Report ===============
Date: 2024-03-15
P&L: -$1,415.00 (-1.41%)
Trades: 1
Portfolio Value: $98,585.00
Connected: Yes
=======================================
```

**特点：**
- 真实金钱：真实账户中的资金在变化
- 真实执行：订单实际发送给经纪商
- 真实成本：佣金、滑点等真实扣除
- 异常监控：需要监控连接、亏损、异常成交等
- 持续关注：交易中任何时刻可能出现意外情况
- 可追溯性：每笔交易都有经纪商的确认

---

## 10. 架构启示：我们能学到什么

LEAN 的设计为后端系统架构提供了深刻启示，前端工程师学习这些原则对全栈开发非常有帮助。

### 10.1 接口隔离原则（Interface Segregation Principle）

LEAN 没有创建一个巨大的 `IEngine` 接口，而是将其分解为 7 个专注的接口：

```csharp
// 不好的设计
public interface IEngine
{
    BaseDataCollection GetData();
    void PlaceOrder(Order order);
    void CancelOrder(int orderId);
    void LogResult(Result result);
    void Initialize();
    void ScheduleEvent(ScheduledEvent @event);
    IBrokerage GetBrokerage();
}

// 好的设计（LEAN）
public interface IDataFeed { ... }
public interface ITransactionHandler { ... }
public interface IResultHandler { ... }
public interface ISetupHandler { ... }
public interface IRealTimeHandler { ... }
public interface IBrokerage { ... }
```

**为前端开发者的启示：**

```typescript
// 不好的设计
interface DataLayer {
  fetchFromAPI(url: string): Promise<any>;
  validateData(data: any): boolean;
  transformData(data: any): any;
  cacheData(key: string, data: any): void;
  getCachedData(key: string): any;
  preloadData(urls: string[]): Promise<void>;
}

// 好的设计
interface DataFetcher {
  fetch(url: string): Promise<any>;
}

interface DataValidator {
  validate(data: any): boolean;
}

interface DataTransformer {
  transform(data: any): any;
}

interface DataCache {
  set(key: string, data: any): void;
  get(key: string): any;
}

interface DataPreloader {
  preload(urls: string[]): Promise<void>;
}
```

**好处：**
- 单一职责：每个接口只负责一件事
- 易于测试：可以独立 mock 每个接口
- 易于扩展：添加新功能不需要修改现有接口
- 易于组合：灵活组合接口实现不同的功能组合

### 10.2 为可测试性而设计

LEAN 的架构的根本目的就是**使策略代码能在不同环境中测试**：

```
策略代码（测试对象）
    ↓
依赖注入（隔离测试环境）
    ↓
回测环境（快速验证）→ 模拟盘（真实条件）→ 实盘（真实金钱）
```

**前端应用中的类比：**

```typescript
// 组件代码（测试对象）
export function UserProfile({ userId }: { userId: string }) {
  const { user, loading, error } = useUser(userId);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return <div>{user.name}</div>;
}

// 不同的依赖注入配置
// 单元测试：mock useUser hook
// 集成测试：连接真实 API
// E2E 测试：通过浏览器完整流程
```

关键思想：**通过依赖注入，同一个组件可以在多种环境中运行和测试**。

### 10.3 依赖注入的威力

LEAN 的 Composer 类展示了 DI 的强大：

```csharp
// 配置驱动
var handlers = Composer.CreateHandlers(config, algorithm);

// 无需改代码，只需改配置
// config.json: "brokerage": "BacktestingBrokerage"  → 回测
// config.json: "brokerage": "InteractiveBrokersBrokerage"  → 实盘
```

**前端框架中的 DI 例子：**

```typescript
// Angular DI
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}
}

// 在单元测试中注入 mock HttpClient
const mockHttp = jasmine.createSpyObj('HttpClient', ['get']);
const service = new UserService(mockHttp);

// 在生产中注入真实 HttpClient
```

```jsx
// React Context DI
const DataContext = createContext<DataContextType>(null);

export function App() {
  return (
    <DataContext.Provider value={realDataService}>
      <Dashboard />
    </DataContext.Provider>
  );
}

// 在测试中
export function TestApp() {
  return (
    <DataContext.Provider value={mockDataService}>
      <Dashboard />
    </DataContext.Provider>
  );
}
```

### 10.4 配置驱动架构

LEAN 使用 config.json 驱动整个系统的行为：

```json
{
  "environment": "backtesting",  // 改一个值，整个系统变模式
  "composer": { ... }
}
```

**前端应用中的配置驱动：**

```typescript
// src/config.ts
export const config = {
  env: process.env.NODE_ENV,

  api: {
    baseUrl: process.env.API_BASE_URL || 'http://localhost:3000',
    timeout: 30000,
  },

  feature: {
    betaFeatures: process.env.BETA_FEATURES === 'true',
    darkMode: true,
  },

  analytics: {
    enabled: process.env.ANALYTICS_ENABLED !== 'false',
    provider: process.env.ANALYTICS_PROVIDER || 'segment',
  },
};

// 使用配置驱动不同的行为
if (config.env === 'production') {
  enableErrorTracking();
  disableDebugLogging();
}

if (config.feature.betaFeatures) {
  renderBetaUI();
}
```

**好处：**
- 不需要代码改动就能改变行为
- 环境差异完全由配置管理
- 可以通过环境变量快速切换

### 10.5 对比：回测 ↔ SQLite，实盘 ↔ PostgreSQL

如果你曾经写过支持多个数据库的 ORM 代码，就能理解 LEAN 的设计：

```typescript
// 定义数据库接口
interface Database {
  query(sql: string): Promise<any>;
  exec(sql: string): Promise<void>;
}

// 回测 ↔ SQLite（内存/本地）
class SQLiteDatabase implements Database {
  private db: Database;

  async query(sql: string) {
    return this.db.all(sql);
  }
}

// 实盘 ↔ PostgreSQL（远程/生产）
class PostgresDatabase implements Database {
  private pool: Pool;

  async query(sql: string) {
    const result = await this.pool.query(sql);
    return result.rows;
  }
}

// 应用代码（不知道用的哪个数据库）
class UserRepository {
  constructor(private db: Database) {}

  async getUser(id: string) {
    return this.db.query(`SELECT * FROM users WHERE id = '${id}'`);
  }
}

// 依赖注入
const db = process.env.NODE_ENV === 'test'
  ? new SQLiteDatabase()
  : new PostgresDatabase();

const userRepo = new UserRepository(db);
```

LEAN 的设计完全类似，只不过接口是交易相关的。

### 10.6 设计的通用原则

总结 LEAN 架构的核心设计原则：

1. **接口隔离**：将大接口分解为小、专注的接口
2. **依赖注入**：不在代码中硬编码依赖，而是注入
3. **配置驱动**：通过配置改变行为，不修改代码
4. **分层架构**：算法层与基础设施层分离
5. **可测试性**：设计使得在多种环境中能够测试
6. **可扩展性**：添加新环境或功能不需要修改现有代码

这些原则对任何大型系统都适用：Web 应用、移动应用、数据管道等。

---

## 总结

LEAN 引擎的回测和实盘统一抽象设计展示了软件工程的最佳实践：

| 方面 | 说明 |
|------|------|
| **优雅之处** | 同一份代码在完全不同的环境中运行 |
| **核心机制** | 7 个关键接口 + 依赖注入 + 配置驱动 |
| **可测试性** | 从快速回测到模拟盘再到实盘的完整流程 |
| **风险管理** | 通过模拟盘在真实条件下验证策略 |
| **架构启示** | 接口隔离、DI、配置驱动等通用原则 |

无论你是量化工程师还是全栈工程师，理解这个设计都能提升你的系统设计能力。

---

[上一篇: 订单执行与真实性建模](./05-order-execution.md) | [下一篇: 开源生态与插件架构](./07-open-source-ecosystem.md)
