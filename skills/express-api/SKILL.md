---
name: express-api
description: Express REST API設計パターン。ミドルウェア構成、エラーハンドリング、認証、レート制限、バリデーション
category: バックエンド
command: /express-api
version: 1.0.0
tags: [node, express, api, rest, middleware]
---

# Express REST API 設計パターン

## 概要

Express.jsでプロダクション品質のREST APIを構築するための包括的パターン集。
ミドルウェアの正しい構成順序、統一エラーハンドリング、入力バリデーション、認証、レート制限を網羅する。

## Steps

1. プロジェクト構造を設計する（レイヤードアーキテクチャ）
2. ミドルウェアチェーンを正しい順序で構成する
3. ルーティングとコントローラーを分離する
4. 入力バリデーションを全エンドポイントに適用する
5. 統一エラーハンドリングを実装する
6. 認証・認可ミドルウェアを追加する
7. レート制限を設定する
8. レスポンス形式を統一する

## プロジェクト構造

```
src/
  app.ts                # Express アプリケーション設定
  server.ts             # サーバー起動
  routes/
    index.ts            # ルート集約
    users.routes.ts     # ユーザー関連ルート
    products.routes.ts  # 商品関連ルート
  controllers/
    users.controller.ts
    products.controller.ts
  services/
    users.service.ts
    products.service.ts
  middleware/
    auth.ts             # 認証ミドルウェア
    validate.ts         # バリデーションミドルウェア
    error-handler.ts    # エラーハンドラー
    rate-limit.ts       # レート制限
    request-id.ts       # リクエストID付与
    logger.ts           # リクエストログ
  schemas/
    users.schema.ts     # Zodバリデーションスキーマ
  types/
    index.ts            # 型定義
  utils/
    app-error.ts        # カスタムエラークラス
    async-handler.ts    # 非同期ラッパー
    response.ts         # レスポンスヘルパー
```

## ミドルウェア構成順序（重要）

```typescript
// src/app.ts
import express from 'express'
import helmet from 'helmet'
import cors from 'cors'
import compression from 'compression'
import { requestId } from './middleware/request-id'
import { requestLogger } from './middleware/logger'
import { rateLimiter } from './middleware/rate-limit'
import { errorHandler } from './middleware/error-handler'
import { notFoundHandler } from './middleware/not-found'
import { routes } from './routes'

const app = express()

// 1. セキュリティヘッダー（最初）
app.use(helmet())

// 2. CORS
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
  credentials: true,
}))

// 3. リクエストID（ログの前に）
app.use(requestId)

// 4. リクエストログ
app.use(requestLogger)

// 5. レート制限（パース前に拒否）
app.use(rateLimiter)

// 6. ボディパース
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true }))

// 7. 圧縮
app.use(compression())

// 8. ルーティング
app.use('/api', routes)

// 9. 404ハンドラー（ルートの後）
app.use(notFoundHandler)

// 10. エラーハンドラー（最後）
app.use(errorHandler)

export { app }
```

## カスタムエラークラス

```typescript
// src/utils/app-error.ts
export class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly message: string,
    public readonly code?: string,
    public readonly details?: unknown
  ) {
    super(message)
    this.name = 'AppError'
  }

  static badRequest(message: string, details?: unknown): AppError {
    return new AppError(400, message, 'BAD_REQUEST', details)
  }

  static unauthorized(message = '認証が必要です'): AppError {
    return new AppError(401, message, 'UNAUTHORIZED')
  }

  static forbidden(message = 'アクセス権限がありません'): AppError {
    return new AppError(403, message, 'FORBIDDEN')
  }

  static notFound(resource = 'リソース'): AppError {
    return new AppError(404, `${resource}が見つかりません`, 'NOT_FOUND')
  }

  static conflict(message: string): AppError {
    return new AppError(409, message, 'CONFLICT')
  }

  static tooManyRequests(message = 'リクエストが多すぎます'): AppError {
    return new AppError(429, message, 'TOO_MANY_REQUESTS')
  }
}
```

## 統一エラーハンドラー

```typescript
// src/middleware/error-handler.ts
import { ErrorRequestHandler } from 'express'
import { ZodError } from 'zod'
import { AppError } from '../utils/app-error'

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  // Zodバリデーションエラー
  if (err instanceof ZodError) {
    return res.status(400).json({
      success: false,
      error: 'バリデーションエラー',
      code: 'VALIDATION_ERROR',
      details: err.errors.map((e) => ({
        path: e.path.join('.'),
        message: e.message,
      })),
    })
  }

  // アプリケーションエラー
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      error: err.message,
      code: err.code,
      ...(err.details ? { details: err.details } : {}),
    })
  }

  // 予期しないエラー（本番では詳細を隠す）
  const isDev = process.env.NODE_ENV === 'development'
  return res.status(500).json({
    success: false,
    error: isDev ? err.message : 'サーバー内部エラーが発生しました',
    code: 'INTERNAL_ERROR',
    ...(isDev ? { stack: err.stack } : {}),
  })
}
```

## 非同期ハンドラーラッパー

```typescript
// src/utils/async-handler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express'

export function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<unknown>
): RequestHandler {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next)
  }
}
```

## Zodバリデーションミドルウェア

```typescript
// src/middleware/validate.ts
import { AnyZodObject, ZodError } from 'zod'
import { Request, Response, NextFunction } from 'express'

export function validate(schema: AnyZodObject) {
  return (req: Request, _res: Response, next: NextFunction) => {
    try {
      schema.parse({
        body: req.body,
        query: req.query,
        params: req.params,
      })
      next()
    } catch (error) {
      next(error)
    }
  }
}

// 使用例: スキーマ定義
// src/schemas/users.schema.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  body: z.object({
    email: z.string().email('有効なメールアドレスを入力してください'),
    name: z.string().min(1, '名前は必須です').max(100),
    password: z.string().min(8, 'パスワードは8文字以上'),
  }),
})

export const getUserSchema = z.object({
  params: z.object({
    id: z.string().uuid('有効なIDを指定してください'),
  }),
})
```

## コントローラーパターン

```typescript
// src/controllers/users.controller.ts
import { Request, Response } from 'express'
import { asyncHandler } from '../utils/async-handler'
import { AppError } from '../utils/app-error'
import { usersService } from '../services/users.service'

export const getUser = asyncHandler(async (req: Request, res: Response) => {
  const user = await usersService.findById(req.params.id)
  if (!user) {
    throw AppError.notFound('ユーザー')
  }
  res.json({ success: true, data: user })
})

export const createUser = asyncHandler(async (req: Request, res: Response) => {
  const existing = await usersService.findByEmail(req.body.email)
  if (existing) {
    throw AppError.conflict('このメールアドレスは既に使用されています')
  }
  const user = await usersService.create(req.body)
  res.status(201).json({ success: true, data: user })
})

export const listUsers = asyncHandler(async (req: Request, res: Response) => {
  const page = Number(req.query.page) || 1
  const limit = Math.min(Number(req.query.limit) || 20, 100)
  const result = await usersService.findAll({ page, limit })
  res.json({
    success: true,
    data: result.items,
    meta: { total: result.total, page, limit },
  })
})
```

## ルーティング

```typescript
// src/routes/users.routes.ts
import { Router } from 'express'
import { validate } from '../middleware/validate'
import { authenticate } from '../middleware/auth'
import { createUserSchema, getUserSchema } from '../schemas/users.schema'
import * as usersController from '../controllers/users.controller'

const router = Router()

router.get('/', authenticate, usersController.listUsers)
router.get('/:id', authenticate, validate(getUserSchema), usersController.getUser)
router.post('/', validate(createUserSchema), usersController.createUser)

export { router as usersRoutes }
```

## レート制限

```typescript
// src/middleware/rate-limit.ts
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'
import { createClient } from 'redis'

const redisClient = createClient({ url: process.env.REDIS_URL })
redisClient.connect()

// グローバルレート制限
export const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
  message: { success: false, error: 'リクエストが多すぎます', code: 'TOO_MANY_REQUESTS' },
})

// エンドポイント別レート制限
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { success: false, error: 'ログイン試行回数が上限を超えました', code: 'TOO_MANY_REQUESTS' },
})
```

## レスポンス統一ヘルパー

```typescript
// src/utils/response.ts
import { Response } from 'express'

interface PaginationMeta {
  total: number
  page: number
  limit: number
  totalPages: number
}

export function sendSuccess<T>(res: Response, data: T, statusCode = 200): void {
  res.status(statusCode).json({ success: true, data })
}

export function sendPaginated<T>(
  res: Response,
  items: T[],
  meta: PaginationMeta
): void {
  res.json({ success: true, data: items, meta })
}

export function sendCreated<T>(res: Response, data: T): void {
  sendSuccess(res, data, 201)
}

export function sendNoContent(res: Response): void {
  res.status(204).send()
}
```

## ベストプラクティス

1. **ミドルウェア順序を守る**: セキュリティ → ログ → レート制限 → パース → ルーティング → エラー
2. **全ルートでバリデーション**: Zodスキーマで入力を必ず検証する
3. **asyncHandlerで例外捕捉**: try-catchの代わりにラッパーを使う
4. **エラーは統一形式**: AppErrorクラスで全エラーを統一する
5. **レスポンスも統一形式**: `{ success, data, error, meta }` を守る
6. **環境変数は起動時検証**: 必須環境変数がなければ即座に落とす
7. **graceful shutdown**: SIGTERM受信時に接続を閉じてから終了する
8. **ヘルスチェック**: `/health` エンドポイントを認証なしで公開する

## アンチパターン

- コントローラーにビジネスロジックを書く（サービス層に分離せよ）
- `app.use(cors())` を無条件で使う（originを明示せよ）
- `express.json()` のlimit未設定（DoS攻撃のリスク）
- エラーハンドラーで `res.send(err.message)` する（情報漏洩）
- `req.body` を直接DBに渡す（バリデーション必須）
