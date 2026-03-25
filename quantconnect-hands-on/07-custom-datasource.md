# 二开进阶：自定义数据源

## 目录

1. [什么是自定义数据源](#什么是自定义数据源)
2. [LEAN 自定义数据架构](#lean-自定义数据架构)
3. [Lab 1: 接入 CSV 文件数据源](#lab-1-接入-csv-文件数据源)
4. [Lab 2: 接入 REST API 数据源](#lab-2-接入-rest-api-数据源)
5. [Lab 3: 自定义 Universe Selection 数据源](#lab-3-自定义-universe-selection-数据源)
6. [数据格式规范](#数据格式规范)
7. [测试与验证](#测试与验证)
8. [从数据源到交易信号的完整管线](#从数据源到交易信号的完整管线)
9. [常见问题](#常见问题)

---

## 什么是自定义数据源

LEAN 的核心设计理念之一是：**数据可以来自任何地方**。

不仅限于市场价格。任何能够表示为 **时间序列的数据** 都可以被 LEAN 消费：

- **情绪数据**：VIX、恐惧指数、社交媒体情绪评分
- **宏观数据**：GDP、失业率、利率、消费者信心指数
- **基本面数据**：每股收益、债务比率、现金流
- **另类数据**：卫星图像、运输流量、超市售价、信用卡交易
- **专有数据**：你自己的研究输出、机器学习模型的预测、内部风险评分

这意味着你可以：

1. 在一个统一的框架内混合使用市场数据和非市场数据
2. 使用自定义数据来触发交易信号
3. 在回测中完整地再现你的想法（包括数据采购步骤）

### 用例

**案例 1**：情绪套利

你有一个外部数据源提供的股票情绪评分（1-100）。当情绪分数与价格动作背离时，你想交易。

**案例 2**：替代数据

你购买了卫星图像数据，显示某个零售商的停车场占用率。这是先行指标——停车场拥挤 → 销售可能增长 → 股价可能上升。

**案例 3**：多资产策略

你想在美国股票上交易，但使用中国债券的收益率作为信号。自定义数据源让你轻松做到这一点。

---

## LEAN 自定义数据架构

### 核心概念

在 LEAN 中，所有数据（市场和自定义）都继承自 **BaseData** 基类。

```python
class CustomDataExample(BaseData):
    
    # 声明数据字段
    sentiment_score = None
    
    def GetSource(self, config, date, isLive):
        """
        告诉 LEAN 去哪里找这个数据。
        
        参数:
            config: SubscriptionDataConfig 对象
            date: 日期（用于多文件数据源）
            isLive: 是否在实盘/Paper Trading（True）还是回测（False）
        
        返回:
            SubscriptionDataSource 对象，包含 URL 或文件路径
        """
        # 例如：返回本地 CSV 文件路径或 HTTP URL
        pass
    
    def Reader(self, config, line, date, isLive):
        """
        解析单行数据。
        
        LEAN 会逐行读取你的数据源，
        对每一行调用 Reader()。
        
        参数:
            config: SubscriptionDataConfig
            line: 单行文本（字符串）
            date: 日期
            isLive: 是否实盘
        
        返回:
            此行解析后的 BaseData 实例（或 None 表示跳过此行）
        """
        pass
```

### 数据流

```
初始化策略 (Initialize)
    ↓
调用 self.AddData(CustomDataType, symbol)
    ↓
LEAN 调用 CustomDataType.GetSource() 获取数据位置
    ↓
LEAN 打开数据文件/URL
    ↓
逐行读取，对每行调用 CustomDataType.Reader()
    ↓
解析后的数据对象被纳入 TimeSlice
    ↓
策略的 OnData() 接收这些数据对象
    ↓
你可以通过 data[CustomDataType] 访问
```

### 与 BaseData 的关键交互

```python
class SentimentData(BaseData):
    
    def GetSource(self, config, date, isLive):
        # URL 模式（LEAN 会自动替换 {0} 为日期）
        url = f"https://api.example.com/sentiment/{date.strftime('%Y-%m-%d')}"
        
        return SubscriptionDataSource(
            source=url,
            format=DataFeedFormat.Csv,
            dataType=DataType.TradeBar  # 通常用 TradeBar 或 Csv
        )
    
    def Reader(self, config, line, date, isLive):
        if not line.strip():
            return None  # 跳过空行
        
        # 假设 CSV 格式：Symbol,Date,Time,SentimentScore,Confidence
        parts = line.split(',')
        
        obj = SentimentData()
        obj.Symbol = parts[0]
        obj.Time = datetime.strptime(f"{parts[1]} {parts[2]}", "%Y-%m-%d %H:%M")
        obj.sentiment_score = float(parts[3])
        obj.confidence = float(parts[4])
        obj.Value = obj.sentiment_score  # LEAN 期望 Value 属性
        
        return obj
```

更多细节见：[架构参考：数据管线](../quantconnect-deep-dive/03-data-pipeline.md)

---

## Lab 1: 接入 CSV 文件数据源

### 场景

你的团队维护了一个每日更新的 CSV 文件，包含每只股票的情绪评分。该文件存储在 QuantConnect 的数据文件夹中。

文件路径：`data/custom/sentiment/AAPL_sentiment.csv`

CSV 格式：
```
Date,Time,SentimentScore,Confidence
2023-01-01,16:00,0.72,0.95
2023-01-02,16:00,0.68,0.92
2023-01-03,16:00,0.81,0.98
```

### 步骤 1：定义 SentimentData 类

```python
# 文件: custom_data.py

from AlgorithmImports import *
from datetime import datetime

class SentimentData(BaseData):
    """
    自定义数据类：股票情绪评分
    
    数据源：CSV 文件，包含每日情绪评分和置信度。
    LEAN 会自动调用 GetSource() 获取数据位置，
    然后调用 Reader() 解析每一行。
    """
    
    def __init__(self):
        super().__init__()
        self.sentiment_score = 0.0  # 0.0 - 1.0
        self.confidence = 0.0        # 0.0 - 1.0
    
    def GetSource(self, config, date, isLive):
        """
        告诉 LEAN 去哪里找这个数据。
        
        这里我们使用 QuantConnect 的内置路径格式。
        LEAN 会自动查找 data/ 文件夹内的相对路径。
        """
        
        # 从 config 中提取代码
        ticker = config.Symbol.Value
        
        # 构造文件路径
        # QuantConnect 期望相对于 'data/' 目录的路径
        path = f"custom/sentiment/{ticker}_sentiment.csv"
        
        # 返回数据源
        # 参数：source（路径或 URL）、format（数据格式）
        source = SubscriptionDataSource(
            source=path,
            format=DataFeedFormat.Csv  # CSV 格式
        )
        
        return source
    
    def Reader(self, config, line, date, isLive):
        """
        解析 CSV 的单一行。
        
        LEAN 会逐行调用此方法。
        """
        
        # 跳过空行或标题行
        if not line or line.startswith('Date'):
            return None
        
        try:
            parts = line.split(',')
            
            # 解析字段
            date_str = parts[0].strip()           # "2023-01-01"
            time_str = parts[1].strip()           # "16:00"
            sentiment_str = parts[2].strip()      # "0.72"
            confidence_str = parts[3].strip()     # "0.95"
            
            # 构造时间戳
            timestamp = datetime.strptime(
                f"{date_str} {time_str}",
                "%Y-%m-%d %H:%M"
            )
            
            # 创建数据对象
            obj = SentimentData()
            obj.Symbol = config.Symbol  # 使用配置中的代码
            obj.Time = timestamp
            obj.sentiment_score = float(sentiment_str)
            obj.confidence = float(confidence_str)
            
            # LEAN 期望 Value 属性，通常是主要数据点
            obj.Value = obj.sentiment_score
            
            return obj
        
        except Exception as e:
            # 如果解析失败，返回 None（跳过此行）
            return None
```

### 步骤 2：在策略中订阅

```python
class SentimentCsvStrategy(QCAlgorithm):
    """
    使用 CSV 情绪数据源的策略。
    
    逻辑：
    - 当情绪分数 > 0.7 且置信度 > 0.9 时，买入
    - 当情绪分数 < 0.3 或置信度下降时，卖出
    """
    
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 添加股票数据
        self.stock = self.AddEquity("AAPL", Resolution.Daily).Symbol
        
        # 添加自定义数据源
        # AddData() 会告诉 LEAN：
        # 1. 订阅 SentimentData 数据类
        # 2. 为代码 "AAPL" 加载数据
        # 3. 按日分辨率
        self.AddData(
            dataType=SentimentData,
            symbol="AAPL",
            resolution=Resolution.Daily
        )
        
        # 内部持仓跟踪
        self.last_sentiment = None
    
    def OnData(self, data):
        """
        每个 bar 调用一次。
        
        data 对象包含该 bar 的所有已订阅数据：
        - data[self.stock]：AAPL 的 OHLCV
        - data[SentimentData]：AAPL 的情绪数据
        """
        
        # 检查是否有股票数据
        if self.stock not in data:
            return
        
        price = data[self.stock].Close
        
        # 检查是否有情绪数据
        # 注意：市场数据和自定义数据可能不总是同时到达
        if SentimentData in data:
            sentiment_obj = data[SentimentData]
            sentiment_score = sentiment_obj.sentiment_score
            confidence = sentiment_obj.confidence
            
            self.last_sentiment = sentiment_score
            
            # 交易逻辑
            if not self.Portfolio[self.stock].Invested:
                # 买入条件：高情绪且高置信度
                if sentiment_score > 0.7 and confidence > 0.9:
                    self.SetHoldings(self.stock, 0.95)
                    self.Debug(
                        f"BUY: sentiment={sentiment_score:.2f}, "
                        f"confidence={confidence:.2f}, price={price:.2f}"
                    )
            
            else:  # 已持仓
                # 卖出条件：情绪转差或置信度下降
                if sentiment_score < 0.4 or confidence < 0.75:
                    self.Liquidate(self.stock)
                    self.Debug(
                        f"SELL: sentiment={sentiment_score:.2f}, "
                        f"confidence={confidence:.2f}, price={price:.2f}"
                    )
```

### 步骤 3：准备 CSV 文件

在你的 QuantConnect 项目中：

1. 在左侧菜单中选择 **Data** → **Local**
2. 创建文件夹结构：`data/custom/sentiment/`
3. 上传 `AAPL_sentiment.csv`

CSV 格式示例：

```csv
Date,Time,SentimentScore,Confidence
2023-01-01,16:00,0.72,0.95
2023-01-02,16:00,0.68,0.92
2023-01-03,16:00,0.81,0.98
2023-01-04,16:00,0.55,0.87
2023-01-05,16:00,0.42,0.91
```

### 步骤 4：运行和验证

运行回测，观察：

1. 日志是否显示 BUY/SELL 信号
2. 性能指标是否合理
3. 情绪数据是否正确解析

---

## Lab 2: 接入 REST API 数据源

### 场景

与其依赖本地 CSV，你想实时获取外部 API 的数据。例如，获取恐惧和贪婪指数（一个衡量市场情绪的 0-100 评分）。

我们将使用：https://api.alternative.me/fng/ （Fear & Greed Index API）

### 实现

```python
# 文件: custom_data.py

import requests
from datetime import datetime, timedelta
from AlgorithmImports import *

class FearGreedData(BaseData):
    """
    自定义数据类：恐惧和贪婪指数
    
    从公共 API 获取 Fear & Greed Index。
    
    使用方法：
    1. GetSource() 返回 API 端点
    2. Reader() 解析 JSON 响应
    """
    
    def __init__(self):
        super().__init__()
        self.fear_greed_index = 0  # 0-100 评分
        self.classification = ""   # "Extreme Fear" 等
    
    def GetSource(self, config, date, isLive):
        """
        返回 API 端点。
        
        QuantConnect 支持 HTTP 数据源。
        LEAN 会自动处理 HTTP 请求和响应。
        """
        
        # 构造 API 调用
        # 这个 API 返回最新的恐惧指数数据
        url = "https://api.alternative.me/fng/?limit=1&format=json"
        
        # 如果是实盘，API 可能提供实时数据
        # 如果是回测，我们可能需要历史数据
        # 这里为简化起见，我们总是获取最新的
        
        source = SubscriptionDataSource(
            source=url,
            format=DataFeedFormat.Json  # JSON 格式，而不是 CSV
        )
        
        return source
    
    def Reader(self, config, line, date, isLive):
        """
        解析 JSON 响应。
        
        Fear & Greed API 返回 JSON 格式的数据。
        """
        
        if not line:
            return None
        
        try:
            import json
            
            # 解析 JSON
            # API 返回：{"data": [{"value": "75", "value_classification": "Greed", ...}]}
            data = json.loads(line)
            
            # 检查响应结构
            if 'data' not in data or len(data['data']) == 0:
                return None
            
            latest = data['data'][0]
            
            # 创建数据对象
            obj = FearGreedData()
            obj.Symbol = config.Symbol
            obj.Time = datetime.utcnow()
            obj.fear_greed_index = int(latest.get('value', 0))
            obj.classification = latest.get('value_classification', 'Unknown')
            obj.Value = obj.fear_greed_index
            
            return obj
        
        except Exception as e:
            return None
```

### 在策略中使用

```python
class FearGreedStrategy(QCAlgorithm):
    """
    基于恐惧指数的宏观对冲策略。
    
    逻辑：
    - 恐惧指数 < 30：极端恐惧，买入风险资产（VTI）
    - 恐惧指数 > 70：极端贪婪，卖出或转向安全资产（TLT）
    """
    
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 添加风险资产和安全资产
        self.risky = self.AddEquity("VTI", Resolution.Daily).Symbol  # 全市场 ETF
        self.safe = self.AddEquity("TLT", Resolution.Daily).Symbol   # 债券 ETF
        
        # 添加自定义恐惧指数数据
        # 注意：我们用虚拟代码 "FEARGREED"，实际数据来自 API
        self.AddData(
            dataType=FearGreedData,
            symbol="FEARGREED",
            resolution=Resolution.Daily
        )
        
        # 当前配置（风险/安全）
        self.current_position = "neutral"
    
    def OnData(self, data):
        """
        每日检查恐惧指数，调整组合配置。
        """
        
        if FearGreedData not in data:
            return
        
        fg_obj = data[FearGreedData]
        index = fg_obj.fear_greed_index
        
        # 简单的状态机
        if index < 30 and self.current_position != "risky":
            # 极端恐惧 → 买入风险资产，卖出安全资产
            self.SetHoldings(self.risky, 1.0)
            self.Liquidate(self.safe)
            self.current_position = "risky"
            self.Debug(f"Extreme Fear ({index}): Moving to risky assets")
        
        elif index > 70 and self.current_position != "safe":
            # 极端贪婪 → 买入安全资产，卖出风险资产
            self.SetHoldings(self.safe, 1.0)
            self.Liquidate(self.risky)
            self.current_position = "safe"
            self.Debug(f"Extreme Greed ({index}): Moving to safe assets")
        
        elif 30 <= index <= 70 and self.current_position != "balanced":
            # 中等 → 平衡配置
            self.SetHoldings(self.risky, 0.5)
            self.SetHoldings(self.safe, 0.5)
            self.current_position = "balanced"
            self.Debug(f"Neutral ({index}): Balanced portfolio")
```

### 处理 API 速率限制

如果你的 API 有速率限制，在 Reader() 中添加缓存逻辑：

```python
class CachedFearGreedData(BaseData):
    
    # 类级别缓存
    _cache = {}
    _cache_time = None
    _cache_ttl = timedelta(hours=1)  # 缓存 1 小时
    
    def Reader(self, config, line, date, isLive):
        now = datetime.utcnow()
        
        # 如果缓存有效，使用缓存值
        if (self._cache_time is not None and 
            now - self._cache_time < self._cache_ttl and
            'value' in self._cache):
            
            obj = CachedFearGreedData()
            obj.Symbol = config.Symbol
            obj.Time = now
            obj.Value = self._cache['value']
            obj.fear_greed_index = self._cache['value']
            return obj
        
        # 否则，获取新数据
        # ... 调用 API ...
        # 更新缓存
        # self._cache = {...}
        # self._cache_time = now
```

---

## Lab 3: 自定义 Universe Selection 数据源

### 场景

你维护了一个排名列表，每日更新，显示 500 只股票按 "质量分数" 排序。你想根据这个排名选择宇宙（要交易的股票集合）。

例如：只交易排名前 50 名的股票。

### 步骤 1：定义 RankingData 类

```python
# 文件: custom_data.py

from AlgorithmImports import *
from datetime import datetime

class StockRankingData(BaseData):
    """
    自定义数据类：股票排名
    
    数据格式（CSV）：
    Rank,Symbol,QualityScore,Sector
    1,AAPL,0.95,Technology
    2,MSFT,0.94,Technology
    3,JNJ,0.92,Healthcare
    ...
    """
    
    def __init__(self):
        super().__init__()
        self.rank = 0
        self.quality_score = 0.0
        self.sector = ""
    
    def GetSource(self, config, date, isLive):
        # 每日排名文件
        # 假设文件名包含日期
        filename = f"ranking_{date.strftime('%Y%m%d')}.csv"
        path = f"custom/rankings/{filename}"
        
        source = SubscriptionDataSource(
            source=path,
            format=DataFeedFormat.Csv
        )
        
        return source
    
    def Reader(self, config, line, date, isLive):
        if not line or line.startswith('Rank'):
            return None
        
        try:
            parts = line.split(',')
            
            obj = StockRankingData()
            obj.Symbol = parts[1].strip()  # Symbol
            obj.Time = datetime.combine(date, datetime.min.time())
            obj.rank = int(parts[0].strip())
            obj.quality_score = float(parts[2].strip())
            obj.sector = parts[3].strip()
            obj.Value = obj.quality_score
            
            return obj
        except:
            return None
```

### 步骤 2：在策略中使用 Universe Selection

```python
class DynamicUniverseStrategy(QCAlgorithm):
    """
    基于自定义排名数据动态选择股票宇宙。
    
    逻辑：
    - 每日加载排名数据
    - 选择前 50 名股票交易
    - 交易均等权重组合
    """
    
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 订阅排名数据
        self.AddData(
            dataType=StockRankingData,
            symbol="RANKINGS",  # 虚拟代码
            resolution=Resolution.Daily
        )
        
        # 跟踪当前持仓
        self.current_universe = []
        self.max_universe_size = 50
    
    def OnData(self, data):
        if StockRankingData not in data:
            return
        
        ranking_data = data[StockRankingData]
        
        # 排名数据包含所有股票信息
        # 通常你会在 Initialize() 或专门的方法中处理
        # 这里简化演示
        
        # 实际上，对于 Universe Selection，
        # 更好的方法是使用自定义 Filter
        # 见下面的高级示例
```

### 步骤 3：高级方式 - 自定义 Universe Filter

```python
class RankingBasedUniverseFilter(BaseData):
    """
    专为 Universe Selection 设计的数据源。
    """
    
    def __init__(self):
        super().__init__()
        self.rankings_dict = {}  # {Symbol: rank}
    
    def GetSource(self, config, date, isLive):
        filename = f"ranking_{date.strftime('%Y%m%d')}.csv"
        path = f"custom/rankings/{filename}"
        
        source = SubscriptionDataSource(
            source=path,
            format=DataFeedFormat.Csv
        )
        return source
    
    def Reader(self, config, line, date, isLive):
        if not line or line.startswith('Rank'):
            return None
        
        parts = line.split(',')
        symbol = parts[1].strip()
        rank = int(parts[0].strip())
        
        # 存储排名信息
        if not hasattr(RankingBasedUniverseFilter, '_rankings_data'):
            RankingBasedUniverseFilter._rankings_data = {}
        
        RankingBasedUniverseFilter._rankings_data[symbol] = rank
        
        # 返回单个数据点
        obj = RankingBasedUniverseFilter()
        obj.Symbol = symbol
        obj.Time = datetime.combine(date, datetime.min.time())
        obj.rank = rank
        obj.Value = rank
        
        return obj


class DynamicRankingStrategy(QCAlgorithm):
    """
    使用排名数据进行动态 Universe Selection。
    """
    
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 设置 Universe：每日根据排名更新
        self.AddUniverseSelection(
            self.SelectRankedStocks
        )
        
        # 订阅排名数据
        self.AddData(
            dataType=RankingBasedUniverseFilter,
            symbol="RANKINGS",
            resolution=Resolution.Daily
        )
        
        self.rankings = {}
    
    def SelectRankedStocks(self, fundamental_universe):
        """
        Universe Selection 回调函数。
        
        返回要交易的股票列表。
        """
        
        # 这里你会使用 self.rankings 来选择前 50 名
        # 为简化，返回一个静态列表
        # 实际使用中会动态根据最新排名更新
        
        ranked_symbols = sorted(
            self.rankings.items(),
            key=lambda x: x[1]  # 按排名排序
        )[:50]
        
        return [Symbol.Create(sym, SecurityType.Equity, "USA") 
                for sym, rank in ranked_symbols]
    
    def OnData(self, data):
        if RankingBasedUniverseFilter in data:
            # 更新排名缓存
            for obj in data[RankingBasedUniverseFilter]:
                self.rankings[obj.Symbol.Value] = obj.rank
```

---

## 数据格式规范

### CSV 格式要求

自定义 CSV 数据源应遵循：

```
日期,时间,字段1,字段2,...
YYYY-MM-DD,HH:MM,value1,value2,...
2023-01-01,16:00,100.5,50
2023-01-02,16:00,101.2,51
```

**时间戳规则**：
- **日期格式**：`YYYY-MM-DD`（必须）
- **时间格式**：`HH:MM` 或 `HH:MM:SS`（可选，默认 00:00）
- **时区**：默认为美国东部时间 (ET)。如果数据是其他时区，需要在 Reader() 中转换

### 时区处理

```python
from pytz import timezone

def Reader(self, config, line, date, isLive):
    parts = line.split(',')
    
    # 假设数据是 UTC
    utc_tz = timezone('UTC')
    et_tz = timezone('US/Eastern')
    
    # 解析时间
    dt = datetime.strptime(f"{date.date()} {parts[1]}", "%Y-%m-%d %H:%M")
    dt_utc = utc_tz.localize(dt)
    
    # 转换为东部时间（QuantConnect 使用）
    dt_et = dt_utc.astimezone(et_tz)
    
    obj = MyData()
    obj.Time = dt_et
    return obj
```

### 处理缺失数据

**方案 1**：使用 NaN 或空值

```python
def Reader(self, config, line, date, isLive):
    parts = line.split(',')
    
    # 如果字段为空或 "NaN"，跳过此行或使用默认值
    if parts[2].strip() in ['', 'NaN', 'NA']:
        return None  # 跳过
    
    value = float(parts[2])
    # ...
```

**方案 2**：前向填充（使用前一个有效值）

```python
class DataWithForwardFill(BaseData):
    
    _last_value = None
    
    def Reader(self, config, line, date, isLive):
        parts = line.split(',')
        
        if parts[2].strip() in ['', 'NaN']:
            # 使用前一个值
            if self._last_value is None:
                return None
            value = self._last_value
        else:
            value = float(parts[2])
            self._last_value = value
        
        obj = DataWithForwardFill()
        obj.Value = value
        return obj
```

### 文件组织最佳实践

```
data/
├── custom/
│   ├── sentiment/
│   │   ├── AAPL_sentiment.csv
│   │   ├── MSFT_sentiment.csv
│   │   └── ...
│   ├── rankings/
│   │   ├── ranking_20230101.csv
│   │   ├── ranking_20230102.csv
│   │   └── ...
│   └── alternative/
│       ├── satellite_imagery_counts.csv
│       └── ...
```

---

## 测试与验证

### 验证数据完整性

在 Research 笔记本中：

```python
# 导入自定义数据类
from custom_data import SentimentData

# 获取历史数据
history = qb.GetFundamentalData(["AAPL"], date(2023, 1, 1), date(2023, 3, 31))

# 或者直接从 CSV 加载
import pandas as pd
df = pd.read_csv("data/custom/sentiment/AAPL_sentiment.csv")

print(df.head())
print(df.describe())
print(df.isnull().sum())  # 检查缺失值
```

### 数据对齐检查

确保自定义数据与市场数据的时间对齐：

```python
class AlignmentCheckStrategy(QCAlgorithm):
    
    def OnData(self, data):
        # 市场数据
        if self.symbol in data:
            market_time = data[self.symbol].Time
            market_price = data[self.symbol].Close
        
        # 自定义数据
        if SentimentData in data:
            sentiment_time = data[SentimentData].Time
            sentiment = data[SentimentData].sentiment_score
            
            # 检查时间是否匹配
            if market_time != sentiment_time:
                self.Debug(f"TIME MISMATCH: Market {market_time}, Sentiment {sentiment_time}")
```

### 极端值检查

```python
def Reader(self, config, line, date, isLive):
    parts = line.split(',')
    value = float(parts[2])
    
    # 检查极端值
    if value < -1000 or value > 10000:
        self.Debug(f"Extreme value detected: {value}")
        return None  # 跳过
    
    obj = MyData()
    obj.Value = value
    return obj
```

### 回测 vs 实盘数据源

如果你的数据源在回测和实盘中不同：

```python
def GetSource(self, config, date, isLive):
    if isLive:
        # 实盘：使用 API
        url = "https://api.example.com/latest"
    else:
        # 回测：使用历史 CSV
        url = f"data/custom/historical/{date.strftime('%Y%m%d')}.csv"
    
    return SubscriptionDataSource(source=url, format=DataFeedFormat.Csv)
```

---

## 从数据源到交易信号的完整管线

### 端到端示例

```python
# 文件：end_to_end_strategy.py

from AlgorithmImports import *
from custom_data import SentimentData, StockRankingData
from datetime import datetime

class EndToEndStrategy(QCAlgorithm):
    """
    完整的数据 → 指标 → 信号 → 交易 管线。
    
    流程：
    1. 加载两个自定义数据源（情绪 + 排名）
    2. 基于排名选择宇宙
    3. 基于情绪生成交易信号
    4. 执行交易
    """
    
    def Initialize(self):
        self.SetStartDate(2023, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        
        # 订阅两个自定义数据源
        self.AddData(SentimentData, "SENTIMENT", Resolution.Daily)
        self.AddData(StockRankingData, "RANKINGS", Resolution.Daily)
        
        # 初始化排名缓存
        self.rankings = {}
        self.top_stocks = []
    
    def OnData(self, data):
        """
        综合处理多个数据源。
        """
        
        # ========== 更新排名信息 ==========
        if StockRankingData in data:
            ranking_obj = data[StockRankingData]
            self.rankings[ranking_obj.Symbol] = ranking_obj.rank
            
            # 选择前 20 名
            self.top_stocks = sorted(
                self.rankings.items(),
                key=lambda x: x[1]
            )[:20]
        
        # ========== 根据情绪生成信号 ==========
        if SentimentData in data:
            sentiment_obj = data[SentimentData]
            sentiment = sentiment_obj.sentiment_score
            
            # 只对排名前 20 的股票交易
            for symbol, rank in self.top_stocks:
                # 获取市场价格
                if symbol not in data:
                    continue
                
                price = data[symbol].Close
                
                # 交易逻辑
                if sentiment > 0.75:  # 强烈看涨情绪
                    # 买入排名股票
                    self.SetHoldings(symbol, 1.0 / len(self.top_stocks))
                    self.Debug(
                        f"BUY {symbol} (rank {rank}): "
                        f"sentiment={sentiment:.2f}, price={price:.2f}"
                    )
                
                elif sentiment < 0.25:  # 强烈看跌情绪
                    # 卖出
                    self.Liquidate(symbol)
                    self.Debug(f"SELL {symbol}: sentiment={sentiment:.2f}")
    
    def OnEndOfDay(self, symbol):
        """
        每天结束时的检查。
        """
        # 可选：记录 Portfolio 状态
        self.Plot(
            "Holdings",
            "Count",
            len(self.Portfolio.Positions)
        )
```

### 数据验证清单

在运行完整策略前，检查：

- ✅ CSV 文件格式正确（日期、时间、值）
- ✅ 时间戳和时区设置正确
- ✅ 缺失数据处理得当
- ✅ 自定义数据与市场数据的频率匹配
- ✅ Reader() 方法处理所有可能的异常
- ✅ GetSource() 返回有效的路径或 URL
- ✅ 回测中使用的文件确实存在于 `data/` 文件夹

---

## 常见问题

### Q1: 自定义数据如何与市场数据同步？

**A**: LEAN 会根据分辨率自动同步。如果市场数据和自定义数据都设置为 `Resolution.Daily`，它们会在同一 TimeSlice 中到达。

但要注意：

- 自定义数据可能没有市场数据那样频繁
- 使用 `if DataType in data:` 检查数据是否存在

```python
def OnData(self, data):
    # 总是检查数据存在性
    if self.symbol in data and SentimentData in data:
        market_price = data[self.symbol].Close
        sentiment = data[SentimentData].sentiment_score
```

### Q2: 自定义数据可以有不同的分辨率吗？

**A**: 可以。例如，市场数据分钟级，自定义数据日级：

```python
self.AddEquity("AAPL", Resolution.Minute)
self.AddData(SentimentData, "AAPL", Resolution.Daily)
```

但在这种情况下，你需要检查何时有新的自定义数据点：

```python
def OnData(self, data):
    # 每分钟调用
    if SentimentData in data:
        # 只在新情绪数据到达时执行（每天一次）
        sentiment = data[SentimentData].sentiment_score
```

### Q3: 如何处理自定义数据的延迟？

**A**: 在实盘中，数据可能延迟（例如，API 响应慢）。策略应该：

1. 检查时间戳是否过期

```python
def OnData(self, data):
    if SentimentData in data:
        sentiment_obj = data[SentimentData]
        age = self.Time - sentiment_obj.Time
        
        if age > timedelta(days=2):
            # 数据过于陈旧，不使用
            return
```

2. 使用缓存的最后已知值

```python
class SafeStrategyWithCaching(QCAlgorithm):
    
    def Initialize(self):
        self.last_sentiment = None
    
    def OnData(self, data):
        if SentimentData in data:
            self.last_sentiment = data[SentimentData].sentiment_score
        
        # 即使没有新数据，也使用缓存值
        if self.last_sentiment is not None:
            # 交易...
```

### Q4: 自定义数据的成本是什么？

**A**: 在 QuantConnect 上，自定义数据订阅有额外费用（取决于数据源的许可证）。内置市场数据和社区数据通常免费。

### Q5: 我可以在实盘中使用自定义数据吗？

**A**: 完全可以。但要确保：

1. 数据源在实盘中有效（API 可用、文件可访问）
2. GetSource() 返回的 `isLive=True` 时的正确端点
3. 处理 API 速率限制和超时

```python
def GetSource(self, config, date, isLive):
    if isLive:
        url = "https://api.example.com/realtime"
    else:
        url = f"data/custom/historical/{date.strftime('%Y%m%d')}.csv"
    
    return SubscriptionDataSource(source=url, format=DataFeedFormat.Csv)
```

### Q6: 自定义数据可以触发算法调整吗？

**A**: 可以。例如，根据自定义风险评分调整杠杆：

```python
class DynamicLeverageStrategy(QCAlgorithm):
    
    def OnData(self, data):
        if RiskScoreData in data:
            risk = data[RiskScoreData].score  # 0-100
            
            # 根据风险调整杠杆
            leverage = max(1, (100 - risk) / 50)  # 风险低 → 杠杆高
            
            for symbol in self.portfolio:
                self.SetHoldings(symbol, target_leverage=leverage)
```

### Q7: 如何对自定义数据进行单元测试？

**A**: 创建模拟数据并测试 Reader() 方法：

```python
import unittest

class TestSentimentDataReader(unittest.TestCase):
    
    def setUp(self):
        self.data = SentimentData()
    
    def test_reader_valid_line(self):
        line = "2023-01-01,16:00,0.72,0.95"
        config = MagicMock()
        config.Symbol = Symbol.Create("AAPL", SecurityType.Equity, "USA")
        
        result = self.data.Reader(config, line, datetime(2023, 1, 1), False)
        
        self.assertIsNotNone(result)
        self.assertEqual(result.sentiment_score, 0.72)
        self.assertEqual(result.confidence, 0.95)
    
    def test_reader_invalid_line(self):
        line = "invalid,data,here"
        config = MagicMock()
        
        result = self.data.Reader(config, line, datetime(2023, 1, 1), False)
        
        self.assertIsNone(result)
```

### Q8: 自定义数据会影响回测速度吗？

**A**: 可能会。每次加载、解析和处理自定义数据都有成本。优化建议：

1. 使用缓存来减少 API 调用
2. 在本地存储数据（CSV 比 API 快）
3. 减少不必要的数据字段
4. 避免在 Reader() 中进行复杂计算

---

## 总结

自定义数据源是 LEAN 最强大的扩展点之一。通过这些 Lab，你已经学会了：

1. ✅ 实现 BaseData 子类，定义数据格式
2. ✅ 处理 CSV 和 JSON 数据源
3. ✅ 集成 REST API 数据
4. ✅ 构建动态 Universe Selection
5. ✅ 验证和调试自定义数据
6. ✅ 从多个数据源构建完整的交易管线

下一步是更高级的二次开发：**券商插件**，允许你针对特定券商定制订单执行和结算逻辑。

---

## 导航

**[上一篇: 二开入门：自定义指标](./06-custom-indicator.md)** | **[下一篇: 二开高阶：券商插件开发](./08-custom-brokerage.md)**

**[架构参考：数据管线](../quantconnect-deep-dive/03-data-pipeline.md)**
