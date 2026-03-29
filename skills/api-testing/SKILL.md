---
name: api-testing
description: APIテスト設計パターン。Supertest、コントラクトテスト、スナップショット、負荷テスト、モック戦略
category: テスト
command: /api-test
version: 1.0.0
tags:
  - api-testing
  - supertest
  - contract
  - load-test
  - mock
---

# API Testing Patterns

APIテストの設計・実装パターン集。ユニット・インテグレーション・コントラクト・負荷テストをカバー。

## When to Activate

- APIエンドポイントのテストを書くとき
- コントラクトテストを導入するとき
- テストデータ管理戦略を設計するとき
- 負荷テストを実施するとき
- 外部APIのモック戦略を決めるとき

## Steps

### Step 1: テスト環境セットアップ

```bash
pnpm add -D vitest supertest @types/supertest msw
```

```typescript
// test/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest'
import { setupServer } from 'msw/node'
import { handlers } from './mocks/handlers'

export const mockServer = setupServer(...handlers)

beforeAll(() => mockServer.listen({ onUnhandledRequest: 'error' }))
afterEach(() => mockServer.resetHandlers())
afterAll(() => mockServer.close())
```

### Step 2: Supertestインテグレーションテスト

```typescript
// test/api/users.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import supertest from 'supertest'
import { app } from '../../src/app'
import { db } from '../../src/db'
import { users } from '../../src/db/schema'

const request = supertest(app)

describe('POST /api/users', () => {
  beforeEach(async () => {
    await db.delete(users)
  })

  it('正常なリクエストでユーザーを作成する', async () => {
    const response = await request
      .post('/api/users')
      .send({ email: 'test@example.com', name: 'Test User' })
      .expect(201)
      .expect('Content-Type', /json/)

    expect(response.body).toMatchObject({
      success: true,
      data: {
        email: 'test@example.com',
        name: 'Test User',
      },
    })
    expect(response.body.data.id).toBeDefined()
  })

  it('バリデーションエラーを返す', async () => {
    const response = await request
      .post('/api/users')
      .send({ email: 'invalid', name: '' })
      .expect(400)

    expect(response.body).toMatchObject({
      error: {
        code: 'VALIDATION_ERROR',
        details: {
          email: expect.arrayContaining([expect.any(String)]),
          name: expect.arrayContaining([expect.any(String)]),
        },
      },
    })
  })

  it('重複メールアドレスで409を返す', async () => {
    await request
      .post('/api/users')
      .send({ email: 'dup@example.com', name: 'First' })
      .expect(201)

    await request
      .post('/api/users')
      .send({ email: 'dup@example.com', name: 'Second' })
      .expect(409)
  })
})

describe('GET /api/users/:id', () => {
  it('存在するユーザーを返す', async () => {
    const created = await request
      .post('/api/users')
      .send({ email: 'find@example.com', name: 'Find Me' })
    const userId = created.body.data.id

    const response = await request
      .get(`/api/users/${userId}`)
      .expect(200)

    expect(response.body.data.email).toBe('find@example.com')
  })

  it('存在しないユーザーで404を返す', async () => {
    await request
      .get('/api/users/00000000-0000-0000-0000-000000000000')
      .expect(404)
  })
})
```

### Step 3: 認証テスト

```typescript
// test/helpers/auth.ts
import jwt from 'jsonwebtoken'

function generateTestToken(payload: { userId: string; role: string }): string {
  return jwt.sign(payload, process.env.JWT_SECRET!, { expiresIn: '1h' })
}

function authHeader(token: string): Record<string, string> {
  return { Authorization: `Bearer ${token}` }
}

// テスト内で使用
describe('GET /api/admin/users', () => {
  it('管理者はユーザー一覧を取得できる', async () => {
    const token = generateTestToken({ userId: 'admin-1', role: 'admin' })

    await request
      .get('/api/admin/users')
      .set(authHeader(token))
      .expect(200)
  })

  it('一般ユーザーは403を返す', async () => {
    const token = generateTestToken({ userId: 'user-1', role: 'user' })

    await request
      .get('/api/admin/users')
      .set(authHeader(token))
      .expect(403)
  })

  it('トークンなしで401を返す', async () => {
    await request
      .get('/api/admin/users')
      .expect(401)
  })
})
```

### Step 4: MSWモック（外部API）

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  // Stripe API mock
  http.post('https://api.stripe.com/v1/charges', async ({ request }) => {
    const body = await request.formData()
    const amount = body.get('amount')

    if (Number(amount) > 999999) {
      return HttpResponse.json(
        { error: { message: 'Amount too large' } },
        { status: 400 },
      )
    }

    return HttpResponse.json({
      id: 'ch_test_123',
      amount: Number(amount),
      status: 'succeeded',
    })
  }),

  // SendGrid mock
  http.post('https://api.sendgrid.com/v3/mail/send', () => {
    return HttpResponse.json({ message: 'success' }, { status: 202 })
  }),
]

// テスト内でオーバーライド
import { mockServer } from '../setup'

it('Stripe APIエラー時に適切にハンドリングする', async () => {
  mockServer.use(
    http.post('https://api.stripe.com/v1/charges', () => {
      return HttpResponse.json(
        { error: { message: 'Card declined' } },
        { status: 402 },
      )
    }),
  )

  const response = await request
    .post('/api/payments')
    .send({ amount: 1000, token: 'tok_test' })
    .expect(402)

  expect(response.body.error.code).toBe('PAYMENT_FAILED')
})
```

## コントラクトテスト

### Zodスキーマベース

```typescript
// contracts/user.contract.ts
import { z } from 'zod'

export const UserResponseContract = z.object({
  success: z.literal(true),
  data: z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    name: z.string().min(1),
    role: z.enum(['admin', 'user', 'viewer']),
    createdAt: z.string().datetime(),
  }),
})

export const UserListResponseContract = z.object({
  success: z.literal(true),
  data: z.array(UserResponseContract.shape.data),
  meta: z.object({
    total: z.number().int().nonneg(),
    page: z.number().int().positive(),
    limit: z.number().int().positive(),
  }),
})

// テストで検証
it('レスポンスがコントラクトに準拠している', async () => {
  const response = await request.get('/api/users').expect(200)
  const result = UserListResponseContract.safeParse(response.body)

  expect(result.success).toBe(true)
  if (!result.success) {
    console.error('Contract violation:', result.error.format())
  }
})
```

### スナップショットテスト

```typescript
it('APIレスポンス構造が変わっていない', async () => {
  const response = await request.get('/api/products/1').expect(200)

  // 動的フィールドを固定値に置換
  const normalized = {
    ...response.body,
    data: {
      ...response.body.data,
      id: '[UUID]',
      createdAt: '[TIMESTAMP]',
      updatedAt: '[TIMESTAMP]',
    },
  }

  expect(normalized).toMatchSnapshot()
})
```

## ページネーションテスト

```typescript
describe('GET /api/products (pagination)', () => {
  beforeEach(async () => {
    // 25件のテストデータを投入
    await seedProducts(25)
  })

  it('デフォルトで20件返す', async () => {
    const response = await request.get('/api/products').expect(200)

    expect(response.body.data).toHaveLength(20)
    expect(response.body.meta.total).toBe(25)
    expect(response.body.meta.page).toBe(1)
  })

  it('2ページ目を取得できる', async () => {
    const response = await request
      .get('/api/products?page=2&limit=20')
      .expect(200)

    expect(response.body.data).toHaveLength(5)
    expect(response.body.meta.page).toBe(2)
  })

  it('limitの上限を超えない', async () => {
    const response = await request
      .get('/api/products?limit=1000')
      .expect(200)

    expect(response.body.data.length).toBeLessThanOrEqual(100)
  })
})
```

## 負荷テスト（k6）

```javascript
// load-test/api.js
import http from 'k6/http'
import { check, sleep } from 'k6'

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // ランプアップ
    { duration: '1m', target: 50 },     // 定常状態
    { duration: '30s', target: 100 },   // ピーク
    { duration: '30s', target: 0 },     // ランプダウン
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95%ile < 500ms
    http_req_failed: ['rate<0.01'],     // エラー率 < 1%
  },
}

export default function () {
  const res = http.get('http://localhost:3000/api/products')

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has data': (r) => JSON.parse(r.body).data.length > 0,
  })

  sleep(1)
}
```

```bash
# 実行
k6 run load-test/api.js

# レポート出力
k6 run --out json=results.json load-test/api.js
```

## テストデータ管理

```typescript
// test/factories/user.factory.ts
import { faker } from '@faker-js/faker'

interface UserFactoryOptions {
  email?: string
  name?: string
  role?: 'admin' | 'user' | 'viewer'
}

function buildUser(overrides: UserFactoryOptions = {}) {
  return {
    email: overrides.email ?? faker.internet.email(),
    name: overrides.name ?? faker.person.fullName(),
    role: overrides.role ?? 'user',
  }
}

async function createUser(overrides: UserFactoryOptions = {}) {
  const data = buildUser(overrides)
  const [user] = await db.insert(users).values(data).returning()
  return user
}

// テストで使用
const admin = await createUser({ role: 'admin' })
const regularUser = await createUser()
```

## テストチェックリスト

| Category | Tests |
|----------|-------|
| 正常系 | 200/201レスポンス、データ構造、ページネーション |
| バリデーション | 400エラー、必須フィールド、型チェック |
| 認証 | 401（未認証）、403（権限不足）、トークン期限切れ |
| エラー | 404（Not Found）、409（Conflict）、500（Server Error） |
| エッジケース | 空配列、上限値、特殊文字、SQLインジェクション |
| パフォーマンス | レスポンス時間、同時リクエスト |

## ファイル構成

```
test/
├── setup.ts              # グローバルセットアップ
├── helpers/
│   └── auth.ts           # 認証ヘルパー
├── factories/
│   └── user.factory.ts   # テストデータファクトリ
├── mocks/
│   └── handlers.ts       # MSWハンドラー
├── api/
│   ├── users.test.ts     # ユーザーAPI
│   ├── products.test.ts  # 商品API
│   └── payments.test.ts  # 決済API
├── contracts/
│   └── user.contract.ts  # コントラクト定義
└── load/
    └── api.js            # k6負荷テスト
```

## Related

- Skill: `playwright-testing` - E2Eテスト
- Skill: `tdd-workflow` - TDDワークフロー
- Skill: `error-handling-patterns` - エラーハンドリング
- Agent: `tdd-guide` - TDDガイドエージェント
