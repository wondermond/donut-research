# Spread Arbitrage - Configuration Parameters

本文档详细说明价差套利策略的所有可配置参数，包括默认值、来源追溯和调优建议。

## 目录

- [配置文件结构](#配置文件结构)
- [策略参数](#策略参数)
- [风控参数](#风控参数)
- [执行参数](#执行参数)
- [监控参数](#监控参数)
- [参数调优建议](#参数调优建议)

---

## 配置文件结构

### 默认配置（config/default.json）

```json
{
  "mode": "paper",
  "exchange": "mock",
  "symbols": ["BTC/USD", "ETH/USD"],

  "strategy": {
    "spreadThreshold": 0.008,
    "liquidationThreshold": 1000000,
    "takeProfitPercent": 0.04,
    "stopLossPercent": 0.015,
    "trailingStopPercent": 0.02,
    "trailingStopTrigger": 0.02,
    "maxHoldingMinutes": 20,
    "positionSizePercent": 0.03,
    "minVolumeMultiplier": 3.0,
    "momentumWindow": 60,
    "cooldownMinutes": 10
  },

  "risk": {
    "maxPositions": 2,
    "maxCapitalPerTrade": 0.05,
    "maxDailyLoss": 0.10,
    "maxLeverage": 3.0,
    "requirePositiveBalance": true,
    "minAccountBalance": 1000,
    "requireConfirmation": false
  },

  "execution": {
    "orderType": "market",
    "slippage": 0.001,
    "retryAttempts": 3,
    "retryDelay": 1000,
    "timeoutMs": 10000
  },

  "monitoring": {
    "priceUpdateInterval": 1000,
    "liquidationPollInterval": 5000,
    "positionCheckInterval": 2000,
    "enableDashboard": true,
    "dashboardPort": 3000
  },

  "logging": {
    "level": "info",
    "outputDir": "./logs",
    "enableFileLogging": true,
    "enableConsoleLogging": true,
    "structured": true
  },

  "alerts": {
    "enableTelegram": false,
    "enableDiscord": false,
    "onSignal": true,
    "onEntry": true,
    "onExit": true,
    "onError": true
  }
}
```

---

## 策略参数

### spreadThreshold（价差阈值）

- **默认值**：`0.008` (0.8%)
- **类型**：`number` (百分比，小数形式)
- **说明**：合约/现货价差必须超过此阈值才会触发信号
- **来源**：0xMumu 经验值
- **调优建议**：
  - 降低（如 0.003）→ 信号更多但假信号增加
  - 提高（如 0.012）→ 信号更少但质量更高
  - 建议回测不同值，找到胜率和信号频率的平衡点

**示例**：
```json
{
  "strategy": {
    "spreadThreshold": 0.008  // 0.8%
  }
}
```

### liquidationThreshold（爆仓阈值）

- **默认值**：`1000000` (1M USD)
- **类型**：`number` (美元)
- **说明**：爆仓量必须超过此阈值才会触发信号，过滤小额爆仓
- **来源**：经验值，可调整
- **调优建议**：
  - 降低（如 500000）→ 捕捉更多中小型爆仓事件
  - 提高（如 2000000）→ 只关注大型爆仓，信号更可靠
  - 根据交易对流动性调整（BTC/ETH 可以提高，山寨币需要降低）

**示例**：
```json
{
  "strategy": {
    "liquidationThreshold": 1000000  // $1M
  }
}
```

### takeProfitPercent（止盈百分比）

- **默认值**：`0.04` (4%)
- **类型**：`number` (百分比，小数形式)
- **说明**：目标盈利百分比，达到后自动平仓
- **来源**：Notes-0xMumu.md 第 49 行 "可以吃3-5%"
- **访谈原文**：
  > "有耐心等待最好买点的交易员，喜欢在暴跌里寻找机会，比如等 ADL等，因为暴跌会有极短时间的反弹，可以吃3-5%"
- **调优建议**：
  - 3-5% 是合理区间，基于历史价差回归速度
  - 如果持仓时间经常超时，可以降低止盈目标
  - 如果经常被洗盘，可以提高止盈目标

**示例**：
```json
{
  "strategy": {
    "takeProfitPercent": 0.04  // 4%
  }
}
```

### stopLossPercent（止损百分比）

- **默认值**：`0.015` (1.5%)
- **类型**：`number` (百分比，小数形式)
- **说明**：最大亏损百分比，达到后强制平仓
- **来源**：基于 4% 止盈和盈亏比 > 2:1 推算
- **推算依据**：
  - 目标盈亏比：2.67:1 (4% / 1.5%)
  - 胜率 73% 时期望收益为正
- **调优建议**：
  - 严格执行止损，不要扩大止损幅度
  - 如果频繁止损，检查入场信号质量而非放宽止损

**示例**：
```json
{
  "strategy": {
    "stopLossPercent": 0.015  // 1.5%
  }
}
```

### trailingStopPercent（移动止损距离）

- **默认值**：`0.02` (2%)
- **类型**：`number` (百分比，小数形式)
- **说明**：启动移动止损后，价格回撤此百分比即平仓
- **来源**：Notes-0xMumu.md 第 55 行
- **访谈原文**：
  > "如果脱离了成本区间，会设移动止损"
- **调优建议**：
  - 太小（如 1%）→ 容易被短期波动洗盘
  - 太大（如 3%）→ 回吐过多利润
  - 建议 1.5-2.5% 区间

**示例**：
```json
{
  "strategy": {
    "trailingStopPercent": 0.02,       // 2% 回撤平仓
    "trailingStopTrigger": 0.02        // 盈利 >2% 启动
  }
}
```

### maxHoldingMinutes（最大持仓时间）

- **默认值**：`20` (分钟)
- **类型**：`number` (分钟)
- **说明**：超过此时间后强制平仓，防止价差长期不回归
- **来源**：Notes-0xMumu.md 第 49 行 "短窗口机会"
- **访谈原文**：
  > "暴跌会有极短时间的反弹，可以吃3-5%"（强调短时间窗口）
- **调优建议**：
  - 回测平均持仓时间，设为平均值的 1.5-2 倍
  - 如果经常超时，检查止盈目标是否过高

**示例**：
```json
{
  "strategy": {
    "maxHoldingMinutes": 20  // 20 分钟
  }
}
```

### positionSizePercent（仓位大小百分比）

- **默认值**：`0.03` (3%)
- **类型**：`number` (百分比，小数形式)
- **说明**：标准仓位占总资金的比例
- **来源**：Notes-0xMumu.md 第 51 行，山寨币 5000U 占比推算
- **访谈原文**：
  > "山寨币会更长，因为仓位小，控制5000u 20倍以内，这是能够忍受的亏损上限"
- **推算**：假设总资金 10 万 U，5000U 占 5%，考虑主流币风险更低，降到 3%
- **调优建议**：
  - 新手建议 1-2%
  - 有经验后可提升到 3-5%
  - 绝不超过 10%

**示例**：
```json
{
  "strategy": {
    "positionSizePercent": 0.03  // 3% 资金
  }
}
```

### minVolumeMultiplier（成交量倍数）

- **默认值**：`3.0`
- **类型**：`number`
- **说明**：成交量必须是基线的此倍数才触发信号，过滤低波动假信号
- **来源**：经验值
- **调优建议**：
  - 降低（如 2.0）→ 捕捉更多信号但噪音增加
  - 提高（如 5.0）→ 只在极端波动时交易
  - 根据交易对特性调整

**示例**：
```json
{
  "strategy": {
    "minVolumeMultiplier": 3.0  // 3 倍基线成交量
  }
}
```

### cooldownMinutes（冷却期）

- **默认值**：`10` (分钟)
- **类型**：`number` (分钟)
- **说明**：两次交易之间的最小间隔，防止过度交易
- **来源**：防止过度交易的风控措施
- **调优建议**：
  - 如果信号频繁，可以提高到 15-20 分钟
  - 如果信号稀少，可以降低到 5 分钟

**示例**：
```json
{
  "strategy": {
    "cooldownMinutes": 10  // 10 分钟冷却
  }
}
```

---

## 风控参数

### maxPositions（最大持仓数）

- **默认值**：`2`
- **类型**：`number`
- **说明**：同时允许的最大持仓数量
- **来源**：Notes-0xMumu.md 第 36 行
- **访谈原文**：
  > "为什么适合做震荡：和仓位管理有关系，如果仓位很重，以及杠杆很高，就很难长时间拿，因为会时时刻刻盯，导致涨跌一点点就会跑"
- **调优建议**：
  - 短线策略建议 1-3 个
  - 超过 3 个容易分散注意力，管理困难

**示例**：
```json
{
  "risk": {
    "maxPositions": 2
  }
}
```

### maxCapitalPerTrade（单笔最大资金占比）

- **默认值**：`0.05` (5%)
- **类型**：`number` (百分比，小数形式)
- **说明**：单笔交易最多使用总资金的此百分比
- **来源**：风控标准
- **调优建议**：
  - 保守型：3%
  - 平衡型：5%
  - 激进型：7%（不建议超过 10%）

**示例**：
```json
{
  "risk": {
    "maxCapitalPerTrade": 0.05  // 5% 上限
  }
}
```

### maxDailyLoss（每日最大亏损）

- **默认值**：`0.10` (10%)
- **类型**：`number` (百分比，小数形式)
- **说明**：当日亏损达到此比例后触发熔断，停止交易
- **来源**：熔断机制
- **调优建议**：
  - 严格执行，达到后立即停止
  - 建议 8-12% 区间
  - 新手可设为 5%

**示例**：
```json
{
  "risk": {
    "maxDailyLoss": 0.10  // 10% 熔断
  }
}
```

### maxLeverage（最大杠杆倍数）

- **默认值**：`3.0`
- **类型**：`number`
- **说明**：账户总杠杆不能超过此倍数
- **来源**：Notes-0xMumu.md 第 38 行
- **访谈原文**：
  > "仓位管理说的是，占自己的仓位比例。自己更关注实际杠杆"
- **调优建议**：
  - 价差套利是相对低风险策略，2-3 倍杠杆合理
  - 不建议超过 5 倍

**示例**：
```json
{
  "risk": {
    "maxLeverage": 3.0
  }
}
```

### minAccountBalance（最小账户余额）

- **默认值**：`1000` (USD)
- **类型**：`number` (美元)
- **说明**：余额低于此值后停止交易
- **来源**：风控下限
- **调优建议**：
  - 根据交易所最小订单量和预期仓位设置
  - 建议至少能开 5-10 单的金额

**示例**：
```json
{
  "risk": {
    "minAccountBalance": 1000
  }
}
```

### requireConfirmation（需要手动确认）

- **默认值**：`false`
- **类型**：`boolean`
- **说明**：实盘模式下是否需要手动确认每笔交易
- **来源**：安全开关
- **调优建议**：
  - 初期建议设为 `true`
  - 充分验证后可设为 `false`

**示例**：
```json
{
  "risk": {
    "requireConfirmation": true  // 实盘需确认
  }
}
```

---

## 参数调优建议

### 回测流程

1. **基准测试**：使用默认参数回测 3-6 个月历史数据
2. **单参数扫描**：每次只改变一个参数，观察影响
3. **组合优化**：找到关键参数后，进行组合优化
4. **稳健性测试**：在不同市场条件下验证

### 优化目标

| 指标 | 目标值 | 权重 |
|------|--------|------|
| **胜率** | > 65% | 高 |
| **盈亏比** | > 2.0 | 高 |
| **最大回撤** | < 20% | 高 |
| **年化收益** | > 50% | 中 |
| **夏普比率** | > 1.5 | 中 |
| **信号频率** | 适中 | 低 |

### 参数敏感性

**高敏感**（需要谨慎调整）：
- `takeProfitPercent`
- `stopLossPercent`
- `spreadThreshold`

**中敏感**（可适度调整）：
- `liquidationThreshold`
- `maxHoldingMinutes`
- `positionSizePercent`

**低敏感**（可自由调整）：
- `minVolumeMultiplier`
- `cooldownMinutes`

---

**相关文档**：
- [SKILL.md](SKILL.md) - 完整 Skill 文档
- [strategy-logic.md](strategy-logic.md) - 策略逻辑
- [risk-management.md](risk-management.md) - 风控规则

**最后更新**：2026-02-12
