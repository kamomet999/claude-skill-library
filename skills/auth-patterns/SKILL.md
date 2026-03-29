---
name: auth-patterns
description: 認証・認可パターン集。JWT、OAuth2.0、RBAC、セッション管理、MFA、パスキー
category: バックエンド
command: /auth-patterns
version: 1.0.0
tags: [auth, jwt, oauth, rbac, session, mfa]
---

# 認証・認可パターン集

## 概要

プロダクション品質の認証・認可システムを構築するためのパターン集。
JWT、OAuth 2.0、ロールベースアクセス制御（RBAC）、セッション管理、多要素認証（MFA）、パスキーを網羅する。

## Steps

1. 認証方式を選択する（JWT vs セッション）
2. パスワードハッシュとトークン管理を実装する
3. アクセストークン / リフレッシュトークンのフローを構築する
4. RBAC（ロールベースアクセス制御）を実装する
5. OAuth 2.0ソーシャルログインを追加する
6. MFA（多要素認証）を実装する
7. セキュリティ強化（レート制限、ブルートフォース防止）
8. パスキー（WebAuthn）を検討する

## 認証方式の選択基準

| 方式 | ユースケース | メリット | デメリット |
|------|------------|---------|-----------|
| JWT | SPA、モバイル、マイクロサービス | ステートレス、スケーラブル | 即時無効化が困難 |
| セッション | 従来型Webアプリ | 即時無効化可能、シンプル | サーバー状態が必要 |
| OAuth 2.0 | ソーシャルログイン、API連携 | 標準化、委任認可 | 実装が複雑 |
| パスキー | パスワードレス認証 | フィッシング耐性、UX良好 | ブラウザ対応状況 |

## JWT認証の実装

```typescript
// src/auth/jwt.ts
import jwt from 'jsonwebtoken'
import { z } from 'zod'

const JWT_SECRET = process.env.JWT_SECRET
const JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET

if (!JWT_SECRET || !JWT_REFRESH_SECRET) {
  throw new Error('JWT_SECRET and JWT_REFRESH_SECRET must be set')
}

const tokenPayloadSchema = z.object({
  sub: z.string(),
  email: z.string().email(),
  roles: z.array(z.string()),
})

type TokenPayload = z.infer<typeof tokenPayloadSchema>

export function generateTokenPair(payload: TokenPayload) {
  const accessToken = jwt.sign(payload, JWT_SECRET, {
    expiresIn: '15m',          // 短い有効期限
    issuer: 'my-app',
    audience: 'my-app-client',
  })

  const refreshToken = jwt.sign(
    { sub: payload.sub, type: 'refresh' },
    JWT_REFRESH_SECRET,
    { expiresIn: '7d' }        // 長い有効期限
  )

  return { accessToken, refreshToken }
}

export function verifyAccessToken(token: string): TokenPayload {
  try {
    const decoded = jwt.verify(token, JWT_SECRET, {
      issuer: 'my-app',
      audience: 'my-app-client',
    })
    return tokenPayloadSchema.parse(decoded)
  } catch (error) {
    throw new Error('無効なアクセストークンです')
  }
}

export function verifyRefreshToken(token: string): { sub: string } {
  try {
    const decoded = jwt.verify(token, JWT_REFRESH_SECRET) as { sub: string; type: string }
    if (decoded.type !== 'refresh') {
      throw new Error('Invalid token type')
    }
    return { sub: decoded.sub }
  } catch (error) {
    throw new Error('無効なリフレッシュトークンです')
  }
}
```

## パスワードハッシュ

```typescript
// src/auth/password.ts
import argon2 from 'argon2'

// Argon2id推奨（bcryptより強力）
export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536,    // 64MB
    timeCost: 3,
    parallelism: 4,
  })
}

export async function verifyPassword(hash: string, password: string): Promise<boolean> {
  try {
    return await argon2.verify(hash, password)
  } catch {
    return false
  }
}
```

## リフレッシュトークンのローテーション

```typescript
// src/auth/token-service.ts
import { db } from '../db'
import { generateTokenPair, verifyRefreshToken } from './jwt'
import crypto from 'crypto'

export async function refreshTokens(currentRefreshToken: string) {
  // 1. リフレッシュトークンを検証
  const { sub: userId } = verifyRefreshToken(currentRefreshToken)

  // 2. DBに保存されたトークンと照合
  const tokenHash = crypto.createHash('sha256').update(currentRefreshToken).digest('hex')
  const storedToken = await db.refreshToken.findUnique({ where: { tokenHash } })

  if (!storedToken) {
    // トークン再利用攻撃の可能性 -> 全トークン無効化
    await db.refreshToken.deleteMany({ where: { userId } })
    throw new Error('トークンが無効化されています。再ログインしてください')
  }

  if (storedToken.expiresAt < new Date()) {
    await db.refreshToken.delete({ where: { id: storedToken.id } })
    throw new Error('リフレッシュトークンの期限が切れています')
  }

  // 3. 古いトークンを削除
  await db.refreshToken.delete({ where: { id: storedToken.id } })

  // 4. 新しいトークンペアを生成
  const user = await db.user.findUniqueOrThrow({ where: { id: userId } })
  const tokens = generateTokenPair({
    sub: user.id,
    email: user.email,
    roles: user.roles,
  })

  // 5. 新しいリフレッシュトークンをDBに保存
  const newTokenHash = crypto.createHash('sha256').update(tokens.refreshToken).digest('hex')
  await db.refreshToken.create({
    data: {
      tokenHash: newTokenHash,
      userId: user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  })

  return tokens
}
```

## RBAC（ロールベースアクセス制御）

```typescript
// src/auth/rbac.ts

// 権限定義
export const Permissions = {
  // ユーザー管理
  USER_READ: 'user:read',
  USER_WRITE: 'user:write',
  USER_DELETE: 'user:delete',
  // 商品管理
  PRODUCT_READ: 'product:read',
  PRODUCT_WRITE: 'product:write',
  PRODUCT_DELETE: 'product:delete',
  // 管理者
  ADMIN_PANEL: 'admin:panel',
  ADMIN_SETTINGS: 'admin:settings',
} as const

type Permission = typeof Permissions[keyof typeof Permissions]

// ロール -> 権限マッピング
const rolePermissions: Record<string, Permission[]> = {
  viewer: [
    Permissions.USER_READ,
    Permissions.PRODUCT_READ,
  ],
  editor: [
    Permissions.USER_READ,
    Permissions.USER_WRITE,
    Permissions.PRODUCT_READ,
    Permissions.PRODUCT_WRITE,
  ],
  admin: [
    // 全権限
    ...Object.values(Permissions),
  ],
}

export function hasPermission(userRoles: string[], requiredPermission: Permission): boolean {
  return userRoles.some((role) => {
    const permissions = rolePermissions[role]
    return permissions?.includes(requiredPermission) ?? false
  })
}

export function hasAnyPermission(userRoles: string[], requiredPermissions: Permission[]): boolean {
  return requiredPermissions.some((p) => hasPermission(userRoles, p))
}

export function hasAllPermissions(userRoles: string[], requiredPermissions: Permission[]): boolean {
  return requiredPermissions.every((p) => hasPermission(userRoles, p))
}

// Expressミドルウェア
import { Request, Response, NextFunction } from 'express'

export function requirePermission(...permissions: Permission[]) {
  return (req: Request, _res: Response, next: NextFunction) => {
    const user = req.user
    if (!user) {
      return next(new AppError(401, '認証が必要です'))
    }
    if (!hasAllPermissions(user.roles, permissions)) {
      return next(new AppError(403, 'アクセス権限がありません'))
    }
    next()
  }
}

// 使用例
// router.delete('/users/:id', requirePermission(Permissions.USER_DELETE), deleteUser)
```

## OAuth 2.0（Google認証）

```typescript
// src/auth/oauth/google.ts
import { OAuth2Client } from 'google-auth-library'
import { db } from '../../db'
import { generateTokenPair } from '../jwt'

const client = new OAuth2Client({
  clientId: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  redirectUri: process.env.GOOGLE_REDIRECT_URI,
})

export function getAuthUrl(state: string): string {
  return client.generateAuthUrl({
    access_type: 'offline',
    scope: ['openid', 'email', 'profile'],
    state,   // CSRF防止
    prompt: 'consent',
  })
}

export async function handleCallback(code: string) {
  // 1. 認可コードをトークンに交換
  const { tokens } = await client.getToken(code)
  client.setCredentials(tokens)

  // 2. IDトークンからユーザー情報を取得
  const ticket = await client.verifyIdToken({
    idToken: tokens.id_token!,
    audience: process.env.GOOGLE_CLIENT_ID,
  })
  const payload = ticket.getPayload()
  if (!payload?.email) throw new Error('メールアドレスが取得できません')

  // 3. ユーザーをupsert
  const user = await db.user.upsert({
    where: { email: payload.email },
    update: {
      name: payload.name ?? undefined,
      avatar: payload.picture ?? undefined,
      googleId: payload.sub,
    },
    create: {
      email: payload.email,
      name: payload.name ?? 'Unknown',
      avatar: payload.picture ?? null,
      googleId: payload.sub,
      roles: ['viewer'],
    },
  })

  // 4. JWTを発行
  return generateTokenPair({
    sub: user.id,
    email: user.email,
    roles: user.roles,
  })
}
```

## MFA（TOTP）

```typescript
// src/auth/mfa.ts
import { authenticator } from 'otplib'
import QRCode from 'qrcode'
import { db } from '../db'

// MFAセットアップ（QRコード生成）
export async function setupMFA(userId: string) {
  const user = await db.user.findUniqueOrThrow({ where: { id: userId } })
  const secret = authenticator.generateSecret()

  // シークレットを暗号化して保存（まだ有効化しない）
  await db.user.update({
    where: { id: userId },
    data: { mfaSecret: encrypt(secret), mfaEnabled: false },
  })

  const otpauthUrl = authenticator.keyuri(user.email, 'MyApp', secret)
  const qrCodeDataUrl = await QRCode.toDataURL(otpauthUrl)

  return { secret, qrCodeDataUrl }
}

// MFA検証 & 有効化
export async function verifyAndEnableMFA(userId: string, token: string): Promise<boolean> {
  const user = await db.user.findUniqueOrThrow({ where: { id: userId } })
  if (!user.mfaSecret) throw new Error('MFAが設定されていません')

  const secret = decrypt(user.mfaSecret)
  const isValid = authenticator.verify({ token, secret })

  if (isValid) {
    // バックアップコードを生成
    const backupCodes = Array.from({ length: 10 }, () =>
      crypto.randomBytes(4).toString('hex')
    )

    await db.user.update({
      where: { id: userId },
      data: {
        mfaEnabled: true,
        mfaBackupCodes: backupCodes.map((c) => hashSync(c)),
      },
    })

    return true
  }

  return false
}

// ログイン時のMFA検証
export async function verifyMFAToken(userId: string, token: string): Promise<boolean> {
  const user = await db.user.findUniqueOrThrow({ where: { id: userId } })
  if (!user.mfaSecret || !user.mfaEnabled) {
    throw new Error('MFAが有効化されていません')
  }

  const secret = decrypt(user.mfaSecret)

  // TOTPトークンの検証
  if (authenticator.verify({ token, secret })) {
    return true
  }

  // バックアップコードの検証
  const matchIndex = user.mfaBackupCodes.findIndex((hash) => compareSync(token, hash))
  if (matchIndex >= 0) {
    // 使用済みのバックアップコードを削除
    const updatedCodes = [...user.mfaBackupCodes]
    updatedCodes.splice(matchIndex, 1)
    await db.user.update({
      where: { id: userId },
      data: { mfaBackupCodes: updatedCodes },
    })
    return true
  }

  return false
}
```

## 認証ミドルウェア（Express）

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express'
import { verifyAccessToken } from '../auth/jwt'

declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string
        email: string
        roles: string[]
      }
    }
  }
}

export function authenticate(req: Request, _res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization
  if (!authHeader?.startsWith('Bearer ')) {
    return next(new AppError(401, '認証トークンが必要です'))
  }

  try {
    const token = authHeader.slice(7)
    const payload = verifyAccessToken(token)
    req.user = {
      id: payload.sub,
      email: payload.email,
      roles: payload.roles,
    }
    next()
  } catch {
    next(new AppError(401, '無効または期限切れのトークンです'))
  }
}
```

## セキュリティチェックリスト

```typescript
// ログインエンドポイントの完全な実装例
export const login = asyncHandler(async (req: Request, res: Response) => {
  const { email, password, mfaToken } = req.body

  // 1. ユーザー検索（存在しなくてもタイミング攻撃を防ぐ）
  const user = await db.user.findUnique({ where: { email } })
  const dummyHash = '$argon2id$v=19$m=65536,t=3,p=4$dummy'
  const isValid = await verifyPassword(user?.passwordHash ?? dummyHash, password)

  if (!user || !isValid) {
    // 具体的な理由を明かさない
    throw new AppError(401, 'メールアドレスまたはパスワードが正しくありません')
  }

  // 2. アカウントロック確認
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    throw new AppError(423, 'アカウントが一時的にロックされています')
  }

  // 3. MFA確認
  if (user.mfaEnabled) {
    if (!mfaToken) {
      return res.json({ success: true, requiresMFA: true })
    }
    const mfaValid = await verifyMFAToken(user.id, mfaToken)
    if (!mfaValid) {
      throw new AppError(401, 'MFAコードが正しくありません')
    }
  }

  // 4. ログイン失敗回数リセット
  await db.user.update({
    where: { id: user.id },
    data: { failedLoginAttempts: 0, lockedUntil: null },
  })

  // 5. トークン発行
  const tokens = generateTokenPair({
    sub: user.id,
    email: user.email,
    roles: user.roles,
  })

  // 6. リフレッシュトークンをHttpOnly Cookieで返す
  res.cookie('refreshToken', tokens.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,
    path: '/api/auth/refresh',
  })

  res.json({ success: true, data: { accessToken: tokens.accessToken } })
})
```

## ベストプラクティス

1. **パスワードハッシュはArgon2id**: bcryptよりメモリハード関数が望ましい
2. **リフレッシュトークンのローテーション**: 使用するたびに新しいトークンを発行する
3. **トークン再利用検知**: 再利用を検出したら全トークンを無効化する
4. **HttpOnly Cookie**: リフレッシュトークンはJSからアクセス不可にする
5. **タイミング攻撃防止**: ユーザーが存在しない場合もハッシュ比較を行う
6. **MFAバックアップコード**: TOTP端末紛失に備えてバックアップコードを発行する
7. **アカウントロック**: 5回失敗で15分ロック
8. **エラーメッセージ**: 「メールが存在しません」ではなく「認証情報が正しくありません」

## アンチパターン

- JWTにパスワードや機密情報を含める
- アクセストークンの有効期限を長くする（15分以内推奨）
- リフレッシュトークンをlocalStorageに保存する（XSS脆弱性）
- パスワードをSHA256でハッシュする（レインボーテーブル攻撃に弱い）
- ログインエラーで「パスワードが違います」と返す（アカウント列挙攻撃）
- CORSを `*` に設定する（認証付きAPIでは必ずoriginを指定）
