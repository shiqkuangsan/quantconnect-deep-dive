# LEAN 数据管线与订阅系统深度指南

> 数据是量化系统的血液。本文详细解析 LEAN 引擎如何高效地摄取、处理、同步和分发市场数据。

**导航**: [上一篇: LEAN 引擎架构总览](./02-lean-architecture-overview.md) | [下一篇: Algorithm Framework 五层流水线](./04-algorithm-framework.md)

---

## 目录

1. [数据管线概述](#数据管线概述)
2. [数据分辨率层级](#数据分辨率层级)
3. [Subscription 生命周期](#subscription-生命周期)
4. [Enumerator 栈](#enumerator-栈数据读取管道)
5. [TimeSlice 同步机制](#timeslice-同步机制)
6. [回测模式的数据流](#回测模式的数据流)
7. [实盘模式的数据流](#实盘模式的数据流)
8. [自定义数据源](#自定义数据源)
9. [Universe Selection 与动态订阅](#universe-selection-与动态订阅)
10. [性能考量与优化](#性能考量与优化)

---

## 数据管线概述

### 核心理念

LEAN 的数据管线遵循一个优雅的函数式流处理模型。数据从多个源头（行情商、交易所、数据提供商）进入系统，经过层层转换和过滤，最终以规范化的格式呈现给策略算法。

这个设计的妙处在于：
- **分离关注点**：数据获取、解析、聚合、同步各自独立
- **可组合性**：通过 Enumerator 链可以灵活组合各种处理逻辑
- **可扩展性**：支持自定义数据源和数据类型无缝接入

### 完整数据流管线

```
┌─────────────────────────────────────────────────────────────────┐
│                      数据管线全景                                │
└─────────────────────────────────────────────────────────────────┘

外部数据源层
├─ 交易所行情 (Exchange Feeds)
│  ├─ 实时 Tick 数据
│  ├─ L1/L2 行情
│  └─ 成交数据
├─ 数据供应商 (Data Providers)
│  ├─ Bloomberg, Reuters
│  ├─ QuantConnect 历史数据库
│  └─ 第三方 API (加密货币、期货等)
└─ 企业数据 (Alternative Data)
   ├─ 财报数据
   ├─ 情绪数据
   └─ 卫星图像数据
         ↓
    DataProvider 抽象层
    (为不同数据源提供统一接口)
         ↓
┌─────────────────────────────────────────────────────────────────┐
│          SubscriptionDataReader (订阅数据读取器)               │
│  - 根据 Subscription 配置读取源数据                            │
│  - 处理数据格式解析 (ZIP, CSV, JSON 等)                       │
│  - 初始化 Enumerator 链                                        │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│          Enumerator Stack (转换管道)                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ RateLimiter(限流)                                        │  │
│  │ ↓                                                        │  │
│  │ FillForward(前向填充缺失数据)                            │  │
│  │ ↓                                                        │  │
│  │ Filter(应用交易所时间、退市等)                          │  │
│  │ ↓                                                        │  │
│  │ AggregationEnumerator(tick → bar 聚合)                  │  │
│  │ ↓                                                        │  │
│  │ SynchronizingEnumerator(跨品种、跨分辨率同步)           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  每个 Subscription 有自己的 Enumerator 链                      │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│          Synchronizer (全局同步器)                              │
│                                                                 │
│  维护优先级队列 (堆):                                          │
│  - 所有 Enumerator 输出混合                                    │
│  - 按时间戳排序                                                │
│  - 回测: 预读数据，按时间顺序驱动                             │
│  - 实盘: 等待数据到达，时间窗口内聚合                        │
│                                                                 │
│  输出: TimeSlice 序列                                          │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│          TimeSlice (时间切片)                                   │
│                                                                 │
│  包含在单一时间点上所有品种的数据快照:                         │
│  - Slice: 规范化数据 (Bars, Ticks, Dividends 等)              │
│  - SecurityChanges: 新增/删除的品种                           │
│  - DataFeedPacket: 底层数据包                                 │
│                                                                 │
│  时间戳: 该 TimeSlice 的有效时刻                              │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│          Algorithm.OnData(Slice) (算法接收层)                   │
│                                                                 │
│  策略代码在此处理数据:                                          │
│  - 访问 Slice.Bars[Symbol]                                    │
│  - 访问 Slice.QuoteBars[Symbol]                               │
│  - 访问 Slice.Ticks[Symbol]                                   │
│  - 检查 SecurityChanges.AddedSecurities                       │
│  - 更新指标、评估信号、下单                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 关键组件关系

| 组件 | 职责 | 类比(前端) |
|------|------|----------|
| **DataProvider** | 从源头获取原始数据 | API 客户端 |
| **SubscriptionDataReader** | 解析格式、初始化读取 | Response 拦截器 |
| **Enumerator Stack** | 数据转换管道 | RxJS pipe chain |
| **Synchronizer** | 跨品种/分辨率同步 | Promise.all() 聚合 |
| **TimeSlice** | 时间点快照 | Redux action payload |
| **Slice** | 用户可见数据 | 规范化的状态对象 |

---

## 数据分辨率层级

### 分辨率体系

LEAN 支持 6 个数据分辨率等级，从最细粒度到最粗粒度：

```
┌──────────────┬──────────────┬─────────────┬───────────────┐
│  分辨率      │  数据点频率  │  数据量级   │  典型场景     │
├──────────────┼──────────────┼─────────────┼───────────────┤
│ Tick         │ 每次成交     │ 超大 (GB)   │ 市场微观结构  │
│ Second       │ 每秒         │ 很大 (MB)   │ 高频策略      │
│ Minute       │ 每分钟       │ 中等 (KB)   │ 日内交易      │
│ Hour         │ 每小时       │ 小 (B)      │ 中期交易      │
│ Daily        │ 每天         │ 极小 (B)    │ 长期投资      │
│ NotApplicable│ 无固定周期   │ 任意        │ 企业数据      │
└──────────────┴──────────────┴─────────────┴───────────────┘
```

### 分辨率细节与数据类型

#### 1. Tick 分辨率

**含义**: 市场上每笔成交都是一个 Tick 数据点。

```python
# 策略代码示例
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Tick)

def OnData(self, slice):
    # 获取所有 AAPL tick 数据
    for tick in slice.Ticks["AAPL"]:
        print(f"Tick at {tick.Time}: Price={tick.LastPrice}, Size={tick.Quantity}")
        # LastPrice: 成交价
        # Quantity: 成交量
        # Bid/Ask: 报价
```

**物理数据格式** (`/data/equity/usa/tick/aapl/20210101_trade.zip`):
```
20210101_trade.zip
├─ 20210101_aapl_trade.csv
│  └─ 列: timestamp, price, size, bid, ask
└─ 类似文件用于不同日期
```

**Tick 数据类型**:
- `TradeBar` (成交): 价格、成交量、成交时间
- `QuoteBar` (报价): Bid/Ask 价格、大小、报价时间

#### 2. Second 分辨率

**含义**: 按秒聚合数据（OHLCV）。

```python
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Second)

def OnData(self, slice):
    bar = slice.Bars["AAPL"]  # 秒 bar
    # Open, High, Low, Close, Volume
```

**数据存储** (`/data/equity/usa/second/aapl/`):
```
20210101_aapl.csv
Time,O,H,L,C,V
09:30:00,130.45,130.50,130.40,130.48,50000
09:30:01,130.48,130.55,130.46,130.52,45000
...
```

#### 3. Minute 分辨率

**含义**: 按分钟聚合（最常用）。

```python
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Minute)
    self.AddEquity("BTC/USD", Resolution.Minute, Market.BINANCE)

def OnData(self, slice):
    # 在每个分钟边界触发
    bar = slice.Bars["AAPL"]  # 分钟 bar
    print(f"{bar.Time}: OHLCV = {bar.Open}/{bar.High}/{bar.Low}/{bar.Close}/{bar.Volume}")
```

**存储结构** (`/data/equity/usa/minute/aapl/20210101_trade.zip`):
```
ZIP 压缩包
├─ 20210101_aapl.csv  (该天所有分钟数据)
   │  Time,O,H,L,C,V
   │  2021-01-01 09:30:00,130.15,130.50,130.10,130.45,5000000
   │  2021-01-01 09:31:00,130.45,130.70,130.40,130.65,4800000
   │  ...
```

**优势**:
- 数据量可管理（1 股票 × 250 交易日 × 390 分钟 ≈ 97,500 条记录）
- 足够捕捉日内行为
- 适合大多数日内交易策略

#### 4. Hour 分辨率

**含义**: 按小时聚合。

```python
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Hour)

def OnData(self, slice):
    # 在每个小时结束时触发
    bar = slice.Bars["AAPL"]  # 小时 bar
```

**应用**: 交易日有 6.5 个小时，1 股票 × 250 交易日 × 6.5 小时 ≈ 1,625 条记录。适合中期策略。

#### 5. Daily 分辨率

**含义**: 日线数据。

```python
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Daily)

def OnData(self, slice):
    # 每个交易日调用一次
    bar = slice.Bars["AAPL"]
```

**存储** (`/data/equity/usa/daily/aapl.zip`):
```
aapl.csv
Time,O,H,L,C,V
2021-01-04,130.02,131.41,129.89,131.01,107414600
2021-01-05,131.04,132.29,131.03,131.92,106575400
...
```

**特点**: 文件极小，整个股票历史可能只有几 KB。

#### 6. 分辨率自动聚合

**关键原理**: 你可以订阅多个分辨率，低分辨率数据自动从更高分辨率生成。

```python
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Tick)    # 订阅 Tick
    # LEAN 自动生成:
    # - 每个 Second bar 从 Tick 聚合
    # - 每个 Minute bar 从 Second 聚合
    # - 每个 Hour bar 从 Minute 聚合
    # - 每个 Daily bar 从 Hour 聚合

def OnData(self, slice):
    # 同一个 OnData() 中可能有多个分辨率
    if slice.ContainsKey("AAPL"):
        tick_data = slice.Ticks.get("AAPL", [])
        bar_minute = slice.Bars.get("AAPL", None)
        # 需要自行区分
```

**聚合层级**:
```
Tick → (每 1 秒) → SecondBar
SecondBar → (每 60 秒) → MinuteBar
MinuteBar → (每 60 分钟) → HourBar
HourBar → (每交易日结束) → DailyBar
```

**内存中的聚合实现**:

```csharp
// 简化的 LEAN 聚合逻辑
public class AggregationEnumerator
{
    private List<Tick> _tickBuffer = new();
    private DateTime _barStart;

    public IEnumerable<TradeBar> Aggregate(IEnumerable<Tick> ticks, Resolution resolution)
    {
        foreach (var tick in ticks)
        {
            _tickBuffer.Add(tick);

            // 检查是否应该闭合 bar
            if (ShouldCloseBar(tick.Time, resolution))
            {
                yield return BuildBar(_tickBuffer, resolution);
                _tickBuffer.Clear();
                _barStart = GetBarStart(tick.Time, resolution);
            }
        }
    }

    private TradeBar BuildBar(List<Tick> ticks, Resolution resolution)
    {
        return new TradeBar
        {
            Open = ticks[0].LastPrice,
            High = ticks.Max(t => t.LastPrice),
            Low = ticks.Min(t => t.LastPrice),
            Close = ticks.Last().LastPrice,
            Volume = ticks.Sum(t => t.Quantity),
            Time = _barStart,
            Period = GetPeriod(resolution)
        };
    }
}
```

### 数据文件物理结构

LEAN 数据库使用分层目录结构和 ZIP 压缩：

```
/data/
├─ equity/
│  ├─ usa/
│  │  ├─ tick/
│  │  │  ├─ aapl/
│  │  │  │  ├─ 20210101_trade.zip
│  │  │  │  ├─ 20210102_trade.zip
│  │  │  │  └─ ...
│  │  │  ├─ msft/
│  │  │  └─ ...
│  │  ├─ second/
│  │  │  ├─ aapl/
│  │  │  │  ├─ 20210101_aapl.csv
│  │  │  │  └─ ...
│  │  │  └─ ...
│  │  ├─ minute/
│  │  │  ├─ aapl/
│  │  │  │  ├─ 20210101_trade.zip
│  │  │  │  │  └─ 20210101_aapl.csv
│  │  │  │  ├─ 20210102_trade.zip
│  │  │  │  └─ ...
│  │  │  └─ ...
│  │  ├─ hour/
│  │  │  ├─ aapl/
│  │  │  │  └─ aapl.csv  (所有日期, 未压缩)
│  │  │  └─ ...
│  │  └─ daily/
│  │     ├─ aapl.zip
│  │     └─ ...
│  ├─ cfd/
│  └─ ...
├─ crypto/
│  ├─ binance/
│  │  ├─ minute/
│  │  │  ├─ btcusd/
│  │  │  └─ ...
│  │  └─ ...
│  └─ ...
├─ forex/
│  ├─ fxcm/
│  │  ├─ minute/
│  │  └─ ...
│  └─ ...
├─ futures/
│  ├─ cme/
│  │  ├─ minute/
│  │  └─ ...
│  └─ ...
└─ fundamental/
   ├─ financials/
   │  └─ ...
   └─ ...
```

**为何使用 ZIP？**
1. **压缩率高**: 行情数据重复性强，压缩率通常 90%+
2. **随机访问**: ZIP 支持按文件检索，不需解压整个归档
3. **日期分割**: 按日期打包便于快速定位历史数据
4. **带宽节省**: 数据库到本地网络传输快

**CSV 格式示例** (Minute bar):
```
Time,O,H,L,C,V
2021-01-04 09:30:00,130.15,130.50,130.10,130.45,5100000
2021-01-04 09:31:00,130.45,130.70,130.40,130.65,4800000
2021-01-04 09:32:00,130.65,130.80,130.60,130.75,3900000
```

---

## Subscription 生命周期

### Subscription 是什么？

一个 `Subscription` 对象是告诉 LEAN 引擎："我需要这个品种的这种类型的数据，从现在开始"。

```csharp
public class Subscription
{
    public Symbol Symbol { get; set; }              // 品种符号
    public Resolution Resolution { get; set; }      // 分辨率
    public Type DataType { get; set; }              // 数据类型 (TradeBar, Tick, etc)
    public TickType TickType { get; set; }          // Tick 类型 (Trade, Quote, OpenInterest)
    public SubscriptionConfiguration Configuration { get; set; }  // 配置
    public IEnumerator<BaseData> Enumerator { get; set; }         // 数据读取器
    public DateTime StartTime { get; set; }         // 订阅开始时间
    public DateTime EndTime { get; set; }           // 订阅结束时间
    public bool IsRemoved { get; set; }             // 是否已移除
}
```

### 创建 Subscription 的过程

#### Step 1: 调用 AddEquity()

```python
def Initialize(self):
    # Python API
    self.AddEquity("AAPL", Resolution.Minute)
```

后台发生：
```csharp
// C# 实现（简化）
public Subscription AddEquity(string ticker, Resolution resolution)
{
    var symbol = Symbol.Create(ticker, SecurityType.Equity, Market.USA);

    var config = new SubscriptionDataConfig(
        dataType: typeof(TradeBar),
        symbol: symbol,
        resolution: resolution,
        timeZone: TimeZones.NewYork,
        dataTimeZone: TimeZones.NewYork,
        isCustomData: false,
        fillForwardData: true,     // 默认启用 fill forward
        extendedMarketHours: false, // 不包含盘前盘后
        isFilteredSubscriptionOnly: false
    );

    return AddSubscription(config);
}

private Subscription AddSubscription(SubscriptionDataConfig config)
{
    // 1. 创建 Subscription 对象
    var subscription = new Subscription(config);

    // 2. 根据运行模式创建 DataReader
    subscription.Enumerator = CreateEnumerator(config);

    // 3. 注册到 SubscriptionManager
    SubscriptionManager.Add(subscription);

    return subscription;
}
```

#### Step 2: SubscriptionManager 管理

`SubscriptionManager` 是所有活跃订阅的中央注册表：

```csharp
public class SubscriptionManager
{
    private readonly Dictionary<Symbol, List<Subscription>> _subscriptions = new();

    public void Add(Subscription subscription)
    {
        var symbol = subscription.Symbol;
        if (!_subscriptions.ContainsKey(symbol))
            _subscriptions[symbol] = new List<Subscription>();

        _subscriptions[symbol].Add(subscription);

        // 触发事件，用于 SecurityManager 初始化新品种
        OnSubscriptionAdded?.Invoke(subscription);
    }

    public List<Subscription> GetBySymbol(Symbol symbol)
        => _subscriptions[symbol];

    public List<Subscription> GetAll()
        => _subscriptions.Values.SelectMany(x => x).ToList();
}
```

**时间线**:
```
Initialize() 执行
    ↓
AddEquity("AAPL", Resolution.Minute)
    ↓
SubscriptionDataConfig 创建
    ↓
Enumerator 链初始化
    ↓
SubscriptionManager.Add()
    ↓
SecurityManager 创建 Security 对象
    ↓
开始数据驱动 (OnData 调用开始)
```

### Subscription 配置详解

```csharp
public class SubscriptionDataConfig
{
    // 数据来源
    public Type DataType { get; set; }              // TradeBar, Tick, 自定义数据
    public Symbol Symbol { get; set; }              // AAPL, BTC/USD 等
    public Resolution Resolution { get; set; }      // Minute, Daily 等
    public TickType TickType { get; set; }          // Trade, Quote, OpenInterest

    // 时间配置
    public string TimeZone { get; set; }            // 数据所在时区
    public string DataTimeZone { get; set; }        // 数据源时区
    public DateTime StartTime { get; set; }         // 订阅开始
    public DateTime EndTime { get; set; }           // 订阅结束

    // 处理方式
    public bool FillForwardData { get; set; }       // 缺失数据是否前向填充
    public bool ExtendedMarketHours { get; set; }   // 是否包含盘前盘后
    public bool IsCustomData { get; set; }          // 是否自定义数据
    public decimal PriceScaleFactor { get; set; }   // 价格缩放因子 (分拆处理)

    // 特殊属性
    public Dictionary<string, object> Properties { get; set; }  // 额外配置
}
```

**常见配置场景**:

```python
# 场景 1: 标准美股分钟数据（最常见）
config1 = self.AddEquity("AAPL", Resolution.Minute)
# DataType=TradeBar, FillForward=True, ExtendedHours=False

# 场景 2: Tick 数据，不填充缺失
config2 = self.AddEquity("AAPL", Resolution.Tick)
# DataType=Tick, FillForward=False (Tick 不填充)

# 场景 3: 盘前盘后数据
config3 = self.AddEquity("AAPL", Resolution.Minute)
config3.ExtendedMarketHours = True

# 场景 4: 自定义数据源
class MyCustomData(PythonData):
    def GetSource(self, config, date, isLiveMode):
        return SubscriptionDataSource(url, SubscriptionTransportMedium.RemoteFile)
    def Reader(self, config, line, date, isLiveMode):
        # 解析自定义格式
        return MyCustomData()

config4 = self.AddData(MyCustomData, "my-symbol", Resolution.Daily)
```

### 动态订阅示例

```python
class DynamicUniverseAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 静态订阅
        self.AddEquity("SPY", Resolution.Minute)

        # 动态订阅 (Universe Selection)
        self.AddUniverse(self.SelectCoarseUniverse)

    def SelectCoarseUniverse(self, coarse):
        """每天调用，返回要交易的符号列表"""
        # 过滤: 日成交金额 > 1000 万，价格 > $5
        selected = [c for c in coarse
                   if c.DollarVolume > 10000000 and c.Price > 5][:100]
        return [x.Symbol for x in selected]

    def OnData(self, slice):
        """
        - SPY 每分钟都有数据
        - Universe 中的品种数据每天变化
        - 新增品种自动创建 Subscription
        - 移除的品种自动销毁 Subscription
        """
        if not self.Portfolio.Invested:
            for symbol in self.Universe.Members:
                self.SetHoldings(symbol, 0.01)  # 平均权重
```

**动态 Subscription 流程**:
```
SelectCoarseUniverse() 返回 [AAPL, MSFT, GOOGL]
    ↓
与前一天对比
    ↓
新增: AAPL, MSFT, GOOGL → 创建 3 个新 Subscription
删除: (前一天的品种)     → 销毁对应 Subscription
    ↓
对每个新品种加载历史数据 (Warmup)
    ↓
OnData() 收到完整的 SecurityChanges
    ↓
strategy_code.OnSecuritiesChanged() 触发
```

---

## Enumerator 栈（数据读取管道）

### 核心设计思想

LEAN 的数据管线采用 **Decorator Pattern** (装饰模式) 和 **Chain of Responsibility** (职责链模式)。每个 Enumerator 负责一项具体的转换任务，通过链式组合实现复杂的数据处理流程。

这是函数式编程在 C# 中的优雅运用：每个 Enumerator 都是一个 `IEnumerator<BaseData>`，通过 C# yield 机制实现惰性求值。

```
对标前端: RxJS 的 pipe() 组合

// 前端 RxJS
source$.pipe(
    filter(x => x.price > 100),
    debounceTime(100),
    map(x => x.symbol),
    switchMap(symbol => fetchData(symbol))
).subscribe(data => console.log(data));

// LEAN Enumerator 栈
raw_data
    |> filter (exchange_hours)
    |> debounce (fill_forward)
    |> aggregate (tick_to_bar)
    |> synchronize (multiple_symbols)
    => algorithm.OnData()
```

### 标准 Enumerator 链

一个典型的 Subscription 的 Enumerator 链如下：

```
┌──────────────────────────────────────────────────────────────┐
│           Enumerator Stack 示例 (Minute Resolution)         │
└──────────────────────────────────────────────────────────────┘

1. SourceEnumerator
   └─ 从磁盘/网络读取原始数据
      yields: Tick, TradeBar (未聚合/未过滤)

2. RateLimitEnumerator
   └─ 限流 (可选)
      实现: 每秒最多 X 条数据
      yields: 与输入相同，但按速率限制

3. AggregationEnumerator (条件性)
   └─ 如果源数据是更高分辨率 (Tick/Second)
   └─ 聚合到目标分辨率 (Minute)
      输入: Tick
      输出: TradeBar (聚合后)
      实现: 缓冲 tick → 分钟边界关闭 bar

4. FillForwardEnumerator
   └─ 向前填充缺失的 bar
      输入: TradeBar (可能有缺口)
      输出: TradeBar (缺口填充)
      实现:
         - 检查时间差
         - 如果缺口 > 1 分钟且 FillForward=true
         - 用前一 bar 的 close 价格创建空 bar

         例: 14:00 有数据，14:02 才出现下一条
         则自动创建 14:01 bar (open=14:00_close, close=14:00_close)

5. FilterEnumerator
   └─ 过滤不交易的时段
      输入: TradeBar
      输出: TradeBar (仅交易时段)
      实现:
         - 检查是否在交易所营业时间内
         - 检查品种是否已退市
         - 过滤盘前盘后 (如配置)
         yields: 只有有效的 bar

6. SynchronizingEnumerator
   └─ 跨 Subscription 同步
      多个 Enumerator 的输出混合
      按时间戳排序
      => 输出给 Synchronizer (全局层)
```

### 关键 Enumerator 详解

#### 1. RateLimitEnumerator (限流)

**目的**: 防止数据流淹没系统，特别是 Tick 数据。

```csharp
public class RateLimitEnumerator : IEnumerator<BaseData>
{
    private readonly IEnumerator<BaseData> _source;
    private readonly int _rateLimit;  // 每秒最多条数
    private DateTime _lastEmissionTime;
    private Queue<BaseData> _buffer = new();

    public RateLimitEnumerator(IEnumerator<BaseData> source, int rateLimit)
    {
        _source = source;
        _rateLimit = rateLimit;
    }

    public bool MoveNext()
    {
        while (_source.MoveNext())
        {
            var data = _source.Current;

            // 简化: 如果距离上次发射不足 1/rateLimit 秒，缓冲
            var timeSinceLastEmission = (data.Time - _lastEmissionTime).TotalMilliseconds;
            var minInterval = 1000.0 / _rateLimit;

            if (timeSinceLastEmission < minInterval)
            {
                _buffer.Enqueue(data);
            }
            else
            {
                Current = data;
                _lastEmissionTime = data.Time;
                return true;
            }
        }
        return false;
    }

    public BaseData Current { get; private set; }
}
```

**应用场景**:
```python
# 如果你有实时 Tick 数据流，每秒 10,000 ticks
# 限流到每秒 1,000 条，防止 OnData 调用过于频繁
config.RateLimit = 1000
```

#### 2. FillForwardEnumerator (前向填充)

这是最重要的 Enumerator。它解决市场中普遍存在的问题：**不是所有时间都有成交**。

**问题场景**:
```
股票 AAPL Minute bar 数据:
14:00 成交
14:01 无成交
14:02 有成交 (bar: O=130.00, H=130.20, L=129.90, C=130.10)
14:03 无成交
14:04 有成交 (bar: O=130.10, H=130.50, L=130.00, C=130.45)

问题: 14:01 和 14:03 的数据在哪里？
```

**解决方案: Fill Forward**

```csharp
public class FillForwardEnumerator : IEnumerator<BaseData>
{
    private readonly IEnumerator<BaseData> _source;
    private readonly Resolution _resolution;
    private BaseData _lastData;
    private DateTime _lastEmissionTime;

    public bool MoveNext()
    {
        if (_source.MoveNext())
        {
            var data = _source.Current;

            // 计算应有的下一个时间点
            var expectedNextTime = GetNextExpectedTime(_lastEmissionTime, _resolution);

            // 有缺口: 在缺口中创建 fill-forward bars
            while (expectedNextTime < data.Time)
            {
                var fillBar = _lastData.Clone();
                fillBar.Time = expectedNextTime;

                yield return fillBar;

                expectedNextTime = GetNextExpectedTime(expectedNextTime, _resolution);
            }

            _lastData = data;
            _lastEmissionTime = data.Time;
            yield return data;
        }
    }

    private DateTime GetNextExpectedTime(DateTime current, Resolution resolution)
    {
        return resolution switch
        {
            Resolution.Minute => current.AddMinutes(1),
            Resolution.Hour => current.AddHours(1),
            Resolution.Daily => current.AddDays(1),
            _ => current
        };
    }
}
```

**效果对比**:

```
原始数据 (Source):
14:00, 130.00
14:02, 130.10
14:04, 130.45

FillForward=True:
14:00, 130.00       ← 原始
14:01, 130.00       ← 填充 (Close=130.00)
14:02, 130.10       ← 原始
14:03, 130.10       ← 填充 (Close=130.10)
14:04, 130.45       ← 原始

FillForward=False:
14:00, 130.00
14:02, 130.10
14:04, 130.45
```

**关键点**:
- Fill-forward bar 的 OHLC 都等于前一 bar 的 close
- Volume 为 0（表示非真实成交）
- 这使得指标计算连续有效（不会有缺失值）

#### 3. FilterEnumerator (过滤)

**目的**: 移除不交易的时段、已退市品种等。

```csharp
public class FilterEnumerator : IEnumerator<BaseData>
{
    private readonly IEnumerator<BaseData> _source;
    private readonly Security _security;  // 用于获取交易时间、退市状态等

    public bool MoveNext()
    {
        while (_source.MoveNext())
        {
            var data = _source.Current;

            // 检查 1: 是否在交易时间内
            if (!_security.IsOpen(data.Time))
                continue;

            // 检查 2: 品种是否已退市
            if (_security.IsDelisted(data.Time))
                continue;

            // 检查 3: 是否是扩展时间 (如不包含盘前盘后)
            if (!config.ExtendedMarketHours && !IsRegularHours(data.Time))
                continue;

            Current = data;
            return true;
        }
        return false;
    }

    public BaseData Current { get; private set; }
}
```

#### 4. AggregationEnumerator (聚合)

**目的**: 将高分辨率数据聚合到低分辨率。

```csharp
public class AggregationEnumerator : IEnumerator<BaseData>
{
    private readonly IEnumerator<BaseData> _source;  // 输入: Tick 或 Second
    private readonly Resolution _targetResolution;
    private List<Tick> _tickBuffer = new();
    private DateTime _barStartTime;
    private bool _barClosed = true;

    public bool MoveNext()
    {
        while (_source.MoveNext())
        {
            var tick = (Tick)_source.Current;

            // 初始化 bar
            if (_barClosed)
            {
                _barStartTime = GetBarStart(tick.Time, _targetResolution);
                _tickBuffer.Clear();
                _barClosed = false;
            }

            _tickBuffer.Add(tick);

            // 检查 bar 是否应该闭合
            var nextBarStart = GetNextBarStart(_barStartTime, _targetResolution);
            if (tick.Time >= nextBarStart)
            {
                Current = BuildBar(_tickBuffer, _barStartTime);
                _barClosed = true;
                return true;
            }
        }

        // 流结束前，发射最后一个 bar
        if (!_barClosed && _tickBuffer.Count > 0)
        {
            Current = BuildBar(_tickBuffer, _barStartTime);
            _barClosed = true;
            return true;
        }

        return false;
    }

    private TradeBar BuildBar(List<Tick> ticks, DateTime barTime)
    {
        return new TradeBar
        {
            Symbol = ticks[0].Symbol,
            Time = barTime,
            Open = ticks[0].LastPrice,
            High = ticks.Max(t => t.LastPrice),
            Low = ticks.Min(t => t.LastPrice),
            Close = ticks.Last().LastPrice,
            Volume = ticks.Sum(t => t.Quantity),
            Period = TimeSpan.FromMinutes(1)  // 假设目标是 Minute
        };
    }
}
```

### 构建 Enumerator 栈的工厂

LEAN 使用工厂模式构建完整的 Enumerator 栈：

```csharp
public class BaseDataSubscriptionEnumeratorFactory
{
    public IEnumerator<BaseData> CreateEnumerator(
        SubscriptionDataConfig config,
        DateTime startTime,
        DateTime endTime,
        bool isLiveMode)
    {
        // 1. 创建源 Enumerator (从磁盘或网络)
        IEnumerator<BaseData> source;
        if (isLiveMode)
            source = new LiveDataEnumerator(config);
        else
            source = new FileSystemDataEnumerator(config, startTime, endTime);

        // 2. 应用聚合 (如果需要)
        if (NeedsAggregation(config))
            source = new AggregationEnumerator(source, config.Resolution);

        // 3. 应用前向填充
        if (config.FillForwardData)
            source = new FillForwardEnumerator(source, config.Resolution);

        // 4. 应用过滤
        source = new FilterEnumerator(source, config, security);

        // 5. 应用限流 (可选)
        if (config.RateLimitPerSecond.HasValue)
            source = new RateLimitEnumerator(source, config.RateLimitPerSecond.Value);

        return source;
    }
}
```

**对标前端 RxJS**:

```javascript
// RxJS 中的类似构建
const stream$ = rawDataSource$.pipe(
    // 第 1 层: 映射 (对标 AggregationEnumerator)
    map(tick => aggregateToBar(tick)),

    // 第 2 层: 补全缺失值 (对标 FillForwardEnumerator)
    startWith(lastBar),
    concatMap(bar => {
        const nextTime = getNextBarTime(bar.time, resolution);
        if (bar.time < nextTime) {
            return of(bar, { ...bar, time: nextTime });
        }
        return of(bar);
    }),

    // 第 3 层: 过滤 (对标 FilterEnumerator)
    filter(bar => isTradeTime(bar.time)),

    // 第 4 层: 限流 (对标 RateLimitEnumerator)
    throttleTime(1000 / rateLimit),

    // 第 5 层: 订阅
    subscribe(bar => algorithm.OnData(bar))
);
```

---

## TimeSlice 同步机制

### 问题场景

想象你订阅了 3 个品种：AAPL, MSFT, GOOGL，分辨率都是 Minute。

```
AAPL 的 Enumerator 输出:
[14:00] AAPL bar
[14:01] AAPL bar
[14:02] AAPL bar
...

MSFT 的 Enumerator 输出:
[14:00] MSFT bar
[14:01] MSFT bar
[14:02] MSFT bar
...

GOOGL 的 Enumerator 输出:
[14:00] GOOGL bar
[14:01] GOOGL bar
[14:02] GOOGL bar
...

问题: OnData() 应该何时调用？
- 如果 AAPL 生成了 14:00 bar，就立即调用 OnData()?
  → 不行，因为此时 MSFT 和 GOOGL 的 14:00 bar 还没有准备好

- 是否应该等待所有品种都生成 14:00 bar？
  → 正确答案！
```

### TimeSlice 的设计

一个 `TimeSlice` 是特定时刻所有品种数据的快照：

```csharp
public class TimeSlice
{
    /// <summary>这个 TimeSlice 的有效时间</summary>
    public DateTime Time { get; set; }

    /// <summary>用户可见的数据对象</summary>
    public Slice Slice { get; set; }

    /// <summary>品种变化 (新增/删除)</summary>
    public SecurityChanges SecurityChanges { get; set; }

    /// <summary>底层数据包</summary>
    public List<DataFeedPacket> DataFeedPackets { get; set; }
}

public class Slice
{
    public DateTime Time { get; set; }

    // 各种数据容器
    public DataDictionary<TradeBar> Bars { get; set; }              // 价格 bar
    public DataDictionary<QuoteBar> QuoteBars { get; set; }         // 报价 bar
    public DataDictionary<List<Tick>> Ticks { get; set; }           // Tick 数据
    public DataDictionary<OpenInterest> OpenInterest { get; set; }  // 期权持仓
    public DataDictionary<OptionChain> OptionChains { get; set; }   // 期权链
    public DataDictionary<FuturesChain> FuturesChains { get; set; } // 期货链
    public DataDictionary<Dividend> Dividends { get; set; }         // 分红
    public DataDictionary<Split> Splits { get; set; }               // 分拆
}
```

**算法代码中的使用**:

```python
def OnData(self, slice):
    # slice 包含截至当前时刻所有品种的数据

    # 访问价格 bar
    if "AAPL" in slice.Bars:
        aapl_bar = slice.Bars["AAPL"]
        print(f"AAPL: {aapl_bar.Close}")

    # 访问 Tick 数据
    if "AAPL" in slice.Ticks:
        for tick in slice.Ticks["AAPL"]:
            print(f"Tick: {tick.LastPrice}")

    # 检查分红拆股
    for kvp in slice.Dividends.items():
        symbol, dividend = kvp
        print(f"{symbol} 分红: {dividend.Distribution}")

    # 检查新增品种 (Universe Selection 时)
    for security in slice.SecurityChanges.AddedSecurities:
        print(f"新增: {security.Symbol}")
```

### Synchronizer 的工作原理

`Synchronizer` 是全局协调器，负责将所有 Subscription 的 Enumerator 输出同步成 TimeSlice 序列。

```csharp
public class Synchronizer
{
    private readonly PriorityQueue<SubscriptionDataPacket> _queue;  // 按时间排序
    private readonly List<IEnumerator<BaseData>> _enumerators;      // 所有 Sub 的 Enumerator
    private readonly UniverseSelection _universeSelection;

    public IEnumerable<TimeSlice> Sync()
    {
        // 预加载: 从每个 Enumerator 读取第一条数据
        foreach (var enumerator in _enumerators)
        {
            if (enumerator.MoveNext())
            {
                _queue.Enqueue(new SubscriptionDataPacket(
                    data: enumerator.Current,
                    timestamp: enumerator.Current.Time
                ));
            }
        }

        // 主循环
        while (_queue.Count > 0)
        {
            var currentTime = _queue.Peek().Timestamp;  // 堆顶的最早时间
            var timeSliceData = new Dictionary<Symbol, List<BaseData>>();

            // 1. 收集当前时刻的所有数据
            while (_queue.Count > 0 && _queue.Peek().Timestamp == currentTime)
            {
                var packet = _queue.Dequeue();
                var symbol = packet.Data.Symbol;

                if (!timeSliceData.ContainsKey(symbol))
                    timeSliceData[symbol] = new List<BaseData>();

                timeSliceData[symbol].Add(packet.Data);

                // 2. 从该 Enumerator 读取下一条数据
                if (packet.Enumerator.MoveNext())
                {
                    _queue.Enqueue(new SubscriptionDataPacket(
                        data: packet.Enumerator.Current,
                        timestamp: packet.Enumerator.Current.Time,
                        enumerator: packet.Enumerator
                    ));
                }
            }

            // 3. 处理 Universe Selection (可能添加/删除品种)
            var securityChanges = _universeSelection.ProcessUniverse(currentTime);

            // 4. 为新品种预热数据
            foreach (var security in securityChanges.AddedSecurities)
            {
                // 加载历史数据用于指标初始化
                LoadWarmupData(security);
            }

            // 5. 构建 Slice 对象
            var slice = new Slice(currentTime, timeSliceData, securityChanges);

            // 6. 发射 TimeSlice
            yield return new TimeSlice
            {
                Time = currentTime,
                Slice = slice,
                SecurityChanges = securityChanges
            };
        }
    }
}
```

### 回测 vs 实盘的同步差异

#### 回测模式 (FileSystemDataFeed)

```
预读阶段:
┌─────────────────────────────────────────┐
│ 从磁盘加载所有历史数据                  │
│ 构建优先级队列                          │
└─────────────────────────────────────────┘
         ↓
推送阶段:
┌─────────────────────────────────────────┐
│ 时间推进 (尽可能快，CPU 限制)          │
│ Synchronizer 从堆中弹出数据              │
│ 构建 TimeSlice                          │
│ 调用 OnData()                           │
│ 直到堆空                                │
└─────────────────────────────────────────┘

特点:
✓ 完全确定性 (重复运行结果相同)
✓ 速度快 (内存 I/O，不受网络延迟)
✓ 可控 (随时暂停/修改时间)
```

**实盘模式 (LiveTradingDataFeed)**

```
实时数据流:
┌─────────────────────────────────────────┐
│ 品种 1 的实时数据接收 (WebSocket)       │
│ 品种 2 的实时数据接收 (REST API)        │
│ 品种 3 的实时数据接收 (Broker Feed)     │
└─────────────────────────────────────────┘
         ↓
聚合阶段:
┌─────────────────────────────────────────┐
│ Synchronizer 维护时间窗口                │
│ (e.g., 等待最多 100ms 收集同一时刻数据) │
│ 超时或接收完整，发射 TimeSlice          │
│ 调用 OnData()                           │
└─────────────────────────────────────────┘

特点:
✓ 实时性好 (毫秒级延迟)
✗ 数据不完整 (缺失品种 → 等待或跳过)
✗ 非确定性 (网络抖动影响顺序)
✓ 自动热身 (初始化阶段加载历史数据)
```

### 时间窗口聚合示例

实盘中，Synchronizer 使用时间窗口确保各品种数据同步：

```csharp
public class RealTimeDataFeed : IDataFeed
{
    private readonly TimeSpan _synchronizationWindow = TimeSpan.FromMilliseconds(100);
    private Dictionary<Symbol, Queue<BaseData>> _buffers = new();
    private DateTime _lastEmitTime = DateTime.MinValue;

    public IEnumerable<TimeSlice> Sync()
    {
        while (IsRunning)
        {
            var now = DateTime.UtcNow;
            var windowEnd = now - _synchronizationWindow;

            // 检查是否有数据已经超过时间窗口
            var shouldEmit = false;
            foreach (var buffer in _buffers.Values)
            {
                if (buffer.Count > 0 && buffer.Peek().Time <= windowEnd)
                {
                    shouldEmit = true;
                    break;
                }
            }

            if (shouldEmit)
            {
                var currentTime = GetEarliestTimestamp();
                var timeSliceData = new Dictionary<Symbol, List<BaseData>>();

                // 收集所有 <= currentTime 的数据
                foreach (var symbol in _buffers.Keys.ToList())
                {
                    var buffer = _buffers[symbol];
                    while (buffer.Count > 0 && buffer.Peek().Time <= currentTime)
                    {
                        var data = buffer.Dequeue();
                        if (!timeSliceData.ContainsKey(symbol))
                            timeSliceData[symbol] = new List<BaseData>();
                        timeSliceData[symbol].Add(data);
                    }
                }

                var slice = new Slice(currentTime, timeSliceData);
                yield return new TimeSlice { Time = currentTime, Slice = slice };
                _lastEmitTime = currentTime;
            }
            else
            {
                // 等待更多数据到达
                Thread.Sleep(10);
            }
        }
    }
}
```

---

## 回测模式的数据流

### FileSystemDataFeed 架构

回测模式从磁盘的 `/data` 目录读取历史数据。这是一个高度优化的流式读取系统。

```csharp
public class FileSystemDataFeed : IDataFeed
{
    private readonly string _dataPath;  // e.g., "/data"
    private readonly List<Subscription> _subscriptions;
    private Synchronizer _synchronizer;

    public void Initialize(
        List<Subscription> subscriptions,
        DateTime startTime,
        DateTime endTime,
        bool fillForwardData = true)
    {
        _subscriptions = subscriptions;

        // 为每个 Subscription 初始化 Enumerator
        foreach (var subscription in subscriptions)
        {
            var config = subscription.Configuration;
            var enumerator = CreateEnumerator(config, startTime, endTime);
            subscription.Enumerator = enumerator;
        }

        // 初始化全局同步器
        _synchronizer = new Synchronizer(
            subscriptions.Select(s => s.Enumerator).ToList(),
            startTime,
            endTime
        );
    }

    private IEnumerator<BaseData> CreateEnumerator(
        SubscriptionDataConfig config,
        DateTime startTime,
        DateTime endTime)
    {
        // 1. 确定数据文件位置
        var dataPath = GetDataPath(config);  // e.g., "/data/equity/usa/minute/aapl"

        // 2. 创建源 Enumerator (文件系统读取)
        IEnumerator<BaseData> source =
            new FileSystemDataEnumerator(dataPath, config, startTime, endTime);

        // 3. 应用 Enumerator 栈
        source = ApplyEnumeratorStack(source, config);

        return source;
    }

    public IEnumerable<TimeSlice> GetNextTimeSlices()
    {
        // 使用 Synchronizer 同步所有 Subscription 的数据
        foreach (var timeSlice in _synchronizer.Sync())
        {
            yield return timeSlice;
        }
    }
}
```

### 数据加载过程

#### 步骤 1: 确定要加载的文件

```csharp
private List<string> GetDataFiles(
    string basePath,
    Symbol symbol,
    Resolution resolution,
    DateTime startDate,
    DateTime endDate)
{
    var files = new List<string>();
    var currentDate = startDate.Date;

    while (currentDate <= endDate.Date)
    {
        // 根据分辨率构建文件路径
        var filePath = resolution switch
        {
            Resolution.Tick =>
                $"{basePath}/tick/{symbol.Value.ToLower()}/{currentDate:yyyyMMdd}_trade.zip",

            Resolution.Minute =>
                $"{basePath}/minute/{symbol.Value.ToLower()}/{currentDate:yyyyMMdd}_trade.zip",

            Resolution.Hour =>
                $"{basePath}/hour/{symbol.Value.ToLower()}/{symbol.Value.ToLower()}.csv",

            Resolution.Daily =>
                $"{basePath}/daily/{symbol.Value.ToLower()}.zip",
        };

        if (File.Exists(filePath))
            files.Add(filePath);

        currentDate = currentDate.AddDays(1);
    }

    return files;
}
```

#### 步骤 2: 流式读取和解析

```csharp
public class FileSystemDataEnumerator : IEnumerator<BaseData>
{
    private readonly List<string> _files;
    private readonly SubscriptionDataConfig _config;
    private int _fileIndex = -1;
    private ZipArchive _currentZip;
    private StreamReader _currentReader;
    private BaseData _current;
    private string _line;

    public bool MoveNext()
    {
        while (true)
        {
            // 读取当前行
            if (_currentReader != null)
            {
                _line = _currentReader.ReadLine();
                if (_line != null)
                {
                    _current = ParseLine(_line);
                    return true;
                }

                // 当前文件读取完毕
                _currentReader?.Dispose();
                _currentZip?.Dispose();
            }

            // 尝试打开下一个文件
            _fileIndex++;
            if (_fileIndex >= _files.Count)
                return false;  // 所有文件读取完毕

            var zipPath = _files[_fileIndex];
            _currentZip = ZipFile.Open(zipPath, ZipArchiveMode.Read);

            // 在 ZIP 中找到 CSV 文件
            var csvEntry = _currentZip.Entries[0];  // 假设只有一个 CSV
            _currentReader = new StreamReader(csvEntry.Open());

            // 跳过 CSV 标题行
            _currentReader.ReadLine();
        }
    }

    private BaseData ParseLine(string line)
    {
        // 根据 _config.DataType 解析
        // 例如: "2021-01-04 09:30:00,130.15,130.50,130.10,130.45,5100000"

        var fields = line.Split(',');
        var time = DateTime.ParseExact(fields[0], "yyyy-MM-dd HH:mm:ss",
            CultureInfo.InvariantCulture);

        return new TradeBar
        {
            Time = time,
            Symbol = _config.Symbol,
            Open = decimal.Parse(fields[1]),
            High = decimal.Parse(fields[2]),
            Low = decimal.Parse(fields[3]),
            Close = decimal.Parse(fields[4]),
            Volume = long.Parse(fields[5])
        };
    }

    public BaseData Current => _current;
}
```

### 并行数据加载

对于多品种策略，LEAN 使用并行加载优化性能：

```csharp
public class ParallelFileSystemDataFeed : IDataFeed
{
    public void Initialize(List<Subscription> subscriptions,
        DateTime startTime, DateTime endTime)
    {
        // 并行初始化每个 Subscription 的 Enumerator
        Parallel.ForEach(subscriptions, subscription =>
        {
            var config = subscription.Configuration;
            var enumerator = CreateEnumerator(config, startTime, endTime);
            lock (_lock)  // 线程安全
            {
                subscription.Enumerator = enumerator;
            }
        });
    }
}
```

**性能指标**:
- 单品种日线数据: 瞬间 (KB)
- 单品种分钟数据: <100ms (数 MB)
- 100 品种分钟数据: <1s (100+ MB)
- 全市场 Tick 数据: 数分钟 (GB+)

### 内存管理

关键是**流式读取**，不把所有数据加载到内存：

```csharp
// ✗ 错误方式: 一次性加载所有数据
var allData = LoadAllData(startDate, endDate);  // 可能是 GB！
foreach (var bar in allData)
{
    yield return bar;
}

// ✓ 正确方式: 逐条读取
foreach (var line in ReadLinesFromZip(zipPath))
{
    var bar = ParseLine(line);
    yield return bar;  // 立即发射，不存储
}
```

**内存使用量**:
- Enumerator 栈: 缓冲数个 bar (~KB)
- Synchronizer 优先级队列: 最多 N 个品种的最新 bar (~KB)
- 用户算法: Indicator 缓冲 + 订单记录 (可控制)
- 总计: 对于 100 品种，通常 < 10MB

---

## 实盘模式的数据流

### LiveTradingDataFeed 架构

实盘交易从多个实时数据源摄取数据：

```csharp
public class LiveTradingDataFeed : IDataFeed
{
    private readonly IBrokerage _brokerage;              // 券商行情 (IB, Alpaca 等)
    private readonly IDataProvider[] _dataProviders;    // 额外数据源 (加密货币、商品等)
    private readonly ConcurrentQueue<DataFeedPacket> _dataQueue;
    private Synchronizer _synchronizer;
    private Thread _dataThread;

    public void Initialize(List<Subscription> subscriptions)
    {
        // 1. 为每个 Subscription 初始化实时 Enumerator
        foreach (var subscription in subscriptions)
        {
            var config = subscription.Configuration;
            IEnumerator<BaseData> enumerator;

            if (config.DataType == typeof(Tick) ||
                config.Resolution == Resolution.Tick)
            {
                // Tick 数据: 从券商订阅
                enumerator = new LiveTickDataEnumerator(
                    _brokerage, config.Symbol);
            }
            else
            {
                // Bar 数据: 需要自己聚合 Tick → Bar
                var tickEnumerator = new LiveTickDataEnumerator(
                    _brokerage, config.Symbol);

                enumerator = new AggregationEnumerator(
                    tickEnumerator, config.Resolution);
            }

            subscription.Enumerator = enumerator;
        }

        // 2. 初始化同步器
        _synchronizer = new RealtimeSynchronizer(
            subscriptions.Select(s => s.Enumerator).ToList()
        );

        // 3. 启动后台数据接收线程
        _dataThread = new Thread(DataReceiveLoop)
        {
            IsBackground = true,
            Name = "LiveDataFeed"
        };
        _dataThread.Start();
    }

    private void DataReceiveLoop()
    {
        while (IsRunning)
        {
            try
            {
                // 不断接收来自各源的数据，放入队列
                foreach (var provider in _dataProviders)
                {
                    var data = provider.GetNextData();
                    if (data != null)
                    {
                        _dataQueue.Enqueue(new DataFeedPacket(data));
                    }
                }
            }
            catch (Exception ex)
            {
                // 日志、报警等
                Logger.Error($"Data reception error: {ex.Message}");
            }

            Thread.Sleep(1);  // 让出 CPU
        }
    }

    public IEnumerable<TimeSlice> GetNextTimeSlices()
    {
        foreach (var timeSlice in _synchronizer.Sync())
        {
            yield return timeSlice;
        }
    }
}
```

### 实时数据源集成

#### 1. 券商行情源 (Brokerage)

```csharp
public class LiveTickDataEnumerator : IEnumerator<BaseData>
{
    private readonly IBrokerage _brokerage;
    private readonly Symbol _symbol;
    private BlockingCollection<Tick> _tickQueue;

    public LiveTickDataEnumerator(IBrokerage brokerage, Symbol symbol)
    {
        _brokerage = brokerage;
        _symbol = symbol;
        _tickQueue = new BlockingCollection<Tick>();

        // 订阅行情
        _brokerage.OnTick += HandleNewTick;
        _brokerage.Subscribe(_symbol);
    }

    private void HandleNewTick(Tick tick)
    {
        if (tick.Symbol == _symbol)
        {
            _tickQueue.Add(tick);  // 线程安全队列
        }
    }

    public bool MoveNext()
    {
        // 等待下一个 Tick (最多等待 5 秒)
        if (_tickQueue.TryTake(out var tick, TimeSpan.FromSeconds(5)))
        {
            Current = tick;
            return true;
        }

        return false;  // 超时
    }

    public BaseData Current { get; private set; }
}
```

**集成示例**:
```csharp
// Interactive Brokers 行情
IBrokerage ib = new InteractiveBrokersBrokerage(accountId, password);
dataFeed.AddBrokerageDataProvider(ib);

// Alpaca 加密货币
var alpaca = new AlpacaCryptoBrokerage(apiKey, apiSecret);
dataFeed.AddBrokerageDataProvider(alpaca);

// 第三方数据商
var newsProvider = new NewsDataProvider(apiKey);
dataFeed.AddDataProvider(newsProvider);
```

#### 2. 第三方数据源 (REST API)

```csharp
public class RestDataProvider : IDataProvider
{
    private readonly HttpClient _client;
    private readonly string _apiUrl;
    private readonly ConcurrentDictionary<Symbol, DateTime> _lastFetchTime;

    public BaseData GetNextData()
    {
        // 对每个订阅的符号，定期轮询 REST API
        foreach (var symbol in _symbols)
        {
            // 频率限制: 每 5 秒一次
            if (DateTime.UtcNow - _lastFetchTime[symbol] < TimeSpan.FromSeconds(5))
                continue;

            try
            {
                var response = _client.GetAsync(
                    $"{_apiUrl}/symbol/{symbol}/quote")
                    .Result;

                var json = response.Content.ReadAsStringAsync().Result;
                var data = JsonConvert.DeserializeObject<QuoteBar>(json);

                _lastFetchTime[symbol] = DateTime.UtcNow;
                return data;
            }
            catch (Exception ex)
            {
                Logger.Warn($"Failed to fetch {symbol}: {ex.Message}");
            }
        }

        return null;
    }
}
```

#### 3. WebSocket 数据源

```csharp
public class WebSocketDataProvider : IDataProvider
{
    private readonly ClientWebSocket _socket;
    private readonly BlockingCollection<BaseData> _dataQueue;

    public async Task ConnectAsync(string url)
    {
        _socket = new ClientWebSocket();
        await _socket.ConnectAsync(new Uri(url), CancellationToken.None);

        // 启动后台接收
        _ = ReceiveLoop();
    }

    private async Task ReceiveLoop()
    {
        var buffer = new byte[4096];

        while (_socket.State == WebSocketState.Open)
        {
            var result = await _socket.ReceiveAsync(
                new ArraySegment<byte>(buffer),
                CancellationToken.None);

            if (result.MessageType == WebSocketMessageType.Text)
            {
                var json = Encoding.UTF8.GetString(buffer, 0, result.Count);
                var tick = JsonConvert.DeserializeObject<Tick>(json);

                _dataQueue.Add(tick);  // 线程安全
            }
        }
    }

    public BaseData GetNextData()
    {
        _dataQueue.TryTake(out var data, TimeSpan.FromMilliseconds(100));
        return data;
    }
}
```

**连接示例**:
```python
def Initialize(self):
    # 使用 Polygon.io WebSocket 行情
    self.AddData(
        PolygonWebSocketData,
        "AAPL",
        Resolution.Minute,
        dataNormalizationMode=DataNormalizationMode.Raw
    )

# Polygon.io 会在后台不断推送数据到 WebSocket
# LEAN 接收并聚合成 Minute bar
# OnData() 每分钟被调用一次
```

### 热身期 (Warmup)

实盘交易开始前，需要加载历史数据初始化指标：

```python
def Initialize(self):
    self.SetWarmup(timedelta(days=30))  # 预热 30 天历史数据
    self.AddEquity("AAPL", Resolution.Daily)
    self.indicator = SimpleMovingAverage(20)

def OnData(self, slice):
    if self.IsWarmingUp:
        # 加载历史数据，不执行交易逻辑
        self.indicator.Update(slice["AAPL"])
        return

    # 热身完成，开始交易
    if not self.indicator.IsReady:
        return

    # 交易逻辑
    if slice["AAPL"].Close > self.indicator.Current.Value:
        self.SetHoldings("AAPL", 1.0)
```

**热身流程**:

```
实盘开始时间: 2024-03-15 09:30:00
SetWarmup(30 days)
    ↓
加载历史: 2024-02-13 → 2024-03-14 (30 个交易日)
    ↓
逐日调用 OnData()，但 IsWarmingUp=True
    ↓
指标、账户状态初始化完成
    ↓
2024-03-15 09:30:00 正式开始，IsWarmingUp=False
    ↓
正常交易

好处:
✓ 指标有足够历史数据
✓ 账户 Leverage 正确初始化
✓ 避免第一笔交易逻辑错误
```

### 数据延迟和缺失处理

实盘中常见数据问题：

```python
class DataReliabilityHandler:
    def __init__(self, algorithm):
        self.algorithm = algorithm
        self.last_data_time = {}  # 记录每个品种的最后数据时间
        self.data_gap_threshold = timedelta(minutes=5)  # 5 分钟无数据报警

    def OnData(self, slice):
        # 问题 1: 延迟数据 (数据到达时间 > 实际时间)
        for symbol, bar in slice.Bars.items():
            if datetime.utcnow() - bar.Time > timedelta(minutes=1):
                logger.warn(f"{symbol} 数据延迟: {bar.Time}")

        # 问题 2: 缺失品种 (某个品种没有数据)
        missing = set(self.algorithm.portfolio.keys()) - set(slice.Bars.keys())
        for symbol in missing:
            last_time = self.last_data_time.get(symbol)
            if last_time and datetime.utcnow() - last_time > self.data_gap_threshold:
                logger.error(f"{symbol} 超过 5 分钟无数据!")
                # 可以选择: 平仓, 发报警, 或继续等待

        # 问题 3: 异常值 (价格跳跃)
        for symbol, bar in slice.Bars.items():
            last_bar = self.get_last_bar(symbol)
            if last_bar and bar.Close / last_bar.Close > 1.1:
                logger.warn(f"{symbol} 价格跳跃 10%+: {last_bar.Close} → {bar.Close}")

        # 更新记录
        for symbol in slice.Bars.keys():
            self.last_data_time[symbol] = datetime.utcnow()
```

---

## 自定义数据源

### 扩展 BaseData

任何自定义数据都必须继承 `BaseData`：

```csharp
public abstract class BaseData : ICloneable
{
    public Symbol Symbol { get; set; }
    public DateTime Time { get; set; }
    public decimal Value { get; set; }

    /// <summary>
    /// 数据读取方法: 从 CSV 行解析数据
    /// </summary>
    public abstract BaseData Reader(SubscriptionDataConfig config,
        string line, DateTime date, bool isLiveMode);

    /// <summary>
    /// 数据源方法: 指定从何处读取数据
    /// </summary>
    public abstract SubscriptionDataSource GetSource(SubscriptionDataConfig config,
        DateTime date, bool isLiveMode);

    /// <summary>时间戳（用于排序）</summary>
    public override DateTime Time { get; set; }

    /// <summary>克隆（用于 Fill Forward）</summary>
    public abstract object Clone();
}
```

### 示例 1: 情绪数据

```python
from clr import AddReference
AddReference("QuantConnect.Common")
from QuantConnect import *

class SentimentData(PythonData):
    """
    情绪评分数据

    数据源格式 (CSV):
    Date,Symbol,SentimentScore,News
    2024-01-05,AAPL,0.75,Apple announced new product
    """

    def GetSource(self, config, date, isLiveMode):
        """指定数据文件位置"""

        # 本地文件
        if not isLiveMode:
            base_path = f"/data/alternative/sentiment/{date.strftime('%Y/%m')}"
            filename = f"{base_path}/{config.Symbol}_sentiment.csv"

            return SubscriptionDataSource(
                filename,
                SubscriptionTransportMedium.LocalFile
            )

        # 实盘: 从 API 获取
        else:
            return SubscriptionDataSource(
                f"https://api.example.com/sentiment/{config.Symbol}",
                SubscriptionTransportMedium.Rest,
                SubscriptionTransportMedium.RemoteFile
            )

    def Reader(self, config, line, date, isLiveMode):
        """解析数据行"""

        if not line:
            return None

        data = line.split(',')

        return SentimentData {
            Symbol = config.Symbol,
            Time = datetime.strptime(data[0], "%Y-%m-%d"),
            Value = float(data[2]),  # SentimentScore
            SentimentScore = float(data[2]),
            News = data[3]
        }

    def __init__(self):
        self.Value = 0
        self.SentimentScore = 0
        self.News = ""


class SentimentStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2024, 1, 1)
        self.SetEndDate(2024, 3, 1)
        self.SetCash(10000)

        # 添加情绪数据源
        self.AddData(SentimentData, "AAPL", Resolution.Daily)
        self.AddEquity("AAPL", Resolution.Daily)

    def OnData(self, slice):
        # 访问情绪数据
        if "AAPL" in slice:
            sentiment = slice["AAPL"]

            if sentiment.SentimentScore > 0.7:
                # 强正面情绪: 买入
                self.SetHoldings("AAPL", 1.0)
            elif sentiment.SentimentScore < 0.3:
                # 强负面情绪: 卖出
                self.Liquidate("AAPL")

            self.Log(f"Sentiment: {sentiment.SentimentScore:.2f}, News: {sentiment.News}")
```

### 示例 2: 加密货币 OHLCV

```python
class CryptoCurrencyData(PythonData):
    """
    加密货币分钟数据

    数据源: 本地 JSON 文件
    """

    def GetSource(self, config, date, isLiveMode):
        symbol = config.Symbol.Value.lower()  # "btc/usd" → "btc_usd"

        if isLiveMode:
            # 实盘: 订阅 WebSocket
            return SubscriptionDataSource(
                f"wss://stream.binance.com/ws/{symbol}@klines_1m",
                SubscriptionTransportMedium.WebSocket
            )
        else:
            # 回测: 本地 CSV
            filename = f"/data/crypto/binance/minute/{symbol}/{date.strftime('%Y%m%d')}.csv"
            return SubscriptionDataSource(
                filename,
                SubscriptionTransportMedium.LocalFile
            )

    def Reader(self, config, line, date, isLiveMode):
        if not line:
            return None

        # 处理两种格式
        if isLiveMode:
            # WebSocket 消息格式 (JSON)
            import json
            msg = json.loads(line)
            k = msg['k']

            return CryptoCurrencyData {
                Symbol = config.Symbol,
                Time = datetime.fromtimestamp(k['t'] / 1000),
                Open = float(k['o']),
                High = float(k['h']),
                Low = float(k['l']),
                Close = float(k['c']),
                Volume = float(k['v']),
                Value = float(k['c'])
            }
        else:
            # CSV 格式 (本地回测)
            data = line.split(',')
            return CryptoCurrencyData {
                Symbol = config.Symbol,
                Time = datetime.strptime(data[0], "%Y-%m-%d %H:%M:%S"),
                Open = float(data[1]),
                High = float(data[2]),
                Low = float(data[3]),
                Close = float(data[4]),
                Volume = float(data[5]),
                Value = float(data[4])
            }

    def __init__(self):
        self.Open = 0
        self.High = 0
        self.Low = 0
        self.Close = 0
        self.Volume = 0
```

### 自定义数据的 Enumerator 集成

当使用自定义数据时，LEAN 自动处理：

```
AddData(SentimentData, "AAPL", Resolution.Daily)
    ↓
创建 SubscriptionDataConfig
    ├─ DataType = SentimentData
    ├─ Symbol = AAPL
    ├─ Resolution = Daily
    └─ IsCustomData = True
    ↓
SubscriptionDataFactory.CreateEnumerator()
    ├─ FileSystemDataEnumerator 读取文件
    ├─ 调用 SentimentData.Reader() 解析每行
    ├─ FillForwardEnumerator (如配置)
    └─ FilterEnumerator (应用交易所时间等)
    ↓
同步到主 Synchronizer
    ↓
OnData(slice) 中访问
    if slice.ContainsKey("AAPL"):
        sentiment = slice["AAPL"]
```

---

## Universe Selection 与动态订阅

### Universe Selection 的目的

Universe Selection 允许策略在运行时动态调整交易品种集合，而不是固定订阅。

```python
class DynamicUniverseStrategy(QCAlgorithm):
    def Initialize(self):
        # 不指定具体品种，而是定义选择规则
        self.AddUniverse(self.SelectUniverse)

    def SelectUniverse(self, coarse_data):
        """
        每天调用（对于 Daily Resolution）
        或每分钟调用（对于 Minute Resolution）

        输入: CoarseFundamental 列表（所有美股的基本数据）
        输出: 要交易的 Symbol 列表
        """

        # 步骤 1: 过滤流动性好的品种
        filtered = [c for c in coarse_data
                   if c.DollarVolume > 10000000  # 日成交金额 > 1000万
                   and c.Price > 5]               # 价格 > $5

        # 步骤 2: 按某个指标排序
        sorted_symbols = sorted(filtered,
                               key=lambda x: x.DollarVolume,
                               reverse=True)

        # 步骤 3: 选择前 N 个
        return [x.Symbol for x in sorted_symbols[:100]]
```

### 基础 Universe (CoarseFundamental)

`CoarseFundamental` 包含每个品种的基本市场数据：

```csharp
public class CoarseFundamental
{
    public Symbol Symbol { get; set; }
    public decimal Price { get; set; }                  // 最新价
    public long Volume { get; set; }                    // 日成交量
    public decimal DollarVolume { get; set; }           // 日成交金额
    public bool HasFundamentalData { get; set; }        // 是否有财报数据
    public bool IsEtf { get; set; }                     // 是否 ETF
    public DateTime DelistingDate { get; set; }         // 退市日期（如果已退市）
}
```

### 进阶 Universe (FineFundamental)

对于更精细的选择，可以使用 `FineFundamental`：

```python
class FineFundamentalStrategy(QCAlgorithm):
    def Initialize(self):
        # 先用粗选筛选流动性好的品种
        self.AddUniverse(
            self.SelectCoarse,
            self.SelectFine
        )

    def SelectCoarse(self, coarse):
        """第 1 层: 粗选 (速度快)"""
        return [c.Symbol for c in coarse
               if c.DollarVolume > 10000000
               and not c.IsEtf][:200]  # 前 200 个流动性最好的

    def SelectFine(self, fine):
        """第 2 层: 细选 (可以访问财务数据)"""
        # 访问财务数据
        fine_filtered = [f for f in fine
                        if f.ValuationRatios.PeRatio < 20       # PE < 20
                        and f.OperationRatios.ROA > 0.1]        # ROA > 10%

        # 按 PE 排序，选择最便宜的 50 个
        sorted_by_pe = sorted(fine_filtered,
                             key=lambda x: x.ValuationRatios.PeRatio)
        return [f.Symbol for f in sorted_by_pe[:50]]
```

### Universe Selection 的数据流

```
每日/每分钟时刻:
┌─────────────────────────────────────┐
│ SelectUniverse() 调用               │
└─────────────────────────────────────┘
         ↓
加载当天所有美股的 CoarseFundamental
(包括新上市、退市的)
         ↓
策略代码执行过滤逻辑
         ↓
返回选定的 Symbol 列表
例: [AAPL, MSFT, GOOGL, AMZN, ...]
         ↓
对比前一时刻的列表
┌─────────────────────────────────────┐
│ AddedSecurities = 新增品种          │
│ RemovedSecurities = 删除品种        │
└─────────────────────────────────────┘
         ↓
处理新增品种:
├─ 为每个新品种创建 Subscription
├─ 初始化 Enumerator 链
├─ 加载 Warmup 历史数据
└─ 初始化指标、缓存等
         ↓
处理删除品种:
├─ 销毁 Subscription
├─ 平仓所有头寸
└─ 释放相关资源
         ↓
OnSecuritiesChanged() 回调
(策略可以做额外初始化)
         ↓
OnData() 接收完整的 SecurityChanges
策略代码可以:
- 检查 AddedSecurities 进行初始化
- 检查 RemovedSecurities 进行清理
```

### Universe Selection 示例: 动态追踪高成长股

```python
class DynamicGrowthStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)
        self.SetCash(100000)

        # 添加动态 Universe
        self.AddUniverse(self.SelectUniverse)

        # 跟踪已持仓品种
        self.holdings_by_sector = {}

    def SelectUniverse(self, coarse):
        """
        选择过去 252 个交易日涨幅最大的 20 个品种
        """
        # 过滤基本条件
        filtered = [c for c in coarse
                   if c.DollarVolume > 5000000
                   and c.Price > 10
                   and c.HasFundamentalData]

        # 计算每个品种的 252 日收益率
        returns = []
        for c in filtered:
            # 获取历史价格
            hist = self.History([c.Symbol], 252, Resolution.Daily)
            if len(hist) > 1:
                ret = (hist.iloc[-1]['close'] - hist.iloc[0]['close']) / hist.iloc[0]['close']
                returns.append((c.Symbol, ret))

        # 排序并选择前 20
        top_20 = sorted(returns, key=lambda x: x[1], reverse=True)[:20]
        return [x[0] for x in top_20]

    def OnSecuritiesChanged(self, changes):
        """
        当 Universe 改变时调用

        changes.AddedSecurities: 新增品种
        changes.RemovedSecurities: 删除品种
        """
        for security in changes.AddedSecurities:
            # 初始化新品种
            symbol = security.Symbol

            # 创建指标
            self.RegisterIndicator(symbol,
                                  SimpleMovingAverage(20),
                                  Resolution.Daily)

            self.Log(f"Added: {symbol}")

        for security in changes.RemovedSecurities:
            # 平仓删除的品种
            if self.Portfolio[security.Symbol].Quantity != 0:
                self.Liquidate(security.Symbol)

            self.Log(f"Removed: {security.Symbol}")

    def OnData(self, slice):
        """正常的 OnData，现在处理动态品种集合"""

        # 检查是否有新品种被添加
        if hasattr(slice, 'SecurityChanges'):
            for security in slice.SecurityChanges.AddedSecurities:
                # 新增品种的初始化逻辑
                pass

        # 为每个当前持有的品种检查信号
        for symbol in self.Portfolio.Keys:
            if not self.Portfolio[symbol].Invested:
                continue

            if symbol not in slice.Bars:
                continue  # 该品种没有数据

            bar = slice.Bars[symbol]

            # 简单策略: 跌破 20 日均线则卖出
            if bar.Close < self.Indicator[symbol].Current.Value:
                self.Liquidate(symbol)
```

### 性能考量: Universe 更新频率

```python
# 低频更新 (日线)
self.AddUniverse(self.SelectUniverse)  # 默认每天一次

# 手动控制更新频率
class ManualUniverseSelection(QCAlgorithm):
    def Initialize(self):
        self.last_universe_update = None
        self.universe_update_interval = timedelta(days=5)  # 每 5 天更新一次

        self.AddUniverse(self.SelectUniverse)

    def SelectUniverse(self, coarse):
        # 如果距离上次更新不足 5 天，返回 None (保持现有)
        if (self.last_universe_update and
            self.Time - self.last_universe_update < self.universe_update_interval):
            return Universe.Unchanged

        self.last_universe_update = self.Time

        # 进行完整的 Universe 选择
        selected = [c.Symbol for c in coarse
                   if c.DollarVolume > 10000000][:100]

        return selected
```

---

## 性能考量与优化

### 1. 数据格式与压缩

**ZIP 压缩的优势**:

```
原始 CSV (1 年 Minute 数据):
- 390 分钟/天 × 250 交易日 = 97,500 行
- 每行 ~50 字节 = 4.9 MB
- 全市场 3000 股票 = 14.7 GB

压缩后 (ZIP):
- 压缩率: 90%+ (重复数据多)
- 4.9 MB → 0.5 MB
- 全市场 = 1.5 GB

加载速度:
- 磁盘 I/O: 50 MB/s (现代 SSD)
- ZIP 解压: CPU 限制 (~100 MB/s)
- 总时间 (全市场): ~30 秒
```

### 2. 流式读取 vs 全量加载

**内存使用对比**:

```
场景: 100 品种, Minute 数据, 1 年

全量加载 (❌ 不推荐):
┌──────────────────────────────┐
│ Minute Bar 对象               │
│ 每个 Bar: ~200 字节            │
│ 97,500 bars × 100 symbols     │
│ = 1,950,000 对象              │
│ = ~400 MB 内存                 │
│ + 指标缓冲、订单等             │
│ 总计: ~1 GB                    │
└──────────────────────────────┘

流式读取 (✓ LEAN 方式):
┌──────────────────────────────┐
│ Enumerator 栈 (~1 MB)         │
│ - Synchronizer 队列 (100 items)│
│ - 指标缓冲 (250 days × 100)    │
│ 总计: ~50 MB                  │
│                               │
│ 性能: 相同速度                 │
│ 内存: 20 倍改进!               │
└──────────────────────────────┘
```

### 3. 并行数据加载

```csharp
// 顺序加载 (慢)
foreach (var subscription in subscriptions)
{
    var enumerator = CreateEnumerator(subscription);  // 每个 ~100ms
}
// 总时间: 100 订阅 × 100ms = 10 秒

// 并行加载 (快)
Parallel.ForEach(subscriptions,
    new ParallelOptions { MaxDegreeOfParallelism = 8 },  // 8 个线程
    subscription =>
    {
        var enumerator = CreateEnumerator(subscription);
    }
);
// 总时间: 100 订阅 / 8 线程× 100ms ≈ 1.3 秒
```

### 4. 多分辨率数据聚合性能

```
场景: 1 品种从 Tick 聚合到 Daily

Tick → Second (聚合):
  输入: ~500,000 ticks/天
  聚合: 390 second bars
  缓冲: ~1000 ticks (内存)
  CPU: 低 (简单迭代)

Second → Minute → Hour → Daily (级联):
  ┌─────────────────┐
  │ 500K ticks      │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 390 second bars │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 390 minute bars │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 6.5 hour bars   │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 1 daily bar     │
  └─────────────────┘

优化: 如果只需要 Hour bar，跳过 Minute
  500K ticks → (聚合到 Hour) → 6.5 hour bars
  避免中间层的对象创建
```

### 5. 索引和查询优化

**快速定位数据文件**:

```csharp
// 构建索引 (一次性，第一次运行)
public class DataFileIndex
{
    // 索引: Symbol → [(Resolution, MinDate, MaxDate)]
    private Dictionary<string, List<DateRange>> _index;

    public List<string> FindFiles(Symbol symbol, Resolution resolution,
        DateTime startDate, DateTime endDate)
    {
        // O(1) 查找
        var ranges = _index[symbol.Value];

        return ranges
            .Where(r => r.Resolution == resolution
                    && r.DateRange.Overlaps(startDate, endDate))
            .SelectMany(r => r.Files)
            .ToList();
    }
}

// 首次初始化时扫描 /data 目录，构建索引
// 后续加载 symbol 时使用索引，O(1) 查找
// 避免每次都遍历 /data/equity/usa/minute/ 目录
```

### 6. 缓存策略

```python
class DataCache:
    """
    缓存最近加载的 bar，避免重复读磁盘
    """

    def __init__(self, max_size_mb=100):
        self.cache = OrderedDict()  # 按加载顺序
        self.max_size = max_size_mb * 1024 * 1024
        self.current_size = 0

    def get(self, symbol, date, resolution):
        """O(1) 查询"""
        key = (symbol, date, resolution)
        if key in self.cache:
            return self.cache[key]
        return None

    def put(self, symbol, date, resolution, data):
        """存储，超过限制则移除最旧的"""
        key = (symbol, date, resolution)

        # 估算大小
        size = len(str(data).encode())

        # 移除旧项目
        while self.current_size + size > self.max_size and self.cache:
            oldest_key, oldest_data = self.cache.popitem(last=False)
            self.current_size -= len(str(oldest_data).encode())

        self.cache[key] = data
        self.current_size += size

# 应用: 避免重复读同一日期的文件
file_cache = DataCache()
for symbol in symbols:
    # 第 1 次读: 磁盘 I/O
    data = file_cache.get(symbol, date, resolution) or load_from_disk(...)
    file_cache.put(symbol, date, resolution, data)

    # 第 2 次读: 内存命中
    data = file_cache.get(symbol, date, resolution)  # 快!
```

### 7. 数据分辨率选择指南

```
使用场景选择指南:

Tick 分辨率:
  ✓ 超高频策略 (sub-second)
  ✓ 市场微观结构研究
  ✗ 数据量太大 (GB/year)
  ✗ 回测速度慢

Second 分辨率:
  ✓ 高频策略 (秒级)
  ✓ 事件驱动策略
  ~ 数据量中等 (MB/year)
  ~ 回测速度中等

Minute 分辨率 (推荐):
  ✓ 日内交易
  ✓ 数据大小合理 (KB/year)
  ✓ 回测速度快
  ✓ 足够捕捉价格行为

Hour 分辨率:
  ✓ 短期 swing 交易
  ✓ 非常紧凑的数据集
  ~ 少了很多细节

Daily 分辨率:
  ✓ 长期趋势策略
  ✓ 数据极小
  ✓ 回测瞬间完成
  ✗ 完全错过日内行为
```

**实际数据量对比** (1 股票 × 1 年):

| 分辨率 | 数据点 | 单文件大小 | 年度增长 |
|--------|--------|----------|----------|
| Tick | ~10M | 100 MB | 1 GB |
| Second | 390K | 4 MB | 40 MB |
| Minute | 97.5K | 1 MB | 10 MB |
| Hour | 1.6K | 20 KB | 200 KB |
| Daily | 250 | 3 KB | 30 KB |

---

## 总结

### LEAN 数据管线的设计优雅性

```
问题: 如何统一处理多来源、多品种、多分辨率的数据？

LEAN 的解决方案:
1. 统一的 BaseData 接口
   └─ TradeBar, Tick, QuoteBar, 自定义数据无缝集成

2. Enumerator 组合管道
   └─ RxJS 风格的 pipe chain
   └─ 每个 Enumerator 负责一项转换
   └─ 易于测试、扩展、优化

3. 全局 Synchronizer
   └─ 自动对齐多品种、多分辨率
   └─ 确保算法接收一致的 TimeSlice

4. 流式处理
   └─ 恒定的内存占用
   └─ 线性的处理时间
   └─ 支持任意大的数据集

结果: 用户代码的简洁性
```

```python
# 用户只需这么写:
def Initialize(self):
    self.AddEquity("AAPL", Resolution.Minute)
    self.AddUniverse(self.SelectUniverse)
    self.AddData(MyCustomData, "SYMBOL", Resolution.Daily)

def OnData(self, slice):
    # 自动处理: 数据解析、聚合、同步、过滤、缺失填充
    # 用户只需关注交易逻辑
    if slice.ContainsKey("AAPL"):
        price = slice.Bars["AAPL"].Close
        # ...
```

### 关键设计模式

| 模式 | 应用 | 优势 |
|------|------|------|
| **Factory** | DataFeed, EnumeratorFactory | 灵活构建复杂对象 |
| **Decorator** | Enumerator 栈 | 可组合的数据转换 |
| **Chain of Responsibility** | Filter 链 | 模块化的验证逻辑 |
| **Priority Queue** | Synchronizer | O(logN) 时间对齐 |
| **Iterator** | IEnumerator<T> | 惰性求值、低内存 |
| **Observer** | 数据回调、事件 | 异步数据注入 |

### 性能指标

对于 100 品种 × 1 年 × Minute 分辨率的回测：

```
内存占用: ~50 MB
加载时间: ~30 秒 (IO 限制)
回测速度: 实时时间的 50~200 倍
  (取决于 CPU, 指标复杂度, 订单逻辑)
```

---

**导航**: [上一篇: LEAN 引擎架构总览](./02-lean-architecture-overview.md) | [下一篇: Algorithm Framework 五层流水线](./04-algorithm-framework.md)

**关键术语速查**:

- **TimeSlice**: 特定时刻所有品种的数据快照
- **Enumerator**: 可组合的数据读取/转换单元
- **Subscription**: 对某个品种某个分辨率数据的订阅
- **Fill Forward**: 用前一 bar 填充缺失的 bar
- **Synchronizer**: 跨品种/分辨率数据同步器
- **Universe Selection**: 动态选择交易品种的机制
- **Warmup**: 实盘交易前加载历史数据初始化指标
