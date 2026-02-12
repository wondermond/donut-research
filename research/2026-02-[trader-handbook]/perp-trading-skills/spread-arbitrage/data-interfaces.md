# Spread Arbitrage - Data Interfaces

本文档定义了价差套利策略所需的所有数据结构和交易所适配器接口。

## 目录

- [交易所适配器接口](#交易所适配器接口)
- [核心数据结构](#核心数据结构)
- [事件类型](#事件类型)
- [示例实现](#示例实现)

---

## 交易所适配器接口

### IExchangeAdapter（抽象基类）

交易所无关的适配器接口，所有交易所实现必须继承此接口。

```typescript
export abstract class IExchangeAdapter {
  abstract name: string;

  // ========== 连接管理 ==========
  abstract connect(): Promise<void>;
  abstract disconnect(): Promise<void>;
  abstract isConnected(): boolean;

  // ========== 市场数据流 ==========
  abstract subscribePrices(symbols: string[]): AsyncIterator<PriceUpdate>;
  abstract subscribeLiquidations(symbols: string[]): AsyncIterator<LiquidationEvent>;
  abstract unsubscribe(symbols: string[]): Promise<void>;

  // ========== 市场数据查询 ==========
  abstract getOrderBook(symbol: string, depth?: number): Promise<OrderBook>;
  abstract getTicker(symbol: string): Promise<PriceUpdate>;
  abstract getRecentLiquidations(symbol: string, limit?: number): Promise<LiquidationEvent[]>;

  // ========== 交易操作 ==========
  abstract createOrder(order: OrderRequest): Promise<OrderResponse>;
  abstract cancelOrder(orderId: string): Promise<void>;
  abstract modifyOrder(orderId: string, changes: Partial<OrderRequest>): Promise<OrderResponse>;

  // ========== 持仓管理 ==========
  abstract getPosition(symbol: string): Promise<Position | null>;
  abstract getOpenPositions(): Promise<Position[]>;
  abstract closePosition(symbol: string): Promise<void>;

  // ========== 账户信息 ==========
  abstract getBalance(currency?: string): Promise<Balance>;
  abstract getBalances(): Promise<Balance[]>;
  abstract getOpenOrders(symbol?: string): Promise<OrderResponse[]>;
}
```

---

## 核心数据结构

### 1. PriceUpdate（价格更新）

```typescript
interface PriceUpdate {
  symbol: string;           // 交易对，如 "BTC/USD"
  timestamp: number;        // Unix 时间戳（毫秒）

  // 价格数据
  spot: number;             // 现货价格
  futures: number;          // 合约价格
  spread: number;           // 绝对价差（futures - spot）
  spreadPercent: number;    // 百分比价差（spread / spot）

  // 成交量
  volume24h: number;        // 24 小时成交量（USD）

  // 买卖盘口
  bidAsk: {
    spotBid: number;        // 现货买一价
    spotAsk: number;        // 现货卖一价
    futuresBid: number;     // 合约买一价
    futuresAsk: number;     // 合约卖一价
  };
}
```

**示例**：
```json
{
  "symbol": "BTC/USD",
  "timestamp": 1707734445123,
  "spot": 48920,
  "futures": 48500,
  "spread": -420,
  "spreadPercent": -0.0086,
  "volume24h": 3450000000,
  "bidAsk": {
    "spotBid": 48919,
    "spotAsk": 48921,
    "futuresBid": 48499,
    "futuresAsk": 48501
  }
}
```

### 2. LiquidationEvent（爆仓事件）

```typescript
interface LiquidationEvent {
  symbol: string;           // 交易对
  timestamp: number;        // 爆仓时间戳

  // 爆仓信息
  side: 'long' | 'short';   // 爆仓方向
  volume: number;           // 爆仓量（USD 价值）
  price: number;            // 爆仓价格
  quantity: number;         // 爆仓数量（币）

  // 爆仓类型
  type: 'liquidation' | 'adl';  // liquidation: 强平 | adl: 自动减仓
}
```

**示例**：
```json
{
  "symbol": "BTC/USD",
  "timestamp": 1707734400000,
  "side": "long",
  "volume": 2500000,
  "price": 48500,
  "quantity": 51.55,
  "type": "liquidation"
}
```

### 3. OrderBook（订单簿）

```typescript
interface OrderBook {
  symbol: string;
  timestamp: number;
  bids: [number, number][];   // [价格, 数量][]
  asks: [number, number][];   // [价格, 数量][]
}
```

**示例**：
```json
{
  "symbol": "BTC/USD",
  "timestamp": 1707734445123,
  "bids": [
    [48499, 1.5],
    [48498, 2.3],
    [48497, 3.1]
  ],
  "asks": [
    [48501, 1.2],
    [48502, 2.1],
    [48503, 2.8]
  ]
}
```

### 4. Balance（账户余额）

```typescript
interface Balance {
  currency: string;     // 币种，如 "USD", "BTC"
  total: number;        // 总余额
  available: number;    // 可用余额
  locked: number;       // 锁定余额（挂单、持仓保证金等）
}
```

**示例**：
```json
{
  "currency": "USD",
  "total": 10000,
  "available": 8500,
  "locked": 1500
}
```

### 5. Position（持仓）

```typescript
interface Position {
  symbol: string;
  side: 'long' | 'short';
  size: number;             // 持仓数量
  entryPrice: number;       // 开仓均价
  currentPrice: number;     // 当前价格
  pnl: number;              // 未实现盈亏（USD）
  pnlPercent: number;       // 盈亏百分比
  leverage?: number;        // 杠杆倍数
  liquidationPrice?: number; // 强平价格
}
```

**示例**：
```json
{
  "symbol": "BTC/USD",
  "side": "long",
  "size": 0.1,
  "entryPrice": 48500,
  "currentPrice": 49200,
  "pnl": 70,
  "pnlPercent": 0.0144,
  "leverage": 3.0,
  "liquidationPrice": 46800
}
```

### 6. OrderRequest（订单请求）

```typescript
interface OrderRequest {
  symbol: string;
  type: 'market' | 'limit';
  side: 'buy' | 'sell';
  amount: number;           // 订单数量
  price?: number;           // 限价单价格（市价单不需要）

  // 止盈止损
  stopLoss?: number;        // 止损价格
  takeProfit?: number;      // 止盈价格

  // 其他选项
  reduceOnly?: boolean;     // 只减仓（平仓订单）
  postOnly?: boolean;       // 只做 Maker
  timeInForce?: 'GTC' | 'IOC' | 'FOK';  // 订单有效期
}
```

**示例**：
```json
{
  "symbol": "BTC/USD",
  "type": "market",
  "side": "buy",
  "amount": 0.1,
  "stopLoss": 47772.5,
  "takeProfit": 50470,
  "reduceOnly": false
}
```

### 7. OrderResponse（订单响应）

```typescript
interface OrderResponse {
  orderId: string;
  status: 'pending' | 'filled' | 'partially_filled' | 'cancelled';
  filled: number;           // 已成交数量
  remaining: number;        // 剩余数量
  averagePrice?: number;    // 平均成交价
  timestamp: number;        // 订单时间戳
}
```

**示例**：
```json
{
  "orderId": "order-20240212-001",
  "status": "filled",
  "filled": 0.1,
  "remaining": 0,
  "averagePrice": 48502,
  "timestamp": 1707734445500
}
```

---

## 事件类型

### EntrySignal（入场信号）

```typescript
interface EntrySignal {
  symbol: string;
  timestamp: number;
  side: 'long' | 'short';

  // 入场参数
  entryPrice: number;
  positionSize: number;     // 由风控计算

  // 风控参数
  takeProfitPrice: number;
  stopLossPrice: number;
  trailingStopDistance: number;

  // 信号元数据
  confidence: number;       // 0-1 置信度评分
  reason: string;           // 信号原因描述
  spread: number;
  spreadPercent: number;
  liquidationVolume: number;
  volumeMultiplier: number;

  // 可选上下文
  orderBookImbalance?: number;
  priceVelocity?: number;
}
```

### ExitSignal（出场信号）

```typescript
interface ExitSignal {
  symbol: string;
  timestamp: number;
  reason: 'take_profit' | 'stop_loss' | 'trailing_stop' | 'timeout';
  currentPrice: number;
}
```

### TradeResult（交易结果）

```typescript
interface TradeResult {
  // 基础信息
  tradeId: string;
  symbol: string;
  side: 'long' | 'short';

  // 入场
  entryTime: number;
  entryPrice: number;
  entrySize: number;

  // 出场
  exitTime: number;
  exitPrice: number;
  exitReason: string;

  // 盈亏
  pnl: number;
  pnlPercent: number;
  fees: number;
  realizedPnL: number;      // pnl - fees

  // 持仓时长
  holdingMinutes: number;

  // 信号上下文
  signalContext?: EntrySignal;
}
```

---

## 示例实现

### MockAdapter（模拟适配器）

用于测试和纸面交易的模拟实现。

```typescript
export class MockAdapter extends IExchangeAdapter {
  name = 'mock';
  private priceGenerator: PriceGenerator;
  private liquidationGenerator: LiquidationGenerator;

  async connect(): Promise<void> {
    console.log('[MockAdapter] Connected');
  }

  async disconnect(): Promise<void> {
    console.log('[MockAdapter] Disconnected');
  }

  isConnected(): boolean {
    return true;
  }

  async *subscribePrices(symbols: string[]): AsyncIterator<PriceUpdate> {
    while (true) {
      for (const symbol of symbols) {
        yield this.priceGenerator.generate(symbol);
      }
      await sleep(1000);  // 每秒一次
    }
  }

  async *subscribeLiquidations(symbols: string[]): AsyncIterator<LiquidationEvent> {
    while (true) {
      // 随机生成爆仓事件（10% 概率）
      if (Math.random() < 0.1) {
        const symbol = symbols[Math.floor(Math.random() * symbols.length)];
        yield this.liquidationGenerator.generate(symbol);
      }
      await sleep(5000);  // 每 5 秒检查一次
    }
  }

  async getOrderBook(symbol: string, depth = 10): Promise<OrderBook> {
    const price = this.priceGenerator.getCurrentPrice(symbol);
    return {
      symbol,
      timestamp: Date.now(),
      bids: Array.from({ length: depth }, (_, i) => [price - i, 1 + Math.random()]),
      asks: Array.from({ length: depth }, (_, i) => [price + i, 1 + Math.random()])
    };
  }

  async createOrder(order: OrderRequest): Promise<OrderResponse> {
    console.log('[MockAdapter] Order created:', order);
    return {
      orderId: `mock-${Date.now()}`,
      status: 'filled',
      filled: order.amount,
      remaining: 0,
      averagePrice: order.price || this.priceGenerator.getCurrentPrice(order.symbol),
      timestamp: Date.now()
    };
  }

  // ... 其他方法的模拟实现
}
```

---

## 数据验证

### 验证函数

```typescript
// 验证价格更新
function validatePriceUpdate(price: PriceUpdate): boolean {
  if (!price.symbol || !price.timestamp) return false;
  if (price.spot <= 0 || price.futures <= 0) return false;
  if (Math.abs(price.spreadPercent) > 0.5) {
    console.warn('Abnormal spread:', price.spreadPercent);
    return false;  // 价差超过 50% 视为异常
  }
  return true;
}

// 验证爆仓事件
function validateLiquidationEvent(liq: LiquidationEvent): boolean {
  if (!liq.symbol || !liq.timestamp) return false;
  if (!['long', 'short'].includes(liq.side)) return false;
  if (liq.volume <= 0 || liq.price <= 0) return false;
  return true;
}
```

---

## 数据转换

### 通用格式转换

不同交易所的数据格式可能不同，需要转换为统一格式。

```typescript
// Hyperliquid 格式 → 通用格式
function hyperliquidToPriceUpdate(data: HyperliquidPriceData): PriceUpdate {
  return {
    symbol: data.coin,
    timestamp: data.time,
    spot: data.markPx,
    futures: data.lastPx,
    spread: data.lastPx - data.markPx,
    spreadPercent: (data.lastPx - data.markPx) / data.markPx,
    volume24h: data.volume24h,
    bidAsk: {
      spotBid: data.markBid,
      spotAsk: data.markAsk,
      futuresBid: data.bid,
      futuresAsk: data.ask
    }
  };
}

// Binance 格式 → 通用格式
function binanceToPriceUpdate(spot: BinanceSpotData, futures: BinanceFuturesData): PriceUpdate {
  return {
    symbol: `${spot.symbol}/USD`,
    timestamp: spot.timestamp,
    spot: parseFloat(spot.price),
    futures: parseFloat(futures.markPrice),
    spread: parseFloat(futures.markPrice) - parseFloat(spot.price),
    spreadPercent: (parseFloat(futures.markPrice) - parseFloat(spot.price)) / parseFloat(spot.price),
    volume24h: parseFloat(futures.volume),
    bidAsk: {
      spotBid: parseFloat(spot.bidPrice),
      spotAsk: parseFloat(spot.askPrice),
      futuresBid: parseFloat(futures.bidPrice),
      futuresAsk: parseFloat(futures.askPrice)
    }
  };
}
```

---

## 性能优化

### 1. 数据池复用

避免频繁创建对象：

```typescript
class PriceUpdatePool {
  private pool: PriceUpdate[] = [];

  get(): PriceUpdate {
    return this.pool.pop() || this.create();
  }

  release(update: PriceUpdate): void {
    this.pool.push(update);
  }

  private create(): PriceUpdate {
    return {
      symbol: '',
      timestamp: 0,
      spot: 0,
      futures: 0,
      spread: 0,
      spreadPercent: 0,
      volume24h: 0,
      bidAsk: { spotBid: 0, spotAsk: 0, futuresBid: 0, futuresAsk: 0 }
    };
  }
}
```

### 2. 批量查询

减少 API 调用次数：

```typescript
async getBatchPrices(symbols: string[]): Promise<Map<string, PriceUpdate>> {
  const promises = symbols.map(symbol => this.getTicker(symbol));
  const results = await Promise.all(promises);
  return new Map(symbols.map((symbol, i) => [symbol, results[i]]));
}
```

---

**相关文档**：
- [SKILL.md](SKILL.md) - 完整 Skill 文档
- [architecture.md](architecture.md) - 架构设计
- [strategy-logic.md](strategy-logic.md) - 策略逻辑

**最后更新**：2026-02-12
