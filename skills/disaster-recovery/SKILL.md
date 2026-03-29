---
name: disaster-recovery
description: 災害復旧計画設計。バックアップ戦略、RTO/RPO、フェイルオーバー、カオスエンジニアリング
category: DevOps
command: /disaster-recovery
version: 1.0.0
tags: [disaster-recovery, backup, failover, rto, chaos]
---

# 災害復旧計画設計スキル

## 概要

プロダクション環境の災害復旧（DR）計画設計スキル。バックアップ戦略、RTO/RPO 定義、
フェイルオーバー設計、カオスエンジニアリングによる耐障害性検証をカバーする。

## Steps

### Step 1: RTO / RPO の定義

| 指標 | 定義 | 質問 |
|------|------|------|
| **RTO** (Recovery Time Objective) | 復旧にかかる最大許容時間 | どれだけの時間ダウンを許容できるか？ |
| **RPO** (Recovery Point Objective) | 許容できるデータ損失の最大期間 | どれだけのデータ損失を許容できるか？ |

**ティア分類:**

| ティア | RTO | RPO | 戦略 | コスト | 例 |
|--------|-----|-----|------|--------|-----|
| Tier 1 | < 15分 | 0（データ損失なし） | Active-Active | 極高 | 決済システム |
| Tier 2 | < 1時間 | < 15分 | Hot Standby | 高 | ECサイト |
| Tier 3 | < 4時間 | < 1時間 | Warm Standby | 中 | 管理画面 |
| Tier 4 | < 24時間 | < 24時間 | Cold Standby | 低 | バッチ処理 |
| Tier 5 | < 72時間 | < 7日 | Backup/Restore | 最低 | アーカイブ |

### Step 2: バックアップ戦略（3-2-1 ルール）

**3-2-1 ルール:**
- **3** コピーのデータを保持
- **2** 種類の異なるメディアに保存
- **1** つはオフサイト（別リージョン/別クラウド）に保存

```yaml
# PostgreSQL バックアップ設定
# バックアップスクリプト
```

```bash
#!/bin/bash
# backup-postgres.sh

set -euo pipefail

DB_NAME="myapp_production"
BACKUP_DIR="/var/backups/postgres"
S3_BUCKET="s3://myapp-backups/postgres"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# 物理バックアップ（pg_basebackup）
pg_basebackup \
  -h localhost \
  -U replication_user \
  -D "${BACKUP_DIR}/base_${TIMESTAMP}" \
  -Ft -z -Xs -P

# 論理バックアップ（pg_dump）
pg_dump \
  -h localhost \
  -U backup_user \
  -Fc \
  -f "${BACKUP_DIR}/dump_${DB_NAME}_${TIMESTAMP}.custom" \
  "${DB_NAME}"

# S3 にアップロード（暗号化 + 別リージョン）
aws s3 cp \
  "${BACKUP_DIR}/dump_${DB_NAME}_${TIMESTAMP}.custom" \
  "${S3_BUCKET}/${TIMESTAMP}/" \
  --sse aws:kms \
  --storage-class STANDARD_IA

# クロスリージョンコピー
aws s3 cp \
  "${S3_BUCKET}/${TIMESTAMP}/" \
  "s3://myapp-backups-dr/postgres/${TIMESTAMP}/" \
  --source-region ap-northeast-1 \
  --region us-west-2 \
  --recursive

# ローカルの古いバックアップを削除
find "${BACKUP_DIR}" -name "dump_*" -mtime +${RETENTION_DAYS} -delete

# バックアップ検証
echo "Verifying backup integrity..."
pg_restore --list "${BACKUP_DIR}/dump_${DB_NAME}_${TIMESTAMP}.custom" > /dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "Backup verification: OK"
else
  echo "Backup verification: FAILED" >&2
  # アラート送信
  curl -X POST "${SLACK_WEBHOOK}" \
    -H 'Content-Type: application/json' \
    -d '{"text":"CRITICAL: Backup verification failed for '"${DB_NAME}"'"}'
  exit 1
fi
```

**WAL アーカイブ（継続的バックアップ）:**

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://myapp-wal-archive/%f --sse aws:kms'
archive_timeout = 300  # 5分ごとに強制アーカイブ
```

### Step 3: データベースレプリケーション

```yaml
# PostgreSQL ストリーミングレプリケーション（Kubernetes）
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-db
  namespace: production
spec:
  instances: 3  # プライマリ1 + レプリカ2
  imageName: ghcr.io/cloudnative-pg/postgresql:16.2

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      wal_level: "replica"
      synchronous_commit: "on"

  bootstrap:
    initdb:
      database: myapp
      owner: myapp

  storage:
    size: 100Gi
    storageClass: gp3

  backup:
    barmanObjectStore:
      destinationPath: s3://myapp-backups/cnpg/
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
    retentionPolicy: "30d"

  # 自動フェイルオーバー設定
  failoverDelay: 0
  switchoverDelay: 60

  # アフィニティ（レプリカを異なるノードに配置）
  affinity:
    topologyKey: kubernetes.io/hostname
```

### Step 4: マルチリージョンフェイルオーバー

```hcl
# Terraform: Route 53 ヘルスチェック + フェイルオーバー
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.example.com"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/healthz"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "primary-health-check"
  }
}

resource "aws_route53_record" "failover_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "failover_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }

  set_identifier = "secondary"
}
```

### Step 5: 復旧手順書（Runbook）

```markdown
# 障害復旧手順書

## 1. 障害検知
- [ ] アラート受信確認
- [ ] 影響範囲の特定（サービス、ユーザー数）
- [ ] インシデントチャネル作成（Slack #incident-YYYYMMDD）
- [ ] ステータスページ更新

## 2. 初期対応（15分以内）
- [ ] オンコール担当が応答
- [ ] 障害の種類を分類:
  - [ ] アプリケーション障害 -> Step 3A
  - [ ] データベース障害 -> Step 3B
  - [ ] インフラ障害 -> Step 3C
  - [ ] セキュリティインシデント -> セキュリティ手順書へ

## 3A. アプリケーション復旧
- [ ] 直近のデプロイを確認: `kubectl rollout history deployment/myapp`
- [ ] ロールバック: `kubectl rollout undo deployment/myapp`
- [ ] ヘルスチェック確認
- [ ] ログ確認: エラーの根本原因特定

## 3B. データベース復旧
- [ ] レプリカの状態確認
- [ ] 自動フェイルオーバーの確認
- [ ] 手動フェイルオーバー（必要時）:
      `kubectl cnpg promote myapp-db myapp-db-2`
- [ ] アプリケーションの接続先更新
- [ ] データ整合性チェック

## 3C. インフラ復旧（リージョン障害）
- [ ] DNS フェイルオーバー確認
- [ ] セカンダリリージョンの状態確認
- [ ] 手動 DNS 切り替え（自動でない場合）
- [ ] データ同期状態の確認
- [ ] セカンダリでの動作確認

## 4. 事後対応
- [ ] ポストモーテム作成（48時間以内）
- [ ] 根本原因分析
- [ ] 再発防止策の策定
- [ ] 改善アクションの追跡
```

### Step 6: カオスエンジニアリング

```yaml
# Chaos Mesh（Kubernetes）
# Pod Kill 実験
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 24h"
---
# ネットワーク遅延実験
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-test
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "25"
  duration: "5m"
  scheduler:
    cron: "@every 48h"
---
# ディスク障害実験
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-fault-test
  namespace: chaos-testing
spec:
  action: fault
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  volumePath: /data
  path: "*"
  errno: 5  # EIO
  percent: 50
  duration: "2m"
```

**カオス実験の安全ルール:**

```yaml
# 実験の段階的導入
phases:
  1_game_day:
    description: "チーム全員が見守る中で手動実行"
    scope: staging
    notification: true
    abort_conditions:
      - error_rate > 10%
      - p99_latency > 5s

  2_automated_staging:
    description: "ステージングで自動実行"
    scope: staging
    schedule: "weekdays 10:00-16:00"
    abort_conditions:
      - error_rate > 5%

  3_automated_production:
    description: "本番で自動実行（小規模）"
    scope: production
    schedule: "weekdays 10:00-14:00"
    blast_radius: "1 pod only"
    abort_conditions:
      - error_rate > 1%
      - slo_burn_rate > 5x
```

### Step 7: 復旧テスト自動化

```yaml
# .github/workflows/dr-test.yml
name: Disaster Recovery Test

on:
  schedule:
    - cron: '0 2 1 * *'  # 毎月1日の午前2時
  workflow_dispatch:

jobs:
  backup-restore-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4

      - name: Download latest backup
        run: |
          aws s3 cp \
            "$(aws s3 ls s3://myapp-backups/postgres/ --recursive | sort | tail -1 | awk '{print "s3://myapp-backups/" $4}')" \
            /tmp/latest-backup.custom

      - name: Restore backup
        run: |
          pg_restore \
            -h localhost -U postgres \
            -d postgres \
            --create \
            /tmp/latest-backup.custom

      - name: Verify data integrity
        run: |
          psql -h localhost -U postgres -d myapp_production -c "
            SELECT
              (SELECT count(*) FROM users) as user_count,
              (SELECT count(*) FROM orders) as order_count,
              (SELECT max(created_at) FROM orders) as latest_order
          " -t

      - name: Report result
        if: always()
        run: |
          STATUS="${{ job.status }}"
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\":\"DR Test Result: ${STATUS}\"}"
```

## ポストモーテム テンプレート

```markdown
# ポストモーテム: [インシデントタイトル]
**日時**: YYYY-MM-DD HH:MM - HH:MM (JST)
**影響時間**: X時間Y分
**影響範囲**: ユーザー数、サービス名
**重要度**: SEV1 / SEV2 / SEV3

## タイムライン
| 時刻 | イベント |
|------|---------|
| HH:MM | 障害検知（アラート/ユーザー報告） |
| HH:MM | インシデント対応開始 |
| HH:MM | 原因特定 |
| HH:MM | 対処実施 |
| HH:MM | サービス復旧確認 |

## 根本原因
[技術的な根本原因の説明]

## 再発防止策
| アクション | 担当 | 期限 | 優先度 |
|-----------|------|------|--------|
| [具体的なアクション] | @name | YYYY-MM-DD | P1 |
```

## ベストプラクティス

1. **定期テスト**: 復旧手順を月1回以上テストする（テストしていない DR 計画は計画ではない）
2. **自動化**: フェイルオーバーは可能な限り自動化する
3. **3-2-1 ルール**: バックアップの多重化を徹底
4. **バックアップ検証**: バックアップの取得だけでなく、リストア検証も自動化
5. **ポストモーテム文化**: 非難しない振り返りで改善を積み重ねる
6. **カオス実験**: 障害が起きてから慌てるのではなく、計画的に耐障害性を検証
7. **ランブック整備**: 手順書は常に最新に保ち、新メンバーでも実行可能にする
8. **SLO 連携**: DR の RTO/RPO は SLO と整合性を取る
