---
name: queue-worker
description: Bull/BullMQジョブキュー設計、リトライ戦略、デッドレター、優先度キュー、並行制御
category: バックエンド
command: /queue-worker
version: 1.0.0
tags: [queue, bull, redis, async, worker]
---

# ジョブキュー設計パターン（BullMQ）

## 概要

BullMQを使ったプロダクション品質のジョブキューシステム設計パターン集。
リトライ戦略、デッドレターキュー、優先度制御、並行処理、モニタリングを網羅する。

## Steps

1. BullMQ + Redis の接続設定を行う
2. キュー定義とジョブ型を設計する
3. ワーカーを実装する（プロセッサー分離）
4. リトライ戦略を設定する（指数バックオフ）
5. デッドレターキューを実装する
6. 優先度キューと遅延ジョブを設定する
7. 並行制御とレート制限を調整する
8. モニタリングとアラートを追加する

## プロジェクト構造

```
src/
  queues/
    connection.ts         # Redis接続設定
    email.queue.ts        # メール送信キュー
    image.queue.ts        # 画像処理キュー
    export.queue.ts       # データエクスポートキュー
    index.ts              # キュー集約
  workers/
    email.worker.ts       # メールワーカー
    image.worker.ts       # 画像処理ワーカー
    export.worker.ts      # エクスポートワーカー
    index.ts              # ワーカー起動
  processors/
    email.processor.ts    # メール処理ロジック
    image.processor.ts    # 画像処理ロジック
  types/
    jobs.ts               # ジョブデータ型定義
  monitoring/
    bull-board.ts         # BullBoard UI設定
    metrics.ts            # Prometheusメトリクス
```

## Redis接続設定

```typescript
// src/queues/connection.ts
import { ConnectionOptions } from 'bullmq'

export const redisConnection: ConnectionOptions = {
  host: process.env.REDIS_HOST ?? 'localhost',
  port: Number(process.env.REDIS_PORT) ?? 6379,
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null,  // BullMQ要件
  enableReadyCheck: false,
  retryStrategy: (times: number) => {
    if (times > 10) return null  // 10回失敗で諦める
    return Math.min(times * 200, 5000)
  },
}
```

## ジョブ型定義

```typescript
// src/types/jobs.ts
export interface EmailJobData {
  to: string
  subject: string
  template: string
  variables: Record<string, string>
  priority?: 'high' | 'normal' | 'low'
}

export interface ImageJobData {
  sourceKey: string
  outputKey: string
  operations: ImageOperation[]
}

export type ImageOperation =
  | { type: 'resize'; width: number; height: number }
  | { type: 'watermark'; text: string }
  | { type: 'compress'; quality: number }

export interface ExportJobData {
  userId: string
  format: 'csv' | 'xlsx'
  filters: Record<string, unknown>
  callbackUrl?: string
}

// ジョブ結果型
export interface JobResult<T = unknown> {
  success: boolean
  data?: T
  error?: string
  duration: number
}
```

## キュー定義

```typescript
// src/queues/email.queue.ts
import { Queue, QueueOptions } from 'bullmq'
import { redisConnection } from './connection'
import { EmailJobData } from '../types/jobs'

const queueOptions: QueueOptions = {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,  // 2s -> 4s -> 8s
    },
    removeOnComplete: {
      age: 24 * 3600,   // 24時間後に完了ジョブを削除
      count: 1000,       // 最新1000件は保持
    },
    removeOnFail: {
      age: 7 * 24 * 3600,  // 7日後に失敗ジョブを削除
    },
  },
}

export const emailQueue = new Queue<EmailJobData>('email', queueOptions)

// ジョブ追加ヘルパー
export async function enqueueEmail(data: EmailJobData) {
  const priority = data.priority === 'high' ? 1 : data.priority === 'low' ? 3 : 2

  return emailQueue.add('send-email', data, {
    priority,
    // 重複防止: 同じメール+テンプレートは5分以内に1回
    jobId: `email-${data.to}-${data.template}-${Math.floor(Date.now() / 300000)}`,
  })
}

// 遅延ジョブ（スケジュール送信）
export async function scheduleEmail(data: EmailJobData, sendAt: Date) {
  const delay = sendAt.getTime() - Date.now()
  if (delay <= 0) {
    return enqueueEmail(data)
  }
  return emailQueue.add('send-email', data, { delay })
}
```

## ワーカー実装

```typescript
// src/workers/email.worker.ts
import { Worker, Job } from 'bullmq'
import { redisConnection } from '../queues/connection'
import { EmailJobData, JobResult } from '../types/jobs'
import { processEmail } from '../processors/email.processor'

export const emailWorker = new Worker<EmailJobData, JobResult>(
  'email',
  async (job: Job<EmailJobData>) => {
    const startTime = Date.now()

    try {
      // 進捗報告
      await job.updateProgress(10)

      const result = await processEmail(job.data)

      await job.updateProgress(100)

      return {
        success: true,
        data: result,
        duration: Date.now() - startTime,
      }
    } catch (error) {
      const message = error instanceof Error ? error.message : 'Unknown error'

      // 最終リトライかどうか判定
      const isLastAttempt = job.attemptsMade >= (job.opts.attempts ?? 3) - 1
      if (isLastAttempt) {
        // デッドレターキューに送る
        await moveToDeadLetter(job, message)
      }

      throw error  // BullMQにリトライさせる
    }
  },
  {
    connection: redisConnection,
    concurrency: 5,           // 同時処理数
    limiter: {
      max: 100,               // 最大100ジョブ
      duration: 60_000,       // 1分あたり
    },
    lockDuration: 30_000,     // ジョブロック30秒
    stalledInterval: 15_000,  // stalled検出間隔
  }
)

// イベントハンドリング
emailWorker.on('completed', (job, result) => {
  console.info(`Job ${job.id} completed in ${result.duration}ms`)
})

emailWorker.on('failed', (job, error) => {
  console.error(`Job ${job?.id} failed: ${error.message}`, {
    attempts: job?.attemptsMade,
    data: job?.data,
  })
})

emailWorker.on('stalled', (jobId) => {
  console.warn(`Job ${jobId} has stalled`)
})
```

## デッドレターキュー

```typescript
// src/queues/dead-letter.ts
import { Queue, Job } from 'bullmq'
import { redisConnection } from './connection'

interface DeadLetterData {
  originalQueue: string
  originalJobId: string
  originalData: unknown
  failedReason: string
  failedAt: string
  attempts: number
}

export const deadLetterQueue = new Queue<DeadLetterData>('dead-letter', {
  connection: redisConnection,
  defaultJobOptions: {
    removeOnComplete: false,  // 手動確認まで保持
    removeOnFail: false,
  },
})

export async function moveToDeadLetter(job: Job, reason: string): Promise<void> {
  await deadLetterQueue.add('dead-letter', {
    originalQueue: job.queueName,
    originalJobId: job.id ?? 'unknown',
    originalData: job.data,
    failedReason: reason,
    failedAt: new Date().toISOString(),
    attempts: job.attemptsMade,
  })
}

// デッドレターからリトライ
export async function retryDeadLetter(deadLetterJobId: string): Promise<void> {
  const job = await deadLetterQueue.getJob(deadLetterJobId)
  if (!job) throw new Error('Dead letter job not found')

  const data = job.data
  const originalQueue = new Queue(data.originalQueue, { connection: redisConnection })

  await originalQueue.add('retry-from-dead-letter', data.originalData as Record<string, unknown>, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
  })

  await job.remove()
}
```

## 優先度キューパターン

```typescript
// 優先度: 1 (最高) ~ 2^21 (最低)
// 同一優先度内はFIFO

// 緊急通知
await notificationQueue.add('urgent', data, { priority: 1 })

// 通常通知
await notificationQueue.add('normal', data, { priority: 5 })

// バッチ処理（低優先度）
await notificationQueue.add('batch', data, { priority: 10 })
```

## リトライ戦略パターン

```typescript
// 1. 指数バックオフ（デフォルト推奨）
const exponentialBackoff = {
  attempts: 5,
  backoff: { type: 'exponential' as const, delay: 1000 },
  // 1s -> 2s -> 4s -> 8s -> 16s
}

// 2. 固定間隔
const fixedBackoff = {
  attempts: 3,
  backoff: { type: 'fixed' as const, delay: 5000 },
  // 5s -> 5s -> 5s
}

// 3. カスタム戦略（エラー種別で分岐）
const customBackoff = {
  attempts: 5,
  backoff: {
    type: 'custom' as const,
  },
}

// ワーカー設定でカスタムバックオフを定義
const worker = new Worker('my-queue', processor, {
  settings: {
    backoffStrategy: (attemptsMade: number, type: string, err: Error) => {
      // レート制限エラーは長めに待つ
      if (err.message.includes('rate limit')) {
        return 60_000 * attemptsMade  // 1分 -> 2分 -> 3分
      }
      // ネットワークエラーは指数バックオフ
      if (err.message.includes('ECONNREFUSED')) {
        return Math.pow(2, attemptsMade) * 1000
      }
      // その他は固定5秒
      return 5000
    },
  },
})
```

## 繰り返しジョブ（Cron）

```typescript
// 毎日午前3時にクリーンアップ
await cleanupQueue.add('daily-cleanup', {}, {
  repeat: {
    pattern: '0 3 * * *',       // cron式
    tz: 'Asia/Tokyo',
  },
  jobId: 'daily-cleanup',       // 重複防止
})

// 5分ごとにヘルスチェック
await healthQueue.add('health-check', {}, {
  repeat: {
    every: 5 * 60 * 1000,       // ミリ秒指定
  },
  jobId: 'health-check',
})
```

## フロー（親子ジョブ）

```typescript
// 複雑なワークフローを親子関係で表現
import { FlowProducer } from 'bullmq'

const flowProducer = new FlowProducer({ connection: redisConnection })

// 画像処理パイプライン: リサイズ全完了後にまとめて通知
await flowProducer.add({
  name: 'notify-completion',
  queueName: 'notification',
  data: { userId: 'user-123', message: '画像処理が完了しました' },
  children: [
    {
      name: 'resize-thumbnail',
      queueName: 'image',
      data: { sourceKey: 'img.jpg', width: 150, height: 150 },
    },
    {
      name: 'resize-medium',
      queueName: 'image',
      data: { sourceKey: 'img.jpg', width: 800, height: 600 },
    },
    {
      name: 'resize-large',
      queueName: 'image',
      data: { sourceKey: 'img.jpg', width: 1920, height: 1080 },
    },
  ],
})
```

## モニタリング（Bull Board）

```typescript
// src/monitoring/bull-board.ts
import { createBullBoard } from '@bull-board/api'
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter'
import { ExpressAdapter } from '@bull-board/express'
import { emailQueue } from '../queues/email.queue'
import { imageQueue } from '../queues/image.queue'
import { deadLetterQueue } from '../queues/dead-letter'

const serverAdapter = new ExpressAdapter()
serverAdapter.setBasePath('/admin/queues')

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue),
    new BullMQAdapter(imageQueue),
    new BullMQAdapter(deadLetterQueue),
  ],
  serverAdapter,
})

export { serverAdapter as bullBoardAdapter }

// Expressアプリに追加
// app.use('/admin/queues', authenticate, authorize('admin'), bullBoardAdapter.getRouter())
```

## Graceful Shutdown

```typescript
// src/workers/index.ts
import { emailWorker } from './email.worker'
import { imageWorker } from './image.worker'

const workers = [emailWorker, imageWorker]

async function shutdown(signal: string): Promise<void> {
  console.info(`${signal} received. Graceful shutdown...`)

  // 新しいジョブの取得を停止し、処理中のジョブの完了を待つ
  await Promise.all(
    workers.map((w) => w.close())
  )

  console.info('All workers closed')
  process.exit(0)
}

process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT', () => shutdown('SIGINT'))
```

## ベストプラクティス

1. **ジョブデータは最小限**: 大きなデータはS3/DBに保存し、キーだけをジョブに渡す
2. **冪等性を保証**: 同じジョブが2回実行されても安全なように設計する
3. **タイムアウト設定**: `lockDuration` でジョブの最大実行時間を制限する
4. **進捗報告**: 長時間ジョブは `job.updateProgress()` で進捗を報告する
5. **デッドレター必須**: 全リトライ失敗後の受け皿を必ず用意する
6. **concurrencyの調整**: CPU負荷の高い処理は1-2、I/O待ちは10-50
7. **ジョブIDで重複防止**: 同じ処理が二重投入されないようにする
8. **graceful shutdown**: SIGTERMで処理中ジョブの完了を待ってから終了する

## アンチパターン

- ジョブデータにバイナリ/巨大JSONを直接格納する
- リトライなしでジョブを投入する（ネットワークは必ず失敗する）
- ワーカーとAPIサーバーを同一プロセスで動かす
- concurrencyを無制限にする（リソース枯渇の原因）
- Redisの永続化設定を忘れる（再起動でジョブが消える）
