# 研究环境与数据探索

> **定位**：从策略编写的 "工厂模式" 转向数据探索的 "实验室模式"。
>
> Research 环境是量化交易员的私人实验室，在这里你可以快速原型验证、可视化分析、不受约束地探索数据和交易想法。

**所需时间**：60 分钟
**前置条件**：完成文档 04，本地开发环境已就绪
**完成后**：能够独立在 Jupyter Notebook 中进行数据分析和策略验证

---

## Research 环境是什么？

### 对标：其他量化平台

如果你用过 **Backtrader** 或 **VectorBT**，Research 环境类似于 Jupyter Notebook；如果你用过 **Python 数据科学工作流**，Research 环境就是这样——但专为量化交易优化。

### QuantConnect 中的 Research

Research 在 QuantConnect 中有两种形态：

| 环境 | 访问方式 | 用途 | 优点 | 缺点 |
|------|--------|------|------|------|
| **云端 Research** | web.quantconnect.com 网页 | 快速原型、分享 notebook | 无需本地配置、协作方便 | 需网络、排队等待 |
| **本地 Research** | `lean research` 命令 | 完整离线开发、数据完全私有 | 速度快、隐私高、可离线 | 需本地配置 |

对于 2-3 人创业团队，**本地 Research 最适合**：完全离线、速度快、无成本限制。

---

## 云端 Research 环境速览

### 1. 访问云端 Research

1. 登录 [web.quantconnect.com](https://www.quantconnect.com)
2. 左侧导航栏找到 **"Research"**（或点击 **"Lab"** 按钮）
3. 点击 **"+ New Notebook"**

### 2. Research Notebook 基础

你会看到一个 Jupyter 环境，已经预装好所有 QuantConnect 库。顶部是工具栏，左侧是文件浏览器。

### 3. QuantBook 对象

云端 Research 中的核心对象是 **QuantBook**，它类似于策略中的 **QCAlgorithm**，但专为数据探索优化：

```python
from AlgorithmImports import *

# 创建 QuantBook 实例
qb = QuantBook()

# 订阅数据
qb.AddEquity("SPY", Resolution.Daily)

# 获取历史数据
history = qb.History("SPY", 500, Resolution.Daily)

# 查看数据
print(history.head())
```

输出示例：
```
                  open      high       low     close   volume
time
2020-01-02  300.14    300.63   299.59   300.63    23698700
2020-01-03  301.15    302.14   301.08   301.93    19877334
...
```

### 4. 基本操作

```python
# 添加资产
qb.AddEquity("QQQ", Resolution.Daily)
qb.AddCrypto("BTCUSD", Resolution.Hourly)

# 获取实时价格
spy_price = qb.GetLastPrice("SPY")
print(f"SPY 当前价格: ${spy_price}")

# 获取历史数据（日线，最近 100 天）
spy_history = qb.History("SPY", 100, Resolution.Daily)

# 获取特定日期范围
spy_history = qb.History(["SPY"], "2020-01-01", "2023-12-31", Resolution.Daily)
```

---

## 本地 Research 环境（推荐）

### Step 1：启动本地 Jupyter

假设你在项目根目录（文档 04 中创建的 `my-quant-project`）：

```bash
lean research
```

**第一次运行会下载 Docker 镜像，耗时 2-5 分钟。**

完成后，输出示例：
```
Starting research environment...
Jupyter Lab is running at: http://localhost:8888/?token=abc123xyz
```

浏览器会自动打开 Jupyter Lab 页面。如果没有，手动访问 `http://localhost:8888`。

### Step 2：创建 Notebook

1. 在 Jupyter Lab 中，右键点击 **"Python 3"** 创建新 notebook
2. 命名为 `research.ipynb`
3. 在第一个 cell 中导入库

```python
from AlgorithmImports import *
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 设置图表样式
sns.set_style("whitegrid")
plt.rcParams['figure.figsize'] = (12, 6)
```

### Step 3：创建 QuantBook

```python
# 创建 QuantBook 实例（本地模式下也是同一对象）
qb = QuantBook()

# 添加资产
qb.AddEquity("SPY", Resolution.Daily)

print("QuantBook 已初始化")
```

### Step 4：本地与云端的关键区别

| 功能 | 云端 | 本地 | 说明 |
|------|------|------|------|
| 数据来源 | QuantConnect 服务器 | 本地 `data/` 文件夹 | 本地需提前下载数据 |
| 执行速度 | 较慢（网络延迟） | 快 | 本地 I/O 优化 |
| 隐私 | 数据在云端 | 完全本地 | 本地保留所有分析结果 |
| 离线使用 | 不支持 | 支持 | 本地无需网络 |
| Notebook 保存 | 云端仓库 | 项目目录 | 本地更灵活 |

**建议**：用本地 Research 做所有数据分析和策略验证，保证隐私和速度。

---

## 数据探索实战 Lab 1：股票数据分析

### 目标

使用 QuantConnect 的历史数据 API，获取 5 年 SPY 日线数据，进行基础统计和可视化分析。

### 完整代码

在 Jupyter Notebook 中逐个运行以下 cell：

#### Cell 1：初始化和数据获取

```python
from AlgorithmImports import *
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats

# 初始化 QuantBook
qb = QuantBook()
qb.AddEquity("SPY", Resolution.Daily)

# 获取 5 年数据（从 2019-01-01 到 2024-01-01）
symbol = "SPY"
history = qb.History(symbol, 5*252, Resolution.Daily)  # 252 ≈ 每年交易日

print(f"获取数据行数: {len(history)}")
print(f"数据时间范围: {history.index[0]} 至 {history.index[-1]}")
print(history.head())
```

**预期输出：**
```
获取数据行数: 1260
数据时间范围: 2019-01-02 09:30:00 至 2024-01-01 16:00:00
                         open      high       low     close    volume
time
2019-01-02 09:30:00  253.62    256.93    253.62   256.93   57617700
2019-01-03 09:30:00  256.67    258.26    256.10   257.92   47887500
...
```

#### Cell 2：转换为 DataFrame 并计算收益

```python
# 转换为更易操作的 DataFrame 格式
df = history[['close']].copy()
df.columns = ['price']

# 计算日收益率
df['daily_return'] = df['price'].pct_change()

# 计算累计收益率（投资 $1 的增长）
df['cumulative_return'] = (1 + df['daily_return']).cumprod() - 1

# 计算 20 日滚动平均收益
df['rolling_mean_return'] = df['daily_return'].rolling(window=20).mean()

# 计算 20 日滚动波动率
df['rolling_volatility'] = df['daily_return'].rolling(window=20).std()

print("数据预处理完成")
print(df.head(25))
print("\n基础统计:")
print(df['daily_return'].describe())
```

**预期输出：**
```
基础统计:
count    1260.000000
mean        0.000651
std         0.010823
min        -0.067990
25%        -0.005773
50%         0.000652
75%         0.006959
max         0.085476
Name: daily_return, dtype: float64
```

#### Cell 3：绘制价格与累计收益

```python
fig, axes = plt.subplots(2, 1, figsize=(14, 8))

# 子图 1：价格走势
axes[0].plot(df.index, df['price'], linewidth=1.5, color='blue', label='SPY Price')
axes[0].set_title('SPY 5 年日线价格', fontsize=14, fontweight='bold')
axes[0].set_ylabel('Price ($)')
axes[0].legend()
axes[0].grid(alpha=0.3)

# 子图 2：累计收益
axes[1].plot(df.index, df['cumulative_return'] * 100, linewidth=1.5, color='green', label='Cumulative Return')
axes[1].axhline(y=0, color='red', linestyle='--', alpha=0.5)
axes[1].set_title('SPY 5 年累计收益率', fontsize=14, fontweight='bold')
axes[1].set_ylabel('Return (%)')
axes[1].set_xlabel('Date')
axes[1].legend()
axes[1].grid(alpha=0.3)

plt.tight_layout()
plt.show()

print(f"总收益率: {df['cumulative_return'].iloc[-1] * 100:.2f}%")
```

**预期输出：** 一个 2 行的图表，展示价格上升趋势和对应的累计收益。

#### Cell 4：收益分布分析

```python
# 日收益率直方图
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# 直方图
axes[0].hist(df['daily_return'].dropna(), bins=100, color='blue', alpha=0.7, edgecolor='black')
axes[0].axvline(df['daily_return'].mean(), color='red', linestyle='--', linewidth=2, label=f"Mean: {df['daily_return'].mean():.4f}")
axes[0].set_title('日收益率分布', fontsize=12, fontweight='bold')
axes[0].set_xlabel('Daily Return')
axes[0].set_ylabel('Frequency')
axes[0].legend()
axes[0].grid(alpha=0.3)

# Q-Q 图（检验正态分布）
stats.probplot(df['daily_return'].dropna(), dist="norm", plot=axes[1])
axes[1].set_title('Q-Q 图：收益率正态性检验', fontsize=12, fontweight='bold')
axes[1].grid(alpha=0.3)

plt.tight_layout()
plt.show()

# 计算偏度和峰度
skewness = df['daily_return'].skew()
kurtosis = df['daily_return'].kurtosis()
print(f"偏度（Skewness）: {skewness:.4f}  # 负值表示左尾较长（崩盘风险高）")
print(f"峰度（Kurtosis）: {kurtosis:.4f}   # 高峰度表示极端事件更频繁")
```

#### Cell 5：滚动波动率和回撤分析

```python
# 计算最大回撤
def calculate_max_drawdown(returns):
    """计算累计最大回撤"""
    cumsum = (1 + returns).cumprod()
    cummax = cumsum.expanding().max()
    drawdown = (cumsum - cummax) / cummax
    return drawdown.min()

max_dd = calculate_max_drawdown(df['daily_return'].dropna())
print(f"最大回撤: {max_dd * 100:.2f}%")

# 绘制波动率和回撤
fig, axes = plt.subplots(2, 1, figsize=(14, 8))

# 子图 1：滚动波动率
axes[0].plot(df.index, df['rolling_volatility'] * 100, linewidth=1, color='orange', label='20D Rolling Volatility')
axes[0].fill_between(df.index, 0, df['rolling_volatility'] * 100, alpha=0.3, color='orange')
axes[0].set_title('20 日滚动波动率', fontsize=12, fontweight='bold')
axes[0].set_ylabel('Volatility (%)')
axes[0].legend()
axes[0].grid(alpha=0.3)

# 子图 2：回撤曲线
running_max = (1 + df['daily_return']).cumprod().expanding().max()
drawdown = ((1 + df['daily_return']).cumprod() - running_max) / running_max
axes[1].fill_between(df.index, 0, drawdown * 100, alpha=0.5, color='red', label='Drawdown')
axes[1].set_title('账户回撤曲线', fontsize=12, fontweight='bold')
axes[1].set_ylabel('Drawdown (%)')
axes[1].set_xlabel('Date')
axes[1].legend()
axes[1].grid(alpha=0.3)

plt.tight_layout()
plt.show()
```

#### Cell 6：计算 Sharpe Ratio

```python
# Sharpe Ratio = (平均年收益 - 无风险收益) / 年波动率
annual_return = df['daily_return'].mean() * 252  # 252 = 一年交易日数
annual_volatility = df['daily_return'].std() * np.sqrt(252)
risk_free_rate = 0.04  # 假设无风险利率为 4%（美国 10 年期国债）

sharpe_ratio = (annual_return - risk_free_rate) / annual_volatility

print(f"年化收益率: {annual_return * 100:.2f}%")
print(f"年化波动率: {annual_volatility * 100:.2f}%")
print(f"Sharpe Ratio: {sharpe_ratio:.2f}")
print(f"\nSharpe Ratio 解释:")
print(f"  > 1.0: 较好")
print(f"  > 2.0: 很好")
print(f"  > 3.0: 优秀")
```

**预期输出：**
```
年化收益率: 12.43%
年化波动率: 18.76%
Sharpe Ratio: 0.45
```

---

## 数据探索实战 Lab 2：多资产相关性分析

### 目标

获取多个资产类别（股票、债券、黄金、加密货币）的历史数据，计算相关性矩阵，理解资产配置的多元化效果。

### 完整代码

#### Cell 1：获取多资产数据

```python
# 初始化 QuantBook
qb = QuantBook()

# 定义资产池
assets = {
    "SPY": "美国股票指数",
    "TLT": "美国长期国债 ETF",
    "GLD": "黄金 ETF",
    # "BTCUSD": "比特币"  # 如果需要加密货币，取消注释
}

# 添加资产到 QuantBook
for symbol in assets.keys():
    if symbol == "BTCUSD":
        qb.AddCrypto(symbol, Resolution.Daily)
    else:
        qb.AddEquity(symbol, Resolution.Daily)

# 获取 3 年历史数据
history = {}
for symbol in assets.keys():
    h = qb.History(symbol, 3*252, Resolution.Daily)
    history[symbol] = h['close']

# 转换为 DataFrame
price_data = pd.concat(history, axis=1)
price_data.columns = assets.keys()

print(f"数据范围: {price_data.index[0]} 至 {price_data.index[-1]}")
print(f"资产数: {len(assets)}")
print(price_data.head())
```

#### Cell 2：计算日收益率和相关性

```python
# 计算日收益率
returns = price_data.pct_change().dropna()

# 计算相关性矩阵
correlation_matrix = returns.corr()

print("相关性矩阵:")
print(correlation_matrix.round(3))

# 绘制相关性热力图
plt.figure(figsize=(10, 8))
sns.heatmap(correlation_matrix,
            annot=True,           # 显示数值
            fmt='.3f',            # 保留 3 位小数
            cmap='RdYlGn_r',      # 红黄绿渐变色
            center=0,             # 以 0 为中心
            square=True,          # 正方形格子
            cbar_kws={'label': 'Correlation'},
            linewidths=1)
plt.title('多资产相关性矩阵（3 年日线）', fontsize=14, fontweight='bold')
plt.tight_layout()
plt.show()
```

**预期输出：**
```
相关性矩阵:
         SPY    TLT    GLD
SPY    1.000 -0.234  0.156
TLT   -0.234  1.000 -0.089
GLD    0.156 -0.089  1.000
```

#### Cell 3：多元化收益分析

```python
# 3 资产等权投资组合（33% 各一个）
portfolio_weight = pd.Series({
    'SPY': 0.33,
    'TLT': 0.33,
    'GLD': 0.34
})

# 计算投资组合日收益率
portfolio_daily_return = (returns * portfolio_weight).sum(axis=1)

# 计算投资组合的年化收益和波动率
portfolio_annual_return = portfolio_daily_return.mean() * 252
portfolio_annual_vol = portfolio_daily_return.std() * np.sqrt(252)

# 对比单一资产
print("投资组合 vs 单资产对比:")
print("-" * 60)
for symbol in assets.keys():
    single_annual_return = returns[symbol].mean() * 252
    single_annual_vol = returns[symbol].std() * np.sqrt(252)
    print(f"{symbol:<8} 年收益: {single_annual_return*100:>7.2f}%  年波动: {single_annual_vol*100:>6.2f}%")

print("-" * 60)
print(f"{'组合':<8} 年收益: {portfolio_annual_return*100:>7.2f}%  年波动: {portfolio_annual_vol*100:>6.2f}%")

# 可视化对比
fig, ax = plt.subplots(figsize=(10, 6))

symbols_list = list(assets.keys()) + ['Portfolio']
returns_list = [returns[s].mean() * 252 * 100 for s in assets.keys()] + [portfolio_annual_return * 100]
vols_list = [returns[s].std() * np.sqrt(252) * 100 for s in assets.keys()] + [portfolio_annual_vol * 100]

colors = ['blue', 'orange', 'green', 'red']
ax.scatter(vols_list, returns_list, s=200, alpha=0.6, c=colors)

for i, symbol in enumerate(symbols_list):
    ax.annotate(symbol, (vols_list[i], returns_list[i]),
                xytext=(5, 5), textcoords='offset points', fontsize=10)

ax.set_xlabel('年化波动率 (%)', fontsize=12)
ax.set_ylabel('年化收益率 (%)', fontsize=12)
ax.set_title('风险-收益散点图', fontsize=14, fontweight='bold')
ax.grid(alpha=0.3)
plt.tight_layout()
plt.show()

print("\n结论：等权投资组合通常具有更低的波动率（风险分散）")
```

---

## 策略原型验证

### 背景

在正式编写策略代码前，你可以在 Research Notebook 中快速验证交易想法的有效性。这样做可以节省大量宝贵的回测计算资源。

### 例：RSI 低位反弹策略验证

#### Cell 1：假设验证

```python
# 获取 SPY 2 年数据
qb = QuantBook()
qb.AddEquity("SPY", Resolution.Daily)
history = qb.History("SPY", 2*252, Resolution.Daily)

df = history[['close']].copy()
df.columns = ['price']

# 计算 RSI（Relative Strength Index）
def calculate_rsi(prices, period=14):
    """计算 RSI 指标"""
    delta = prices.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

df['rsi'] = calculate_rsi(df['price'], period=14)
df['daily_return'] = df['price'].pct_change()

print("RSI 计算完成")
print(df[['price', 'rsi', 'daily_return']].head(20))
```

#### Cell 2：统计分析

```python
# 定义"低位"：RSI < 30
df['is_oversold'] = df['rsi'] < 30

# 计算"低位后的次日收益"
df['next_day_return'] = df['daily_return'].shift(-1)

# 筛选低位的交易日
oversold_data = df[df['is_oversold']].copy()

print(f"超卖日数（RSI < 30）: {len(oversold_data)}")
print(f"平均超卖次日收益: {oversold_data['next_day_return'].mean() * 100:.3f}%")
print(f"平均非超卖次日收益: {df[~df['is_oversold']]['next_day_return'].mean() * 100:.3f}%")

# 5 日累计收益
df['next_5day_return'] = df['daily_return'].rolling(5).sum().shift(-1)
oversold_5day = oversold_data['next_5day_return'].mean()
non_oversold_5day = df[~df['is_oversold']]['next_5day_return'].mean()

print(f"\n5 日累计收益对比:")
print(f"超卖后 5 日平均收益: {oversold_5day * 100:.2f}%")
print(f"非超卖 5 日平均收益: {non_oversold_5day * 100:.2f}%")
```

#### Cell 3：统计显著性检验

```python
from scipy.stats import ttest_ind

oversold_returns = oversold_data['next_day_return'].dropna()
non_oversold_returns = df[~df['is_oversold']]['next_day_return'].dropna()

# 独立样本 t 检验
t_stat, p_value = ttest_ind(oversold_returns, non_oversold_returns)

print(f"t 检验结果:")
print(f"  t 统计量: {t_stat:.4f}")
print(f"  p 值: {p_value:.4f}")
print(f"  结论: {'差异显著（p<0.05）' if p_value < 0.05 else '差异不显著（无法拒绝零假设）'}")
```

#### Cell 4：决策

基于以上分析，如果发现 "RSI 低位后次日收益显著高于平均"，你有信心将其升级为完整策略（在文档 04 中编写为 main.py）。

如果没有发现有意义的差异，说明这个想法可能不可行，节省了编码和回测的时间。

---

## 数据源与数据质量

### 可用数据类型

QuantConnect 本地提供以下数据：

| 资产类别 | 可用性 | 分辨率 | 时间范围 | 备注 |
|---------|--------|--------|---------|------|
| **美股** | ✅ 完整 | Tick, Minute, Hour, Daily | 2010+ | SPY, QQQ, IWM 等主流个股 |
| **期货** | ✅ 完整 | Minute, Hour, Daily | 2010+ | ES, NQ, CL, GC 等 |
| **期权** | ✅ 完整 | Daily（期权链） | 2016+ | 美股期权 |
| **外汇** | ✅ 完整 | Minute, Hour, Daily | 2010+ | EURUSD, GBPUSD 等 |
| **加密** | ✅ 完整 | Minute, Hour, Daily | 2015+ | BTC, ETH 等，多个交易所 |

### 数据质量检查

在数据分析前，检查数据完整性很重要：

```python
# 检查缺失值
missing_count = history.isnull().sum()
print(f"缺失值: {missing_count}")

# 检查时间连续性
index_diff = history.index.to_series().diff()
print(f"最大时间间隔: {index_diff.max()}")

# 检查异常值（例如价格跳变）
history['price_change_pct'] = history['close'].pct_change()
outliers = history[abs(history['price_change_pct']) > 0.1]  # 单日 > 10% 涨跌
print(f"异常波动日数: {len(outliers)}")
```

### 常见数据问题

1. **股票分割（Stock Splits）**
   - QuantConnect 数据已调整，无需人工处理

2. **分红（Dividends）**
   - 日线收益率数据已包含分红调整
   - 使用 `qb.History()` 获取的 OHLCV 数据已调整

3. **退市（Delistings）**
   - 查询退市股票会返回空数据
   - 建议使用 Universe Selection 处理

---

## 实用技巧与最佳实践

### 技巧 1：缓存历史数据加速迭代

第一次 `qb.History()` 调用可能较慢（从磁盘读取）。加个缓存可以加速开发：

```python
import pickle

# 首次调用：保存到文件
history = qb.History("SPY", 5*252, Resolution.Daily)
history.to_pickle('spy_5year.pkl')

# 之后的重新运行：直接读取
history = pd.read_pickle('spy_5year.pkl')
```

### 技巧 2：导出分析结果

将 Notebook 的分析结果导出为 CSV，便于团队共享：

```python
# 导出相关性矩阵
correlation_matrix.to_csv('correlation_matrix.csv')

# 导出统计摘要
summary_stats = df[['price', 'daily_return', 'rsi']].describe()
summary_stats.to_csv('summary_stats.csv')
```

### 技巧 3：并行计算多个策略参数

快速扫描参数空间：

```python
import itertools

# 定义参数组合
fast_periods = [10, 20, 30]
slow_periods = [50, 100, 150]

results = []

for fast, slow in itertools.product(fast_periods, slow_periods):
    if fast >= slow:  # 跳过无效组合
        continue

    # 计算此组合的指标
    fast_sma = df['price'].rolling(fast).mean()
    slow_sma = df['price'].rolling(slow).mean()

    # 简单信号统计（仅为示例）
    signal = (fast_sma > slow_sma).astype(int)
    signal_changes = signal.diff().abs().sum()

    results.append({
        'fast': fast,
        'slow': slow,
        'signal_changes': signal_changes
    })

results_df = pd.DataFrame(results)
print(results_df)
```

### 技巧 4：分享 Notebook

如果团队成员需要查看你的分析，导出为 HTML：

```python
# 在 Jupyter 中，菜单: File → Export As → Export as HTML

# 或命令行：
# jupyter nbconvert --to html research.ipynb
```

---

## 从 Notebook 升级为正式策略

### 工作流

```
Research Notebook         Main Strategy
─────────────────         ──────────────
  验证想法                  编写完整代码
  可视化分析                加入风控逻辑
  快速迭代                  参数化配置
        ↓
    想法确认无误
        ↓
    转化为 main.py
        ↓
    使用 lean backtest 验证
```

### 示例转化

**Notebook 中验证的交易规则：**
```python
# RSI < 30 时买入，RSI > 70 时卖出
buy_signal = df['rsi'] < 30
sell_signal = df['rsi'] > 70
```

**转化为正式策略代码（main.py）：**
```python
class RSIStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)
        self.AddEquity("SPY", Resolution.Daily)
        self.rsi = self.RSI("SPY", 14)

    def OnData(self, data):
        if not self.rsi.IsReady:
            return

        rsi_value = self.rsi.Current.Value

        if rsi_value < 30 and not self.Portfolio.Invested:
            self.SetHoldings("SPY", 1.0)
        elif rsi_value > 70 and self.Portfolio.Invested:
            self.Liquidate("SPY")
```

---

## 总结与检查清单

完成本篇后，你应该能够：

- ✅ 启动本地 Jupyter Research 环境
- ✅ 使用 QuantBook 获取和处理历史数据
- ✅ 进行基础统计分析（收益、波动率、Sharpe Ratio）
- ✅ 绘制专业的财务图表
- ✅ 计算相关性并理解多元化效果
- ✅ 在 Notebook 中快速验证交易想法
- ✅ 将 Notebook 分析升级为正式策略

### 常见问题

**Q：Notebook 中的分析结果与 lean backtest 的结果不一致？**
A：两个常见原因：
1. Notebook 使用简化的计算（如未考虑滑点、手续费）
2. 数据范围或参数不同。确保两者使用相同的时间范围和参数。

**Q：如何在 Notebook 中加入交易成本（手续费、滑点）？**
A：Notebook 通常用于快速探索，严格的交易成本建模留给正式策略。如需详细，可使用 `slippage_pct` 等参数手工调整。

**Q：Notebook 支持回测吗？**
A：不完全支持。Notebook 适合数据探索和信号验证。完整回测（包括订单执行、风控）应在正式策略中使用 `lean backtest` 完成。

---

**导航**
[上一篇: 本地开发环境搭建](./04-local-setup.md) | [下一篇: 二开入门：自定义指标](./06-custom-indicator.md)
