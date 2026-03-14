# QuantConnect 深度拆解系列

> 面向有基础量化概念认知的前端/全栈开发者，系统性理解 QuantConnect 平台能力与 LEAN 引擎架构。

## 阅读建议

建议按编号顺序阅读。每篇文档都是自包含的，但后续文档会引用前面建立的概念。

## 文档目录

### 第一部分：认识平台

| 编号 | 文档 | 内容 | 阅读时间 |
|-----|------|------|---------|
| 01 | [平台全景与定位](./01-platform-overview.md) | QuantConnect 是什么、解决什么问题、产品形态、定价、与竞品的差异 | ~10 min |

### 第二部分：架构原理（核心）

| 编号 | 文档 | 内容 | 阅读时间 |
|-----|------|------|---------|
| 02 | [LEAN 引擎架构总览](./02-lean-architecture-overview.md) | 引擎全局视角——模块划分、执行流程、线程模型、配置驱动 | ~15 min |
| 03 | [数据管线与订阅系统](./03-data-pipeline.md) | DataFeed 子系统——数据如何从磁盘/网络流入算法、Subscription 生命周期、TimeSlice 同步机制 | ~15 min |
| 04 | [Algorithm Framework 五层流水线](./04-algorithm-framework.md) | 策略框架的关注分离设计——Universe → Alpha → Portfolio → Risk → Execution 每层详解 | ~15 min |
| 05 | [订单执行与真实性建模](./05-order-execution.md) | TransactionHandler 子系统——订单生命周期、FillModel / SlippageModel / FeeModel / BuyingPowerModel | ~12 min |
| 06 | [回测与实盘的统一抽象](./06-backtest-vs-live.md) | 回测和实盘如何共享同一份代码和执行路径——接口抽象、配置切换、差异点 | ~12 min |

### 第三部分：生态与实操

| 编号 | 文档 | 内容 | 阅读时间 |
|-----|------|------|---------|
| 07 | [开源生态与插件架构](./07-open-source-ecosystem.md) | GitHub 92+ 仓库全景、Brokerages / DataSource 插件设计模式、如何贡献 | ~10 min |
| 08 | [本地部署实操指南](./08-local-deployment.md) | 从零搭建本地 LEAN 环境——Docker / lean-cli / 源码编译三条路径 | ~15 min |

### 第四部分：评估与决策

| 编号 | 文档 | 内容 | 阅读时间 |
|-----|------|------|---------|
| 09 | [技术选型评估与竞品对比](./09-tech-evaluation.md) | 与 Backtrader / Zipline / VectorBT 等对比、适用场景分析、决策框架 | ~12 min |

---

## 全系列关键概念速查

如果你在阅读中遇到不理解的术语，可以到对应文档中查找详细解释：

- **TimeSlice** → 03 数据管线
- **Insight** → 04 Algorithm Framework
- **Reality Modeling** → 05 订单执行
- **Handler 体系** → 02 架构总览 / 06 回测与实盘
- **IBrokerage 接口** → 07 插件架构
- **lean-cli** → 08 本地部署

## 参考资源

- [QuantConnect/Lean GitHub](https://github.com/QuantConnect/Lean)
- [QuantConnect 官方文档](https://www.quantconnect.com/docs/v2)
- [LEAN Engine 文档](https://www.quantconnect.com/docs/v2/lean-engine)
- [lean-cli GitHub](https://github.com/QuantConnect/lean-cli)
