---
name: i18n-localization
description: 国際化・多言語対応設計。next-intl、ICUメッセージ、RTL対応、日付・通貨フォーマット、翻訳ワークフローの実践ガイド
category: フロントエンド
command: i18n-localization
version: 1.0.0
tags:
  - i18n
  - l10n
  - next-intl
  - translation
  - rtl
---

# 国際化・多言語対応設計スキル

## 概要

Webアプリケーションの国際化（i18n）とローカライゼーション（l10n）の設計・実装ガイド。
next-intlを軸に、ICUメッセージフォーマット、RTL対応、日付・通貨フォーマット、翻訳ワークフローをカバーする。

## Steps

### Step 1: next-intlのセットアップ（App Router）

```bash
npm install next-intl
```

```
プロジェクト構成:
src/
├── app/
│   └── [locale]/
│       ├── layout.tsx
│       ├── page.tsx
│       └── products/
│           └── page.tsx
├── i18n/
│   ├── request.ts          # サーバーサイドi18n設定
│   ├── routing.ts          # ルーティング設定
│   └── navigation.ts       # ナビゲーションヘルパー
├── messages/
│   ├── ja.json
│   ├── en.json
│   └── zh.json
└── middleware.ts
```

```ts
// src/i18n/routing.ts
import { defineRouting } from 'next-intl/routing'

export const routing = defineRouting({
  locales: ['ja', 'en', 'zh'],
  defaultLocale: 'ja',
  localePrefix: 'as-needed', // デフォルトロケールはURLプレフィックスなし
})

// src/i18n/navigation.ts
import { createNavigation } from 'next-intl/navigation'
import { routing } from './routing'

export const { Link, redirect, usePathname, useRouter } =
  createNavigation(routing)

// src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server'
import { routing } from './routing'

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale
  if (!locale || !routing.locales.includes(locale as any)) {
    locale = routing.defaultLocale
  }

  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  }
})
```

```ts
// middleware.ts
import createMiddleware from 'next-intl/middleware'
import { routing } from './i18n/routing'

export default createMiddleware(routing)

export const config = {
  matcher: ['/', '/(ja|en|zh)/:path*'],
}
```

```tsx
// src/app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl'
import { getMessages, setRequestLocale } from 'next-intl/server'
import { routing } from '@/i18n/routing'

export function generateStaticParams() {
  return routing.locales.map(locale => ({ locale }))
}

export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode
  params: Promise<{ locale: string }>
}) {
  const { locale } = await params
  setRequestLocale(locale)
  const messages = await getMessages()

  return (
    <html lang={locale} dir={locale === 'ar' ? 'rtl' : 'ltr'}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  )
}
```

### Step 2: メッセージファイル設計

```json
// messages/ja.json
{
  "common": {
    "loading": "読み込み中...",
    "error": "エラーが発生しました",
    "save": "保存",
    "cancel": "キャンセル",
    "delete": "削除",
    "confirm": "確認",
    "back": "戻る",
    "next": "次へ"
  },
  "auth": {
    "login": "ログイン",
    "logout": "ログアウト",
    "email": "メールアドレス",
    "password": "パスワード",
    "forgotPassword": "パスワードをお忘れですか？",
    "loginError": "メールアドレスまたはパスワードが正しくありません"
  },
  "products": {
    "title": "商品一覧",
    "searchPlaceholder": "商品を検索...",
    "noResults": "「{query}」に一致する商品が見つかりません",
    "itemCount": "{count, plural, =0 {商品なし} one {1件の商品} other {#件の商品}}",
    "price": "{price, number, ::currency/JPY}",
    "addedAt": "{date, date, medium}に追加",
    "status": {
      "inStock": "在庫あり",
      "outOfStock": "在庫切れ",
      "preOrder": "予約受付中"
    }
  },
  "validation": {
    "required": "{field}を入力してください",
    "email": "正しいメールアドレスを入力してください",
    "minLength": "{field}は{min}文字以上で入力してください",
    "maxLength": "{field}は{max}文字以内で入力してください"
  }
}
```

```json
// messages/en.json
{
  "common": {
    "loading": "Loading...",
    "error": "An error occurred",
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "confirm": "Confirm",
    "back": "Back",
    "next": "Next"
  },
  "products": {
    "title": "Products",
    "searchPlaceholder": "Search products...",
    "noResults": "No products found for \"{query}\"",
    "itemCount": "{count, plural, =0 {No products} one {1 product} other {# products}}",
    "price": "{price, number, ::currency/USD}",
    "addedAt": "Added on {date, date, medium}",
    "status": {
      "inStock": "In Stock",
      "outOfStock": "Out of Stock",
      "preOrder": "Pre-order"
    }
  },
  "validation": {
    "required": "{field} is required",
    "email": "Please enter a valid email address",
    "minLength": "{field} must be at least {min} characters",
    "maxLength": "{field} must be at most {max} characters"
  }
}
```

### Step 3: ICUメッセージフォーマット

```tsx
import { useTranslations } from 'next-intl'

function ProductList({ products, query }: Props) {
  const t = useTranslations('products')

  return (
    <div>
      <h1>{t('title')}</h1>

      {/* 単純な文字列 */}
      <input placeholder={t('searchPlaceholder')} />

      {/* 変数埋め込み */}
      {products.length === 0 && <p>{t('noResults', { query })}</p>}

      {/* 複数形（ICU plural） */}
      <p>{t('itemCount', { count: products.length })}</p>

      {products.map(p => (
        <div key={p.id}>
          <span>{p.name}</span>
          {/* 通貨フォーマット */}
          <span>{t('price', { price: p.price })}</span>
          {/* 日付フォーマット */}
          <span>{t('addedAt', { date: new Date(p.createdAt) })}</span>
          {/* ネストされたキー */}
          <span>{t(`status.${p.status}`)}</span>
        </div>
      ))}
    </div>
  )
}
```

**ICUメッセージ構文リファレンス**:

```
// 複数形
{count, plural,
  =0 {商品なし}
  one {1件の商品}
  other {#件の商品}
}

// 性別（select）
{gender, select,
  male {彼}
  female {彼女}
  other {その人}
}がログインしました

// 数値フォーマット
{amount, number, ::currency/JPY}         // ¥1,200
{percent, number, ::percent}             // 85%
{count, number, ::compact-short}         // 1.2K

// 日付フォーマット
{date, date, short}                      // 2024/01/15
{date, date, medium}                     // 2024年1月15日
{date, date, long}                       // 2024年1月15日月曜日
{date, time, short}                      // 14:30
```

### Step 4: Server Componentsでの使用

```tsx
// Server Component（async可能）
import { getTranslations, setRequestLocale } from 'next-intl/server'

async function ProductsPage({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params
  setRequestLocale(locale)
  const t = await getTranslations('products')

  const products = await getProducts()

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('itemCount', { count: products.length })}</p>
      {/* Client Componentに翻訳済みテキストを渡す */}
      <ProductFilter placeholder={t('searchPlaceholder')} />
      <ProductGrid products={products} />
    </div>
  )
}
```

### Step 5: 日付・通貨・数値フォーマット

```tsx
import { useFormatter, useLocale } from 'next-intl'

function FormattedPrice({ amount, currency }: Props) {
  const format = useFormatter()

  // 通貨
  const price = format.number(amount, {
    style: 'currency',
    currency: currency, // 'JPY', 'USD', etc.
  })
  // ja: ¥1,200  en: $12.00

  // 相対日時
  const relativeTime = format.relativeTime(new Date('2024-01-01'))
  // ja: 3ヶ月前  en: 3 months ago

  // 日付範囲
  const dateRange = format.dateTimeRange(
    new Date('2024-01-01'),
    new Date('2024-01-31'),
    { year: 'numeric', month: 'long', day: 'numeric' }
  )
  // ja: 2024年1月1日～31日  en: January 1 – 31, 2024

  // リスト
  const list = format.list(['東京', '大阪', '名古屋'], { type: 'conjunction' })
  // ja: 東京、大阪、名古屋  en: Tokyo, Osaka, and Nagoya

  return <span>{price}</span>
}

// Server Componentでの使用
import { getFormatter } from 'next-intl/server'

async function ServerPrice({ amount }: Props) {
  const format = await getFormatter()
  return <span>{format.number(amount, { style: 'currency', currency: 'JPY' })}</span>
}
```

### Step 6: RTL（右から左）対応

```tsx
// レイアウトのRTL対応
// Tailwind CSSのRTLサポート
<div className="flex gap-4 rtl:flex-row-reverse">
  <Icon className="mr-2 rtl:ml-2 rtl:mr-0" />
  <span>{text}</span>
</div>

// logical properties（推奨）
// margin-left -> margin-inline-start
// padding-right -> padding-inline-end
<div className="ms-4 ps-2 text-start">
  {/* ms = margin-start, ps = padding-start */}
  {/* text-start: LTRではleft、RTLではright */}
  {content}
</div>

// Tailwind CSSの論理プロパティ
// me-4 = margin-inline-end
// ps-2 = padding-inline-start
// rounded-s-lg = border-start-start-radius + border-end-start-radius
// border-e = border-inline-end

// RTL対応のアイコン（矢印など反転が必要なもの）
function DirectionalIcon({ direction }: { direction: 'forward' | 'back' }) {
  const locale = useLocale()
  const isRTL = locale === 'ar' || locale === 'he'

  return (
    <ChevronIcon
      className={cn(
        direction === 'forward' && isRTL && 'rotate-180',
        direction === 'back' && !isRTL && 'rotate-180'
      )}
    />
  )
}
```

### Step 7: 言語切り替えUI

```tsx
'use client'

import { useLocale } from 'next-intl'
import { useRouter, usePathname } from '@/i18n/navigation'
import { routing } from '@/i18n/routing'

const localeLabels: Record<string, string> = {
  ja: '日本語',
  en: 'English',
  zh: '中文',
}

function LocaleSwitcher() {
  const locale = useLocale()
  const router = useRouter()
  const pathname = usePathname()

  function handleChange(newLocale: string) {
    router.replace(pathname, { locale: newLocale })
  }

  return (
    <div>
      <label htmlFor="locale-select" className="sr-only">
        言語を選択
      </label>
      <select
        id="locale-select"
        value={locale}
        onChange={e => handleChange(e.target.value)}
        className="rounded-md border px-3 py-1.5 text-sm"
      >
        {routing.locales.map(loc => (
          <option key={loc} value={loc}>
            {localeLabels[loc]}
          </option>
        ))}
      </select>
    </div>
  )
}
```

### Step 8: 翻訳ワークフロー

```
開発フロー:
1. 開発者がja.json（基底言語）にキーと日本語テキストを追加
2. CIで未翻訳キーを検出（他の言語ファイルとの差分チェック）
3. 翻訳者（または翻訳サービス）が他言語ファイルを更新
4. PRレビューで翻訳内容を確認
5. マージ後にデプロイ
```

```ts
// scripts/check-translations.ts
// CIで実行: 翻訳漏れを検出
import ja from '../messages/ja.json'
import en from '../messages/en.json'
import zh from '../messages/zh.json'

function flattenKeys(obj: Record<string, unknown>, prefix = ''): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key
    if (typeof value === 'object' && value !== null) {
      return flattenKeys(value as Record<string, unknown>, fullKey)
    }
    return [fullKey]
  })
}

const baseKeys = new Set(flattenKeys(ja))
const locales = { en, zh } as Record<string, Record<string, unknown>>

let hasError = false

for (const [locale, messages] of Object.entries(locales)) {
  const keys = new Set(flattenKeys(messages))

  // 基底言語にあって翻訳にないキー
  for (const key of baseKeys) {
    if (!keys.has(key)) {
      console.error(`[${locale}] Missing key: ${key}`)
      hasError = true
    }
  }

  // 翻訳にあって基底言語にないキー（不要キー）
  for (const key of keys) {
    if (!baseKeys.has(key)) {
      console.warn(`[${locale}] Unused key: ${key}`)
    }
  }
}

if (hasError) process.exit(1)
```

## メッセージキーの命名規則

```
推奨構造:
{namespace}.{context}.{element}

例:
products.list.title        -> "商品一覧"
products.list.empty        -> "商品がありません"
products.detail.addToCart   -> "カートに追加"
auth.login.title           -> "ログイン"
auth.login.submitButton    -> "ログインする"
common.actions.save        -> "保存"
common.actions.cancel      -> "キャンセル"
validation.required         -> "{field}を入力してください"
error.notFound             -> "ページが見つかりません"
```

## チェックリスト

- [ ] 全テキストがメッセージファイルから取得される（ハードコードなし）
- [ ] ICU複数形を数値表示に使用
- [ ] 日付・通貨はIntl APIまたはuseFormatterでフォーマット
- [ ] 文字列結合でメッセージを組み立てない（ICUメッセージを使う）
- [ ] RTL対応（logical properties使用）
- [ ] html lang属性がロケールに応じて設定
- [ ] 翻訳漏れチェックがCIに組み込まれている
- [ ] 言語切り替えUIが提供されている
- [ ] SEO対応（hreflang、canonical URL）

## ベストプラクティス

1. **文字列結合を避ける**: `t('hello') + name`ではなく`t('hello', { name })`を使う（語順が言語で異なる）
2. **コンテキストを含める**: 同じ「開く」でも"open_file"と"open_door"は翻訳が異なる場合がある
3. **デザインに余白を**: 英語 -> ドイツ語で30%、日本語 -> 英語で50%テキスト長が変わることがある
4. **基底言語を決める**: 開発チームの言語を基底にし、他言語は翻訳として管理
5. **型安全に**: next-intlの型生成機能を活用してキーのtypoを防ぐ
