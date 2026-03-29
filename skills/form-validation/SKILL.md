---
name: form-validation
description: フォームバリデーション設計。React Hook Form、Zod、エラーUX、アクセシビリティ、多段階フォームの実践ガイド
category: フロントエンド
command: form-validation
version: 1.0.0
tags:
  - form
  - validation
  - zod
  - react-hook-form
  - ux
---

# フォームバリデーション設計スキル

## 概要

React Hook FormとZodを組み合わせた型安全なフォームバリデーションの設計ガイド。
エラーUX、アクセシビリティ、多段階フォーム、サーバーサイドバリデーション連携をカバーする。

## Steps

### Step 1: Zodスキーマ定義

```ts
import { z } from 'zod'

// 基本的なユーザー登録スキーマ
const userRegistrationSchema = z.object({
  email: z
    .string()
    .min(1, 'メールアドレスを入力してください')
    .email('正しいメールアドレスを入力してください'),
  password: z
    .string()
    .min(8, 'パスワードは8文字以上で入力してください')
    .regex(/[A-Z]/, '大文字を1文字以上含めてください')
    .regex(/[0-9]/, '数字を1文字以上含めてください'),
  confirmPassword: z.string(),
  name: z
    .string()
    .min(1, '名前を入力してください')
    .max(50, '名前は50文字以内で入力してください'),
  age: z
    .number({ invalid_type_error: '数値を入力してください' })
    .int('整数を入力してください')
    .min(13, '13歳以上である必要があります')
    .max(150, '正しい年齢を入力してください'),
  agreeToTerms: z.literal(true, {
    errorMap: () => ({ message: '利用規約への同意が必要です' }),
  }),
}).refine(data => data.password === data.confirmPassword, {
  message: 'パスワードが一致しません',
  path: ['confirmPassword'],
})

type UserRegistration = z.infer<typeof userRegistrationSchema>
```

**再利用可能なスキーマパーツ**:

```ts
// 共通バリデーションを切り出す
const emailSchema = z
  .string()
  .min(1, 'メールアドレスを入力してください')
  .email('正しいメールアドレスを入力してください')

const passwordSchema = z
  .string()
  .min(8, 'パスワードは8文字以上で入力してください')
  .regex(/[A-Z]/, '大文字を1文字以上含めてください')
  .regex(/[0-9]/, '数字を1文字以上含めてください')

const japanesePhoneSchema = z
  .string()
  .regex(/^0\d{9,10}$/, '正しい電話番号を入力してください')

const postalCodeSchema = z
  .string()
  .regex(/^\d{3}-?\d{4}$/, '正しい郵便番号を入力してください')

// 条件付きバリデーション
const shippingSchema = z.discriminatedUnion('method', [
  z.object({
    method: z.literal('delivery'),
    address: z.string().min(1, '住所を入力してください'),
    postalCode: postalCodeSchema,
  }),
  z.object({
    method: z.literal('pickup'),
    storeId: z.string().min(1, '店舗を選択してください'),
  }),
])
```

### Step 2: React Hook Form統合

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'

function RegistrationForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isDirty },
    setError,
    reset,
  } = useForm<UserRegistration>({
    resolver: zodResolver(userRegistrationSchema),
    defaultValues: {
      email: '',
      password: '',
      confirmPassword: '',
      name: '',
      agreeToTerms: false,
    },
    mode: 'onBlur', // フォーカスが外れた時にバリデーション
  })

  const onSubmit = async (data: UserRegistration) => {
    try {
      await registerUser(data)
      reset()
    } catch (error) {
      if (error instanceof ApiError && error.code === 'EMAIL_EXISTS') {
        setError('email', {
          message: 'このメールアドレスは既に登録されています',
        })
      } else {
        setError('root', {
          message: '登録に失敗しました。しばらくしてからお試しください。',
        })
      }
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      {errors.root && (
        <div role="alert" className="rounded-md bg-red-50 p-4 text-red-800">
          {errors.root.message}
        </div>
      )}

      <FormField label="メールアドレス" error={errors.email}>
        <input
          {...register('email')}
          type="email"
          autoComplete="email"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
      </FormField>

      <FormField label="パスワード" error={errors.password}>
        <input
          {...register('password')}
          type="password"
          autoComplete="new-password"
          aria-invalid={!!errors.password}
        />
      </FormField>

      <button type="submit" disabled={isSubmitting || !isDirty}>
        {isSubmitting ? '送信中...' : '登録'}
      </button>
    </form>
  )
}
```

### Step 3: アクセシブルなFormFieldコンポーネント

```tsx
import { type FieldError } from 'react-hook-form'

interface FormFieldProps {
  label: string
  error?: FieldError
  hint?: string
  required?: boolean
  children: React.ReactElement
}

function FormField({ label, error, hint, required, children }: FormFieldProps) {
  const id = useId()
  const errorId = `${id}-error`
  const hintId = `${id}-hint`

  const describedBy = [
    error ? errorId : null,
    hint ? hintId : null,
  ].filter(Boolean).join(' ') || undefined

  return (
    <div className="space-y-1.5">
      <label htmlFor={id} className="block text-sm font-medium text-content">
        {label}
        {required && <span className="ml-1 text-red-500" aria-hidden="true">*</span>}
        {required && <span className="sr-only">（必須）</span>}
      </label>

      {hint && (
        <p id={hintId} className="text-sm text-content-tertiary">
          {hint}
        </p>
      )}

      {cloneElement(children, {
        id,
        'aria-describedby': describedBy,
        'aria-invalid': !!error,
        className: cn(
          children.props.className,
          'block w-full rounded-input border px-3 py-2 text-body',
          'focus:outline-none focus:ring-2 focus:ring-brand-500',
          error
            ? 'border-red-500 text-red-900'
            : 'border-gray-300 dark:border-gray-600'
        ),
      })}

      {error && (
        <p id={errorId} role="alert" className="text-sm text-red-600">
          {error.message}
        </p>
      )}
    </div>
  )
}
```

### Step 4: 多段階フォーム（ウィザード）

```tsx
import { z } from 'zod'
import { useForm, FormProvider } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { useState, useCallback } from 'react'

// 各ステップのスキーマ
const stepSchemas = [
  z.object({
    name: z.string().min(1, '名前を入力してください'),
    email: emailSchema,
  }),
  z.object({
    company: z.string().min(1, '会社名を入力してください'),
    role: z.string().min(1, '役職を選択してください'),
  }),
  z.object({
    plan: z.enum(['free', 'pro', 'enterprise']),
    agreeToTerms: z.literal(true),
  }),
] as const

// 全体スキーマ（最終送信時に使用）
const fullSchema = z.object({
  name: z.string().min(1),
  email: emailSchema,
  company: z.string().min(1),
  role: z.string().min(1),
  plan: z.enum(['free', 'pro', 'enterprise']),
  agreeToTerms: z.literal(true),
})

type FormData = z.infer<typeof fullSchema>

function MultiStepForm() {
  const [step, setStep] = useState(0)
  const currentSchema = stepSchemas[step]

  const methods = useForm<FormData>({
    resolver: zodResolver(currentSchema),
    mode: 'onBlur',
  })

  const goNext = useCallback(async () => {
    const valid = await methods.trigger()
    if (valid) setStep(s => Math.min(s + 1, stepSchemas.length - 1))
  }, [methods])

  const goBack = useCallback(() => {
    setStep(s => Math.max(s - 1, 0))
  }, [])

  const onSubmit = async (data: FormData) => {
    const validated = fullSchema.parse(data)
    await submitRegistration(validated)
  }

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {/* プログレスバー */}
        <nav aria-label="登録ステップ">
          <ol className="flex gap-2">
            {['基本情報', '会社情報', 'プラン選択'].map((label, i) => (
              <li
                key={label}
                className={cn(
                  'flex-1 rounded-full h-2',
                  i <= step ? 'bg-brand-500' : 'bg-gray-200'
                )}
                aria-current={i === step ? 'step' : undefined}
              >
                <span className="sr-only">{label}</span>
              </li>
            ))}
          </ol>
        </nav>

        {/* ステップコンテンツ */}
        {step === 0 && <StepBasicInfo />}
        {step === 1 && <StepCompanyInfo />}
        {step === 2 && <StepPlanSelect />}

        {/* ナビゲーション */}
        <div className="flex justify-between mt-6">
          {step > 0 && (
            <button type="button" onClick={goBack}>戻る</button>
          )}
          {step < stepSchemas.length - 1 ? (
            <button type="button" onClick={goNext}>次へ</button>
          ) : (
            <button type="submit" disabled={methods.formState.isSubmitting}>
              登録する
            </button>
          )}
        </div>
      </form>
    </FormProvider>
  )
}
```

### Step 5: リアルタイムバリデーション（非同期）

```tsx
// メールアドレスの重複チェック（デバウンス付き）
const emailSchema = z
  .string()
  .email('正しいメールアドレスを入力してください')
  .refine(async (email) => {
    const response = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`)
    const { available } = await response.json()
    return available
  }, 'このメールアドレスは既に使用されています')

// useFormでの非同期バリデーション（debounce推奨）
const { register } = useForm({
  resolver: zodResolver(schema),
  mode: 'onChange',         // 入力のたびにバリデーション
  delayError: 500,          // エラー表示を500ms遅延
})
```

### Step 6: エラーUXパターン

```tsx
// パスワード強度インジケーター
function PasswordStrength({ password }: { password: string }) {
  const strength = useMemo(() => {
    let score = 0
    if (password.length >= 8) score++
    if (/[A-Z]/.test(password)) score++
    if (/[0-9]/.test(password)) score++
    if (/[^A-Za-z0-9]/.test(password)) score++
    return score
  }, [password])

  const labels = ['弱い', '普通', '強い', '非常に強い']
  const colors = ['bg-red-500', 'bg-yellow-500', 'bg-blue-500', 'bg-green-500']

  return (
    <div aria-label={`パスワード強度: ${labels[strength - 1] ?? '未入力'}`}>
      <div className="flex gap-1">
        {[0, 1, 2, 3].map(i => (
          <div
            key={i}
            className={cn(
              'h-1 flex-1 rounded-full transition-colors',
              i < strength ? colors[strength - 1] : 'bg-gray-200'
            )}
          />
        ))}
      </div>
      {strength > 0 && (
        <p className="mt-1 text-xs text-content-secondary">
          {labels[strength - 1]}
        </p>
      )}
    </div>
  )
}
```

## バリデーション戦略

| タイミング | 使用場面 | mode |
|-----------|---------|------|
| onBlur | 一般的なフォーム（推奨デフォルト） | `onBlur` |
| onChange | リアルタイムフィードバックが必要 | `onChange` + `delayError` |
| onSubmit | シンプルなフォーム、サーバーエラー主体 | `onSubmit` |
| all | 送信後は全変更を即時チェック | `onTouched` |

## チェックリスト

- [ ] Zodスキーマで型安全なバリデーション
- [ ] エラーメッセージは日本語で具体的に
- [ ] `aria-invalid`、`aria-describedby`、`role="alert"`を設定
- [ ] 必須フィールドにvisual + sr-only両方のインジケーター
- [ ] フォーカス管理（エラー時は最初のエラーフィールドにフォーカス）
- [ ] サーバーエラーを`setError`で表示
- [ ] 送信中はボタンを無効化 + ローディング表示
- [ ] `noValidate`でブラウザデフォルトバリデーションを無効化

## ベストプラクティス

1. **スキーマファースト**: UIより先にZodスキーマを定義する
2. **エラーメッセージは具体的に**: 「無効です」ではなく「8文字以上で入力してください」
3. **早すぎるエラー表示を避ける**: `onBlur`またはsubmit後に表示開始
4. **サーバーバリデーションも必須**: クライアントバリデーションはUX用、セキュリティはサーバーで担保
5. **フォームの状態をURLと同期**: 多段階フォームのステップをURLに反映
