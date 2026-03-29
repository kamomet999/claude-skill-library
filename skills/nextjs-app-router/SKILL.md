---
name: nextjs-app-router
description: Next.js App Router完全ガイド。Server Components、Streaming、並列ルート、インターセプト、キャッシュ戦略の実践ガイド
category: フロントエンド
command: nextjs-app-router
version: 1.0.0
tags:
  - nextjs
  - app-router
  - rsc
  - streaming
  - cache
---

# Next.js App Router 完全ガイド

## 概要

Next.js App Routerの設計パターンと実装ガイド。
React Server Components、Streaming SSR、並列ルート、インターセプトルート、キャッシュ戦略を体系的にカバーする。

## Steps

### Step 1: ディレクトリ構成

```
src/
├── app/
│   ├── layout.tsx              # ルートレイアウト（必須）
│   ├── page.tsx                # / ページ
│   ├── loading.tsx             # Suspense fallback
│   ├── error.tsx               # エラーバウンダリ
│   ├── not-found.tsx           # 404ページ
│   ├── global-error.tsx        # ルートレベルエラー
│   ├── (marketing)/            # ルートグループ（URLに影響しない）
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── about/page.tsx
│   ├── (app)/                  # アプリケーション部分
│   │   ├── layout.tsx          # 認証レイアウト
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── @analytics/     # 並列ルート
│   │   │   │   ├── page.tsx
│   │   │   │   └── loading.tsx
│   │   │   └── @activity/      # 並列ルート
│   │   │       ├── page.tsx
│   │   │       └── loading.tsx
│   │   └── products/
│   │       ├── page.tsx
│   │       ├── [id]/
│   │       │   └── page.tsx
│   │       └── (..)products/[id]/ # インターセプトルート
│   │           └── page.tsx
│   └── api/
│       └── [...route]/route.ts
├── components/
│   ├── ui/                     # 汎用UIコンポーネント
│   └── features/               # 機能別コンポーネント
├── lib/                        # ユーティリティ
└── types/                      # 型定義
```

### Step 2: Server Components vs Client Components

```tsx
// === Server Component（デフォルト） ===
// データフェッチ、重い依存、機密データにアクセス可能
// app/products/page.tsx
async function ProductsPage() {
  // サーバーで直接データベースにアクセス
  const products = await db.product.findMany({
    orderBy: { createdAt: 'desc' },
    take: 20,
  })

  return (
    <div>
      <h1>商品一覧</h1>
      {/* Server Componentのまま渡せる */}
      <ProductList products={products} />
      {/* インタラクティブ部分だけClient Component */}
      <ProductFilter />
    </div>
  )
}

// === Client Component ===
// 'use client'ディレクティブが必要
// useState, useEffect, イベントハンドラ、ブラウザAPI
'use client'

import { useState } from 'react'

function ProductFilter() {
  const [query, setQuery] = useState('')

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="検索..."
    />
  )
}
```

**判断基準**:

```
Server Component を使う:
  - データフェッチ
  - バックエンドリソースへのアクセス
  - 機密情報（APIキー、トークン）
  - 大きな依存パッケージ（バンドルに含まれない）
  - SEOが必要なコンテンツ

Client Component を使う:
  - useState, useEffect, useReducer
  - イベントハンドラ (onClick, onChange)
  - ブラウザAPI (window, localStorage)
  - カスタムフック
  - React.createContext
```

### Step 3: データフェッチパターン

```tsx
// Server Componentでの直接フェッチ
async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await getProduct(id)

  if (!product) notFound()

  return <ProductDetail product={product} />
}

// 並列データフェッチ（Promise.all）
async function DashboardPage() {
  // 並列に実行される
  const [stats, recentOrders, topProducts] = await Promise.all([
    getStats(),
    getRecentOrders(),
    getTopProducts(),
  ])

  return (
    <div>
      <StatsCards stats={stats} />
      <RecentOrders orders={recentOrders} />
      <TopProducts products={topProducts} />
    </div>
  )
}

// Streamingで段階的表示
async function DashboardPage() {
  // statsは即座に表示、他はストリーミング
  const stats = await getStats()

  return (
    <div>
      <StatsCards stats={stats} />
      <Suspense fallback={<OrdersSkeleton />}>
        <RecentOrdersAsync />
      </Suspense>
      <Suspense fallback={<ProductsSkeleton />}>
        <TopProductsAsync />
      </Suspense>
    </div>
  )
}

async function RecentOrdersAsync() {
  const orders = await getRecentOrders() // 遅いクエリ
  return <RecentOrders orders={orders} />
}
```

### Step 4: キャッシュ戦略

```tsx
// fetch APIのキャッシュ制御
// 静的データ（ビルド時にキャッシュ）
const data = await fetch('https://api.example.com/products', {
  cache: 'force-cache', // デフォルト
})

// 動的データ（毎リクエスト取得）
const data = await fetch('https://api.example.com/products', {
  cache: 'no-store',
})

// 時間ベースの再検証
const data = await fetch('https://api.example.com/products', {
  next: { revalidate: 3600 }, // 1時間ごとに再検証
})

// タグベースの再検証
const data = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] },
})

// Server Actionでの再検証トリガー
'use server'

import { revalidateTag, revalidatePath } from 'next/cache'

async function updateProduct(id: string, data: ProductData) {
  await db.product.update({ where: { id }, data })

  revalidateTag('products')          // タグで再検証
  revalidatePath('/products')        // パスで再検証
  revalidatePath('/products/[id]', 'page') // 動的パス
}
```

**unstable_cache（データベースクエリのキャッシュ）**:

```tsx
import { unstable_cache } from 'next/cache'

const getProducts = unstable_cache(
  async (category: string) => {
    return db.product.findMany({
      where: { category },
      orderBy: { createdAt: 'desc' },
    })
  },
  ['products'],           // キャッシュキー
  {
    tags: ['products'],   // 再検証タグ
    revalidate: 3600,     // 秒数
  }
)
```

### Step 5: 並列ルート

```tsx
// app/(app)/dashboard/layout.tsx
// 複数のスロットを同時にレンダリング
function DashboardLayout({
  children,
  analytics,
  activity,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  activity: React.ReactNode
}) {
  return (
    <div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
      <div className="lg:col-span-2">{children}</div>
      <div className="space-y-6">
        {analytics}
        {activity}
      </div>
    </div>
  )
}

// 各スロットは独立してloading.tsxを持てる
// app/(app)/dashboard/@analytics/loading.tsx
function AnalyticsLoading() {
  return <Skeleton className="h-64" />
}
```

### Step 6: Server Actions

```tsx
// app/products/actions.ts
'use server'

import { z } from 'zod'
import { revalidatePath } from 'next/cache'

const createProductSchema = z.object({
  name: z.string().min(1).max(100),
  price: z.number().positive(),
  description: z.string().max(1000),
})

export async function createProduct(formData: FormData) {
  // バリデーション
  const parsed = createProductSchema.safeParse({
    name: formData.get('name'),
    price: Number(formData.get('price')),
    description: formData.get('description'),
  })

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }

  // データベース操作
  try {
    await db.product.create({ data: parsed.data })
    revalidatePath('/products')
    return { success: true }
  } catch (error) {
    return { error: { _form: ['商品の作成に失敗しました'] } }
  }
}

// Client Componentでの使用
'use client'

import { useActionState } from 'react'
import { createProduct } from './actions'

function CreateProductForm() {
  const [state, formAction, isPending] = useActionState(createProduct, null)

  return (
    <form action={formAction}>
      <input name="name" required />
      {state?.error?.name && <p className="text-red-500">{state.error.name}</p>}

      <input name="price" type="number" required />

      <textarea name="description" />

      <button type="submit" disabled={isPending}>
        {isPending ? '作成中...' : '商品を作成'}
      </button>
    </form>
  )
}
```

### Step 7: Middleware

```ts
// middleware.ts（プロジェクトルート）
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // 認証チェック
  const token = request.cookies.get('session')?.value
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // ロケールリダイレクト
  const locale = request.headers.get('accept-language')?.split(',')[0]?.split('-')[0]
  if (pathname === '/' && locale === 'ja') {
    return NextResponse.rewrite(new URL('/ja', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

## メタデータ

```tsx
// 静的メタデータ
export const metadata: Metadata = {
  title: '商品一覧 | MyStore',
  description: '商品の一覧ページです',
}

// 動的メタデータ
export async function generateMetadata(
  { params }: { params: Promise<{ id: string }> }
): Promise<Metadata> {
  const { id } = await params
  const product = await getProduct(id)

  return {
    title: `${product.name} | MyStore`,
    description: product.description,
    openGraph: { images: [product.imageUrl] },
  }
}
```

## チェックリスト

- [ ] デフォルトでServer Component、必要な部分だけClient Component
- [ ] 'use client'の境界を最小限に（leaf componentに寄せる）
- [ ] 並列データフェッチ（Promise.all）を活用
- [ ] Suspenseでストリーミングを実装
- [ ] loading.tsx / error.tsx を各ルートに設置
- [ ] Server Actionsでフォーム送信を処理
- [ ] キャッシュ戦略を明示的に設定
- [ ] revalidateTag/revalidatePathで適切に再検証

## ベストプラクティス

1. **Client Componentの境界を下げる**: ページ全体ではなく、インタラクティブな部品だけをClient Componentにする
2. **compositionパターン**: Server ComponentをClient Componentのchildrenとして渡す
3. **Streaming First**: 遅いデータはSuspenseで包んで段階的に表示
4. **Server Actions > API Routes**: フォーム送信やデータ更新はServer Actionsを優先
5. **キャッシュを理解する**: デフォルトのキャッシュ動作を把握し、意図的に制御する
