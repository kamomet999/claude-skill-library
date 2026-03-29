---
name: playwright-testing
description: Playwright E2Eテスト設計パターン。Page Object Model、フィクスチャ、並列実行、ビジュアルテスト
category: テスト
command: /playwright
version: 1.0.0
tags:
  - playwright
  - e2e
  - testing
  - pom
  - visual
---

# Playwright Testing Patterns

Playwrightを使ったE2Eテストの設計・実装パターン集。

## When to Activate

- E2Eテストを新規セットアップするとき
- Page Object Modelを設計するとき
- テストのフレーキネス（不安定性）を改善するとき
- ビジュアルリグレッションテストを導入するとき
- CI/CDにPlaywrightを組み込むとき

## Steps

### Step 1: セットアップ

```bash
pnpm create playwright
# or
pnpm add -D @playwright/test
pnpm exec playwright install
```

### Step 2: playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    ...(process.env.CI ? [['github'] as const] : []),
  ],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 13'] } },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
})
```

### Step 3: Page Object Model

```typescript
// e2e/pages/login.page.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('メールアドレス')
    this.passwordInput = page.getByLabel('パスワード')
    this.submitButton = page.getByRole('button', { name: 'ログイン' })
    this.errorMessage = page.getByRole('alert')
  }

  async goto(): Promise<void> {
    await this.page.goto('/login')
  }

  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }

  async expectError(message: string): Promise<void> {
    await expect(this.errorMessage).toContainText(message)
  }

  async expectLoggedIn(): Promise<void> {
    await expect(this.page).toHaveURL('/dashboard')
  }
}

// e2e/pages/dashboard.page.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class DashboardPage {
  readonly page: Page
  readonly heading: Locator
  readonly userMenu: Locator
  readonly logoutButton: Locator

  constructor(page: Page) {
    this.page = page
    this.heading = page.getByRole('heading', { name: 'ダッシュボード' })
    this.userMenu = page.getByTestId('user-menu')
    this.logoutButton = page.getByRole('menuitem', { name: 'ログアウト' })
  }

  async expectLoaded(): Promise<void> {
    await expect(this.heading).toBeVisible()
  }

  async logout(): Promise<void> {
    await this.userMenu.click()
    await this.logoutButton.click()
  }
}
```

### Step 4: カスタムフィクスチャ

```typescript
// e2e/fixtures.ts
import { test as base } from '@playwright/test'
import { LoginPage } from './pages/login.page'
import { DashboardPage } from './pages/dashboard.page'

interface TestFixtures {
  loginPage: LoginPage
  dashboardPage: DashboardPage
  authenticatedPage: Page
}

export const test = base.extend<TestFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page))
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page))
  },
  authenticatedPage: async ({ page }, use) => {
    // 認証状態をセットアップ
    await page.goto('/login')
    const loginPage = new LoginPage(page)
    await loginPage.login('test@example.com', 'password123')
    await page.waitForURL('/dashboard')
    await use(page)
  },
})

export { expect } from '@playwright/test'
```

### Step 5: テスト実装

```typescript
// e2e/auth/login.spec.ts
import { test, expect } from '../fixtures'

test.describe('ログイン', () => {
  test('正しい認証情報でログインできる', async ({ loginPage }) => {
    await loginPage.goto()
    await loginPage.login('test@example.com', 'password123')
    await loginPage.expectLoggedIn()
  })

  test('間違ったパスワードでエラーが表示される', async ({ loginPage }) => {
    await loginPage.goto()
    await loginPage.login('test@example.com', 'wrong')
    await loginPage.expectError('メールアドレスまたはパスワードが正しくありません')
  })

  test('空のフォーム送信でバリデーションエラーが表示される', async ({ loginPage }) => {
    await loginPage.goto()
    await loginPage.submitButton.click()
    await expect(loginPage.emailInput).toHaveAttribute('aria-invalid', 'true')
  })
})
```

## Advanced Patterns

### 認証状態の永続化（storageState）

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test'

const authFile = 'e2e/.auth/user.json'

setup('authenticate', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('メールアドレス').fill('test@example.com')
  await page.getByLabel('パスワード').fill('password123')
  await page.getByRole('button', { name: 'ログイン' }).click()
  await page.waitForURL('/dashboard')
  await page.context().storageState({ path: authFile })
})

// playwright.config.ts で使用
// projects: [
//   { name: 'setup', testMatch: /.*\.setup\.ts/ },
//   { name: 'chromium', use: { storageState: authFile }, dependencies: ['setup'] },
// ]
```

### API Mocking

```typescript
test('APIエラー時にフォールバックUIが表示される', async ({ page }) => {
  await page.route('**/api/products', (route) =>
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal Server Error' }),
    })
  )

  await page.goto('/products')
  await expect(page.getByText('データの読み込みに失敗しました')).toBeVisible()
})

test('ローディング状態が表示される', async ({ page }) => {
  await page.route('**/api/products', async (route) => {
    await new Promise((r) => setTimeout(r, 3000))
    await route.continue()
  })

  await page.goto('/products')
  await expect(page.getByTestId('skeleton-loader')).toBeVisible()
})
```

### ビジュアルリグレッション

```typescript
test('ダッシュボードのスナップショット', async ({ page }) => {
  await page.goto('/dashboard')
  await page.waitForLoadState('networkidle')

  // フルページスクリーンショット比較
  await expect(page).toHaveScreenshot('dashboard.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.01,
  })

  // コンポーネント単位の比較
  const chart = page.getByTestId('revenue-chart')
  await expect(chart).toHaveScreenshot('revenue-chart.png')
})

// アニメーション無効化
test.use({
  actionTimeout: 10000,
  // CSSアニメーションを無効化してスナップショットを安定化
})
```

### データ駆動テスト

```typescript
const testUsers = [
  { role: 'admin', expectedTabs: ['ユーザー管理', '設定', 'ログ'] },
  { role: 'editor', expectedTabs: ['コンテンツ', '設定'] },
  { role: 'viewer', expectedTabs: ['ダッシュボード'] },
]

for (const { role, expectedTabs } of testUsers) {
  test(`${role}ロールのユーザーは適切なタブが表示される`, async ({ page }) => {
    await loginAs(page, role)
    await page.goto('/dashboard')

    for (const tab of expectedTabs) {
      await expect(page.getByRole('tab', { name: tab })).toBeVisible()
    }
  })
}
```

### アクセシビリティテスト

```typescript
import AxeBuilder from '@axe-core/playwright'

test('ダッシュボードにアクセシビリティ違反がない', async ({ page }) => {
  await page.goto('/dashboard')

  const accessibilityScanResults = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze()

  expect(accessibilityScanResults.violations).toEqual([])
})
```

## ロケーター優先順位

| Priority | Locator | Example |
|----------|---------|---------|
| 1 | getByRole | `getByRole('button', { name: '送信' })` |
| 2 | getByLabel | `getByLabel('メールアドレス')` |
| 3 | getByText | `getByText('ログイン')` |
| 4 | getByTestId | `getByTestId('submit-btn')` |
| 5 | CSS selector | `locator('.btn-primary')` (最終手段) |

## フレーキネス対策

| Problem | Solution |
|---------|----------|
| 要素が見つからない | `waitFor()`やAutomatic waitingを信頼する |
| タイミング問題 | `expect().toBeVisible()`で待機 |
| アニメーション | `animations: 'disabled'`設定 |
| ネットワーク | `waitForResponse()`や`route()`でモック |
| テスト間依存 | 各テストを独立に、フィクスチャでセットアップ |
| Date.now() | `page.clock`で時刻を固定 |

## CI/CD設定（GitHub Actions）

```yaml
name: E2E Tests
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      - run: pnpm exec playwright install --with-deps
      - run: pnpm exec playwright test
      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

## ファイル構成

```
e2e/
├── fixtures.ts           # カスタムフィクスチャ
├── auth.setup.ts         # 認証セットアップ
├── .auth/                # 認証状態（.gitignore）
├── pages/                # Page Objects
│   ├── login.page.ts
│   ├── dashboard.page.ts
│   └── settings.page.ts
├── auth/                 # 認証テスト
│   └── login.spec.ts
├── dashboard/            # ダッシュボードテスト
│   └── overview.spec.ts
└── utils/                # テストユーティリティ
    └── helpers.ts
playwright.config.ts
```

## Related

- Skill: `api-testing` - APIテスト設計
- Skill: `tdd-workflow` - TDDワークフロー
- Agent: `e2e-runner` - E2Eテスト実行エージェント
