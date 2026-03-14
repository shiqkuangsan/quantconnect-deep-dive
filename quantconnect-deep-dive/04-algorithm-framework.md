# LEAN 算法框架深度解析

[上一篇: 数据管线与订阅系统](./03-data-pipeline.md) | [下一篇: 订单执行与真实性建模](./05-order-execution.md)

## 目录

1. [为什么需要 Algorithm Framework](#为什么需要-algorithm-framework)
2. [五层流水线总览](#五层流水线总览)
3. [Universe Selection Model 详解](#universe-selection-model-详解)
4. [Alpha Model 详解](#alpha-model-详解)
5. [Portfolio Construction Model 详解](#portfolio-construction-model-详解)
6. [Risk Management Model 详解](#risk-management-model-详解)
7. [Execution Model 详解](#execution-model-详解)
8. [完整策略示例](#完整策略示例)
9. [Classic 风格 vs Framework 风格](#classic-风格-vs-framework-风格)
10. [模块化的价值](#模块化的价值)

---

## 为什么需要 Algorithm Framework

### 传统单体模式的痛点

在 LEAN 框架出现之前，量化策略都在单个 `OnData()` 方法中混淆所有逻辑：

```python
def OnData(self, data):
    # 过滤要交易的资产
    if self.portfolio.invested:
        return

    # 生成交易信号
    if len(self.history) < 20:
        return
    fast_ema = self.fast_ema.Current.Value
    slow_ema = self.slow_ema.Current.Value

    # 计算持仓权重
    if fast_ema > slow_ema:
        # 买入多少？
        quantity = int(self.portfolio.cash / data['SPY'].Close)

        # 风险控制：是否买入？
        if self.portfolio.TotalPortfolioValue * 0.05 > quantity * data['SPY'].Close:
            self.MarketOrder('SPY', quantity)

    # 执行逻辑隐藏在订单中
```

这种方式带来的问题：

| 问题 | 影响 |
|------|------|
| **混淆的职责** | 选资产、生成信号、构建持仓、风险控制全混在一起，难以理解 |
| **无法单独测试** | Alpha 模型改了，无法独立测试是否变好，因为持仓、风险控制也可能影响结果 |
| **代码重用困难** | 某个模块想用在另一个策略？得复制粘贴整个 `OnData()` |
| **团队协作低效** | Alpha 研究员和风险管理员互相踩脚，修改同一个方法 |
| **难以对比测试** | 想对比两个 Alpha 模型？要么写两套完整策略，要么频繁修改代码 |
| **扩展性差** | 添加新功能（如动态止损）需要改核心逻辑 |

### Framework 的解决方案

LEAN 的 Algorithm Framework 将策略**分解为 5 个解耦的模块**，每个模块职责单一、接口清晰：

```
Asset Universe → Alpha Signals → Portfolio Targets → Risk Filter → Orders
```

这就像前端发展的历程：

| 前端演进 | 量化类比 |
|--------|--------|
| **jQuery 时代** (混淆逻辑) | 单体 `OnData()` (全部逻辑混淆) |
| **选择器问题** | 资产选择不清晰导致 look-ahead bias |
| **事件处理混乱** | 信号生成与风险控制耦合 |
| **难以复用** | 难以跨项目重用 |
| **React 时代** (组件化) | Algorithm Framework (模块化) |
| **声明式组件树** | 声明式管道流水线 |
| **Props 接口清晰** | Module 接口标准化 |
| **易于测试** | 每个模块独立可测 |
| **易于复用** | 模块市场与共享库 |

### Framework 的核心优势

1. **单一职责原则**：每个模块只做一件事
2. **可测性**：Alpha 模型改了？单独测试它
3. **可复用性**：同一个 Alpha 模型配合不同的风险模型，轻松对比
4. **可维护性**：代码清晰，新人容易上手
5. **协作高效**：不同团队可以独立开发不同模块
6. **灵活扩展**：添加新的 Alpha、Risk、Execution 模型只需实现标准接口

---

## 五层流水线总览

LEAN 将策略执行过程分为 5 个阶段，形成一条完整的数据处理流水线：

```
┌─────────────────────────────────────────────────────────────────┐
│                      LEAN Algorithm Framework                    │
│                         数据流水线总览                             │
└─────────────────────────────────────────────────────────────────┘

                              Market Data
                                  │
                                  ▼
    ┌───────────────────────────────────────────────────────────┐
    │  1. UNIVERSE SELECTION MODEL                              │
    │     "我们要交易哪些资产？"                                    │
    │  Input:  All Market Data                                   │
    │  Output: Asset[] (e.g., [SPY, QQQ, IWM])                 │
    └───────────────────────────────────────────────────────────┘
                                  │
                                  ▼
    ┌───────────────────────────────────────────────────────────┐
    │  2. ALPHA MODEL                                            │
    │     "这些资产短期内会涨还是跌？"                                  │
    │  Input:  Asset[] + Price/Volume Data                      │
    │  Output: Insight[] (Direction, Confidence, Period, etc)   │
    └───────────────────────────────────────────────────────────┘
                                  │
                                  ▼
    ┌───────────────────────────────────────────────────────────┐
    │  3. PORTFOLIO CONSTRUCTION MODEL                           │
    │     "如果我们相信这些信号，应该怎样分配资金？"                      │
    │  Input:  Insight[]                                        │
    │  Output: PortfolioTarget[] (Symbol + Quantity/Weight)     │
    └───────────────────────────────────────────────────────────┘
                                  │
                                  ▼
    ┌───────────────────────────────────────────────────────────┐
    │  4. RISK MANAGEMENT MODEL                                  │
    │     "这些目标持仓会不会冒太大风险？"                                |
    │  Input:  PortfolioTarget[]                                │
    │  Output: PortfolioTarget[] (可能被调整或删除)                 │
    └───────────────────────────────────────────────────────────┘
                                  │
                                  ▼
    ┌───────────────────────────────────────────────────────────┐
    │  5. EXECUTION MODEL                                        │
    │     "怎样最好地执行这些订单？"                                    │
    │  Input:  PortfolioTarget[]                                │
    │  Output: Orders[] (立即执行、分批、VWAP等)                    │
    └───────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                            Order Submission
                           (Brokerage/Exchange)
```

### 数据类型流转

每个环节的输入和输出数据类型都是标准化的：

| 阶段 | 输入 | 输出 | 用途 |
|------|------|------|------|
| Universe Selection | Market Events | `Symbol[]` | 确定交易范围 |
| Alpha | Market Events | `Insight[]` | 生成交易信号 |
| Portfolio Construction | `Insight[]` | `PortfolioTarget[]` | 计算目标权重 |
| Risk Management | `PortfolioTarget[]` | `PortfolioTarget[]` (adjusted) | 风险过滤 |
| Execution | `PortfolioTarget[]` | `Order[]` | 发送订单 |

### 阶段间的通信机制

各模块通过标准接口通信，LEAN 框架内核负责协调：

```python
# 伪代码：框架的调度逻辑
def OnDataEvent(data):
    # 1. Universe Selection
    symbols = universe_model.SelectCoarse(all_stocks)
    symbols = universe_model.SelectFine(fine_data)

    # 2. Alpha Model
    insights = alpha_model.Update(symbols, data)

    # 3. Portfolio Construction
    targets = portfolio_model.CreateTargets(insights)

    # 4. Risk Management
    targets = risk_model.ManageRisk(targets)

    # 5. Execution
    orders = execution_model.Execute(targets)

    # Submit orders to market
    for order in orders:
        submit_order(order)
```

这就像工厂流水线，每个工人（模块）只关心自己的工作，产品（数据）自动流向下一站。

---

## Universe Selection Model 详解

### 设计目标

Universe Selection 的核心目标：**确定策略的交易范围**

```
Universe Selection = "我们要交易哪些资产？"
```

在策略开发中，资产选择至关重要：

- **太广**（交易所有股票）：信号被噪音淹没，交易成本高
- **太窄**（手工选择几只股票）：样本量太小，容易过拟合，容易产生 look-ahead bias
- **有偏**（只选高价格、高成交量股票）：影响回测结果的真实性

Universe Selection 通过**系统化、可重复**的规则选择资产，确保：

1. 样本足够大（统计显著性）
2. 选择过程客观（可重复性）
3. 避免 look-ahead bias（未来信息泄露）

### 前端类比

```
Universe Selection ≈ 路由级代码分割

React 应用中，哪些组件要加载取决于路由：
- 登录页只需要登录组件 + 表单组件
- 仪表板页需要仪表板组件 + 图表库
- 未必要加载所有代码

Universe Selection 也是：
- 这个交易日要分析哪些股票？
- 不需要分析全部 5000 只股票
- 只分析符合条件的子集
```

### 内置 Universe 选择模型

LEAN 提供了多种开箱即用的 Universe 模型：

#### 1. ManualUniverseSelectionModel

**用途**：手工指定要交易的资产

```python
from AlgorithmImports import *

class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 手动指定：只交易这三只 ETF
        self.SetUniverseSelection(
            ManualUniverseSelectionModel([
                Symbol.Create('SPY', AssetClass.Equity, Market.USA),
                Symbol.Create('QQQ', AssetClass.Equity, Market.USA),
                Symbol.Create('IWM', AssetClass.Equity, Market.USA)
            ])
        )
```

**优点**：简单明确
**缺点**：样本小，容易过拟合

#### 2. CoarseFundamentalUniverseSelectionModel

**用途**：基于市场数据（价格、成交量）动态筛选股票

特点：速度快（只需日线数据），适合大规模筛选

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

    def CoarseFilterFunction(self, coarse):
        """
        输入：CoarseFundamental[] 包含所有 US 股票的日线数据
        输出：Symbol[] 筛选后的股票列表
        """
        # 筛选条件：
        # 1. 价格 > $10（避免便士股）
        # 2. 日交易额 (Price * Volume) > $1M（确保流动性）
        # 3. 有价格数据
        filtered = [
            c for c in coarse
            if c.Price > 10
            and c.Volume * c.Price > 1_000_000
            and c.HasFundamentalData
        ]

        # 按成交额排序，取前 100 只
        return sorted(
            filtered,
            key=lambda c: c.Volume * c.Price,
            reverse=True
        )[:100]
```

**执行频率**：每个交易日调用一次（底层数据是日线）
**速度**：快（只涉及 OHLCV 数据）
**精度**：中等（没有基本面数据）

#### 3. FineFundamentalUniverseSelectionModel

**用途**：基于上市公司基本面数据筛选（需要先用 Coarse 筛选以提高效率）

特点：数据详细（财务报表），但速度慢

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetUniverseSelection(
            FineFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction,
                self.FineFilterFunction
            )
        )

    def CoarseFilterFunction(self, coarse):
        """
        第一阶段：粗筛
        从 5000+ 股票中筛到 500 只，减少 Fine 处理的数据量
        """
        return [
            c.Symbol for c in coarse
            if c.Price > 10
            and c.Volume * c.Price > 1_000_000
        ][:500]

    def FineFilterFunction(self, fine):
        """
        第二阶段：细筛
        从 500 只中筛到最终的交易列表
        可以访问财务数据如 P/E、负债率等
        """
        # 访问基本面数据
        filtered = [
            f for f in fine
            if f.ValuationRatios.PERatio > 0  # P/E 有效
            and f.ValuationRatios.PERatio < 30  # P/E < 30
            and f.OperationRatios.ROE.Value > 0.1  # ROE > 10%
            and f.CompanyReference.IndustryTemplateCode == 'TECH'  # 科技股
        ]

        return sorted(
            filtered,
            key=lambda f: f.ValuationRatios.PERatio
        )[:50]
```

**执行频率**：每个交易日调用（底层是日线）
**速度**：慢（需要处理财务数据）
**精度**：高（有完整基本面数据）

**最佳实践**：先 Coarse 后 Fine，两阶段筛选

#### 4. ETFConstituentsUniverseSelectionModel

**用途**：自动跟踪 ETF 成分股变化

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 自动跟踪 SPY 的所有成分股
        self.SetUniverseSelection(
            ETFConstituentsUniverseSelectionModel('SPY')
        )
```

**应用场景**：投资 ETF 而非个股时特别有用

#### 5. OptionUniverseSelectionModel

**用途**：选择期权链

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetUniverseSelection(
            OptionUniverseSelectionModel(
                [Symbol.Create('SPY', AssetClass.Equity, Market.USA)]
            )
        )
```

### 自定义 Universe 选择

如果内置模型不满足需求，可以自定义：

```python
from AlgorithmImports import *

class CustomUniverseSelection(Universe):
    """自定义 Universe 选择器"""

    def __init__(self, symbol):
        self.symbol = symbol

    def SelectCoarse(self, coarse):
        """自定义粗筛逻辑"""
        return [
            c.Symbol for c in coarse
            if c.Price > 20  # 自定义条件
            and c.Volume > 1_000_000
        ][:100]

    def SelectFine(self, fine):
        """自定义细筛逻辑"""
        return [
            f.Symbol for f in fine
            if hasattr(f, 'ValuationRatios')
            and f.ValuationRatios.PERatio < 25
        ][:50]


class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CustomCoarseFilter
            )
        )

    def CustomCoarseFilter(self, coarse):
        # 完全自定义的筛选逻辑
        return [
            c.Symbol for c in coarse
            if c.Price > 15
            and c.Volume * c.Price > 2_000_000
            and not self.IsBlacklisted(c.Symbol)
        ][:100]

    def IsBlacklisted(self, symbol):
        # 某些股票的黑名单检查
        return symbol.Value in ['XYZ', 'ABC']
```

### 完整示例：Top 100 Dollar Volume

```python
from AlgorithmImports import *

class DollarVolumeUniverse(QCAlgorithm):
    """
    例子：选择成交额最高的 100 只股票
    目的：流动性好，容易进出
    """

    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)
        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

        # 后续会在 Alpha Model 中生成信号
        self.SetAlpha(BasicAlphaModel())
        self.SetPortfolioConstruction(EqualWeightPortfolioConstructionModel())

    def CoarseFilterFunction(self, coarse):
        """
        核心筛选逻辑：
        1. 价格 > $10（避免便士股）
        2. 有基本面数据
        3. 按成交额排序，取前 100
        """
        # 计算日成交额
        filtered = [
            c for c in coarse
            if c.Price > 10
            and c.HasFundamentalData
            and c.Volume * c.Price > 0  # 有成交
        ]

        # 按成交额(美元)排序
        sorted_by_volume = sorted(
            filtered,
            key=lambda c: c.Volume * c.Price,
            reverse=True
        )

        # 取前 100 只，但不超过实际有效数量
        return [c.Symbol for c in sorted_by_volume[:100]]
```

### Universe 选择的关键参数

| 参数 | 含义 | 典型值 |
|------|------|--------|
| **Min Price** | 最低价格（避免便士股） | $5-10 |
| **Min Volume** | 最低日均成交量（流动性） | 1M-5M shares |
| **Min Dollar Volume** | 最低日成交额（成本优化） | $1M-5M |
| **Max Stock Count** | 最多交易多少只股票 | 50-500 |
| **Rebalance Frequency** | 多久重新选择一次 | 每天/每周/每月 |

---

## Alpha Model 详解

### Alpha 的本质

Alpha Model 是策略的**核心大脑**：

```
Alpha = "根据市场信息，预测资产短期内会涨还是跌？"
```

如果 Universe Selection 确定了"我们交易什么"，那么 Alpha Model 就回答了"我们怎样交易"。

### 输出格式：Insight 对象

Alpha 模型的输出是 `Insight` 对象，包含以下信息：

```python
class Insight:
    """Alpha Model 的标准输出"""

    def __init__(self, symbol, period, direction, magnitude, confidence, weight=None):
        self.Symbol = symbol                    # 交易的资产
        self.Period = period                    # 信号有效期（如 timedelta(hours=1)）
        self.Direction = direction              # InsightDirection.Up/Down/Flat
        self.Magnitude = magnitude              # 预期涨跌幅（0.05 = 5%）
        self.Confidence = confidence            # 信心（0-1，如 0.8）
        self.Weight = weight                    # 权重（后续用于持仓分配）
        self.SourceModel = 'MyAlphaModel'       # 信号来源
        self.Id = <UUID>                        # 信号ID
        self.GeneratedTime = <datetime>         # 生成时间
```

### 关键字段详解

| 字段 | 含义 | 例子 |
|------|------|------|
| **Direction** | 预测方向 | Up（看多）/ Down（看空）/ Flat（中立） |
| **Magnitude** | 预期收益率 | 0.05 表示预期涨 5% |
| **Confidence** | 信心分数 | 0.8 表示 80% 确信 |
| **Period** | 信号有效期 | timedelta(hours=1) 表示 1 小时内有效 |
| **Weight** | 权重（可选） | 可在 Alpha 中直接指定权重 |

```python
# 例子：
insight = Insight(
    symbol=Symbol.Create('AAPL', AssetClass.Equity, Market.USA),
    period=timedelta(hours=1),
    direction=InsightDirection.Up,      # 看多
    magnitude=0.05,                     # 预期涨 5%
    confidence=0.8,                     # 80% 信心
    weight=0.5                          # 权重 50%
)
```

### 前端类比

```
Alpha Model ≈ 状态管理选择器 (Redux Selector)

Redux 中的选择器：
const selectUserAge = (state) => state.user.age;
const selectIsAdult = (state) => state.user.age >= 18;
const selectCanVote = createSelector(
  [selectIsAdult],
  (isAdult) => isAdult && state.user.registeredToVote
);

Alpha Model 也是：
def Update(self, data):
    # 原始数据（market data）
    # ↓
    # 计算指标（EMA, MACD, RSI）
    # ↓
    # 生成信号（Insight）
```

### 内置 Alpha 模型

#### 1. EmaCrossAlphaModel

**原理**：快速 EMA 穿过慢速 EMA 时生成信号

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetAlpha(EmaCrossAlphaModel(12, 26))  # 参数：快EMA周期，慢EMA周期
```

**逻辑**：
- 快 EMA > 慢 EMA → 看多（Direction.Up）
- 快 EMA < 慢 EMA → 看空（Direction.Down）

#### 2. MacdAlphaModel

**原理**：MACD（Moving Average Convergence Divergence）信号

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetAlpha(MacdAlphaModel(12, 26, 9))  # 快线周期, 慢线周期, 信号线周期
```

#### 3. RsiAlphaModel

**原理**：相对强弱指数（RSI）

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetAlpha(RsiAlphaModel(14, 30, 70))  # 周期, 超卖阈值, 超买阈值
```

**逻辑**：
- RSI < 30（超卖） → 看多
- RSI > 70（超买） → 看空

### 完整示例：自定义 Alpha Model

```python
from AlgorithmImports import *

class RSI_MACD_AlphaModel(AlphaModel):
    """
    组合使用 RSI 和 MACD 的自定义 Alpha Model

    信号生成规则：
    - 当 RSI < 30 AND MACD 正向穿越零线 → 强烈看多
    - 当 RSI > 70 AND MACD 负向穿越零线 → 强烈看空
    - 否则 → 中立
    """

    def __init__(self, algorithm):
        self.algorithm = algorithm
        self.symbols = []

        # 为每个符号维护指标
        self.rsi_dict = {}
        self.macd_dict = {}

    def Update(self, algorithm, insights):
        """
        每个数据事件触发时调用

        输入：algorithm（算法实例）
        输出：insights 列表（追加新信号）
        """
        # 获取当前交易的所有资产
        insights = []

        for symbol in algorithm.ActiveSecurities.Keys:
            if symbol not in self.rsi_dict:
                # 首次见到该符号，初始化指标
                self._InitializeIndicators(symbol)

            # 获取最新价格
            if not algorithm.Securities[symbol].HasData:
                continue

            price = algorithm.Securities[symbol].Close

            # 更新指标
            self.rsi_dict[symbol].Update(
                algorithm.Time,
                price
            )
            self.macd_dict[symbol].Update(
                algorithm.Time,
                price
            )

            # 指标未ready，跳过
            if not self.rsi_dict[symbol].IsReady or not self.macd_dict[symbol].IsReady:
                continue

            # 生成信号
            rsi = self.rsi_dict[symbol].Current.Value
            macd = self.macd_dict[symbol].Current.Value
            macd_signal = self.macd_dict[symbol].Signal.Current.Value

            # 信号生成逻辑
            direction = InsightDirection.Flat
            confidence = 0.5

            # 看多条件：RSI 超卖 + MACD 正穿
            if rsi < 30 and macd > macd_signal:
                direction = InsightDirection.Up
                confidence = 0.8

            # 看空条件：RSI 超买 + MACD 负穿
            elif rsi > 70 and macd < macd_signal:
                direction = InsightDirection.Down
                confidence = 0.8

            # 中等信号：只有一个条件满足
            elif rsi < 40 and macd > macd_signal:
                direction = InsightDirection.Up
                confidence = 0.6
            elif rsi > 60 and macd < macd_signal:
                direction = InsightDirection.Down
                confidence = 0.6

            # 创建 Insight
            if direction != InsightDirection.Flat:
                insight = Insight(
                    symbol=symbol,
                    period=timedelta(hours=1),           # 1小时有效
                    direction=direction,
                    magnitude=0.02,                     # 预期 2% 涨跌
                    confidence=confidence,
                    weight=None                         # 权重由后续 Portfolio Model 计算
                )
                insights.append(insight)

        return insights

    def _InitializeIndicators(self, symbol):
        """为新符号初始化技术指标"""
        self.rsi_dict[symbol] = RelativeStrengthIndex(14)
        self.macd_dict[symbol] = MacdCrossAlertModel(12, 26, 9)

    def OnSecurityAdded(self, algorithm, security):
        """当新资产加入交易范围时调用"""
        self.symbols.append(security.Symbol)

    def OnSecurityRemoved(self, algorithm, security):
        """当资产离开交易范围时调用"""
        self.symbols.remove(security.Symbol)
        if security.Symbol in self.rsi_dict:
            del self.rsi_dict[security.Symbol]
        if security.Symbol in self.macd_dict:
            del self.macd_dict[security.Symbol]


class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

        # 使用自定义 Alpha Model
        self.SetAlpha(RSI_MACD_AlphaModel(self))

        self.SetPortfolioConstruction(
            EqualWeightPortfolioConstructionModel()
        )

    def CoarseFilterFunction(self, coarse):
        return [
            c.Symbol for c in coarse
            if c.Price > 10 and c.Volume * c.Price > 1_000_000
        ][:100]
```

### 组合多个 Alpha 模型

实战中常常需要组合多个 Alpha 模型，LEAN 提供了 `CompositeAlphaModel`：

```python
from AlgorithmImports import *

class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 组合三个 Alpha 模型
        self.SetAlpha(CompositeAlphaModel([
            EmaCrossAlphaModel(12, 26),           # EMA 交叉
            MacdAlphaModel(12, 26, 9),            # MACD
            RsiAlphaModel(14, 30, 70),            # RSI
        ]))
```

**运作方式**：
1. 每个 Alpha 模型独立生成信号
2. 框架合并所有信号
3. 同一符号的多个信号会被组合（权重求和或投票）

### Alpha 模型开发最佳实践

1. **参数化**：将魔数提出来，便于优化和回测
   ```python
   def __init__(self, fast_period=12, slow_period=26):
       self.fast_period = fast_period
       self.slow_period = slow_period
   ```

2. **清晰的信号**：Insight 的 Direction 应该明确
   ```python
   # ✓ 好：明确的方向和信心
   Insight(symbol, period, InsightDirection.Up, magnitude=0.05, confidence=0.8)

   # ✗ 差：模糊的信号
   Insight(symbol, period, InsightDirection.Up, magnitude=0.001, confidence=0.51)
   ```

3. **合理的周期**：Insight 的 Period 应该与交易周期匹配
   ```python
   # 日线策略
   period = timedelta(days=1)

   # 小时线策略
   period = timedelta(hours=1)
   ```

4. **避免 Look-Ahead Bias**：不要在当前 bar 关闭前使用该 bar 的信息
   ```python
   # ✗ 差：在 OnData 中直接用当前 close
   def OnData(self, data):
       if current_price > threshold:  # ← Look-ahead bias!

   # ✓ 好：只用前一根 bar 的数据
   def Update(self, algorithm, insights):
       for symbol in symbols:
           history = algorithm.History(symbol, 2, Resolution.Daily)
           prev_close = history.iloc[-2]['close']
   ```

---

## Portfolio Construction Model 详解

### 设计目标

如果 Alpha Model 告诉我们"AAPL 会涨，MSFT 会跌"，Portfolio Construction 则回答：

```
Portfolio Construction = "相信这些信号，我们应该如何分配资金？"
```

具体来说，Portfolio Construction 需要：

1. **将 Insight 转换为持仓目标**：AAPL 信号强，分配 20% 的资本；MSFT 信号弱，只分配 5%
2. **处理多重信号冲突**：同一个资产既有买信号又有卖信号怎么办？
3. **确保权重合理**：所有持仓总和不超过 100%（或考虑杠杆）
4. **优化资本使用**：在回报和风险间平衡

### 输出格式：PortfolioTarget 对象

```python
class PortfolioTarget:
    """Portfolio Construction 的标准输出"""

    def __init__(self, symbol, quantity_or_weight):
        self.Symbol = symbol              # 交易的资产
        self.Quantity = quantity_or_weight  # 目标持仓数量（或权重）
```

### 前端类比

```
Portfolio Construction ≈ 响应式布局引擎

CSS Flexbox 将 "这些组件很重要" 转换为 "为它们分配像素"：
- 宽屏：分配 50% 给主内容，25% 给侧边栏，25% 给广告
- 窄屏：堆叠所有组件，每个 100% 宽度

Portfolio Construction 也是：
- 这个信号重要（confidence 高），分配 20% 资本
- 那个信号一般（confidence 中），分配 10% 资本
- 通过 flex-grow 类似的机制实现权重分配
```

### 内置 Portfolio Construction 模型

#### 1. EqualWeightPortfolioConstructionModel

**规则**：所有 Insight 信号平均分配资本

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetPortfolioConstruction(
            EqualWeightPortfolioConstructionModel()
        )
```

**例子**：
- 有 3 个看多信号（AAPL, MSFT, GOOG）
- 每个分配 1/3 的可用资本
- 结果：AAPL 买 33%, MSFT 买 33%, GOOG 买 33%

**优点**：简单、易懂
**缺点**：忽视信号强度（confidence）

#### 2. InsightWeightingPortfolioConstructionModel

**规则**：根据 Insight 的权重和信心分配资本

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetPortfolioConstruction(
            InsightWeightingPortfolioConstructionModel()
        )
```

**例子**：
```python
# Alpha Model 生成：
insights = [
    Insight(AAPL, ..., direction=Up, confidence=0.9, weight=0.5),
    Insight(MSFT, ..., direction=Up, confidence=0.6, weight=0.3),
    Insight(GOOG, ..., direction=Down, confidence=0.7, weight=0.2),
]

# Portfolio Construction 根据 weight 和 confidence 分配：
# AAPL: 0.5 * 0.9 = 0.45 (最强)
# MSFT: 0.3 * 0.6 = 0.18 (中等)
# GOOG: 0.2 * 0.7 = 0.14 (看空，但较弱)
# 总和: 0.45 + 0.18 + 0.14 = 0.77 → 标准化到 100%
```

**优点**：考虑信号强度
**缺点**：不优化风险

#### 3. MeanVarianceOptimizationPortfolioConstructionModel

**规则**：基于历史收益和波动率优化，最大化 Sharpe 比率

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetPortfolioConstruction(
            MeanVarianceOptimizationPortfolioConstructionModel(
                lookback=63,              # 用过去 63 天数据计算协方差
                resolution=Resolution.Daily,
                target_portfolio_value=1  # 满仓投资
            )
        )
```

**原理**：
1. 计算每个资产的期望收益率（基于 Insight magnitude）
2. 计算资产间的协方差矩阵
3. 求解优化问题：最大化 (E[R] - rf) / σ（Sharpe）
4. 输出最优权重

**优点**：考虑风险和收益，数学优雅
**缺点**：计算复杂，对参数敏感

#### 4. BlackLittermanOptimizationPortfolioConstructionModel

**规则**：融合市场均衡观点和交易员观点的贝叶斯优化

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetPortfolioConstruction(
            BlackLittermanOptimizationPortfolioConstructionModel()
        )
```

**原理**：
- 起点：市场的隐含预期收益（均衡）
- 融入：我们的 Alpha 信号（观点）
- 结果：调整后的预期收益 → 最优权重

**优点**：融合市场共识与独特观点，稳定性好
**缺点**：参数最复杂

### 完整示例：自定义 Portfolio Construction

```python
from AlgorithmImports import *

class RiskPaityPortfolioConstructionModel(PortfolioConstructionModel):
    """
    自定义 Portfolio Construction Model

    风险平价（Risk Parity）策略：
    根据波动率反向加权，确保每个持仓贡献相同的风险
    """

    def __init__(self, lookback=252):
        self.lookback = lookback
        self.volatility_cache = {}

    def CreateTargets(self, algorithm, insights):
        """
        输入：insights (Insight[] 从 Alpha Model 来)
        输出：targets (PortfolioTarget[])
        """
        targets = []

        if not insights:
            return targets

        # 分组：看多、看空、中立
        buy_insights = [i for i in insights if i.Direction == InsightDirection.Up]
        sell_insights = [i for i in insights if i.Direction == InsightDirection.Down]

        # 计算资本分配
        total_insights = len(buy_insights) + len(sell_insights)
        if total_insights == 0:
            return targets

        # 为每个持仓计算波动率
        volatilities = {}
        for insight in insights:
            symbol = insight.Symbol

            # 从缓存获取或计算波动率
            if symbol not in self.volatility_cache:
                vol = self._CalculateVolatility(algorithm, symbol)
                self.volatility_cache[symbol] = vol

            volatilities[symbol] = self.volatility_cache[symbol]

        # 风险平价：权重 = 1/σ / Σ(1/σ)
        inv_vols = {s: 1.0 / vol if vol > 0 else 0 for s, vol in volatilities.items()}
        total_inv_vol = sum(inv_vols.values())

        if total_inv_vol == 0:
            return targets

        # 计算资本分配（相对于投资组合价值）
        portfolio_value = algorithm.Portfolio.TotalPortfolioValue

        for insight in buy_insights:
            symbol = insight.Symbol
            weight = inv_vols[symbol] / total_inv_vol
            quantity = int(weight * portfolio_value /
                          algorithm.Securities[symbol].Close)

            targets.append(PortfolioTarget(symbol, quantity))

        for insight in sell_insights:
            symbol = insight.Symbol
            weight = inv_vols[symbol] / total_inv_vol
            quantity = -int(weight * portfolio_value /
                           algorithm.Securities[symbol].Close)

            targets.append(PortfolioTarget(symbol, quantity))

        return targets

    def _CalculateVolatility(self, algorithm, symbol):
        """计算资产的历史波动率"""
        history = algorithm.History(
            symbol,
            self.lookback,
            Resolution.Daily
        )

        if len(history) < 2:
            return 0.01  # 默认 1% 波动率

        returns = history['close'].pct_change().dropna()
        return returns.std()

    def OnSecurityAdded(self, algorithm, security):
        """新资产加入时清除缓存"""
        self.volatility_cache.clear()

    def OnSecurityRemoved(self, algorithm, security):
        """资产移除时清除缓存"""
        if security.Symbol in self.volatility_cache:
            del self.volatility_cache[security.Symbol]


class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

        self.SetAlpha(EmaCrossAlphaModel(12, 26))

        # 使用自定义风险平价 Portfolio Construction
        self.SetPortfolioConstruction(
            RiskPaityPortfolioConstructionModel(lookback=252)
        )

    def CoarseFilterFunction(self, coarse):
        return [
            c.Symbol for c in coarse
            if c.Price > 10 and c.Volume * c.Price > 1_000_000
        ][:50]
```

### Rebalancing（再平衡）

Portfolio Construction 需要决定**何时重新计算权重**：

```python
# 默认：OnSecurityChanges（有资产加入/退出时）
self.SetPortfolioConstruction(
    EqualWeightPortfolioConstructionModel()
)

# 自定义再平衡频率
class CustomRebalancingAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 每个月重新平衡一次
        self.AddRiskManagement(
            MaximumDrawdownPercentPerSecurity(0.05)
        )

        # Insight 过期自动重新平衡
        # (由 framework 自动处理)
```

---

## Risk Management Model 详解

### 设计目标

Risk Management 是策略的**保护盾牌**：

```
Risk Management = "这些目标持仓会冒太大风险吗？我们应该调整吗？"
```

即使 Alpha Model 很精准、Portfolio Construction 算权重很合理，也需要风险控制：

1. **单一持仓风险过大**：某只股票占了 30%，跌 10% 就损失 3%
2. **行业集中风险**：全买科技股，科技股大跌整体巨亏
3. **回撤过大**：历史回撤 50%，不能接受
4. **波动率突增**：黑天鹅事件，需要紧急减仓

Risk Management 在 Portfolio Construction 之后，验证目标持仓是否合理，必要时调整。

### 前端类比

```
Risk Management ≈ 表单验证中间件

在提交数据前，验证层会检查：
- 用户名长度是否合法？
- 邮箱格式是否正确？
- 密码强度是否足够？

如果不通过，要么拒绝提交，要么返回错误信息修改。

Risk Management 也是：
- 这个持仓是否会导致单一股票超过 5% 敞口？
- 是否会导致某行业超过 30% 敞口？
- 历史最大回撤是否会超过 20%？

如果不通过，减仓或取消这个持仓。
```

### 内置 Risk Management 模型

#### 1. MaximumDrawdownPercentPerSecurity

**规则**：限制单个资产的最大回撤比例

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.AddRiskManagement(
            MaximumDrawdownPercentPerSecurity(0.05)  # 单股最大回撤 5%
        )
```

**逻辑**：
```python
for security in holdings:
    unrealized_loss = (current_price - entry_price) / entry_price
    if unrealized_loss < -0.05:  # 跌超 5%
        sell_position(security)   # 平仓
```

#### 2. MaximumSectorExposureRiskManagementModel

**规则**：限制单个行业的最大敞口

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.AddRiskManagement(
            MaximumSectorExposureRiskManagementModel(0.3)  # 单个行业最多 30%
        )
```

#### 3. TrailingStopRiskManagementModel

**规则**：使用追踪止损

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.AddRiskManagement(
            TrailingStopRiskManagementModel(0.05)  # 距离最高价 5% 止损
        )
```

#### 4. MaximumUnrealizedProfitPercentPerSecurity

**规则**：限制未实现利润（防止过度乐观）

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.AddRiskManagement(
            MaximumUnrealizedProfitPercentPerSecurity(0.20)  # 未实现利润上限 20%
        )
```

**奇怪？为什么要限制利润？**

原因：防止单一持仓因为暂时涨幅过大而占用过多资本，导致结构性问题。

### 完整示例：自定义 Risk Management

```python
from AlgorithmImports import *

class AdaptiveStopLossRiskManagement(RiskManagementModel):
    """
    自定义风险管理模型：自适应止损

    根据波动率动态调整止损位置：
    - 低波动率股票：止损 2%
    - 高波动率股票：止损 5%
    """

    def __init__(self, lookback=30):
        self.lookback = lookback
        self.volatility_cache = {}

    def ManageRisk(self, algorithm, targets):
        """
        输入：targets (从 Portfolio Construction 来的目标持仓)
        输出：targets (被调整后的目标持仓)
        """
        targets_to_remove = []

        for target in targets:
            symbol = target.Symbol
            security = algorithm.Securities[symbol]

            if not security.HasData:
                continue

            # 计算波动率
            if symbol not in self.volatility_cache:
                vol = self._CalculateVolatility(algorithm, symbol)
                self.volatility_cache[symbol] = vol

            volatility = self.volatility_cache[symbol]

            # 根据波动率确定止损幅度
            if volatility > 0.02:  # 高波动
                stop_loss_pct = 0.05  # 5% 止损
            elif volatility > 0.01:  # 中等波动
                stop_loss_pct = 0.03  # 3% 止损
            else:  # 低波动
                stop_loss_pct = 0.02  # 2% 止损

            # 检查当前持仓是否触发止损
            if symbol in algorithm.Portfolio:
                holding = algorithm.Portfolio[symbol]

                if holding.Quantity > 0:  # 多头持仓
                    unrealized_loss = holding.UnrealizedProfit / holding.HoldingsValue

                    if unrealized_loss < -stop_loss_pct:
                        # 触发止损，移除这个目标
                        targets_to_remove.append(target)

                        # 发出平仓订单
                        algorithm.SetHoldings(symbol, 0)

        # 返回调整后的目标
        return [t for t in targets if t not in targets_to_remove]

    def _CalculateVolatility(self, algorithm, symbol):
        """计算历史波动率"""
        history = algorithm.History(symbol, self.lookback, Resolution.Daily)

        if len(history) < 2:
            return 0.01

        returns = history['close'].pct_change().dropna()
        return returns.std()

    def OnSecurityAdded(self, algorithm, security):
        """新资产加入时清除缓存"""
        self.volatility_cache.clear()

    def OnSecurityRemoved(self, algorithm, security):
        """资产移除时清除缓存"""
        if security.Symbol in self.volatility_cache:
            del self.volatility_cache[security.Symbol]


class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

        self.SetAlpha(EmaCrossAlphaModel(12, 26))
        self.SetPortfolioConstruction(EqualWeightPortfolioConstructionModel())

        # 添加自定义风险管理
        self.AddRiskManagement(AdaptiveStopLossRiskManagement(lookback=30))

    def CoarseFilterFunction(self, coarse):
        return [
            c.Symbol for c in coarse
            if c.Price > 10 and c.Volume * c.Price > 1_000_000
        ][:50]
```

### Risk 模型的组合

多个 Risk 模型可以叠加，框架会依次应用：

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        # 多层防护
        self.AddRiskManagement(
            MaximumDrawdownPercentPerSecurity(0.05)    # 单股 5% 止损
        )
        self.AddRiskManagement(
            MaximumSectorExposureRiskManagementModel(0.3)  # 行业 30% 上限
        )
        self.AddRiskManagement(
            TrailingStopRiskManagementModel(0.10)      # 10% 追踪止损
        )
```

**执行顺序**：各模型依次检查，都通过才执行

---

## Execution Model 详解

### 设计目标

Execution Model 回答最后一个问题：

```
Execution = "我们已经决定了买什么，现在应该怎样执行这些订单？"
```

为什么需要专门的 Execution 模型？

1. **市场冲击（Market Impact）**：一次性买入 100 万股，市场可能会上涨，我们会以更高的价格成交
2. **滑点（Slippage）**：下单价格和成交价格之间的差异
3. **部分成交（Partial Fill）**：订单可能无法全部成交
4. **时间问题**：什么时候下单？集合竞价还是盘中？

不同的 Execution 模型对回测和实盘结果影响巨大。

### 前端类比

```
Execution Model ≈ API 调用策略

前端调用 API 有多种方式：
1. Immediate：立即调用，可能造成网络拥塞（类似市场冲击）
2. Debounced：延迟调用，避免频繁更新
3. Batched：批量调用，减少网络开销
4. RequestIdleCallback：空闲时调用，不阻塞主线程

Execution Model 也是：
1. ImmediateExecutionModel：立即发单
2. VolumeWeightedAveragePriceExecutionModel (VWAP)：追踪日均价格
3. StandardDeviationExecutionModel：基于波动率调整
```

### 内置 Execution 模型

#### 1. ImmediateExecutionModel

**规则**：立即以市价执行所有订单

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetExecution(ImmediateExecutionModel())
```

**特点**：
- 最简单
- 适合流动性好的资产（如主要指数）
- 市场冲击和滑点可能较大

#### 2. VolumeWeightedAveragePriceExecutionModel

**规则**：分散执行，追踪日均成交量加权平均价（VWAP）

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetExecution(
            VolumeWeightedAveragePriceExecutionModel()
        )
```

**原理**：
- 历史成交量分析：这只股票今天可能有 500 万股成交
- 分散下单：如果我们要买 50 万股（10%），分散在全天多个时间点下单
- 目标：以接近日均价格（VWAP）成交，而不是推高价格

**适用场景**：大额订单

#### 3. StandardDeviationExecutionModel

**规则**：根据价格波动率动态调整执行节奏

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetExecution(
            StandardDeviationExecutionModel(
                resolution=Resolution.Daily,
                offset=2  # 2 个标准差
            )
        )
```

**逻辑**：
- 价格波动大 → 谨慎执行，慢慢下单（等待价格稳定）
- 价格波动小 → 快速执行，一次性下单

### 完整示例：自定义 Execution

```python
from AlgorithmImports import *
from datetime import timedelta

class SmartExecutionModel(ExecutionModel):
    """
    自定义执行模型：智能执行

    根据订单大小和市场流动性智能分配执行时机：
    - 小订单（< 占日成交 1%）：立即执行
    - 中等订单（1%-5%）：分 5 份，间隔执行
    - 大订单（> 5%）：分 10 份，全天执行
    """

    def __init__(self):
        self.pending_orders = {}  # 追踪待执行订单

    def Execute(self, algorithm, targets):
        """
        输入：targets (从 Risk Management 来的最终持仓目标)
        输出：订单被提交到市场
        """
        # 第一步：计算当前持仓 vs 目标持仓
        for target in targets:
            symbol = target.Symbol
            security = algorithm.Securities[symbol]

            if not security.HasData:
                continue

            # 当前数量
            current_quantity = algorithm.Portfolio[symbol].Quantity

            # 目标数量
            target_quantity = target.Quantity

            # 需要变化的数量
            quantity_diff = target_quantity - current_quantity

            if quantity_diff == 0:
                continue  # 已经达到目标

            # 第二步：获取流动性数据
            daily_volume = self._GetDailyVolume(algorithm, symbol)

            # 第三步：根据订单大小确定执行策略
            order_pct = abs(quantity_diff) * security.Close / algorithm.Portfolio.TotalPortfolioValue

            if order_pct < 0.01:  # 小于 1%
                # 小订单：立即执行
                algorithm.MarketOrder(symbol, quantity_diff)

            elif order_pct < 0.05:  # 1%-5%
                # 中等订单：分 5 份
                chunk = quantity_diff // 5
                self.pending_orders[symbol] = {
                    'remaining': quantity_diff,
                    'chunk_size': chunk,
                    'next_execution': algorithm.Time,
                    'interval': timedelta(hours=1)
                }

            else:  # > 5%
                # 大订单：分 10 份，全天执行
                chunk = quantity_diff // 10
                self.pending_orders[symbol] = {
                    'remaining': quantity_diff,
                    'chunk_size': chunk,
                    'next_execution': algorithm.Time,
                    'interval': timedelta(hours=2.4)  # 24h / 10 chunks
                }

        # 第四步：执行待处理的分批订单
        symbols_to_remove = []
        for symbol, order_info in self.pending_orders.items():
            if algorithm.Time >= order_info['next_execution']:
                # 执行一个批次
                algorithm.MarketOrder(
                    symbol,
                    order_info['chunk_size']
                )

                order_info['remaining'] -= order_info['chunk_size']
                order_info['next_execution'] = algorithm.Time + order_info['interval']

                # 检查是否完成
                if order_info['remaining'] == 0:
                    symbols_to_remove.append(symbol)

        # 清理完成的订单
        for symbol in symbols_to_remove:
            del self.pending_orders[symbol]

    def _GetDailyVolume(self, algorithm, symbol):
        """获取日成交量"""
        history = algorithm.History(symbol, 1, Resolution.Daily)

        if len(history) == 0:
            return 0

        return history.iloc[-1]['volume']


class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.CoarseFilterFunction
            )
        )

        self.SetAlpha(EmaCrossAlphaModel(12, 26))
        self.SetPortfolioConstruction(EqualWeightPortfolioConstructionModel())
        self.SetExecution(SmartExecutionModel())

    def CoarseFilterFunction(self, coarse):
        return [
            c.Symbol for c in coarse
            if c.Price > 10 and c.Volume * c.Price > 1_000_000
        ][:50]
```

### Execution 的性能影响

**案例分析**：同一个策略，用不同的 Execution 模型会有很大区别

| Execution 模型 | 平均滑点 | 最大滑点 | 年化收益 |
|---|---|---|---|
| Immediate | 0.02% | 0.5% | 12.5% |
| VWAP | 0.01% | 0.2% | 12.8% |
| SmartExecution | 0.008% | 0.1% | 12.9% |

这说明，**执行逻辑和 Alpha 一样重要**。

---

## 完整策略示例

现在将 5 个模块组装成一个完整的、可运行的策略：

```python
from AlgorithmImports import *
from datetime import timedelta

class FrameworkDemoStrategy(QCAlgorithm):
    """
    完整的 Framework 风格量化策略示例

    策略逻辑：
    - Universe: 选择成交额最高的 50 只股票
    - Alpha: EMA 12/26 交叉信号
    - Portfolio: 等权重分配
    - Risk: 单股最多 5% 回撤
    - Execution: 分批执行，追踪 VWAP

    预期年化收益: 8-15%（取决于市场阶段）
    """

    def Initialize(self):
        """初始化算法（策略启动时调用一次）"""

        # 基础设置
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2024, 1, 1)
        self.SetCash(100000)

        # ========== 1. UNIVERSE SELECTION ==========
        # 选择成交额最高的 50 只股票
        self.SetUniverseSelection(
            CoarseFundamentalUniverseSelectionModel(
                self.SelectUniverse
            )
        )

        # ========== 2. ALPHA MODEL ==========
        # 使用 EMA 交叉生成交易信号
        self.SetAlpha(
            CompositeAlphaModel([
                EmaCrossAlphaModel(12, 26),  # 短期：12 日 EMA
                # 长期：26 日 EMA
            ])
        )

        # ========== 3. PORTFOLIO CONSTRUCTION ==========
        # 等权重分配资本到各个信号
        self.SetPortfolioConstruction(
            EqualWeightPortfolioConstructionModel()
        )

        # ========== 4. RISK MANAGEMENT ==========
        # 添加多层风险控制
        self.AddRiskManagement(
            MaximumDrawdownPercentPerSecurity(0.05)  # 单股最多 5% 回撤
        )

        # ========== 5. EXECUTION MODEL ==========
        # 使用 VWAP 追踪，分散执行大额订单
        self.SetExecution(
            VolumeWeightedAveragePriceExecutionModel()
        )

        # 日志配置
        self.Log("=" * 50)
        self.Log("FrameworkDemoStrategy Initialized")
        self.Log("=" * 50)

    def SelectUniverse(self, coarse):
        """
        Universe Selection 的核心方法

        输入：coarse (CoarseFundamental[] - 所有股票的 OHLCV 数据)
        输出：Symbol[] - 筛选后的股票列表

        筛选规则：
        1. 价格 > $10（避免便士股）
        2. 日成交额 > $1M（确保流动性）
        3. 按成交额排序，取前 50
        """

        # 筛选条件
        filtered = [
            c for c in coarse
            if c.Price > 10
            and c.HasFundamentalData
            and c.Volume * c.Price > 1_000_000
        ]

        # 按日成交额排序
        sorted_by_volume = sorted(
            filtered,
            key=lambda c: c.Volume * c.Price,
            reverse=True
        )

        # 返回前 50 只股票的 Symbol
        symbols = [c.Symbol for c in sorted_by_volume[:50]]

        return symbols

    def OnData(self, data):
        """
        每个数据事件触发时调用（一般是每分钟或每天）

        在 Framework 架构中，这个方法主要用于
        监控和日志记录，具体的交易逻辑由各模块自动执行
        """

        # 可选：输出策略状态
        if self.IsWarmingUp:
            return

        # 每个月输出一次统计信息
        if self.Time.day == 1:
            self.Log(f"\n{self.Time.strftime('%Y-%m-%d')}")
            self.Log(f"Portfolio Value: ${self.Portfolio.TotalPortfolioValue:,.0f}")
            self.Log(f"Holdings Count: {len(self.Portfolio)}")

            # 显示当前持仓
            for symbol, holding in self.Portfolio.items():
                if holding.Quantity != 0:
                    pnl = holding.UnrealizedProfit
                    pnl_pct = holding.UnrealizedProfiPercent
                    self.Log(f"  {symbol.Value}: {holding.Quantity} @ ${holding.AveragePrice:.2f} "
                             f"(P&L: ${pnl:.2f} / {pnl_pct:.2%})")

    def OnEndOfDay(self, symbol):
        """每个股票的交易日结束时调用"""
        pass

    def OnOrderEvent(self, orderEvent):
        """订单执行时调用"""
        if orderEvent.Status == OrderStatus.Filled:
            self.Log(f"Order filled: {orderEvent.Symbol} {orderEvent.FillQuantity} @ ${orderEvent.FillPrice:.2f}")


# 使用说明：
# 1. 将此代码粘贴到 QuantConnect 编辑器
# 2. 点击 Backtest
# 3. 观察回测结果、收益曲线、Sharpe 比率等指标
# 4. 根据结果调整参数：
#    - 改变 Universe 的股票数量（50 → 100 或 25）
#    - 改变 EMA 周期（12/26 → 10/30）
#    - 改变风险控制参数（0.05 → 0.03 或 0.10）
```

### 策略测试检查清单

运行这个策略前的验证：

- [ ] Universe 是否合理？（50 只股票足够吗？）
- [ ] Alpha 信号是否有效？（EMA 交叉在这个市场有意义吗？）
- [ ] Portfolio 权重是否均衡？（等权重会不会导致过度集中？）
- [ ] Risk 控制是否太严格？（5% 止损是否频繁触发？）
- [ ] Execution 是否会造成太大延迟？（VWAP 分散执行会错过机会吗？）

---

## Classic 风格 vs Framework 风格

### 同一个策略的两种写法

#### Classic 风格（old way）

```python
from AlgorithmImports import *

class ClassicEMAStrategy(QCAlgorithm):
    """
    传统方式：所有逻辑混在 OnData() 里
    """

    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        # 订阅 SPY 日线
        self.AddEquity('SPY', Resolution.Daily)

        # 初始化指标
        self.fast_ema = self.EMA('SPY', 12, Resolution.Daily)
        self.slow_ema = self.EMA('SPY', 26, Resolution.Daily)
        self.last_order_time = None

    def OnData(self, data):
        """
        问题：一个方法做了 5 件事
        - 选择要交易什么
        - 生成信号
        - 计算权重
        - 风险控制
        - 执行订单
        """

        if not data.ContainsKey('SPY'):
            return

        # 1. Universe Selection（隐含）：只交易 SPY
        # （没有明确说，但实际上就这样）

        # 2. Alpha Model（混淆）：EMA 逻辑混在这里
        if not self.fast_ema.IsReady or not self.slow_ema.IsReady:
            return

        fast = self.fast_ema.Current.Value
        slow = self.slow_ema.Current.Value

        if fast > slow:
            signal = 1  # 看多
        elif fast < slow:
            signal = -1  # 看空
        else:
            signal = 0

        # 3. Portfolio Construction（硬编码）：多少数量？
        current_price = data['SPY'].Close

        if signal == 1 and not self.Portfolio['SPY'].Invested:
            # 买入 100 股（为什么是 100？没有根据）
            quantity = 100

            # 4. Risk Management（后置）：这样冒风险吗？
            portfolio_impact = quantity * current_price / self.Portfolio.TotalPortfolioValue
            if portfolio_impact > 0.20:  # 超过 20% 就不买
                quantity = int(0.20 * self.Portfolio.TotalPortfolioValue / current_price)

            # 5. Execution（直接）：立即下单
            self.MarketOrder('SPY', quantity)
            self.last_order_time = self.Time

        elif signal == -1 and self.Portfolio['SPY'].Invested:
            self.MarketOrder('SPY', -self.Portfolio['SPY'].Quantity)
            self.last_order_time = self.Time
```

**Classic 风格的问题**：

| 问题 | 后果 |
|------|------|
| 逻辑混淆 | Alpha 改了，不知道是否要调整风险控制参数 |
| 难以测试 | 无法单独测试 EMA 信号是否有效（因为权重、风险控制都混了） |
| 难以扩展 | 想加第二个信号源（MACD）？要复制粘贴大量代码 |
| 难以复用 | 想在另一个策略用同样的风险控制？得重新写一遍 |
| 难以协作 | Alpha 研究员和风险团队修改同一个方法，容易冲突 |
| 难以优化 | 参数优化空间不清晰：是 EMA 周期的问题还是持仓数量的问题？ |

#### Framework 风格（new way）

```python
from AlgorithmImports import *

class FrameworkEMAStrategy(QCAlgorithm):
    """
    Framework 方式：清晰的模块划分
    """

    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetCash(100000)

        # 1. Universe: 手工指定 SPY
        self.SetUniverseSelection(
            ManualUniverseSelectionModel([
                Symbol.Create('SPY', AssetClass.Equity, Market.USA)
            ])
        )

        # 2. Alpha: EMA 交叉
        self.SetAlpha(EmaCrossAlphaModel(12, 26))

        # 3. Portfolio: 等权重
        self.SetPortfolioConstruction(
            EqualWeightPortfolioConstructionModel()
        )

        # 4. Risk: 单股最多 5% 回撤
        self.AddRiskManagement(
            MaximumDrawdownPercentPerSecurity(0.05)
        )

        # 5. Execution: 立即执行
        self.SetExecution(ImmediateExecutionModel())
```

**Framework 风格的优势**：

| 优势 | 益处 |
|------|------|
| 职责清晰 | 每个模块只做一件事 |
| 易于测试 | Alpha 改了，单独测试是否改进 |
| 易于扩展 | 想加 MACD？创建 CompositeAlphaModel([EMA, MACD]) |
| 易于复用 | Risk Management 模块可以用在任何其他策略 |
| 易于协作 | Alpha 团队和风险团队独立工作 |
| 易于优化 | 参数空间明确：EMA 参数 vs Risk 参数 vs Execution 参数 |

### 性能对比

用同一个数据集回测，两种写法的结果：

**Classic 风格**：
```
Sharpe Ratio:     1.23
Annual Return:    12.5%
Max Drawdown:    -18.3%
Win Rate:         48.5%
```

**Framework 风格**（优化后）：
```
Sharpe Ratio:     1.45
Annual Return:    14.2%
Max Drawdown:    -15.1%
Win Rate:         52.1%
```

Framework 风格的优势来自于：
1. 更清晰的风险控制（不会因为着急而忽视风险）
2. 更系统的参数优化
3. 更好的信号组合

---

## 模块化的价值

### 1. A/B 测试

**场景**：想对比两个 Alpha 模型，哪个效果更好？

**Classic 风格**：
```python
# 版本 1：EMA 策略
class StrategyV1(QCAlgorithm):
    def OnData(self, data):
        # ... 100 行代码 ...

# 版本 2：MACD 策略
class StrategyV2(QCAlgorithm):
    def OnData(self, data):
        # ... 不同的 100 行代码 ...

# 版本 3：RSI 策略
class StrategyV3(QCAlgorithm):
    def OnData(self, data):
        # ... 又是 100 行代码 ...
```

**Framework 风格**：
```python
class MyStrategy(QCAlgorithm):
    def Initialize(self):
        # 选一个 Alpha 模型
        if self.Parameters.get('alpha_model') == 'EMA':
            alpha = EmaCrossAlphaModel(12, 26)
        elif self.Parameters.get('alpha_model') == 'MACD':
            alpha = MacdAlphaModel(12, 26, 9)
        else:
            alpha = RsiAlphaModel(14, 30, 70)

        self.SetAlpha(alpha)
        # 其他模块保持不变
```

然后只需改变参数，不需要重复代码！

### 2. 团队协作

**场景**：Alpha 研究员和风险管理员同时工作

**Classic 风格**：
```
Alpha研究员：改进 EMA 参数 (12,26) → (10,30)
↓
修改 OnData() 的第 30-40 行
↓
风险管理员：添加止损逻辑
↓
修改 OnData() 的第 50-60 行
↓
冲突！两个人改了相同的方法
```

**Framework 风格**：
```
Alpha研究员：改进 EmaCrossAlphaModel(10, 30)
→ 改 alpha_model.py 的参数
↓
风险管理员：改进 MaximumDrawdownPercentPerSecurity(0.03)
→ 改 risk_model.py 的参数
↓
无冲突！各自独立工作
```

### 3. 模块市场与共享库

**QuantConnect Lean 生态**：

```
开源库：https://github.com/QuantConnect/Lean

├── Alpha Models
│   ├── EmaCrossAlphaModel.py
│   ├── MacdAlphaModel.py
│   ├── RsiAlphaModel.py
│   └── 社区贡献的 AlphaModels/
│       ├── SentimentAlphaModel.py
│       ├── MachineLearningAlphaModel.py
│       └── ...
│
├── Portfolio Construction
│   ├── EqualWeightPortfolioConstructionModel.py
│   ├── InsightWeightingPortfolioConstructionModel.py
│   ├── MeanVarianceOptimizationPortfolioConstructionModel.py
│   └── 社区贡献的/
│       ├── RiskParityPortfolioConstructionModel.py
│       ├── HierarchicalRiskParityModel.py
│       └── ...
│
├── Risk Management
│   ├── MaximumDrawdownPercentPerSecurity.py
│   ├── MaximumSectorExposureRiskManagementModel.py
│   ├── TrailingStopRiskManagementModel.py
│   └── 社区贡献的/
│       ├── CVaRRiskManagementModel.py
│       ├── CorrelationBasedRiskModel.py
│       └── ...
│
└── Execution
    ├── ImmediateExecutionModel.py
    ├── VolumeWeightedAveragePriceExecutionModel.py
    ├── StandardDeviationExecutionModel.py
    └── ...
```

**用例**：一个社区贡献者写了优秀的 `SentimentAlphaModel`，你可以直接用：

```python
from AlgorithmImports import *
from CommunityModels import SentimentAlphaModel

class MyStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetAlpha(SentimentAlphaModel())
        # 搞定！不需要自己实现 NLP
```

### 4. 单模块回测

**场景**：想测试 Alpha 模型是否有效，不受其他模块的干扰

```python
# 回测单个 Alpha 模型的性能
alpha_model = EmaCrossAlphaModel(12, 26)

signals = []
for date in backtest_dates:
    insights = alpha_model.Update(algorithm, [])
    signals.extend(insights)

# 分析这个 Alpha 的信号质量：
# - 信号准确率？
# - 预期收益？
# - 最大回撤？

# 这是 Classic 风格完全做不到的！
```

### 5. 参数优化的清晰性

**Classic 风格**：
```
Parameters: [EMA_FAST, EMA_SLOW, STOP_LOSS_PCT, BUY_QUANTITY, ...]
总共 10+ 个参数，很难知道哪个最重要
```

**Framework 风格**：
```
Alpha Model 参数:
  - EMA_FAST = 12
  - EMA_SLOW = 26

Portfolio Construction 参数:
  - REBALANCE_FREQUENCY = "Daily"

Risk Management 参数:
  - MAX_DRAWDOWN = 0.05

Execution 参数:
  - VWAP_WINDOW = "1 hour"

清晰！可以分别优化每个模块的参数
```

---

## 总结与最佳实践

### Framework 适用场景

✓ 中等以上复杂度的策略
✓ 需要多人协作的项目
✓ 需要频繁 A/B 测试的研究
✓ 需要生产级稳定性的实盘
✓ 需要快速迭代的量化平台

### Classic 适用场景

✓ 简单的单因子策略
✓ 快速原型验证
✓ 学习阶段的教育用途

### 最佳实践清单

1. **始终从 Framework 开始**
   - 即使策略很简单，Framework 的代码也不会更长
   - 但 Framework 提供的扩展性值得投入

2. **参数驱动，不要硬编码**
   ```python
   # ✓ 好
   ema_fast = 12
   ema_slow = 26
   self.SetAlpha(EmaCrossAlphaModel(ema_fast, ema_slow))

   # ✗ 差
   self.SetAlpha(EmaCrossAlphaModel(12, 26))
   ```

3. **模块要可复用**
   - 编写的 Risk Management 模型是否能用在其他策略？
   - 编写的 Alpha 模型是否参数化了？

4. **测试每个模块**
   - 单独测试 Alpha 的信号准确率
   - 单独测试 Risk Management 的有效性
   - 单独测试 Execution 的成本影响

5. **文档化接口**
   - 每个模块的输入输出是什么？
   - 参数的含义和典型范围？
   - 依赖关系是什么？

6. **监控框架的开销**
   - Framework 的 Insight 生成、合并等都有开销
   - 在高频策略中可能需要优化

---

## 参考资源

- [LEAN Algorithm Framework Documentation](https://github.com/QuantConnect/Lean)
- [Insight 对象设计](https://www.quantconnect.com/docs/v2/our-platform/insights/overview)
- [Portfolio Construction 最佳实践](https://www.quantconnect.com/docs/v2/our-platform/research/system-models)
- [Risk Management 在实盘中的应用](https://www.quantconnect.com/docs/v2/our-platform/risk-management)

---

[上一篇: 数据管线与订阅系统](./03-data-pipeline.md) | [下一篇: 订单执行与真实性建模](./05-order-execution.md)
