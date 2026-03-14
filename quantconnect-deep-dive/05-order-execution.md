# LEAN订单执行与现实建模系统

[上一篇: Algorithm Framework 五层流水线](./04-algorithm-framework.md) | [下一篇: 回测与实盘的统一抽象](./06-backtest-vs-live.md)

## 目录

1. [为什么订单执行如此重要](#为什么订单执行如此重要)
2. [订单类型全景](#订单类型全景)
3. [订单生命周期](#订单生命周期)
4. [TransactionHandler 架构](#transactionhandler-架构)
5. [Reality Modeling 体系](#reality-modeling-体系)
6. [BrokerageModel 统一入口](#brokeragemodel-统一入口)
7. [回测中的常见陷阱](#回测中的常见陷阱)
8. [从前端视角理解](#从前端视角理解)

---

## 为什么订单执行如此重要

在量化交易中，有一个残酷的事实：**纸面收益和实际收益之间往往相差巨大**。

### 纸面上的完美策略

假设你设计了一个策略：
- 逻辑严密，参数优化得无可挑剔
- 回测夏普率高达2.5
- 最大回撤仅10%

但当这个策略上线运行时，你发现：
- 实际收益比回测低40%
- 滑点比预期多3倍
- 手续费吃掉了大部分利润
- 大额订单经常无法按预期价格成交

### 为什么会这样？

这就像前端开发中的"localhost vs production"问题：

| localhost (回测) | production (实盘) |
|---|---|
| 假设订单能完美成交 | 市场流动性有限 |
| 没有网络延迟 | 有真实的订单延迟 |
| 不考虑手续费 | 要支付真实手续费 |
| 认为价格瞬间变化 | 价格变化有成本(滑点) |
| 资金无限制使用 | 要遵守保证金规则 |

LEAN框架的核心洞察就是：**不要等到实盘才发现这些问题，在回测阶段就要精确模拟现实**。这就是"Reality Modeling"的意义。

### Reality Modeling 的四个支柱

```
┌─────────────────────────────────────────────┐
│        Strategy Algorithm                   │
└────────────────┬────────────────────────────┘
                 │ Orders
        ┌────────▼────────┐
        │ FillModel       │  ← 订单如何成交
        │ SlippageModel   │  ← 成交价格如何偏离
        │ FeeModel        │  ← 交易成本
        │ BuyingPowerModel│  ← 资金限制
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │ Realistic P&L   │
        └─────────────────┘
```

如果忽视这些模型，你的回测结果就像一段"只在localhost通过的测试代码"。

---

## 订单类型全景

LEAN支持丰富的订单类型，每一种都有特定的使用场景。

### 1. MarketOrder（市价单）

**定义**：立即以市场价格成交

**何时使用**：
- 需要快速成交，不关心具体价格
- 流动性充足的资产
- 急速进出头寸

**特点**：
- 几乎100%确保成交
- 成交价格可能与下单价格相差较大（滑点）
- 无法控制成交价格

**Python示例**：

```python
def initialize(self):
    self.add_equity("SPY", Resolution.Daily)

def on_data(self, slice):
    if not self.portfolio.invested:
        # 以市价立即买入100股
        self.market_order("SPY", 100)
```

**价格行为图示**：

```
价格 │                     ↗ 成交价格（可能更高）
    │                    /
    │                   /
    │ ─────────────────  下单时价格
    │                  /
    │                 /
    │
    └──────────────────── 时间
      下单    立即成交
```

---

### 2. LimitOrder（限价单）

**定义**：仅在价格达到指定限价或更优时才成交

**何时使用**：
- 想控制成交价格
- 愿意为了更好的价格等待
- 有明确的进出价位

**特点**：
- 成交价格可控
- 可能无法成交（如果价格没有触及限价）
- 适合有耐心的策略

**Python示例**：

```python
def on_data(self, slice):
    # 仅当价格≤95时才买入
    if slice["SPY"].close > 95:
        self.limit_order("SPY", 100, 95)

    # 卖出指令：仅当价格≥110时才卖
    if self.portfolio.invested and slice["SPY"].close < 110:
        self.limit_order("SPY", -100, 110)
```

**价格行为图示**：

```
价格 │
    │  110 ─── 触及，订单执行！
    │  /\      /
    │ /  \    /
    │/    \  /
    │      \/
    │  95 ─── 限价线，仅在这里或更好的价格执行
    │
    └──────────────────── 时间
      下单   等待   触及   成交
```

---

### 3. StopMarketOrder（止损市价单）

**定义**：当价格触及止损位后，以市价单执行

**何时使用**：
- 需要快速止损
- 不想超过某个价格的损失
- 优先级：止损 > 成交价格

**特点**：
- 被触发后以市价成交，速度快
- 成交价格可能远低于止损价（在快速下跌时）
- 风险："突破止损"可能造成更大损失

**Python示例**：

```python
def on_data(self, slice):
    if not self.portfolio.invested:
        self.market_order("SPY", 100)
    else:
        # 设置止损：如果价格跌至95以下，以市价卖出
        self.stop_market_order("SPY", -100, 95)
```

**价格行为图示**：

```
价格 │
    │  105 ← 买入价格
    │   /\
    │  /  \___
    │       95 ← 止损线触发
    │       |\  以市价成交！
    │       | \
    │       |  \ 可能跌至90甚至更低
    │       |
    │      90
    │
    └──────────────────── 时间
      买入   等待   触发  成交
```

---

### 4. StopLimitOrder（止损限价单）

**定义**：当价格触及止损位后，以限价单执行

**何时使用**：
- 需要控制止损的成交价格
- 不想在快速下跌时"滑脱"太远
- 愿意冒不成交的风险

**特点**：
- 被触发后变成限价单
- 可能止损失败（触发后仍未达到限价）
- 比止损市价单更安全，但成交概率低

**Python示例**：

```python
def on_data(self, slice):
    if not self.portfolio.invested:
        self.market_order("SPY", 100)
    else:
        # 触发价95，但仅在94或更高价格成交
        # 如果快速跌至92，可能无法成交！
        self.stop_limit_order("SPY", -100, stop_price=95, limit_price=94)
```

---

### 5. MarketOnOpenOrder / MarketOnCloseOrder

**定义**：指定在开盘或收盘时执行的市价单

**何时使用**：
- 想在特定时间点执行
- 利用开盘/收盘的流动性

**Python示例**：

```python
def on_data(self, slice):
    # 在明天开盘时以市价买入
    self.market_on_open_order("SPY", 100)

    # 在今天收盘时以市价卖出
    self.market_on_close_order("SPY", -100)
```

---

### 6. LimitIfTouchedOrder（触及即限价单）

**定义**：当价格触及某个位置后，激活一个限价单

**何时使用**：
- 需要分步骤进出
- 基于某个触发条件设置限价单

**Python示例**：

```python
# 如果价格触及100，则在95位置挂出卖单
# 适合"跌到某个高点就设置卖单"的策略
self.limit_if_touched_order("SPY", -100, trigger_price=100, limit_price=95)
```

---

### 7. TrailingStopOrder（追踪止损单）

**定义**：止损线随价格上升而跟随（但不下降）

**何时使用**：
- 想锁定利润，同时保留上行空间
- 趋势跟随策略

**Python示例**：

```python
def on_data(self, slice):
    if not self.portfolio.invested:
        self.market_order("SPY", 100)
    else:
        # 设置追踪止损：距离当前价格3%
        # 如果价格从105上升到110，止损线会从102上升到107
        # 这样可以锁定利润同时让趋势继续
        self.trailing_stop_order("SPY", -100, trailing_percent=0.03)
```

**价格行为图示**：

```
价格 │
    │         110 ← 新高
    │        /\     止损线也上升到107
    │       /  \
    │  105 /    \ 止损线 = 当前价 * (1 - 0.03)
    │  /\/
    │ /  \
    │     \
    │      102 ← 初始止损线
    │
    └──────────────────── 时间
```

---

### 8. OptionExerciseOrder（期权行权单）

**定义**：行使期权合约

**何时使用**：
- 期权即将过期
- 期权深度价内

**Python示例**：

```python
# 行使看涨期权
self.exercise_option("SPY_CALL", contract_symbol, 100)
```

---

### 9. Combo 订单（组合订单）

**定义**：同时下达多条腿的订单（如期权价差组合）

**类型**：
- `ComboMarketOrder`：所有腿以市价成交
- `ComboLimitOrder`：组合限价
- `ComboLegLimitOrder`：每腿独立限价

**Python示例**：

```python
# 买入看涨期权，同时卖出更高行权价的看涨期权（牛市价差）
legs = [
    ComboLegData(OptionSymbol1, 1, OrderDirection.Buy),
    ComboLegData(OptionSymbol2, -1, OrderDirection.Sell)
]
self.combo_market_order(legs, 1)
```

---

## 订单生命周期

订单从创建到完成，要经历一系列状态变化。理解这个状态机对于正确处理订单至关重要。

### 订单状态图

```
                    ┌─────────────────┐
                    │      New        │
                    └────────┬────────┘
                             │
                             ├─── Invalid
                             │
                    ┌────────▼────────┐
                    │   Submitted     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
         ┌──────────┤ PartiallyFilled │
         │          └─────────────────┘
         │
    ┌────▼─────┐
    │  Updated │ (价格或数量改变)
    └──────────┘
         │
    ┌────▼─────┐
    │ Filled    │ (完全成交 ✓)
    └───────────┘

    其他路径：
    Submitted ──→ Canceled
    New ──→ Canceled
    Submitted ──→ Invalid
```

### 订单状态详解

**New（新订单）**
- 刚创建，尚未提交给成交系统
- 可以在此状态下修改或取消
- 对应的OrderEvent：订单创建完成

**Submitted（已提交）**
- 订单已提交给成交处理引擎
- 等待成交或拒绝
- 此时通常无法修改

**PartiallyFilled（部分成交）**
- 订单的一部分已成交，一部分仍在等待
- 例如：买100股，已成交60股，还剩40股待成交
- 可以选择取消剩余部分

**Filled（完全成交）**
- 订单100%成交
- 最终状态（不可逆）
- 头寸已更新，资金已转移

**Updated（已更新）**
- 订单在已提交后被修改（价格或数量）
- 重新提交给成交系统
- 可能用于调整止损或扩大头寸

**Canceled（已取消）**
- 订单被主动取消或被系统拒绝
- 可能是用户请求，也可能是因为条件无法满足
- 最终状态

**Invalid（无效）**
- 订单不合法或违反规则
- 例如：尝试买入不存在的资产，或违反Pattern Day Trader规则
- 最终状态

### OrderEvent 对象

每次订单状态变化都会产生一个`OrderEvent`对象，携带关键信息：

```python
class OrderEvent:
    OrderId: int              # 订单唯一ID
    Symbol: str               # 资产代码
    UtcTime: datetime        # 事件时间
    Status: OrderStatus      # 当前状态
    FillQuantity: float      # 本次成交数量
    FillPrice: float         # 成交价格
    OrderFee: OrderFee       # 手续费
    Direction: OrderDirection # 买/卖
    IsAssignment: bool       # 是否是期权行权

def on_order_event(self, order_event):
    if order_event.Status == OrderStatus.Filled:
        self.log(f"订单 {order_event.OrderId} 完全成交！")
        self.log(f"成交价格: {order_event.FillPrice}")
        self.log(f"成交数量: {order_event.FillQuantity}")
        self.log(f"手续费: {order_event.OrderFee.Value}")
    elif order_event.Status == OrderStatus.PartiallyFilled:
        self.log(f"订单部分成交，已成交 {order_event.FillQuantity} 股")
    elif order_event.Status == OrderStatus.Canceled:
        self.log(f"订单被取消")
```

---

## TransactionHandler 架构

订单的流动，从算法生成到最终成交，涉及一整套处理流程。

### 架构图

```
┌──────────────────────────┐
│ Algorithm                │
│ self.market_order(...)   │
└────────────┬─────────────┘
             │
      ┌──────▼──────┐
      │ OrderProcessor
      │ - 订单验证
      │ - 订单去重
      │ - 提交队列
      └──────┬──────┘
             │
   ┌─────────▼──────────┐
   │ ITransactionHandler│
   │                    │
   ├─ BacktestingTH    │ (回测模式)
   │  - 调用FillModel
   │  - 调用SlippageModel
   │  - 调用FeeModel
   │  - 模拟成交
   │                    │
   └─ BrokerageH       │ (实盘模式)
      - 调用真实API
      - 提交给券商
      - 接收成交回执

             │
      ┌──────▼──────┐
      │ Portfolio   │
      │ - 更新头寸
      │ - 更新资金
      └─────────────┘
```

### ITransactionHandler 接口

```python
class ITransactionHandler:
    """订单处理的抽象接口"""

    def process_order(self, order):
        """处理一个订单"""
        pass

    def get_open_orders(self):
        """获取所有未成交订单"""
        pass

    def cancel_order(self, order):
        """取消一个订单"""
        pass
```

### BacktestingTransactionHandler（回测处理器）

在回测中，LEAN不能真正提交订单给真实市场。取而代之，它使用`BacktestingTransactionHandler`来模拟订单成交过程：

```python
class BacktestingTransactionHandler(ITransactionHandler):
    """
    回测模式下的订单处理
    - 不提交给真实券商
    - 使用FillModel决定如何成交
    - 使用SlippageModel模拟滑点
    - 使用FeeModel计算手续费
    """

    def process_order(self, order):
        # 1. 检查订单是否合法
        if not self.validate(order):
            order.status = OrderStatus.Invalid
            return

        # 2. 调用FillModel决定成交
        fill = self.fill_model.fill_order(order, current_bar)

        # 3. 应用滑点
        slippage = self.slippage_model.calculate(
            order.quantity,
            self.get_market_volume()
        )
        fill.fill_price += slippage

        # 4. 计算手续费
        fee = self.fee_model.calculate(order, fill.fill_price)

        # 5. 检查是否有足够资金
        if not self.buying_power_model.has_sufficient_power(order):
            order.status = OrderStatus.Canceled
            return

        # 6. 执行成交
        order.status = OrderStatus.Filled
        portfolio.process_fill(fill, fee)
```

### BrokerageTransactionHandler（券商处理器）

在实盘交易中，订单会被提交给真实的券商API。

```python
class BrokerageTransactionHandler(ITransactionHandler):
    """
    实盘模式下的订单处理
    - 直接提交给券商API
    - 异步接收成交回执
    - 处理拒绝和部分成交
    """

    def process_order(self, order):
        # 1. 验证订单
        if not self.validate(order):
            order.status = OrderStatus.Invalid
            return

        # 2. 提交给券商
        try:
            broker_response = self.brokerage.submit_order(order)
            order.broker_order_id = broker_response.order_id
            order.status = OrderStatus.Submitted
        except Exception as e:
            order.status = OrderStatus.Invalid
            self.log(f"订单提交失败: {e}")

    def on_order_update(self, broker_update):
        """接收来自券商的成交更新"""
        order = self.get_order(broker_update.order_id)
        order.fill_quantity += broker_update.fill_qty

        if order.fill_quantity == order.quantity:
            order.status = OrderStatus.Filled
        else:
            order.status = OrderStatus.PartiallyFilled

        portfolio.process_fill(order)
```

### 线程安全性

在实盘交易中，订单可能从多个地方同时提交：
- 主算法线程
- 事件回调线程
- 手动干预线程

LEAN通过以下方式保证线程安全：

```python
import threading

class ThreadSafeOrderProcessor:
    def __init__(self):
        self.lock = threading.RLock()
        self.pending_orders = []

    def submit_order(self, order):
        """线程安全的订单提交"""
        with self.lock:  # 获取锁
            self.validate(order)
            self.pending_orders.append(order)
            self.transaction_handler.process_order(order)
            # 释放锁

    def cancel_order(self, order_id):
        """线程安全的订单取消"""
        with self.lock:
            order = self.get_order(order_id)
            self.transaction_handler.cancel_order(order)
```

---

## Reality Modeling 体系

这是LEAN框架中最强大的部分。它通过一系列模型，将实盘交易的复杂性引入回测环境。

### Reality Modeling 的五个组件

```
                    ┌─────────────────────────┐
                    │  Strategy Order         │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
    ┌─────▼──────┐        ┌─────▼──────┐        ┌─────▼──────┐
    │ FillModel  │        │FeeModel    │        │BuyingPower │
    │            │        │            │        │Model       │
    │订单如何成交│        │手续费成本  │        │资金限制    │
    └─────┬──────┘        └─────┬──────┘        └─────┬──────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │ SlippageModel          │
                    │ 滑点(价格偏离)         │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │ SettlementModel        │
                    │ T+1/T+2结算            │
                    └────────────────────────┘
```

### 1. FillModel（成交模型）

**问题**：订单在什么时候、以什么价格成交？

**回答**：这就是FillModel的职责。

#### ImmediateFillModel（即时成交模型）

最简单的实现：订单在条形图收盘时立即成交。

**缺点**：太不现实，会导致：
- 过度乐观的订单成功率
- 忽视日内波动
- 看前瞻偏差（知道当天收盘价）

```python
class ImmediateFillModel(IFillModel):
    """订单在bar收盘立即成交"""

    def fill_order(self, order, bar):
        """
        Args:
            order: 订单对象
            bar: 当前K线 (Open, High, Low, Close, Volume)
        """
        # 对于买单，检查是否能在当天成交
        if order.direction == OrderDirection.Buy:
            # 简单地用收盘价成交
            fill_price = bar.close
            fill_quantity = order.quantity
        else:  # 卖单
            fill_price = bar.close
            fill_quantity = order.quantity

        return Fill(fill_price, fill_quantity)
```

**问题**：如果订单是限价单呢？`ImmediateFillModel`无法正确处理。

#### EquityFillModel（股票成交模型）

更现实的实现，考虑OHLC和成交量：

```python
class EquityFillModel(IFillModel):
    """
    根据K线的OHLC数据模拟成交
    - 检查限价是否被触及
    - 考虑成交量是否充足
    - 处理部分成交
    """

    def fill_order(self, order, bar):

        # 1. 市价单：直接用close成交（或open）
        if order.type == OrderType.Market:
            return self._fill_market_order(order, bar)

        # 2. 限价单：检查限价是否被触及
        if order.type == OrderType.Limit:
            return self._fill_limit_order(order, bar)

        # 3. 止损单：检查止损价是否被触及
        if order.type == OrderType.StopMarket:
            return self._fill_stop_order(order, bar)

        return None  # 订单未成交

    def _fill_limit_order(self, order, bar):
        """限价单成交逻辑"""

        # 买单限价：价格要≤限价
        if order.direction == OrderDirection.Buy:
            # 检查是否在这个bar中触及限价
            if order.limit_price >= bar.low:
                # 限价被触及！
                # 成交价格：取lower (限价, 当日最低价)
                fill_price = min(order.limit_price, bar.close)
                return Fill(fill_price, order.quantity)

        # 卖单限价：价格要≥限价
        else:
            if order.limit_price <= bar.high:
                # 限价被触及！
                fill_price = max(order.limit_price, bar.close)
                return Fill(fill_price, order.quantity)

        return None  # 限价未被触及，不成交

    def _fill_market_order(self, order, bar):
        """市价单成交逻辑"""

        # 考虑成交量
        if bar.volume == 0:
            return None  # 无成交量，拒绝

        # 检查是否有足够的流动性
        if order.quantity > bar.volume * 0.1:  # 不能超过当日成交量的10%
            # 部分成交
            actual_quantity = int(bar.volume * 0.1)
            return Fill(bar.close, actual_quantity, is_partial=True)

        # 充足流动性，完全成交
        return Fill(bar.close, order.quantity)
```

**止损单的成交逻辑**：

```python
def _fill_stop_order(self, order, bar):
    """止损市价单成交逻辑"""

    # 买入止损（在价格上升时触发）
    if order.direction == OrderDirection.Buy:
        if bar.high >= order.stop_price:
            # 止损被触发，以市价成交
            fill_price = max(order.stop_price, bar.close)
            return Fill(fill_price, order.quantity)

    # 卖出止损（在价格下跌时触发）
    else:
        if bar.low <= order.stop_price:
            # 止损被触发，可能成交价会远低于止损价（快速下跌）
            # 为了现实，假设成交价是：stop_price 或更低
            fill_price = min(order.stop_price, bar.close)
            return Fill(fill_price, order.quantity)

    return None
```

#### 自定义FillModel

你可以创建自己的成交模型来模拟特定的交易所或资产行为：

```python
class MyCustomFillModel(IFillModel):
    """
    自定义成交模型示例：
    - 模拟订单簿深度
    - 考虑市场微观结构
    - 模拟做市商行为
    """

    def fill_order(self, order, bar, order_book_depth=None):

        # 如果有订单簿数据，使用真实的买卖盘
        if order_book_depth:
            return self._fill_from_order_book(order, order_book_depth)

        # 否则使用OHLC数据
        return self._fill_from_ohlc(order, bar)

    def _fill_from_order_book(self, order, depth):
        """根据订单簿成交"""

        if order.direction == OrderDirection.Buy:
            # 从卖盘（ask）取流动性
            asks = depth.asks  # [(price, volume), ...]

            remaining_quantity = order.quantity
            total_cost = 0

            for ask_price, ask_volume in asks:
                if remaining_quantity == 0:
                    break

                # 这个价位能成交多少
                fill_qty = min(remaining_quantity, ask_volume)
                total_cost += fill_qty * ask_price
                remaining_quantity -= fill_qty

            if total_cost > 0:
                avg_price = total_cost / (order.quantity - remaining_quantity)
                return Fill(avg_price, order.quantity - remaining_quantity)

        return None

    def _fill_from_ohlc(self, order, bar):
        """回退到OHLC数据"""
        # 使用EquityFillModel的逻辑
        pass
```

### 2. SlippageModel（滑点模型）

**问题**：为什么成交价格会比预期差？

**答案**：市场冲击、流动性成本、做市商价差。

#### ConstantSlippageModel（固定滑点模型）

最简单的模型：每笔订单都有固定的滑点。

```python
class ConstantSlippageModel(ISlippageModel):
    """固定滑点模型"""

    def __init__(self, slippage_percent=0.001):
        """
        Args:
            slippage_percent: 滑点比例（默认0.1%）
        """
        self.slippage_percent = slippage_percent

    def calculate(self, order, fill):
        """
        计算滑点并修正成交价格
        """

        slippage = fill.price * self.slippage_percent

        # 买单：滑点向上（买得更贵）
        if order.direction == OrderDirection.Buy:
            fill.price += slippage

        # 卖单：滑点向下（卖得更便宜）
        else:
            fill.price -= slippage

        return fill
```

**使用示例**：

```python
def set_slippage(self, model):
    """在initialize中设置"""
    self.set_slippage(ConstantSlippageModel(0.002))  # 0.2%固定滑点
```

#### VolumeShareSlippageModel（成交量比例滑点模型）

更现实的模型：滑点与订单规模和市场成交量有关。

```python
class VolumeShareSlippageModel(ISlippageModel):
    """
    基于成交量比例的滑点模型

    核心思想：你的订单越大，相对市场成交量，滑点越大
    这反映了市场微观结构：大订单会移动价格
    """

    def calculate(self, order, fill, market_volume):
        """
        Args:
            order: 订单对象
            fill: 成交对象
            market_volume: 当日成交量
        """

        # 订单所占市场的比例
        volume_share = order.quantity / market_volume

        # 滑点 = 基础滑点 + 成交量比例惩罚
        # 例如: 0.1% + 50% * volume_share
        base_slippage = 0.001
        volume_penalty = 0.005 * volume_share  # 每1%的volume_share增加0.5%滑点

        total_slippage = (base_slippage + volume_penalty) * fill.price

        if order.direction == OrderDirection.Buy:
            fill.price += total_slippage
        else:
            fill.price -= total_slippage

        return fill
```

**为什么这很重要？**

想象你有一个"赚钱"的策略：
- 每笔交易利润率2%
- 但成交量比例滑点可能高达1.5%
- 结果：净利润只有0.5%

在不考虑这一点的回测中，你会看到美好的2%回报。而在实盘中，你只会得到0.5%。

#### 自定义SlippageModel

```python
class MyAdvancedSlippageModel(ISlippageModel):
    """
    高级滑点模型：考虑市场微观结构、波动率、时间
    """

    def calculate(self, order, fill, context):
        """
        Args:
            context: 包含以下信息的上下文
                - market_volume
                - volatility
                - bid_ask_spread
                - time_of_day
        """

        base_slippage = context.bid_ask_spread / 2  # 至少是价差的一半

        # 波动率越高，滑点越大（做市商保护自己）
        volatility_factor = context.volatility * 0.01

        # 成交量越小，滑点越大（流动性越低）
        volume_factor = 1000000 / max(context.market_volume, 100000)

        # 日中时段滑点：开盘/收盘最高
        time_factor = self._get_time_factor(context.time_of_day)

        total_slippage = (
            base_slippage +
            base_slippage * volatility_factor +
            base_slippage * volume_factor +
            base_slippage * time_factor
        ) * fill.price

        if order.direction == OrderDirection.Buy:
            fill.price += total_slippage
        else:
            fill.price -= total_slippage

        return fill
```

### 3. FeeModel（手续费模型）

**问题**：交易要花多少钱？

**答案**：这取决于你的券商和交易所。

#### InteractiveBrokersFeeModel（IB手续费模型）

```python
class InteractiveBrokersFeeModel(IFeeModel):
    """
    模拟Interactive Brokers的手续费结构：
    - 股票：$1/交易（最少$1，最多0.2%）
    - 期权：$0.65/合约（最少$0.65）
    """

    def calculate(self, order, fill):
        """
        Args:
            order: 订单
            fill: 成交

        Returns:
            OrderFee: 手续费对象
        """

        security = self.get_security(order.symbol)

        if security.type == SecurityType.Equity:
            # 股票：$1/交易，但不超过成交金额的0.2%
            min_fee = 1.0
            max_fee = fill.price * fill.quantity * 0.002

            fee = max(min_fee, min(max_fee, 1.0))

        elif security.type == SecurityType.Option:
            # 期权：$0.65/合约
            fee = 0.65 * fill.quantity

        else:
            fee = 0

        return OrderFee(fee)
```

#### 现实情景

考虑一个日交易策略：
- 每天进出SPY 100次
- 每笔交易IB手续费$1
- 日成本：$100

如果你的日均利润只有$50，这个策略就亏损了。

但在某些回测中（如果没有正确设置FeeModel），手续费可能被忽视，导致看起来很赚钱。

#### AlpacaFeeModel / CryptoBinanceFeeModel

```python
class AlpacaFeeModel(IFeeModel):
    """美股免佣金（maker和taker都是0）"""

    def calculate(self, order, fill):
        return OrderFee(0)  # 完全免费！


class BinanceFeeModel(IFeeModel):
    """币圈Binance的费用"""

    def calculate(self, order, fill):
        # Maker: 0.1%, Taker: 0.1%
        # 在没有更多信息时，假设是taker
        fee_percent = 0.001
        fee = fill.price * fill.quantity * fee_percent
        return OrderFee(fee)
```

#### 自定义FeeModel

```python
class MyBrokerFeeModel(IFeeModel):
    """
    自定义费用模型：
    - 交易次数越多，折扣越多
    - 交易量越大，费用越低
    """

    def __init__(self):
        self.trade_count = 0
        self.monthly_volume = 0

    def calculate(self, order, fill):

        # 基础费率
        base_fee_percent = 0.001

        # 交易次数折扣
        if self.trade_count > 100:
            base_fee_percent *= 0.8  # 20%折扣
        if self.trade_count > 1000:
            base_fee_percent *= 0.6  # 40%折扣

        # 成交量折扣
        order_value = fill.price * fill.quantity
        if self.monthly_volume > 100000:
            base_fee_percent *= 0.5  # 50%折扣

        fee = order_value * base_fee_percent

        # 更新统计
        self.trade_count += 1
        self.monthly_volume += order_value

        return OrderFee(fee)
```

### 4. BuyingPowerModel（购买力模型）

**问题**：我有多少钱可以交易？

**答案**：这取决于现金、保证金规则和规章制度。

#### CashBuyingPowerModel（现金账户）

最简单的模型：你只能用现金购买，不能融资。

```python
class CashBuyingPowerModel(IBuyingPowerModel):
    """
    现金账户：
    - 只能用现金买入
    - 不能融资做空
    - 不能融资加杠杆
    """

    def get_buying_power(self, portfolio):
        """可用资金就是现金余额"""
        return portfolio.cash

    def can_submit_order(self, order, portfolio):
        """检查是否能提交订单"""

        if order.direction == OrderDirection.Buy:
            # 买入：检查是否有足够现金
            cost = order.quantity * self.get_price(order.symbol)
            return cost <= portfolio.cash
        else:
            # 卖出：检查是否持有足够股票
            holdings = portfolio.get_holdings(order.symbol)
            return order.quantity <= holdings.quantity
```

#### SecurityMarginModel（保证金账户）

更现实的模型：允许融资杠杆。

```python
class SecurityMarginModel(IBuyingPowerModel):
    """
    保证金账户：
    - 允许融资买入
    - 允许融资做空
    - 维持保证金要求

    标准美股保证金规则：
    - 初始保证金：50%（买$100需要$50现金）
    - 维持保证金：25%（头寸价值下跌时，保证金不能低于25%）
    """

    def __init__(self, initial_margin=0.5, maintenance_margin=0.25):
        self.initial_margin = initial_margin
        self.maintenance_margin = maintenance_margin

    def get_buying_power(self, portfolio):
        """
        可用购买力 =
            (现金 + 证券价值) * 杠杆倍数 - 已占用购买力
        """

        total_value = portfolio.cash + portfolio.portfolio_value

        # 最大购买力 = 总资产 / 初始保证金
        max_buying_power = total_value / self.initial_margin

        # 已用购买力 = 融资金额（如果有的话）
        used_power = self.get_used_buying_power(portfolio)

        # 可用 = 最大 - 已用
        return max(0, max_buying_power - used_power)

    def can_submit_order(self, order, portfolio):
        """检查是否能提交订单"""

        if order.direction == OrderDirection.Buy:
            # 所需购买力
            required_power = order.quantity * self.get_price(order.symbol) * self.initial_margin
            available_power = self.get_buying_power(portfolio)
            return required_power <= available_power
        else:
            # 卖出通常不受限制，除非有空头覆盖要求
            return True

    def check_maintenance_requirement(self, portfolio):
        """检查是否违反维持保证金要求"""

        for position in portfolio.positions:
            current_value = position.quantity * position.price
            required_margin = current_value * self.maintenance_margin
            available_equity = portfolio.cash + portfolio.portfolio_value

            if available_equity < required_margin:
                # 违反维持保证金 → margin call!
                return False

        return True
```

**杠杆示例**：

```
假设你有$10,000现金，2:1保证金（初始保证金50%）

买入：
- 可购买价值 = $10,000 / 50% = $20,000的股票
- 这意味着你买了$20,000的股票，借入$10,000

维持保证金：
- 假设股票跌到$18,000
- 你的净资产 = $10,000 + ($18,000 - $10,000) = $8,000
- 维持保证金要求 = $18,000 * 25% = $4,500
- 你有$8,000 > $4,500，OK，还能继续

如果继续跌到$15,000：
- 净资产 = $10,000 + ($15,000 - $10,000) = $5,000
- 维持保证金要求 = $15,000 * 25% = $3,750
- 你有$5,000 > $3,750，仍OK

如果跌到$13,000：
- 净资产 = $10,000 + ($13,000 - $10,000) = $3,000
- 维持保证金要求 = $13,000 * 25% = $3,250
- 你有$3,000 < $3,250，VIOLATION! Margin call!
```

#### PatternDayTraderModel（日内交易规则）

美国SEC的Pattern Day Trader (PDT)规则：
- 5个交易日内有4或以上的日内往返交易 → 必须维持$25,000最低资产

```python
class PatternDayTraderModel(IBuyingPowerModel):
    """
    PDT规则模拟
    """

    def __init__(self, min_equity_requirement=25000):
        self.min_equity_requirement = min_equity_requirement
        self.trades = []  # 记录最近的交易

    def can_submit_order(self, order, portfolio):
        """检查是否违反PDT规则"""

        # 检查账户是否满足最低要求
        if portfolio.total_portfolio_value < self.min_equity_requirement:
            # 低于$25k → 受PDT限制
            if self.is_day_trade(order):
                return False

        return True

    def is_day_trade(self, order):
        """检查是否是日内交易"""

        # 日内交易 = 同一天内买和卖同一个资产
        for trade in self.trades:
            if (trade.symbol == order.symbol and
                trade.date == today() and
                trade.direction != order.direction):
                return True

        return False
```

### 5. SettlementModel（结算模型）

**问题**：成交后，资金什么时候可用？

**答案**：这取决于结算周期。

#### T+1 / T+2结算

美国股票市场是T+2结算：交易后第二个工作日才能得到资金。

```python
class SettlementModel(ISettlementModel):
    """
    模拟结算周期的影响
    """

    def __init__(self, settlement_days=2):
        self.settlement_days = settlement_days
        self.unsettled_cash = []  # [(amount, settlement_date), ...]

    def get_available_cash(self, portfolio, current_date):
        """获取可用现金（已结算的）"""

        available = portfolio.cash

        for amount, settlement_date in self.unsettled_cash:
            if settlement_date <= current_date:
                available += amount

        return available

    def on_fill(self, fill, current_date):
        """处理成交，记录何时资金可用"""

        settlement_date = current_date + timedelta(days=self.settlement_days)

        if fill.direction == OrderDirection.Sell:
            # 卖出资金在settlement_date可用
            self.unsettled_cash.append((fill.value, settlement_date))
        else:
            # 买入资金立即从现金中扣除（已在成交时扣除）
            pass
```

**现实影响**：

你在周一卖了SPY赚$10,000，想在周一立即用这笔钱买其他股票。但在T+2结算中，资金要到周三才可用。如果你的策略没有考虑这一点，可能会被保证金不足拒绝。

---

## BrokerageModel 统一入口

每个券商的特点都不同。LEAN通过`BrokerageModel`将所有的成交、滑点、手续费、保证金模型整合在一起。

### BrokerageModel 是什么？

```python
class IBrokerageModel:
    """券商模型：集成所有现实约束"""

    @property
    def fill_model(self):
        """如何成交"""
        pass

    @property
    def slippage_model(self):
        """滑点"""
        pass

    @property
    def fee_model(self):
        """手续费"""
        pass

    @property
    def buying_power_model(self):
        """购买力/保证金"""
        pass

    @property
    def settlement_model(self):
        """结算周期"""
        pass
```

### 预设的BrokerageModel

LEAN预装了多个真实券商的模型：

#### InteractiveBrokersBrokerageModel

```python
class InteractiveBrokersBrokerageModel(IBrokerageModel):
    """
    模拟Interactive Brokers：
    - 填充：EquityFillModel
    - 滑点：VolumeShareSlippageModel
    - 手续费：$1/股票，$0.65/期权
    - 保证金：标准保证金（50%初始，25%维持）
    """

    def __init__(self):
        self._fill_model = EquityFillModel()
        self._slippage_model = VolumeShareSlippageModel(0.001)
        self._fee_model = InteractiveBrokersFeeModel()
        self._buying_power_model = SecurityMarginModel(0.5, 0.25)
        self._settlement_model = SettlementModel(2)  # T+2
```

#### AlpacaBrokerageModel

```python
class AlpacaBrokerageModel(IBrokerageModel):
    """
    模拟Alpaca（美股佣金为0）：
    - 填充：EquityFillModel
    - 滑点：0.001 (0.1%)
    - 手续费：$0（免佣金）
    - 保证金：可用1:3杠杆（$2,000最低）
    """
    pass
```

#### CryptoBinanceBrokerageModel

```python
class BinanceBrokerageModel(IBrokerageModel):
    """
    模拟币圈Binance：
    - 填充：CryptoFillModel（24小时市场）
    - 滑点：0.1%（更高，币市流动性差）
    - 手续费：0.1% (maker/taker)
    - 保证金：可用15:1杠杆（期货）
    """
    pass
```

### 在算法中设置BrokerageModel

```python
class MyStrategy(QCAlgorithm):

    def initialize(self):
        self.set_start_date(2020, 1, 1)
        self.set_end_date(2023, 12, 31)
        self.set_cash(10000)

        # 选择券商模型
        self.set_brokerage_model(BrokerageModel.InteractiveBrokers)
        # 或者
        # self.set_brokerage_model(BrokerageModel.Alpaca)
        # 或者
        # self.set_brokerage_model(InteractiveBrokersBrokerageModel())
```

### 为不同资产选择不同模型

```python
class MyStrategy(QCAlgorithm):

    def initialize(self):
        spy = self.add_equity("SPY", Resolution.Daily)
        btc = self.add_crypto("BTCUSD", Resolution.Daily)

        # 为SPY使用IB模型
        self.set_brokerage_model_for_security(
            spy.symbol,
            InteractiveBrokersBrokerageModel()
        )

        # 为BTC使用Binance模型
        self.set_brokerage_model_for_security(
            btc.symbol,
            BinanceBrokerageModel()
        )
```

---

## 回测中的常见陷阱

### 1. 看前瞻偏差（Look-ahead Bias）

**陷阱**：使用当日收盘价成交，但策略在当日开盘时就做了决策。

```python
# ❌ 错误的方式
def on_data(self, slice):
    # 在开盘时看到收盘价信息（不可能！）
    if slice["SPY"].close > 100:
        self.market_order("SPY", 100)  # 这个订单会在什么时候成交？
```

如果使用`ImmediateFillModel`，订单会在当日收盘以收盘价成交。但实际上你是在开盘时做的决策，应该在开盘时以开盘价成交。

**解决方案**：使用`EquityFillModel`并确保理解成交时机。

### 2. 存活者偏差（Survivorship Bias）

**陷阱**：只用仍然存活的公司历史数据进行回测。

```
例如：你的策略在2000-2020年SP 500上表现很好
但这只包括活到2020年的公司
许多在2000-2020年倒闭的公司被排除了
这导致了向上的偏差！
```

LEAN通过`CoarseUniverseSelection`和`FineUniverseSelection`提供历史数据，包括已下市的公司。

### 3. 不现实的成交假设

**陷阱**：假设所有订单都能完全以期望的价格成交。

**举例**：
- 忽视成交量限制（订单远大于当日成交量）
- 忽视流动性（小盘股无法快速成交）
- 忽视限价单可能无法成交

**解决方案**：
```python
def initialize(self):
    # 使用更现实的成交模型
    fill_model = EquityFillModel()
    self.transactions.set_fill_model(fill_model)

    # 检查成交量限制
    self.set_filter(lambda x: x.volume > 100000)
```

### 4. 忽视市场冲击

**陷阱**：假设大额订单不会影响价格。

```
例如：你的策略在某一天想买100万股SPY
但当日成交量只有500万股
你的订单占10%的成交量！
必然会有巨大的滑点

但如果回测中没有使用VolumeShareSlippageModel，
你可能看到$0滑点的假象
```

**解决方案**：
```python
def initialize(self):
    slippage = VolumeShareSlippageModel()
    self.transactions.set_slippage_model(slippage)
```

### 5. 忽视时间风险

**陷阱**：假设在日K线中，成交时间不重要。

```
例如：你的策略在每日收盘时做决策
但实际上从决策到成交有30分钟的延迟
在这30分钟内，价格可能已经变化了
```

**解决方案**：
```python
def initialize(self):
    # 使用分钟级数据获得更高的时间分辨率
    self.add_equity("SPY", Resolution.Minute)

    # 或明确考虑延迟
    self.set_realistic_latency(timedelta(minutes=30))
```

### 6. Pattern Day Trader 规则被忽视

**陷阱**：账户低于$25,000时，仍然进行频繁的日内交易。

```python
# ❌ 在$10,000账户上进行日内交易
class MyDayTradingStrategy(QCAlgorithm):

    def initialize(self):
        self.set_cash(10000)  # 低于$25k最低要求
        # ... 但然后进行日内交易

    def on_data(self, slice):
        # 上午买，下午卖 → PDT规则应该禁止这个！
        if self.portfolio.invested:
            self.sell("SPY", 100)
        else:
            self.buy("SPY", 100)
```

**解决方案**：
```python
def initialize(self):
    # 要么账户$25k+，要么不做日内交易
    self.set_cash(25000)

    # 或使用PDT model检查
    self.set_buying_power_model(PatternDayTraderModel())
```

### LEAN如何解决这些陷阱

LEAN的Reality Modeling系统通过以下方式解决了这些问题：

| 陷阱 | LEAN的解决方案 |
|---|---|
| 看前瞻偏差 | EquityFillModel 精确模拟成交时机 |
| 存活者偏差 | 提供完整历史数据，包括下市公司 |
| 不现实成交 | 检查成交量、流动性 |
| 市场冲击 | VolumeShareSlippageModel |
| 时间风险 | 支持分钟/秒级数据 |
| PDT规则 | PatternDayTraderModel自动检查 |

---

## 从前端视角理解

如果你是一个前端开发者，这些概念可能听起来很陌生。但实际上，它们与前端开发中的"集成测试环境"很相似。

### 类比对照表

| 前端概念 | 量化交易概念 | 目的 |
|---|---|---|
| Mock API | FillModel | 模拟服务返回数据 |
| 网络延迟模拟 | SlippageModel | 模拟现实中的时间成本 |
| API 速率限制 | FeeModel | 模拟调用成本 |
| 资源配额管理 | BuyingPowerModel | 模拟资源限制 |
| 整个测试环境 | BrokerageModel | 完整的集成测试 |

### 详细类比

#### FillModel ≈ API 响应 Mock

```
前端：
├─ 在localhost，API立即返回200 (ImmediateFillModel)
├─ 但在生产环境，可能返回503或超时
├─ 所以要用MockAdapter模拟各种情况
│
量化：
├─ 在简单回测，订单立即成交 (ImmediateFillModel)
├─ 但在真实市场，可能无法成交
├─ 所以要用EquityFillModel模拟各种场景
```

#### SlippageModel ≈ 网络抖动

```
前端：
├─ localhost: API响应时间 < 1ms
├─ 生产: API响应时间 50-200ms + 抖动
├─ 网络抖动导致用户体验变差
│
量化：
├─ 简单假设: 成交价 = 市场价
├─ 现实: 成交价 = 市场价 + 滑点
├─ 滑点导致收益变差
```

#### FeeModel ≈ API 计费

```
前端：
├─ 免费API: 无成本
├─ 付费API: 按调用次数计费
├─ CDN: 按流量计费
├─ 如果忽视计费，ROI会被高估
│
量化：
├─ 0佣金券商（Alpaca): 无成本
├─ 传统券商（IB): 按交易计费
├─ 如果忽视手续费，利润会被高估
```

#### BuyingPowerModel ≈ 资源配额

```
前端：
├─ 免费服务器：磁盘/内存有限
├─ 付费服务器：可升级配额
├─ 如果代码超出配额，应用崩溃
├─ 所以要提前规划资源使用
│
量化：
├─ 现金账户：只能用现金，余额有限
├─ 保证金账户：可借钱，但有限制
├─ 如果超出购买力，订单被拒
├─ 所以要提前规划资金使用
```

#### 整个 Reality Modeling ≈ 集成测试环境

```
前端：
开发流程
  ↓
Unit Tests (单元测试，只测单个函数)
  ↓
Integration Tests (集成测试，测试多个组件)
  ↓
Staging Environment (预发布环境，最接近生产)
  ↓
Production (真实生产环境)

量化：
策略开发
  ↓
理想化回测 (假设完美成交，无滑点)
  ↓
现实化回测 (Reality Modeling，考虑所有成本)
  ↓
纸上交易 (模拟账户，真实市场数据但虚拟交易)
  ↓
实盘交易 (真实账户，真实资金)
```

### 为什么类比很重要？

很多成功的前端工程师在进入量化交易时犯的一个错误是：
- 他们认为"在localhost能跑就能在生产跑"
- 他们忽视了现实中的延迟、成本、限制

类似地，量化新手常犯的错误是：
- 他们认为"回测能赚钱就能实盘赚钱"
- 他们忽视了现实中的滑点、手续费、流动性问题

**Reality Modeling就是量化交易的"集成测试环境"**。

它让你在纸面交易阶段（还没投入真实资金）就发现这些问题。

---

## 总结

### 核心要点

1. **订单执行决定了策略的真实收益**
   - 纸面收益 ≠ 实际收益
   - 差异来自滑点、手续费、流动性

2. **LEAN提供了丰富的订单类型**
   - 市价、限价、止损...
   - 每一种都有特定的用途和陷阱

3. **订单生命周期是一个状态机**
   - 理解状态转换很关键
   - OrderEvent携带关键信息

4. **TransactionHandler 是订单流动的核心**
   - BacktestingTH：模拟成交
   - BrokerageH：真实成交

5. **Reality Modeling 是LEAN的杀手级特性**
   - FillModel: 成交时机和价格
   - SlippageModel: 滑点
   - FeeModel: 手续费
   - BuyingPowerModel: 资金限制
   - SettlementModel: 结算周期
   - 这些组件共同决定了真实的P&L

6. **BrokerageModel 统一了所有配置**
   - 每个券商都有自己的特性
   - 一键切换不同券商

7. **掉进陷阱很容易**
   - 看前瞻、存活者、忽视成本...
   - 但LEAN的现实建模帮你避免了这些

### 下一步

现在你理解了订单如何执行和成交。但在实盘和回测中，有什么根本的区别吗？它们能完全统一吗？

这就是下一篇的主题：[回测与实盘的统一抽象](./06-backtest-vs-live.md)

---

**导航**: [上一篇: Algorithm Framework 五层流水线](./04-algorithm-framework.md) | [下一篇: 回测与实盘的统一抽象](./06-backtest-vs-live.md)

**阅读时间**: 约25-30分钟
**难度**: 中等 (需要理解基础的量化概念，但没有数学推导)
**关键词**: 订单类型、成交模型、滑点、手续费、保证金、Reality Modeling、BrokerageModel
