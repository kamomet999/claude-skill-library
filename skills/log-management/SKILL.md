---
name: log-management
description: ログ管理設計。構造化ログ、集約、ELK/Loki、ログレベル戦略、コスト管理
category: DevOps
command: /log-management
version: 1.0.0
tags: [logging, elk, loki, structured-log, observability]
---

# ログ管理設計スキル

## 概要

プロダクション環境のログ管理設計スキル。構造化ログの実装、集約パイプライン構築、
ELK Stack / Grafana Loki の設定、ログレベル戦略、コスト管理をカバーする。

## Steps

### Step 1: ログレベル戦略

| レベル | 用途 | 本番出力 | 例 |
|--------|------|---------|-----|
| FATAL | アプリケーション停止を要する致命的エラー | YES | DB接続不可、メモリ枯渇 |
| ERROR | 処理失敗だがアプリは継続 | YES | API呼び出し失敗、バリデーションエラー |
| WARN | 潜在的な問題、注意が必要 | YES | リトライ発生、レート制限接近 |
| INFO | 主要な業務イベント | YES | ユーザーログイン、注文処理完了 |
| DEBUG | 開発・調査用の詳細情報 | NO* | 変数値、SQL クエリ |
| TRACE | 非常に詳細なトレース | NO | 関数の出入り、全リクエストボディ |

*DEBUG は環境変数で動的に有効化可能にしておく。

### Step 2: 構造化ログの実装

```typescript
// logger.ts - 構造化ログライブラリ（pino ベース）
import pino from 'pino'

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level(label) {
      return { level: label }
    },
  },
  serializers: {
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
  },
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'password',
      'creditCard',
      'ssn',
      '*.token',
      '*.secret',
    ],
    censor: '[REDACTED]',
  },
})

export { logger }

// 使用例
import { logger } from './logger'

// コンテキスト付きロガー
const orderLogger = logger.child({
  service: 'order-service',
  version: process.env.APP_VERSION,
})

// 業務ログ
orderLogger.info(
  { orderId: 'ORD-12345', userId: 'USR-67890', amount: 5000 },
  'Order placed successfully'
)

// エラーログ
orderLogger.error(
  { err: error, orderId: 'ORD-12345', operation: 'payment' },
  'Payment processing failed'
)
```

**出力例（JSON）:**

```json
{
  "level": "info",
  "time": "2025-03-15T10:30:00.000Z",
  "service": "order-service",
  "version": "1.2.0",
  "orderId": "ORD-12345",
  "userId": "USR-67890",
  "amount": 5000,
  "msg": "Order placed successfully"
}
```

### Step 3: リクエスト追跡ミドルウェア

```typescript
// request-logger.ts
import { randomUUID } from 'crypto'
import { logger } from './logger'

export function requestLogger(req, res, next) {
  const requestId = req.headers['x-request-id'] || randomUUID()
  const startTime = Date.now()

  // リクエストごとの子ロガー
  req.log = logger.child({
    requestId,
    method: req.method,
    path: req.path,
    userAgent: req.headers['user-agent'],
    ip: req.ip,
  })

  // レスポンスヘッダにリクエスト ID を付与
  res.setHeader('X-Request-ID', requestId)

  // リクエスト開始ログ
  req.log.info('Request started')

  // レスポンス完了時
  res.on('finish', () => {
    const duration = Date.now() - startTime

    const logData = {
      statusCode: res.statusCode,
      duration,
      contentLength: res.getHeader('content-length'),
    }

    if (res.statusCode >= 500) {
      req.log.error(logData, 'Request failed')
    } else if (res.statusCode >= 400) {
      req.log.warn(logData, 'Request client error')
    } else {
      req.log.info(logData, 'Request completed')
    }
  })

  next()
}
```

### Step 4: ELK Stack 構成

```yaml
# docker-compose-elk.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    ulimits:
      memlock:
        soft: -1
        hard: -1

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.12.0
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash

volumes:
  es-data:
```

```yaml
# logstash/pipeline/main.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # JSON パース
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # タイムスタンプ正規化
  date {
    match => ["time", "ISO8601"]
    target => "@timestamp"
  }

  # GeoIP（IP アドレスから位置情報）
  if [ip] {
    geoip {
      source => "ip"
      target => "geoip"
    }
  }

  # 個人情報マスキング
  mutate {
    gsub => [
      "message", "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", "[EMAIL_REDACTED]",
      "message", "\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b", "[CARD_REDACTED]"
    ]
  }

  # 不要フィールド削除
  mutate {
    remove_field => ["agent", "ecs", "host"]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    user => "elastic"
    password => "${ELASTIC_PASSWORD}"
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"
  }
}
```

### Step 5: Grafana Loki（軽量代替）

```yaml
# loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d
  max_query_series: 500
  max_query_parallelism: 2

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h

# S3 バックエンド（本番用）
# common:
#   storage:
#     s3:
#       s3: s3://ap-northeast-1/myapp-logs
#       bucketnames: myapp-logs
```

```yaml
# promtail-config.yml（ログ収集エージェント）
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Docker コンテナログ
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: container
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: stream
    pipeline_stages:
      # JSON ログのパース
      - json:
          expressions:
            level: level
            service: service
            msg: msg
            requestId: requestId
      - labels:
          level:
          service:
      - timestamp:
          source: time
          format: RFC3339

  # システムログ
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          __path__: /var/log/syslog
```

### Step 6: ログクエリ例

**LogQL（Loki）:**

```logql
# エラーログを検索
{service="order-service"} |= "error" | json | level="error"

# 特定リクエスト ID を追跡
{service=~".*"} | json | requestId="abc-123-def"

# レスポンスタイムが1秒超のリクエスト
{service="api-gateway"} | json | duration > 1000

# エラーレートの計算（5分間）
sum(rate({service="order-service"} | json | level="error" [5m]))
/
sum(rate({service="order-service"} [5m]))

# 上位エラーメッセージ
topk(10, sum by (msg) (count_over_time(
  {service="order-service"} | json | level="error" [1h]
)))
```

**KQL（Kibana）:**

```
# エラー検索
level: "error" AND service: "order-service"

# 時間範囲 + ステータスコード
statusCode >= 500 AND duration > 1000

# 特定ユーザーの操作追跡
userId: "USR-67890" AND NOT level: "debug"
```

### Step 7: コスト管理

```yaml
# ILM（Index Lifecycle Management）- Elasticsearch
# ilm-policy.json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**コスト削減戦略:**

| 戦略 | 効果 | 実装方法 |
|------|------|---------|
| ログレベルフィルタ | 40-60% 削減 | 本番は INFO 以上のみ |
| サンプリング | 30-50% 削減 | 成功リクエストを10%のみ保存 |
| フィールド削除 | 10-20% 削減 | 不要フィールドを除外 |
| 圧縮 | 60-80% 削減 | gzip/zstd 圧縮 |
| 保持期間 | 線形削減 | 90日 -> 30日 |
| Cold/Frozen ストレージ | 50-70% 削減 | S3 Glacier へ移行 |

### サンプリング実装

```typescript
// sampling-logger.ts
import { logger } from './logger'

const SAMPLING_RATE = parseFloat(process.env.LOG_SAMPLING_RATE || '1.0')

export function sampledLog(level: string, data: object, message: string) {
  // エラー以上は常に記録
  if (level === 'error' || level === 'fatal') {
    logger[level](data, message)
    return
  }

  // それ以外はサンプリング
  if (Math.random() < SAMPLING_RATE) {
    logger[level]({ ...data, sampled: true, samplingRate: SAMPLING_RATE }, message)
  }
}
```

## ベストプラクティス

1. **構造化ログ必須**: テキストログではなく JSON 構造化ログを使う
2. **コンテキスト伝播**: requestId をリクエストチェーン全体で引き回す
3. **機密情報のマスキング**: パスワード、トークン、カード番号を redact
4. **ログレベルの動的変更**: 環境変数で再デプロイなしにレベル変更可能に
5. **リテンション設計**: 法的要件とコストのバランスを取る
6. **アラート連携**: ERROR/FATAL ログを監視システムに連携
7. **サンプリング**: 高トラフィック環境では成功ログをサンプリング
8. **ラベルのカーディナリティ**: Loki ではラベル値が多すぎるとパフォーマンス劣化

## Loki vs ELK 比較

| 項目 | Grafana Loki | ELK Stack |
|------|-------------|-----------|
| コスト | 低（インデックスなし） | 高（全文インデックス） |
| 検索速度 | ラベル検索は高速、全文は遅い | 全文検索が高速 |
| ストレージ | S3/GCS に直接保存可 | 専用ストレージが必要 |
| 運用コスト | 低（シンプル） | 高（JVM チューニング等） |
| 適合シナリオ | Kubernetes + Grafana 環境 | 全文検索が必須の場合 |
| 推奨規模 | 小〜中規模 | 中〜大規模 |
