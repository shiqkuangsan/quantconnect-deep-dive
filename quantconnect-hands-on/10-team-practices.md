# 团队协作与工程化实践

> **前置要求**：已完成 01-09 篇，至少有一个策略运行 2+ 周。
> **目标受众**：2-3 人初创团队，准备从个人开发转向团队协作。
> **学习成果**：建立团队工程规范，实现高效的策略开发、测试和部署流程。

---

## 代码组织规范

### 项目目录结构

适合 2-3 人团队的标准结构：

```
quant-trading-repo/
├── README.md                           # 项目简介
├── .gitignore                          # Git 忽略配置
├── requirements.txt                    # Python 依赖
├── config.json                         # 全局配置（不提交敏感内容）
├── .env.example                        # 环境变量模板（安全示例）
│
├── strategies/                         # 策略代码
│   ├── momentum/                       # 动量策略
│   │   ├── __init__.py
│   │   ├── momentum_strategy.py         # 主策略代码
│   │   ├── config.json                 # 策略参数
│   │   ├── indicators.py               # 自定义指标
│   │   └── tests/
│   │       ├── test_momentum.py
│   │       └── test_indicators.py
│   ├── mean_reversion/                 # 均值回归策略
│   │   ├── __init__.py
│   │   ├── mean_reversion_strategy.py
│   │   ├── config.json
│   │   └── tests/
│   └── multi_factor/                   # 多因子策略
│       ├── __init__.py
│       ├── multi_factor_strategy.py
│       ├── config.json
│       └── tests/
│
├── indicators/                         # 共享指标库
│   ├── __init__.py
│   ├── technical.py                    # 技术指标（MA, RSI, MACD）
│   ├── statistical.py                  # 统计指标（Zscore, etc）
│   └── tests/
│
├── data/                               # 数据处理模块
│   ├── __init__.py
│   ├── data_loader.py                  # 数据加载器
│   ├── data_clean.py                   # 数据清洗
│   └── tests/
│
├── risk/                               # 风控模块（所有策略共用）
│   ├── __init__.py
│   ├── risk_manager.py                 # 风控管理器
│   ├── position_sizer.py               # 头寸规模计算
│   └── tests/
│
├── backtest/                           # 回测与分析
│   ├── __init__.py
│   ├── backtest_runner.py              # 回测执行器
│   ├── analyzer.py                     # 结果分析
│   └── reports/                        # 回测报告输出
│
├── deployment/                         # 部署脚本
│   ├── __init__.py
│   ├── deploy_paper.py                 # 部署到纸面账户
│   ├── deploy_live.py                  # 部署到实盘
│   ├── health_check.py                 # 健康检查
│   └── monitoring.py                   # 监控脚本
│
├── docs/                               # 文档
│   ├── README.md
│   ├── architecture.md                 # 架构设计
│   ├── strategy_template.md            # 策略开发指南
│   └── deployment.md                   # 部署指南
│
├── scripts/                            # 工具脚本
│   ├── init_project.sh                 # 项目初始化
│   ├── run_backtest.sh                 # 运行回测
│   ├── deploy.sh                       # 部署脚本
│   └── monitor.sh                      # 监控脚本
│
└── logs/                               # 日志目录（.gitignore）
    ├── backtest/
    ├── live/
    └── errors/
```

### 命名规范

#### 1. 文件和目录命名

- **策略文件**：snake_case，前缀为策略类型
  ```
  momentum_strategy.py
  mean_reversion_strategy.py
  multi_factor_strategy.py
  ```

- **配置文件**：全小写 + 下划线
  ```
  config.json
  local_config.json
  ```

- **测试文件**：test_ 前缀
  ```
  test_momentum_strategy.py
  test_risk_manager.py
  ```

#### 2. Python 类和函数命名

```python
# 类名：PascalCase
class MomentumStrategy(QCAlgorithm):
    pass

class RiskManager:
    pass

# 函数名：snake_case
def calculate_moving_average(prices, window):
    pass

def validate_order(order):
    pass

# 常量：UPPER_CASE
MAX_POSITION_SIZE = 0.1
RISK_FREE_RATE = 0.02
```

#### 3. 配置参数命名

```json
{
  "strategy": {
    "name": "momentum",
    "version": "1.0.0",
    "description": "Momentum-based trading strategy"
  },
  "parameters": {
    "momentum_lookback": 20,
    "entry_threshold": 0.02,
    "exit_threshold": 0.01
  },
  "risk": {
    "max_position_size": 0.1,
    "daily_loss_limit": 0.02,
    "stop_loss": 0.05
  },
  "portfolio": {
    "initial_cash": 100000,
    "base_currency": "USD"
  }
}
```

### .gitignore 配置

```gitignore
# 环境变量和密钥
.env
.env.local
.env.*.local
config.local.json

# 敏感文件
api_keys.txt
secrets.json
credentials/

# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
.vscode/
.idea/

# 数据和日志
data/raw/
logs/
*.log
backtest_results/
*.csv

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# QuantConnect 特定
lean-logs/
research/
```

---

## Git 工作流

### 分支策略：Git Flow

适合持续交付的流程：

```
main (生产分支) ← release-* ← develop ← feature-*
                       ↓
                    hotfix-*
```

#### 分支说明

| 分支 | 目的 | 生命周期 |
|------|------|---------|
| `main` | 生产环境，永远可部署 | 长期 |
| `develop` | 开发主分支，包含最新代码 | 长期 |
| `feature/*` | 新功能或改进 | 短期 |
| `release/*` | 准备发布版本 | 短期 |
| `hotfix/*` | 紧急生产环境修复 | 短期 |

#### 工作流程

**开发新功能**

```bash
# 1. 从 develop 创建 feature 分支
git checkout develop
git pull origin develop
git checkout -b feature/new-momentum-indicator

# 2. 在本地开发
# ... 编写代码，提交 commits ...

git add .
git commit -m "Add momentum indicator calculation"

# 3. 推送到远程
git push origin feature/new-momentum-indicator

# 4. 创建 Pull Request（在 GitHub / GitLab）
# - 标题：简洁说明
# - 描述：详细说明做了什么、为什么这样做
# - 关联 Issue（如果有）

# 5. 等待代码审查（Code Review）
# - 至少 1 人同意
# - CI 测试通过

# 6. Merge 到 develop
git checkout develop
git pull origin develop
git merge feature/new-momentum-indicator
git push origin develop

# 7. 删除 feature 分支
git push origin --delete feature/new-momentum-indicator
git branch -d feature/new-momentum-indicator
```

**发布新版本**

```bash
# 1. 从 develop 创建 release 分支
git checkout develop
git checkout -b release/1.1.0

# 2. 在 release 分支上只做 bugfix 和文档更新
# ... 修复发现的 bug ...

# 3. 更新版本号
echo "1.1.0" > VERSION.txt
git commit -m "Bump version to 1.1.0"

# 4. Merge 到 main
git checkout main
git pull origin main
git merge release/1.1.0
git tag -a v1.1.0 -m "Release version 1.1.0"
git push origin main --tags

# 5. 同步回 develop
git checkout develop
git merge release/1.1.0
git push origin develop

# 6. 删除 release 分支
git push origin --delete release/1.1.0
```

### Commit 消息规范

采用 [Conventional Commits](https://www.conventionalcommits.org/)：

```
<type>(<scope>): <subject>

<body>

<footer>
```

类型（type）：

```
feat:     新功能
fix:      修复 bug
docs:     文档更新
style:    代码格式（不影响功能）
refactor: 代码重构
test:     添加或修改测试
chore:    构建、依赖、配置文件更新
perf:     性能优化
```

**例子**

```bash
# 好的 commit
git commit -m "feat(momentum): add RSI-based entry signal

- Implement RSI(14) calculation
- Add entry condition: RSI > 70
- Add exit condition: RSI < 30
- Backtest shows +2% improvement"

# 不好的 commit
git commit -m "update code"
```

### Pull Request 审查清单

团队成员审查 PR 时的检查清单：

```markdown
## PR 审查清单

### 代码质量
- [ ] 代码遵循项目风格指南
- [ ] 没有明显的逻辑错误
- [ ] 边界情况都有考虑

### 风控与安全
- [ ] 没有硬编码的 API Key
- [ ] 风控参数在合理范围
- [ ] 没有危险的异常处理（如捕获所有异常）

### 策略逻辑
- [ ] 新增指标都经过论证
- [ ] 参数都有合理的默认值
- [ ] 没有"未来看"（data snooping）

### 测试与性能
- [ ] 单元测试覆盖率 > 80%
- [ ] 回测表现与说明相符
- [ ] 纸面交易验证通过

### 文档
- [ ] README 已更新（如需要）
- [ ] 复杂逻辑有注释
- [ ] 配置参数已文档化

### 是否 Approve？
- [ ] 全部通过 → Approve
- [ ] 有问题 → Request Changes
- [ ] 需要进一步讨论 → Comment
```

---

## Algorithm Framework 模块分工

对于 2-3 人团队，推荐的分工方式：

### 分工矩阵

```
Person A: Alpha Research
├── Universe Selection
│   └── 选择交易品种
├── Signal Generation
│   └── 生成交易信号
└── 参数优化
    └── 回测不同参数组合

Person B: Portfolio Management & Execution
├── Position Sizing
│   └── 计算每笔订单的大小
├── Risk Management
│   └── 止损、风控
├── Order Execution
│   └── 订单类型、提交逻辑
└── 滑点模型
    └── 成交价格计算

Person C: Infrastructure & Monitoring
├── Data Pipeline
│   └── 数据加载、清洗、验证
├── Backtesting System
│   └── 回测执行、性能分析
├── Deployment & Monitoring
│   └── 上线、日常监控、告警
└── DevOps
    └── CI/CD、自动化测试
```

### 分工示例：Momentum Strategy

**Person A: Alpha 研究**

```python
# strategy_a_universe.py
def SelectUniverse():
    """
    选择流动性最好的 200 支股票
    """
    symbols = [
        "AAPL", "MSFT", "GOOGL",
        ... (200 symbols)
    ]
    return symbols

# strategy_a_signals.py
def GenerateSignal(data, params):
    """
    计算动量指标
    
    参数：
    - lookback: 回望窗口 (default: 20)
    - threshold: 信号阈值 (default: 0.02)
    """
    momentum = calculate_momentum(data, lookback=params['lookback'])
    signal = momentum > params['threshold']
    return signal

def calculate_momentum(prices, lookback=20):
    return (prices[-1] - prices[-lookback]) / prices[-lookback]
```

**Person B: 风险管理与执行**

```python
# strategy_b_risk.py
def CalculatePositionSize(symbol, price, portfolio_value, max_risk=0.01):
    """
    根据风险预算计算头寸大小
    
    目标：单笔损失不超过组合的 1%
    """
    atr = CalculateATR(symbol, 14)  # 波动率
    stop_loss = 0.02 * price  # 止损点
    
    risk = portfolio_value * max_risk
    position_size = risk / stop_loss
    
    # 不超过最大持仓限制
    max_size = portfolio_value * 0.1 / price
    return min(position_size, max_size)

def ApplyRiskControl(order, portfolio):
    """
    在执行前应用风控
    
    检查：
    1. 是否超过头寸限制
    2. 是否触发日亏损限制
    3. 是否超过杠杆限制
    """
    if PortfolioExceedsLimit(portfolio):
        return None  # 拒绝订单
    
    return order

# strategy_b_execution.py
def ExecuteOrder(signal, data, position_size):
    """
    执行订单
    
    - 信号确认后立即执行
    - 使用 Market 订单（简单起见）
    - 记录执行时间和成交价
    """
    if signal:
        return MarketOrder(symbol, position_size)
    return None
```

**Person C: 基础设施与监控**

```python
# data/data_loader.py
class DataLoader:
    """
    管理数据加载和验证
    """
    def LoadHistoricalData(self, symbols, start_date, end_date):
        """加载历史数据"""
        pass
    
    def ValidateData(self, data):
        """检查数据完整性、异常值"""
        pass

# backtest/backtest_runner.py
class BacktestRunner:
    """
    执行回测，生成报告
    """
    def RunBacktest(self, strategy_config):
        """
        1. 加载数据
        2. 初始化引擎
        3. 运行回测
        4. 生成报告
        """
        pass
    
    def CompareResults(self, baseline, current):
        """
        对比两个回测结果
        
        检查：是否有性能回归？
        """
        pass

# deployment/monitor.py
class StrategyMonitor:
    """
    监控实盘运行
    """
    def MonitorHealthCheck(self):
        """健康检查"""
        pass
    
    def SendAlert(self, level, message):
        """发送告警"""
        pass
    
    def GenerateDailyReport(self):
        """生成日报"""
        pass
```

### 接口契约（Interface Contract）

团队成员之间通过定义清晰的接口来解耦：

```python
# shared_interfaces.py

from abc import ABC, abstractmethod

class SignalGenerator(ABC):
    """信号生成器接口"""
    
    @abstractmethod
    def Generate(self, data: Dict, params: Dict) -> bool:
        """
        生成交易信号
        
        Args:
            data: 历史价格数据
            params: 参数字典
        
        Returns:
            bool: True=买入信号，False=卖出/无信号
        """
        pass

class RiskManager(ABC):
    """风控管理器接口"""
    
    @abstractmethod
    def CheckOrder(self, order: Order, portfolio: Portfolio) -> bool:
        """检查订单是否违反风控"""
        pass
    
    @abstractmethod
    def CalculatePositionSize(self, symbol: str, capital: float) -> int:
        """计算头寸大小"""
        pass

class DataProvider(ABC):
    """数据提供者接口"""
    
    @abstractmethod
    def LoadData(self, symbols: List[str], start: Date, end: Date) -> DataFrame:
        """加载历史数据"""
        pass
    
    @abstractmethod
    def ValidateData(self, data: DataFrame) -> bool:
        """验证数据完整性"""
        pass
```

---

## CI/CD 自动化回测

### GitHub Actions 工作流

当有人提交 PR 时，自动运行回测：

```yaml
# .github/workflows/backtest.yml

name: Automated Backtest

on:
  pull_request:
    branches:
      - develop
      - main

jobs:
  backtest:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest coverage
      
      - name: Run unit tests
        run: |
          pytest tests/ --cov=. --cov-report=xml
      
      - name: Run backtest (baseline)
        run: |
          python -m backtest.backtest_runner \
            --strategy momentum \
            --start-date 2023-01-01 \
            --end-date 2024-01-01 \
            --output baseline_report.json
      
      - name: Run backtest (current branch)
        run: |
          python -m backtest.backtest_runner \
            --strategy momentum \
            --start-date 2023-01-01 \
            --end-date 2024-01-01 \
            --output current_report.json
      
      - name: Compare results
        run: |
          python scripts/compare_reports.py \
            baseline_report.json \
            current_report.json \
            > comparison.md
      
      - name: Comment results on PR
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const comparison = fs.readFileSync('comparison.md', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comparison
            });
```

### 回测对比脚本

```python
# scripts/compare_reports.py

import json
import sys

def CompareMetrics(baseline, current):
    """对比两个回测结果"""
    
    metrics = {
        'total_return': baseline['total_return'] - current['total_return'],
        'sharpe': baseline['sharpe'] - current['sharpe'],
        'max_drawdown': current['max_drawdown'] - baseline['max_drawdown'],  # 越小越好
        'win_rate': baseline['win_rate'] - current['win_rate'],
    }
    
    return metrics

def GenerateComparison(baseline_file, current_file, output_file):
    """生成对比报告"""
    
    with open(baseline_file) as f:
        baseline = json.load(f)
    
    with open(current_file) as f:
        current = json.load(f)
    
    diff = CompareMetrics(baseline, current)
    
    report = f"""
## 回测性能对比

| 指标 | Baseline | Current | 变化 |
|------|----------|---------|------|
| Total Return | {baseline['total_return']:.2%} | {current['total_return']:.2%} | {diff['total_return']:+.2%} |
| Sharpe Ratio | {baseline['sharpe']:.2f} | {current['sharpe']:.2f} | {diff['sharpe']:+.2f} |
| Max Drawdown | {baseline['max_drawdown']:.2%} | {current['max_drawdown']:.2%} | {diff['max_drawdown']:+.2%} |
| Win Rate | {baseline['win_rate']:.2%} | {current['win_rate']:.2%} | {diff['win_rate']:+.2%} |

### 分析
"""
    
    # 性能回归检测
    if diff['total_return'] < -0.02:  # 回报下降 > 2%
        report += "⚠️ **性能回归**：总收益下降 > 2%\n"
    
    if diff['sharpe'] < -0.5:
        report += "⚠️ **风险调整回报下降**：Sharpe 下降 > 0.5\n"
    
    if diff['win_rate'] < -0.05:
        report += "⚠️ **胜率下降**：胜率下降 > 5%\n"
    
    with open(output_file, 'w') as f:
        f.write(report)

if __name__ == "__main__":
    baseline_file = sys.argv[1]
    current_file = sys.argv[2]
    output_file = sys.argv[3] if len(sys.argv) > 3 else "comparison.md"
    
    GenerateComparison(baseline_file, current_file, output_file)
```

---

## 策略版本管理

### 参数化配置

所有参数都应该外化到配置文件，不要硬编码：

```python
# strategies/momentum/momentum_strategy.py

class MomentumStrategy(QCAlgorithm):
    
    def Initialize(self):
        # 从配置文件加载参数
        config = self.LoadConfig("config.json")
        
        self.momentum_lookback = config['parameters']['momentum_lookback']
        self.entry_threshold = config['parameters']['entry_threshold']
        self.exit_threshold = config['parameters']['exit_threshold']
        
        self.risk_config = config['risk']
        
        self.AddEquity("SPY", Resolution.Daily)
    
    def LoadConfig(self, filename):
        import json
        with open(filename) as f:
            return json.load(f)
    
    def OnData(self, data):
        # 使用参数化的参数
        momentum = self.CalculateMomentum(self.momentum_lookback)
        
        if momentum > self.entry_threshold:
            self.Buy("SPY", 100)
```

配置文件：

```json
{
  "strategy": {
    "name": "momentum",
    "version": "1.2.0",
    "deployed_date": "2024-03-26"
  },
  "parameters": {
    "momentum_lookback": 20,
    "entry_threshold": 0.02,
    "exit_threshold": 0.01,
    "description": "entry when momentum > 2%, exit when momentum < 1%"
  },
  "risk": {
    "max_position_size": 0.1,
    "daily_loss_limit": 0.02,
    "stop_loss": 0.05
  }
}
```

### A/B 测试不同配置

```python
# backtest/ab_test.py

def RunABTest(strategy_name, configs, data_range):
    """
    对比多个参数组合
    """
    results = {}
    
    for config_name, config in configs.items():
        backtest_result = RunBacktest(
            strategy_name,
            config,
            data_range
        )
        
        results[config_name] = {
            'total_return': backtest_result.total_return,
            'sharpe': backtest_result.sharpe,
            'max_drawdown': backtest_result.max_drawdown,
            'win_rate': backtest_result.win_rate
        }
    
    return results

# 使用示例
configs = {
    'baseline': {'momentum_lookback': 20, 'entry_threshold': 0.02},
    'conservative': {'momentum_lookback': 30, 'entry_threshold': 0.03},
    'aggressive': {'momentum_lookback': 10, 'entry_threshold': 0.01},
}

results = RunABTest('momentum', configs, ('2023-01-01', '2024-01-01'))

# 输出表格
import pandas as pd
df = pd.DataFrame(results).T
print(df.to_string())
```

### 版本控制与部署

```
Versions:
├── v1.0.0 (2024-01-15)
│   ├── baseline
│   ├── deployed: 2024-02-01 (Live)
│   └── status: 运行中
│
├── v1.1.0 (2024-02-20)
│   ├── improvement: 加入 RSI 过滤器
│   ├── deployed: 2024-03-01 (Paper)
│   └── status: 测试中
│
└── v1.2.0 (2024-03-26)
    ├── improvement: 优化头寸规模
    ├── deployed: None
    └── status: 开发中
```

部署命令：

```bash
# 部署特定版本到纸面
./deploy.sh --version v1.1.0 --environment paper

# 部署特定版本到实盘
./deploy.sh --version v1.0.0 --environment live

# 回滚到上一个版本
./deploy.sh --version v1.0.0 --environment live --action rollback
```

---

## 知识管理

### 策略文档模板

每个策略都应该有完整的文档：

```markdown
# Momentum Strategy v1.0

## 概览
简介：基于动量指标的中频交易策略
目标收益：年化 15-20%
承诺风险：最大回撤 < 20%

## 核心逻辑

### 信号生成
- 指标：20 日动量
- 买入条件：动量 > 2%
- 卖出条件：动量 < 1% 或止损

### 头寸管理
- 单个头寸：最多 10% 资金
- 最大持仓数：10 个
- 杠杆：1.0x

## 回测结果

### 美股（2023-2024）
- 总收益：+18.3%
- Sharpe：1.25
- 最大回撤：-12.5%
- 胜率：52%

### 实盘运行
- 运行期间：2024-02-01 至今
- 实际收益：+5.2%（2 个月）
- 回测 vs 实盘偏差：-2% （符合预期）

## 参数说明

| 参数 | 值 | 作用 | 调整建议 |
|------|-----|------|---------|
| momentum_lookback | 20 | 动量计算窗口 | 降低 → 更敏感，提高 → 更稳定 |
| entry_threshold | 0.02 | 买入阈值 | 降低 → 增加交易次数，提高 → 降低交易次数 |

## 已知问题与局限

1. **流动性不足**：小盘股可能难以成交
2. **高频噪声**：短期动量可能被日内波动干扰
3. **市场制度**：在高波动率市场失效

## 改进计划

- [ ] 加入流动性过滤（日均成交量 > $10M）
- [ ] 测试多个时间框架的组合
- [ ] 研究在高波动率期间的参数调整

## 部署与维护

### 最后部署
- 版本：v1.0.0
- 日期：2024-02-01
- 部署环境：Live（美股）

### 维护计划
- 周报：每周五 17:00
- 参数调整：每月一次
- 重大更新：每季度一次
```

### 研究日志

记录研究过程和发现：

```markdown
# 研究日志

## 2024-03-20：测试均值回归策略

**目标**：改进动量策略的反向拐点检测

**实验**：
- 动量 + RSI 过滤器
- 设置：RSI(14) > 70 时过滤买入

**结果**：
- 胜率：51% → 53%
- 总收益：18% → 19%
- 交易次数：150 → 120（减少 20%）

**结论**：有效，将纳入 v1.1.0

---

## 2024-03-15：调查为什么纸面成交不如回测

**问题**：纸面成交比回测低 2%

**调查**：
1. 检查成交延迟 → 平均 3 秒（符合预期）
2. 检查滑点 → 平均 0.01%（合理）
3. 对比交易次数 → 纸面 120 笔，回测 150 笔

**根本原因**：流动性 → 回测忽视了流动性限制

**改进**：
- 加入流动性检查（日均成交量 > $10M）
- 重新回测 → 接近纸面表现

---

## 发现的交易法则

1. **在市场开盘 30 分钟内避免交易** → 波动率太高
2. **持仓 > 3 个的效果更差** → 监控压力增加
3. **月末避免交易** → 机制流动被锁定
```

### 周会议模板

```markdown
# 周会议记录 (2024-03-26)

**参加者**：Alice (Alpha), Bob (Risk), Charlie (Infrastructure)

**时间**：30 分钟

## 上周进展

### Alice - Alpha 研究
- ✅ 完成 RSI 过滤器开发
- ✅ 回测显示 +1% 性能提升
- 🚧 正在测试 MACD 指标

### Bob - 风险管理
- ✅ 修复日损失限制的 Bug
- ✅ 实盘运行 2 周，无风控触发
- 📊 平均日回报 +0.25%，符合预期

### Charlie - 基础设施
- ✅ 搭建 GitHub Actions CI/CD
- ✅ 自动回测系统上线
- 📊 每次 PR 自动运行回测，耗时 15 分钟

## 当前问题与讨论

### 问题 1：实盘持仓规模
- **现状**：当前用 1% 资金做 Alpha
- **议题**：是否增加到 5%？
- **决定**：再运行 1 周后评估，目标是年化 15%

### 问题 2：是否换券商
- **现状**：Alpaca，佣金 0，滑点约 0.01%
- **议题**：需要考虑 Interactive Brokers 吗？
- **决定**：目前 Alpaca 足够，后续如需国际市场再换

## 下周计划

- Alice：完成 MACD 测试，准备 v1.1 PR
- Bob：监控实盘，周末提交周报
- Charlie：优化 CI/CD，目标运行时间 < 10 分钟

## 下次会议
- 时间：2024-04-02
- 地点：Zoom
```

---

## 安全与合规

### API Key 管理

**绝对不要提交密钥到 Git**

```bash
# ❌ 错误做法
cat > config.py << EOF
API_KEY = "abc123def456"
API_SECRET = "secret789"
EOF
git add config.py  # 危险！

# ✅ 正确做法：使用环境变量
export BROKER_API_KEY="abc123def456"
export BROKER_API_SECRET="secret789"

# Python 中使用
import os
api_key = os.getenv("BROKER_API_KEY")
```

`.env.example` 模板（提交到 Git）：

```bash
# 复制此文件为 .env 并填入真实值
BROKER_API_KEY=your_key_here
BROKER_API_SECRET=your_secret_here
DATABASE_URL=postgresql://user:password@localhost/db
```

### 访问控制

```python
# deployment/security.py

class AccessControl:
    """
    控制谁可以做什么
    """
    
    def __init__(self):
        self.permissions = {
            'alice': ['backtest', 'paper_trading', 'review_code'],
            'bob': ['backtest', 'paper_trading', 'live_trading'],
            'charlie': ['backtest', 'deploy_infrastructure'],
        }
    
    def CanDeploy(self, user, environment):
        """检查用户是否有部署权限"""
        if environment == 'live':
            # 实盘部署需要团队共识
            return len(self.GetApprovals(user)) >= 2
        
        return True
    
    def GetApprovals(self, user):
        """获取批准人列表"""
        # 在实际中，这会查询 GitHub PR 的 review
        pass
```

### 审计日志

```python
# deployment/audit.py

class AuditLogger:
    """
    记录所有重要操作（合规要求）
    """
    
    def LogAction(self, user, action, details):
        """
        记录操作
        
        例：
        - user: alice
        - action: deploy_strategy
        - details: {strategy: momentum, version: v1.1.0, env: live}
        """
        log_entry = {
            'timestamp': datetime.now(),
            'user': user,
            'action': action,
            'details': details,
            'status': 'success'
        }
        
        # 写入不可改日志
        self._WriteToImmutableLog(log_entry)
    
    def _WriteToImmutableLog(self, entry):
        """写入不可改的日志（例如数据库）"""
        pass
```

---

## 团队成长路线图

### 第 1-4 周：基础建设

- **周 1-2**：搭建项目框架和 CI/CD
- **周 2-3**：开发第一个 MVP 策略（简单动量）
- **周 3-4**：纸面交易验证，发现问题

**输出**：
- 首个运行中的策略
- 自动化回测系统
- 团队协作流程

### 第 2-3 月：初期运营

- **月 2**：实盘交易第一个策略，初期资金 $5K
- **月 2-3**：持续优化和改进参数
- **月 3**：总结经验教训，准备扩展

**输出**：
- 1-2 个运行中的实盘策略
- 完整的监控和告警系统
- 团队知识库和文档

**关键指标**：
- 实盘月回报率：> 1%
- 风控触发次数：0
- 代码覆盖率：> 80%

### 第 3-6 月：扩展阶段

- **月 4**：开发 2-3 个新策略（均值回归、多因子等）
- **月 4-5**：多策略组合，分散风险
- **月 5-6**：考虑增加投入或团队规模

**输出**：
- 3-5 个运行中的策略
- 自动化投资组合管理
- 机制性运维流程

**关键指标**：
- 月回报率：> 1%（累计，所有策略）
- 最大回撤：< 15%
- 团队规模：可能需要增加到 4-5 人

---

## 总结

团队工程化的关键步骤：

1. **规范**：统一的代码风格、目录结构、命名规范
2. **流程**：清晰的 Git 工作流、Code Review、发布流程
3. **分工**：明确的职责分工和接口契约
4. **自动化**：CI/CD、自动回测、性能对比
5. **版本管理**：参数化配置、A/B 测试、版本控制
6. **知识管理**：文档、研究日志、团队会议
7. **安全与合规**：密钥管理、访问控制、审计日志

这样做的好处：

- ✅ 多人可以高效协作
- ✅ 降低错误和遗漏的风险
- ✅ 加速迭代和优化
- ✅ 为融资或扩展做准备

---

## 参考资源

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
- [Code Review Best Practices](https://google.github.io/eng-practices/review/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [12 Factor App](https://12factor.net/) - 现代应用的最佳实践

**导航**：[上一篇: 从回测到实盘](./09-go-live-checklist.md) | [返回目录](./README.md)
