# Lean 算法交易引擎 — 从零入门使用路线规划图

> 基于 [lean分析文档0.md](./lean分析文档0.md) 与 [Lean深度分析报告.md](./Lean深度分析报告.md) 提炼的渐进式学习路径，目标读者为**零基础量化开发者**。

---

## 路线总览

```
Phase 0  →  Phase 1  →  Phase 2  →  Phase 3  →  Phase 4  →  Phase 5  →  Phase 6
前置认知    环境搭建    第一策略    回测验证    风险风控    进阶技巧    实盘就绪
(1天)       (1-2天)    (3-5天)    (1-2周)    (2-3周)    (1-2月)    (持续)
```

---

## Phase 0：前置认知（1天）

> **目标**：理解 Lean 是什么、能做什么、不能做什么，建立正确的心理预期。

| 步骤 | 内容 | 关键要点 |
|------|------|----------|
| 0.1 | 阅读两份分析报告的第 1 节（概述） | 理解事件驱动引擎的概念、C#/Python 混合架构、多市场覆盖范围 |
| 0.2 | 浏览 [Lean GitHub](https://github.com/QuantConnect/Lean) | 关注 18.5k stars、更新频率、Issue 活跃度 |
| 0.3 | 理解核心概念 | 算法框架（QCAlgorithm）、数据层（Data/）、经纪商插件（Brokerages/）、结果处理（Results/） |
| 0.4 | 建立正确预期 | **前 10 个策略应全部失败**（报告强调），这是学习过程不是挫折 |

**可交付成果**：能用一句话向别人解释 Lean 是什么。

---

## Phase 1：环境搭建（1–2天）

> **目标**：成功运行 `lean backtest` 并获得第一个回测结果。

### 1.1 安装 Lean CLI（推荐路径）

```bash
# Step 1: 安装 Lean CLI
pip install lean

# Step 2: 验证安装
lean --version
```

### 1.2 配置 Docker（Lean CLI 依赖 Docker 运行策略）

```bash
# 检查 Docker 是否已安装
docker --version

# 如未安装，参考 https://docs.docker.com/get-docker/
```

### 1.3 创建第一个项目

```bash
lean project-create --language python MyFirstStrategy
```

目录结构：
```
MyFirstStrategy/
├── main.py          # 策略主文件
├── research.ipynb   # Jupyter 研究笔记本
└── config.json      # 项目配置
```

### 1.4 运行首次回测

```bash
lean backtest --project MyFirstStrategy
```

### 1.5（可选）安装开发辅助工具

```bash
pip install quantconnect-stubs    # IDE 自动补全
```

### 1.6（可选）启动研究环境

```bash
lean research    # 启动 Jupyter Lab（Docker 内）
```

**检查点**：看到回测输出统计（夏普比率、年化收益、最大回撤等）。

---

## Phase 2：编写第一个策略（3–5天）

> **目标**：理解 QCAlgorithm 生命周期，手写一个可运行的趋势跟踪策略。

### 2.1 理解算法生命周期

```
Initialize() → OnData(Slice data) 循环 → OnOrderEvent() → OnEndOfDay()
     │               │                      │                 │
  设置参数     每次数据推送触发         订单状态变更       每日收盘后回调
```

### 2.2 第一个策略：买入持有 SPY

```python
from AlgorithmImports import *

class BuyAndHoldSPY(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)       # 回测起始日期
        self.SetEndDate(2023, 12, 31)        # 回测结束日期
        self.SetCash(100000)                 # 初始资金
        self.AddEquity("SPY", Resolution.Daily)

    def OnData(self, data):
        if not self.Portfolio.Invested:
            self.SetHoldings("SPY", 1.0)     # 全仓买入
```

### 2.3 第二个策略：移动平均线交叉

```python
class MACrossover(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)
        self.SetCash(100000)
        self.spy = self.AddEquity("SPY", Resolution.Daily).Symbol
        self.fast = self.SMA("SPY", 50, Resolution.Daily)   # 快线
        self.slow = self.SMA("SPY", 200, Resolution.Daily)  # 慢线

    def OnData(self, data):
        if not self.fast.IsReady or not self.slow.IsReady:
            return
        if self.fast.Current.Value > self.slow.Current.Value:
            self.SetHoldings(self.spy, 1.0)   # 金叉买入
        else:
            self.Liquidate(self.spy)           # 死叉卖出
```

### 2.4 添加定时调度（避免流动性不足时段）

```python
self.Schedule.On(
    self.DateRules.EveryDay("SPY"),
    self.TimeRules.AfterMarketOpen("SPY", 30),  # 开盘后 30 分钟执行
    self.Rebalance
)
```

**检查点**：能独立写出带指标判断的策略，并理解回测报告中的基本指标。

---

## Phase 3：回测验证与偏差防范（1–2周）

> **目标**：学会验证策略的"真实"表现，而不是被漂亮的曲线欺骗。

### 3.1 理解回测报告

```
回测输出 → 关注以下指标：
├── Total Return（总收益）
├── Sharpe Ratio（夏普比率，>1 较好）
├── Max Drawdown（最大回撤，<20% 较安全）
├── Win Rate（胜率）
└── CAGR（复合年增长率）
```

### 3.2 四大回测偏差检查

| 偏差类型 | 含义 | 检查方法 |
|----------|------|----------|
| **前视偏差** | 策略使用了未来数据 | 确认 `OnData` 只访问当前切片的数据 |
| **幸存者偏差** | 只用现存股票做回测 | 引入退市数据（Delisting 事件处理） |
| **过拟合风险** | 参数调得太"完美" | 留出样本外测试期（如 2023-2025），只在前半段调参 |
| **交易成本低估** | 忽略了滑点和手续费 | 使用 `SetBrokerageModel()`，后续可自定义 `IFeeModel`、`ISlippageModel` |

### 3.3 实践：样本外验证

```python
# 训练期
self.SetStartDate(2018, 1, 1)
self.SetEndDate(2022, 12, 31)

# 测试期（另开一次回测，参数锁定不变）
# self.SetStartDate(2023, 1, 1)
# self.SetEndDate(2025, 12, 31)
```

### 3.4 数据质量自查

- 价格 ≤ 0 的数据点是否被过滤？
- 成交量 = 0 的交易日是否被处理？
- 时间序列是否有断点？

**检查点**：完成一次完整的样本内调参 + 样本外验证流程，能解释为什么回测回报 ≠ 实盘回报。

---

## Phase 4：风险控制体系（2–3周）

> **目标**：为策略添加三层风控，理解"活下来"比"赚得多"更重要。

### 4.1 第一层：策略级风控

```python
def Initialize(self):
    # 最大仓位限制
    self.Settings.MaximumOrderQuantity = 1000

    # 日内交易限制
    self.Settings.DailyLimitBuyQuantity = 5000

    # 最大回撤熔断（20% 自动停止）
    self.SetMaximumDrawdownPercent(20.0)

    # 保留 5% 现金缓冲
    self.SetHoldings(self.symbol, 0.95)  # 而不是 1.0
```

### 4.2 波动率自适应仓位

```python
# 根据市场状态动态调整风险暴露
if self.Securities[self.symbol].VolatilityModel.Volatility > 0.3:
    self.SetHoldings(self.symbol, 0.5)   # 高波动时减半仓位
else:
    self.SetHoldings(self.symbol, 0.95)  # 正常波动时正常仓位
```

### 4.3 第二层：系统级风控（概念理解）

- `AlgorithmTimeLimitManager`：防止无限循环
- 实时异常监控（`LiveTradingResultHandler`）
- 订单超时与自动撤销

### 4.4 第三层：运营级风控（概念理解）

- 每日盈亏报告
- 关键指标监控（夏普比率、最大回撤、胜率的趋势变化）
- 异常交易模式检测（频率突变、单标的重仓等）

### 4.5 防止过度交易

```python
class CoolingOffPeriod:
    """最小交易间隔 — 行为金融学防御"""
    def __init__(self, minutes=30):
        self.last_trade = {}
        self.min_interval = timedelta(minutes=minutes)

    def can_trade(self, symbol, current_time):
        if symbol in self.last_trade:
            if current_time - self.last_trade[symbol] < self.min_interval:
                return False
        self.last_trade[symbol] = current_time
        return True
```

**检查点**：策略在任何情况下都不会让单日亏损超过总资金的 20%，仓位能随波动率自适应。

---

## Phase 5：进阶技巧（1–2月）

> **目标**：掌握自定义模型、另类数据和市场状态识别。

### 5.1 自定义交易成本模型

```python
class CustomFeeModel(FeeModel):
    def GetOrderFee(self, parameters):
        # 自定义手续费逻辑（如阶梯费率）
        fee = max(1.0, parameters.Order.AbsoluteQuantity * 0.005)
        return OrderFee(CashAmount(fee, "USD"))
```

### 5.2 自定义执行模型（避免不利时段交易）

```python
class AvoidOpeningAuction(ExecutionModel):
    def Execute(self, algorithm, targets):
        # 开盘前 5 分钟不下单
        if algorithm.Time.TimeOfDay < timedelta(hours=9, minutes=35):
            return
        # 盘口太薄时减半下单
        for target in targets:
            algorithm.SetHoldings(target.Symbol, target.Quantity * 0.5)
```

### 5.3 引入另类数据

- 新闻情绪评分（`NewsSentiment` 自定义数据类型）
- 社交媒体热度
- 供应链/卫星数据

示例（来自报告）：
```python
if slice.ContainsKey("NEWS_SENTIMENT"):
    sentiment = slice.Get[NewsSentiment]("NEWS_SENTIMENT")
    # 将非结构化数据转化为交易信号
```

### 5.4 市场状态识别

```python
class MarketRegime:
    HIGH_VOL = "high_volatility"    # 高波动 → 减仓/对冲
    LOW_VOL  = "low_volatility"     # 低波动 → 趋势跟踪
    CRISIS   = "crisis"             # 危机   → 现金为王

    @staticmethod
    def detect(algorithm):
        vol = algorithm.Securities["SPY"].VolatilityModel.Volatility
        if vol > 0.4:
            return MarketRegime.CRISIS
        elif vol > 0.25:
            return MarketRegime.HIGH_VOL
        else:
            return MarketRegime.LOW_VOL
```

### 5.5 参数优化

```bash
# 利用多核并行加速大规模参数扫描
lean optimize --parallel 8 --max-concurrent-backtests 4
```

**检查点**：能根据不同市场环境切换策略行为，策略不再"一刀切"。

---

## Phase 6：实盘就绪（持续迭代）

> **目标**：将策略从回测环境推进到模拟/实盘，建立持续监控体系。

### 6.1 渐进式实盘部署（报告推荐）

```
小资金实盘(1%) → 监控1个月 → 中资金(10%) → 监控1季度 → 全量(100%)
```

### 6.2 实盘前检查清单

- [ ] 策略已在样本外数据上验证通过
- [ ] 已设置最大回撤熔断
- [ ] 已配置经纪商费用模型（与实盘一致）
- [ ] 订单超时与重连逻辑已实现
- [ ] 每日盈亏报告自动生成
- [ ] 异常检测规则已配置

### 6.3 压力测试场景

至少对以下场景进行回测：
- 2008 年金融危机
- 2020 年疫情暴跌
- 闪崩场景（2010 Flash Crash）

```python
# 伪代码示例（来自报告）
def StressTest(self, scenarios):
    for scenario in ["2008金融危机", "2020疫情暴跌", "闪电崩盘"]:
        result = self.RunBacktest(scenario)
        if result.MaxDrawdown > 0.4:
            self.Log(f"策略在 {scenario} 中失效，回撤 >40%")
```

### 6.4 建立交易日志文化

- 记录每次异常交易的心理状态和市场环境
- 定期复盘：成功/失败交易的根本原因
- 积累极端市场情境的应对经验

---

## 学习节奏建议

| 阶段 | 时间 | 每周投入 | 核心产出 |
|------|------|----------|----------|
| Phase 0-1 | 第1周 | 5-10小时 | lean backtest 跑通 |
| Phase 2 | 第2周 | 10小时 | 3个可运行策略 |
| Phase 3 | 第3-4周 | 10小时/周 | 1个通过样本外验证的策略 |
| Phase 4 | 第5-7周 | 15小时/周 | 带三层风控的策略 |
| Phase 5 | 第8-15周 | 15小时/周 | 自适应策略框架 |
| Phase 6 | 持续 | 持续 | 模拟盘/小资金实盘运行 |

---

## 关键原则（贯穿全路线）

1. **先跑通再优化** — 不要一开始就追求完美策略
2. **样本外验证是底线** — 永远留一段数据不用于调参
3. **活下来比赚得多重要** — 风控代码比策略代码更关键
4. **成功会掩盖风险** — 策略表现最好的时候，反而是最应该警惕的时候
5. **记录一切** — 交易日志是量化直觉从"暗默知识"变为"显性知识"的唯一路径

---

*路线图生成于 2026-04-24，基于 Lean 分析文档提炼。*
