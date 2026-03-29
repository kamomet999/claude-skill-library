---
name: ci-cd-github-actions
description: GitHub Actions CI/CD設計。マトリックスビルド、キャッシュ、シークレット管理、自動リリース
category: DevOps
command: /ci-cd-github-actions
version: 1.0.0
tags: [github-actions, ci, cd, automation, release]
---

# GitHub Actions CI/CD 設計スキル

## 概要

GitHub Actions を使った CI/CD パイプライン設計スキル。
マトリックスビルド、依存キャッシュ、シークレット管理、自動リリースまでカバーする。

## Steps

### Step 1: 基本 CI ワークフロー

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      fail-fast: false
      matrix:
        node-version: [18, 20, 22]
        shard: [1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --shard=${{ matrix.shard }}/3
      - name: Upload coverage
        if: matrix.node-version == 20 && matrix.shard == 1
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/lcov.info

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7
```

### Step 2: キャッシュ戦略

```yaml
# 高度なキャッシュ設定
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # pnpm キャッシュ
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      # Turborepo キャッシュ（モノレポ向け）
      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-${{ github.sha }}
          restore-keys: turbo-

      # Next.js ビルドキャッシュ
      - name: Cache Next.js
        uses: actions/cache@v4
        with:
          path: |
            ${{ github.workspace }}/.next/cache
          key: nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-${{ hashFiles('**/*.ts', '**/*.tsx') }}
          restore-keys: |
            nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-
            nextjs-

      # Docker レイヤーキャッシュ
      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: docker-${{ hashFiles('Dockerfile') }}-${{ github.sha }}
          restore-keys: docker-${{ hashFiles('Dockerfile') }}-

      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

### Step 3: Docker ビルド & プッシュ

```yaml
# .github/workflows/docker.yml
name: Docker Build

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=

      - uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NODE_ENV=production
```

### Step 4: シークレット管理

```yaml
# シークレットの安全な使い方
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Environment protection rules を使う
    steps:
      - uses: actions/checkout@v4

      # OIDC で AWS 認証（長期キー不要）
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-arn: arn:aws:iam::123456789012:role/github-actions
          aws-region: ap-northeast-1

      # シークレットをファイルに書き出す場合
      - name: Create env file
        run: |
          cat > .env.production << 'ENVEOF'
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          REDIS_URL=${{ secrets.REDIS_URL }}
          API_SECRET=${{ secrets.API_SECRET }}
          ENVEOF

      # デプロイ
      - run: ./scripts/deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}

      # クリーンアップ（失敗時も実行）
      - name: Cleanup
        if: always()
        run: rm -f .env.production
```

**シークレット管理のベストプラクティス:**

```yaml
# Environment ごとにシークレットを分離
# Settings > Environments で設定:
#   - production: Required reviewers, deployment branches (main only)
#   - staging: deployment branches (develop, release/*)
#   - development: No restrictions

# OIDC による短期トークン（推奨）
permissions:
  id-token: write  # OIDC トークン発行に必要

# Repository secrets vs Environment secrets
# - Repository: 全環境共通（DOCKER_USERNAME など）
# - Environment: 環境固有（DATABASE_URL など）
```

### Step 5: 自動リリース

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          release-type: node

  publish:
    needs: release-please
    if: needs.release-please.outputs.release_created
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: 'https://registry.npmjs.org'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - run: pnpm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Step 6: E2E テスト

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  pull_request:
    branches: [main]
  deployment_status:

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile

      - name: Install Playwright
        run: pnpm exec playwright install --with-deps chromium

      - name: Run E2E
        run: pnpm exec playwright test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          BASE_URL: http://localhost:3000

      - name: Upload report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14
```

### Step 7: Reusable Workflow

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      KUBE_CONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - name: Set kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config

      - name: Deploy
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ inputs.image-tag }} \
            -n ${{ inputs.environment }}
          kubectl rollout status deployment/myapp -n ${{ inputs.environment }} --timeout=300s
```

**呼び出し側:**

```yaml
# .github/workflows/deploy-production.yml
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image-tag: ${{ github.sha }}
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

## ベストプラクティス

1. **concurrency**: 同一ブランチの重複ビルドをキャンセルする
2. **fail-fast: false**: マトリックスビルドで1つ失敗しても他を続行
3. **--frozen-lockfile**: CI では lockfile を厳密に使う
4. **timeout-minutes**: ジョブにタイムアウトを必ず設定
5. **permissions**: 最小権限の原則を適用（デフォルトは read-only）
6. **OIDC**: 長期シークレットの代わりに OIDC 短期トークンを使う
7. **artifact retention**: 不要なアーティファクトの保持期間を短く設定
8. **Environment protection**: production には必ず承認ルールを設定

## デバッグ

```yaml
# デバッグ用ステップ
- name: Debug context
  if: runner.debug == '1'
  run: |
    echo "Event: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"

# ローカルでのテスト（act を使用）
# act -j test --secret-file .secrets
```
