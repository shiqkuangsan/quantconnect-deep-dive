# 本地开发环境搭建

> **定位**：从云端转向本地，建立独立的开发、测试、迭代流程。这是从 "玩家" 向 "开发者" 的过渡。
>
> 本篇通过 lean-cli + Docker 实现**本地策略开发 → 本地回测 → 结果查看**的完整闭环。

**所需时间**：60 分钟
**前置条件**：完成文档 01-03，对 QuantConnect 和策略开发有基本认识
**完成后**：本地能独立运行策略回测，无需依赖云端

---

## 为什么需要本地开发环境？

在文档 01-03 中，你一直在 QuantConnect 的云端 IDE 中开发和回测。这对学习很方便，但当你的团队要向前推进时，本地开发就变得必须：

### 1. **开发速度**
- 云端每次回测需要排队（免费账户有限制）
- 本地回测立即执行，快速迭代
- 2-3 人的小团队通常能将周期缩短 30%-50%

### 2. **隐私与数据安全**
- 策略代码留在本地，不上传云端
- 敏感的交易逻辑、参数调试过程完全私密
- 符合某些机构的合规要求

### 3. **自定义与扩展性**
- 集成外部数据源、自定义指标、自建工具链
- 使用 VS Code 等本地编辑器，熟悉的开发体验
- 调试能力大幅提升（断点、日志等）

### 4. **离线工作**
- 无网络时仍能本地开发、测试
- 适合出差、旅途中的工作

### 5. **成本控制**
- 本地回测完全免费（只需下载数据一次）
- 云端回测按 CPU 小时计费
- 大量迭代的成本差异明显

---

## 方案选择：lean-cli + Docker（推荐）

QuantConnect 官方提供多种本地运行方式：

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **lean-cli + Docker** | 简单、官方支持、无需编译、跨平台一致 | 需要 Docker | ⭐⭐⭐⭐⭐ |
| Lean 源码编译 | 完全控制、可深度修改 | 编译复杂、耗时长、容易出错 | ⭐⭐ |
| Docker 直接运行 | 最小化依赖 | 需要手工管理卷、命令冗长 | ⭐⭐⭐ |

**对于 2-3 人团队，强烈推荐 lean-cli + Docker**，原因：

1. **傻瓜式上手**：一条命令初始化项目、一条命令运行回测
2. **官方维护**：lean-cli 是 QuantConnect 官方工具，及时更新
3. **跨平台一致**：Mac / Windows / Linux 体验相同
4. **自动化配置**：无需手工配置数据路径、依赖等

本篇**只涵盖 lean-cli + Docker** 这一方案。

---

## 前置准备清单

在开始前，请确保你的电脑满足以下条件：

### 1. Python 3.8+

检查方法：
```bash
python --version
# 或
python3 --version
```

**预期输出：**
```
Python 3.8.0
```
或更新版本（3.9, 3.10, 3.11 都支持）

**如果没有安装：**
- **macOS**：`brew install python@3.11`
- **Windows**：访问 [python.org](https://www.python.org/downloads/) 下载安装程序
- **Linux**：`sudo apt-get install python3.11`

### 2. pip（Python 包管理器）

检查方法：
```bash
pip --version
# 或
pip3 --version
```

**预期输出：**
```
pip 21.0.0 from ... (python 3.11)
```

通常 Python 安装时已包含 pip，无需额外操作。

### 3. Docker Desktop

检查方法：
```bash
docker --version
```

**预期输出：**
```
Docker version 20.10.0, build ...
```

**如果没有安装：**
- 访问 [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
- 选择你的操作系统，按指南安装
- **安装后必须启动 Docker Desktop 应用**（任务栏会出现 Docker 图标）

**重要**：lean-cli 会自动使用 Docker 运行 LEAN 引擎。如果 Docker 未运行，会报错。

### 4. VS Code + Python 扩展

可选但推荐。

安装步骤：
1. 下载 [Visual Studio Code](https://code.visualstudio.com/)
2. 启动 VS Code，进入扩展市场（Ctrl+Shift+X / Cmd+Shift+X）
3. 搜索 **"Python"**，安装官方的 Microsoft Python 扩展

### 5. Git（可选）

如果你的团队使用 Git 做版本控制：
```bash
git --version
```

**如果需要安装：**
- 访问 [git-scm.com](https://git-scm.com/download)
- 按平台指南安装

---

## Step 1：安装 lean-cli

lean-cli 是 QuantConnect 官方命令行工具，用于管理本地项目、运行回测、管理数据。

### 1.1 通过 pip 安装

```bash
pip install lean
# 或如果你同时有 Python 2：
pip3 install lean
```

**预期输出：**
```
Successfully installed lean-x.x.x
```

### 1.2 验证安装

```bash
lean --version
```

**预期输出：**
```
LEAN CLI version X.X.X
```

如果看到版本号，恭喜，安装成功！

### 1.3 常见问题

**问题：`pip: command not found`**
→ 使用 `pip3` 代替 `pip`

**问题：`lean: command not found`**
→ 检查 pip 安装路径。在 macOS 上，可能需要将 `~/.local/bin` 加入 PATH，或重启终端

---

## Step 2：初始化项目

lean-cli 提供一条命令来生成项目骨架。

### 2.1 创建项目目录

```bash
mkdir my-quant-project
cd my-quant-project
```

### 2.2 初始化项目

```bash
lean init
```

**交互式提示：**

```
? Enter project name: My First Quant Strategy
? Select project language: Python (或 C# 自选)
? Select project type: Algorithm (标准选项)
```

- **Project name**：你的项目名称，可以是任何描述性名称
- **Language**：Python 或 C#（本篇以 Python 为例）
- **Type**：Algorithm（标准策略），IDE 会自动生成 main.py

### 2.3 生成的项目结构

完成后，你会看到：

```
my-quant-project/
├── main.py                    # 主策略文件
├── lean.json                  # 项目配置文件
├── data/                      # 本地数据存储目录
│   └── (初始为空)
└── backtests/                 # 回测结果存储目录
    └── (初始为空)
```

### 2.4 理解 lean.json

这是项目的配置文件。打开 `lean.json`：

```json
{
  "name": "My First Quant Strategy",
  "description": "A strategy developed using the Lean CLI",
  "author": "Your Name",
  "active": true,
  "parameters": {
    "min-bar-history": 300,
    "min-trailing-days": 20
  },
  "data-folder": "data",
  "results-folder": "backtests",
  "version": "1",
  "algorithm-language": "Python"
}
```

关键字段：
- **name**：项目名
- **data-folder**：数据存储位置（相对于项目根目录）
- **results-folder**：回测结果输出位置
- **algorithm-language**：Python 或 C#

暂时不需要修改，默认配置可用。

---

## Step 3：写你的第一个本地策略

现在写一个简单的 SMA 均线交叉策略，存放在 `main.py`。

### 3.1 打开 main.py

你会看到一个模板：

```python
# QUANTCONNECT.COM - Democratizing Finance, Empowering Individuals.
# Lean Algorithmic Trading Engine v2.0. Copyright 2014 QuantConnect Corporation.

from AlgorithmImports import *

class MyFirstQuantStrategy(QCAlgorithm):

    def Initialize(self):
        # 设置回测参数
        self.SetStartDate(2020, 1, 1)
        self.SetEndDate(2023, 12, 31)
        self.SetCash(100000)

        # 订阅数据
        self.symbol = "SPY"
        self.AddEquity(self.symbol, Resolution.Daily)

        # 初始化指标
        self.fast_sma = self.SMA(self.symbol, 20, Resolution.Daily)
        self.slow_sma = self.SMA(self.symbol, 50, Resolution.Daily)

        # 状态跟踪
        self.in_position = False

    def OnData(self, data):
        # 等待指标预热
        if not self.fast_sma.IsReady or not self.slow_sma.IsReady:
            return

        # 获取当前价格
        if not data.ContainsKey(self.symbol):
            return

        current_price = data[self.symbol].Close

        # 买入信号：快线上穿慢线
        if not self.in_position and self.fast_sma.Current.Value > self.slow_sma.Current.Value:
            self.SetHoldings(self.symbol, 1.0)  # 全仓买入
            self.in_position = True
            self.Log(f"买入 - 价格: {current_price:.2f}")

        # 卖出信号：快线下穿慢线
        if self.in_position and self.fast_sma.Current.Value < self.slow_sma.Current.Value:
            self.Liquidate(self.symbol)  # 平仓
            self.in_position = False
            self.Log(f"卖出 - 价格: {current_price:.2f}")
```

### 3.2 理解代码结构

| 部分 | 作用 |
|------|------|
| `Initialize()` | 初始化：设置时间段、资金、订阅数据、创建指标 |
| `OnData(data)` | 每个数据点（日线）触发一次，包含信号生成和下单逻辑 |
| `SetHoldings()` | 以指定权重持仓 |
| `Liquidate()` | 平仓卖出 |
| `Log()` | 打印日志，便于调试 |

### 3.3 保存文件

编辑完后，保存文件（Ctrl+S / Cmd+S）。

---

## Step 4：本地运行回测

现在使用 lean-cli 在本地执行回测。

### 4.1 执行回测命令

回到项目根目录，运行：

```bash
cd my-quant-project
lean backtest "My First Quant Strategy"
```

**注意**：策略名称必须与 lean.json 中的 `name` 字段匹配。

### 4.2 第一次运行会发生什么？

```
[初次运行会下载 Docker 镜像，耗时 2-5 分钟，只需一次]

Pulling LEAN Engine Docker image...
latest: Pulling from quantconnect/lean
2d473b3ef3d1: Pull complete
...
Status: Downloaded newer image for quantconnect/lean:latest

Starting backtest...
```

然后开始回测：

```
Loading algorithm...
Initializing algorithm...
Fetching data...
Running backtest from 2020-01-01 to 2023-12-31...
[==================================================] 100%

Backtest completed in 12.34 seconds
```

### 4.3 回测完成

```
Backtest Results:
├─ Total Return: 45.23%
├─ Sharpe Ratio: 1.32
├─ Max Drawdown: -18.5%
├─ Total Trades: 42
└─ Win Rate: 52.4%

Results saved to: backtests/My First Quant Strategy_YYYYMMDD_HHMMSS.json
```

**恭喜！你的第一个本地回测成功了。**

### 4.4 如果回测失败

常见错误与排查：

| 错误信息 | 原因 | 解决方案 |
|---------|------|--------|
| `Docker daemon is not running` | Docker 应用未启动 | 打开 Docker Desktop |
| `name 'SPY' is not recognized` | 数据缺失 | 见 Step 7 数据管理 |
| `SyntaxError in main.py` | 代码语法错误 | 检查 Python 代码，修正后重试 |
| `lean: command not found` | lean-cli 未安装或 PATH 错误 | 重新安装或重启终端 |

---

## Step 5：查看和分析回测结果

回测完成后，结果存储在 `backtests/` 目录。

### 5.1 结果文件位置

```
backtests/
└── My First Quant Strategy_20240326_143022.json
```

文件名格式：`{策略名}_{YYYYMMDD}_{HHMMSS}.json`

### 5.2 查看结果

方法 1：使用 lean-cli 的结果查看器

```bash
lean results list
```

输出示例：
```
Recent Backtest Results:
1. My First Quant Strategy_20240326_143022
   - Date: 2024-03-26 14:30:22
   - Return: 45.23%
   - Sharpe: 1.32
```

方法 2：在浏览器中查看

lean-cli 会自动生成可视化的 HTML 报告。使用以下命令：

```bash
lean results show "My First Quant Strategy_20240326_143022"
```

系统会自动打开浏览器，展示：
- 净值曲线
- 收益分布
- 月度收益表
- 交易列表
- 风险指标

### 5.3 理解关键指标

| 指标 | 含义 |
|------|------|
| **Total Return** | 总收益率，例如 45.23% 意味着账户增长了 45% |
| **Sharpe Ratio** | 风险调整后的收益，值越高越好（通常 > 1 算不错） |
| **Max Drawdown** | 最大回撤，账户从高点跌到低点的幅度，例如 -18.5% |
| **Total Trades** | 总交易次数 |
| **Win Rate** | 盈利交易占比 |

### 5.4 导出结果

结果已经以 JSON 格式保存在 `backtests/` 目录。可以：

- 在 Excel 中打开 JSON，分析详细数据
- 使用 Python 脚本进一步处理
- 上传到团队的数据分析平台

---

## Step 6：VS Code 开发体验优化

本地开发的真正优势在于工具链。VS Code 配合 QuantConnect 插件可以大幅提升效率。

### 6.1 安装 QuantConnect Stubs（自动补全）

QuantConnect 提供了类型定义文件（stubs），让 VS Code 能够理解 QCAlgorithm、Indicators 等对象，从而实现自动补全。

```bash
pip install quantconnect-stubs
```

安装后，在 VS Code 中编辑 `main.py` 时，会自动弹出 API 建议：

```
self.AddEquity(     # 输入时弹出
├─ symbol: str
├─ resolution: Resolution
└─ fillDataTypes: str
```

### 6.2 配置 VS Code settings.json

打开 VS Code，按 Ctrl+Shift+P（Mac: Cmd+Shift+P），搜索 "Preferences: Open Settings (JSON)"。

添加以下配置：

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": true,
  "[python]": {
    "editor.defaultFormatter": "ms-python.python",
    "editor.formatOnSave": true,
    "editor.rulers": [80, 120]
  }
}
```

说明：
- 设定 Python 解释器路径
- 启用 linting（代码检查）
- 保存时自动格式化

### 6.3 调试技巧

**方法 1：使用 Log() 打印调试**

在 OnData 中加入日志：

```python
def OnData(self, data):
    self.Log(f"当前价格: {data[self.symbol].Close}")
    self.Log(f"快线: {self.fast_sma.Current.Value:.2f}")
    self.Log(f"慢线: {self.slow_sma.Current.Value:.2f}")
```

运行回测后，在结果 JSON 中会包含所有日志。

**方法 2：在 VS Code 中设置断点**

lean-cli 支持 Python 调试器（pdb）。在代码中加入：

```python
def OnData(self, data):
    import pdb; pdb.set_trace()  # 执行到这里时暂停
    # ... 后续代码
```

运行回测时会在此暂停，允许单步调试。

### 6.4 高效工作流

推荐的开发流程：

```
1. 编辑 main.py
   ↓
2. 保存（Ctrl+S）
   ↓
3. 在终端运行 lean backtest "策略名"
   ↓
4. 查看结果（lean results show）
   ↓
5. 根据结果调整参数，回到第 1 步
```

为了加速，可以在 VS Code 中配置快捷任务。创建 `.vscode/tasks.json`：

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Run Backtest",
      "type": "shell",
      "command": "lean",
      "args": ["backtest", "My First Quant Strategy"],
      "group": {
        "kind": "test",
        "isDefault": true
      },
      "presentation": {
        "reveal": "always"
      }
    }
  ]
}
```

之后按 Ctrl+Shift+B（Mac: Cmd+Shift+B）即可快速运行回测。

---

## Step 7：数据管理

lean-cli 通过 QuantConnect 的数据 API 提供数据。本地策略使用的数据来自两个渠道：

### 7.1 QuantConnect 免费数据包

QuantConnect 官方提供的免费数据包括：

- **股票（US Equities）**：SPY, QQQ, IWM 等主流指数和个股，从 2010 年至今
- **期货（Futures）**：主要品种（ES, NQ, CL 等）
- **期权（Options）**：美股期权链
- **加密货币（Crypto）**：BTC, ETH 等

首次使用时，lean-cli 会自动从 QuantConnect 的服务器下载必要的数据。

### 7.2 下载数据到本地

你也可以主动预下载数据：

```bash
lean data download --symbol SPY --resolution Daily --start 2015-01-01 --end 2024-01-01
```

参数解释：
- `--symbol`：股票代码
- `--resolution`：分辨率（Daily, Minute, Hour 等）
- `--start` / `--end`：日期范围

下载完成后，数据存储在 `data/` 目录：

```
data/
├── equity/
│   └── usa/
│       └── daily/
│           └── SPY.zip
└── ...
```

### 7.3 数据格式与 OOTC

QuantConnect 使用自有的数据格式（OOTC - QuantConnect Open Trade Container），经过优化压缩，加载速度快。

如果你要使用 CSV 数据：

**支持 CSV 导入**（详见文档 05）。

### 7.4 数据文件夹结构

典型的完整目录：

```
data/
├── equity/
│   └── usa/
│       ├── daily/          # 日线数据
│       │   ├── SPY.zip
│       │   └── QQQ.zip
│       └── minute/         # 分钟线数据
│           └── SPY.zip
├── option/
│   └── usa/
│       └── daily/
│           └── SPY.zip
└── crypto/
    └── coinbase/           # 交易所数据
        └── daily/
            └── BTCUSD.zip
```

### 7.5 检查数据可用性

在策略运行前，检查数据是否已下载：

```bash
lean data list
```

输出示例：
```
Available Data:
├─ equity/usa/daily/SPY
├─ equity/usa/daily/QQQ
├─ crypto/coinbase/daily/BTCUSD
└─ ...
```

如果某个数据缺失，lean-cli 在回测时会自动下载。

---

## Step 8：常见问题排查

### 问题 1：Docker 未运行

**错误信息：**
```
Error: Docker daemon is not running. Please start Docker and try again.
```

**解决方案：**
1. 打开 Docker Desktop 应用
2. 等待启动完成（任务栏出现 Docker 图标）
3. 重新运行 `lean backtest`

### 问题 2：权限错误（Permission Denied）

**错误信息：**
```
PermissionError: [Errno 13] Permission denied: '/path/to/data'
```

**解决方案（Linux/macOS）：**
```bash
chmod -R 755 data/
```

### 问题 3：Python 版本不匹配

**错误信息：**
```
SyntaxError: invalid syntax (Python 2 vs 3 issue)
```

**检查 Python 版本：**
```bash
python --version
```

**如果是 Python 2：**
- 使用 `python3` 代替 `python`
- 或升级到 Python 3.8+

### 问题 4：数据未找到

**错误信息：**
```
No data available for symbol SPY from 2020-01-01 to 2024-01-01
```

**解决方案：**

Step 1：检查数据是否已下载
```bash
lean data list | grep SPY
```

Step 2：如果未下载，手动下载
```bash
lean data download --symbol SPY --resolution Daily --start 2020-01-01 --end 2024-01-01
```

Step 3：检查策略中的日期范围是否超出数据范围
```python
self.SetStartDate(2020, 1, 1)  # 确保在数据可用范围内
```

### 问题 5：内存不足（Out of Memory）

**错误信息：**
```
MemoryError: Unable to allocate X.XX GiB for array
```

**原因：** 策略耗时太长或订阅数据过多

**解决方案：**

方法 1：缩短回测时间范围
```python
self.SetStartDate(2023, 1, 1)   # 改为近 1 年
self.SetEndDate(2023, 12, 31)
```

方法 2：增加 Docker 容器内存限制
```bash
lean backtest "My First Quant Strategy" --docker-memory 4G
```

方法 3：优化策略（减少订阅数据数量）

---

## Step 9：团队共享配置

如果你的团队使用 Git 进行版本控制，以下是推荐的共享方式。

### 9.1 .gitignore 配置

创建 `.gitignore` 文件，排除不必要的文件：

```
# 局部数据和结果（勿上传）
/data/
/backtests/

# Python 环境（每人本地安装）
__pycache__/
*.pyc
.venv/
venv/

# IDE 配置（个人偏好）
.vscode/
.idea/

# 操作系统文件
.DS_Store
Thumbs.db
```

### 9.2 项目模板

创建 `README.md` 指导新成员：

```markdown
# My Quant Strategy

## 快速开始

1. 克隆仓库
   ```bash
   git clone <repo-url>
   cd my-quant-project
   ```

2. 安装依赖
   ```bash
   pip install lean
   ```

3. 运行回测
   ```bash
   lean backtest "My First Quant Strategy"
   ```

4. 查看结果
   ```bash
   lean results show
   ```

## 文件说明

- `main.py` — 主策略文件
- `lean.json` — 项目配置
- `data/` — 数据文件（由 lean 自动管理）
- `backtests/` — 回测结果

## 开发工作流

1. 在 `main.py` 中编辑策略
2. 运行 `lean backtest`
3. 查看结果并迭代
```

### 9.3 参数化与配置

如果团队成员需要不同的参数配置，使用 `lean.json` 的 `parameters` 字段：

```json
{
  "name": "My Strategy",
  "parameters": {
    "symbol": "SPY",
    "fast_period": 20,
    "slow_period": 50
  }
}
```

然后在 `main.py` 中读取：

```python
def Initialize(self):
    symbol = self.GetParameter("symbol", "SPY")
    fast = int(self.GetParameter("fast_period", "20"))
    slow = int(self.GetParameter("slow_period", "50"))
```

成员可以修改 `lean.json` 中的参数，而无需改动代码。

### 9.4 持续集成（可选）

如果你的团队想要自动化测试，可以配置 GitHub Actions：

创建 `.github/workflows/backtest.yml`：

```yaml
name: Backtest

on: [push, pull_request]

jobs:
  backtest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.11
      - name: Install dependencies
        run: |
          pip install lean
      - name: Run backtest
        run: |
          lean backtest "My Strategy"
```

这样，每次提交代码时，GitHub 会自动运行回测，确保代码不破坏策略逻辑。

---

## 总结与下一步

你现在已经：

✅ 安装了 lean-cli 和 Docker
✅ 初始化了本地项目
✅ 编写并运行了第一个本地策略回测
✅ 理解了结果的含义
✅ 优化了开发环境
✅ 掌握了数据管理和团队协作方式

**下一步**：在文档 05 中，我们会介绍 **Research 环境**，即 Jupyter Notebook 在 QuantConnect 中的用法。你将学会：

- 在本地 Jupyter 中快速探索数据
- 验证交易想法（无需完整策略代码）
- 生成可视化分析图表
- 从 Notebook 升级到正式策略

---

**导航**
[上一篇: 5 个策略实战](./03-strategy-labs.md) | [下一篇: 研究环境与数据探索](./05-research-and-data.md)
