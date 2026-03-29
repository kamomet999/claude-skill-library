---
name: microservices-design
description: マイクロサービス設計パターン。サービス分割、通信パターン、Saga、CQRS、イベント駆動アーキテクチャ
category: アーキテクチャ
command: /microservices
version: 1.0.0
tags:
  - microservices
  - saga
  - cqrs
  - event-driven
  - ddd
---

# Microservices Design Patterns

マイクロサービスアーキテクチャの設計・実装パターン集。

## When to Activate

- モノリスからマイクロサービスへの分割を計画するとき
- サービス間通信パターンを選定するとき
- 分散トランザクション（Saga）を設計するとき
- CQRS/Event Sourcingを導入するとき
- イベント駆動アーキテクチャを設計するとき

## Steps

### Step 1: サービス分割判断

```
モノリスのままでいいケース:
- チーム5人以下
- ドメインが単純
- トラフィックが均一
- 独立デプロイの必要なし

マイクロサービスにすべきケース:
- チーム10人以上で独立開発したい
- ドメインごとにスケーリング要件が異なる
- 異なる技術スタックが必要
- 障害分離が重要
```

### Step 2: ドメイン境界の特定（DDD）

```
Bounded Context の特定:
1. イベントストーミングでドメインイベントを洗い出す
2. アグリゲートをグループ化
3. コンテキスト境界を定義
4. コンテキスト間の関係を明確化

例: ECサイト
┌─────────────┐  ┌──────────────┐  ┌─────────────┐
│  注文サービス  │  │  在庫サービス   │  │ 決済サービス  │
│  - Order     │  │  - Product    │  │ - Payment   │
│  - OrderItem │  │  - Stock      │  │ - Refund    │
│  - Cart      │  │  - Warehouse  │  │ - Invoice   │
└──────┬───────┘  └──────┬───────┘  └──────┬──────┘
       │                 │                  │
       └────── Events ───┴──── Events ──────┘
```

### Step 3: 通信パターン選定

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| REST (sync) | CRUD、クエリ | シンプル、標準的 | カップリング、連鎖障害 |
| gRPC (sync) | 内部通信、高性能 | 型安全、高速 | 学習コスト |
| Message Queue (async) | イベント通知、ジョブ | 疎結合、信頼性 | 複雑性、最終整合性 |
| Event Streaming (async) | イベントログ、CQRS | 監査、リプレイ | ストレージ、複雑性 |

## Communication Patterns

### API Gateway

```typescript
// API Gateway（Express + httpProxy）
import express from 'express'
import { createProxyMiddleware } from 'http-proxy-middleware'

const app = express()

// 認証ミドルウェア
app.use(authMiddleware)

// ルーティング
app.use('/api/orders', createProxyMiddleware({
  target: 'http://order-service:3001',
  pathRewrite: { '^/api/orders': '' },
}))

app.use('/api/products', createProxyMiddleware({
  target: 'http://product-service:3002',
  pathRewrite: { '^/api/products': '' },
}))

app.use('/api/payments', createProxyMiddleware({
  target: 'http://payment-service:3003',
  pathRewrite: { '^/api/payments': '' },
}))

// Rate limiting
app.use(rateLimit({ windowMs: 60000, max: 100 }))
```

### Service-to-Service（gRPC）

```protobuf
// proto/inventory.proto
syntax = "proto3";

service InventoryService {
  rpc CheckStock (CheckStockRequest) returns (CheckStockResponse);
  rpc ReserveStock (ReserveStockRequest) returns (ReserveStockResponse);
}

message CheckStockRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message CheckStockResponse {
  bool available = 1;
  int32 current_stock = 2;
}
```

### Event-Driven（メッセージブローカー）

```typescript
// イベント定義
interface DomainEvent {
  eventId: string
  eventType: string
  aggregateId: string
  timestamp: string
  payload: unknown
  metadata: {
    correlationId: string
    causationId: string
    version: number
  }
}

// イベント発行
async function publishEvent(event: DomainEvent): Promise<void> {
  await messageQueue.publish(event.eventType, {
    ...event,
    eventId: crypto.randomUUID(),
    timestamp: new Date().toISOString(),
  })
}

// 注文作成時のイベント
await publishEvent({
  eventId: '',
  eventType: 'order.created',
  aggregateId: orderId,
  timestamp: '',
  payload: { orderId, items, totalAmount },
  metadata: {
    correlationId: requestId,
    causationId: commandId,
    version: 1,
  },
})
```

## Saga Pattern（分散トランザクション）

### Choreography Saga（イベント駆動）

```
注文サービス → order.created
    ↓
在庫サービス → stock.reserved（成功） / stock.reservation_failed（失敗）
    ↓
決済サービス → payment.completed（成功） / payment.failed（失敗）
    ↓
注文サービス → order.confirmed / order.cancelled

補償トランザクション:
  payment.failed → 在庫サービス: stock.released
  stock.reservation_failed → 注文サービス: order.cancelled
```

### Orchestration Saga（オーケストレーター）

```typescript
// saga/create-order.saga.ts
interface SagaStep {
  name: string
  execute: (context: SagaContext) => Promise<void>
  compensate: (context: SagaContext) => Promise<void>
}

class CreateOrderSaga {
  private readonly steps: SagaStep[] = [
    {
      name: 'reserve_stock',
      execute: async (ctx) => {
        const result = await inventoryService.reserveStock(ctx.items)
        return { ...ctx, reservationId: result.id }
      },
      compensate: async (ctx) => {
        await inventoryService.releaseStock(ctx.reservationId)
      },
    },
    {
      name: 'process_payment',
      execute: async (ctx) => {
        const result = await paymentService.charge(ctx.totalAmount)
        return { ...ctx, paymentId: result.id }
      },
      compensate: async (ctx) => {
        await paymentService.refund(ctx.paymentId)
      },
    },
    {
      name: 'confirm_order',
      execute: async (ctx) => {
        await orderService.confirm(ctx.orderId)
      },
      compensate: async (ctx) => {
        await orderService.cancel(ctx.orderId)
      },
    },
  ]

  async execute(context: SagaContext): Promise<SagaResult> {
    const completedSteps: SagaStep[] = []

    try {
      for (const step of this.steps) {
        await step.execute(context)
        completedSteps.push(step)
      }
      return { success: true }
    } catch (error) {
      // 補償トランザクション（逆順で実行）
      for (const step of completedSteps.reverse()) {
        try {
          await step.compensate(context)
        } catch (compensateError) {
          console.error(`Compensation failed for ${step.name}:`, compensateError)
          // Dead letter queueに送信
        }
      }
      return { success: false, error: String(error) }
    }
  }
}
```

## CQRS Pattern

```
Command側（書き込み）:
  Client → API → Command Handler → Domain Model → Event Store → Event Bus
                                                                     ↓
Query側（読み込み）:                                           Event Handler
  Client → API → Query Handler → Read Model (Materialized View)  ↑
                                                                  │
                                                          Read DB Updated
```

```typescript
// Command側
interface CreateOrderCommand {
  type: 'CreateOrder'
  userId: string
  items: OrderItem[]
}

async function handleCreateOrder(cmd: CreateOrderCommand): Promise<string> {
  const order = Order.create(cmd.userId, cmd.items)
  await eventStore.append(order.id, order.uncommittedEvents)
  return order.id
}

// Query側（Materialized View）
interface OrderSummaryView {
  orderId: string
  userName: string
  totalAmount: number
  status: string
  itemCount: number
}

// イベントハンドラーでRead Modelを更新
async function onOrderCreated(event: OrderCreatedEvent): Promise<void> {
  await readDb.insert('order_summaries', {
    orderId: event.aggregateId,
    userName: event.payload.userName,
    totalAmount: event.payload.totalAmount,
    status: 'created',
    itemCount: event.payload.items.length,
  })
}
```

## サービス間の回復力パターン

### Circuit Breaker

```typescript
enum CircuitState {
  CLOSED = 'CLOSED',       // 正常
  OPEN = 'OPEN',           // 遮断中
  HALF_OPEN = 'HALF_OPEN', // テスト中
}

class CircuitBreaker {
  private state = CircuitState.CLOSED
  private failureCount = 0
  private lastFailureTime = 0

  constructor(
    private readonly threshold: number = 5,
    private readonly resetTimeoutMs: number = 30000,
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.state = CircuitState.HALF_OPEN
      } else {
        throw new Error('Circuit breaker is OPEN')
      }
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess(): void {
    this.failureCount = 0
    this.state = CircuitState.CLOSED
  }

  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = Date.now()
    if (this.failureCount >= this.threshold) {
      this.state = CircuitState.OPEN
    }
  }
}
```

## ヘルスチェック

```typescript
// 各サービスに必須
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDb(),
    redis: await checkRedis(),
    messageQueue: await checkMQ(),
  }

  const healthy = Object.values(checks).every((c) => c.status === 'up')

  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks,
    timestamp: new Date().toISOString(),
  })
})
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| 分散モノリス | 全サービスが同時デプロイ必要 | 境界を見直し、APIバージョニング |
| 共有DB | サービス間がDB結合 | サービスごとにDB分離 |
| 同期チェーン | A→B→C→Dの連鎖呼び出し | 非同期イベント + キャッシュ |
| 過度な分割 | 1関数=1サービス | ドメイン境界で分割 |
| イベントスープ | イベントが乱立 | イベントカタログ整備 |

## Best Practices

- サービスは独立してデプロイ・スケール可能にする
- DBはサービスごとに分離（共有DBは絶対避ける）
- 冪等性を全エンドポイントで保証する
- Correlation IDで分散トレーシングを実現する
- API契約をスキーマ（OpenAPI/Protobuf）で管理する
- 障害を前提に設計する（タイムアウト、リトライ、フォールバック）

## Related

- Skill: `error-handling-patterns` - リトライ・サーキットブレーカー
- Skill: `redis-patterns` - キャッシュ・メッセージング
- Agent: `architect` - アーキテクチャ設計
