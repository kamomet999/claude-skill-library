---
name: error-handling-patterns
description: エラーハンドリング設計パターン。Result型、カスタムエラー階層、リトライ、サーキットブレーカー、回復力設計
category: アーキテクチャ
command: /error-handling
version: 1.0.0
tags:
  - error-handling
  - result
  - retry
  - circuit-breaker
  - resilience
---

# Error Handling Patterns

堅牢なエラーハンドリングの設計・実装パターン集。

## When to Activate

- API・サービスのエラー処理を設計するとき
- Result型パターンを導入するとき
- リトライ・サーキットブレーカーを実装するとき
- カスタムエラー階層を設計するとき
- 外部サービス連携の回復力を高めるとき

## Steps

### Step 1: カスタムエラー階層

```typescript
// errors/base.ts
export class AppError extends Error {
  readonly statusCode: number
  readonly code: string
  readonly isOperational: boolean

  constructor(
    message: string,
    statusCode: number,
    code: string,
    isOperational: boolean = true,
  ) {
    super(message)
    this.name = this.constructor.name
    this.statusCode = statusCode
    this.code = code
    this.isOperational = isOperational
    Error.captureStackTrace(this, this.constructor)
  }
}

// errors/http.ts
export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} not found: ${id}`, 404, 'NOT_FOUND')
  }
}

export class ValidationError extends AppError {
  readonly details: Record<string, string[]>

  constructor(details: Record<string, string[]>) {
    super('Validation failed', 400, 'VALIDATION_ERROR')
    this.details = details
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT')
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Authentication required') {
    super(message, 401, 'UNAUTHORIZED')
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Insufficient permissions') {
    super(message, 403, 'FORBIDDEN')
  }
}

export class RateLimitError extends AppError {
  readonly retryAfter: number

  constructor(retryAfter: number) {
    super('Rate limit exceeded', 429, 'RATE_LIMIT_EXCEEDED')
    this.retryAfter = retryAfter
  }
}

// errors/external.ts（外部サービスエラー）
export class ExternalServiceError extends AppError {
  readonly service: string

  constructor(service: string, message: string) {
    super(`External service error (${service}): ${message}`, 502, 'EXTERNAL_SERVICE_ERROR')
    this.service = service
  }
}
```

### Step 2: Result型パターン（例外を投げない）

```typescript
// result.ts
type Result<T, E = AppError> =
  | { success: true; data: T }
  | { success: false; error: E }

function ok<T>(data: T): Result<T, never> {
  return { success: true, data }
}

function err<E>(error: E): Result<never, E> {
  return { success: false, error }
}

// 使用例
async function findUser(id: string): Promise<Result<User, NotFoundError>> {
  const user = await db.query.users.findFirst({
    where: eq(users.id, id),
  })

  if (!user) {
    return err(new NotFoundError('User', id))
  }

  return ok(user)
}

// 呼び出し側
const result = await findUser(userId)
if (!result.success) {
  // result.error は NotFoundError と型推論される
  return res.status(result.error.statusCode).json({
    error: result.error.message,
    code: result.error.code,
  })
}
// result.data は User と型推論される
const user = result.data
```

### Step 3: Expressエラーミドルウェア

```typescript
// middleware/error-handler.ts
import type { ErrorRequestHandler } from 'express'

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  // Operational Error（予期されたエラー）
  if (err instanceof AppError && err.isOperational) {
    const response: Record<string, unknown> = {
      error: {
        code: err.code,
        message: err.message,
      },
    }

    if (err instanceof ValidationError) {
      response.error = { ...response.error, details: err.details }
    }

    if (err instanceof RateLimitError) {
      res.setHeader('Retry-After', err.retryAfter)
    }

    return res.status(err.statusCode).json(response)
  }

  // Programming Error（予期しないエラー）
  console.error('Unexpected error:', err)

  // 本番では詳細を隠す
  const message = process.env.NODE_ENV === 'production'
    ? 'Internal server error'
    : err.message

  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message,
    },
  })
}
```

### Step 4: リトライパターン

```typescript
// retry.ts
interface RetryOptions {
  maxRetries: number
  baseDelayMs: number
  maxDelayMs: number
  backoffMultiplier: number
  retryableErrors?: string[]
}

const defaultOptions: RetryOptions = {
  maxRetries: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {},
): Promise<T> {
  const opts = { ...defaultOptions, ...options }
  let lastError: Error | undefined

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error))

      // リトライ不可能なエラーは即座に投げる
      if (error instanceof AppError && error.isOperational) {
        throw error
      }

      if (attempt === opts.maxRetries) {
        break
      }

      // Exponential Backoff + Jitter
      const delay = Math.min(
        opts.baseDelayMs * Math.pow(opts.backoffMultiplier, attempt) + Math.random() * 1000,
        opts.maxDelayMs,
      )

      console.error(`Retry ${attempt + 1}/${opts.maxRetries} after ${delay}ms:`, lastError.message)
      await new Promise((resolve) => setTimeout(resolve, delay))
    }
  }

  throw lastError
}

// 使用例
const data = await withRetry(
  () => fetch('https://api.example.com/data').then((r) => r.json()),
  { maxRetries: 3, baseDelayMs: 500 },
)
```

### Step 5: サーキットブレーカー

```typescript
// circuit-breaker.ts
enum CircuitState {
  CLOSED = 'CLOSED',
  OPEN = 'OPEN',
  HALF_OPEN = 'HALF_OPEN',
}

interface CircuitBreakerOptions {
  failureThreshold: number
  resetTimeoutMs: number
  halfOpenMaxAttempts: number
  onStateChange?: (from: CircuitState, to: CircuitState) => void
}

class CircuitBreaker {
  private state = CircuitState.CLOSED
  private failureCount = 0
  private halfOpenAttempts = 0
  private lastFailureTime = 0

  constructor(
    private readonly name: string,
    private readonly options: CircuitBreakerOptions,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    this.evaluateState()

    if (this.state === CircuitState.OPEN) {
      throw new ExternalServiceError(
        this.name,
        `Circuit breaker is OPEN. Retry after ${this.options.resetTimeoutMs}ms`,
      )
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

  private evaluateState(): void {
    if (
      this.state === CircuitState.OPEN &&
      Date.now() - this.lastFailureTime >= this.options.resetTimeoutMs
    ) {
      this.transition(CircuitState.HALF_OPEN)
    }
  }

  private onSuccess(): void {
    if (this.state === CircuitState.HALF_OPEN) {
      this.transition(CircuitState.CLOSED)
    }
    this.failureCount = 0
    this.halfOpenAttempts = 0
  }

  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = Date.now()

    if (this.state === CircuitState.HALF_OPEN) {
      this.halfOpenAttempts++
      if (this.halfOpenAttempts >= this.options.halfOpenMaxAttempts) {
        this.transition(CircuitState.OPEN)
      }
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.transition(CircuitState.OPEN)
    }
  }

  private transition(to: CircuitState): void {
    const from = this.state
    this.state = to
    this.options.onStateChange?.(from, to)
  }

  getState(): CircuitState {
    return this.state
  }
}

// 使用例
const paymentBreaker = new CircuitBreaker('payment-api', {
  failureThreshold: 5,
  resetTimeoutMs: 30000,
  halfOpenMaxAttempts: 2,
  onStateChange: (from, to) => {
    console.error(`Circuit breaker [payment-api]: ${from} -> ${to}`)
  },
})

const result = await paymentBreaker.execute(() =>
  fetch('https://payment-api.example.com/charge', { method: 'POST', body })
)
```

## パターン組み合わせ

### リトライ + サーキットブレーカー + フォールバック

```typescript
async function resilientCall<T>(
  primary: () => Promise<T>,
  fallback: () => Promise<T>,
  breaker: CircuitBreaker,
): Promise<T> {
  try {
    return await breaker.execute(() =>
      withRetry(primary, { maxRetries: 2, baseDelayMs: 500 })
    )
  } catch (error) {
    console.error('Primary failed, using fallback:', error)
    return fallback()
  }
}

// 使用例
const products = await resilientCall(
  () => productService.getAll(),        // 主系統
  () => cache.get('products:all'),      // フォールバック（キャッシュ）
  productServiceBreaker,
)
```

## エラーレスポンス標準形式

```typescript
// RFC 7807 Problem Details準拠
interface ProblemDetails {
  type: string           // エラー種別URI
  title: string          // 人間可読なタイトル
  status: number         // HTTPステータス
  detail?: string        // 詳細説明
  instance?: string      // エラー発生箇所
  errors?: Record<string, string[]>  // バリデーションエラー
  traceId?: string       // トレースID
}

// 例
{
  "type": "/errors/validation",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request body contains invalid fields",
  "errors": {
    "email": ["Invalid email format"],
    "age": ["Must be between 0 and 150"]
  },
  "traceId": "abc-123-def"
}
```

## Best Practices

| Practice | Do | Don't |
|----------|-----|-------|
| エラー分類 | Operational vs Programming | 全部catchで握りつぶす |
| エラーメッセージ | ユーザーフレンドリー | スタックトレースを返す |
| ログ | 構造化ログ + traceId | console.log |
| リトライ | 冪等操作のみ | POST/DELETEを無条件リトライ |
| フォールバック | キャッシュ or デフォルト値 | エラー無視 |
| バリデーション | 入口で早期検証 | 深い階層で初めて検証 |

## Anti-Patterns

- `catch (e) {}` - エラーを握りつぶす
- `throw new Error('error')` - 詳細情報のないエラー
- try/catchの乱用 - 制御フローにexceptionを使う
- エラーログにユーザーデータを含める - 個人情報漏洩
- リトライ無制限 - 障害を悪化させる

## Related

- Skill: `backend-patterns` - API設計パターン
- Skill: `microservices-design` - マイクロサービス設計
- Agent: `code-reviewer` - コードレビュー
