---
name: drizzle-orm
description: Drizzle ORMスキーマ定義、マイグレーション、クエリビルダー、リレーション、型安全なデータベース操作パターン集
category: データベース
command: /drizzle
version: 1.0.0
tags:
  - drizzle
  - orm
  - typescript
  - migration
  - schema
---

# Drizzle ORM Patterns

TypeScript-first ORM。SQL-likeな記法で型安全なデータベース操作を実現する。

## When to Activate

- 新規プロジェクトでDB層を設計するとき
- スキーマ定義やマイグレーションを作成するとき
- 型安全なクエリを構築するとき
- リレーションやJOINを設計するとき
- 既存のPrismaやTypeORMから移行するとき

## Steps

### Step 1: プロジェクトセットアップ

```bash
# PostgreSQL
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit

# SQLite (Turso/LibSQL)
pnpm add drizzle-orm @libsql/client
pnpm add -D drizzle-kit

# MySQL
pnpm add drizzle-orm mysql2
pnpm add -D drizzle-kit
```

### Step 2: drizzle.config.ts

```typescript
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/db/schema/index.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
})
```

### Step 3: スキーマ定義

```typescript
// src/db/schema/users.ts
import { pgTable, text, timestamp, integer, boolean, uuid } from 'drizzle-orm/pg-core'
import { createInsertSchema, createSelectSchema } from 'drizzle-zod'
import { relations } from 'drizzle-orm'

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  role: text('role', { enum: ['admin', 'user', 'viewer'] }).default('user').notNull(),
  isActive: boolean('is_active').default(true).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
})

// Zodバリデーション自動生成
export const insertUserSchema = createInsertSchema(users, {
  email: (schema) => schema.email.email(),
  name: (schema) => schema.name.min(1).max(100),
})
export const selectUserSchema = createSelectSchema(users)

export type User = typeof users.$inferSelect
export type NewUser = typeof users.$inferInsert
```

### Step 4: リレーション定義

```typescript
// src/db/schema/posts.ts
import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { users } from './users'

export const posts = pgTable('posts', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: uuid('author_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
})

// src/db/schema/relations.ts
import { relations } from 'drizzle-orm'
import { users } from './users'
import { posts } from './posts'

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}))
```

### Step 5: DBクライアント初期化

```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

const connectionString = process.env.DATABASE_URL!
const client = postgres(connectionString, { max: 10 })

export const db = drizzle(client, { schema })
```

### Step 6: マイグレーション

```bash
# マイグレーション生成
pnpm drizzle-kit generate

# マイグレーション適用
pnpm drizzle-kit migrate

# スキーマ変更をDBに直接反映（開発用）
pnpm drizzle-kit push

# DBスキーマからスキーマファイル生成
pnpm drizzle-kit pull

# Drizzle Studio（GUIツール）
pnpm drizzle-kit studio
```

## Query Patterns

### 基本CRUD

```typescript
import { eq, and, or, gt, like, inArray, desc, sql } from 'drizzle-orm'

// SELECT
const allUsers = await db.select().from(users)
const activeUsers = await db.select().from(users).where(eq(users.isActive, true))

// SELECT with conditions
const results = await db.select().from(users).where(
  and(
    eq(users.role, 'admin'),
    gt(users.createdAt, new Date('2024-01-01'))
  )
)

// INSERT
const [newUser] = await db.insert(users).values({
  email: 'test@example.com',
  name: 'Test User',
}).returning()

// UPDATE (immutable pattern: returns new object)
const [updated] = await db.update(users)
  .set({ name: 'New Name', updatedAt: new Date() })
  .where(eq(users.id, userId))
  .returning()

// DELETE
await db.delete(users).where(eq(users.id, userId))

// UPSERT
await db.insert(users)
  .values({ email: 'test@example.com', name: 'Test' })
  .onConflictDoUpdate({
    target: users.email,
    set: { name: 'Updated Name', updatedAt: new Date() },
  })
```

### リレーショナルクエリ（Query API）

```typescript
// ネストされたリレーション取得
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: {
      orderBy: (posts, { desc }) => [desc(posts.createdAt)],
      limit: 5,
    },
  },
  where: eq(users.isActive, true),
})

// 単一レコード取得
const user = await db.query.users.findFirst({
  where: eq(users.email, 'test@example.com'),
  with: { posts: true },
})
```

### 高度なクエリ

```typescript
// サブクエリ
const subquery = db.select({ authorId: posts.authorId })
  .from(posts)
  .groupBy(posts.authorId)
  .having(gt(sql`count(*)`, 5))
  .as('active_authors')

const activeAuthors = await db.select()
  .from(users)
  .innerJoin(subquery, eq(users.id, subquery.authorId))

// トランザクション
const result = await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({ email, name }).returning()
  await tx.insert(posts).values({ title, content, authorId: user.id })
  return user
})

// Prepared Statements
const getUserByEmail = db.query.users.findFirst({
  where: eq(users.email, sql.placeholder('email')),
}).prepare('get_user_by_email')

const user = await getUserByEmail.execute({ email: 'test@example.com' })

// カーソルページネーション
const nextPage = await db.select()
  .from(posts)
  .where(gt(posts.id, lastId))
  .orderBy(posts.id)
  .limit(20)
```

## インデックス定義

```typescript
import { pgTable, text, index, uniqueIndex } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull(),
  name: text('name').notNull(),
  tenantId: uuid('tenant_id').notNull(),
}, (table) => [
  uniqueIndex('users_email_idx').on(table.email),
  index('users_tenant_idx').on(table.tenantId),
  index('users_tenant_email_idx').on(table.tenantId, table.email),
])
```

## Best Practices

| Practice | Do | Don't |
|----------|-----|-------|
| スキーマ分割 | 機能ごとにファイル分割 | 1ファイルに全テーブル |
| 型推論 | `$inferSelect`/`$inferInsert` | 手動で型定義 |
| バリデーション | drizzle-zod連携 | スキーマと別にZod定義 |
| マイグレーション | `generate` + `migrate` | `push`を本番で使用 |
| ページネーション | カーソルベース | OFFSET/LIMIT |
| トランザクション | `db.transaction()` | 手動BEGIN/COMMIT |

## ファイル構成テンプレート

```
src/db/
├── index.ts              # DBクライアント初期化
├── schema/
│   ├── index.ts          # 全スキーマのre-export
│   ├── users.ts          # usersテーブル + 型
│   ├── posts.ts          # postsテーブル + 型
│   └── relations.ts      # リレーション定義
├── queries/
│   ├── users.ts          # ユーザー関連クエリ
│   └── posts.ts          # 投稿関連クエリ
└── migrations/           # 自動生成
drizzle.config.ts
```

## Related

- Skill: `postgres-patterns` - PostgreSQLクエリ最適化
- Skill: `supabase-rls` - Row Level Security設計
- Skill: `backend-patterns` - API設計パターン
