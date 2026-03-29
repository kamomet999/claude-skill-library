---
name: react-performance
description: Reactパフォーマンス最適化。memo、useMemo、useCallback、仮想化、コード分割、バンドルサイズ削減の実践ガイド
category: フロントエンド
command: react-performance
version: 1.0.0
tags:
  - react
  - performance
  - memo
  - virtualization
  - bundle
---

# React パフォーマンス最適化スキル

## 概要

Reactアプリケーションのパフォーマンスを体系的に分析・改善するためのスキル。
不要な再レンダリングの排除、メモ化戦略、仮想化、コード分割、バンドルサイズ最適化をカバーする。

## Steps

### Step 1: パフォーマンスボトルネックの特定

1. React DevTools Profilerで再レンダリング頻度を確認
2. `why-did-you-render` ライブラリで不要な再レンダリングを検出
3. Lighthouseスコアを計測（LCP、FID、CLS）
4. バンドルアナライザーでサイズを可視化

### Step 2: メモ化戦略の適用

**React.memo** - コンポーネントレベルのメモ化:

```tsx
// BAD: 親の再レンダリングで毎回再レンダリング
function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>
}

// GOOD: propsが変わらない限り再レンダリングしない
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>
})

// カスタム比較関数（深い比較が必要な場合）
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>
}, (prevProps, nextProps) => prevProps.user.id === nextProps.user.id)
```

**useMemo** - 計算結果のメモ化:

```tsx
// BAD: 毎レンダリングでフィルタリング実行
function UserList({ users, query }: Props) {
  const filtered = users.filter(u => u.name.includes(query))
  return <List items={filtered} />
}

// GOOD: users or queryが変わった時だけ再計算
function UserList({ users, query }: Props) {
  const filtered = useMemo(
    () => users.filter(u => u.name.includes(query)),
    [users, query]
  )
  return <List items={filtered} />
}
```

**useCallback** - 関数参照の安定化:

```tsx
// BAD: 毎レンダリングで新しい関数参照を生成
function Parent() {
  const [count, setCount] = useState(0)
  const handleClick = () => setCount(c => c + 1)
  return <MemoizedChild onClick={handleClick} />
}

// GOOD: 関数参照が安定するのでchildの再レンダリングを防止
function Parent() {
  const [count, setCount] = useState(0)
  const handleClick = useCallback(() => setCount(c => c + 1), [])
  return <MemoizedChild onClick={handleClick} />
}
```

### Step 3: メモ化が不要なケース（重要）

```tsx
// メモ化が不要: プリミティブ値のみのprops
// React.memoのオーバーヘッドの方が大きい
function SimpleLabel({ text }: { text: string }) {
  return <span>{text}</span>
}

// メモ化が不要: 毎回異なるpropsが渡される場合
// 比較コストが無駄になる
const AlwaysChanging = memo(function AlwaysChanging({ timestamp }: Props) {
  return <span>{timestamp}</span>
})

// useMemoが不要: 単純な計算
// メモ化のオーバーヘッド > 再計算コスト
const doubled = useMemo(() => count * 2, [count]) // 不要
const doubled = count * 2 // これで十分
```

### Step 4: 仮想化（大量リスト）

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <ItemRow item={items[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Step 5: コード分割

```tsx
// ルートレベルの分割（React.lazy）
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  )
}

// コンポーネントレベルの分割（条件付きロード）
const HeavyChart = lazy(() => import('./components/HeavyChart'))

function AnalyticsPanel({ showChart }: Props) {
  return (
    <div>
      <Summary />
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  )
}

// named exportの分割
const UserProfile = lazy(() =>
  import('./components/UserModule').then(mod => ({
    default: mod.UserProfile
  }))
)
```

### Step 6: バンドルサイズ最適化

```bash
# バンドル分析
npx @next/bundle-analyzer
# または
npx vite-bundle-visualizer

# 重いライブラリの代替
# moment.js (300KB) -> date-fns (tree-shakeable) or dayjs (2KB)
# lodash (70KB) -> lodash-es (tree-shakeable) or 個別import
# axios (13KB) -> fetch API (built-in)
```

```tsx
// BAD: lodash全体をインポート
import _ from 'lodash'
const sorted = _.sortBy(items, 'name')

// GOOD: 個別関数のみインポート
import sortBy from 'lodash-es/sortBy'
const sorted = sortBy(items, 'name')

// BEST: ネイティブで書ける場合は依存なし
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name))
```

### Step 7: 画像最適化

```tsx
// Next.js Image最適化
import Image from 'next/image'

function ProductCard({ product }: Props) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={300}
      height={200}
      placeholder="blur"
      blurDataURL={product.blurHash}
      sizes="(max-width: 768px) 100vw, 300px"
      loading="lazy"
    />
  )
}
```

## パフォーマンスチェックリスト

- [ ] React DevTools Profilerで不要な再レンダリングがないか確認
- [ ] リスト表示（50件以上）に仮想化を適用
- [ ] ルートレベルのコード分割を実装
- [ ] 重いライブラリのtree-shaking or 代替を検討
- [ ] 画像にlazy loading、適切なサイズ、next/imageを使用
- [ ] useMemo/useCallbackは計測してから適用（早すぎる最適化を避ける）
- [ ] Web Vitals（LCP < 2.5s, FID < 100ms, CLS < 0.1）を満たす

## ベストプラクティス

1. **計測してから最適化**: React Profiler、Lighthouse、bundle analyzerで数値を出す
2. **メモ化は銀の弾丸ではない**: 比較コスト > 再計算コストなら逆効果
3. **状態の配置を最適化**: 状態を使うコンポーネントに近づける（state colocation）
4. **children patternで再レンダリング回避**: 親の状態変更が子に波及しない構造にする
5. **key属性を適切に使う**: リストのkeyにindexを使わない（データが変わる場合）

## 参考リンク

- [React公式 - パフォーマンス](https://react.dev/learn/render-and-commit)
- [TanStack Virtual](https://tanstack.com/virtual/latest)
- [web.dev - Web Vitals](https://web.dev/vitals/)
