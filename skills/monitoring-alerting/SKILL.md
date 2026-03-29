---
name: monitoring-alerting
description: 監視・アラート設計。Prometheus、Grafana、Datadog、SLI/SLO/SLA、アラート疲れ防止
category: DevOps
command: /monitoring-alerting
version: 1.0.0
tags: [monitoring, prometheus, grafana, slo, alerting]
---

# 監視・アラート設計スキル

## 概要

プロダクション環境の監視とアラート設計スキル。
Prometheus/Grafana による監視基盤構築、SLI/SLO 定義、アラート疲れを防ぐ戦略をカバーする。

## Steps

### Step 1: 監視の4つのゴールデンシグナル

Google SRE が提唱する4つのシグナルを基本指標とする。

| シグナル | 説明 | 具体例 |
|---------|------|--------|
| Latency | リクエストの処理時間 | p50, p95, p99 レスポンスタイム |
| Traffic | システムへのリクエスト量 | RPS（Requests Per Second） |
| Errors | 失敗したリクエストの割合 | 5xx エラーレート |
| Saturation | リソースの飽和度 | CPU/メモリ/ディスク使用率 |

### Step 2: Prometheus 設定

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # アプリケーションメトリクス
  - job_name: 'myapp'
    metrics_path: /metrics
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}:$1
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

  # Node メトリクス
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: replace
        target_label: __address__
        regex: (.+):(.+)
        replacement: ${1}:9100

  # PostgreSQL メトリクス
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Step 3: アプリケーション計装

```typescript
// metrics.ts - Prometheus クライアント設定
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client'

const register = new Registry()
collectDefaultMetrics({ register })

// HTTP リクエストカウンター
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'HTTP リクエスト総数',
  labelNames: ['method', 'path', 'status'],
  registers: [register],
})

// レスポンスタイム
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP リクエスト処理時間（秒）',
  labelNames: ['method', 'path', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register],
})

// アクティブ接続数
export const activeConnections = new Gauge({
  name: 'active_connections',
  help: '現在のアクティブ接続数',
  registers: [register],
})

// ビジネスメトリクス
export const ordersProcessed = new Counter({
  name: 'orders_processed_total',
  help: '処理済み注文数',
  labelNames: ['status', 'payment_method'],
  registers: [register],
})

// Express ミドルウェア
export function metricsMiddleware(req, res, next) {
  const end = httpRequestDuration.startTimer({
    method: req.method,
    path: req.route?.path || req.path,
  })

  activeConnections.inc()

  res.on('finish', () => {
    const labels = {
      method: req.method,
      path: req.route?.path || req.path,
      status: res.statusCode,
    }
    httpRequestsTotal.inc(labels)
    end({ status: res.statusCode })
    activeConnections.dec()
  })

  next()
}

// /metrics エンドポイント
export async function metricsHandler(req, res) {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
}
```

### Step 4: SLI/SLO 定義

```yaml
# slo.yml - SLO 定義
slos:
  - name: api-availability
    description: "API の可用性 SLO"
    sli:
      type: availability
      query: |
        sum(rate(http_requests_total{status!~"5.."}[5m]))
        /
        sum(rate(http_requests_total[5m]))
    objective: 99.9   # 99.9% 可用性
    window: 30d        # 30日ローリングウィンドウ
    # エラーバジェット: 30日 * 24h * 60min * 0.1% = 43.2分

  - name: api-latency
    description: "API レスポンスタイム SLO"
    sli:
      type: latency
      query: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
        )
    objective:
      p95: 0.5    # 95パーセンタイルが500ms以下
      p99: 2.0    # 99パーセンタイルが2秒以下
    window: 30d
```

**エラーバジェット計算:**

| SLO | 月間ダウンタイム許容 | 日間ダウンタイム許容 |
|-----|-------------------|-------------------|
| 99.9% | 43.2分 | 1.44分 |
| 99.95% | 21.6分 | 43.2秒 |
| 99.99% | 4.32分 | 8.64秒 |

### Step 5: アラートルール設計

```yaml
# rules/alerts.yml
groups:
  - name: availability
    rules:
      # 高エラーレート（即時対応）
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "エラーレートが5%を超過"
          description: "過去5分間のエラーレート: {{ $value | humanizePercentage }}"
          runbook: "https://wiki.example.com/runbook/high-error-rate"

      # レイテンシ悪化
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 レイテンシが1秒を超過"
          description: "現在のp95: {{ $value }}秒"

  - name: resources
    rules:
      # メモリ使用率
      - alert: HighMemoryUsage
        expr: |
          container_memory_working_set_bytes
          /
          container_spec_memory_limit_bytes
          > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "メモリ使用率が85%を超過"

      # ディスク容量
      - alert: DiskSpaceRunningLow
        expr: |
          predict_linear(
            node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*60*60
          ) < 0
        labels:
          severity: critical
        annotations:
          summary: "24時間以内にディスク枯渇の予測"

  - name: slo
    rules:
      # エラーバジェット消費レート
      - alert: ErrorBudgetBurnRate
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            )
          ) / (1 - 0.999) > 14.4
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "エラーバジェットの急速な消費を検知"
          description: "現在のバーンレート: {{ $value }}x（月間バジェットの14.4倍以上）"
```

### Step 6: Alertmanager 設定

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    # Critical: Slack + PagerDuty
    - match:
        severity: critical
      receiver: 'critical-alerts'
      repeat_interval: 1h
      continue: true

    # Warning: Slack のみ
    - match:
        severity: warning
      receiver: 'warning-alerts'
      repeat_interval: 4h

    # Info: 日次ダイジェスト
    - match:
        severity: info
      receiver: 'info-digest'
      group_wait: 1h
      repeat_interval: 24h

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts-default'
        send_resolved: true

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    pagerduty_configs:
      - service_key: '<pagerduty-key>'
        severity: critical

  - name: 'warning-alerts'
    slack_configs:
      - channel: '#alerts-warning'
        send_resolved: true

  - name: 'info-digest'
    slack_configs:
      - channel: '#alerts-info'

# アラート抑制（メンテナンス時など）
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'namespace']
```

### Step 7: Grafana ダッシュボード

```json
{
  "dashboard": {
    "title": "Service Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "targets": [{
          "expr": "sum(rate(http_requests_total[5m])) by (status)",
          "legendFormat": "{{status}}"
        }]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
          "legendFormat": "Error Rate"
        }],
        "thresholds": {
          "steps": [
            {"value": 0, "color": "green"},
            {"value": 0.01, "color": "yellow"},
            {"value": 0.05, "color": "red"}
          ]
        }
      },
      {
        "title": "Latency (p50/p95/p99)",
        "type": "timeseries",
        "targets": [
          {"expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p50"},
          {"expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p95"},
          {"expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p99"}
        ]
      },
      {
        "title": "Error Budget Remaining",
        "type": "gauge",
        "targets": [{
          "expr": "1 - (sum(increase(http_requests_total{status=~\"5..\"}[30d])) / sum(increase(http_requests_total[30d]))) / 0.001"
        }],
        "thresholds": {
          "steps": [
            {"value": 0, "color": "red"},
            {"value": 0.25, "color": "yellow"},
            {"value": 0.5, "color": "green"}
          ]
        }
      }
    ]
  }
}
```

## アラート疲れ防止戦略

1. **重要度の3段階**: critical（即時対応）、warning（営業時間内対応）、info（記録のみ）
2. **for 句の活用**: 一時的なスパイクで発火しないよう `for: 5m` を設定
3. **group_by**: 関連アラートをグルーピングして通知数を削減
4. **inhibit_rules**: critical 発火時に同一原因の warning を抑制
5. **repeat_interval**: 同一アラートの再通知間隔を十分に空ける
6. **Runbook リンク**: アラートに対応手順書を必ず添付
7. **定期レビュー**: 月1回、発火しなかった/常に発火しているアラートを見直す
8. **エラーバジェット駆動**: 症状ベースのアラート（原因ベースではなく）

## ベストプラクティス

- **症状に基づくアラート**: CPU が高い → アラートではなく、レスポンスタイムが遅い → アラート
- **マルチウィンドウバーンレート**: 短期（5分）+ 長期（1時間）の両方で判定
- **ダッシュボードの階層化**: Overview → Service → Component → Debug の4階層
- **メトリクスの命名規則**: `<namespace>_<subsystem>_<name>_<unit>` （例: `myapp_http_requests_total`）
- **カーディナリティ管理**: ラベルの値が無制限に増えないよう注意（user_id をラベルにしない）
