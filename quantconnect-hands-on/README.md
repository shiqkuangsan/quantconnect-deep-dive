# QuantConnect 团队上手指南

> **定位：实操指南** — 面向创业团队的 "跟着做" 系列。每篇都是可执行的 lab，边做边学。
>
> 如果在实操过程中想深入理解某个设计原理，请查阅 [架构参考手册](../quantconnect-deep-dive/README.md)。

## 团队背景假设

- 2-3 人开发团队，学习能力强
- 了解量化交易基本概念（回测、实盘、策略）
- 有编程经验但不一定用过 C# 或量化平台
- 目标：先学会用 → 再理解架构 → 最终具备二次开发能力

## 学习路径总览

```
第一周：跑起来，建立直觉
  01 → 02 → 03

第二周：本地化，掌握工具链
  04 → 05

第三周：二次开发，从浅到深
  06 → 07 → 08

第四周：上线实战 + 团队规范
  09 → 10
```

## 文档目录

### 阶段一：跑起来（第 1 周）

| # | 文档 | 你将学到 | 耗时 |
|---|------|---------|------|
| 01 | [30 分钟跑通第一个策略](./01-first-strategy.md) | 注册 → 在线 IDE → 写策略 → 回测 → 看懂报告 | 30 min |
| 02 | [策略开发基础：API 实操手册](./02-api-handbook.md) | QCAlgorithm 核心 API：数据、下单、指标、调度、日志 | 60 min |
| 03 | [从简单到复杂：5 个策略实战](./03-strategy-labs.md) | 均线交叉 → RSI → 多因子 → 期权 → 加密货币，难度递进 | 半天 |

### 阶段二：本地化（第 2 周）

| # | 文档 | 你将学到 | 耗时 |
|---|------|---------|------|
| 04 | [本地开发环境搭建](./04-local-setup.md) | lean-cli + Docker + VS Code 跑通本地回测 | 60 min |
| 05 | [研究环境与数据探索](./05-research-and-data.md) | Jupyter Research、数据下载、可视化分析、策略原型 | 60 min |

### 阶段三：二次开发（第 3 周）

| # | 文档 | 你将学到 | 耗时 |
|---|------|---------|------|
| 06 | [二开入门：自定义指标](./06-custom-indicator.md) | 从零写指标 → 注册到引擎 → 在策略中使用 | 60 min |
| 07 | [二开进阶：自定义数据源](./07-custom-datasource.md) | 接入外部 API 数据 → BaseData 扩展 → 在策略中消费 | 90 min |
| 08 | [二开高阶：券商插件开发](./08-custom-brokerage.md) | IBrokerage 接口 → 插件骨架 → 模拟对接 → 测试 | 半天 |

### 阶段四：上线与协作（第 4 周）

| # | 文档 | 你将学到 | 耗时 |
|---|------|---------|------|
| 09 | [从回测到实盘：上线清单](./09-go-live-checklist.md) | Paper Trading → 券商配置 → 风控 → 监控 → 首次上线 | 60 min |
| 10 | [团队协作与工程化实践](./10-team-practices.md) | Git 工作流、代码规范、模块分工、CI 自动化 | 60 min |

---

## 与架构参考手册的对照关系

实操中遇到 "为什么" 的问题时，去这里找答案：

| 实操中的疑问 | 查阅架构手册 |
|------------|------------|
| TimeSlice 到底是什么？数据怎么流进来的？ | [deep-dive/03 数据管线](../quantconnect-deep-dive/03-data-pipeline.md) |
| Algorithm Framework 五层各是干什么的？ | [deep-dive/04 五层流水线](../quantconnect-deep-dive/04-algorithm-framework.md) |
| 回测和实盘为什么能用同一份代码？ | [deep-dive/06 回测与实盘](../quantconnect-deep-dive/06-backtest-vs-live.md) |
| FillModel / SlippageModel 这些是什么？ | [deep-dive/05 订单执行](../quantconnect-deep-dive/05-order-execution.md) |
| 插件仓库怎么组织的？IBrokerage 完整接口？ | [deep-dive/07 插件架构](../quantconnect-deep-dive/07-open-source-ecosystem.md) |
| 和 Backtrader / Zipline 比怎么样？ | [deep-dive/09 技术评估](../quantconnect-deep-dive/09-tech-evaluation.md) |

## 参考资源

- [QuantConnect 官方文档](https://www.quantconnect.com/docs/v2)
- [QuantConnect/Lean GitHub](https://github.com/QuantConnect/Lean)
- [lean-cli GitHub](https://github.com/QuantConnect/lean-cli)
- [QuantConnect 社区论坛](https://www.quantconnect.com/forum)
