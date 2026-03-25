# 二开高阶：券商插件开发

> **前置要求**：已完成 06、07 篇，对 LEAN 框架和数据源插件有基本理解。
> **目标受众**：2-3 人初创开发团队，需要对接非官方支持的券商。
> **学习成果**：能够从零开发一个券商插件，集成自有或小众交易所。

---

## 为什么要开发券商插件

QuantConnect 官方支持的券商有限（Interactive Brokers、Alpaca、OANDA 等），但现实中你的团队可能面临以下场景：

1. **小众交易所**：对接中国期货公司、新兴 DEX、或私有交易系统
2. **自建风控网关**：在官方券商上层增加自定义风控策略
3. **成本优化**：使用佣金更低但 API 不标准的小券商
4. **技术整合**：将现有的成熟交易系统与 LEAN 引擎统一

**核心价值**：不是为了"更换"券商，而是为了让 LEAN 引擎与任何交易系统对话。

---

## IBrokerage 接口全景

每个券商插件的核心是实现 `IBrokerage` 接口。这个接口定义了 LEAN 与任何交易执行系统的契约。

### 完整接口定义（带中文注释）

```csharp
/// <summary>
/// IBrokerage 是券商网关的标准接口
/// LEAN 通过这个接口与任何交易系统通信
/// </summary>
public interface IBrokerage
{
    // ==================== 连接管理 ====================
    
    /// <summary>连接到券商</summary>
    void Connect();
    
    /// <summary>断开连接</summary>
    void Disconnect();
    
    /// <summary>获取连接状态</summary>
    bool IsConnected { get; }
    
    // ==================== 订单操作 ====================
    
    /// <summary>
    /// 下单
    /// </summary>
    /// <param name="order">LEAN Order 对象，包含代码、数量、方向等</param>
    /// <returns>订单是否成功提交</returns>
    void PlaceOrder(Order order);
    
    /// <summary>
    /// 修改订单（如果券商支持）
    /// </summary>
    /// <param name="order">修改后的订单对象，必须包含原订单 ID</param>
    /// <returns>修改是否成功</returns>
    void UpdateOrder(Order order);
    
    /// <summary>
    /// 撤单
    /// </summary>
    /// <param name="order">要撤销的订单</param>
    /// <returns>撤单是否成功</returns>
    void CancelOrder(Order order);
    
    // ==================== 账户信息查询 ====================
    
    /// <summary>
    /// 获取持仓信息
    /// 返回当前账户的所有持仓（按证券代码）
    /// </summary>
    List<Holding> GetAccountHoldings();
    
    /// <summary>
    /// 获取现金余额
    /// </summary>
    /// <returns>账户现金（美元或本币）</returns>
    decimal GetCashBalance();
    
    // ==================== 事件回调 ====================
    
    /// <summary>
    /// 订单状态变更事件
    /// 当券商反馈订单被成交、部分成交、被拒单等，触发此事件
    /// </summary>
    event EventHandler<OrderEvent> OrderStatusChanged;
    
    /// <summary>
    /// 账户变更事件
    /// 当账户现金或持仓发生变化，触发此事件
    /// </summary>
    event EventHandler<AccountEvent> AccountChanged;
    
    // ==================== 其他 ====================
    
    /// <summary>获取券商名称</summary>
    string Name { get; }
    
    /// <summary>
    /// 检验订单在该券商是否有效
    /// 例：某些券商不支持 GTC（取消前有效）订单
    /// </summary>
    bool ValidateOrder(Order order);
}
```

### 关键接口要素详解

| 方法 | 职责 | 实现难度 | 测试复杂度 |
|------|------|--------|----------|
| `Connect()` | 建立与券商的连接（WebSocket / REST） | 中 | 中 |
| `PlaceOrder()` | 提交订单，获取委托号 | 高 | 高 |
| `CancelOrder()` | 撤销未成交订单 | 中 | 高 |
| `GetAccountHoldings()` | 查询当前持仓 | 中 | 低 |
| `GetCashBalance()` | 查询可用资金 | 低 | 低 |
| `OrderStatusChanged` | 推送订单成交 / 拒单事件 | 高 | 高 |

---

## 券商插件项目结构

基于 QuantConnect 官方仓库 [Lean.Brokerages.InteractiveBrokers](https://github.com/QuantConnect/Lean.Brokerages.InteractiveBrokers) 的标准结构，一个产品级的券商插件应该包含：

### 目录结构

```
Lean.Brokerages.YourBroker/
├── YourBrokerBrokerage.cs          # 主类：实现 IBrokerage
├── YourBrokerBrokerageFactory.cs   # 工厂类：依赖注入
├── YourBrokerBrokerageModel.cs     # 佣金 / 成交模式
├── YourBrokerOrder.cs              # 外部订单对象（可选）
├── YourBrokerException.cs          # 自定义异常
├── Messages/                        # WebSocket 消息模型
│   ├── OrderUpdate.cs
│   ├── AccountUpdate.cs
│   └── ...
├── Tests/
│   ├── YourBrokerBrokerageTests.cs
│   └── YourBrokerBrokerageModelTests.cs
├── Lean.Brokerages.YourBroker.csproj
└── README.md
```

### 核心类详解

#### 1. YourBrokerBrokerage.cs

```csharp
using System;
using System.Collections.Generic;
using QuantConnect;
using QuantConnect.Brokerages;
using QuantConnect.Data;
using QuantConnect.Orders;

namespace QuantConnect.Brokerages.YourBroker
{
    /// <summary>
    /// 券商实现类
    /// 负责与外部交易系统的全部交互逻辑
    /// </summary>
    public class YourBrokerBrokerage : Brokerage
    {
        private readonly string _apiKey;
        private readonly string _apiSecret;
        private WebSocketClient _client;
        private bool _isConnected;
        
        // 维护订单状态缓存，用于查询和事件推送
        private readonly Dictionary<int, Order> _orderMap = new();
        
        public override string Name => "YourBroker";
        public override bool IsConnected => _isConnected;
        
        public YourBrokerBrokerage(string apiKey, string apiSecret)
        {
            _apiKey = apiKey;
            _apiSecret = apiSecret;
        }
        
        /// <summary>连接到券商 WebSocket</summary>
        public override void Connect()
        {
            if (_isConnected) return;
            
            _client = new WebSocketClient(_apiKey, _apiSecret);
            _client.OnMessage += HandleMessage;
            _client.Connect();
            
            _isConnected = true;
            OnConnected();
        }
        
        /// <summary>断开连接</summary>
        public override void Disconnect()
        {
            if (!_isConnected) return;
            
            _client?.Disconnect();
            _isConnected = false;
            OnDisconnected();
        }
        
        /// <summary>下单核心逻辑</summary>
        public override void PlaceOrder(Order order)
        {
            // 1. 验证订单
            if (!ValidateOrder(order))
            {
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Invalid
                });
                return;
            }
            
            try
            {
                // 2. 构建订单请求（转换成券商的 API 格式）
                var brokerOrder = new BrokerOrderRequest
                {
                    Symbol = order.Symbol.Value,
                    Quantity = order.Quantity,
                    Direction = order.Direction == OrderDirection.Buy ? "BUY" : "SELL",
                    Price = order is LimitOrder lo ? lo.LimitPrice : 0,
                    OrderType = GetBrokerOrderType(order),
                    TimeInForce = GetTimeInForce(order)
                };
                
                // 3. 通过 API 提交订单
                var brokerOrderId = _client.PlaceOrder(brokerOrder);
                
                // 4. 缓存订单映射（LEAN订单ID → 券商订单ID）
                order.BrokerId.Add(brokerOrderId);
                _orderMap[order.Id] = order;
                
                // 5. 发送订单提交事件
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Submitted
                });
            }
            catch (Exception ex)
            {
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Invalid,
                    Message = $"Order placement failed: {ex.Message}"
                });
            }
        }
        
        /// <summary>撤单</summary>
        public override void CancelOrder(Order order)
        {
            try
            {
                if (order.BrokerId.Count == 0)
                {
                    OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                    {
                        Status = OrderStatus.CancelPending
                    });
                    return;
                }
                
                var brokerOrderId = order.BrokerId[0];
                _client.CancelOrder(brokerOrderId);
                
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.CancelPending
                });
            }
            catch (Exception ex)
            {
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.CancelPending,
                    Message = $"Cancel failed: {ex.Message}"
                });
            }
        }
        
        /// <summary>获取账户持仓</summary>
        public override List<Holding> GetAccountHoldings()
        {
            var holdings = new List<Holding>();
            var positions = _client.GetPositions();
            
            foreach (var pos in positions)
            {
                holdings.Add(new Holding
                {
                    Symbol = Symbol.Create(pos.Symbol, SecurityType.Equity, Market.USA),
                    Quantity = pos.Quantity,
                    AveragePrice = pos.AveragePrice,
                    MarketPrice = pos.LastPrice,
                    MarketValue = pos.Quantity * pos.LastPrice,
                    UnrealizedPnL = pos.UnrealizedPnL
                });
            }
            
            return holdings;
        }
        
        /// <summary>获取账户现金</summary>
        public override decimal GetCashBalance()
        {
            return _client.GetAccountBalance().AvailableCash;
        }
        
        /// <summary>处理来自券商的 WebSocket 消息（关键回调）</summary>
        private void HandleMessage(string message)
        {
            try
            {
                // 1. 解析 JSON 消息
                var json = JsonConvert.DeserializeObject<dynamic>(message);
                var type = json["type"]?.ToString();
                
                switch (type)
                {
                    case "ORDER_UPDATE":
                        HandleOrderUpdate(json);
                        break;
                    case "ACCOUNT_UPDATE":
                        HandleAccountUpdate(json);
                        break;
                    case "TRADE":
                        HandleTrade(json);
                        break;
                }
            }
            catch (Exception ex)
            {
                OnMessage(new BrokerageMessageEvent(
                    BrokerageMessageType.Warning,
                    0,
                    $"Failed to parse message: {ex.Message}"
                ));
            }
        }
        
        /// <summary>处理订单更新</summary>
        private void HandleOrderUpdate(dynamic json)
        {
            var brokerOrderId = json["order_id"]?.ToString();
            var status = json["status"]?.ToString();
            
            // 根据券商订单ID找到对应的 LEAN Order
            var leanOrder = FindOrderByBrokerId(brokerOrderId);
            if (leanOrder == null) return;
            
            var orderStatus = ConvertBrokerStatus(status);
            OnOrderEvent(new OrderEvent(leanOrder, DateTime.UtcNow)
            {
                Status = orderStatus
            });
        }
        
        /// <summary>处理成交（部分或全部）</summary>
        private void HandleTrade(dynamic json)
        {
            var brokerOrderId = json["order_id"]?.ToString();
            var quantity = (decimal)json["quantity"];
            var price = (decimal)json["price"];
            var commission = (decimal)json["commission"];
            
            var leanOrder = FindOrderByBrokerId(brokerOrderId);
            if (leanOrder == null) return;
            
            OnOrderEvent(new OrderEvent(leanOrder, DateTime.UtcNow)
            {
                Status = OrderStatus.Filled,
                FillQuantity = quantity,
                FillPrice = price
            });
        }
        
        /// <summary>验证订单是否符合券商规范</summary>
        public override bool ValidateOrder(Order order)
        {
            // 例：某些券商不支持 GTC 订单
            if (order.TimeInForce is GoodTilCanceledTimeInForce && !SupportsGTC)
            {
                return false;
            }
            
            // 验证数量是否符合最小手数要求
            if (order.Quantity < MinimumOrderSize)
            {
                return false;
            }
            
            return true;
        }
        
        private bool SupportsGTC => true;
        private decimal MinimumOrderSize => 1;
        
        // ============== 辅助方法 ==============
        
        private Order FindOrderByBrokerId(string brokerOrderId)
        {
            foreach (var order in _orderMap.Values)
            {
                if (order.BrokerId.Contains(brokerOrderId))
                    return order;
            }
            return null;
        }
        
        private string GetBrokerOrderType(Order order)
        {
            return order switch
            {
                LimitOrder => "LIMIT",
                MarketOrder => "MARKET",
                StopMarketOrder => "STOP",
                _ => "MARKET"
            };
        }
        
        private string GetTimeInForce(Order order)
        {
            return order.TimeInForce switch
            {
                DayTimeInForce => "DAY",
                GoodTilCanceledTimeInForce => "GTC",
                _ => "DAY"
            };
        }
        
        private OrderStatus ConvertBrokerStatus(string status)
        {
            return status switch
            {
                "PENDING" => OrderStatus.Submitted,
                "FILLED" => OrderStatus.Filled,
                "CANCELED" => OrderStatus.Canceled,
                "REJECTED" => OrderStatus.Invalid,
                "PARTIAL" => OrderStatus.PartiallyFilled,
                _ => OrderStatus.None
            };
        }
        
        public void Dispose()
        {
            Disconnect();
            _client?.Dispose();
        }
    }
}
```

#### 2. YourBrokerBrokerageFactory.cs（依赖注入）

```csharp
using QuantConnect.Interfaces;

namespace QuantConnect.Brokerages.YourBroker
{
    public class YourBrokerBrokerageFactory : BrokerageFactory
    {
        public override IBrokerage CreateBrokerage(
            LiveNodePacket packet,
            IAlgorithm algorithm,
            IOrderProvider orderProvider,
            IPriceProvider priceProvider)
        {
            var apiKey = packet.BrokerageData["api-key"];
            var apiSecret = packet.BrokerageData["api-secret"];
            
            var brokerage = new YourBrokerBrokerage(apiKey, apiSecret);
            
            // 注册佣金和成交模式
            var model = new YourBrokerBrokerageModel();
            algorithm.SetBrokerageModel(model);
            
            return brokerage;
        }
    }
}
```

#### 3. YourBrokerBrokerageModel.cs（佣金与成交规则）

```csharp
using QuantConnect.Orders;

namespace QuantConnect.Brokerages.YourBroker
{
    /// <summary>
    /// 定义券商的成交规则和费用结构
    /// </summary>
    public class YourBrokerBrokerageModel : DefaultBrokerageModel
    {
        public YourBrokerBrokerageModel()
            : base(AccountType.Margin)
        {
        }
        
        /// <summary>计算佣金</summary>
        public override void ApplyFees(List<OrderEvent> orderEvents)
        {
            foreach (var orderEvent in orderEvents)
            {
                // 例：每笔交易固定 10 美元
                orderEvent.BrokerageFeeCurrency = Currencies.USD;
                orderEvent.BrokerageFee = 10m;
                
                // 或者按百分比计算
                // orderEvent.BrokerageFee = orderEvent.FillPrice * 
                //     orderEvent.FillQuantity * 0.001m; // 0.1%
            }
        }
        
        /// <summary>定义成交模式</summary>
        public override IFillModel GetFillModel(Security security)
        {
            // 使用自定义成交模式（见下文）
            return new YourBrokerFillModel();
        }
        
        /// <summary>定义滑点模式</summary>
        public override ISlippageModel GetSlippageModel(Security security)
        {
            return new ConstantSlippageModel(0.01m); // 固定 0.01 美元滑点
        }
    }
}
```

---

## Lab: 开发一个模拟券商插件（MockBrokerage）

现在我们从零开始，开发一个完整的 **模拟券商插件**，用于测试和演示。

### 项目创建

```bash
# 创建 C# 类库项目
dotnet new classlib -n QuantConnect.Brokerages.Mock -f net6.0

# 添加 LEAN 核心 NuGet 包
cd QuantConnect.Brokerages.Mock
dotnet add package QuantConnect.Common
dotnet add package QuantConnect.Lean
```

### 实现 MockBrokerage

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using QuantConnect;
using QuantConnect.Brokerages;
using QuantConnect.Orders;
using QuantConnect.Data;

namespace QuantConnect.Brokerages.Mock
{
    /// <summary>
    /// MockBrokerage：完整的模拟券商实现
    /// 功能：
    /// - 随机延迟模拟网络延迟
    /// - 模拟部分成交
    /// - 模拟订单拒绝
    /// - 维护账户状态
    /// </summary>
    public class MockBrokerage : Brokerage, IDisposable
    {
        private bool _isConnected;
        private decimal _cashBalance;
        private readonly Dictionary<int, Order> _activeOrders = new();
        private readonly Dictionary<int, List<OrderEvent>> _orderHistory = new();
        private readonly Random _random = new(42);
        private readonly object _lock = new();
        
        public override string Name => "MockBrokerage";
        public override bool IsConnected => _isConnected;
        
        public MockBrokerage(decimal initialCash = 100000m)
        {
            _cashBalance = initialCash;
        }
        
        public override void Connect()
        {
            _isConnected = true;
            OnConnected();
            Log("MockBrokerage connected");
        }
        
        public override void Disconnect()
        {
            _isConnected = false;
            OnDisconnected();
            Log("MockBrokerage disconnected");
        }
        
        /// <summary>
        /// 下单：模拟异步填充过程
        /// 策略：
        /// 1. 立即返回 Submitted 状态
        /// 2. 启动后台任务模拟成交
        /// 3. 随机决定是否成交、拒单、部分成交
        /// </summary>
        public override void PlaceOrder(Order order)
        {
            if (!IsConnected)
            {
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Invalid,
                    Message = "Brokerage not connected"
                });
                return;
            }
            
            lock (_lock)
            {
                // 分配委托号
                order.BrokerId.Add(order.Id.ToString());
                _activeOrders[order.Id] = order;
            }
            
            // 立即发送 Submitted 事件
            OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
            {
                Status = OrderStatus.Submitted,
                Message = "Order submitted to mock broker"
            });
            
            // 启动后台填充任务
            Task.Run(() => SimulateFill(order));
        }
        
        /// <summary>撤单</summary>
        public override void CancelOrder(Order order)
        {
            lock (_lock)
            {
                if (!_activeOrders.ContainsKey(order.Id))
                {
                    return;
                }
                
                _activeOrders.Remove(order.Id);
            }
            
            OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
            {
                Status = OrderStatus.Canceled,
                Message = "Order canceled"
            });
        }
        
        /// <summary>获取持仓（演示用，硬编码）</summary>
        public override List<Holding> GetAccountHoldings()
        {
            return new List<Holding>();
        }
        
        /// <summary>获取现金</summary>
        public override decimal GetCashBalance()
        {
            lock (_lock)
            {
                return _cashBalance;
            }
        }
        
        /// <summary>验证订单</summary>
        public override bool ValidateOrder(Order order)
        {
            // 简单验证：数量 > 0
            return order.Quantity > 0;
        }
        
        // ============== 模拟逻辑 ==============
        
        /// <summary>后台任务：模拟订单填充过程</summary>
        private void SimulateFill(Order order)
        {
            // 1. 随机延迟（100-500ms）
            var delayMs = _random.Next(100, 500);
            Thread.Sleep(delayMs);
            
            // 2. 随机决定订单命运
            var dice = _random.Next(0, 100);
            
            if (dice < 5)
            {
                // 5% 概率：拒单
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Invalid,
                    Message = "Order rejected by mock broker (simulated)"
                });
            }
            else if (dice < 20)
            {
                // 15% 概率：部分成交
                var fillQuantity = (long)(order.Quantity * 0.7m);
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.PartiallyFilled,
                    FillQuantity = fillQuantity,
                    FillPrice = 100m, // 演示价格
                    Message = $"Partial fill: {fillQuantity}/{order.Quantity}"
                });
                
                // 再等待一段时间，填充剩余部分
                Thread.Sleep(200);
                
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Filled,
                    FillQuantity = order.Quantity - fillQuantity,
                    FillPrice = 100m,
                    Message = "Remaining quantity filled"
                });
            }
            else
            {
                // 80% 概率：完全成交
                OnOrderEvent(new OrderEvent(order, DateTime.UtcNow)
                {
                    Status = OrderStatus.Filled,
                    FillQuantity = order.Quantity,
                    FillPrice = 100m,
                    Message = "Order fully filled"
                });
            }
            
            // 3. 更新账户现金
            lock (_lock)
            {
                _activeOrders.Remove(order.Id);
                // 演示：不真实更新现金，只是记录历史
                if (!_orderHistory.ContainsKey(order.Id))
                {
                    _orderHistory[order.Id] = new List<OrderEvent>();
                }
            }
        }
        
        private void Log(string message)
        {
            Console.WriteLine($"[MockBrokerage] {DateTime.UtcNow:HH:mm:ss.fff} {message}");
        }
    }
}
```

### 在策略中使用 MockBrokerage

```python
# 在 QC 本地或云平台上使用（注意：QC 实盘通常用 C# 驱动，Python 用于策略逻辑）
# 这里展示如何在 C# 中注册和使用

from AlgorithmImports import *

class MockBrokerageStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2024, 1, 1)
        self.SetEndDate(2024, 12, 31)
        self.SetCash(100000)
        
        # 注册 MockBrokerage（伪代码，实际需在 C# 中）
        # self.SetBrokerageModel(BrokerageModel.Mock)
        
        self.AddEquity("AAPL", Resolution.Daily)
        self.BuyCount = 0
    
    def OnData(self, data):
        if self.BuyCount < 3:
            self.Buy("AAPL", 10)
            self.BuyCount += 1
```

### 测试 MockBrokerage

```csharp
[TestClass]
public class MockBrokerageTests
{
    private MockBrokerage _brokerage;
    
    [TestInitialize]
    public void Setup()
    {
        _brokerage = new MockBrokerage(100000m);
        _brokerage.Connect();
    }
    
    [TestMethod]
    public void TestOrderSubmission()
    {
        var order = new MarketOrder(
            Symbol.Create("AAPL", SecurityType.Equity, Market.USA),
            100,
            DateTime.UtcNow
        );
        
        var orderEventReceived = false;
        _brokerage.OrderStatusChanged += (sender, e) =>
        {
            if (e.Status == OrderStatus.Submitted)
            {
                orderEventReceived = true;
            }
        };
        
        _brokerage.PlaceOrder(order);
        
        Assert.IsTrue(orderEventReceived, "Should receive order submission event");
    }
    
    [TestMethod]
    public void TestCashBalance()
    {
        var balance = _brokerage.GetCashBalance();
        Assert.AreEqual(100000m, balance);
    }
    
    [TestCleanup]
    public void Teardown()
    {
        _brokerage.Disconnect();
        _brokerage.Dispose();
    }
}
```

---

## 真实场景：对接 REST API 券商

现在考虑对接一个真实的小券商（假设为 "ChinaEx"），它提供：
- REST API 用于下单、查询
- WebSocket 用于推送成交和账户更新

### 架构设计

```
┌─────────────────────────────────────────┐
│         LEAN Algorithm Engine           │
│  (调用 IBrokerage 接口)                 │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│    ChinaExBrokerage (IBrokerage)        │
│                                         │
│  ┌─────────────────┐  ┌──────────────┐ │
│  │  REST Client    │  │ WebSocket    │ │
│  │  (order mgmt)   │  │ (market data)│ │
│  └────────┬────────┘  └──────┬───────┘ │
│           │                  │         │
└───────────┼──────────────────┼─────────┘
            │                  │
        ┌───▼──────────────────▼────┐
        │   ChinaEx API Server      │
        │  - POST /orders           │
        │  - DELETE /orders/{id}    │
        │  - GET /account           │
        │  - WS /stream             │
        └──────────────────────────┘
```

### 核心实现要点

```csharp
public class ChinaExBrokerage : Brokerage
{
    private readonly string _apiKey;
    private readonly string _apiSecret;
    private readonly HttpClient _httpClient;
    private WebSocketClient _wsClient;
    
    // 订单ID映射：LEAN Order ID → 券商订单ID
    private readonly Dictionary<int, string> _orderIdMap = new();
    
    // 请求队列：防止 API 限流
    private readonly Queue<OrderRequest> _requestQueue = new();
    private readonly SemaphoreSlim _rateLimiter;
    
    // 响应缓存：确保状态一致性
    private readonly ConcurrentDictionary<string, OrderStatus> _statusCache = new();
    
    public ChinaExBrokerage(string apiKey, string apiSecret)
    {
        _apiKey = apiKey;
        _apiSecret = apiSecret;
        _httpClient = new HttpClient();
        _rateLimiter = new SemaphoreSlim(10); // 最多同时 10 个请求
    }
    
    /// <summary>
    /// 连接生命周期：
    /// 1. 验证 API Key
    /// 2. 同步账户信息
    /// 3. 建立 WebSocket 连接
    /// 4. 订阅推送
    /// </summary>
    public override void Connect()
    {
        try
        {
            // 1. 验证连接
            ValidateConnection();
            
            // 2. 启动 WebSocket
            _wsClient = new WebSocketClient(_apiKey, _apiSecret);
            _wsClient.OnMessage += HandleWebSocketMessage;
            _wsClient.Connect();
            
            // 3. 订阅账户更新和订单更新
            _wsClient.Subscribe("orders");
            _wsClient.Subscribe("account");
            
            OnConnected();
        }
        catch (Exception ex)
        {
            OnMessage(new BrokerageMessageEvent(
                BrokerageMessageType.Error,
                0,
                $"Connection failed: {ex.Message}"
            ));
        }
    }
    
    /// <summary>
    /// 订单生命周期：
    /// SUBMITTED → [PARTIALLY_FILLED]* → FILLED | CANCELED | REJECTED
    /// 
    /// 处理挑战：
    /// 1. 网络重试：order 可能重复提交
    /// 2. 部分成交：需要追踪累计成交量
    /// 3. 订单修改：某些券商支持，需要比对状态
    /// </summary>
    public override void PlaceOrder(Order order)
    {
        if (!ValidateOrder(order))
        {
            OnOrderEvent(CreateOrderEvent(order, OrderStatus.Invalid));
            return;
        }
        
        // 使用信号量控制并发数
        _rateLimiter.Wait();
        
        try
        {
            // 构建 API 请求
            var payload = new OrderPayload
            {
                Symbol = order.Symbol.Value,
                Side = order.Direction == OrderDirection.Buy ? "BUY" : "SELL",
                Quantity = order.Quantity,
                OrderType = GetOrderType(order),
                Price = order is LimitOrder lo ? lo.LimitPrice : (decimal?)null,
                TimeInForce = "GTC"
            };
            
            // 签名请求（HMAC-SHA256）
            var signature = HmacSha256(JsonConvert.SerializeObject(payload), _apiSecret);
            
            // 发送 POST /orders
            var request = new HttpRequestMessage(HttpMethod.Post, "/api/orders")
            {
                Content = new StringContent(
                    JsonConvert.SerializeObject(payload),
                    Encoding.UTF8,
                    "application/json"
                )
            };
            request.Headers.Add("X-API-Key", _apiKey);
            request.Headers.Add("X-Signature", signature);
            request.Headers.Add("X-Timestamp", DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString());
            
            var response = _httpClient.SendAsync(request).GetAwaiter().GetResult();
            
            if (response.IsSuccessStatusCode)
            {
                var content = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
                var result = JsonConvert.DeserializeObject<OrderResponse>(content);
                
                // 保存映射：LEAN ID → 券商 ID
                order.BrokerId.Add(result.OrderId);
                _orderIdMap[order.Id] = result.OrderId;
                
                // 状态缓存
                _statusCache[result.OrderId] = OrderStatus.Submitted;
                
                OnOrderEvent(CreateOrderEvent(order, OrderStatus.Submitted));
            }
            else
            {
                var error = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
                OnOrderEvent(CreateOrderEvent(order, OrderStatus.Invalid, error));
            }
        }
        catch (Exception ex)
        {
            OnOrderEvent(CreateOrderEvent(order, OrderStatus.Invalid, ex.Message));
        }
        finally
        {
            _rateLimiter.Release();
        }
    }
    
    /// <summary>
    /// 撤单：异步操作，需要等待确认
    /// </summary>
    public override void CancelOrder(Order order)
    {
        if (order.BrokerId.Count == 0)
        {
            OnOrderEvent(CreateOrderEvent(order, OrderStatus.CancelPending));
            return;
        }
        
        var brokerOrderId = order.BrokerId[0];
        
        _rateLimiter.Wait();
        try
        {
            var request = new HttpRequestMessage(
                HttpMethod.Delete,
                $"/api/orders/{brokerOrderId}"
            );
            
            var signature = HmacSha256(brokerOrderId, _apiSecret);
            request.Headers.Add("X-API-Key", _apiKey);
            request.Headers.Add("X-Signature", signature);
            
            var response = _httpClient.SendAsync(request).GetAwaiter().GetResult();
            
            if (response.IsSuccessStatusCode)
            {
                OnOrderEvent(CreateOrderEvent(order, OrderStatus.CancelPending));
            }
        }
        catch (Exception ex)
        {
            OnMessage(new BrokerageMessageEvent(
                BrokerageMessageType.Warning,
                0,
                $"Cancel order {brokerOrderId} failed: {ex.Message}"
            ));
        }
        finally
        {
            _rateLimiter.Release();
        }
    }
    
    /// <summary>
    /// WebSocket 消息处理：处理来自券商的实时推送
    /// 这是最关键的方法，决定了策略能否看到真实的成交信息
    /// </summary>
    private void HandleWebSocketMessage(string message)
    {
        try
        {
            var json = JObject.Parse(message);
            var type = json["type"]?.ToString();
            
            switch (type)
            {
                case "ORDER_UPDATE":
                    {
                        var brokerOrderId = json["order_id"]?.ToString();
                        var status = json["status"]?.ToString(); // SUBMITTED, FILLED, CANCELED, etc.
                        var filledQty = (decimal?)json["filled_qty"] ?? 0;
                        var filledPrice = (decimal?)json["filled_price"] ?? 0;
                        
                        // 从映射表找到 LEAN Order
                        var leanOrderId = _orderIdMap
                            .FirstOrDefault(x => x.Value == brokerOrderId).Key;
                        
                        if (leanOrderId == 0) return;
                        
                        var leanOrder = new Order { Id = leanOrderId };
                        
                        // 根据状态转换
                        var orderStatus = ConvertStatus(status);
                        _statusCache[brokerOrderId] = orderStatus;
                        
                        var orderEvent = CreateOrderEvent(
                            leanOrder,
                            orderStatus
                        );
                        
                        // 如果是成交，记录价格和数量
                        if (filledQty > 0)
                        {
                            orderEvent.FillQuantity = filledQty;
                            orderEvent.FillPrice = filledPrice;
                        }
                        
                        OnOrderEvent(orderEvent);
                        break;
                    }
                
                case "ACCOUNT_UPDATE":
                    {
                        var balance = (decimal?)json["available_balance"] ?? 0;
                        OnMessage(new BrokerageMessageEvent(
                            BrokerageMessageType.Information,
                            0,
                            $"Account balance: {balance}"
                        ));
                        break;
                    }
            }
        }
        catch (Exception ex)
        {
            OnMessage(new BrokerageMessageEvent(
                BrokerageMessageType.Warning,
                0,
                $"Failed to handle message: {ex.Message}"
            ));
        }
    }
    
    /// <summary>
    /// 错误处理和重试逻辑
    /// 关键问题：如果网络中断，LEAN 和券商的状态会不同步
    /// 解决方案：定期对账
    /// </summary>
    private void ReconcileOrders()
    {
        // 定时任务（每 30 秒）：查询券商未结订单，与本地缓存对比
        
        try
        {
            var brokerOrders = GetOpenOrders(); // REST API 查询
            
            foreach (var brokerOrder in brokerOrders)
            {
                var leanOrderId = _orderIdMap
                    .FirstOrDefault(x => x.Value == brokerOrder.OrderId).Key;
                
                if (leanOrderId == 0) continue;
                
                var cachedStatus = _statusCache.GetOrAdd(
                    brokerOrder.OrderId,
                    OrderStatus.Submitted
                );
                
                var actualStatus = ConvertStatus(brokerOrder.Status);
                
                // 如果状态不一致，更新缓存并推送事件
                if (cachedStatus != actualStatus)
                {
                    _statusCache[brokerOrder.OrderId] = actualStatus;
                    
                    var order = new Order { Id = leanOrderId };
                    OnOrderEvent(CreateOrderEvent(order, actualStatus));
                }
            }
        }
        catch (Exception ex)
        {
            OnMessage(new BrokerageMessageEvent(
                BrokerageMessageType.Warning,
                0,
                $"Reconciliation failed: {ex.Message}"
            ));
        }
    }
    
    // ============== 其他方法 ==============
    
    private List<BrokerOrder> GetOpenOrders()
    {
        // GET /api/orders?status=open
        // 返回当前未结订单列表
        throw new NotImplementedException();
    }
    
    private string HmacSha256(string message, string secret)
    {
        var key = Encoding.UTF8.GetBytes(secret);
        using var hmac = new HMACSHA256(key);
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(message));
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }
    
    private void ValidateConnection()
    {
        // 测试 API Key 是否有效
        var request = new HttpRequestMessage(HttpMethod.Get, "/api/account");
        var response = _httpClient.SendAsync(request).GetAwaiter().GetResult();
        
        if (!response.IsSuccessStatusCode)
        {
            throw new InvalidOperationException("API Key validation failed");
        }
    }
    
    private OrderStatus ConvertStatus(string status)
    {
        return status switch
        {
            "SUBMITTED" => OrderStatus.Submitted,
            "PARTIALLY_FILLED" => OrderStatus.PartiallyFilled,
            "FILLED" => OrderStatus.Filled,
            "CANCELED" => OrderStatus.Canceled,
            "REJECTED" => OrderStatus.Invalid,
            _ => OrderStatus.None
        };
    }
    
    private string GetOrderType(Order order)
    {
        return order switch
        {
            LimitOrder => "LIMIT",
            StopMarketOrder => "STOP_MARKET",
            _ => "MARKET"
        };
    }
    
    private OrderEvent CreateOrderEvent(
        Order order,
        OrderStatus status,
        string message = null)
    {
        return new OrderEvent(order, DateTime.UtcNow)
        {
            Status = status,
            Message = message
        };
    }
    
    public override List<Holding> GetAccountHoldings()
    {
        // 从 REST API 查询持仓
        throw new NotImplementedException();
    }
    
    public override decimal GetCashBalance()
    {
        // 从 REST API 查询现金
        throw new NotImplementedException();
    }
    
    public override bool ValidateOrder(Order order)
    {
        // 检查订单是否符合券商规范
        return order.Quantity > 0;
    }
    
    public override void Disconnect()
    {
        _wsClient?.Disconnect();
    }
}
```

---

## 测试策略

### 单元测试

```csharp
[TestClass]
public class ChinaExBrokerageTests
{
    private ChinaExBrokerage _brokerage;
    
    [TestInitialize]
    public void Setup()
    {
        _brokerage = new ChinaExBrokerage("test-key", "test-secret");
    }
    
    [TestMethod]
    public void TestOrderValidation()
    {
        var order = new MarketOrder(
            Symbol.Create("AAPL", SecurityType.Equity, Market.USA),
            100,
            DateTime.UtcNow
        );
        
        Assert.IsTrue(_brokerage.ValidateOrder(order));
    }
    
    [TestMethod]
    public void TestWebSocketMessageParsing()
    {
        // 模拟券商推送的 WebSocket 消息
        var message = JsonConvert.SerializeObject(new
        {
            type = "ORDER_UPDATE",
            order_id = "123456",
            status = "FILLED",
            filled_qty = 100m,
            filled_price = 101.50m
        });
        
        // 验证消息被正确处理
        // _brokerage.HandleMessage(message);
    }
}
```

### 集成测试

```csharp
[TestClass]
public class ChinaExBrokerageIntegrationTests
{
    [TestMethod]
    public void TestLiveOrderFlow()
    {
        var brokerage = new ChinaExBrokerage(
            Environment.GetEnvironmentVariable("BROKER_API_KEY"),
            Environment.GetEnvironmentVariable("BROKER_API_SECRET")
        );
        
        brokerage.Connect();
        
        try
        {
            var order = new MarketOrder(
                Symbol.Create("AAPL", SecurityType.Equity, Market.USA),
                10,
                DateTime.UtcNow
            );
            
            var filled = false;
            brokerage.OrderStatusChanged += (sender, e) =>
            {
                if (e.Status == OrderStatus.Filled)
                    filled = true;
            };
            
            brokerage.PlaceOrder(order);
            
            // 等待填充（最多 5 秒）
            var sw = Stopwatch.StartNew();
            while (!filled && sw.ElapsedMilliseconds < 5000)
            {
                Thread.Sleep(100);
            }
            
            Assert.IsTrue(filled, "Order should be filled within 5 seconds");
        }
        finally
        {
            brokerage.Disconnect();
        }
    }
}
```

---

## 部署与维护

### 打包为 NuGet

```xml
<!-- Lean.Brokerages.ChinaEx.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <PackageId>QuantConnect.Brokerages.ChinaEx</PackageId>
    <Version>1.0.0</Version>
    <Authors>Your Company</Authors>
    <Description>ChinaEx brokerage plugin for QuantConnect LEAN</Description>
  </PropertyGroup>
</Project>
```

### 在 LEAN 中注册插件

LEAN 通过扫描 `Lean.Brokerages.*` NuGet 包自动发现插件：

```csharp
// LEAN 核心代码会自动加载所有 IBrokerageFactory 实现
public class BrokerageFactory
{
    public static IBrokerage CreateBrokerage(
        string name,
        IAlgorithm algorithm,
        LiveNodePacket packet)
    {
        // 通过反射加载所有 XxxBrokerageFactory
        var factoryType = Type.GetType($"QuantConnect.Brokerages.{name}.{name}BrokerageFactory");
        var factory = Activator.CreateInstance(factoryType) as BrokerageFactory;
        return factory.CreateBrokerage(packet, algorithm, ...);
    }
}
```

---

## 从 Python 到 C# 的过渡

由于券商插件通常需要 C#，这里为 Python 开发者准备的速查表：

| 概念 | Python | C# |
|------|--------|-----|
| 类定义 | `class Foo:` | `public class Foo { }` |
| 继承 | `class Child(Parent):` | `public class Child : Parent { }` |
| 接口实现 | 无显式语法 | `public class X : IFoo { }` |
| 属性 | `self.value` | `this.Value` 或属性访问器 |
| 方法重写 | `def method(self):` | `public override void Method()` |
| 异步 | `async def foo():` await | `async Task Method() { await ... }` |
| 字典 | `dict` | `Dictionary<K, V>` |
| 列表 | `list` | `List<T>` |
| 异常 | `except Exception as e:` | `catch (Exception ex)` |

### 实际例子：Python 与 C# 对比

**Python（回测脚本）**
```python
def Initialize(self):
    self.SetBrokerageModel("ChinaEx")
    self.Buy("AAPL", 100)
```

**C#（券商实现）**
```csharp
public class ChinaExBrokerageFactory : BrokerageFactory
{
    public override IBrokerage CreateBrokerage(
        LiveNodePacket packet,
        IAlgorithm algorithm,
        IOrderProvider orderProvider,
        IPriceProvider priceProvider)
    {
        var apiKey = packet.BrokerageData["api-key"];
        var brokerage = new ChinaExBrokerage(apiKey);
        return brokerage;
    }
}

public class ChinaExBrokerage : Brokerage
{
    public override void PlaceOrder(Order order)
    {
        // 实现订单逻辑
    }
}
```

---

## 总结

开发券商插件的关键步骤：

1. **理解 IBrokerage 接口** —— 这是 LEAN 与任何交易系统的桥梁
2. **实现核心方法** —— PlaceOrder, CancelOrder, GetAccountHoldings 等
3. **处理异步事件** —— OrderStatusChanged 事件推送
4. **测试错误场景** —— 网络延迟、拒单、部分成交
5. **维护状态一致性** —— 定期对账，防止 LEAN 与券商状态不同步
6. **打包发布** —— 作为 NuGet 包供 LEAN 自动加载

下一篇，我们讨论如何将测试好的策略部署到生产环境。

---

## 参考资源

- [QuantConnect 官方券商插件仓库](https://github.com/QuantConnect/Lean.Brokerages)
- [IBrokerage 接口文档](https://github.com/QuantConnect/Lean/blob/master/Brokerages/IBrokerage.cs)
- [Interactive Brokers 插件实现](https://github.com/QuantConnect/Lean.Brokerages.InteractiveBrokers)
- C# async/await 教程：https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/

**导航**：[上一篇: 自定义数据源](./07-custom-datasource.md) | [下一篇: 从回测到实盘](./09-go-live-checklist.md)
