---
name: spread-arbitrage
description: Monitors liquidation events and captures spread arbitrage opportunities between spot and futures markets during volatile moves. Implements the "needle catching" strategy from 0xMumu - opening longs at bottoms and shorts at tops during liquidation cascades.
version: 1.0.0
author: Perp Trading Skills
source: Notes-0xMumu.md
requires:
  - node >= 18.0.0
  - typescript >= 5.0.0
---

# Spread Arbitrage Agent Skill

## Overview

This skill implements a spread arbitrage strategy that monitors for liquidation events during violent market moves and captures the price differential between spot and futures markets. Based on [0xMumu's trading methodology](../Notes-0xMumu.md), it targets 3-5% rebounds in short time windows.

**核心思路**（来自 0xMumu 访谈）：
> "短线时期，很喜欢的操作是暴跌的时候接针，因为这是出现了爆仓，出现了合约和现货的差价，自己希望吃掉这种差价，开多开到低点、开空开到高点"（Notes-0xMumu.md 第 39 行）

### Key Characteristics

| 特性 | 描述 | 来源 |
|------|------|------|
| **入场信号** | ADL/爆仓事件 + 价差超过阈值 | Notes-0xMumu.md 第 39、79 行 |
| **持仓时间** | 15-30 分钟（小时级到日内级） | Notes-0xMumu.md 第 40、49 行 |
| **止盈目标** | 3-5% | Notes-0xMumu.md 第 49 行 |
| **风控方式** | 开仓即带 TP/SL + 移动止损 | Notes-0xMumu.md 第 55 行 |
| **适用行情** | 震荡、下跌 | Notes-0xMumu.md 第 36 行 |
| **弱点** | 单边上涨行情容易失效 | Notes-0xMumu.md 第 60 行 |

---

## Strategy Background

### 策略起源

0xMumu 在 LUNA 事件中经历了爆仓教育后，开始系统性研究合约交易，形成了这套价差套利策略。核心洞察是：

1. **爆仓机制**：暴跌时大量多头爆仓 → 合约价格被砸穿 → 出现合约/现货价差
2. **价差回归**：极端价差是短期失衡，会在几分钟到几十分钟内回归
3. **接针时机**：在爆仓瀑布的最低点（或最高点）入场，吃价差回归的利润

### Entry Logic（入场逻辑）

**条件 1：爆仓事件检测**
- 监控大量 ADL（Auto-Deleveraging）或清算事件
- 爆仓量级超过阈值（如 > $1M USD）
- 来源：Notes-0xMumu.md 第 39 行 "出现了爆仓"

**条件 2：价差异常**
- 合约价格 vs 现货价格差异超过阈值（如 > 0.8%）
- 价差方向与爆仓方向一致
- 来源：Notes-0xMumu.md 第 39 行 "出现了合约和现货的差价"

**条件 3：成交量激增**
- 成交量为基线的 3 倍以上
- 过滤假信号和小波动

**交易方向判断**：
- 多头爆仓 + 合约价格 < 现货价格 → **做多**（抄底）
- 空头爆仓 + 合约价格 > 现货价格 → **做空**（摸顶）

### Exit Logic（出场逻辑）

**止盈条件**（任一触发即出场）：
1. 价格达到 3-5% 目标（默认 4%）- 来源：Notes-0xMumu.md 第 49 行
2. 移动止损触发（盈利 > 2% 后启动，回撤 2% 出场）- 来源：Notes-0xMumu.md 第 55 行
3. 持仓时间超过 20 分钟 - 来源：Notes-0xMumu.md 第 49 行 "短窗口机会"

**止损条件**：
- 止损点位：入场价格的 -1.5%
- 来源：基于 4% 止盈和盈亏比 > 2:1 推算

### Risk Profile（风险特征）

| 风险类型 | 表现 | 应对措施 |
|---------|------|---------|
| **单边上涨** | 喜欢摸顶，容易被拉爆 | 严格止损 + 减少仓位 |
| **假突破** | 价差未回归继续扩大 | 时间止损 + 移动止损 |
| **流动性枯竭** | 价差回归但无法平仓 | 订单簿深度验证 |
| **连续爆仓** | 价差持续扩大 | 冷却期机制 |

**来源**：Notes-0xMumu.md 第 60 行 "单边上涨行情就会失效，因为喜欢摸顶，就会被单边行情拉爆"

---

## Prerequisites

### 环境变量

```bash
# 交易所 API（实盘模式）
EXCHANGE_API_KEY=your_api_key
EXCHANGE_API_SECRET=your_api_secret

# 告警通知（可选）
TELEGRAM_BOT_TOKEN=your_telegram_token
TELEGRAM_CHAT_ID=your_chat_id

# 日志级别
LOG_LEVEL=info                # debug, info, warn, error
```

### 配置文件

创建 `config/local.json` 覆盖默认配置：

```json
{
  "mode": "paper",                  // paper | live
  "exchange": "hyperliquid",
  "symbols": ["BTC/USD", "ETH/USD"],

  "strategy": {
    "spreadThreshold": 0.008,       // 0.8% 价差阈值
    "liquidationThreshold": 1000000, // $1M 爆仓阈值
    "takeProfitPercent": 0.04,      // 4% 止盈
    "stopLossPercent": 0.015,       // 1.5% 止损
    "trailingStopPercent": 0.02,    // 2% 移动止损
    "maxHoldingMinutes": 20         // 20 分钟最大持仓
  },

  "risk": {
    "maxPositions": 2,              // 最多 2 个并发持仓
    "maxCapitalPerTrade": 0.05,     // 单笔最多 5% 资金
    "maxDailyLoss": 0.10            // 每日最大亏损 10%
  }
}
```

---

## Quick Start

### 1. 监控模式（纸面交易）

启动监控，不执行真实交易：

```bash
node /path/to/skills/spread-arbitrage/scripts/monitor.js \
  --mode=paper \
  --symbols=BTC/USD,ETH/USD
```

**输出示例**：
```
[2024-02-12 10:30:45] [INFO] Spread alert: BTC/USD
  Spread: 1.2% (futures: $48500, spot: $48920)
  Liquidations: $2.5M long liquidations in last 5min
  Volume: 5.2x baseline
  Signal confidence: 0.78

[2024-02-12 10:30:46] [INFO] Entry signal generated
  Symbol: BTC/USD
  Side: long
  Entry: $48500
  TP: $50470 (+4.06%)
  SL: $47772.5 (-1.50%)
  Position size: 0.1 BTC ($4850)

[2024-02-12 10:30:46] [INFO] Paper trade executed
  Order ID: paper-20240212-001
  Status: filled
  Filled: 0.1 BTC @ $48500
```

### 2. 实盘模式（需手动确认）

执行真实交易（谨慎使用）：

```bash
node /path/to/skills/spread-arbitrage/scripts/monitor.js \
  --mode=live \
  --symbols=BTC/USD \
  --require-confirmation=true
```

**确认提示**：
```
[CONFIRMATION REQUIRED]
Trade Signal:
  Symbol: BTC/USD
  Side: long
  Entry: $48500
  Position: 0.1 BTC ($4850)
  TP: $50470 (+4.06%)
  SL: $47772.5 (-1.50%)
  Confidence: 0.78

Approve this trade? [y/N]:
```

### 3. 回测模式

测试策略在历史数据上的表现：

```bash
node /path/to/skills/spread-arbitrage/scripts/backtest.js \
  --start=2024-01-01 \
  --end=2024-01-31 \
  --symbols=BTC/USD \
  --output=results/backtest-jan2024.json
```

### 4. 仪表板

启动实时监控仪表板：

```bash
node /path/to/skills/spread-arbitrage/scripts/dashboard.js
```

访问 `http://localhost:3000` 查看：
- 实时价差图表
- 爆仓事件时间线
- 持仓状态
- 性能指标

---

## Configuration Options

### 策略参数（strategy）

| 参数 | 默认值 | 说明 | 来源 |
|------|--------|------|------|
| `spreadThreshold` | 0.008 (0.8%) | 最小价差阈值，低于此值不触发信号 | 0xMumu 经验值 |
| `liquidationThreshold` | 1000000 | 最小爆仓量（USD），过滤小额爆仓 | 经验值，可调整 |
| `takeProfitPercent` | 0.04 (4%) | 止盈目标百分比 | Notes-0xMumu.md 第 49 行 |
| `stopLossPercent` | 0.015 (1.5%) | 止损百分比 | 基于盈亏比 > 2:1 推算 |
| `trailingStopPercent` | 0.02 (2%) | 移动止损距离 | Notes-0xMumu.md 第 55 行 |
| `trailingStopTrigger` | 0.02 (2%) | 触发移动止损的盈利阈值 | 推算 |
| `maxHoldingMinutes` | 20 | 最大持仓时间（分钟） | Notes-0xMumu.md 第 49 行 |
| `positionSizePercent` | 0.03 (3%) | 标准仓位占资金比例 | Notes-0xMumu.md 第 51 行 |
| `minVolumeMultiplier` | 3.0 | 成交量需为基线的倍数 | 过滤假信号 |
| `momentumWindow` | 60 | 动量窗口（秒） | 计算价格速度 |
| `cooldownMinutes` | 10 | 交易冷却期（分钟） | 防止过度交易 |

### 风控参数（risk）

| 参数 | 默认值 | 说明 | 来源 |
|------|--------|------|------|
| `maxPositions` | 2 | 最大并发持仓数 | Notes-0xMumu.md 第 36 行 |
| `maxCapitalPerTrade` | 0.05 (5%) | 单笔最大资金占比 | 风控标准 |
| `maxDailyLoss` | 0.10 (10%) | 每日最大亏损比例 | 熔断机制 |
| `maxLeverage` | 3.0 | 最大杠杆倍数 | Notes-0xMumu.md 第 38 行 |
| `minAccountBalance` | 1000 | 最小账户余额（USD） | 风控下限 |
| `requireConfirmation` | false | 实盘是否需要确认 | 安全开关 |

### 执行参数（execution）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `orderType` | "market" | 订单类型（market/limit） |
| `slippage` | 0.001 (0.1%) | 滑点容忍度 |
| `retryAttempts` | 3 | 失败重试次数 |
| `retryDelay` | 1000 | 重试延迟（毫秒） |
| `timeoutMs` | 10000 | 订单超时（毫秒） |

### 监控参数（monitoring）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `priceUpdateInterval` | 1000 | 价格更新间隔（毫秒） |
| `liquidationPollInterval` | 5000 | 爆仓事件轮询间隔（毫秒） |
| `positionCheckInterval` | 2000 | 持仓检查间隔（毫秒） |
| `enableDashboard` | true | 是否启用仪表板 |
| `dashboardPort` | 3000 | 仪表板端口 |

---

## Architecture

### 模块组成

```
┌─────────────────────────────────────────────────────────────┐
│                    Spread Arbitrage Agent                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────┐     ┌───────────────────┐          │
│  │  Price Monitor    │────▶│  Spread Monitor   │          │
│  │  (WebSocket)      │     │  (Calculator)     │          │
│  └───────────────────┘     └───────────────────┘          │
│                                     │                      │
│  ┌───────────────────┐             │                      │
│  │  Liquidation Mon  │             │                      │
│  │  (Events API)     │─────────────┘                      │
│  └───────────────────┘             │                      │
│                                     ▼                      │
│                          ┌──────────────────┐             │
│                          │ Signal Generator │             │
│                          └──────────────────┘             │
│                                     │                      │
│                                     ▼                      │
│                          ┌──────────────────┐             │
│                          │ Risk Validator   │             │
│                          └──────────────────┘             │
│                                     │                      │
│                          ┌──────────┴──────────┐          │
│                          │                     │          │
│                          ▼                     ▼          │
│                  ┌──────────────┐    ┌──────────────┐    │
│                  │ Paper Trader │    │ Live Trader  │    │
│                  └──────────────┘    └──────────────┘    │
│                          │                     │          │
│                          └──────────┬──────────┘          │
│                                     ▼                      │
│                          ┌──────────────────┐             │
│                          │ Position Manager │             │
│                          └──────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

详细架构设计见 [architecture.md](architecture.md)。

### 数据流向

1. **监控层**：PriceMonitor 订阅价格流 → SpreadMonitor 计算价差 → LiquidationMonitor 检测爆仓
2. **策略层**：SignalGenerator 综合评估 → 生成入场信号
3. **风控层**：RiskValidator 多重检查 → 计算仓位大小
4. **执行层**：根据模式选择 PaperTrader 或 LiveTrader → 提交订单
5. **管理层**：PositionManager 持续监控 → 检查出场条件 → 自动平仓

---

## Usage Patterns

### Pattern 1: 监控和告警

仅监控市场，发现信号时发送告警：

```javascript
const { SpreadArbitrageAgent } = require('./src');

const agent = new SpreadArbitrageAgent({
  mode: 'paper',
  onSignal: (signal) => {
    console.log('Entry signal detected:', signal);
    sendTelegramAlert(signal);  // 发送到 Telegram
  }
});

await agent.start();
```

### Pattern 2: 自动纸面交易

自动执行纸面交易，记录性能：

```javascript
const agent = new SpreadArbitrageAgent({
  mode: 'paper',
  autoExecute: true,
  onTrade: (trade) => {
    console.log('Paper trade executed:', trade);
    saveTradeToDB(trade);
  },
  onExit: (result) => {
    console.log('Position closed:', result);
    updateMetrics(result);
  }
});

await agent.start();
```

### Pattern 3: 半自动实盘（推荐）

生成信号后需要手动确认：

```javascript
const agent = new SpreadArbitrageAgent({
  mode: 'live',
  requireConfirmation: true,
  onSignal: async (signal) => {
    // 发送通知
    await sendTelegramAlert(signal);

    // 等待用户确认
    const confirmed = await askUserConfirmation(signal);
    return confirmed;
  }
});

await agent.start();
```

### Pattern 4: 完全自动实盘（高风险）

**警告**：仅在充分回测和小资金验证后使用！

```javascript
const agent = new SpreadArbitrageAgent({
  mode: 'live',
  requireConfirmation: false,
  autoExecute: true,

  // 额外的安全检查
  onSignal: (signal) => {
    // 自定义过滤逻辑
    if (signal.confidence < 0.7) return false;
    if (signal.liquidationVolume < 5000000) return false;
    return true;
  }
});

await agent.start();
```

---

## Output and Logging

### 交易日志格式

```json
{
  "timestamp": "2024-02-12T10:30:45.123Z",
  "type": "ENTRY",
  "symbol": "BTC/USD",
  "side": "long",
  "entryPrice": 48500,
  "size": 0.1,
  "entryCapital": 4850,

  "signalContext": {
    "spread": 0.012,
    "spreadPercent": 0.012,
    "liquidationVolume": 2500000,
    "liquidationSide": "long",
    "volumeMultiplier": 5.2,
    "confidence": 0.78
  },

  "riskParameters": {
    "takeProfitPrice": 50470,
    "stopLossPrice": 47772.5,
    "trailingStopDistance": 0.02,
    "maxHoldingMinutes": 20
  }
}
```

### 出场日志格式

```json
{
  "timestamp": "2024-02-12 10:45:30.456Z",
  "type": "EXIT",
  "symbol": "BTC/USD",
  "side": "long",
  "exitPrice": 50100,
  "exitReason": "Take profit hit",

  "performance": {
    "entryPrice": 48500,
    "exitPrice": 50100,
    "pnl": 160,
    "pnlPercent": 0.033,
    "holdingMinutes": 14.75,
    "realizedPnL": 160,
    "fees": 4.85
  }
}
```

### 性能指标

```json
{
  "period": "2024-02-01 to 2024-02-12",
  "totalTrades": 45,
  "winningTrades": 33,
  "losingTrades": 12,
  "winRate": 0.73,

  "pnl": {
    "avgProfit": 0.042,
    "avgLoss": -0.015,
    "profitFactor": 2.8,
    "totalPnL": 1240,
    "totalPnLPercent": 0.124
  },

  "risk": {
    "maxDrawdown": -0.08,
    "maxDrawdownUSD": -800,
    "sharpeRatio": 1.85,
    "sortinoRatio": 2.31
  },

  "holding": {
    "avgHoldingMinutes": 16.3,
    "minHoldingMinutes": 5.2,
    "maxHoldingMinutes": 20.0
  }
}
```

---

## Troubleshooting

### 常见问题

#### 1. 没有信号生成

**问题**：运行很久但没有检测到任何信号

**可能原因**：
- 价差阈值设置过高
- 爆仓阈值设置过高
- 市场波动较小

**解决方案**：
```json
// 降低阈值（仅用于测试）
{
  "strategy": {
    "spreadThreshold": 0.003,        // 从 0.8% 降到 0.3%
    "liquidationThreshold": 500000   // 从 $1M 降到 $500K
  }
}
```

**验证**：
```bash
# 检查监控日志
tail -f logs/monitor.log | grep "spread-alert"
```

#### 2. 信号太多（假信号）

**问题**：生成大量信号但胜率很低

**可能原因**：
- 阈值设置过低
- 成交量过滤未生效
- 市场噪音较大

**解决方案**：
```json
{
  "strategy": {
    "spreadThreshold": 0.012,        // 提高到 1.2%
    "minVolumeMultiplier": 5.0,      // 提高到 5 倍
    "liquidationThreshold": 2000000  // 提高到 $2M
  }
}
```

#### 3. WebSocket 断连

**问题**：价格数据流中断

**可能原因**：
- 网络不稳定
- 交易所 API 限流
- 认证失效

**解决方案**：
- 检查网络连接
- 实现自动重连逻辑
- 验证 API Key 有效性

```javascript
// 添加重连逻辑
adapter.on('disconnect', () => {
  console.warn('WebSocket disconnected, reconnecting...');
  setTimeout(() => adapter.reconnect(), 5000);
});
```

#### 4. 订单被拒绝

**问题**：实盘订单提交失败

**可能原因**：
- 余额不足
- 仓位限制
- 交易所风控

**解决方案**：
```bash
# 检查账户余额
node scripts/check-balance.js

# 检查持仓限制
node scripts/check-positions.js
```

#### 5. 止盈止损未触发

**问题**：价格达到目标但未平仓

**可能原因**：
- PositionManager 未运行
- 价格数据延迟
- 订单提交失败

**解决方案**：
```bash
# 检查 PositionManager 日志
tail -f logs/position-manager.log

# 手动平仓（紧急情况）
node scripts/emergency-close-all.js
```

---

## Safety Features

### 1. 熔断机制（Circuit Breaker）

达到每日亏损限制后自动停止交易：

```
Daily loss: -$1000 (10% of capital)
[CIRCUIT BREAKER TRIGGERED]
All trading stopped for today.
Resume time: 2024-02-13 00:00:00 UTC
```

### 2. 仓位限制（Position Limits）

防止过度交易：

```
Current positions: 2/2
[POSITION LIMIT REACHED]
Cannot open new position until existing positions are closed.
```

### 3. 确认模式（Confirmation Mode）

实盘交易需要手动批准：

```
[CONFIRMATION REQUIRED]
Approve this trade? [y/N]:
```

### 4. 冷却期（Cooldown Period）

防止高频交易：

```
Last trade: 2024-02-12 10:30:45 (5 minutes ago)
[COOLDOWN ACTIVE]
Next trade allowed in: 5 minutes
```

### 5. 合理性检查（Sanity Checks）

订单提交前验证：

- 价格合理性（不偏离市场价 > 5%）
- 数量合理性（不超过账户余额）
- 杠杆合理性（不超过配置限制）

---

## Extending the Skill

### 添加新的交易所

1. 创建适配器：`src/adapters/your-exchange.ts`
2. 实现接口：`IExchangeAdapter`
3. 添加配置：`config/exchanges/your-exchange.json`
4. 注册适配器：`src/adapters/index.ts`

示例：

```typescript
// src/adapters/binance.ts
import { IExchangeAdapter, PriceUpdate, LiquidationEvent } from './base';

export class BinanceAdapter extends IExchangeAdapter {
  name = 'binance';

  async connect(): Promise<void> {
    // 实现 Binance WebSocket 连接
  }

  async *subscribePrices(symbols: string[]): AsyncIterator<PriceUpdate> {
    // 实现价格订阅
  }

  // ... 实现其他接口方法
}
```

### 添加自定义信号过滤器

扩展 `SignalGenerator` 类：

```typescript
// src/custom/enhanced-signal-generator.ts
import { SignalGenerator, EntrySignal } from '../strategy/signal-generator';

export class EnhancedSignalGenerator extends SignalGenerator {
  protected async generateSignal(
    price: PriceUpdate,
    liquidation: LiquidationEvent
  ): Promise<EntrySignal | null> {
    const baseSignal = await super.generateSignal(price, liquidation);

    if (!baseSignal) return null;

    // 自定义过滤：只在特定时间段交易
    const hour = new Date().getHours();
    if (hour < 9 || hour > 21) {
      return null;  // 过滤夜间信号
    }

    // 自定义过滤：检查 RSI
    const rsi = await this.calculateRSI(price.symbol);
    if (baseSignal.side === 'long' && rsi > 30) {
      return null;  // 过滤非超卖的做多信号
    }

    return baseSignal;
  }
}
```

---

## Performance Optimization

### 1. 降低延迟

- 使用 WebSocket 订阅价格（而非 REST 轮询）
- 部署到靠近交易所服务器的地区
- 使用连接池复用 HTTP 连接

### 2. 批量处理

```typescript
// 批量处理多个交易对
const symbols = ['BTC/USD', 'ETH/USD', 'SOL/USD'];
await Promise.all(
  symbols.map(symbol => this.priceMonitor.subscribe(symbol))
);
```

### 3. 缓存静态数据

```typescript
// 缓存订单簿深度数据（避免重复请求）
const orderBookCache = new Map();
const cachedOrderBook = orderBookCache.get(symbol);
if (cachedOrderBook && Date.now() - cachedOrderBook.timestamp < 1000) {
  return cachedOrderBook.data;
}
```

---

## Security Considerations

### API Key 管理

- ❌ 不要硬编码 API Key
- ✅ 使用环境变量
- ✅ 使用只读 API Key（仅监控模式）
- ✅ 在交易所启用 IP 白名单
- ✅ 启用 2FA

### 风险控制

- 从小资金开始（如 $1000）
- 纸面交易至少 1 周
- 监控实盘至少 1 周（手动确认）
- 逐步增加资金

### 日志安全

- 不要记录敏感信息（API Key、私钥）
- 定期清理旧日志
- 加密存储交易记录

---

## References

### 策略来源
- [Notes-0xMumu.md](../Notes-0xMumu.md) - 访谈记录（第 39-85 行）

### 技术文档
- [data-interfaces.md](data-interfaces.md) - 数据接口定义
- [architecture.md](architecture.md) - 架构设计
- [strategy-logic.md](strategy-logic.md) - 策略逻辑详解
- [risk-management.md](risk-management.md) - 风控规则
- [configuration.md](configuration.md) - 配置参数详解

### 外部资源
- Hyperliquid 文档：https://hyperliquid.gitbook.io/
- Binance API 文档：https://binance-docs.github.io/apidocs/
- OpenClaw/Moltbot Gateway：Agent 运行时框架

---

## Support

如有问题或建议：
- GitHub Issues：*(待添加)*
- Discord：*(待添加)*
- 文档：[README.md](../README.md)

---

**状态**：🟡 设计文档阶段
**最后更新**：2026-02-12
**版本**：1.0.0
