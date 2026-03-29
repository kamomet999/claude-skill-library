---
name: shell-scripting
description: シェルスクリプトベストプラクティス。Bash安全設定、エラー処理、引数パース、並列実行、ポータビリティ
category: ユーティリティ
command: /shell
version: 1.0.0
tags:
  - bash
  - shell
  - script
  - automation
  - cli
---

# Shell Scripting Best Practices

堅牢で保守しやすいシェルスクリプトの設計・実装パターン集。

## When to Activate

- シェルスクリプトを新規作成するとき
- 既存スクリプトの品質を改善するとき
- CI/CDパイプラインのスクリプトを書くとき
- デプロイ・セットアップ自動化スクリプトを書くとき
- 引数パースや並列実行を実装するとき

## Steps

### Step 1: テンプレート（必ずこれで始める）

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# set -e: エラー時に即座に終了
# set -u: 未定義変数をエラーにする
# set -o pipefail: パイプ中のエラーを検知
# IFS: 安全なフィールドセパレータ

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
```

### Step 2: ログ関数

```bash
# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m' # No Color

log_info() {
  echo -e "${BLUE}[INFO]${NC} $*" >&2
}

log_success() {
  echo -e "${GREEN}[OK]${NC} $*" >&2
}

log_warn() {
  echo -e "${YELLOW}[WARN]${NC} $*" >&2
}

log_error() {
  echo -e "${RED}[ERROR]${NC} $*" >&2
}

die() {
  log_error "$@"
  exit 1
}
```

### Step 3: 引数パース

```bash
# シンプルなgetopts
usage() {
  cat <<EOF
Usage: ${SCRIPT_NAME} [OPTIONS] <target>

Options:
  -e, --env ENV       環境名 (dev|staging|prod) [default: dev]
  -v, --verbose       詳細出力
  -d, --dry-run       実際には実行しない
  -h, --help          このヘルプを表示

Examples:
  ${SCRIPT_NAME} -e staging deploy
  ${SCRIPT_NAME} --dry-run -e prod migrate
EOF
}

# デフォルト値
ENV="dev"
VERBOSE=false
DRY_RUN=false

parse_args() {
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -e|--env)
        ENV="${2:?'--env requires a value'}"
        shift 2
        ;;
      -v|--verbose)
        VERBOSE=true
        shift
        ;;
      -d|--dry-run)
        DRY_RUN=true
        shift
        ;;
      -h|--help)
        usage
        exit 0
        ;;
      --)
        shift
        break
        ;;
      -*)
        die "Unknown option: $1"
        ;;
      *)
        break
        ;;
    esac
  done

  # 残りの引数
  readonly TARGET="${1:?'target is required. See --help'}"
  readonly ENV
  readonly VERBOSE
  readonly DRY_RUN
}

parse_args "$@"
```

### Step 4: クリーンアップ（trap）

```bash
# 一時ファイル管理
TMPDIR_WORK=""

cleanup() {
  local exit_code=$?
  if [[ -n "${TMPDIR_WORK}" && -d "${TMPDIR_WORK}" ]]; then
    rm -rf "${TMPDIR_WORK}"
    log_info "Cleaned up temp directory"
  fi
  if [[ ${exit_code} -ne 0 ]]; then
    log_error "Script failed with exit code ${exit_code}"
  fi
  exit ${exit_code}
}

trap cleanup EXIT ERR INT TERM

TMPDIR_WORK="$(mktemp -d)"
log_info "Using temp dir: ${TMPDIR_WORK}"
```

### Step 5: 前提条件チェック

```bash
check_prerequisites() {
  local missing=()

  for cmd in docker git curl jq; do
    if ! command -v "${cmd}" &>/dev/null; then
      missing+=("${cmd}")
    fi
  done

  if [[ ${#missing[@]} -gt 0 ]]; then
    die "Missing required commands: ${missing[*]}"
  fi

  # バージョンチェック
  local node_version
  node_version="$(node --version 2>/dev/null || echo "none")"
  if [[ "${node_version}" == "none" ]]; then
    die "Node.js is required"
  fi

  local major
  major="$(echo "${node_version}" | sed 's/v//' | cut -d. -f1)"
  if [[ ${major} -lt 20 ]]; then
    die "Node.js >= 20 required (found ${node_version})"
  fi

  log_success "All prerequisites met"
}

check_prerequisites
```

## 実用パターン

### 安全なファイル操作

```bash
# ファイル存在チェック
[[ -f "${CONFIG_FILE}" ]] || die "Config file not found: ${CONFIG_FILE}"
[[ -r "${CONFIG_FILE}" ]] || die "Config file not readable: ${CONFIG_FILE}"

# ディレクトリ作成（冪等）
mkdir -p "${OUTPUT_DIR}"

# アトミックなファイル書き込み（一時ファイル経由）
write_config() {
  local target="$1"
  local tmp="${target}.tmp.$$"

  cat > "${tmp}" <<EOF
KEY=value
ANOTHER=data
EOF

  mv "${tmp}" "${target}"
  log_success "Wrote config to ${target}"
}

# 安全なファイル読み込み
while IFS= read -r line; do
  [[ -z "${line}" || "${line}" == \#* ]] && continue
  process_line "${line}"
done < "${INPUT_FILE}"
```

### 環境変数読み込み

```bash
# .envファイル読み込み（安全版）
load_env() {
  local env_file="${1:-.env}"
  if [[ ! -f "${env_file}" ]]; then
    log_warn "Env file not found: ${env_file}"
    return
  fi

  while IFS= read -r line; do
    # コメントと空行をスキップ
    [[ -z "${line}" || "${line}" == \#* ]] && continue
    # export を除去
    line="${line#export }"
    # 変数名=値 の形式のみ処理
    if [[ "${line}" =~ ^[A-Za-z_][A-Za-z_0-9]*= ]]; then
      export "${line?}"
    fi
  done < "${env_file}"

  log_info "Loaded env from ${env_file}"
}
```

### 並列実行

```bash
# xargsで並列処理
find ./images -name '*.png' -print0 | \
  xargs -0 -P 4 -I{} optipng -o2 {}

# バックグラウンドジョブ + wait
run_parallel() {
  local pids=()

  for service in api worker scheduler; do
    deploy_service "${service}" &
    pids+=($!)
  done

  local failed=0
  for pid in "${pids[@]}"; do
    if ! wait "${pid}"; then
      ((failed++))
    fi
  done

  if [[ ${failed} -gt 0 ]]; then
    die "${failed} service(s) failed to deploy"
  fi

  log_success "All services deployed"
}
```

### リトライ

```bash
retry() {
  local max_attempts="${1}"
  local delay="${2}"
  shift 2
  local cmd=("$@")

  local attempt=1
  while true; do
    if "${cmd[@]}"; then
      return 0
    fi

    if [[ ${attempt} -ge ${max_attempts} ]]; then
      log_error "Command failed after ${max_attempts} attempts: ${cmd[*]}"
      return 1
    fi

    log_warn "Attempt ${attempt}/${max_attempts} failed. Retrying in ${delay}s..."
    sleep "${delay}"
    ((attempt++))
    # Exponential backoff
    delay=$((delay * 2))
  done
}

# 使用例
retry 3 2 curl -sf "https://api.example.com/health"
```

### 確認プロンプト

```bash
confirm() {
  local message="${1:-Are you sure?}"
  local default="${2:-n}"

  if [[ "${default}" == "y" ]]; then
    local prompt="${message} [Y/n] "
  else
    local prompt="${message} [y/N] "
  fi

  read -rp "${prompt}" response
  response="${response:-${default}}"

  [[ "${response}" =~ ^[Yy]$ ]]
}

# 使用例
if [[ "${ENV}" == "prod" ]]; then
  confirm "Deploy to PRODUCTION?" || die "Aborted"
fi
```

### JSON処理（jq）

```bash
# JSONからフィールド抽出
api_url="$(jq -r '.api.url' config.json)"

# JSON配列をループ
jq -r '.services[] | .name' services.json | while read -r name; do
  log_info "Processing: ${name}"
done

# JSONの結合・変換
jq -s '.[0] * .[1]' defaults.json overrides.json > merged.json

# 条件フィルタ
jq '.[] | select(.status == "active") | .name' users.json
```

### プログレス表示

```bash
progress_bar() {
  local current="$1"
  local total="$2"
  local width=40

  local percentage=$((current * 100 / total))
  local filled=$((current * width / total))
  local empty=$((width - filled))

  printf "\r["
  printf "%${filled}s" '' | tr ' ' '#'
  printf "%${empty}s" '' | tr ' ' '-'
  printf "] %3d%% (%d/%d)" "${percentage}" "${current}" "${total}"

  if [[ ${current} -eq ${total} ]]; then
    echo ""
  fi
}

# 使用例
total=100
for i in $(seq 1 ${total}); do
  progress_bar "${i}" "${total}"
  sleep 0.05
done
```

## デプロイスクリプト例

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/common.sh"

ENV="${1:?'Usage: deploy.sh <env> (dev|staging|prod)'}"

# 環境バリデーション
case "${ENV}" in
  dev|staging|prod) ;;
  *) die "Invalid env: ${ENV}. Must be dev, staging, or prod" ;;
esac

# 本番確認
if [[ "${ENV}" == "prod" ]]; then
  confirm "Deploy to PRODUCTION?" || die "Aborted"
fi

log_info "Deploying to ${ENV}..."

# ビルド
log_info "Building..."
pnpm build || die "Build failed"

# テスト
log_info "Running tests..."
pnpm test || die "Tests failed"

# デプロイ
log_info "Deploying..."
retry 3 5 deploy_to_env "${ENV}"

log_success "Deploy to ${ENV} completed!"
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `set -e`なし | エラーが無視される | 常に`set -euo pipefail` |
| クォートなし変数 | ワード分割、グロブ展開 | `"${var}"` |
| `cd`の放置 | 予期しないディレクトリ | サブシェル `(cd dir && cmd)` |
| `eval`使用 | コードインジェクション | 配列やパラメータ展開 |
| `/bin/bash` | 環境依存 | `/usr/bin/env bash` |
| `cat file \| grep` | 無駄なプロセス | `grep pattern file` |

## ShellCheck

```bash
# インストール
# macOS: brew install shellcheck
# Ubuntu: apt install shellcheck

# 実行
shellcheck script.sh

# CIで使用
shellcheck --severity=warning scripts/*.sh

# 特定ルールの無視
# shellcheck disable=SC2034
UNUSED_VAR="ok"
```

## Best Practices チェックリスト

- [ ] `set -euo pipefail`を設定
- [ ] 全変数をダブルクォート `"${var}"`
- [ ] `trap`でクリーンアップ
- [ ] 前提条件を冒頭でチェック
- [ ] `readonly`で定数を保護
- [ ] `local`で関数内変数をスコープ
- [ ] ShellCheckでlint通過
- [ ] usage関数でヘルプ表示

## Related

- Skill: `docker-optimize` - Docker最適化
- Skill: `monorepo-management` - モノレポ管理（CI/CDスクリプト）
