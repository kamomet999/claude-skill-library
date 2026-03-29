---
name: monorepo-management
description: モノレポ管理パターン。Turborepo/Nx設定、ワークスペース、依存関係、ビルドキャッシュ、変更検知
category: アーキテクチャ
command: /monorepo
version: 1.0.0
tags:
  - monorepo
  - turborepo
  - nx
  - workspace
  - build-cache
---

# Monorepo Management Patterns

Turborepo/Nxを使ったモノレポの設計・運用パターン集。

## When to Activate

- モノレポをセットアップするとき
- Turborepo/Nxの設定を最適化するとき
- ワークスペース間の依存関係を管理するとき
- ビルドキャッシュ戦略を設計するとき
- CI/CDで変更検知ビルドを構築するとき

## Steps

### Step 1: ツール選定

| Tool | Best For | Key Feature |
|------|----------|-------------|
| Turborepo | JS/TSプロジェクト、シンプル | 設定少、高速キャッシュ |
| Nx | 大規模、多言語 | プラグイン豊富、依存グラフ |
| pnpm workspace | 軽量、基本機能で十分 | パッケージマネージャ内蔵 |
| Lerna | レガシー、npm publish | 既存プロジェクト互換 |

### Step 2: Turborepoセットアップ

```bash
pnpm dlx create-turbo@latest my-monorepo
# or 既存プロジェクトに追加
pnpm add -Dw turbo
```

#### ディレクトリ構成

```
my-monorepo/
├── apps/
│   ├── web/                  # Next.jsフロントエンド
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── api/                  # Express/Fastifyバックエンド
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── admin/                # 管理画面
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── ui/                   # 共有UIコンポーネント
│   │   ├── package.json
│   │   └── src/
│   ├── db/                   # DB層（Drizzle schema等）
│   │   ├── package.json
│   │   └── src/
│   ├── config-eslint/        # 共有ESLint設定
│   │   └── package.json
│   ├── config-typescript/    # 共有tsconfig
│   │   └── base.json
│   └── shared/               # 共有型・ユーティリティ
│       ├── package.json
│       └── src/
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── .gitignore
```

### Step 3: pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### Step 4: turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "package.json", "tsconfig.json"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "env": ["DATABASE_URL", "NEXT_PUBLIC_*"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", ".eslintrc*", "eslint.config.*"]
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "test/**", "vitest.config.*"]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig.json"]
    },
    "db:migrate": {
      "cache": false
    }
  }
}
```

### Step 5: ルートpackage.json

```json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "format": "prettier --write \"**/*.{ts,tsx,md}\"",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^2",
    "prettier": "^3"
  },
  "packageManager": "pnpm@9.15.0"
}
```

### Step 6: パッケージ間参照

```json
// apps/web/package.json
{
  "name": "@repo/web",
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/shared": "workspace:*",
    "@repo/db": "workspace:*"
  }
}

// packages/ui/package.json
{
  "name": "@repo/ui",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./button": "./src/components/button.tsx",
    "./card": "./src/components/card.tsx"
  },
  "dependencies": {
    "@repo/shared": "workspace:*"
  }
}
```

## 共有設定パターン

### TypeScript設定

```json
// packages/config-typescript/base.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "isolatedModules": true
  }
}

// apps/web/tsconfig.json
{
  "extends": "@repo/config-typescript/base.json",
  "compilerOptions": {
    "jsx": "preserve",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "plugins": [{ "name": "next" }]
  },
  "include": ["src", "next-env.d.ts"],
  "exclude": ["node_modules"]
}
```

### ESLint設定（Flat Config）

```javascript
// packages/config-eslint/base.js
import js from '@eslint/js'
import tseslint from 'typescript-eslint'

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },
]

// apps/web/eslint.config.js
import baseConfig from '@repo/config-eslint/base.js'

export default [
  ...baseConfig,
  // app-specific rules
]
```

## ビルドキャッシュ戦略

### ローカルキャッシュ

```bash
# デフォルトで有効。./node_modules/.cache/turbo に保存
turbo run build

# キャッシュ無効化
turbo run build --force

# キャッシュ状況確認
turbo run build --summarize
```

### リモートキャッシュ（Vercel）

```bash
# Vercelアカウント連携
pnpm dlx turbo login
pnpm dlx turbo link

# 環境変数でCI設定
# TURBO_TOKEN=xxx
# TURBO_TEAM=my-team
```

### カスタムリモートキャッシュ（S3互換）

```bash
# turbo.json で設定不要。環境変数で制御
TURBO_API=https://my-cache-server.example.com
TURBO_TOKEN=xxx
TURBO_TEAM=my-team
```

## CI/CD（GitHub Actions）

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # turbo --filter に必要

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install

      # 変更されたパッケージのみビルド
      - run: pnpm turbo run build lint test --filter="...[origin/main...HEAD]"
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
```

## フィルタリング

```bash
# 特定パッケージのみ
turbo run build --filter=@repo/web

# 依存関係を含めて
turbo run build --filter=@repo/web...

# 変更されたパッケージのみ（git diff）
turbo run build --filter="...[HEAD~1]"
turbo run build --filter="...[origin/main...HEAD]"

# ディレクトリベース
turbo run build --filter="./apps/*"
turbo run build --filter="./packages/ui"
```

## 新規パッケージ追加テンプレート

```bash
# 1. ディレクトリ作成
mkdir -p packages/new-pkg/src

# 2. package.json
cat > packages/new-pkg/package.json << 'EOF'
{
  "name": "@repo/new-pkg",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "build": "tsc",
    "lint": "eslint src/",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  }
}
EOF

# 3. tsconfig.json
cat > packages/new-pkg/tsconfig.json << 'EOF'
{
  "extends": "@repo/config-typescript/base.json",
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
EOF

# 4. エントリポイント
echo 'export {}' > packages/new-pkg/src/index.ts

# 5. 依存関係インストール
pnpm install
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| 循環依存 | ビルド不可 | 共通部分をsharedパッケージに抽出 |
| 全ビルド | CI遅い | `--filter`で変更分のみビルド |
| パッケージ肥大化 | 結合度が高い | 単一責任でパッケージ分割 |
| バージョン不一致 | 依存地獄 | `syncpack`やpnpm catalog使用 |
| outputs未設定 | キャッシュ効かない | turbo.jsonのoutputsを正確に設定 |

## Best Practices

- `workspace:*`で内部パッケージを参照する
- 各パッケージに独立した`tsconfig.json`を持たせる
- `turbo.json`の`inputs`/`outputs`を正確に設定してキャッシュ効率を上げる
- CIでは`--filter`を使い変更パッケージのみビルドする
- 共有設定（ESLint, TypeScript, Prettier）は専用パッケージにまとめる

## Related

- Skill: `coding-standards` - コーディング標準
- Skill: `docker-optimize` - Docker最適化
- Agent: `architect` - アーキテクチャ設計
