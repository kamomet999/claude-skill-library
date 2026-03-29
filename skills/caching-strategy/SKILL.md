---
name: caching-strategy
description: キャッシュ戦略設計。Redis、CDN、ブラウザキャッシュ、Cache-Aside、Write-Through、TTL
category: バックエンド
command: /caching-strategy
version: 1.0.0
tags: [cache, redis, cdn, performance, ttl]
---

# キャッシュ戦略設計パターン

## 概要

アプリケーションのパフォーマンスを最大化するためのキャッシュ戦略パターン集。
Redis、CDN、ブラウザキャッシュ、各種キャッシュパターン（Cache-Aside、Write-Through、Write-Behind）、TTL設計、キャッシュ無効化を網羅する。

## Steps

1. キャッシュ対象を特定する（読み取り頻度 vs 更新頻度）
2. キャッシュレイヤーを決定する（ブラウザ / CDN / アプリ / DB）
3. キャッシュパターンを選択する
4. TTL戦略を設計する
5. キャッシュ無効化戦略を実装する
6. Redisクライアントを設定する
7. HTTPキャッシュヘッダーを設定する
8. モニタリングとヒット率の追跡を行う

## キャッシュレイヤーの階層

```
ユーザー
  ↓
[ブラウザキャッシュ]        ← Cache-Control, ETag
  ↓
[CDN / エッジキャッシュ]    ← CloudFront, Cloudflare
  ↓
[APIゲートウェイ]           ← レスポンスキャッシュ
  ↓
[アプリケーションキャッシュ]  ← Redis, Memcached
  ↓
[データベース]              ← クエリキャッシュ
```

## キャッシュパターンの選択

| パターン | 読み取り | 書き込み | ユースケース |
|---------|---------|---------|-------------|
| Cache-Aside | アプリがキャッシュ確認→ミスならDB | アプリがDB更新→キャッシュ無効化 | 汎用、最も一般的 |
| Read-Through | キャッシュがDB読み込みを代行 | - | ORM統合 |
| Write-Through | - | キャッシュとDBを同時更新 | データ一貫性重視 |
| Write-Behind | - | キャッシュ更新→非同期でDB | 高書き込み頻度 |
| Refresh-Ahead | TTL切れ前に非同期更新 | - | 予測可能なアクセス |

## Redis接続設定

```typescript
// src/cache/redis-client.ts
import { Redis, Cluster } from 'ioredis'

function createRedisClient(): Redis | Cluster {
  const isCluster = process.env.REDIS_CLUSTER === 'true'

  if (isCluster) {
    return new Cluster(
      JSON.parse(process.env.REDIS_CLUSTER_NODES ?? '[]'),
      {
        redisOptions: { password: process.env.REDIS_PASSWORD },
        scaleReads: 'slave',  // 読み取りはレプリカから
      }
    )
  }

  return new Redis({
    host: process.env.REDIS_HOST ?? 'localhost',
    port: Number(process.env.REDIS_PORT) ?? 6379,
    password: process.env.REDIS_PASSWORD,
    maxRetriesPerRequest: 3,
    retryStrategy: (times) => Math.min(times * 100, 3000),
    enableReadyCheck: true,
    lazyConnect: true,
  })
}

export const redis = createRedisClient()
```

## Cache-Aside パターン（推奨デフォルト）

```typescript
// src/cache/cache-aside.ts
import { redis } from './redis-client'

interface CacheOptions {
  ttl: number           // 秒
  prefix?: string
  serialize?: (data: unknown) => string
  deserialize?: (raw: string) => unknown
}

const defaultOptions: CacheOptions = {
  ttl: 300,  // 5分
  serialize: JSON.stringify,
  deserialize: JSON.parse,
}

export async function cacheAside<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: Partial<CacheOptions> = {}
): Promise<T> {
  const opts = { ...defaultOptions, ...options }
  const cacheKey = opts.prefix ? `${opts.prefix}:${key}` : key

  // 1. キャッシュから取得を試みる
  const cached = await redis.get(cacheKey)
  if (cached !== null) {
    return opts.deserialize!(cached) as T
  }

  // 2. キャッシュミス -> データソースから取得
  const data = await fetcher()

  // 3. キャッシュに保存（nullはキャッシュしない: ネガティブキャッシュが必要なら別途）
  if (data !== null && data !== undefined) {
    await redis.setex(cacheKey, opts.ttl, opts.serialize!(data))
  }

  return data
}

// 使用例
export async function getUserById(id: string) {
  return cacheAside(
    `user:${id}`,
    () => db.user.findUnique({ where: { id } }),
    { ttl: 600, prefix: 'app' }  // 10分キャッシュ
  )
}
```

## キャッシュ無効化

```typescript
// src/cache/invalidation.ts
import { redis } from './redis-client'

// 単一キーの無効化
export async function invalidateCache(key: string): Promise<void> {
  await redis.del(key)
}

// パターンマッチで一括無効化
export async function invalidatePattern(pattern: string): Promise<void> {
  // SCAN で安全にキーを取得（KEYSは本番で使わない）
  let cursor = '0'
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100)
    cursor = nextCursor
    if (keys.length > 0) {
      await redis.del(...keys)
    }
  } while (cursor !== '0')
}

// タグベースの無効化
export async function tagCache(key: string, tags: string[]): Promise<void> {
  const pipeline = redis.pipeline()
  for (const tag of tags) {
    pipeline.sadd(`tag:${tag}`, key)
  }
  await pipeline.exec()
}

export async function invalidateByTag(tag: string): Promise<void> {
  const keys = await redis.smembers(`tag:${tag}`)
  if (keys.length > 0) {
    await redis.del(...keys)
  }
  await redis.del(`tag:${tag}`)
}

// 使用例: ユーザー更新時
export async function updateUser(id: string, data: UpdateUserInput) {
  const user = await db.user.update({ where: { id }, data })

  // 関連キャッシュを無効化
  await Promise.all([
    invalidateCache(`app:user:${id}`),
    invalidateByTag(`user:${id}`),
    invalidatePattern(`app:user-list:*`),  // リスト系も無効化
  ])

  return user
}
```

## Write-Through パターン

```typescript
// src/cache/write-through.ts
export async function writeThrough<T>(
  key: string,
  data: T,
  writer: (data: T) => Promise<T>,
  ttl: number
): Promise<T> {
  // DBとキャッシュを同時に更新
  const result = await writer(data)
  await redis.setex(key, ttl, JSON.stringify(result))
  return result
}

// 使用例
export async function createProduct(input: CreateProductInput) {
  return writeThrough(
    `product:${input.sku}`,
    input,
    async (data) => db.product.create({ data }),
    3600  // 1時間
  )
}
```

## ネガティブキャッシュ

```typescript
// 存在しないデータへの繰り返しクエリを防ぐ
const NEGATIVE_CACHE_TTL = 60  // 短いTTL（1分）
const NEGATIVE_SENTINEL = '__NULL__'

export async function cacheAsideWithNegative<T>(
  key: string,
  fetcher: () => Promise<T | null>,
  ttl: number
): Promise<T | null> {
  const cached = await redis.get(key)

  if (cached === NEGATIVE_SENTINEL) {
    return null  // ネガティブキャッシュヒット
  }

  if (cached !== null) {
    return JSON.parse(cached) as T
  }

  const data = await fetcher()

  if (data === null) {
    // 存在しないことをキャッシュ（短いTTL）
    await redis.setex(key, NEGATIVE_CACHE_TTL, NEGATIVE_SENTINEL)
    return null
  }

  await redis.setex(key, ttl, JSON.stringify(data))
  return data
}
```

## HTTPキャッシュヘッダー

```typescript
// src/middleware/cache-headers.ts
import { Request, Response, NextFunction } from 'express'
import crypto from 'crypto'

// 静的アセット（変更されない）
export function immutableCache(req: Request, res: Response, next: NextFunction): void {
  res.set('Cache-Control', 'public, max-age=31536000, immutable')
  next()
}

// 動的コンテンツ（再検証あり）
export function revalidateCache(maxAge: number) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.set('Cache-Control', `public, max-age=${maxAge}, must-revalidate`)
    next()
  }
}

// プライベートデータ（CDNキャッシュ禁止）
export function privateCache(maxAge: number) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.set('Cache-Control', `private, max-age=${maxAge}`)
    next()
  }
}

// キャッシュ禁止
export function noCache(req: Request, res: Response, next: NextFunction): void {
  res.set('Cache-Control', 'no-store, no-cache, must-revalidate')
  res.set('Pragma', 'no-cache')
  next()
}

// ETagミドルウェア
export function etag(req: Request, res: Response, next: NextFunction): void {
  const originalJson = res.json.bind(res)

  res.json = (body: unknown) => {
    const content = JSON.stringify(body)
    const hash = crypto.createHash('md5').update(content).digest('hex')
    const etagValue = `"${hash}"`

    res.set('ETag', etagValue)

    // If-None-Matchヘッダーと比較
    if (req.headers['if-none-match'] === etagValue) {
      return res.status(304).end()
    }

    return originalJson(body)
  }

  next()
}

// 使用例
// router.get('/products', revalidateCache(300), etag, listProducts)
// router.get('/me', privateCache(60), getProfile)
// router.post('/orders', noCache, createOrder)
```

## TTL設計ガイドライン

```typescript
const TTL = {
  // 超短期（リアルタイム性が必要）
  REALTIME: 5,              // 5秒: 在庫数、レート

  // 短期（頻繁に更新される）
  SHORT: 60,                // 1分: 検索結果、フィード

  // 中期（まあまあ更新される）
  MEDIUM: 300,              // 5分: ユーザープロフィール

  // 長期（あまり更新されない）
  LONG: 3600,               // 1時間: 商品詳細、カテゴリ一覧

  // 超長期（ほぼ不変）
  PERMANENT: 86400,         // 24時間: マスタデータ、設定

  // ネガティブキャッシュ
  NEGATIVE: 60,             // 1分: 存在しないデータ
} as const
```

## キャッシュウォーミング

```typescript
// アプリ起動時やデプロイ時にキャッシュを事前投入
export async function warmCache(): Promise<void> {
  console.info('Cache warming started...')

  // 人気商品トップ100をキャッシュ
  const popularProducts = await db.product.findMany({
    orderBy: { viewCount: 'desc' },
    take: 100,
  })

  const pipeline = redis.pipeline()
  for (const product of popularProducts) {
    pipeline.setex(
      `product:${product.id}`,
      TTL.LONG,
      JSON.stringify(product)
    )
  }
  await pipeline.exec()

  // カテゴリマスタをキャッシュ
  const categories = await db.category.findMany()
  await redis.setex('categories:all', TTL.PERMANENT, JSON.stringify(categories))

  console.info(`Cache warmed: ${popularProducts.length} products, ${categories.length} categories`)
}
```

## キャッシュスタンピード防止

```typescript
// 同時に多数のリクエストがキャッシュミスする問題を防ぐ
import Redlock from 'redlock'

const redlock = new Redlock([redis])

export async function cacheAsideWithLock<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number
): Promise<T> {
  // 1. キャッシュ確認
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached) as T

  // 2. ロックを取得（1つのプロセスだけがDBにアクセス）
  const lockKey = `lock:${key}`
  try {
    const lock = await redlock.acquire([lockKey], 5000)

    try {
      // ロック取得後に再度確認（別プロセスが先にキャッシュした可能性）
      const rechecked = await redis.get(key)
      if (rechecked) return JSON.parse(rechecked) as T

      // DBから取得してキャッシュ
      const data = await fetcher()
      await redis.setex(key, ttl, JSON.stringify(data))
      return data
    } finally {
      await lock.release()
    }
  } catch {
    // ロック取得失敗 -> 短時間待って再試行
    await new Promise((resolve) => setTimeout(resolve, 100))
    const retried = await redis.get(key)
    if (retried) return JSON.parse(retried) as T

    // それでもなければ直接DB
    return fetcher()
  }
}
```

## モニタリング

```typescript
// src/cache/metrics.ts
import { redis } from './redis-client'

interface CacheMetrics {
  hits: number
  misses: number
  hitRate: number
  keyCount: number
  memoryUsage: string
}

export async function getCacheMetrics(): Promise<CacheMetrics> {
  const info = await redis.info('stats')
  const memory = await redis.info('memory')
  const keyCount = await redis.dbsize()

  const hits = parseInt(info.match(/keyspace_hits:(\d+)/)?.[1] ?? '0')
  const misses = parseInt(info.match(/keyspace_misses:(\d+)/)?.[1] ?? '0')
  const total = hits + misses
  const usedMemory = memory.match(/used_memory_human:(.+)/)?.[1]?.trim() ?? 'unknown'

  return {
    hits,
    misses,
    hitRate: total > 0 ? hits / total : 0,
    keyCount,
    memoryUsage: usedMemory,
  }
}

// ヘルスチェックエンドポイントで公開
// GET /health/cache -> { hitRate: 0.95, keyCount: 12345, memoryUsage: "128.5M" }
```

## ベストプラクティス

1. **Cache-Asideをデフォルトに**: 最もシンプルで汎用的
2. **TTLは必ず設定**: TTLなしはメモリリークの原因
3. **キャッシュキーにバージョンを含める**: `v2:user:123` でスキーマ変更に対応
4. **ネガティブキャッシュ**: 存在しないデータのクエリも防ぐ（短TTL）
5. **パイプラインでバッチ操作**: 複数キーの操作は `redis.pipeline()` で
6. **スタンピード防止**: 高トラフィック時はロックまたは確率的早期更新
7. **ヒット率95%以上を目標**: 低い場合はTTLやキャッシュ対象を見直す
8. **キャッシュ障害時はDBにフォールバック**: キャッシュは落ちても動く設計にする

## アンチパターン

- `KEYS *` を本番で使う（`SCAN` を使え）
- TTLなしで無限にキャッシュする
- 全データをキャッシュする（ホットデータだけで十分）
- キャッシュとDBの一貫性を無視する
- キャッシュ障害でアプリ全体が落ちる設計
- 大きなオブジェクトを丸ごとキャッシュする（必要なフィールドだけ）
