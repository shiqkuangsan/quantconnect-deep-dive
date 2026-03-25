# 从回测到实盘：上线清单

> **前置要求**：已完成 01-08 篇，有一个经过充分回测的策略。
> **目标受众**：2-3 人初创团队，即将用真实资金运行策略。
> **学习成果**：能够安全地将策略从回测过渡到实盘，建立完整的风控体系。

---

## 上线前的心理准备

### 残酷的真相

1. **回测不等于实盘**：
   - 回测假设订单立即填充（实盘常有滑点）
   - 回测假设数据无误（实盘有数据错误 / 断线）
   - 回测假设无网络延迟（实盘 API 超时很常见）
   - 回测假设订单成本恒定（实盘 Slippage 可能很大）

2. **黑天鹅事件**：
   - 市场跳空（开盘缺口、熔断）
   - 流动性枯竭（某些时段无法成交）
   - 技术故障（交易所宕机、网络中断）
   - 券商风控（你的策略被限制）

3. **心理挑战**：
   - 看着真实资金浮亏时的恐慌
   - 止损后立即反弹的懊悔
   - 长期没有收益的自我怀疑

### 建议

> **Start small, think big, grow fast.**

- **初期资金**：用你可以承受完全亏损的金额（例 $5,000-$10,000）
- **预期收益**：前 3 个月目标是"存活"，而不是"致富"
- **心态**：每笔亏损都是学费，是改进策略的信息

---

## Phase 1: Paper Trading 验证（1-2 周）

### 什么是 Paper Trading

模拟交易：用真实市场数据，但用虚拟账户（不花真钱）。
- 你的订单在真实市场里下单，但立即被撤销
- 成交价格根据真实行情计算
- 能发现回测时看不到的问题（网络、API、数据）

### 在 QuantConnect 云平台上进行纸面交易

```python
# algorithm.py
def Initialize(self):
    self.SetStartDate(2024, 1, 1)
    self.SetEndDate(2024, 12, 31)
    self.SetCash(100000)
    
    # 关键：设置纸面交易模式
    self.SetBrokerageModel(BrokerageModel.AlpacaPaperTrading)
    
    self.AddEquity("AAPL", Resolution.Daily)

def OnData(self, data):
    if not self.Portfolio.Invested:
        self.Buy("AAPL", 10)
```

在 QC 云平台：
1. 代码编辑器中粘贴上述代码
2. 点击 "Paper Trading" 按钮
3. 等待部署完成
4. 监控实时 PnL、订单、持仓

### 在本地进行纸面交易

```bash
# 使用 lean-cli
lean backtest

# 纸面交易模式
lean live --paper
```

### 监控项清单

| 指标 | 预期值 | 红旗信号 |
|------|--------|--------|
| 订单填充率 | > 95% | < 80%（可能是滑点太大） |
| 平均成交延迟 | 1-5 秒 | > 30 秒（网络问题） |
| 日均交易次数 | 回测值 ±10% | 偏离太多（可能数据问题） |
| 日均滑点 | 0.01-0.05% | > 0.1%（成本过高） |

### 常见问题与解决方案

**问题 1：订单长时间未成交**
```python
# 原因：Market Hours 外下单，或流动性不足
# 解决：
def OnData(self, data):
    # 添加时间检查
    if not self.IsMarketOpen("AAPL"):
        return
    
    if data["AAPL"].Volume < 1000000:  # 日均成交量太低
        return
    
    self.Buy("AAPL", 10)
```

**问题 2：纸面成交与预期相差很大**
```python
# 原因：流动性模式不匹配
# 解决：使用现实的成交模式

def Initialize(self):
    # 使用 VolumeLimitedImpactModel（考虑订单对市场的冲击）
    model = VolumeLimitedImpactModel(0.25)  # 单笔订单不超过 25% 日均成交量
    self.SetFillModel(model)
```

**问题 3：持仓与回测不符**
```python
# 原因：回测中有"未来看"，实盘没有
# 解决：检查数据延迟

def OnData(self, data):
    # 确保使用了足够的历史数据
    history = self.History(100)  # 最近 100 根 K 线
    
    # 延迟 1 根 K 线再交易（避免未来看）
    if len(history) < 100:
        return
    
    # ... 交易逻辑
```

### 纸面交易检查清单

- [ ] 运行至少 5 个交易日
- [ ] 每日监控 PnL 和订单成交
- [ ] 检查是否有异常的成交价格或延迟
- [ ] 验证风控参数是否生效（如最大持仓、日损失限制）
- [ ] 测试应急止损（手动设置亏损超过 $1,000，检查是否自动平仓）
- [ ] 确认监控系统是否工作（邮件告警、日志记录）

---

## Phase 2: 券商账户配置

### 选择合适的券商

对于美股初学者，推荐：

| 券商 | 优点 | 缺点 | API 支持 |
|------|------|------|---------|
| **Alpaca** | 佣金 0，最小账户 $1，API 友好 | 仅美股，滑点较大 | REST + WebSocket |
| **Interactive Brokers** | 支持全球市场，佣金低 | 账户最小 $10K，UI 复杂 | 完整 API |
| **Webull** | 佣金 0，界面友好 | API 功能有限 | 基础 API |
| **thinkorswim** | 工具丰富，数据准确 | 需要 Schwab 账户 | 有限 API |

**推荐流程**：新手 → Alpaca → 后期升级 IB（如需全球市场）

### Alpaca 账户设置（推荐用于学习）

#### 第 1 步：创建账户

1. 访问 https://app.alpaca.markets
2. 用邮箱注册账户
3. 完成身份认证（身份证 / 护照）
4. 资金到账（ACH 转账 1-2 个工作日）

#### 第 2 步：生成 API Key

1. 进入 Dashboard → Account → API Keys
2. 生成 Paper 环境 Key（纸面交易）
3. 生成 Live 环境 Key（实盘）
4. **永远不要提交 Key 到 GitHub**

```bash
# .env 文件（加入 .gitignore）
APCA_API_KEY_ID=your_key_here
APCA_API_SECRET_KEY=your_secret_here
APCA_API_BASE_URL=https://paper-api.alpaca.markets  # Paper
# APCA_API_BASE_URL=https://api.alpaca.markets  # Live
```

#### 第 3 步：在 QuantConnect 中配置

```python
# algorithm.py
def Initialize(self):
    # 设置 Alpaca 纸面账户
    self.SetBrokerageModel(BrokerageModel.AlpacaPaperTrading)
    
    # 设置资金
    self.SetCash(10000)  # 开始资金
```

或本地测试：

```bash
# 使用 lean-cli 连接 Alpaca
lean live \
  --broker alpaca \
  --api-key $APCA_API_KEY_ID \
  --api-secret $APCA_API_SECRET_KEY
```

#### 第 4 步：测试连接

```python
def Initialize(self):
    self.SetBrokerageModel(BrokerageModel.AlpacaPaperTrading)
    self.AddEquity("AAPL", Resolution.Daily)
    
    # 定期检查账户
    self.Schedule.On(
        self.DateRules.EveryDay(),
        self.TimeRules.At(15, 30),  # 市场收盘时
        self.CheckAccount
    )

def CheckAccount(self):
    # 打印账户信息到日志
    self.Log(f"Portfolio Value: {self.Portfolio.TotalPortfolioValue}")
    self.Log(f"Cash: {self.Portfolio.Cash}")
    self.Log(f"Equity: {self.Portfolio.TotalUnrealizedProfit}")
```

### 进阶配置

#### 1. 多币种账户

```python
def Initialize(self):
    # 添加港股（需要国际账户）
    self.AddEquity("0700.HK", Resolution.Daily)  # Tencent
```

#### 2. 配置杠杆

```python
def Initialize(self):
    # Alpaca 最高允许 4 倍杠杆（对日交易者）
    self.SetSecurityInitializer(
        lambda security: security.SetLeverage(2)
    )
```

#### 3. 配置数据源

```python
def Initialize(self):
    # 使用 IQFeed 实时数据源（比默认的更准确）
    self.SetDataNormalizationMode(DataNormalizationMode.Raw)
```

---

## Phase 3: 风控设置

**最重要的一步**。即使策略 100% 正确，缺少风控也会导致灾难。

### 1. 最大持仓限制

```python
def Initialize(self):
    self.max_position_size = 0.1  # 单个头寸不超过总资金的 10%
    self.max_number_of_positions = 10  # 最多 10 个头寸
    
    self.AddEquity("AAPL", Resolution.Daily)

def OnData(self, data):
    # 下单前检查
    total_value = self.Portfolio.TotalPortfolioValue
    position_value = self.Portfolio[symbol].Holdings.HoldingsValue
    
    # 如果已有头寸，不再增加
    if self.Portfolio[symbol].Invested:
        return
    
    # 如果总头寸数达到上限，不再新建
    if len(self.Portfolio.Positions) >= self.max_number_of_positions:
        return
    
    # 只购买不超过 10% 总资金的头寸
    order_value = total_value * self.max_position_size
    quantity = int(order_value / data[symbol].Close)
    
    self.Buy(symbol, quantity)
```

### 2. 日损失限制（Circuit Breaker）

```python
def Initialize(self):
    self.daily_loss_limit = 0.02  # 日亏损 2% 则停止交易
    self.daily_start_portfolio_value = None
    
    self.Schedule.On(
        self.DateRules.EveryDay(),
        self.TimeRules.At(9, 30),  # 市场开盘
        self.ResetDailyLimits
    )
    
    self.Schedule.On(
        self.DateRules.EveryDay(),
        self.TimeRules.Every(
            timedelta(minutes=1)
        ),
        self.CheckDailyLoss
    )

def ResetDailyLimits(self):
    self.daily_start_portfolio_value = self.Portfolio.TotalPortfolioValue

def CheckDailyLoss(self):
    if self.daily_start_portfolio_value is None:
        return
    
    current_loss = (
        (self.Portfolio.TotalPortfolioValue - self.daily_start_portfolio_value) 
        / self.daily_start_portfolio_value
    )
    
    if current_loss < -self.daily_loss_limit:
        self.Log("CIRCUIT BREAKER: Daily loss limit hit")
        # 平仓所有头寸
        for holding in self.Portfolio.Values:
            if holding.Invested:
                self.Liquidate(holding.Symbol)
```

### 3. 个股风险管理

```python
def OnData(self, data):
    symbol = "AAPL"
    
    if self.Portfolio[symbol].Invested:
        # 止损：亏损超过 5% 则卖出
        pnl_pct = self.Portfolio[symbol].UnrealizedProfitPercent
        
        if pnl_pct < -0.05:
            self.Log(f"Stop loss triggered for {symbol}: {pnl_pct:.2%}")
            self.Liquidate(symbol)
        
        # 止盈：获利超过 20% 则部分卖出
        elif pnl_pct > 0.20:
            self.Log(f"Take profit triggered for {symbol}: {pnl_pct:.2%}")
            self.Liquidate(symbol, tag="Take Profit")
```

### 4. 完整的风控框架

```python
class RiskManager:
    def __init__(self, algorithm):
        self.algorithm = algorithm
        self.config = {
            "max_position_size": 0.1,  # 单个头寸最多 10% 资金
            "max_positions": 10,  # 最多 10 个头寸
            "daily_loss_limit": 0.02,  # 日亏损 2% 停止
            "max_leverage": 1.5,  # 最大杠杆 1.5x
            "stop_loss": 0.05,  # 单个头寸亏损 5% 止损
            "take_profit": 0.20,  # 单个头寸获利 20% 止盈
        }
    
    def check_order_validity(self, symbol, quantity, price):
        """在下单前检查是否违反风控"""
        
        # 1. 检查头寸数量
        if len(self.algorithm.Portfolio.Positions) >= self.config["max_positions"]:
            return False, "Max positions reached"
        
        # 2. 检查头寸大小
        order_value = quantity * price
        total_value = self.algorithm.Portfolio.TotalPortfolioValue
        
        if order_value > total_value * self.config["max_position_size"]:
            return False, f"Position size exceeds {self.config['max_position_size']:.0%} limit"
        
        # 3. 检查杠杆
        if order_value > total_value * self.config["max_leverage"]:
            return False, f"Leverage exceeds {self.config['max_leverage']}x"
        
        return True, "OK"
    
    def check_daily_loss(self):
        """检查日亏损是否超过限制"""
        if not hasattr(self, "daily_start_value"):
            self.daily_start_value = self.algorithm.Portfolio.TotalPortfolioValue
            return True
        
        current_loss = (
            (self.algorithm.Portfolio.TotalPortfolioValue - self.daily_start_value)
            / self.daily_start_value
        )
        
        if current_loss < -self.config["daily_loss_limit"]:
            return False
        
        return True
    
    def check_position_pnl(self, holding):
        """检查单个头寸是否需要止损 / 止盈"""
        pnl_pct = holding.UnrealizedProfitPercent
        
        if pnl_pct < -self.config["stop_loss"]:
            return "STOP_LOSS"
        
        if pnl_pct > self.config["take_profit"]:
            return "TAKE_PROFIT"
        
        return None
```

---

## Phase 4: 监控与告警

### 监控指标

```python
class PerformanceMonitor:
    def __init__(self, algorithm):
        self.algorithm = algorithm
        self.hourly_stats = []
        self.daily_stats = []
    
    def record_hourly_stats(self):
        """每小时记录一次指标"""
        stats = {
            "time": self.algorithm.Time,
            "portfolio_value": self.algorithm.Portfolio.TotalPortfolioValue,
            "cash": self.algorithm.Portfolio.Cash,
            "num_positions": len([h for h in self.algorithm.Portfolio.Values if h.Invested]),
            "max_dd": self._calculate_max_drawdown(),
            "daily_pnl": self.algorithm.Portfolio.DailyProfit,
        }
        self.hourly_stats.append(stats)
    
    def record_daily_stats(self):
        """每日记录一次统计"""
        stats = {
            "date": self.algorithm.Time.date(),
            "daily_return": self.algorithm.Portfolio.DailyProfit / self.algorithm.Portfolio.TotalPortfolioValue,
            "num_trades": self._count_daily_trades(),
            "win_rate": self._calculate_win_rate(),
        }
        self.daily_stats.append(stats)
    
    def _calculate_max_drawdown(self):
        """计算当前回撤"""
        # 实现见第 06 篇
        pass
    
    def send_alert(self, level, message):
        """发送告警"""
        if level == "CRITICAL":
            self._send_email(f"[CRITICAL] {message}")
            self._send_sms(f"[CRITICAL] {message}")
        elif level == "WARNING":
            self._send_email(f"[WARNING] {message}")
        elif level == "INFO":
            self.algorithm.Log(f"[INFO] {message}")
```

### 集成通知系统

#### 1. 邮件告警

```python
def Initialize(self):
    # 使用 QuantConnect 内置的邮件功能
    self.notifications = []
    
    self.Schedule.On(
        self.DateRules.EveryDay(),
        self.TimeRules.At(16, 0),  # 每天 4 PM
        self.SendDailyReport
    )

def SendDailyReport(self):
    message = f"""
    === 每日报告 {self.Time.date()} ===
    
    总资产: ${self.Portfolio.TotalPortfolioValue:.2f}
    现金: ${self.Portfolio.Cash:.2f}
    日收益: ${self.Portfolio.DailyProfit:.2f}
    日回报: {self.Portfolio.DailyProfit / self.Portfolio.TotalPortfolioValue:.2%}
    
    持仓数: {len([h for h in self.Portfolio.Values if h.Invested])}
    最大回撤: {self._get_max_drawdown():.2%}
    
    主要交易:
    {self._get_today_trades()}
    """
    
    self.Notify.Email("your_email@example.com", "Daily Report", message)
```

#### 2. SMS 告警（critical only）

```python
def CheckCriticalEvents(self):
    portfolio_value = self.Portfolio.TotalPortfolioValue
    
    if portfolio_value < 5000:  # 资金跌至 $5K
        message = f"ALERT: Portfolio value dropped to ${portfolio_value:.2f}"
        self._send_sms(message)
        self.Liquidate()  # 自动平仓
```

#### 3. 仪表板

使用 QuantConnect 内置仪表板或第三方（Grafana）：

```
┌─────────────────────────────────┐
│   实盘监控仪表板                │
├─────────────────────────────────┤
│ 总资产: $50,234                 │
│ 日收益: $234 (+0.47%)           │
│ 持仓: 5                         │
│                                 │
│ 主要持仓:                       │
│ - AAPL: +3.2% ($5,000)         │
│ - MSFT: -1.5% ($4,800)         │
│ - GOOGL: +0.8% ($4,200)        │
│                                 │
│ 风控:                           │
│ - 日亏损: $-245 (-0.48%)       │
│ - 最大头寸: AAPL (10%)          │
│ - 杠杆: 1.0x                   │
│                                 │
│ 最后更新: 2024-03-26 15:45     │
└─────────────────────────────────┘
```

---

## Phase 5: 首次上线

### 前夜检查清单

上线前 24 小时，逐项验证：

- [ ] API Key 已保存在环境变量（未硬编码）
- [ ] 纸面交易运行 5+ 天，表现符合预期
- [ ] 风控参数已设置（日损失限、头寸限制、止损）
- [ ] 监控系统已测试（邮件、日志能否正常工作）
- [ ] 应急联系方式已准备（如需紧急停止）
- [ ] 初期资金 <= $10,000（可承受全部损失）
- [ ] 代码最后一次 Review（无明显 Bug）

### 第 1 小时：激进监控

```python
def Initialize(self):
    # 第一小时每 30 秒检查一次
    self.Schedule.On(
        self.DateRules.EveryDay(),
        self.TimeRules.Every(timedelta(seconds=30)),
        self.HealthCheck
    )

def HealthCheck(self):
    if self.Time.hour == 9:  # 市场开盘第一小时
        self.Log(f"Health Check: {self.Time}")
        self.Log(f"  Portfolio Value: ${self.Portfolio.TotalPortfolioValue:.2f}")
        self.Log(f"  Cash: ${self.Portfolio.Cash:.2f}")
        self.Log(f"  Open Orders: {len(self.Transactions.GetOrders(lambda o: o.Status == OrderStatus.Submitted))}")
        
        # 如果发现异常，立即停止
        if self.Portfolio.TotalPortfolioValue < 9000:  # 亏损 > 10%
            self.Log("EMERGENCY STOP: Portfolio value dropping too fast")
            self.Liquidate()
```

### 第 1 天：观察与记录

**早上 9:30**（开盘）
- 系统是否正常启动？
- 首笔订单是否正常成交？
- 是否有异常的成交价格？

**上午 10:00-11:30**
- 监控前 30 分钟的成交情况
- 查看订单执行是否有延迟
- 检查 API 是否稳定

**中午 12:00**
- 收盘前总结早盘情况
- 如果没有严重问题，继续持仓到收盘

**下午 3:00（收盘前 30 分钟）**
- 考虑是否平仓（通常建议收盘前平仓，避免隔夜风险）
- 记录全天交易情况

**下午 4:00（收盘后）**
- 生成日报告
- 检查账户 cash balance 是否正确
- 分析任何异常交易

### 第 1 周：验证与改进

逐日监控以下指标：

| 指标 | 目标 | 实际 | 是否调整 |
|------|------|------|---------|
| 日均成交延迟 | < 5s | - | - |
| 订单成功率 | > 95% | - | - |
| 平均滑点 | < 0.01% | - | - |
| 日回报 | ±1% | - | - |

记录每个交易日的观察：

```markdown
# 交易日志

## 2024-03-26 (Day 1)
- 开盘正常，API 响应时间 2-3 秒
- 早盘 10:00 执行了 5 笔买单，全部成交
- 无异常滑点，成交价格与 best bid 相近
- 下午 15:00 人工平仓，锁定 $234 利润
- **结论**：系统运行正常，可继续

## 2024-03-27 (Day 2)
- ...

## 周总结
- 周回报：+1.2% ($600)
- 最大单日亏损：-0.5%
- 系统稳定性：A
- **下周计划**：增加仓位规模，测试风控
```

---

## 运维 SOP（Standard Operating Procedure）

### 每日检查（Daily Routine）

**早上 8:00**（市场开盘前）

```python
def PreMarketCheck(self):
    # 1. 检查系统健康
    self.Log("=== 市场开盘前检查 ===")
    
    # 2. 检查 API 连接
    try:
        portfolio = self.GetPortfolio()
        self.Log(f"✓ API 连接正常")
    except:
        self.Log("✗ API 连接失败，立即告警")
        self._send_alert("CRITICAL", "API Connection Lost")
        return False
    
    # 3. 检查是否有未平仓头寸
    open_positions = len([h for h in portfolio.Values if h.Invested])
    self.Log(f"Open Positions: {open_positions}")
    
    # 4. 检查现金
    cash = portfolio.Cash
    self.Log(f"Available Cash: ${cash:.2f}")
    
    if cash < 1000:
        self.Log("⚠ Cash < $1K, consider stopping")
    
    return True
```

**下午 3:45**（市场收盘前）

```python
def PreCloseCheck(self):
    # 1. 决策：是否平仓隔夜持仓
    for holding in self.Portfolio.Values:
        if holding.Invested:
            pnl = holding.UnrealizedProfit
            pnl_pct = holding.UnrealizedProfitPercent
            
            self.Log(f"{holding.Symbol}: {pnl_pct:+.2%} (${pnl:+.2f})")
            
            # 建议：可以设置自动平仓时间
            if self.Time.hour == 15 and self.Time.minute == 45:
                self.Liquidate(holding.Symbol)
```

**下午 4:00**（市场收盘后）

```python
def PostCloseReport(self):
    self.Log("\n=== 日交易总结 ===")
    
    daily_pnl = self.Portfolio.DailyProfit
    daily_return = daily_pnl / self.Portfolio.TotalPortfolioValue
    
    self.Log(f"日收益: ${daily_pnl:.2f}")
    self.Log(f"日回报: {daily_return:.2%}")
    self.Log(f"总资产: ${self.Portfolio.TotalPortfolioValue:.2f}")
    
    # 发送日报
    self._send_daily_email()
```

### 每周审查（Weekly Review）

每周五下午 5:00：

```python
def WeeklyReview(self):
    """
    审查本周表现，为下周调整做准备
    """
    
    # 1. 周收益统计
    week_pnl = sum([stat["daily_pnl"] for stat in self.daily_stats[-5:]])
    week_return = week_pnl / self.initial_portfolio_value
    
    # 2. 每日回报
    daily_returns = [stat["daily_return"] for stat in self.daily_stats[-5:]]
    avg_daily_return = sum(daily_returns) / len(daily_returns)
    
    # 3. 最大回撤
    max_dd = min(daily_returns)
    
    # 4. 夏普比率
    sharpe = self._calculate_sharpe_ratio(daily_returns)
    
    self.Log(f"""
    === 周报告 ===
    周收益: ${week_pnl:.2f} ({week_return:.2%})
    平均日回报: {avg_daily_return:.2%}
    最大日亏损: {max_dd:.2%}
    夏普比率: {sharpe:.2f}
    
    决策:
    - 是否调整策略参数?
    - 是否增加/减少仓位?
    - 是否有重大 Bug 需要修复?
    """)
    
    # 发送周报给管理层
    self._send_weekly_email()
```

### 何时停止策略

立即平仓 + 停止的情况：

```python
def EmergencyStop(self, reason):
    self.Log(f"🛑 EMERGENCY STOP: {reason}")
    self.Liquidate()  # 平仓所有头寸
    
    # 发送紧急通知
    self._send_sms(f"Emergency Stop: {reason}")
    self._send_email(f"Emergency Stop: {reason}")
    
    # 停止接收新订单
    self.StopStrategy()
```

触发条件：

1. **技术故障**：API 连续失败 > 5 分钟
2. **异常行情**：单个头寸日亏损 > 50%
3. **风控触发**：日亏损 > 2%
4. **数据异常**：股价跳动 > 20%（可能是数据错误）

### 何时更新策略参数

不应立即修改，而是：

1. **收集数据**：至少运行 2 周
2. **离线分析**：Jupyter 笔记本回测新参数
3. **纸面测试**：新参数先在纸面账户验证
4. **小规模上线**：用 10% 资金测试
5. **逐步扩大**：验证有效后再全部切换

---

## 完整上线 Checklist

### Pre-Launch Checklist

**代码与系统**
- [ ] 代码最后审查无 Bug
- [ ] 日志系统工作正常
- [ ] 监控告警已测试
- [ ] 备份已生成

**账户配置**
- [ ] Broker API Key 已保存（不在代码中）
- [ ] 初期资金已存入 ($5K-$10K)
- [ ] 杠杆配置已确认
- [ ] 交易品种已确认

**风控系统**
- [ ] 日损失限已设置
- [ ] 单个头寸限已设置
- [ ] 止损单已启用
- [ ] 应急平仓流程已准备

**监控系统**
- [ ] 邮件告警已测试
- [ ] 仪表板已配置
- [ ] 日志记录已验证
- [ ] 性能监控已启用

**纸面交易**
- [ ] 已运行 5+ 天
- [ ] 成交表现 ✓
- [ ] 滑点可接受 ✓
- [ ] 无重大 Bug ✓

**人员与流程**
- [ ] 团队成员已培训
- [ ] 应急联系方式已备案
- [ ] 交接文档已准备
- [ ] SOP 已发布

### Day 1 Checklist

**开盘前 (8:00)**
- [ ] 检查系统是否启动
- [ ] 确认 API 连接
- [ ] 查看是否有夜间告警

**开盘 (9:30)**
- [ ] 监听第一笔订单
- [ ] 检查填充速度
- [ ] 检查成交价格

**每小时 (10:00, 11:00, 12:00, ...)**
- [ ] 浏览 PnL
- [ ] 检查持仓数量
- [ ] 看是否有异常订单

**收盘前 (15:45)**
- [ ] 检查是否需要平仓
- [ ] 确认所有交易已记录

**收盘后 (16:00)**
- [ ] 查看日报告
- [ ] 备份日志
- [ ] 反思是否有改进空间

### Week 1 Checklist

- [ ] Day 1-2：系统稳定性验证 ✓
- [ ] Day 3-5：策略逻辑验证 ✓
  - 是否有意外的成交？
  - 是否有数据延迟问题？
  - 回测与实盘是否相符？
- [ ] 周末：完整周报 ✓
  - 周回报率
  - 风险指标（最大回撤、夏普）
  - 改进计划

---

## 常见上线问题 FAQ

**Q: 我的策略在回测中赚 20%，但纸面交易只赚 5%，为什么？**

A: 常见原因：
1. 回测中假设订单立即成交，实盘有延迟和滑点
2. 回测中忽视了交易成本（佣金、交易税）
3. 数据不同步：回测用的是日线收盘价，实盘是实时价格
4. 可能存在"未来看"（使用了当日尚未发生的数据）

**解决办法**：重新审查策略，加入现实的成交假设。

---

**Q: 我启动实盘后，API 开始频繁超时，怎么办？**

A: 这很常见，原因可能是：
1. 网络问题（换个网络试试）
2. Broker API 限流（Alpaca 有 API 速率限制）
3. 时区问题（如果 Broker 在美国，而你在中国）

**解决办法**：
```python
def PlaceOrderWithRetry(self, symbol, qty, max_retries=3):
    for attempt in range(max_retries):
        try:
            self.Buy(symbol, qty)
            return True
        except TimeoutException:
            wait_time = 2 ** attempt  # 指数退避
            time.sleep(wait_time)
            continue
    
    self.Log(f"Failed to place order after {max_retries} attempts")
    return False
```

---

**Q: 我看着真实资金在亏损，心理上无法承受，怎么办？**

A: 这是正常的。建议：
1. **心理调整**：初期亏损是学费，不是失败
2. **减少监控**：不要每分钟盯着 PnL
3. **明确规则**：已经设置了风控就不要修改
4. **长期视角**：看周/月的表现，而不是日收益

记住：一个 51% 胜率的策略，长期会赚钱。不要因为一次亏损就改变策略。

---

## 总结

从回测到实盘，关键步骤：

1. **心理准备** —— 认识到真实资金是不同的
2. **纸面验证** —— 运行 1-2 周，发现回测与实盘的差异
3. **风控第一** —— 防守比进攻更重要
4. **监控系统** —— 能发现问题比能赚钱更重要
5. **小资本启动** —— 先用 $5K 验证，再扩大规模
6. **运维 SOP** —— 建立每日 / 周 / 月的标准化流程

下一篇，我们讨论如何在团队环境中进行策略开发和部署。

---

## 参考资源

- [Alpaca 官方文档](https://alpaca.markets/docs/)
- [Interactive Brokers API](https://www.interactivebrokers.com/en/trading/ibapi.php)
- [QuantConnect 实盘部署指南](https://www.quantconnect.com/docs/v2/live-trading)
- 《打败市场的人》（A Man for All Markets）—— Jim Simons 自传，讲述如何从学术转向交易

**导航**：[上一篇: 券商插件开发](./08-custom-brokerage.md) | [下一篇: 团队协作与工程化实践](./10-team-practices.md)
