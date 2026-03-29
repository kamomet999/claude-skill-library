---
name: api-versioning
description: API バージョニング戦略、後方互換性、マイグレーション、deprecation
category: バックエンド
command: /api-versioning
version: 1.0.0
tags: [api, versioning, migration, backward-compatibility]
---

# API バージョニング戦略

## 概要

REST APIのバージョニング戦略を設計するためのパターン集。
URLパス方式、ヘッダー方式、後方互換性の維持、deprecationプロセス、クライアントマイグレーション支援を網羅する。

## Steps

1. バージョニング方式を選択する
2. ルーティング構造を設計する
3. 後方互換性ルールを定義する
4. deprecationプロセスを確立する
5. バージョン間のマイグレーションレイヤーを実装する
6. APIドキュメントのバージョン管理を行う
7. クライアント通知とマイグレーション支援を実装する
8. バージョン廃止スケジュールを管理する

## バージョニング方式の比較

| 方式 | 例 | メリット | デメリット |
|------|------|---------|-----------|
| URLパス | `/api/v1/users` | 明確、キャッシュ容易 | URLが変わる |
| ヘッダー | `Accept: application/vnd.myapp.v1+json` | URLクリーン | 発見しにくい |
| クエリパラメータ | `/api/users?version=1` | 簡単に切り替え | キャッシュが困難 |
| コンテンツネゴシエーション | `Accept: application/json; version=1` | 標準的 | 実装が複雑 |

**推奨: URLパス方式**（最も明確で、デバッグしやすく、キャッシュフレンドリー）

## プロジェクト構造

```
src/
  routes/
    v1/
      index.ts
      users.routes.ts
      products.routes.ts
    v2/
      index.ts
      users.routes.ts       # v2で変更されたエンドポイント
      products.routes.ts    # v1から再エクスポート（変更なし）
    router.ts               # バージョンルーター
  middleware/
    version.ts              # バージョン検出ミドルウェア
    deprecation.ts          # deprecation警告ミドルウェア
  transformers/
    v1-to-v2/
      user.transformer.ts   # v1 <-> v2 の変換
  versioning/
    config.ts               # バージョン設定
    changelog.ts            # 変更履歴
```

## バージョンルーター

```typescript
// src/routes/router.ts
import { Router } from 'express'
import { v1Routes } from './v1'
import { v2Routes } from './v2'
import { deprecationWarning } from '../middleware/deprecation'
import { versionConfig } from '../versioning/config'

const router = Router()

// v1（deprecation警告付き）
router.use('/v1', deprecationWarning('v1'), v1Routes)

// v2（現在のバージョン）
router.use('/v2', v2Routes)

// バージョン未指定 -> 最新バージョンにリダイレクト
router.use('/', (req, res) => {
  const latestVersion = versionConfig.current
  res.redirect(308, `/api/${latestVersion}${req.path}`)
})

export { router as apiRouter }
```

## バージョン設定

```typescript
// src/versioning/config.ts

interface VersionInfo {
  version: string
  status: 'current' | 'deprecated' | 'retired'
  releasedAt: string
  deprecatedAt?: string
  sunsetAt?: string
  changelog: string
}

export const versionConfig = {
  current: 'v2',
  supported: ['v1', 'v2'],
  versions: {
    v1: {
      version: 'v1',
      status: 'deprecated' as const,
      releasedAt: '2024-01-15',
      deprecatedAt: '2025-06-01',
      sunsetAt: '2025-12-01',      // この日以降は404を返す
      changelog: '/docs/changelog/v1',
    },
    v2: {
      version: 'v2',
      status: 'current' as const,
      releasedAt: '2025-06-01',
      changelog: '/docs/changelog/v2',
    },
  } as Record<string, VersionInfo>,
}
```

## Deprecation警告ミドルウェア

```typescript
// src/middleware/deprecation.ts
import { Request, Response, NextFunction } from 'express'
import { versionConfig } from '../versioning/config'

export function deprecationWarning(version: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const info = versionConfig.versions[version]
    if (!info) return next()

    // 廃止済みバージョン
    if (info.status === 'retired') {
      return res.status(410).json({
        success: false,
        error: `API ${version} は廃止されました`,
        code: 'VERSION_RETIRED',
        migration: {
          currentVersion: versionConfig.current,
          migrationGuide: `/docs/migration/${version}-to-${versionConfig.current}`,
        },
      })
    }

    // 非推奨バージョン
    if (info.status === 'deprecated') {
      // RFC 8594 Sunset Header
      if (info.sunsetAt) {
        res.set('Sunset', new Date(info.sunsetAt).toUTCString())
      }

      // Deprecation Header
      if (info.deprecatedAt) {
        res.set('Deprecation', new Date(info.deprecatedAt).toUTCString())
      }

      // Link to newer version
      res.set('Link', `</api/${versionConfig.current}>; rel="successor-version"`)

      // カスタム警告ヘッダー
      res.set('X-API-Warn', `API ${version} は ${info.sunsetAt} に廃止予定です。${versionConfig.current} へ移行してください。`)
    }

    next()
  }
}
```

## バージョン間トランスフォーマー

```typescript
// src/transformers/v1-to-v2/user.transformer.ts

// v1のユーザー型
interface UserV1 {
  id: string
  name: string           // v1: 単一フィールド
  email: string
  phone: string | null   // v1: 任意
  created_at: string     // v1: snake_case
}

// v2のユーザー型
interface UserV2 {
  id: string
  firstName: string      // v2: 分割
  lastName: string       // v2: 分割
  email: string
  phoneNumber: string | null  // v2: リネーム
  createdAt: string      // v2: camelCase
  updatedAt: string      // v2: 追加
}

// v2 -> v1 変換（古いクライアント向け）
export function userV2toV1(user: UserV2): UserV1 {
  return {
    id: user.id,
    name: `${user.firstName} ${user.lastName}`,
    email: user.email,
    phone: user.phoneNumber,
    created_at: user.createdAt,
  }
}

// v1リクエスト -> v2リクエスト変換
export function createUserV1toV2(input: { name: string; email: string }) {
  const [firstName, ...rest] = input.name.split(' ')
  return {
    firstName,
    lastName: rest.join(' ') || '',
    email: input.email,
  }
}
```

## v1ルートの実装（トランスフォーマー経由）

```typescript
// src/routes/v1/users.routes.ts
import { Router } from 'express'
import { asyncHandler } from '../../utils/async-handler'
import { userService } from '../../services/user.service'
import { userV2toV1, createUserV1toV2 } from '../../transformers/v1-to-v2/user.transformer'

const router = Router()

// v1 GET /users/:id -> 内部はv2のサービスを使用し、レスポンスをv1形式に変換
router.get('/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id) // v2形式で取得
  if (!user) {
    return res.status(404).json({ success: false, error: 'User not found' })
  }
  res.json({ success: true, data: userV2toV1(user) })
}))

// v1 POST /users -> v1入力をv2形式に変換して処理
router.post('/', asyncHandler(async (req, res) => {
  const v2Input = createUserV1toV2(req.body)
  const user = await userService.create(v2Input)
  res.status(201).json({ success: true, data: userV2toV1(user) })
}))

export { router as usersV1Routes }
```

## 後方互換性ルール

```typescript
// 破壊的変更（メジャーバージョンアップが必要）
const breakingChanges = [
  'フィールドの削除',
  'フィールドの型変更',
  'フィールド名の変更',
  '必須フィールドの追加（リクエスト）',
  'ステータスコードの変更',
  'エラーコードの変更',
  'エンドポイントURLの変更',
  '認証方式の変更',
]

// 非破壊的変更（マイナーアップデートで可能）
const nonBreakingChanges = [
  'オプショナルフィールドの追加（レスポンス）',
  'オプショナルパラメータの追加（リクエスト）',
  '新しいエンドポイントの追加',
  '新しいHTTPメソッドの追加',
  'エラーメッセージの文言変更',
  'パフォーマンス改善',
  'バグ修正',
]
```

## エンドポイント単位のdeprecation

```typescript
// 個別エンドポイントのdeprecation（バージョン全体ではなく）
import { Request, Response, NextFunction } from 'express'

export function deprecatedEndpoint(
  alternative: string,
  sunsetDate: string
) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.set('Sunset', new Date(sunsetDate).toUTCString())
    res.set('Deprecation', 'true')
    res.set('Link', `<${alternative}>; rel="successor-version"`)
    res.set(
      'X-API-Warn',
      `このエンドポイントは${sunsetDate}に廃止予定です。代わりに ${alternative} を使用してください。`
    )
    next()
  }
}

// 使用例
router.get(
  '/users/search',
  deprecatedEndpoint('/api/v2/users?filter=...', '2025-12-01'),
  searchUsersLegacy
)
```

## バージョン情報エンドポイント

```typescript
// GET /api/versions
router.get('/versions', (req, res) => {
  res.json({
    success: true,
    data: {
      current: versionConfig.current,
      supported: versionConfig.supported,
      versions: Object.values(versionConfig.versions).map((v) => ({
        version: v.version,
        status: v.status,
        releasedAt: v.releasedAt,
        sunsetAt: v.sunsetAt ?? null,
        changelog: v.changelog,
      })),
    },
  })
})
```

## マイグレーションガイドテンプレート

```markdown
# v1 -> v2 マイグレーションガイド

## タイムライン
- v2リリース: 2025-06-01
- v1 deprecation: 2025-06-01
- v1 sunset: 2025-12-01

## 破壊的変更

### 1. ユーザーの `name` フィールドが `firstName` / `lastName` に分割
**v1**: `{ "name": "山田 太郎" }`
**v2**: `{ "firstName": "太郎", "lastName": "山田" }`

### 2. snake_caseからcamelCaseへ変更
**v1**: `created_at`, `updated_at`
**v2**: `createdAt`, `updatedAt`

## 非破壊的変更
- `updatedAt` フィールドの追加（レスポンス）
- `/api/v2/users/bulk` エンドポイントの追加
```

## ベストプラクティス

1. **URLパス方式を使う**: 最も明確でデバッグしやすい
2. **メジャーバージョンのみ**: v1, v2, v3（v1.1のようなマイナーは不要）
3. **最低2バージョン同時サポート**: 移行期間を十分に設ける
4. **Sunset Headerを必ず返す**: RFC 8594に準拠する
5. **サービス層は共有**: ルート/トランスフォーマーだけバージョン分離する
6. **変更なしのエンドポイントは再エクスポート**: コピーしない
7. **マイグレーションガイド必須**: 各バージョンアップ時に作成する
8. **テレメトリで使用状況を監視**: 古いバージョンの利用率を追跡する

## アンチパターン

- バージョンごとにサービス層をコピーする（保守不能になる）
- 後方互換性のない変更をマイナーアップデートで行う
- deprecation警告なしで突然バージョンを廃止する
- 3つ以上のバージョンを同時にサポートする（保守コスト爆発）
- クエリパラメータでバージョニングする（キャッシュが効かない）
