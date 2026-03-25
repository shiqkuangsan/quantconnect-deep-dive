# 二开入门：自定义指标

## 目录

1. [什么是自定义指标](#什么是自定义指标)
2. [LEAN 指标系统架构快览](#lean-指标系统架构快览)
3. [Lab 1: 自定义 VWAP 指标](#lab-1-自定义-vwap-指标)
4. [Lab 2: 自定义动量偏离指标](#lab-2-自定义动量偏离指标)
5. [Lab 3: 组合指标](#lab-3-组合指标)
6. [测试自定义指标](#测试自定义指标)
7. [打包与复用](#打包与复用)
8. [常见问题](#常见问题)

---

## 什么是自定义指标

QuantConnect 内置了数百个技术指标——简单移动平均（SMA）、相对强弱指数（RSI）、MACD、布林带等。这些指标覆盖了大多数常见的技术分析需求。

但现实中，许多量化团队都有 **专有的、针对特定市场或资产的信号计算方法**。这可能是：

- 改进的移动平均线变体（例如，自适应权重，基于波动率调整）
- 融合多个市场数据的复合指标（例如，结合价格、成交量、基本面）
- 专有的机器学习预处理步骤
- 基于内部研究的独创算法

**自定义指标** 就是用来实现这些想法的工具。它允许你：

1. 继承 LEAN 的指标基类
2. 定义自己的计算逻辑
3. 像使用内置指标一样无缝集成到策略中
4. 在团队内复用和迭代

这是 **最简单的二次开发形式**，不涉及修改 LEAN 核心，只需在现有框架内扩展功能。

---

## LEAN 指标系统架构快览

### 指标的核心概念

在 LEAN 中，每个指标都遵循一个统一的生命周期：

```
数据流入 → Update() 方法 → 内部计算 → 存储当前值
                    ↓
              IsReady 状态检查
                    ↓
              策略获取指标值
```

### 关键类和属性

| 概念 | 说明 |
|------|------|
| **IndicatorBase<T>** | 所有指标的抽象基类 |
| **Update(data)** | 每当新数据到达时调用，执行计算逻辑 |
| **IsReady** | 布尔值，表示指标是否已计算出有效值 |
| **WarmUpPeriod** | 指标需要多少条数据才能 IsReady=True |
| **Current** | 最新计算结果（IndicatorDataPoint 对象） |
| **Values** | 历史值的时间序列 |

### LEAN 如何将指标融入数据管线

```
TimeSlice（包含所有当前 bar 数据）
    ↓
Strategy.OnData() 被调用
    ↓
任何已订阅的指标都会接收 Update()
    ↓
指标内部状态更新
    ↓
策略代码可以通过 indicator.Current.Value 访问结果
```

关键是：**数据管线自动推送数据给指标，你只需实现 Update() 逻辑**。

更多细节见：[架构参考：数据管线](../quantconnect-deep-dive/03-data-pipeline.md)

---

## Lab 1: 自定义 VWAP 指标

### 为什么选 VWAP？

**VWAP（Volume-Weighted Average Price，成交量加权平均价格）** 是机构投资者的标准工具。它回答：

> 在今天的交易中，平均每股支付了多少钱？

与简单的收盘价不同，VWAP 考虑了成交量，给大成交量的价格更高的权重。

公式：

```
VWAP = Σ(Price × Volume) / Σ(Volume)
```

**重要**：VWAP 在每个交易日重置。今天的 VWAP 与昨天的完全独立。

QuantConnect 确实有一个内置的 VWAP 指标，但这里我们 **从零开始实现**，以理解自定义指标的结构。

### 实现步骤

#### 步骤 1：定义指标类

```python
# 文件: custom_indicators.py

from AlgorithmImports import *

class CustomVWAP(PythonIndicator):
    """
    自定义 VWAP 指标
    
    在每个交易日，累积价格*成交量，除以总成交量。
    在新的交易日开始时重置。
    """
    
    def __init__(self, name="CustomVWAP"):
        # 初始化基类
        super().__init__()
        
        # 指标名称
        self.Name = name
        
        # 累积的价格*成交量
        self.cumulative_pv = 0.0
        
        # 累积的成交量
        self.cumulative_volume = 0.0
        
        # 跟踪当前是哪一天（用于检测日期变化）
        self.last_date = None
        
        # 指标在接收第一条数据后就可以使用
        # （在实际使用中，可能需要 warm-up）
        self.WarmUpPeriod = 1
```

#### 步骤 2：实现 Update() 方法

```python
    def Update(self, input):
        """
        Update() 是核心计算逻辑。
        
        参数:
            input: TradeBar 对象，包含 Open, High, Low, Close, Volume, Time
        
        流程:
            1. 检测是否是新的一天（重置累积值）
            2. 累积今天的 PV 和 Volume
            3. 计算 VWAP
            4. 返回 True 表示成功更新
        """
        
        # 如果日期改变，说明是新的交易日，重置累积值
        if self.last_date is None or input.Time.date() != self.last_date:
            self.cumulative_pv = 0.0
            self.cumulative_volume = 0.0
            self.last_date = input.Time.date()
        
        # 累积当前 bar 的数据
        # 使用 (High + Low + Close) / 3 作为成交价（也可用 Close）
        typical_price = (input.High + input.Low + input.Close) / 3.0
        self.cumulative_pv += typical_price * input.Volume
        self.cumulative_volume += input.Volume
        
        # 计算 VWAP
        if self.cumulative_volume > 0:
            vwap_value = self.cumulative_pv / self.cumulative_volume
        else:
            vwap_value = input.Close
        
        # 将结果存储到 self.Value（自动更新 Current）
        self.Value = vwap_value
        
        # 返回 True 表示这次更新成功
        return True
```

#### 步骤 3：在策略中使用

```python
class VWAPStrategyExample(QCAlgorithm):
    """
    使用自定义 VWAP 指标的简单策略。
    
    交易信号：
    - 当价格在 VWAP 上方时，市场看涨
    - 当价格在 VWAP 下方时，市场看跌
    """
    
    def Initialize(self):
        # 设置回测环境
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 添加股票
        self.symbol = self.AddEquity("AAPL", Resolution.Daily).Symbol
        
        # 创建并订阅自定义 VWAP 指标
        # 通过继承 PythonIndicator，LEAN 会自动调用 Update()
        self.vwap = CustomVWAP("AAPL_VWAP")
        self.RegisterIndicator(self.symbol, self.vwap, Resolution.Daily)
    
    def OnData(self, data):
        """
        每个 bar 调用一次。
        VWAP 已经在 data 到达前自动更新。
        """
        
        # 检查是否有数据和指标是否就绪
        if not self.vwap.IsReady or self.symbol not in data:
            return
        
        price = data[self.symbol].Close
        vwap = self.vwap.Current.Value
        
        # 交易逻辑
        if not self.Portfolio[self.symbol].Invested:
            # 当价格突破 VWAP 时买入
            if price > vwap * 1.001:  # 略高于 VWAP 确认
                self.SetHoldings(self.symbol, 0.95)
                self.Debug(f"BUY at {price:.2f}, VWAP={vwap:.2f}")
        else:
            # 当价格跌破 VWAP 时卖出
            if price < vwap * 0.999:
                self.Liquidate(self.symbol)
                self.Debug(f"SELL at {price:.2f}, VWAP={vwap:.2f}")
```

### 测试这个指标

在 QuantConnect 的 Backtest 标签页：

1. 将上面的 `CustomVWAP` 类粘贴到你的项目代码中
2. 使用 `VWAPStrategyExample` 策略运行回测
3. 观察日志输出和性能指标

**预期结果**：使用 VWAP 作为支撑/阻力，你应该看到一个合理的入场/出场逻辑。

---

## Lab 2: 自定义动量偏离指标

### 概念

**动量偏离指标（Momentum Deviation Index）** 是我们创造的一个复合指标。它：

1. 计算每日收益率（momentum）
2. 计算最近 N 天收益率的 z-score（有多少个标准差偏离均值）
3. 输出这个 z-score

**交易信号**：

- `z-score < -2`：动量异常低（可能反弹）→ **买入信号**
- `z-score > 2`：动量异常高（可能回调）→ **卖出信号**

这个指标对识别 **过度延伸** 的市场很有用。

### 完整实现

```python
import numpy as np
from collections import deque
from AlgorithmImports import *

class MomentumDeviationIndex(PythonIndicator):
    """
    动量偏离指标
    
    计算最近 N 天收益率的 z-score。
    当 z-score 异常高或低时，表示动量异常。
    """
    
    def __init__(self, lookback_period=20, name="MDI"):
        """
        参数:
            lookback_period: 用于计算 z-score 的历史窗口（天数）
            name: 指标名称
        """
        super().__init__()
        
        self.Name = name
        self.lookback_period = lookback_period
        
        # 用于存储最近 N 天的收益率
        # deque 自动维护固定大小的滑动窗口
        self.returns_window = deque(maxlen=lookback_period)
        
        # 前一天的收盘价（用于计算今天的收益率）
        self.last_close = None
        
        # 为了计算 z-score，需要至少 lookback_period 个数据点
        self.WarmUpPeriod = lookback_period
    
    def Update(self, input):
        """
        每个 bar 更新一次。
        
        流程:
            1. 计算今天的收益率
            2. 添加到窗口
            3. 如果窗口充满，计算 z-score
            4. 返回结果
        """
        
        # 初始化：第一次没有前一天价格
        if self.last_close is None:
            self.last_close = input.Close
            self.Value = 0.0
            return False
        
        # 计算日收益率 (log return)
        daily_return = np.log(input.Close / self.last_close)
        self.returns_window.append(daily_return)
        self.last_close = input.Close
        
        # 如果窗口还没填满，不计算 z-score
        if len(self.returns_window) < self.lookback_period:
            self.Value = 0.0
            return False
        
        # 窗口已满，计算 z-score
        mean_return = np.mean(list(self.returns_window))
        std_return = np.std(list(self.returns_window))
        
        # 避免除以 0
        if std_return < 1e-10:
            z_score = 0.0
        else:
            # z-score = (当前值 - 均值) / 标准差
            current_return = self.returns_window[-1]
            z_score = (current_return - mean_return) / std_return
        
        self.Value = z_score
        return True
```

### 在策略中使用

```python
class MomentumDeviationStrategy(QCAlgorithm):
    """
    基于动量偏离的交易策略。
    
    逻辑：
    - MDI < -2: 动量被过度压制，可能反弹 → BUY
    - MDI > 2: 动量过度膨胀，可能下跌 → SELL
    """
    
    def Initialize(self):
        self.SetStartDate(2022, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        self.symbol = self.AddEquity("SPY", Resolution.Daily).Symbol
        
        # 创建 MDI，lookback_period=20 天
        self.mdi = MomentumDeviationIndex(lookback_period=20)
        self.RegisterIndicator(self.symbol, self.mdi, Resolution.Daily)
    
    def OnData(self, data):
        if not self.mdi.IsReady or self.symbol not in data:
            return
        
        mdi_value = self.mdi.Current.Value
        price = data[self.symbol].Close
        
        # 计数器，避免频繁交易
        if not hasattr(self, 'last_trade_date'):
            self.last_trade_date = None
        
        # 最少间隔 3 天再交易
        if self.last_trade_date and (data.Time.date() - self.last_trade_date).days < 3:
            return
        
        # 买入信号：MDI 跌破 -2
        if not self.Portfolio[self.symbol].Invested:
            if mdi_value < -2:
                self.SetHoldings(self.symbol, 0.9)
                self.Debug(f"BUY signal: MDI={mdi_value:.2f} at price {price:.2f}")
                self.last_trade_date = data.Time.date()
        
        # 卖出信号：MDI 超过 2，或者回到接近 0（动量正常化）
        else:
            if mdi_value > 2 or mdi_value > -0.5:
                self.Liquidate(self.symbol)
                self.Debug(f"SELL signal: MDI={mdi_value:.2f} at price {price:.2f}")
                self.last_trade_date = data.Time.date()
```

### 动量偏离指标的优势和局限

**优势**：
- 捕捉市场过度反应
- 基于统计方法，相对客观
- 可以与其他指标组合

**局限**：
- 历史数据的正态分布假设可能不总是成立（特别是在极端市场中）
- 需要足够的历史数据（lookback_period）才能工作
- 单独使用容易产生虚假信号

---

## Lab 3: 组合指标

### 概念

有时，最强大的信号来自于 **多个指标的综合**。

在这个 Lab 中，我们创建一个 **TrendStrength 指标**，它：

1. 计算 EMA 的斜率（趋势强度）
2. 结合 RSI（是否超买/超卖）
3. 结合成交量趋势（成交量是否确认趋势）
4. 输出一个 -100 到 +100 的综合分数

**分数含义**：
- **+100**：强烈买入信号（上升趋势，RSI 不超买，成交量确认）
- **-100**：强烈卖出信号
- **0**：中性

### 完整实现

```python
from AlgorithmImports import *

class TrendStrength(PythonIndicator):
    """
    组合指标：趋势强度
    
    综合考虑：
    1. EMA 斜率（价格趋势）
    2. RSI（超买/超卖）
    3. 成交量趋势
    
    输出一个 -100 到 +100 的分数。
    """
    
    def __init__(self, ema_period=20, rsi_period=14, volume_period=20, name="TrendStrength"):
        super().__init__()
        
        self.Name = name
        self.ema_period = ema_period
        self.rsi_period = rsi_period
        self.volume_period = volume_period
        
        # 内部指标：EMA（用于计算斜率）
        self.ema = ExponentialMovingAverage(ema_period)
        
        # 内部指标：RSI
        self.rsi = RelativeStrengthIndex(rsi_period)
        
        # 用于计算成交量移动平均
        self.volume_ema = ExponentialMovingAverage(volume_period)
        
        # 存储历史 EMA 值以计算斜率
        self.ema_history = []
        
        # 指标需要足够的 warm-up 时间
        self.WarmUpPeriod = max(ema_period, rsi_period, volume_period) + 5
    
    def Update(self, input):
        """
        综合计算趋势强度分数。
        """
        
        # 更新内部指标
        self.ema.Update(IndicatorDataPoint(input.Time, input.Close))
        self.rsi.Update(IndicatorDataPoint(input.Time, input.Close))
        self.volume_ema.Update(IndicatorDataPoint(input.Time, input.Volume))
        
        # 如果任何指标还未就绪，返回
        if not self.ema.IsReady or not self.rsi.IsReady or not self.volume_ema.IsReady:
            return False
        
        # ========== 组件 1: EMA 斜率 ==========
        current_ema = self.ema.Current.Value
        self.ema_history.append(current_ema)
        
        # 只在有足够历史时计算斜率
        if len(self.ema_history) >= 5:
            # 最近 5 个 EMA 值的斜率
            recent_emas = self.ema_history[-5:]
            ema_slope = (recent_emas[-1] - recent_emas[0]) / recent_emas[0]
            
            # 限制 ema_slope 在 -1 到 1 之间，然后缩放到 -100 到 100
            ema_slope_clamped = max(-0.05, min(0.05, ema_slope))
            ema_component = (ema_slope_clamped / 0.05) * 100  # 范围：-100 到 100
        else:
            ema_component = 0
        
        # ========== 组件 2: RSI ==========
        rsi_value = self.rsi.Current.Value
        
        # RSI 的解释：
        # - RSI > 70：超买，看跌
        # - RSI < 30：超卖，看涨
        # - RSI = 50：中性
        
        # 将 RSI (0-100) 转换为 -100 到 100 的信号
        rsi_component = 2 * (rsi_value - 50)  # 范围：-100 到 100
        
        # ========== 组件 3: 成交量趋势 ==========
        volume_ema_value = self.volume_ema.Current.Value
        current_volume = input.Volume
        
        if volume_ema_value > 0:
            # 当前成交量 vs 平均
            volume_ratio = current_volume / volume_ema_value
            
            # 如果成交量大于平均，看涨信号；如果小于平均，看跌
            # 限制在 0.5 到 1.5 范围内
            volume_ratio_clamped = max(0.7, min(1.3, volume_ratio))
            volume_component = (volume_ratio_clamped - 1) / 0.3 * 100  # 缩放到 -100 到 100
        else:
            volume_component = 0
        
        # ========== 综合计算 ==========
        # 加权平均三个组件
        # EMA 斜率：权重 50%（最重要的趋势指标）
        # RSI：权重 30%（确认极端情况）
        # 成交量：权重 20%（确认）
        
        trend_score = (
            ema_component * 0.5 +
            rsi_component * 0.3 +
            volume_component * 0.2
        )
        
        # 限制在 -100 到 100
        trend_score = max(-100, min(100, trend_score))
        
        self.Value = trend_score
        return True
```

### 在策略中使用

```python
class CompositeIndicatorStrategy(QCAlgorithm):
    """
    使用 TrendStrength 组合指标的策略。
    """
    
    def Initialize(self):
        self.SetStartDate(2022, 6, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        self.symbol = self.AddEquity("QQQ", Resolution.Daily).Symbol
        
        # 创建组合指标
        self.trend_strength = TrendStrength(
            ema_period=20,
            rsi_period=14,
            volume_period=20
        )
        self.RegisterIndicator(self.symbol, self.trend_strength, Resolution.Daily)
        
        # 持仓跟踪
        self.position_size = 0
    
    def OnData(self, data):
        if not self.trend_strength.IsReady or self.symbol not in data:
            return
        
        score = self.trend_strength.Current.Value
        price = data[self.symbol].Close
        
        # 交易逻辑：基于强度分数
        if not self.Portfolio[self.symbol].Invested:
            # 强烈买入信号
            if score > 50:
                self.SetHoldings(self.symbol, 0.9)
                self.Debug(f"STRONG BUY: score={score:.1f}, price={price:.2f}")
            # 弱买入信号
            elif score > 20:
                self.SetHoldings(self.symbol, 0.5)
                self.Debug(f"WEAK BUY: score={score:.1f}, price={price:.2f}")
        
        else:  # 已持仓
            # 强烈卖出信号
            if score < -50:
                self.Liquidate(self.symbol)
                self.Debug(f"STRONG SELL: score={score:.1f}, price={price:.2f}")
            # 弱卖出信号
            elif score < -20:
                # 部分减仓
                self.SetHoldings(self.symbol, 0.3)
                self.Debug(f"WEAK SELL: score={score:.1f}, price={price:.2f}")
```

### 组合指标的优势

- **降低虚假信号**：多个指标的共识比单个指标更可靠
- **适应多种市场条件**：同时考虑趋势、动量和成交量
- **灵活调整权重**：可以根据回测结果调整各组件的权重

---

## 测试自定义指标

### 单元测试方法

创建一个简单的测试脚本来验证你的指标：

```python
# 文件: test_indicators.py

from AlgorithmImports import *
import unittest

class TestCustomVWAP(unittest.TestCase):
    """
    测试 CustomVWAP 指标的单元测试。
    """
    
    def setUp(self):
        """每个测试前初始化"""
        self.vwap = CustomVWAP()
    
    def test_initialization(self):
        """测试初始化状态"""
        self.assertEqual(self.vwap.cumulative_volume, 0.0)
        self.assertEqual(self.vwap.cumulative_pv, 0.0)
    
    def test_single_bar(self):
        """测试单个 bar 的 VWAP 计算"""
        # 创建虚拟 TradeBar
        bar = TradeBar()
        bar.Open = 100
        bar.High = 105
        bar.Low = 99
        bar.Close = 103
        bar.Volume = 1000
        bar.Time = datetime(2023, 1, 1)
        
        self.vwap.Update(bar)
        
        # 单个 bar 的 VWAP 应该等于该 bar 的典型价格
        expected = (105 + 99 + 103) / 3.0
        self.assertAlmostEqual(self.vwap.Value, expected, places=2)
    
    def test_accumulation(self):
        """测试多个 bar 的累积"""
        bars = [
            TradeBar(Time=datetime(2023, 1, 1, 9, 30), Close=100, High=101, Low=99, Volume=1000),
            TradeBar(Time=datetime(2023, 1, 1, 10, 0), Close=102, High=103, Low=101, Volume=1500),
        ]
        
        for bar in bars:
            self.vwap.Update(bar)
        
        # 验证累积量
        self.assertEqual(self.vwap.cumulative_volume, 2500)
    
    def test_daily_reset(self):
        """测试日期变化时的重置"""
        bar1 = TradeBar(Time=datetime(2023, 1, 1), Close=100, High=101, Low=99, Volume=1000)
        bar2 = TradeBar(Time=datetime(2023, 1, 2), Close=105, High=106, Low=104, Volume=1000)
        
        self.vwap.Update(bar1)
        vol_after_first = self.vwap.cumulative_volume
        
        self.vwap.Update(bar2)
        # 第二天应该重置，所以成交量只有 1000
        self.assertEqual(self.vwap.cumulative_volume, 1000)

if __name__ == '__main__':
    unittest.main()
```

### 使用 Research 笔记本验证

在 QuantConnect 的 Research 笔记本中：

```python
# 1. 导入你的自定义指标
from custom_indicators import CustomVWAP, MomentumDeviationIndex

# 2. 获取历史数据
history = qb.History(["SPY"], 50, Resolution.Daily)

# 3. 创建指标并逐条输入数据
vwap = CustomVWAP()
for idx, row in history.iterrows():
    bar = TradeBar()
    bar.Time = idx[1]
    bar.Open = row['open']
    bar.High = row['high']
    bar.Low = row['low']
    bar.Close = row['close']
    bar.Volume = row['volume']
    vwap.Update(bar)

# 4. 提取值并绘图
import matplotlib.pyplot as plt

prices = history['close'].values
vwaps = [vwap.Value]  # 你可能需要重新运行并收集所有值

plt.figure(figsize=(12, 6))
plt.plot(prices, label='Close Price', linewidth=2)
plt.plot(vwaps, label='VWAP', linewidth=2)
plt.legend()
plt.show()
```

### 需要测试的边界情况

| 场景 | 测试内容 |
|------|---------|
| **零成交量** | 当成交量为 0 时，指标应该使用默认值（通常是收盘价） |
| **数据缺失** | 如果某些日期没有数据，指标应该处理不连续的时间戳 |
| **极端价格** | 极高或极低的价格变动应该不会导致指标溢出 |
| **日期边界** | 跨越午夜、周末、节假日时，日期重置逻辑应该正确 |
| **第一个 bar** | 指标的第一条数据应该初始化正确 |

---

## 打包与复用

### 团队共享指标的最佳实践

#### 目录结构

```
quantconnect-project/
├── algorithms/
│   ├── strategy_1.py
│   └── strategy_2.py
├── indicators/
│   ├── __init__.py
│   ├── custom_vwap.py
│   ├── momentum_deviation.py
│   ├── trend_strength.py
│   └── README.md
└── research/
    └── validate_indicators.ipynb
```

#### indicators/__init__.py

```python
"""
自定义指标库

导出所有自定义指标，方便导入。
"""

from .custom_vwap import CustomVWAP
from .momentum_deviation import MomentumDeviationIndex
from .trend_strength import TrendStrength

__all__ = [
    'CustomVWAP',
    'MomentumDeviationIndex',
    'TrendStrength',
]
```

#### 在策略中导入

```python
# 在你的策略文件中
from indicators import CustomVWAP, MomentumDeviationIndex, TrendStrength

class MyStrategy(QCAlgorithm):
    def Initialize(self):
        # ... 设置代码 ...
        
        # 直接使用导入的指标
        self.vwap = CustomVWAP()
        self.mdi = MomentumDeviationIndex()
        self.trend = TrendStrength()
```

#### 指标文档模板（indicators/README.md）

```markdown
# 自定义指标文档

## CustomVWAP
- **说明**：成交量加权平均价格
- **参数**：无
- **WarmUpPeriod**：1
- **用途**：识别机构成本、支撑阻力

## MomentumDeviationIndex
- **说明**：动量的标准差评分
- **参数**：lookback_period (default: 20)
- **WarmUpPeriod**：20+
- **用途**：识别过度反应的市场

## TrendStrength
- **说明**：综合的趋势强度评分
- **参数**：ema_period, rsi_period, volume_period
- **输出范围**：-100 到 +100
- **用途**：多维度确认趋势
```

---

## 常见问题

### Q1: 自定义指标与内置指标有性能差异吗？

**A**: 通常没有显著差异。LEAN 已经为指标执行进行了优化。自定义指标使用相同的数据管线。但如果你在 Update() 中进行复杂的计算（例如，矩阵运算），可能会有性能影响。

**建议**：
- 避免在 Update() 中使用循环或列表推导
- 预先计算能预先计算的东西
- 使用 numpy/pandas 进行向量化操作

### Q2: 如何在自定义指标中使用其他内置指标？

**A**: 完全可以。如 Lab 3 的 TrendStrength 指标所示，你可以创建内部指标实例并在 Update() 中调用它们：

```python
class MyCompositeIndicator(PythonIndicator):
    def __init__(self):
        super().__init__()
        self.sma = SimpleMovingAverage(20)
        self.rsi = RelativeStrengthIndex(14)
    
    def Update(self, input):
        self.sma.Update(IndicatorDataPoint(input.Time, input.Close))
        self.rsi.Update(IndicatorDataPoint(input.Time, input.Close))
        # 然后使用 self.sma.Current.Value 等
```

### Q3: 如果我的指标需要多个数据源（例如，价格和基本面数据）怎么办？

**A**: 这通常需要在策略级别处理，而不是在指标内部。在策略的 OnData() 中接收两种数据，然后手动计算或传给指标。

```python
class MyStrategy(QCAlgorithm):
    def OnData(self, data):
        # 获取价格数据
        price = data[self.stock].Close
        
        # 获取基本面数据（如果已订阅）
        fundamental = data.get(self.fundamental_symbol)
        
        # 手动计算或传给指标
        combined_signal = calculate_combined(price, fundamental)
```

### Q4: 如何在实盘（Paper Trading）中使用自定义指标？

**A**: 完全相同的代码。LEAN 在 Paper Trading 和 Backtesting 中使用相同的算法框架。指标会按照同样的方式接收和处理数据。

### Q5: 自定义指标可以使用机器学习模型吗？

**A**: 可以，但需要小心。你可以在 Update() 中调用 ML 模型，但需要：

1. 在 Initialize() 中加载模型（避免每次 Update 都加载）
2. 确保模型推断速度足够快
3. 避免使用未来数据（数据泄露）

```python
class MLBasedIndicator(PythonIndicator):
    def Initialize(self):
        super().__init__()
        # 预加载模型
        import pickle
        with open('model.pkl', 'rb') as f:
            self.model = pickle.load(f)
    
    def Update(self, input):
        # 提取特征
        features = extract_features(input)
        
        # 推断
        prediction = self.model.predict([features])[0]
        self.Value = prediction
        return True
```

### Q6: 如何调试我的自定义指标？

**A**: 几种方法：

1. **使用 self.Debug()**：在指标内部输出调试信息

```python
def Update(self, input):
    # ... 计算 ...
    if self.Value > threshold:
        self.Debug(f"Indicator value {self.Value} exceeded threshold")
    return True
```

2. **在 Research 笔记本中单步测试**（见上面的验证部分）

3. **检查 Backtest 日志**：在完整回测后查看日志

4. **添加图表**：在回测结果中绘制指标值

```python
def OnData(self, data):
    if self.symbol in data:
        self.Plot("Indicators", "MyIndicator", self.my_indicator.Current.Value)
```

### Q7: 指标的 WarmUpPeriod 应该设置多少？

**A**: WarmUpPeriod 应该是指标产生有效值需要的最小数据点数。

- **VWAP**：1（因为每个单独的 bar 都有有效的 VWAP）
- **20 周期 SMA**：20
- **组合指标**：所有内部指标的最大 WarmUpPeriod

```python
# 例如，如果你的指标使用 20 周期 SMA 和 14 周期 RSI
self.WarmUpPeriod = max(20, 14)  # = 20
```

### Q8: 我的自定义指标在 Paper Trading 中的表现与回测不同，为什么？

**A**: 几个常见原因：

1. **数据分辨率不匹配**：确保 RegisterIndicator 使用与你的交易逻辑相同的分辨率
2. **时间戳问题**：Paper Trading 使用实时时间戳，可能与历史数据不同步
3. **成交量数据**：如果依赖成交量，实时成交量数据可能与历史数据不同
4. **市场时间**：确认你在正确的市场小时内运行

**调试建议**：
- 在 Paper Trading 中也使用 self.Debug() 输出
- 检查算法日志中的时间戳
- 使用较小的头寸进行初始测试

---

## 总结

自定义指标是进入 LEAN 二次开发的最简单大门。通过这些 Lab，你应该已经理解了：

1. ✅ 如何继承 PythonIndicator 并实现 Update()
2. ✅ 如何管理指标的内部状态（累积值、窗口等）
3. ✅ 如何将多个指标组合成更复杂的信号
4. ✅ 如何在策略中注册和使用自定义指标
5. ✅ 如何测试和验证指标的正确性

下一步，你已经准备好进入 **数据源扩展**（Lab 07），这是关于将任意外部数据集成到 LEAN 的系统。

---

## 导航

**[上一篇: 研究环境与数据探索](./05-research-and-data.md)** | **[下一篇: 二开进阶：自定义数据源](./07-custom-datasource.md)**

**[架构参考：数据管线](../quantconnect-deep-dive/03-data-pipeline.md)**
