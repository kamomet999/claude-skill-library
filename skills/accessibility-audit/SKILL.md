---
name: accessibility-audit
description: アクセシビリティ監査・改善。WCAG 2.1、ARIA、キーボードナビゲーション、スクリーンリーダー、カラーコントラストの実践ガイド
category: フロントエンド
command: accessibility-audit
version: 1.0.0
tags:
  - a11y
  - wcag
  - aria
  - keyboard
  - screen-reader
---

# アクセシビリティ監査・改善スキル

## 概要

Webアプリケーションのアクセシビリティを体系的に監査・改善するためのスキル。
WCAG 2.1準拠、ARIAの正しい使い方、キーボードナビゲーション、スクリーンリーダー対応、カラーコントラストをカバーする。

## Steps

### Step 1: 自動チェックツールの導入

```bash
# ESLintプラグイン（開発時）
npm install -D eslint-plugin-jsx-a11y

# テスト（jest/vitest）
npm install -D jest-axe @testing-library/jest-dom

# Storybook addon
npm install -D @storybook/addon-a11y
```

```ts
// eslint.config.js
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  {
    plugins: { 'jsx-a11y': jsxA11y },
    rules: {
      'jsx-a11y/alt-text': 'error',
      'jsx-a11y/anchor-has-content': 'error',
      'jsx-a11y/aria-props': 'error',
      'jsx-a11y/aria-role': 'error',
      'jsx-a11y/click-events-have-key-events': 'error',
      'jsx-a11y/heading-has-content': 'error',
      'jsx-a11y/label-has-associated-control': 'error',
      'jsx-a11y/no-autofocus': 'warn',
      'jsx-a11y/no-noninteractive-element-interactions': 'error',
    },
  },
]
```

```tsx
// テストでのaxeチェック
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'

expect.extend(toHaveNoViolations)

test('LoginForm has no accessibility violations', async () => {
  const { container } = render(<LoginForm />)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})
```

### Step 2: セマンティックHTML

```tsx
// BAD: divだけで構築
function BadPage() {
  return (
    <div className="header">
      <div className="logo">MyApp</div>
      <div className="nav">
        <div onClick={handleClick}>ホーム</div>
      </div>
    </div>
  )
}

// GOOD: セマンティックな要素を使用
function GoodPage() {
  return (
    <header>
      <h1>MyApp</h1>
      <nav aria-label="メインナビゲーション">
        <ul role="list">
          <li><a href="/">ホーム</a></li>
          <li><a href="/products">商品</a></li>
        </ul>
      </nav>
    </header>
  )
}

// ランドマーク構造
function PageLayout({ children }: Props) {
  return (
    <>
      <a href="#main-content" className="sr-only focus:not-sr-only focus:absolute focus:z-50 focus:bg-white focus:p-4">
        メインコンテンツへスキップ
      </a>
      <header role="banner">
        <Nav />
      </header>
      <main id="main-content" role="main">
        {children}
      </main>
      <aside role="complementary" aria-label="サイドバー">
        <Sidebar />
      </aside>
      <footer role="contentinfo">
        <Footer />
      </footer>
    </>
  )
}
```

### Step 3: ARIA属性の正しい使い方

```tsx
// 第一原則: ネイティブHTML要素が使えるならARIAは不要
// BAD
<div role="button" tabIndex={0} onClick={handleClick}>送信</div>
// GOOD
<button onClick={handleClick}>送信</button>

// ライブリージョン（動的な変更をスクリーンリーダーに通知）
function Toast({ message }: Props) {
  return (
    <div role="alert" aria-live="assertive" aria-atomic="true">
      {message}
    </div>
  )
}

// ステータスメッセージ（控えめな通知）
function SearchResults({ count }: Props) {
  return (
    <div role="status" aria-live="polite">
      {count}件の結果が見つかりました
    </div>
  )
}

// 展開/折りたたみ
function AccordionItem({ title, children }: Props) {
  const [isOpen, setIsOpen] = useState(false)
  const contentId = useId()

  return (
    <div>
      <h3>
        <button
          aria-expanded={isOpen}
          aria-controls={contentId}
          onClick={() => setIsOpen(prev => !prev)}
          className="flex w-full items-center justify-between"
        >
          {title}
          <ChevronIcon className={cn('transition-transform', isOpen && 'rotate-180')} />
        </button>
      </h3>
      <div
        id={contentId}
        role="region"
        aria-labelledby={undefined}
        hidden={!isOpen}
      >
        {children}
      </div>
    </div>
  )
}

// タブ
function Tabs({ tabs }: Props) {
  const [activeIndex, setActiveIndex] = useState(0)

  return (
    <div>
      <div role="tablist" aria-label="設定タブ">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={i === activeIndex}
            aria-controls={`panel-${tab.id}`}
            tabIndex={i === activeIndex ? 0 : -1}
            onClick={() => setActiveIndex(i)}
            onKeyDown={(e) => {
              if (e.key === 'ArrowRight') setActiveIndex((i + 1) % tabs.length)
              if (e.key === 'ArrowLeft') setActiveIndex((i - 1 + tabs.length) % tabs.length)
            }}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={i !== activeIndex}
          tabIndex={0}
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

### Step 4: キーボードナビゲーション

```tsx
// フォーカストラップ（モーダル内にフォーカスを閉じ込める）
function useFocusTrap(isActive: boolean) {
  const containerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!isActive) return
    const container = containerRef.current
    if (!container) return

    const focusableSelector =
      'a[href], button:not([disabled]), input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'

    const focusableElements = container.querySelectorAll<HTMLElement>(focusableSelector)
    const firstElement = focusableElements[0]
    const lastElement = focusableElements[focusableElements.length - 1]

    firstElement?.focus()

    function handleKeyDown(e: KeyboardEvent) {
      if (e.key !== 'Tab') return

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault()
          lastElement?.focus()
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault()
          firstElement?.focus()
        }
      }
    }

    container.addEventListener('keydown', handleKeyDown)
    return () => container.removeEventListener('keydown', handleKeyDown)
  }, [isActive])

  return containerRef
}

// Escapeキーでモーダルを閉じる
function Modal({ isOpen, onClose, children }: Props) {
  const containerRef = useFocusTrap(isOpen)

  useEffect(() => {
    if (!isOpen) return
    function handleEscape(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose()
    }
    document.addEventListener('keydown', handleEscape)
    return () => document.removeEventListener('keydown', handleEscape)
  }, [isOpen, onClose])

  if (!isOpen) return null

  return (
    <div
      ref={containerRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <h2 id="modal-title">タイトル</h2>
      {children}
      <button onClick={onClose}>閉じる</button>
    </div>
  )
}

// roving tabindex（コンポジットウィジェット）
function ToolbarButtons({ items }: Props) {
  const [focusedIndex, setFocusedIndex] = useState(0)

  function handleKeyDown(e: React.KeyboardEvent, index: number) {
    switch (e.key) {
      case 'ArrowRight':
        e.preventDefault()
        setFocusedIndex((index + 1) % items.length)
        break
      case 'ArrowLeft':
        e.preventDefault()
        setFocusedIndex((index - 1 + items.length) % items.length)
        break
      case 'Home':
        e.preventDefault()
        setFocusedIndex(0)
        break
      case 'End':
        e.preventDefault()
        setFocusedIndex(items.length - 1)
        break
    }
  }

  return (
    <div role="toolbar" aria-label="テキスト編集">
      {items.map((item, i) => (
        <button
          key={item.id}
          tabIndex={i === focusedIndex ? 0 : -1}
          onKeyDown={(e) => handleKeyDown(e, i)}
          ref={el => { if (i === focusedIndex) el?.focus() }}
        >
          {item.label}
        </button>
      ))}
    </div>
  )
}
```

### Step 5: カラーコントラスト

```
WCAG 2.1 コントラスト比の基準:
  AA（通常テキスト）:    4.5:1 以上
  AA（大きなテキスト）:  3:1 以上（18px bold or 24px以上）
  AAA（通常テキスト）:   7:1 以上
  非テキスト要素:        3:1 以上（アイコン、ボーダー等）
```

```tsx
// 色だけに依存しない情報伝達
// BAD: 色だけでエラーを伝える
<input className={hasError ? 'border-red-500' : 'border-gray-300'} />

// GOOD: 色 + アイコン + テキスト
<div>
  <input
    className={cn(
      'border',
      hasError ? 'border-red-500' : 'border-gray-300'
    )}
    aria-invalid={hasError}
    aria-describedby={hasError ? 'email-error' : undefined}
  />
  {hasError && (
    <p id="email-error" className="mt-1 flex items-center gap-1 text-sm text-red-600">
      <AlertIcon className="h-4 w-4" aria-hidden="true" />
      メールアドレスが無効です
    </p>
  )}
</div>

// フォーカスインジケーター（visible focus）
// Tailwindのデフォルトfocus-visibleを活用
<button className="rounded-md bg-brand-600 px-4 py-2 text-white focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2">
  送信
</button>
```

### Step 6: スクリーンリーダー対応

```tsx
// visually hidden（sr-only）
// 視覚的には隠すが、スクリーンリーダーには読ませる
<button aria-label="お気に入りに追加">
  <HeartIcon aria-hidden="true" />
  <span className="sr-only">お気に入りに追加</span>
</button>

// アイコンボタンにはaria-labelが必須
<button aria-label="閉じる" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>

// テーブルのアクセシビリティ
<table>
  <caption className="sr-only">2024年度売上データ</caption>
  <thead>
    <tr>
      <th scope="col">月</th>
      <th scope="col">売上</th>
      <th scope="col">前年比</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">1月</th>
      <td>1,200,000円</td>
      <td>+15%</td>
    </tr>
  </tbody>
</table>

// 画像のalt属性
// 装飾画像: alt=""（空文字）
<img src="/decorative-line.svg" alt="" aria-hidden="true" />

// 情報を伝える画像: 具体的な説明
<img src="/chart.png" alt="2024年の月別売上推移。1月から12月まで右肩上がり" />

// リンク付き画像
<a href="/products/123">
  <img src="/product.jpg" alt="商品名 - 詳細を見る" />
</a>
```

### Step 7: フォームのアクセシビリティ

```tsx
// 完全なアクセシブルフォームフィールド
function AccessibleField({
  label,
  error,
  hint,
  required,
  children,
}: FormFieldProps) {
  const id = useId()
  const errorId = `${id}-error`
  const hintId = `${id}-hint`

  return (
    <div>
      <label htmlFor={id}>
        {label}
        {required && <span aria-hidden="true" className="text-red-500"> *</span>}
        {required && <span className="sr-only">（必須）</span>}
      </label>

      {hint && (
        <p id={hintId} className="text-sm text-gray-500">
          {hint}
        </p>
      )}

      {cloneElement(children, {
        id,
        'aria-required': required,
        'aria-invalid': !!error,
        'aria-describedby': [
          hint ? hintId : null,
          error ? errorId : null,
        ].filter(Boolean).join(' ') || undefined,
      })}

      {error && (
        <p id={errorId} role="alert" className="text-sm text-red-600">
          {error}
        </p>
      )}
    </div>
  )
}

// ラジオグループ
function RadioGroup({ legend, options, value, onChange }: Props) {
  return (
    <fieldset>
      <legend className="text-sm font-medium">{legend}</legend>
      <div role="radiogroup" className="mt-2 space-y-2">
        {options.map(option => (
          <label key={option.value} className="flex items-center gap-2">
            <input
              type="radio"
              name={legend}
              value={option.value}
              checked={value === option.value}
              onChange={() => onChange(option.value)}
            />
            {option.label}
          </label>
        ))}
      </div>
    </fieldset>
  )
}
```

## 監査チェックリスト

### WCAG 2.1 Level AA

- [ ] **知覚可能**
  - [ ] 全画像にalt属性（装飾画像はalt=""）
  - [ ] カラーコントラスト比 4.5:1以上
  - [ ] 色だけに依存しない情報伝達
  - [ ] テキストの200%拡大で情報損失なし

- [ ] **操作可能**
  - [ ] 全機能がキーボードで操作可能
  - [ ] フォーカスインジケーターが視認可能
  - [ ] フォーカストラップがモーダルに実装済み
  - [ ] スキップリンク設置
  - [ ] prefers-reduced-motion対応

- [ ] **理解可能**
  - [ ] lang属性がhtml要素に設定
  - [ ] フォームエラーが具体的で関連フィールドに紐付け
  - [ ] ラベルが全フォーム要素に紐付け
  - [ ] エラー訂正の方法が提示される

- [ ] **堅牢**
  - [ ] 正しいセマンティックHTML
  - [ ] ARIA属性が適切に使用
  - [ ] name, role, valueがプログラム的に決定可能

## ベストプラクティス

1. **ネイティブHTMLファースト**: ARIAは最後の手段。button、input、selectを使う
2. **自動テスト + 手動テスト**: axeで80%検出、残り20%は手動（キーボード操作、スクリーンリーダー）
3. **当事者テスト**: 実際のスクリーンリーダーユーザーによるテストが最も価値が高い
4. **段階的改善**: 一度に完璧を目指さず、CRITICAL -> HIGH -> MEDIUMの順で改善
5. **デザイン段階から**: 実装後の修正は10倍コストがかかる。デザイン時にa11yを考慮
