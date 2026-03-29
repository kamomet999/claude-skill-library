---
name: graphql-patterns
description: GraphQLスキーマ設計、N+1問題解決、DataLoader、サブスクリプション、フラグメント設計
category: バックエンド
command: /graphql-patterns
version: 1.0.0
tags: [graphql, api, schema, dataloader]
---

# GraphQL 設計パターン

## 概要

GraphQL APIをプロダクション品質で構築するためのパターン集。
スキーマ設計の原則、N+1問題の根本解決、DataLoaderによるバッチ処理、リアルタイムサブスクリプション、再利用可能なフラグメント設計を網羅する。

## Steps

1. スキーマファースト設計でAPIの型を定義する
2. リゾルバーをレイヤード構成で実装する
3. DataLoaderでN+1問題を解決する
4. 入力バリデーションを追加する
5. 認証・認可をディレクティブで実装する
6. サブスクリプションでリアルタイム機能を追加する
7. エラーハンドリングを統一する
8. パフォーマンス最適化（クエリ複雑度制限、永続化クエリ）

## プロジェクト構造

```
src/
  schema/
    typeDefs/
      user.graphql
      product.graphql
      common.graphql       # Pagination, Error等の共通型
    resolvers/
      user.resolver.ts
      product.resolver.ts
    directives/
      auth.directive.ts
      deprecated.directive.ts
    index.ts               # スキーマ統合
  dataloaders/
    user.loader.ts
    product.loader.ts
    index.ts               # DataLoader集約
  services/
    user.service.ts
    product.service.ts
  subscriptions/
    pubsub.ts
    user.subscription.ts
```

## スキーマ設計の原則

```graphql
# schema/typeDefs/common.graphql

# Relay仕様のページネーション
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# 統一エラー型
type FieldError {
  field: String!
  message: String!
}

# Mutation結果の共通インターフェース
interface MutationResult {
  success: Boolean!
  errors: [FieldError!]
}

# ノードインターフェース（グローバルID）
interface Node {
  id: ID!
}
```

```graphql
# schema/typeDefs/user.graphql

type User implements Node {
  id: ID!
  email: String!
  name: String!
  avatar: String
  posts(first: Int, after: String): PostConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  cursor: String!
  node: User!
}

# 入力型はInputサフィックス
input CreateUserInput {
  email: String!
  name: String!
  password: String!
}

input UpdateUserInput {
  name: String
  avatar: String
}

# Mutation結果はPayloadサフィックス
type CreateUserPayload implements MutationResult {
  success: Boolean!
  errors: [FieldError!]
  user: User
}

type Query {
  user(id: ID!): User
  users(first: Int, after: String, filter: UserFilter): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
}

type Subscription {
  userCreated: User!
  userUpdated(id: ID!): User!
}
```

## DataLoader による N+1 問題の解決

```typescript
// src/dataloaders/user.loader.ts
import DataLoader from 'dataloader'
import { db } from '../db'

// バッチ関数: IDの配列を受け取り、同じ順序で結果を返す
async function batchUsers(ids: readonly string[]) {
  const users = await db.user.findMany({
    where: { id: { in: [...ids] } },
  })

  // IDの順序を維持（DataLoaderの要件）
  const userMap = new Map(users.map((u) => [u.id, u]))
  return ids.map((id) => userMap.get(id) ?? null)
}

export function createUserLoader() {
  return new DataLoader(batchUsers, {
    // キャッシュはリクエスト単位（メモリリーク防止）
    cache: true,
    // 最大バッチサイズ
    maxBatchSize: 100,
  })
}

// src/dataloaders/index.ts
import { createUserLoader } from './user.loader'
import { createPostLoader } from './post.loader'

export interface DataLoaders {
  userLoader: ReturnType<typeof createUserLoader>
  postLoader: ReturnType<typeof createPostLoader>
}

// リクエストごとに新しいDataLoaderを作成
export function createLoaders(): DataLoaders {
  return {
    userLoader: createUserLoader(),
    postLoader: createPostLoader(),
  }
}
```

```typescript
// コンテキスト設定
import { createLoaders } from './dataloaders'

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => ({
    loaders: createLoaders(),  // リクエスト毎に新規作成
    user: getUserFromToken(req.headers.authorization),
  }),
})
```

## リゾルバーパターン

```typescript
// src/schema/resolvers/user.resolver.ts
import { GraphQLContext } from '../types'
import { AppError } from '../../utils/app-error'
import { userService } from '../../services/user.service'
import { pubsub, Events } from '../../subscriptions/pubsub'

export const userResolvers = {
  Query: {
    user: async (_parent: unknown, { id }: { id: string }, ctx: GraphQLContext) => {
      return ctx.loaders.userLoader.load(id)
    },

    users: async (_parent: unknown, args: UserConnectionArgs, ctx: GraphQLContext) => {
      return userService.findConnection(args)
    },
  },

  Mutation: {
    createUser: async (_parent: unknown, { input }: { input: CreateUserInput }, ctx: GraphQLContext) => {
      try {
        const user = await userService.create(input)

        // サブスクリプション通知
        pubsub.publish(Events.USER_CREATED, { userCreated: user })

        return { success: true, errors: null, user }
      } catch (error) {
        if (error instanceof AppError) {
          return {
            success: false,
            errors: [{ field: 'email', message: error.message }],
            user: null,
          }
        }
        throw error
      }
    },
  },

  // フィールドリゾルバー（N+1をDataLoaderで解決）
  User: {
    posts: async (parent: User, args: ConnectionArgs, ctx: GraphQLContext) => {
      return ctx.loaders.postsByUserLoader.load({
        userId: parent.id,
        ...args,
      })
    },
  },

  Subscription: {
    userCreated: {
      subscribe: () => pubsub.asyncIterableIterator([Events.USER_CREATED]),
    },
    userUpdated: {
      subscribe: (_parent: unknown, { id }: { id: string }) => {
        return pubsub.asyncIterableIterator([`${Events.USER_UPDATED}.${id}`])
      },
    },
  },
}
```

## 認証ディレクティブ

```typescript
// src/schema/directives/auth.directive.ts
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils'
import { defaultFieldResolver, GraphQLSchema } from 'graphql'
import { AppError } from '../../utils/app-error'

export function authDirectiveTransformer(schema: GraphQLSchema): GraphQLSchema {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const authDirective = getDirective(schema, fieldConfig, 'auth')?.[0]
      if (!authDirective) return fieldConfig

      const requiredRole = authDirective['requires'] as string | undefined
      const { resolve = defaultFieldResolver } = fieldConfig

      fieldConfig.resolve = async (source, args, context, info) => {
        if (!context.user) {
          throw new AppError(401, '認証が必要です')
        }
        if (requiredRole && context.user.role !== requiredRole) {
          throw new AppError(403, 'アクセス権限がありません')
        }
        return resolve(source, args, context, info)
      }

      return fieldConfig
    },
  })
}

// スキーマでの使用
// directive @auth(requires: Role) on FIELD_DEFINITION
// type Query {
//   me: User! @auth
//   adminDashboard: Dashboard! @auth(requires: ADMIN)
// }
```

## クエリ複雑度制限

```typescript
// クエリの深さ・複雑度を制限してDoSを防ぐ
import depthLimit from 'graphql-depth-limit'
import { createComplexityLimitRule } from 'graphql-validation-complexity'

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(10),  // ネスト深さ上限
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],
})
```

## サブスクリプション（PubSub）

```typescript
// src/subscriptions/pubsub.ts
import { RedisPubSub } from 'graphql-redis-subscriptions'
import Redis from 'ioredis'

const options = {
  host: process.env.REDIS_HOST ?? 'localhost',
  port: Number(process.env.REDIS_PORT) ?? 6379,
  retryStrategy: (times: number) => Math.min(times * 50, 2000),
}

// 本番ではRedis PubSubを使用（複数サーバー対応）
export const pubsub = new RedisPubSub({
  publisher: new Redis(options),
  subscriber: new Redis(options),
})

export const Events = {
  USER_CREATED: 'USER_CREATED',
  USER_UPDATED: 'USER_UPDATED',
  POST_CREATED: 'POST_CREATED',
} as const
```

## フラグメント設計（クライアント側）

```graphql
# 再利用可能なフラグメント
fragment UserBasic on User {
  id
  name
  avatar
}

fragment UserFull on User {
  ...UserBasic
  email
  createdAt
  updatedAt
}

fragment PostWithAuthor on Post {
  id
  title
  content
  author {
    ...UserBasic
  }
}

# クエリでの使用
query GetTimeline($first: Int!, $after: String) {
  timeline(first: $first, after: $after) {
    edges {
      node {
        ...PostWithAuthor
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

## エラーハンドリング統一

```typescript
// カスタムエラーフォーマッター
import { GraphQLFormattedError } from 'graphql'

function formatError(formattedError: GraphQLFormattedError, error: unknown): GraphQLFormattedError {
  // バリデーションエラーはそのまま返す
  if (formattedError.extensions?.code === 'GRAPHQL_VALIDATION_FAILED') {
    return formattedError
  }

  // 本番では内部エラーの詳細を隠す
  if (process.env.NODE_ENV === 'production') {
    if (!formattedError.extensions?.code || formattedError.extensions.code === 'INTERNAL_SERVER_ERROR') {
      return {
        message: 'サーバー内部エラーが発生しました',
        extensions: { code: 'INTERNAL_SERVER_ERROR' },
      }
    }
  }

  return formattedError
}
```

## ベストプラクティス

1. **スキーマファースト**: 型定義を先に書き、実装は後から合わせる
2. **Relay仕様のページネーション**: Connection/Edge/PageInfo パターンを使う
3. **DataLoaderは必須**: N+1問題は必ず発生する。初日から導入する
4. **Mutation結果型**: `{ success, errors, data }` で統一する
5. **入力型の分離**: Create/Update で別の Input 型を定義する
6. **グローバルID**: Node インターフェースで型安全なID体系を作る
7. **クエリ複雑度制限**: 深さ制限と複雑度制限でDoSを防ぐ
8. **永続化クエリ**: 本番ではクエリのホワイトリストを使う

## アンチパターン

- リゾルバー内でSQLを直接書く（サービス層に分離せよ）
- DataLoaderをグローバルに1つだけ作る（リクエスト毎に作成せよ）
- 全フィールドをnon-nullにする（進化可能性が失われる）
- RESTのエンドポイント構造をそのままGraphQLに移植する
- サブスクリプションをポーリングの代替としてだけ使う
