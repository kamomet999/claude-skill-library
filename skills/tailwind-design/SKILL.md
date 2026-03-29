---
name: tailwind-design
description: Tailwind CSSデザインシステム構築。カスタムテーマ、コンポーネント設計、レスポンシブ、ダークモード対応の実践ガイド
category: フロントエンド
command: tailwind-design
version: 1.0.0
tags:
  - tailwind
  - css
  - design-system
  - responsive
  - dark-mode
---

# Tailwind CSS デザインシステム構築スキル

## 概要

Tailwind CSSを使ったスケーラブルなデザインシステムの構築ガイド。
カスタムテーマ設計、再利用可能なコンポーネントパターン、レスポンシブ設計、ダークモード実装をカバーする。

## Steps

### Step 1: テーマ設計（tailwind.config.ts）

```ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./src/**/*.{ts,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      // ブランドカラー（semantic naming）
      colors: {
        brand: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
          950: '#172554',
        },
        // セマンティックカラー
        surface: {
          DEFAULT: 'var(--color-surface)',
          secondary: 'var(--color-surface-secondary)',
          elevated: 'var(--color-surface-elevated)',
        },
        content: {
          DEFAULT: 'var(--color-content)',
          secondary: 'var(--color-content-secondary)',
          tertiary: 'var(--color-content-tertiary)',
        },
      },
      // タイポグラフィスケール
      fontSize: {
        'heading-1': ['2.25rem', { lineHeight: '2.5rem', fontWeight: '700' }],
        'heading-2': ['1.875rem', { lineHeight: '2.25rem', fontWeight: '600' }],
        'heading-3': ['1.5rem', { lineHeight: '2rem', fontWeight: '600' }],
        'body-lg': ['1.125rem', { lineHeight: '1.75rem' }],
        'body': ['1rem', { lineHeight: '1.5rem' }],
        'body-sm': ['0.875rem', { lineHeight: '1.25rem' }],
        'caption': ['0.75rem', { lineHeight: '1rem' }],
      },
      // スペーシングスケール（8pxグリッド）
      spacing: {
        '4.5': '1.125rem',
        '13': '3.25rem',
        '15': '3.75rem',
        '18': '4.5rem',
      },
      // ボーダーラジウス
      borderRadius: {
        'card': '0.75rem',
        'button': '0.5rem',
        'input': '0.375rem',
        'badge': '9999px',
      },
      // シャドウ
      boxShadow: {
        'card': '0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1)',
        'card-hover': '0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1)',
        'elevated': '0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1)',
      },
      // アニメーション
      animation: {
        'fade-in': 'fadeIn 0.2s ease-out',
        'slide-up': 'slideUp 0.3s ease-out',
        'scale-in': 'scaleIn 0.15s ease-out',
      },
      keyframes: {
        fadeIn: { from: { opacity: '0' }, to: { opacity: '1' } },
        slideUp: { from: { transform: 'translateY(8px)', opacity: '0' }, to: { transform: 'translateY(0)', opacity: '1' } },
        scaleIn: { from: { transform: 'scale(0.95)', opacity: '0' }, to: { transform: 'scale(1)', opacity: '1' } },
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}

export default config
```

### Step 2: CSS変数によるダークモード

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --color-surface: #ffffff;
    --color-surface-secondary: #f9fafb;
    --color-surface-elevated: #ffffff;
    --color-content: #111827;
    --color-content-secondary: #4b5563;
    --color-content-tertiary: #9ca3af;
  }

  .dark {
    --color-surface: #111827;
    --color-surface-secondary: #1f2937;
    --color-surface-elevated: #374151;
    --color-content: #f9fafb;
    --color-content-secondary: #d1d5db;
    --color-content-tertiary: #6b7280;
  }
}
```

### Step 3: ダークモード切り替え

```tsx
import { useEffect, useState } from 'react'

type Theme = 'light' | 'dark' | 'system'

function useTheme() {
  const [theme, setTheme] = useState<Theme>(() => {
    if (typeof window === 'undefined') return 'system'
    return (localStorage.getItem('theme') as Theme) ?? 'system'
  })

  useEffect(() => {
    const root = document.documentElement
    const systemDark = window.matchMedia('(prefers-color-scheme: dark)')

    function applyTheme(t: Theme) {
      const isDark = t === 'dark' || (t === 'system' && systemDark.matches)
      root.classList.toggle('dark', isDark)
    }

    applyTheme(theme)
    localStorage.setItem('theme', theme)

    if (theme === 'system') {
      const handler = () => applyTheme('system')
      systemDark.addEventListener('change', handler)
      return () => systemDark.removeEventListener('change', handler)
    }
  }, [theme])

  return { theme, setTheme }
}
```

### Step 4: コンポーネント設計パターン

**Variants with cva (class-variance-authority)**:

```tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  // ベーススタイル
  'inline-flex items-center justify-center font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-brand-600 text-white hover:bg-brand-700 active:bg-brand-800',
        secondary: 'bg-surface-secondary text-content border border-gray-200 hover:bg-gray-100 dark:border-gray-700 dark:hover:bg-gray-800',
        ghost: 'text-content hover:bg-surface-secondary',
        danger: 'bg-red-600 text-white hover:bg-red-700',
      },
      size: {
        sm: 'h-8 px-3 text-body-sm rounded-button',
        md: 'h-10 px-4 text-body rounded-button',
        lg: 'h-12 px-6 text-body-lg rounded-button',
        icon: 'h-10 w-10 rounded-button',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
)

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof buttonVariants>

function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    />
  )
}
```

**Card コンポーネント**:

```tsx
const cardVariants = cva('rounded-card border transition-shadow', {
  variants: {
    variant: {
      default: 'bg-surface border-gray-200 dark:border-gray-800 shadow-card',
      elevated: 'bg-surface-elevated border-gray-200 dark:border-gray-700 shadow-elevated',
      interactive: 'bg-surface border-gray-200 dark:border-gray-800 shadow-card hover:shadow-card-hover cursor-pointer',
    },
    padding: {
      none: '',
      sm: 'p-4',
      md: 'p-6',
      lg: 'p-8',
    },
  },
  defaultVariants: { variant: 'default', padding: 'md' },
})
```

### Step 5: レスポンシブ設計パターン

```tsx
// モバイルファーストのグリッドレイアウト
function ProductGrid({ products }: Props) {
  return (
    <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
      {products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  )
}

// レスポンシブナビゲーション
function Nav() {
  return (
    <nav className="flex flex-col gap-2 sm:flex-row sm:items-center sm:gap-6">
      <Logo className="h-8 w-auto" />
      {/* モバイルではハンバーガー、デスクトップではインライン */}
      <div className="hidden sm:flex sm:gap-4">
        <NavLinks />
      </div>
      <MobileMenu className="sm:hidden" />
    </nav>
  )
}

// コンテナの最大幅パターン
function PageLayout({ children }: Props) {
  return (
    <div className="mx-auto w-full max-w-7xl px-4 sm:px-6 lg:px-8">
      {children}
    </div>
  )
}
```

### Step 6: cnユーティリティ

```ts
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs))
}

// 使用例: 条件付きクラスの結合
<div className={cn(
  'rounded-card p-4',
  isActive && 'ring-2 ring-brand-500',
  isDisabled && 'opacity-50 pointer-events-none',
  className // 外部からのオーバーライド
)} />
```

## デザインシステムチェックリスト

- [ ] カラーパレットをセマンティックに定義（brand, surface, content）
- [ ] タイポグラフィスケールを統一
- [ ] スペーシングを8pxグリッドに統一
- [ ] cvaでバリアント管理
- [ ] cn()でクラス結合
- [ ] ダークモードをCSS変数で管理
- [ ] レスポンシブはモバイルファースト
- [ ] フォーカス状態を全インタラクティブ要素に設定

## ベストプラクティス

1. **@applyは最小限に**: ユーティリティクラスの直接使用を優先。@applyはTailwindの利点を損なう
2. **cvaでバリアント管理**: インラインの条件分岐よりcvaの方が保守しやすい
3. **tailwind-mergeでコンフリクト解消**: `cn()`を通すことで`p-4`と`p-6`の競合を防ぐ
4. **コンポーネントトークン**: 色やサイズはCSS変数経由でテーマ対応する
5. **Prettier Plugin**: `prettier-plugin-tailwindcss`でクラス順を自動ソート
