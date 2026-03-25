# 策略开发基础：API 实操手册

> **目标**：建立 API 速查表，通过 **什么→什么时用→代码→预期结果→常见坑** 的格式，帮你快速参考和解决日常开发问题。这不是文档阅读，而是一份 **做菜谱** —— 复制代码，改参数，看结果。

**需要的基础**：已完成 01 - 第一个策略（对策略模板有基本认识）
**目标时间**：边学边试，2-3 小时
**完成后**：你会有一个可复用的 API 代码片段库，以及一个融合所有 API 的完整实战策略

---

## 目录

1. [算法生命周期](#1-算法生命周期)
2. [数据订阅 API](#2-数据订阅-api)
3. [数据访问 API](#3-数据访问-api)
4. [下单 API](#4-下单-api)
5. [技术指标 API](#5-技术指标-api)
6. [调度 API](#6-调度-api)
7. [投资组合与 Securities API](#7-投资组合与-securities-api)
8. [日志与调试 API](#8-日志与调试-api)
9. [风控 API](#9-风控-api)
10. [综合实战示例](#10-综合实战示例)
11. [常见错误与调试](#11-常见错误与调试)

---

## 1. 算法生命周期

算法从启动到结束会按顺序经过若干"钩子"函数。理解它们的调用顺序至关重要。

### initialize()

**什么**：算法初始化函数，整个生命周期中仅运行 **一次**。

**什么时用**：
- 添加数据订阅（股票、期货、期权等）
- 配置指标
- 设置预热时间
- 初始化变量
- 配置调度任务

**代码例**：
```python
def initialize(self):
    # 设置开始和结束日期
    self.set_start_date(2020, 1, 1)
    self.set_end_date(2023, 12, 31)

    # 初始化现金
    self.set_cash(100000)

    # 添加 SPY 数据，分钟级别
    self.add_equity("SPY", Resolution.MINUTE)

    # 创建 SMA 指标，20 日均线
    self.sma = self.sma("SPY", 20, Resolution.DAILY)

    # 设置预热：在算法真正开始交易前，先用历史数据填充指标
    self.set_warm_up(20)

    # 初始化变量
    self.last_signal = None
```

**预期结果**：
- 策略初始化成功，开始加载数据
- 指标开始用历史数据预热（如果设置了预热）

**常见坑**：
- ❌ 在 `initialize()` 中调用 `market_order()` —— 此时还没有价格数据
- ❌ 忘记设置时间范围 —— 算法无法运行
- ❌ 预热时间太短 —— 指标可能还没准备好

---

### on_data(data)

**什么**：每当收到新数据时调用（数据频率取决于 Resolution 设置：TICK、MINUTE、HOURLY 等）。

**什么时用**：
- 检查信号，决定买卖
- 访问当前价格、成交量
- 调整现有持仓
- 大多数交易逻辑都在这里

**代码例**：
```python
def on_data(self, data):
    # 检查 SPY 数据是否存在
    if not data.contains_key("SPY"):
        return

    # 获取当前价格
    current_price = data["SPY"].close

    # 检查指标是否准备好
    if not self.sma.is_ready:
        return

    # 简单交易信号：价格上穿均线时买入
    if self.last_price < self.sma.current.value and current_price > self.sma.current.value:
        self.market_order("SPY", 100)
        self.debug(f"BUY Signal at {current_price}, SMA={self.sma.current.value}")

    # 价格下穿均线时卖出
    elif self.last_price > self.sma.current.value and current_price < self.sma.current.value:
        self.market_order("SPY", -100)
        self.debug(f"SELL Signal at {current_price}, SMA={self.sma.current.value}")

    # 记录当前价格供下次使用
    self.last_price = current_price
```

**预期结果**：
- 每次收到数据时，你会看到调试信息
- 当条件满足时，订单被提交

**常见坑**：
- ❌ 忘记检查 `data.contains_key()` —— 某些时段某些资产可能没有数据
- ❌ 访问未准备好的指标 —— 使用 `is_ready` 检查
- ❌ 在每个 `on_data` 都下单 —— 可能导致过度交易

---

### on_order_event(order_event)

**什么**：每当订单状态发生变化时调用（提交→成交→部分成交→取消等）。

**什么时用**：
- 记录订单执行情况
- 根据订单结果调整策略
- 检测滑点和填充情况

**代码例**：
```python
def on_order_event(self, order_event):
    # 获取订单对象
    order = self.transactions.get_order_by_id(order_event.order_id)

    if order.status == OrderStatus.FILLED:
        self.debug(f"Order {order.id} FILLED at {order_event.fill_price}, "
                   f"Quantity: {order_event.fill_quantity}")

    elif order.status == OrderStatus.CANCELED:
        self.debug(f"Order {order.id} CANCELED")

    elif order.status == OrderStatus.REJECTED:
        self.debug(f"Order {order.id} REJECTED - Reason: {order.order_events[-1].message}")
```

**预期结果**：
- 你会看到每笔订单的完整生命周期日志

**常见坑**：
- ❌ 假设订单一定会成交 —— 可能被拒绝或部分成交
- ❌ 在此处重复下单 —— 可能导致多次成交

---

### on_end_of_day(symbol)

**什么**：在交易日结束时调用，为指定的 symbol 调用一次。

**什么时用**：
- 日间汇总（计算日收益率）
- 清理每日状态（重置计数器）
- 日志记录

**代码例**：
```python
def on_end_of_day(self, symbol):
    # 计算当日持仓收益
    holdings = self.portfolio[symbol]
    if holdings.quantity > 0:
        profit = holdings.unrealized_profit
        self.debug(f"End of Day for {symbol}: "
                   f"Quantity={holdings.quantity}, Profit={profit:.2f}")
```

**预期结果**：
- 每个交易日结束时，看到持仓汇总

**常见坑**：
- ❌ 试图在此处下单 —— 交易日已结束，不会成交
- ❌ 混淆 `on_end_of_day()` 和 `on_end_of_algorithm()` —— 前者每天调用，后者仅在算法结束时调用一次

---

### on_end_of_algorithm()

**什么**：算法结束时（回测完成或实盘停止）调用一次。

**什么时用**：
- 最终统计
- 清理资源（关闭文件、数据库连接等）
- 最终报告

**代码例**：
```python
def on_end_of_algorithm(self):
    # 计算最终统计
    total_profit = self.portfolio.total_portfolio_value - self.portfolio.cash
    win_rate = self._calculate_win_rate()

    self.debug(f"=== Algorithm Complete ===")
    self.debug(f"Total Profit: ${total_profit:.2f}")
    self.debug(f"Win Rate: {win_rate:.2%}")
    self.debug(f"Final Portfolio Value: ${self.portfolio.total_portfolio_value:.2f}")
```

**预期结果**：
- 回测结束时，看到最终统计信息

**常见坑**：
- ❌ 在此处进行计算密集的操作 —— 算法已结束，不会有收益影响
- ✅ 这是做统计和记录的好地方

---

### on_warmup_finished()

**什么**：指标预热完成时调用。如果没有设置预热或预热完成，会在 `on_data()` 之前调用一次。

**什么时用**：
- 确认指标已准备好
- 开始真正的交易逻辑

**代码例**：
```python
def on_warmup_finished(self):
    self.debug("=== Warmup Finished ===")
    self.debug(f"SMA current value: {self.sma.current.value}")
    self.trading_started = True
```

**预期结果**：
- 看到 "Warmup Finished" 消息，然后 `on_data()` 开始被调用

**常见坑**：
- ❌ 忘记等待预热完成就使用指标 —— 使用 `is_ready` 检查
- ✅ 作为标志点，确认策略已进入实际交易阶段

---

### 完整生命周期示例

让我们把所有钩子函数组合在一起，看清调用顺序：

```python
class LifecycleAlgorithm(QCAlgorithm):

    def initialize(self):
        self.debug("[1] initialize() called - ONCE at startup")
        self.set_start_date(2022, 1, 1)
        self.set_end_date(2022, 12, 31)
        self.set_cash(100000)

        self.add_equity("SPY", Resolution.DAILY)
        self.sma = self.sma("SPY", 20)
        self.set_warm_up(20)

        self.last_price = None
        self.trading_started = False

    def on_warmup_finished(self):
        self.debug("[2] on_warmup_finished() called - ONCE after indicators are ready")
        self.trading_started = True

    def on_data(self, data):
        if not self.trading_started:
            return

        # 只在开盘后第一个数据点打印
        if self.time.hour == 9 and self.time.minute == 30:
            self.debug(f"[3a] on_data() called at {self.time} - EVERY data point")

    def on_end_of_day(self, symbol):
        self.debug(f"[3b] on_end_of_day({symbol}) called - EVERY trading day, once per symbol")

    def on_order_event(self, order_event):
        self.debug(f"[3c] on_order_event() called - WHEN order status changes")
        if order_event.status == OrderStatus.FILLED:
            self.debug(f"     Order filled at {order_event.fill_price}")

    def on_end_of_algorithm(self):
        self.debug("[4] on_end_of_algorithm() called - ONCE at the very end")
        self.debug(f"Final Portfolio Value: ${self.portfolio.total_portfolio_value:.2f}")
```

**输出顺序示例**：
```
[1] initialize() called - ONCE at startup
[2] on_warmup_finished() called - ONCE after indicators are ready
[3a] on_data() called at 2022-01-03 09:30:00 - EVERY data point
[3b] on_end_of_day(SPY) called - EVERY trading day, once per symbol
[3a] on_data() called at 2022-01-04 09:30:00 - EVERY data point
[3b] on_end_of_day(SPY) called - EVERY trading day, once per symbol
[3c] on_order_event() called - WHEN order status changes
     Order filled at 123.45
... (重复数千次，直到回测结束) ...
[4] on_end_of_algorithm() called - ONCE at the very end
Final Portfolio Value: $102345.67
```

---

## 2. 数据订阅 API

订阅数据是策略的第一步。你告诉 QuantConnect 你需要什么数据，然后在 `on_data()` 中访问它。

### add_equity() - 美股

**什么**：订阅美国股票数据。

**什么时用**：
- 交易美国上市股票（纳斯达克、纽约证券交易所等）
- 想要股票的开高低收成交量数据

**代码例**：
```python
def initialize(self):
    # 订阅 SPY（标普 500 ETF），分钟级数据
    self.add_equity("SPY", Resolution.MINUTE)

    # 订阅多只股票，日线级别
    self.add_equity("AAPL", Resolution.DAILY)
    self.add_equity("MSFT", Resolution.DAILY)
    self.add_equity("GOOGL", Resolution.DAILY)

    # 订阅 Tick 级数据（最高精度）
    self.add_equity("TSLA", Resolution.TICK)

def on_data(self, data):
    # 访问已订阅的数据
    if data.contains_key("SPY"):
        current_bar = data["SPY"]
        self.debug(f"SPY: O={current_bar.open}, H={current_bar.high}, "
                   f"L={current_bar.low}, C={current_bar.close}, V={current_bar.volume}")
```

**预期结果**：
```
SPY: O=450.23, H=451.89, L=450.12, C=451.45, V=2500000
```

**常见坑**：
- ❌ 使用错误的 Ticker —— 必须是真实的股票代码
- ❌ 订阅不存在的 symbol —— 会在初始化时报错
- ✅ 使用 `data.contains_key()` 检查数据是否存在

---

### add_forex() - 外汇

**什么**：订阅外汇（外币）数据。

**什么时用**：
- 交易货币对（EURUSD、GBPUSD 等）
- 构建跨国策略

**代码例**：
```python
def initialize(self):
    # 订阅欧元/美元，小时级别
    self.add_forex("EURUSD", Resolution.HOUR)

    # 订阅英镑/美元，分钟级别
    self.add_forex("GBPUSD", Resolution.MINUTE)

def on_data(self, data):
    if data.contains_key("EURUSD"):
        price = data["EURUSD"].close
        self.debug(f"EURUSD: {price}")
```

**预期结果**：
```
EURUSD: 1.0850
```

**常见坑**：
- ❌ 货币对格式错误 —— 必须是 "XXXYYY" 的 6 字母格式
- ✅ 外汇市场 24 小时交易，数据更密集

---

### add_crypto() - 加密货币

**什么**：订阅加密货币数据。

**什么时用**：
- 交易比特币、以太坊等
- 7×24 小时交易策略

**代码例**：
```python
def initialize(self):
    # 订阅比特币，小时级别
    self.add_crypto("BTCUSD", Resolution.HOUR)

    # 订阅以太坊，分钟级别
    self.add_crypto("ETHUSD", Resolution.MINUTE)

def on_data(self, data):
    if data.contains_key("BTCUSD"):
        btc_price = data["BTCUSD"].close
        btc_volume = data["BTCUSD"].volume
        self.debug(f"BTC: Price=${btc_price}, Volume={btc_volume} BTC")
```

**预期结果**：
```
BTC: Price=$42567.89, Volume=1250 BTC
```

**常见坑**：
- ❌ 使用 "BTC" 而不是 "BTCUSD" —— 必须包含美元对
- ❌ 加密数据延迟较多 —— 在实盘中要考虑数据延迟

---

### add_future() - 期货

**什么**：订阅期货合约数据。

**什么时用**：
- 交易股指期货（ES、NQ、YM 等）
- 商品期货（CL 石油、GC 黄金等）
- 利用杠杆放大收益（或风险）

**代码例**：
```python
def initialize(self):
    # 订阅 E-mini S&P 500 期货（最活跃的合约）
    future_sp500 = self.add_future(Futures.Indices.SP500_E_MINI, Resolution.MINUTE)

    # 订阅纳斯达克期货
    future_nq = self.add_future(Futures.Indices.NDAQ_E_MINI, Resolution.MINUTE)

    # 订阅粗油期货
    future_cl = self.add_future(Futures.Commodities.CRUDE_OIL_WTI, Resolution.DAILY)

def on_data(self, data):
    # 访问期货数据（通过 Symbol 对象）
    for contract in self.securities.values():
        if contract.type == SecurityType.FUTURE:
            if data.contains_key(contract.symbol):
                self.debug(f"{contract.symbol}: {data[contract.symbol].close}")
```

**预期结果**：
```
ES (ESZ23): 4567.89
NQ (NQZ23): 15234.56
```

**常见坑**：
- ❌ 使用 "ES" 而不是完整的 Future Symbol —— 期货有到期日期
- ❌ 忘记 QuantConnect 会自动处理合约展期 —— 通常不需要手动切换
- ✅ 期货有杠杆，小资金也能做大头寸

---

### add_option() - 期权

**什么**：订阅期权链（某个标的所有期权）。

**什么时用**：
- 构建期权策略（Call/Put 组合）
- 对冲风险
- 高级交易策略

**代码例**：
```python
def initialize(self):
    # 订阅 AAPL 期权
    option = self.add_option("AAPL", Resolution.MINUTE)

    # 配置期权链过滤器（可选）
    # 例如只订阅 2-3 个月到期的平值期权
    option.set_filter(lambda x: x.strikes(-1, 1).expiration(timedelta(days=60), timedelta(days=90)))

def on_data(self, data):
    # 获取 AAPL 的所有期权合约
    chain = self.option_chain_provider.get_option_contract_list("AAPL", self.time)

    for contract in chain:
        # 访问每个期权的价格（需要该期权订阅过数据）
        if data.contains_key(contract):
            option_data = data[contract]
            self.debug(f"{contract}: {option_data.close}")
```

**预期结果**：
```
AAPL 150.0 C (Call 150 strike): $5.23
AAPL 150.0 P (Put 150 strike): $3.45
```

**常见坑**：
- ❌ 期权 API 比其他资产复杂得多 —— 推荐先学基础交易
- ❌ 订阅太多期权 —— 会大幅减慢回测
- ✅ 使用 `set_filter()` 来限制期权数量

---

### Resolution 枚举 - 数据频率

每条订阅线都需要指定数据频率：

```python
# 从最低频率到最高频率
Resolution.DAILY      # 每天一根 K 线
Resolution.HOUR       # 每小时一根 K 线
Resolution.MINUTE     # 每分钟一根 K 线
Resolution.SECOND     # 每秒一根 K 线（较少使用）
Resolution.TICK       # 每笔成交一个数据点（最高精度，数据最多）
```

**如何选择**：
- `DAILY`：日度回测，较快，指标预热需要的历史数据少
- `MINUTE`：分钟级策略，回测时间适中，常见选择
- `TICK`：高频策略或需要非常精确的执行，回测非常慢
- `HOUR`、`SECOND`：较少使用，针对特定场景

```python
def initialize(self):
    # 日线策略：每天一根 K 线，快速回测
    self.add_equity("SPY", Resolution.DAILY)

    # 分钟级策略：需要更多 CPU 时间，但更接近实盘交易
    self.add_equity("QQQ", Resolution.MINUTE)

    # 高频交易：Tick 级数据，极其耗时
    self.add_equity("TSLA", Resolution.TICK)
```

---

## 3. 数据访问 API

一旦订阅了数据，你需要在 `on_data()` 中访问它。这一节讲如何正确地获取价格、成交量和历史数据。

### data["SPY"] / data.bars["SPY"] - 当前 Bar

**什么**：访问最新的 K 线数据（当前 Bar）。

**什么时用**：
- 获取最新开高低收成交量
- 大多数交易决策都基于这个

**代码例**：
```python
def on_data(self, data):
    # 方法 1：直接使用 data["SPY"]
    if data.contains_key("SPY"):
        bar = data["SPY"]  # 这是一个 TradeBar 对象

        open_price = bar.open
        high_price = bar.high
        low_price = bar.low
        close_price = bar.close
        volume = bar.volume

        self.debug(f"SPY Bar: O={open_price:.2f}, H={high_price:.2f}, "
                   f"L={low_price:.2f}, C={close_price:.2f}, V={volume}")

    # 方法 2：使用 data.bars（同样的数据）
    if data.bars.contains_key("SPY"):
        bar = data.bars["SPY"]
        self.debug(f"SPY Close: {bar.close}")
```

**预期结果**：
```
SPY Bar: O=450.23, H=451.89, L=450.12, C=451.45, V=2500000
SPY Close: 451.45
```

**常见坑**：
- ❌ 不检查 data.contains_key() —— 会导致 KeyError
- ❌ 试图访问不存在的 symbol —— 必须先 add_equity()
- ✅ 始终检查数据存在后再使用

---

### TradeBar 属性详解

TradeBar 对象包含一个时间段（Bar）的所有价格信息：

```python
def on_data(self, data):
    if data.contains_key("SPY"):
        bar = data["SPY"]

        # 开盘价
        bar.open

        # 最高价
        bar.high

        # 最低价
        bar.low

        # 收盘价（最常用）
        bar.close

        # 成交量（股数）
        bar.volume

        # Bar 的时间戳
        bar.time
```

**实例代码**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):
        bar = data["SPY"]

        # 计算当前 Bar 的范围（High - Low）
        bar_range = bar.high - bar.low

        # 判断是否是向上 Bar（Close > Open）
        is_bullish = bar.close > bar.open

        # 判断是否是高成交量 Bar
        is_high_volume = bar.volume > 5000000

        self.debug(f"Range: {bar_range:.2f}, Bullish: {is_bullish}, High Vol: {is_high_volume}")
```

---

### self.securities["SPY"].price - 当前价格

**什么**：直接获取某个 symbol 的当前价格（不需要在 on_data 中）。

**什么时用**：
- 在其他函数中快速获取价格（比如调度函数）
- 计算持仓利润
- 实时价格查询

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)

    # 在调度函数中访问当前价格（不在 on_data 中）
    self.schedule.on(self.date_rules.every_day(),
                      self.time_rules.at(15, 30),
                      self.close_positions)

def close_positions(self):
    # 在调度函数中，可以直接访问当前价格
    spy_price = self.securities["SPY"].price
    self.debug(f"SPY current price: ${spy_price}")

    # 根据当前价格做决策
    if spy_price > 450:
        self.liquidate("SPY")
```

**预期结果**：
```
SPY current price: $451.45
```

**常见坑**：
- ❌ 在回测中，价格总是上一个 Bar 的收盘价 —— 不能用来获取中间价格
- ✅ 在实盘中，这是最准确的当前价格

---

### self.portfolio["SPY"].quantity 和其他持仓信息

**什么**：获取你对某个 symbol 的持仓信息。

**什么时用**：
- 检查你持有多少股
- 计算持仓成本、利润
- 决定是否可以继续加仓

**代码例**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):
        holdings = self.portfolio["SPY"]

        # 持有的股数（正数=多头，负数=空头）
        quantity = holdings.quantity

        # 平均持仓成本
        avg_price = holdings.average_fill_price

        # 持仓的未实现利润（美元）
        unrealized_profit = holdings.unrealized_profit

        # 持仓的未实现利润率（百分比）
        profit_percent = holdings.unrealized_profit_percent

        # 持仓的总价值（数量 × 当前价格）
        holdings_value = holdings.absolute_holdings_value

        self.debug(f"SPY Holdings: Qty={quantity}, Avg={avg_price:.2f}, "
                   f"Profit=${unrealized_profit:.2f} ({profit_percent:.2%})")
```

**预期结果**：
```
SPY Holdings: Qty=100, Avg=450.00, Profit=$145.00 (0.32%)
```

**常见坑**：
- ❌ 假设 `quantity` 是浮点数 —— 它是整数
- ❌ 当 quantity=0 时仍然检查 holdings —— 会返回默认值
- ✅ 始终检查 quantity 是否大于 0

---

### self.portfolio 全局投资组合

**什么**：整个投资组合（所有持仓）的聚合信息。

**什么时用**：
- 获取总资产值
- 计算总收益率
- 风险管理（检查总头寸大小）

**代码例**：
```python
def on_end_of_day(self, symbol):
    portfolio = self.portfolio

    # 总资产值（现金 + 持仓价值）
    total_value = portfolio.total_portfolio_value

    # 现金
    cash = portfolio.cash

    # 总持仓价值
    holdings_value = portfolio.total_holdings_value

    # 总已实现利润（已平仓的利润）
    realized_profit = portfolio.total_realized_profit

    # 总未实现利润（持仓利润）
    unrealized_profit = portfolio.total_unrealized_profit

    # 是否持有任何头寸
    is_invested = portfolio.invested

    self.debug(f"Portfolio: Total=${total_value:.2f}, Cash=${cash:.2f}, "
               f"Holdings=${holdings_value:.2f}, Invested={is_invested}")
```

**预期结果**：
```
Portfolio: Total=$105234.56, Cash=$45234.56, Holdings=$60000.00, Invested=True
```

---

### History() API - 获取历史数据

**什么**：获取某个 symbol 的历史 K 线数据，返回一个 Pandas DataFrame。

**什么时用**：
- 计算历史指标（MA、标准差等）
- 回溯测试某个条件在过去是否成立
- 分析历史相关性

**代码例**：
```python
def on_data(self, data):
    # 每 10 个 Bar 计算一次历史数据
    if self.time.minute % 10 == 0:
        # 获取最近 30 天的日线数据
        history = self.history("SPY", 30, Resolution.DAILY)

        # history 是一个 DataFrame，结构如下：
        #         open    high     low   close      volume
        # 2022-01-03 450.23 451.89 450.12 451.45 2500000
        # 2022-01-04 451.50 452.00 451.20 451.95 2300000
        # ...

        self.debug(f"History shape: {history.shape}")  # (30, 5)

        # 获取收盘价列
        closes = history['close']  # Pandas Series

        # 计算简单移动平均
        sma_20 = closes.mean()

        self.debug(f"20-day SMA: {sma_20:.2f}")
```

**预期结果**：
```
History shape: (30, 5)
20-day SMA: 450.78
```

---

## 4. 下单 API

下单是交易的核心。QuantConnect 支持多种订单类型，每种都有特定用途。

### market_order() - 市价单

**什么**：以当前市场价格立即成交的订单。

**什么时用**：
- 需要立即执行的交易
- 不关心精确填充价格
- 追踪趋势

**代码例**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):
        # 买入 100 股 SPY（正数 = 买入）
        order = self.market_order("SPY", 100)

        self.debug(f"Market order submitted: {order}")

        # 卖出 50 股（正数 = 买入，负数 = 卖出）
        order = self.market_order("SPY", -50)

        # 平仓：检查当前持仓，然后反向下单
        holdings = self.portfolio["SPY"].quantity
        if holdings > 0:
            self.market_order("SPY", -holdings)  # 平多头
```

**预期结果（in on_order_event）**：
```
Order 123 FILLED at 451.45, Quantity: 100
```

**常见坑**：
- ❌ 在每个 on_data 都市价下单 —— 可能导致过度交易
- ❌ 假设一定会成交 —— 在极端行情下可能被拒绝
- ✅ 结合信号和头寸管理，避免重复下单

---

### limit_order() - 限价单

**什么**：只有在达到指定价格时才成交的订单。

**什么时用**：
- 精确控制进场价格
- 等待更好的价格
- 降低成本

**代码例**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):
        current_price = data["SPY"].close

        # 买入 SPY，但只有当价格跌到 450 或更低时才成交
        order = self.limit_order("SPY", 100, 450.00)

        self.debug(f"Limit order: Buy 100 SPY at $450.00 (current: ${current_price:.2f})")

        # 卖出时也可以用限价单：只有价格涨到 455 或更高时才卖
        order = self.limit_order("SPY", -100, 455.00)
```

**预期结果**：
- 如果价格达到限价，订单会成交
- 如果价格不达到限价，订单保持待成交状态，直到超时（通常是下一个交易日）

**常见坑**：
- ❌ 限价设置得太离谱 —— 订单永远不会成交
- ❌ 多个限价单堆积 —— 导致头寸控制混乱
- ✅ 定期检查未成交订单，必要时取消

---

### stop_market_order() - 止损单

**什么**：当价格跌到某个水平时，以市价卖出（或买入）。

**什么时用**：
- 设置止损，限制亏损
- 追踪止损（跟随价格）
- 突破时进场

**代码例**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):
        current_price = data["SPY"].close

        # 买入 100 股
        self.market_order("SPY", 100)

        # 设置止损：如果价格跌到 440，就以市价卖出
        stop_loss = current_price * 0.98  # 2% 止损
        order = self.stop_market_order("SPY", -100, stop_loss)

        self.debug(f"Buy at {current_price:.2f}, Stop Loss at {stop_loss:.2f}")
```

**预期结果**：
- 当价格跌到 440 时，自动以市价卖出 100 股

**常见坑**：
- ❌ 止损设置过近 —— 容易被假突破触发
- ❌ 止损设置过远 —— 亏损过大
- ✅ 通常设置 2-5% 的止损，具体看策略

---

### set_holdings() - 设置头寸比例

**什么**：设定某个 symbol 占总资产的比例，自动调整持仓。

**什么时用**：
- 简单的投资组合权重调整
- 仓位管理（每个股票 5% 的资金）
- 配对交易（同时买入和卖出不同股票达到目标配置）

**代码例**：
```python
def on_data(self, data):
    # 每个交易日重新平衡一次
    if self.time.hour == 10 and self.time.minute == 0:
        # 配置 SPY 占总资产的 50%，QQQ 占 30%，其余为现金
        self.set_holdings("SPY", 0.5)
        self.set_holdings("QQQ", 0.3)

        self.debug("Portfolio rebalanced: SPY 50%, QQQ 30%, Cash 20%")
```

**预期结果**：
- 假设总资产 $100,000
  - 买入 $50,000 的 SPY
  - 买入 $30,000 的 QQQ
  - 保留 $20,000 现金

**常见坑**：
- ❌ 权重加起来超过 100% —— 会导致借钱（融资）
- ❌ 频繁调整 —— 交易成本过高
- ✅ 结合调度函数，定期重新平衡

---

### liquidate() / liquidate(symbol) - 平仓

**什么**：卖出所有持仓（或特定 symbol 的持仓）。

**什么时用**：
- 清空所有头寸
- 急速撤出某个头寸
- 算法结束时清仓

**代码例**：
```python
def on_data(self, data):
    # 平仓所有持仓
    self.liquidate()

    # 或平仓特定持仓
    self.liquidate("SPY")

def on_end_of_algorithm(self):
    # 最后一刻清空所有头寸
    self.liquidate()
    self.debug(f"Algorithm ended. Final Cash: ${self.portfolio.cash:.2f}")
```

**预期结果**：
- 所有持仓都被市价卖出

**常见坑**：
- ❌ 在回测结束前忘记平仓 —— 持仓会被强制平仓，可能虚化
- ✅ 在 `on_end_of_algorithm()` 中总是平仓

---

## 5. 技术指标 API

技术指标是交易信号的基础。QuantConnect 提供了大量内置指标，也支持自定义指标。

### 内置指标概览

```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)

    # 简单移动平均（SMA）
    self.sma_20 = self.sma("SPY", 20)

    # 指数移动平均（EMA）
    self.ema_12 = self.ema("SPY", 12)

    # 相对强弱指数（RSI）
    self.rsi = self.rsi("SPY", 14)

    # MACD（快速移动平均线差离值）
    self.macd = self.macd("SPY", 12, 26, 9)

    # 布林带（Bollinger Bands）
    self.bb = self.bb("SPY", 20, 2)
```

---

### SMA() - 简单移动平均

**什么**：某个周期内的平均价格。最基础的平滑指标。

**什么时用**：
- 判断趋势方向（价格 > SMA = 上升趋势）
- 作为支撑和阻力
- 均线交叉策略

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)
    self.sma_20 = self.sma("SPY", 20, Resolution.DAILY)
    self.sma_50 = self.sma("SPY", 50, Resolution.DAILY)
    self.set_warm_up(50)  # 预热 50 天数据

def on_data(self, data):
    # 检查指标是否准备好
    if not self.sma_20.is_ready or not self.sma_50.is_ready:
        return

    current_price = data["SPY"].close
    sma_20_value = self.sma_20.current.value
    sma_50_value = self.sma_50.current.value

    # 黄金交叉：快线上穿慢线，买入信号
    if self.sma_20.previous.value <= self.sma_50.previous.value and \
       sma_20_value > sma_50_value:
        self.market_order("SPY", 100)
        self.debug(f"Golden Cross: SMA20 {sma_20_value:.2f} > SMA50 {sma_50_value:.2f}")
```

**预期结果**：
```
Golden Cross: SMA20 450.23 > SMA50 449.50
```

**常见坑**：
- ❌ 不检查 `is_ready` —— 指标数据不足时返回 NaN
- ❌ 使用 `current.value` 而不是 `previous.value` 判断交叉 —— 会错过交叉点
- ✅ 始终预热足够的历史数据

---

### EMA() - 指数移动平均

**什么**：对最近数据赋予更高权重的移动平均。反应比 SMA 灵敏。

**什么时用**：
- 快速反应趋势变化
- 短期交易信号
- 与 SMA 结合构建混合策略

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.MINUTE)
    self.ema_12 = self.ema("SPY", 12, Resolution.MINUTE)
    self.ema_26 = self.ema("SPY", 26, Resolution.MINUTE)
    self.set_warm_up(26)

def on_data(self, data):
    if not self.ema_12.is_ready:
        return

    ema_12_val = self.ema_12.current.value
    ema_26_val = self.ema_26.current.value

    if ema_12_val > ema_26_val:
        # 短期上升趋势
        if not self.portfolio["SPY"].invested:
            self.market_order("SPY", 100)
```

---

### RSI() - 相对强弱指数

**什么**：衡量价格上升和下降的速度。值在 0-100 之间。

**什么时用**：
- 识别超买（RSI > 70）和超卖（RSI < 30）
- 反转策略（极端值时反向操作）
- 确认趋势强度

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)
    self.rsi = self.rsi("SPY", 14, Resolution.DAILY)
    self.set_warm_up(14)

def on_data(self, data):
    if not self.rsi.is_ready:
        return

    rsi_value = self.rsi.current.value

    # 超卖买入
    if rsi_value < 30:
        self.market_order("SPY", 100)
        self.debug(f"RSI Oversold: {rsi_value:.2f}")

    # 超买卖出
    elif rsi_value > 70:
        self.market_order("SPY", -100)
        self.debug(f"RSI Overbought: {rsi_value:.2f}")
```

**预期结果**：
```
RSI Oversold: 28.45
RSI Overbought: 72.10
```

**常见坑**：
- ❌ 在趋势市场中单独使用 RSI —— 超买/超卖可能持续很久
- ✅ 结合其他指标确认信号

---

### MACD() - 移动平均线差离值

**什么**：两条不同周期 EMA 的差值及其信号线。用于识别趋势和反转。

**什么时用**：
- 识别趋势开始和结束
- 指标柱状图为正时强势，为负时弱势
- MACD 线和信号线的交叉

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)
    # MACD(12, 26, 9)：12 日快线，26 日慢线，9 日信号线
    self.macd = self.macd("SPY", 12, 26, 9, Resolution.DAILY)
    self.set_warm_up(26)

def on_data(self, data):
    if not self.macd.is_ready:
        return

    macd_line = self.macd.macd.current.value      # MACD 线
    signal_line = self.macd.signal.current.value  # 信号线
    histogram = macd_line - signal_line            # 柱状图

    # MACD 线上穿信号线
    if self.macd.macd.previous.value <= self.macd.signal.previous.value and \
       macd_line > signal_line:
        self.market_order("SPY", 100)
        self.debug(f"MACD Bullish Cross: MACD {macd_line:.2f} > Signal {signal_line:.2f}")
```

**预期结果**：
```
MACD Bullish Cross: MACD 1.2345 > Signal 1.1234
```

---

### BB() - 布林带

**什么**：以移动平均线为中心的上下波动带。用于识别异常波动和突破。

**什么时用**：
- 价格接近上轨时，有可能反弹
- 价格突破上轨时，可能开始上升趋势
- 识别波动率变化

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)
    # BB(20, 2)：20 日移动平均，2 个标准差
    self.bb = self.bb("SPY", 20, 2, Resolution.DAILY)
    self.set_warm_up(20)

def on_data(self, data):
    if not self.bb.is_ready:
        return

    current_price = data["SPY"].close
    bb_upper = self.bb.band_high.current.value
    bb_middle = self.bb.middle_band.current.value
    bb_lower = self.bb.band_low.current.value

    # 价格突破下轨，反弹机会
    if current_price < bb_lower:
        self.market_order("SPY", 100)
        self.debug(f"Price {current_price:.2f} below BB Lower {bb_lower:.2f}")

    # 价格突破上轨，利润获利
    elif current_price > bb_upper:
        self.market_order("SPY", -100)
        self.debug(f"Price {current_price:.2f} above BB Upper {bb_upper:.2f}")
```

**预期结果**：
```
Price 449.50 below BB Lower 449.75
```

---

### 指标的通用属性

```python
def on_data(self, data):
    # 当前值（最新的 Bar）
    current_value = self.sma.current.value

    # 上一个 Bar 的值（用于判断交叉）
    previous_value = self.sma.previous.value

    # 指标是否已准备好（有足够的数据）
    is_ready = self.sma.is_ready
```

---

## 6. 调度 API

不是所有操作都应该在每个数据点都执行。调度 API 让你在特定时间或条件下运行代码。

### self.schedule.on() - 基础调度

**什么**：在特定日期和时间运行一个函数。

**什么时用**：
- 定期重新平衡投资组合（每周一）
- 日间收盘前平仓
- 定期风险检查
- 每月统计

**代码例**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.MINUTE)

    # 每天上午 10:30 运行 daily_rebalance 函数
    self.schedule.on(self.date_rules.every_day(),
                      self.time_rules.at(10, 30),
                      self.daily_rebalance)

    # 每个交易周一下午 3:00 运行
    self.schedule.on(self.date_rules.every(DayOfWeek.MONDAY),
                      self.time_rules.at(15, 0),
                      self.weekly_rebalance)

    # 每月第一天开盘后 1 小时运行
    self.schedule.on(self.date_rules.month_start(),
                      self.time_rules.after_market_open("SPY", 60),
                      self.monthly_rebalance)

    # 每天收盘前 10 分钟运行
    self.schedule.on(self.date_rules.every_day(),
                      self.time_rules.before_market_close("SPY", 10),
                      self.close_all_positions)

def daily_rebalance(self):
    self.debug(f"Daily rebalance at {self.time}")
    # 重新平衡逻辑

def weekly_rebalance(self):
    self.debug(f"Weekly rebalance at {self.time}")

def monthly_rebalance(self):
    self.debug(f"Monthly rebalance at {self.time}")

def close_all_positions(self):
    self.debug(f"Closing positions at {self.time}")
    self.liquidate()
```

**预期结果**：
```
Daily rebalance at 2022-01-03 10:30:00
Weekly rebalance at 2022-01-03 15:00:00  # 周一
Monthly rebalance at 2022-02-01 10:30:00  # 月初
Closing positions at 2022-01-03 15:50:00  # 收盘前 10 分钟
```

---

## 7. 投资组合与 Securities API

管理投资组合是风险控制的关键。这一节讲如何查询和管理你的头寸。

### self.portfolio - 投资组合总览

```python
def on_data(self, data):
    portfolio = self.portfolio

    # 总资产值
    total_value = portfolio.total_portfolio_value

    # 现金
    cash = portfolio.cash

    # 总持仓市值
    holdings_value = portfolio.total_holdings_value

    # 是否有任何头寸
    invested = portfolio.invested

    self.debug(f"Portfolio: Total=${total_value:.2f}, Cash=${cash:.2f}, "
               f"Holdings=${holdings_value:.2f}, Invested={invested}")
```

---

### self.portfolio[symbol] - 单个持仓

```python
def on_data(self, data):
    spy_holdings = self.portfolio["SPY"]

    # 持有股数
    quantity = spy_holdings.quantity

    # 平均成本
    avg_cost = spy_holdings.average_fill_price

    # 当前市值
    market_value = spy_holdings.absolute_holdings_value

    # 未实现利润
    unrealized_profit = spy_holdings.unrealized_profit

    # 未实现利润率
    profit_percent = spy_holdings.unrealized_profit_percent

    self.debug(f"SPY: Qty={quantity}, Cost=${avg_cost:.2f}, "
               f"Value=${market_value:.2f}, Profit={profit_percent:.2%}")
```

---

## 8. 日志与调试 API

日志帮助你理解策略在做什么，以及为什么。这对调试至关重要。

### self.debug() - 调试日志

**什么**：输出调试信息到控制台（实时），通常用于交易事件和指标值。

**什么时用**：
- 记录订单执行
- 打印指标值
- 跟踪信号
- 性能测试

**代码例**：
```python
def on_order_event(self, order_event):
    if order_event.status == OrderStatus.FILLED:
        self.debug(f"Order filled: {order_event.order_id} at ${order_event.fill_price:.2f}")

def on_data(self, data):
    if self.time.hour == 10:
        sma_value = self.sma.current.value
        self.debug(f"SMA-20 at {self.time}: {sma_value:.2f}")
```

**预期结果**（控制台）：
```
Order filled: 123 at $451.45
SMA-20 at 2022-01-03 10:00:00: 450.78
```

---

### self.log() - 持久化日志

**什么**：输出到持久化日志（不会实时显示，但会保存在回测报告中）。

**什么时用**：
- 重要事件（大额交易、风险警告）
- 策略更改
- 统计信息（每日、每周总结）

**代码例**：
```python
def on_end_of_day(self, symbol):
    daily_return = self.calculate_daily_return()
    self.log(f"Daily return for {symbol}: {daily_return:.2%}")

def on_end_of_algorithm(self):
    total_trades = len(self.transactions)
    self.log(f"Total trades executed: {total_trades}")
    self.log(f"Final Portfolio Value: ${self.portfolio.total_portfolio_value:.2f}")
```

---

### self.plot() - 绘制自定义图表

**什么**：绘制自定义指标到回测结果的图表中。

**什么时用**：
- 可视化自定义指标
- 显示持仓大小的变化
- 标记交易信号

**代码例**：
```python
def on_data(self, data):
    current_price = data["SPY"].close
    sma_value = self.sma.current.value

    # 绘制图表
    self.plot("Chart1", "SPY Price", current_price)
    self.plot("Chart1", "SMA-20", sma_value)

    # 绘制持仓
    self.plot("Portfolio", "SPY Holdings", self.portfolio["SPY"].quantity)

    # 绘制现金
    self.plot("Portfolio", "Cash", self.portfolio.cash)
```

**预期结果**：
回测结果中会显示包含这些系列的图表。

---

## 9. 风控 API

风险管理防止一次坏交易摧毁整个策略。QuantConnect 提供了多种风控工具。

### set_risk_management() - 配置风控规则

**什么**：设置自动执行的风控规则，触发时会强制平仓。

**什么时用**：
- 防止单次头寸亏损过大
- 防止整体账户亏损过多
- 保护最大连续亏损

**代码例**：
```python
from QuantConnect.Risk import MaximumDrawdownPercent, MaximumDrawdownPercentPerSecurity

def initialize(self):
    self.add_equity("SPY", Resolution.MINUTE)

    # 如果整体账户亏损超过 5%，强制平仓所有头寸
    self.set_risk_management(MaximumDrawdownPercent(0.05))

    # 或者，单个头寸亏损超过 3%，强制平仓该头寸
    self.set_risk_management(MaximumDrawdownPercentPerSecurity(0.03))
```

**预期结果**：
- 如果策略下跌 5% 以上，所有持仓会被强制平仓，避免进一步亏损

**常见坑**：
- ❌ 风控参数设置过宽松 —— 亏损过大才平仓
- ❌ 风控参数设置过严格 —— 频繁被止损
- ✅ 通常 3-5% 的回撤是合理范围

---

### 手动风险检查

除了自动风控，你也可以手动检查风险指标：

```python
def on_end_of_day(self, symbol):
    # 计算最大回撤
    current_value = self.portfolio.total_portfolio_value
    peak_value = self._calculate_peak_value()  # 自定义函数
    drawdown = (peak_value - current_value) / peak_value

    if drawdown > 0.10:  # 回撤超过 10%
        self.error(f"RISK ALERT: Drawdown {drawdown:.2%} exceeds limit")
        # 可选：强制平仓
        # self.liquidate()

def _calculate_peak_value(self):
    """计算历史最高资产值"""
    # 实现方式：维护一个自更新的 peak 值
    if not hasattr(self, 'peak_value'):
        self.peak_value = self.portfolio.total_portfolio_value

    self.peak_value = max(self.peak_value, self.portfolio.total_portfolio_value)
    return self.peak_value
```

---

## 10. 综合实战示例

现在让我们结合上述所有 API，构建一个完整的、生产级别的交易策略。

```python
"""
综合示例：动态股票池的技术指标多头策略

策略逻辑：
1. 每周一选择成交量最高的 10 只股票
2. 对每只股票计算 RSI 和 EMA 信号
3. 当 RSI < 50 且价格 > EMA-12 时买入
4. 当 RSI > 70 或价格 < EMA-12 时卖出
5. 每只股票最多配置 5% 的资金
6. 设置 3% 的单个头寸止损
7. 整体回撤超过 5% 时平仓
"""

from QuantConnect import QCAlgorithm, Resolution, OrderStatus, DayOfWeek
from QuantConnect.Risk import MaximumDrawdownPercent, MaximumDrawdownPercentPerSecurity

class ComprehensiveStrategy(QCAlgorithm):

    def initialize(self):
        """初始化：设置时间、资金、订阅、指标、调度"""

        # 时间和资金设置
        self.set_start_date(2022, 1, 1)
        self.set_end_date(2023, 12, 31)
        self.set_cash(100000)

        # 风险控制
        self.set_risk_management(MaximumDrawdownPercent(0.05))
        self.set_risk_management(MaximumDrawdownPercentPerSecurity(0.03))

        # 动态宇宙：每天选择成交量最高的 10 只股票
        self.add_universe(self.select_high_volume_stocks)

        # 调度函数
        # 每周一上午 10:00 重新平衡
        self.schedule.on(self.date_rules.every(DayOfWeek.MONDAY),
                          self.time_rules.at(10, 0),
                          self.rebalance_universe)

        # 每天下午 3:55 平仓（市场关闭前 5 分钟）
        self.schedule.on(self.date_rules.every_day(),
                          self.time_rules.at(15, 55),
                          self.close_all_positions)

        # 状态变量
        self.indicators = {}  # 保存每只股票的指标
        self.max_position_ratio = 0.05  # 单个头寸最多占 5% 资产
        self.peak_portfolio_value = 100000  # 初始值

    def select_high_volume_stocks(self, coarse):
        """宇宙选择函数：选择成交量最高的 10 只股票"""

        # 过滤：价格 > $5，有成交量
        filtered = [x for x in coarse if x.price > 5 and x.dollar_volume > 0]

        # 按美元成交量排序，取前 10 只
        sorted_by_volume = sorted(filtered,
                                  key=lambda x: x.dollar_volume,
                                  reverse=True)[:10]

        return [x.symbol for x in sorted_by_volume]

    def rebalance_universe(self):
        """每周重新平衡：清空旧持仓，初始化新的指标"""

        # 清空旧的指标字典
        self.indicators.clear()

        # 为宇宙中的每只新股票创建指标
        for symbol in list(self.portfolio.keys()):
            if self.portfolio[symbol].quantity > 0:
                # 平仓旧持仓
                self.market_order(symbol, -self.portfolio[symbol].quantity)
                self.log(f"[REBALANCE] Closed position in {symbol}")

        # 初始化新股票的指标
        for symbol in list(self.securities.keys()):
            try:
                self.indicators[symbol] = {
                    'rsi': self.rsi(symbol, 14, Resolution.MINUTE),
                    'ema12': self.ema(symbol, 12, Resolution.MINUTE),
                    'ema26': self.ema(symbol, 26, Resolution.MINUTE),
                }
                self.log(f"[REBALANCE] Added indicators for {symbol}")
            except:
                pass

    def on_data(self, data):
        """主交易逻辑"""

        # 更新最高资产值（用于计算回撤）
        current_value = self.portfolio.total_portfolio_value
        self.peak_portfolio_value = max(self.peak_portfolio_value, current_value)

        # 对宇宙中的每只股票进行交易逻辑
        for symbol in list(self.indicators.keys()):
            if not data.contains_key(symbol):
                continue

            indicators = self.indicators[symbol]
            current_price = data[symbol].close

            # 检查指标是否准备好
            if not indicators['rsi'].is_ready or not indicators['ema12'].is_ready:
                continue

            rsi_value = indicators['rsi'].current.value
            ema12_value = indicators['ema12'].current.value
            ema26_value = indicators['ema26'].current.value

            holdings = self.portfolio[symbol]

            # ===== 买入信号 =====
            if holdings.quantity == 0:  # 没有持仓时才考虑买入
                # 买入条件：RSI < 50（相对较弱，但有反弹机会）且价格 > EMA-12（短期上升趋势）
                if rsi_value < 50 and current_price > ema12_value:
                    # 计算买入数量：单个头寸最多占总资产的 5%
                    target_value = self.portfolio.total_portfolio_value * self.max_position_ratio
                    shares_to_buy = int(target_value / current_price)

                    if shares_to_buy > 0:
                        self.market_order(symbol, shares_to_buy)
                        self.debug(f"[BUY] {symbol}: Price={current_price:.2f}, "
                                   f"RSI={rsi_value:.2f}, EMA12={ema12_value:.2f}, "
                                   f"Qty={shares_to_buy}")
                        self.plot("Signals", f"{symbol} Buy", current_price)

            # ===== 卖出信号 =====
            elif holdings.quantity > 0:  # 有持仓时考虑卖出
                # 卖出条件 1：RSI > 70（超买）
                if rsi_value > 70:
                    self.market_order(symbol, -holdings.quantity)
                    profit = holdings.unrealized_profit_percent
                    self.debug(f"[SELL-RSI] {symbol}: RSI={rsi_value:.2f}, "
                               f"Profit={profit:.2%}")
                    self.plot("Signals", f"{symbol} Sell", current_price)

                # 卖出条件 2：价格 < EMA-12（短期趋势反转）
                elif current_price < ema12_value:
                    self.market_order(symbol, -holdings.quantity)
                    profit = holdings.unrealized_profit_percent
                    self.debug(f"[SELL-EMA] {symbol}: Price={current_price:.2f} < "
                               f"EMA12={ema12_value:.2f}, Profit={profit:.2%}")
                    self.plot("Signals", f"{symbol} Sell", current_price)

        # ===== 绘制投资组合信息 =====
        self.plot("Portfolio", "Total Value", self.portfolio.total_portfolio_value)
        self.plot("Portfolio", "Cash", self.portfolio.cash)

    def on_order_event(self, order_event):
        """处理订单事件"""

        order = self.transactions.get_order_by_id(order_event.order_id)

        if order.status == OrderStatus.FILLED:
            self.debug(f"[ORDER] {order.symbol}: "
                       f"Direction={'BUY' if order.quantity > 0 else 'SELL'} "
                       f"Qty={abs(order.quantity)}, "
                       f"Fill Price=${order_event.fill_price:.2f}")

    def on_end_of_day(self, symbol):
        """每日结束处理"""

        holdings = self.portfolio[symbol]
        if holdings.quantity > 0:
            # 记录持仓的每日利润
            daily_pnl = holdings.unrealized_profit
            daily_return = holdings.unrealized_profit_percent

            self.plot("Holdings", f"{symbol} Profit", daily_pnl)

    def close_all_positions(self):
        """市场关闭前平仓所有头寸"""

        for symbol in list(self.portfolio.keys()):
            holdings = self.portfolio[symbol]
            if holdings.quantity != 0:
                self.market_order(symbol, -holdings.quantity)
                self.debug(f"[MARKET CLOSE] Closed {symbol}: {holdings.quantity} shares")

    def on_end_of_algorithm(self):
        """算法结束时的统计"""

        # 计算统计指标
        final_value = self.portfolio.total_portfolio_value
        total_return = (final_value - 100000) / 100000

        # 计算最大回撤
        max_drawdown = (self.peak_portfolio_value - final_value) / self.peak_portfolio_value

        # 计算总交易数
        total_trades = len([x for x in self.transactions.orders.values()
                           if x.status == OrderStatus.FILLED])

        # 打印最终统计
        self.log("=" * 50)
        self.log("ALGORITHM COMPLETE - FINAL STATISTICS")
        self.log("=" * 50)
        self.log(f"Initial Capital: $100,000.00")
        self.log(f"Final Value: ${final_value:.2f}")
        self.log(f"Total Return: {total_return:.2%}")
        self.log(f"Max Drawdown: {max_drawdown:.2%}")
        self.log(f"Total Trades: {total_trades}")
        self.log(f"Final Cash: ${self.portfolio.cash:.2f}")
        self.log("=" * 50)

        # 绘制最终结果
        self.plot("Final", "Portfolio Value", final_value)
```

---

## 11. 常见错误与调试

### 错误 1：指标未准备好（NaN 或 None）

**症状**：
```
ERROR: Cannot convert NaN to float
AttributeError: 'NoneType' object has no attribute 'value'
```

**原因**：
- 指标数据不足（没有足够的历史数据）
- 忘记设置预热

**解决**：
```python
def initialize(self):
    self.add_equity("SPY", Resolution.DAILY)
    self.sma = self.sma("SPY", 50)
    self.set_warm_up(50)  # 添加这一行

def on_data(self, data):
    if not self.sma.is_ready:  # 始终检查
        return

    value = self.sma.current.value
```

---

### 错误 2：数据不存在（KeyError）

**症状**：
```
KeyError: 'SYMBOL'
```

**原因**：
- Symbol 不存在或拼写错误
- 新上市股票没有历史数据
- 交易时段外没有数据

**解决**：
```python
def on_data(self, data):
    if data.contains_key("SPY"):  # 始终检查
        price = data["SPY"].close
```

---

### 错误 3：过度交易

**症状**：
- 每个数据点都在下单
- 交易费用很高
- 回测结果不现实

**解决**：
```python
# ❌ 错误：每个数据点都买卖
def on_data(self, data):
    self.market_order("SPY", 100)
    self.market_order("SPY", -100)

# ✅ 正确：只有在信号改变时才交易
def on_data(self, data):
    if self.should_buy() and not self.portfolio["SPY"].invested:
        self.market_order("SPY", 100)
```

---

### 错误 4：忘记平仓

**症状**：
- 回测完成但仍持有头寸
- 最终结果包含持仓的浮动利润，不是实际利润

**解决**：
```python
def on_end_of_algorithm(self):
    self.liquidate()  # 必须平仓
```

---

### 错误 5：回测太慢

**症状**：
- 回测运行时间很长（> 30 分钟）
- CPU 使用率很高

**原因**：
- 使用了 Tick 级别数据
- 订阅了太多 symbol
- History() 调用过于频繁

**解决**：
```python
# ❌ Tick 级数据太详细
self.add_equity("SPY", Resolution.TICK)

# ✅ 改为分钟或日线
self.add_equity("SPY", Resolution.MINUTE)

# ❌ 订阅过多 symbol
for i in range(1000):
    self.add_equity(f"STOCK_{i}", Resolution.MINUTE)

# ✅ 使用宇宙动态订阅，限制数量
self.add_universe(lambda x: sorted(x, key=lambda y: y.dollar_volume, reverse=True)[:50])

# ❌ 在每个 on_data 调用 history()
def on_data(self, data):
    history = self.history("SPY", 100, Resolution.DAILY)  # 每分钟调用 100 次！

# ✅ 少调用 history()
def on_data(self, data):
    if self.time.minute % 60 == 0:  # 每小时才调用一次
        history = self.history("SPY", 100, Resolution.DAILY)
```

---

### 调试技巧

**1. 使用 debug() 打印变量**
```python
def on_data(self, data):
    price = data["SPY"].close
    self.debug(f"Price: {price}, Time: {self.time}")  # 实时查看
```

**2. 检查持仓**
```python
self.debug(f"SPY Holdings: {self.portfolio['SPY'].quantity}")
```

**3. 监控现金**
```python
self.debug(f"Available Cash: ${self.portfolio.cash:.2f}")
```

**4. 跟踪订单状态**
```python
def on_order_event(self, order_event):
    self.debug(f"Order {order_event.order_id}: {order_event.status}")
```

**5. 使用 plot() 可视化**
```python
self.plot("Debug", "SMA Value", self.sma.current.value)
self.plot("Debug", "Portfolio Value", self.portfolio.total_portfolio_value)
```

---

## 总结

这份手册覆盖了 QuantConnect LEAN 引擎的核心 API：

- **生命周期**：理解 initialize → on_warmup_finished → on_data → on_end_of_algorithm 的调用顺序
- **数据**：订阅（add_equity/forex/crypto/future）→ 访问（data[symbol]）→ 历史（history API）
- **交易**：market_order → limit_order → stop_market_order → set_holdings → liquidate
- **指标**：内置 SMA/EMA/RSI/MACD/BB，以及自定义指标
- **调度**：定期重新平衡，市场开盘/收盘时操作
- **风控**：set_risk_management、手动检查、头寸限制
- **日志**：debug → log → error → plot，帮助调试和可视化

**下一步**：
- 复制综合示例到 QuantConnect 平台，运行回测
- 修改参数，看结果如何变化
- 逐步构建自己的策略

---

**导航**
[上一篇: 30 分钟跑通第一个策略](./01-first-strategy.md) | 下一篇（即将推出）
