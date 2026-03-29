---
name: state-management
description: 状態管理パターン選択ガイド。Zustand、Jotai、Redux Toolkit、React Query、コンテキスト使い分けの実践ガイド
category: フロントエンド
command: state-management
version: 1.0.0
tags:
  - state
  - zustand
  - jotai
  - redux
  - react-query
---

# 状態管理パターン選択ガイド

## 概要

Reactアプリケーションにおける状態管理ライブラリの選定基準と実装パターン。
Zustand、Jotai、Redux Toolkit、React Query（TanStack Query）、React Contextの使い分けを明確にする。

## Steps

### Step 1: 状態の分類と管理方法の選定

```
状態の種類          -> 推奨ツール
────────────────────────────────────────
サーバーキャッシュ  -> TanStack Query（React Query）
グローバルUI状態    -> Zustand / Jotai
ローカルUI状態      -> useState / useReducer
フォーム状態        -> React Hook Form
URL状態             -> nuqs / useSearchParams
テーマ・認証        -> React Context
```

**判断フローチャート**:

```
状態はサーバーから取得するデータか？
├─ YES -> TanStack Query
└─ NO -> その状態を使うコンポーネントはいくつか？
    ├─ 1つだけ -> useState / useReducer
    ├─ 親子2-3個 -> props passing（状態のリフトアップ）
    └─ 離れた複数箇所 -> 変更頻度は？
        ├─ 高頻度（入力、アニメ等） -> Zustand / Jotai
        └─ 低頻度（テーマ、認証等） -> React Context
```

### Step 2: Zustand（推奨デフォルト）

軽量・シンプル・TypeScript親和性が高い。中小規模アプリに最適。

```ts
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

// 型定義
interface CartItem {
  id: string
  name: string
  price: number
  quantity: number
}

interface CartState {
  items: CartItem[]
  isOpen: boolean
}

interface CartActions {
  addItem: (item: Omit<CartItem, 'quantity'>) => void
  removeItem: (id: string) => void
  updateQuantity: (id: string, quantity: number) => void
  toggleCart: () => void
  clearCart: () => void
  totalPrice: () => number
}

// ストア作成（immutable patternで書く場合）
const useCartStore = create<CartState & CartActions>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        isOpen: false,

        addItem: (item) =>
          set((state) => {
            const existing = state.items.find(i => i.id === item.id)
            if (existing) {
              return {
                items: state.items.map(i =>
                  i.id === item.id
                    ? { ...i, quantity: i.quantity + 1 }
                    : i
                ),
              }
            }
            return { items: [...state.items, { ...item, quantity: 1 }] }
          }),

        removeItem: (id) =>
          set((state) => ({
            items: state.items.filter(i => i.id !== id),
          })),

        updateQuantity: (id, quantity) =>
          set((state) => ({
            items: quantity <= 0
              ? state.items.filter(i => i.id !== id)
              : state.items.map(i =>
                  i.id === id ? { ...i, quantity } : i
                ),
          })),

        toggleCart: () =>
          set((state) => ({ isOpen: !state.isOpen })),

        clearCart: () => set({ items: [] }),

        totalPrice: () =>
          get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      }),
      { name: 'cart-storage' }
    ),
    { name: 'CartStore' }
  )
)

// コンポーネントでの使用（セレクターで最小限の再レンダリング）
function CartBadge() {
  const count = useCartStore(state => state.items.length)
  return <span>{count}</span>
}

function CartTotal() {
  const totalPrice = useCartStore(state => state.totalPrice)
  return <span>{totalPrice()}円</span>
}
```

**Zustandのスライスパターン（大規模向け）**:

```ts
interface AuthSlice {
  user: User | null
  login: (credentials: Credentials) => Promise<void>
  logout: () => void
}

interface UISlice {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

const createAuthSlice: StateCreator<AuthSlice & UISlice, [], [], AuthSlice> = (set) => ({
  user: null,
  login: async (credentials) => {
    const user = await authApi.login(credentials)
    set({ user })
  },
  logout: () => set({ user: null }),
})

const createUISlice: StateCreator<AuthSlice & UISlice, [], [], UISlice> = (set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
})

const useAppStore = create<AuthSlice & UISlice>()((...a) => ({
  ...createAuthSlice(...a),
  ...createUISlice(...a),
}))
```

### Step 3: Jotai（アトミック状態管理）

コンポーネント間で細粒度の状態共有が必要な場合に最適。

```ts
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai'
import { atomWithStorage } from 'jotai/utils'

// プリミティブatom
const countAtom = atom(0)
const darkModeAtom = atomWithStorage('darkMode', false)

// 派生atom（computed）
const doubledAtom = atom((get) => get(countAtom) * 2)

// 書き込み可能な派生atom
const decrementAtom = atom(
  null,
  (get, set) => set(countAtom, get(countAtom) - 1)
)

// 非同期atom
const userAtom = atom(async () => {
  const response = await fetch('/api/user')
  return response.json() as Promise<User>
})

// フィルター状態の例
const searchQueryAtom = atom('')
const categoryFilterAtom = atom<string | null>(null)
const sortOrderAtom = atom<'asc' | 'desc'>('asc')

const filteredProductsAtom = atom(async (get) => {
  const query = get(searchQueryAtom)
  const category = get(categoryFilterAtom)
  const sort = get(sortOrderAtom)

  const params = new URLSearchParams()
  if (query) params.set('q', query)
  if (category) params.set('category', category)
  params.set('sort', sort)

  const res = await fetch(`/api/products?${params}`)
  return res.json() as Promise<Product[]>
})

// コンポーネントでの使用
function SearchBar() {
  const [query, setQuery] = useAtom(searchQueryAtom)
  return <input value={query} onChange={e => setQuery(e.target.value)} />
}

function ProductList() {
  const products = useAtomValue(filteredProductsAtom)
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
}
```

### Step 4: TanStack Query（サーバー状態管理）

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// クエリキーの定義（ファクトリパターン）
const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: Filters) => [...productKeys.lists(), filters] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
}

// カスタムフック
function useProducts(filters: Filters) {
  return useQuery({
    queryKey: productKeys.list(filters),
    queryFn: () => fetchProducts(filters),
    staleTime: 5 * 60 * 1000,        // 5分間はstale扱いしない
    gcTime: 10 * 60 * 1000,           // 10分間キャッシュ保持
    placeholderData: keepPreviousData, // フィルタ変更時に前データを表示
  })
}

function useProduct(id: string) {
  return useQuery({
    queryKey: productKeys.detail(id),
    queryFn: () => fetchProduct(id),
    enabled: !!id, // idがない場合はフェッチしない
  })
}

// ミューテーション（楽観的更新）
function useUpdateProduct() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: updateProduct,
    onMutate: async (updatedProduct) => {
      // 進行中のクエリをキャンセル
      await queryClient.cancelQueries({
        queryKey: productKeys.detail(updatedProduct.id),
      })

      // 前のデータを保存
      const previous = queryClient.getQueryData(
        productKeys.detail(updatedProduct.id)
      )

      // 楽観的にキャッシュ更新
      queryClient.setQueryData(
        productKeys.detail(updatedProduct.id),
        updatedProduct
      )

      return { previous }
    },
    onError: (_err, updatedProduct, context) => {
      // エラー時にロールバック
      if (context?.previous) {
        queryClient.setQueryData(
          productKeys.detail(updatedProduct.id),
          context.previous
        )
      }
    },
    onSettled: (_data, _err, updatedProduct) => {
      // 成功・失敗問わずリストを再取得
      queryClient.invalidateQueries({ queryKey: productKeys.lists() })
    },
  })
}
```

### Step 5: React Context（低頻度更新のみ）

```tsx
// テーマやロケールなど、変更頻度が低い値に限定
interface AppConfig {
  locale: string
  currency: string
  timezone: string
}

const AppConfigContext = createContext<AppConfig | null>(null)

function useAppConfig(): AppConfig {
  const ctx = useContext(AppConfigContext)
  if (!ctx) throw new Error('useAppConfig must be within AppConfigProvider')
  return ctx
}

// 注意: Contextの値が変わると、全consumerが再レンダリングされる
// 高頻度更新の値はZustand/Jotaiを使う
```

### Step 6: 組み合わせパターン

```
典型的なアプリの状態管理構成:

- サーバーデータ（商品、ユーザー等）     -> TanStack Query
- グローバルUI（サイドバー、モーダル）    -> Zustand
- フォーム                               -> React Hook Form
- テーマ・認証コンテキスト               -> React Context
- ローカルUI（開閉状態、ホバー）         -> useState
```

## 選定比較表

| 項目 | Zustand | Jotai | Redux TK | TanStack Query |
|------|---------|-------|----------|----------------|
| バンドルサイズ | 1.1KB | 2.4KB | 11KB | 13KB |
| 学習コスト | 低 | 低 | 中 | 中 |
| ボイラープレート | 最小 | 最小 | 中 | 少 |
| DevTools | ○ | ○ | ◎ | ◎ |
| 用途 | グローバル状態 | 細粒度状態 | 大規模状態 | サーバー状態 |
| ミドルウェア | persist等 | utils | 豊富 | - |

## チェックリスト

- [ ] サーバー状態とクライアント状態を分離
- [ ] サーバーデータにはTanStack Queryを使用
- [ ] グローバル状態はZustand or Jotai（プロジェクトで統一）
- [ ] Contextは低頻度更新のみに使用
- [ ] Zustandはセレクターで最小限のサブスクリプション
- [ ] 状態は使うコンポーネントに近い場所に配置（state colocation）
- [ ] 不要なグローバル状態を作らない（URLで管理できないか検討）

## ベストプラクティス

1. **サーバー状態 !== クライアント状態**: TanStack Queryでキャッシュ管理を分離する
2. **State Colocation**: 状態は使う場所の近くに置く。安易にグローバル化しない
3. **セレクターで最小サブスクリプション**: `useStore(s => s.count)`で必要な値だけ購読
4. **URL as State**: フィルター、ページネーション、タブはURLパラメータで管理
5. **1プロジェクト1ライブラリ**: ZustandとJotaiを混在させない
