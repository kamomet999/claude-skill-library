---
name: animation-patterns
description: Webアニメーション設計。Framer Motion、CSS Transitions、スクロールアニメーション、パフォーマンス最適化の実践ガイド
category: フロントエンド
command: animation-patterns
version: 1.0.0
tags:
  - animation
  - framer-motion
  - css
  - scroll
  - ux
---

# Web アニメーション設計パターン

## 概要

Webアプリケーションにおけるアニメーションの設計と実装ガイド。
Framer Motion、CSSトランジション、スクロールアニメーション、パフォーマンス最適化をカバーする。

## Steps

### Step 1: アニメーションの目的と原則

```
アニメーションの目的:
1. フィードバック     - ユーザーアクションへの応答
2. コンテキスト       - 要素間の関係を示す
3. 注意誘導          - 重要な変化に注目させる
4. 待機の緩和        - ローディング中の体感時間を短縮

原則:
- 200-500msが心地よいデュレーション（要素サイズに比例）
- ease-out: 要素の出現（最初速く、最後ゆっくり）
- ease-in: 要素の消失（最初ゆっくり、最後速く）
- ease-in-out: 位置の移動
- 同時に動くアニメーションはスタガーをつける（50-100ms間隔）
```

### Step 2: CSSトランジション（軽量な場合）

```tsx
// Tailwind CSSのtransitionユーティリティ
function HoverCard({ children }: Props) {
  return (
    <div className="rounded-lg border bg-white p-6 shadow-sm transition-all duration-200 ease-out hover:-translate-y-1 hover:shadow-md">
      {children}
    </div>
  )
}

// CSSカスタムプロパティでアニメーション管理
// globals.css
const styles = `
  @layer utilities {
    .animate-enter {
      animation: enter 0.2s ease-out;
    }

    .animate-exit {
      animation: exit 0.15s ease-in forwards;
    }
  }

  @keyframes enter {
    from {
      opacity: 0;
      transform: scale(0.95) translateY(8px);
    }
    to {
      opacity: 1;
      transform: scale(1) translateY(0);
    }
  }

  @keyframes exit {
    from {
      opacity: 1;
      transform: scale(1);
    }
    to {
      opacity: 0;
      transform: scale(0.95);
    }
  }
`

// アコーディオン（CSS gridトリック）
function Accordion({ isOpen, children }: Props) {
  return (
    <div
      className="grid transition-[grid-template-rows] duration-300 ease-out"
      style={{ gridTemplateRows: isOpen ? '1fr' : '0fr' }}
    >
      <div className="overflow-hidden">
        {children}
      </div>
    </div>
  )
}
```

### Step 3: Framer Motion基本パターン

```tsx
import { motion, AnimatePresence } from 'framer-motion'

// 基本的な出現アニメーション
function FadeIn({ children }: Props) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 8 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.2, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  )
}

// AnimatePresence（マウント/アンマウント時のアニメーション）
function NotificationList({ notifications }: Props) {
  return (
    <AnimatePresence mode="popLayout">
      {notifications.map(n => (
        <motion.div
          key={n.id}
          layout
          initial={{ opacity: 0, x: 50, scale: 0.95 }}
          animate={{ opacity: 1, x: 0, scale: 1 }}
          exit={{ opacity: 0, x: -50, scale: 0.95 }}
          transition={{ duration: 0.2, ease: 'easeOut' }}
        >
          <Notification notification={n} />
        </motion.div>
      ))}
    </AnimatePresence>
  )
}

// レイアウトアニメーション
function FilterableGrid({ items, filter }: Props) {
  const filtered = items.filter(i => !filter || i.category === filter)

  return (
    <motion.div layout className="grid grid-cols-3 gap-4">
      <AnimatePresence>
        {filtered.map(item => (
          <motion.div
            key={item.id}
            layout
            initial={{ opacity: 0, scale: 0.8 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.8 }}
            transition={{ duration: 0.25 }}
          >
            <Card item={item} />
          </motion.div>
        ))}
      </AnimatePresence>
    </motion.div>
  )
}
```

### Step 4: スタガーアニメーション

```tsx
// リストのスタガー
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.08,
      delayChildren: 0.1,
    },
  },
}

const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.3, ease: 'easeOut' },
  },
}

function StaggeredList({ items }: Props) {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"
    >
      {items.map(item => (
        <motion.li key={item.id} variants={itemVariants}>
          {item.name}
        </motion.li>
      ))}
    </motion.ul>
  )
}

// ページ遷移のスタガー
function PageContent() {
  return (
    <motion.div
      variants={containerVariants}
      initial="hidden"
      animate="visible"
    >
      <motion.h1 variants={itemVariants} className="text-heading-1">
        ダッシュボード
      </motion.h1>
      <motion.div variants={itemVariants}>
        <StatsCards />
      </motion.div>
      <motion.div variants={itemVariants}>
        <RecentActivity />
      </motion.div>
    </motion.div>
  )
}
```

### Step 5: スクロールアニメーション

```tsx
import { motion, useScroll, useTransform } from 'framer-motion'

// Scroll-triggered fade in
function ScrollFadeIn({ children }: Props) {
  const ref = useRef<HTMLDivElement>(null)
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],
  })

  const opacity = useTransform(scrollYProgress, [0, 0.3], [0, 1])
  const y = useTransform(scrollYProgress, [0, 0.3], [50, 0])

  return (
    <motion.div ref={ref} style={{ opacity, y }}>
      {children}
    </motion.div>
  )
}

// パララックス効果
function ParallaxHero({ imageUrl, title }: Props) {
  const ref = useRef<HTMLDivElement>(null)
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start start', 'end start'],
  })

  const backgroundY = useTransform(scrollYProgress, [0, 1], ['0%', '30%'])
  const textOpacity = useTransform(scrollYProgress, [0, 0.5], [1, 0])

  return (
    <div ref={ref} className="relative h-screen overflow-hidden">
      <motion.div
        className="absolute inset-0 bg-cover bg-center"
        style={{
          backgroundImage: `url(${imageUrl})`,
          y: backgroundY,
        }}
      />
      <motion.h1
        className="relative z-10 text-heading-1 text-white"
        style={{ opacity: textOpacity }}
      >
        {title}
      </motion.h1>
    </div>
  )
}

// Intersection Observerベース（軽量版）
function useInView(threshold = 0.1) {
  const ref = useRef<HTMLElement>(null)
  const [isInView, setIsInView] = useState(false)

  useEffect(() => {
    const el = ref.current
    if (!el) return

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true)
          observer.unobserve(el) // 一度だけトリガー
        }
      },
      { threshold }
    )

    observer.observe(el)
    return () => observer.disconnect()
  }, [threshold])

  return { ref, isInView }
}
```

### Step 6: モーダル・ドロワーアニメーション

```tsx
// モーダル
function Modal({ isOpen, onClose, children }: Props) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          {/* オーバーレイ */}
          <motion.div
            className="fixed inset-0 z-40 bg-black/50"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          {/* モーダル本体 */}
          <motion.div
            className="fixed inset-0 z-50 flex items-center justify-center p-4"
            initial={{ opacity: 0, scale: 0.95, y: 10 }}
            animate={{ opacity: 1, scale: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.95, y: 10 }}
            transition={{ duration: 0.2, ease: 'easeOut' }}
          >
            <div className="w-full max-w-lg rounded-lg bg-white p-6 shadow-xl">
              {children}
            </div>
          </motion.div>
        </>
      )}
    </AnimatePresence>
  )
}

// ドロワー（右からスライド）
function Drawer({ isOpen, onClose, children }: Props) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="fixed inset-0 z-40 bg-black/50"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.aside
            className="fixed right-0 top-0 z-50 h-full w-80 bg-white shadow-xl"
            initial={{ x: '100%' }}
            animate={{ x: 0 }}
            exit={{ x: '100%' }}
            transition={{ type: 'spring', damping: 25, stiffness: 300 }}
          >
            {children}
          </motion.aside>
        </>
      )}
    </AnimatePresence>
  )
}
```

### Step 7: パフォーマンス最適化

```
GPUアクセラレーション対象プロパティ（軽い）:
  - transform (translate, scale, rotate)
  - opacity

レイアウトトリガープロパティ（重い・避ける）:
  - width, height
  - top, left, right, bottom
  - margin, padding
  - font-size

ルール:
1. アニメーションはtransformとopacityに限定
2. will-changeは一時的にのみ使用（常時指定はメモリ浪費）
3. prefers-reduced-motionを必ず尊重
4. モバイルでは複雑なアニメーションを簡略化
```

```tsx
// reduced motion対応
function useReducedMotion(): boolean {
  const [reduced, setReduced] = useState(false)

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)')
    setReduced(mq.matches)
    const handler = (e: MediaQueryListEvent) => setReduced(e.matches)
    mq.addEventListener('change', handler)
    return () => mq.removeEventListener('change', handler)
  }, [])

  return reduced
}

// Framer Motionでのreduced motion
// framer-motionはデフォルトでprefers-reduced-motionを尊重する
// ただし明示的に制御したい場合:
<motion.div
  initial={{ opacity: 0, y: reducedMotion ? 0 : 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: reducedMotion ? 0 : 0.3 }}
/>
```

## デュレーションガイド

| 種類 | デュレーション | イージング |
|------|------------|-----------|
| ホバー/フォーカス | 100-150ms | ease-out |
| ボタンクリック | 100ms | ease-out |
| トースト出現 | 200ms | ease-out |
| モーダル出現 | 200-250ms | ease-out |
| ページ遷移 | 200-300ms | ease-in-out |
| リストスタガー | 50-100ms間隔 | ease-out |
| スクロールアニメ | 300-500ms | ease-out |

## チェックリスト

- [ ] アニメーションの目的が明確（フィードバック、コンテキスト、注意誘導）
- [ ] transform/opacityのみ使用（レイアウトプロパティを避ける）
- [ ] prefers-reduced-motionを尊重
- [ ] AnimatePresenceでexit animationを実装
- [ ] デュレーションは200-500ms以内
- [ ] モバイルで複雑なアニメーションを簡略化
- [ ] 60fps維持を確認（DevTools Performance）

## ベストプラクティス

1. **Less is More**: アニメーションは控えめに。過剰なアニメーションはUXを損なう
2. **目的なきアニメーションは削除**: 装飾目的だけのアニメーションは不要
3. **CSS First**: 単純なhover/focusはCSS。複雑な出入りはFramer Motion
4. **reduced motionは必須**: アクセシビリティ要件として必ず対応
5. **springは自然**: 物理ベースのアニメーション（spring）は機械的なeasingより自然
