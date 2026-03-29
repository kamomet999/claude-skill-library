---
name: redis-patterns
description: Redisデータ構造活用パターン。セッション管理、ランキング、Pub/Sub、Luaスクリプト、Streams
category: データベース
command: /redis
version: 1.0.0
tags:
  - redis
  - cache
  - pub-sub
  - session
  - streams
---

# Redis Patterns

Redisのデータ構造を活用した実践的な設計パターン集。

## When to Activate

- キャッシュ戦略を設計するとき
- セッション管理を実装するとき
- リアルタイム機能（Pub/Sub、Streams）を構築するとき
- ランキングやカウンター機能を実装するとき
- 分散ロックや rate limiting を実装するとき

## Steps

### Step 1: 接続セットアップ

```typescript
// ioredis（推奨）
import Redis from 'ioredis'

const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: Number(process.env.REDIS_PORT ?? 6379),
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: 3,
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000)
    return delay
  },
  enableReadyCheck: true,
})

redis.on('error', (err) => {
  console.error('Redis connection error:', err)
})
```

### Step 2: キー命名規則

```
{service}:{entity}:{id}:{field}

例:
  app:user:123:profile
  app:session:abc-def
  app:cache:posts:page:1
  app:rate:api:user:123
  app:lock:order:456
```

## データ構造パターン

### String: キャッシュ + セッション

```typescript
// キャッシュパターン（Cache-Aside）
async function getCachedUser(userId: string): Promise<User> {
  const cacheKey = `app:user:${userId}:profile`
  const cached = await redis.get(cacheKey)

  if (cached) {
    return JSON.parse(cached)
  }

  const user = await db.query.users.findFirst({
    where: eq(users.id, userId),
  })

  if (user) {
    await redis.set(cacheKey, JSON.stringify(user), 'EX', 3600)
  }

  return user
}

// キャッシュ無効化
async function invalidateUserCache(userId: string): Promise<void> {
  await redis.del(`app:user:${userId}:profile`)
}

// セッション管理
async function createSession(userId: string, data: SessionData): Promise<string> {
  const sessionId = crypto.randomUUID()
  const key = `app:session:${sessionId}`
  await redis.set(key, JSON.stringify({ ...data, userId }), 'EX', 86400)
  return sessionId
}
```

### Hash: オブジェクトストレージ

```typescript
// ユーザープロファイル（部分更新が頻繁な場合）
await redis.hset('app:user:123', {
  name: 'Taiki',
  email: 'taiki@example.com',
  loginCount: '42',
})

// 特定フィールドのみ取得
const name = await redis.hget('app:user:123', 'name')

// アトミックなカウンター
await redis.hincrby('app:user:123', 'loginCount', 1)

// 全フィールド取得
const profile = await redis.hgetall('app:user:123')
```

### Sorted Set: ランキング

```typescript
// スコア追加
await redis.zadd('app:leaderboard:weekly', score, playerId)

// トップ10取得（スコア降順）
const top10 = await redis.zrevrange('app:leaderboard:weekly', 0, 9, 'WITHSCORES')

// 特定ユーザーのランク取得
const rank = await redis.zrevrank('app:leaderboard:weekly', playerId)

// スコア範囲で取得
const midRange = await redis.zrangebyscore('app:leaderboard:weekly', 100, 500)

// 期限付きランキング（TTL付き）
const weeklyKey = `app:leaderboard:week:${getWeekNumber()}`
await redis.zadd(weeklyKey, score, playerId)
await redis.expire(weeklyKey, 7 * 86400)
```

### List: キュー

```typescript
// プロデューサー
await redis.lpush('app:queue:emails', JSON.stringify({
  to: 'user@example.com',
  subject: 'Welcome',
  template: 'welcome',
}))

// コンシューマー（ブロッキング）
async function processQueue(): Promise<void> {
  while (true) {
    const result = await redis.brpop('app:queue:emails', 30)
    if (result) {
      const [, message] = result
      const job = JSON.parse(message)
      await sendEmail(job)
    }
  }
}

// 信頼性のあるキュー（RPOPLPUSH）
const job = await redis.rpoplpush('app:queue:pending', 'app:queue:processing')
// 処理完了後
await redis.lrem('app:queue:processing', 1, job)
```

### Set: ユニーク管理

```typescript
// オンラインユーザー追跡
await redis.sadd('app:online', userId)
await redis.srem('app:online', userId)
const onlineCount = await redis.scard('app:online')
const isOnline = await redis.sismember('app:online', userId)

// タグシステム
await redis.sadd(`app:post:${postId}:tags`, 'typescript', 'redis', 'backend')
// 共通タグを持つ投稿
const commonTags = await redis.sinter(`app:post:1:tags`, `app:post:2:tags`)
```

## 高度なパターン

### Rate Limiting（Sliding Window）

```typescript
async function isRateLimited(
  identifier: string,
  maxRequests: number,
  windowSeconds: number
): Promise<boolean> {
  const key = `app:rate:${identifier}`
  const now = Date.now()
  const windowStart = now - windowSeconds * 1000

  const pipeline = redis.pipeline()
  pipeline.zremrangebyscore(key, 0, windowStart)
  pipeline.zadd(key, now, `${now}:${Math.random()}`)
  pipeline.zcard(key)
  pipeline.expire(key, windowSeconds)

  const results = await pipeline.exec()
  const count = results?.[2]?.[1] as number

  return count > maxRequests
}
```

### 分散ロック（Redlock簡易版）

```typescript
async function acquireLock(
  resource: string,
  ttlMs: number
): Promise<string | null> {
  const lockId = crypto.randomUUID()
  const key = `app:lock:${resource}`
  const result = await redis.set(key, lockId, 'PX', ttlMs, 'NX')
  return result === 'OK' ? lockId : null
}

async function releaseLock(resource: string, lockId: string): Promise<boolean> {
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `
  const result = await redis.eval(script, 1, `app:lock:${resource}`, lockId)
  return result === 1
}
```

### Pub/Sub

```typescript
// パブリッシャー
await redis.publish('app:events:order', JSON.stringify({
  type: 'order.created',
  orderId: '123',
  timestamp: Date.now(),
}))

// サブスクライバー（専用接続が必要）
const subscriber = new Redis()
subscriber.subscribe('app:events:order', 'app:events:payment')

subscriber.on('message', (channel, message) => {
  const event = JSON.parse(message)
  handleEvent(channel, event)
})
```

### Streams（永続的メッセージング）

```typescript
// プロデューサー
await redis.xadd('app:stream:orders', '*',
  'action', 'created',
  'orderId', '123',
  'amount', '9800'
)

// コンシューマーグループ作成
await redis.xgroup('CREATE', 'app:stream:orders', 'order-processors', '0', 'MKSTREAM')

// コンシューマー
const messages = await redis.xreadgroup(
  'GROUP', 'order-processors', 'worker-1',
  'COUNT', 10, 'BLOCK', 5000,
  'STREAMS', 'app:stream:orders', '>'
)

// 処理完了ACK
if (messages) {
  for (const [stream, entries] of messages) {
    for (const [id] of entries) {
      await redis.xack('app:stream:orders', 'order-processors', id)
    }
  }
}
```

## Best Practices

| Practice | Do | Don't |
|----------|-----|-------|
| TTL | 必ず設定する | TTLなしで無限に溜める |
| キー名 | コロン区切りで構造化 | 曖昧なキー名 |
| シリアライズ | JSON.stringify | toString() |
| 大きな値 | 圧縮 or 分割 | 1MB超のvalueを直接格納 |
| パイプライン | 複数コマンドをbatch | 1つずつ実行 |
| Pub/Sub | 専用接続 | 通常接続と共有 |
| メモリ | maxmemory-policy設定 | OOM放置 |

## メモリ管理

```bash
# maxmemory設定
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru

# メモリ使用状況
INFO memory
DEBUG OBJECT <key>

# 大きなキーの特定
redis-cli --bigkeys
```

## Related

- Skill: `backend-patterns` - API設計パターン
- Skill: `error-handling-patterns` - リトライ・サーキットブレーカー
- Skill: `microservices-design` - マイクロサービス間通信
