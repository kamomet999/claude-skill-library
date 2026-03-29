---
name: nginx-config
description: Nginx設定最適化。リバースプロキシ、SSL終端、レート制限、gzip、セキュリティヘッダ
category: DevOps
command: /nginx-config
version: 1.0.0
tags: [nginx, reverse-proxy, ssl, rate-limit, config]
---

# Nginx 設定最適化スキル

## 概要

Nginx のプロダクション向け設定スキル。リバースプロキシ、SSL/TLS 終端、レート制限、
圧縮、セキュリティヘッダの設定をカバーする。

## Steps

### Step 1: 基本構成（nginx.conf）

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
pid /run/nginx.pid;

error_log /var/log/nginx/error.log warn;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    # 基本設定
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    charset       utf-8;
    server_tokens off;  # バージョン情報を隠す

    # ログフォーマット（JSON 構造化ログ）
    log_format json_combined escape=json
      '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request_method":"$request_method",'
        '"request_uri":"$request_uri",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_user_agent":"$http_user_agent",'
        '"http_referer":"$http_referer",'
        '"request_id":"$request_id"'
      '}';

    access_log /var/log/nginx/access.log json_combined;

    # パフォーマンス
    sendfile    on;
    tcp_nopush  on;
    tcp_nodelay on;

    # タイムアウト
    keepalive_timeout   65;
    keepalive_requests  1000;
    client_body_timeout 30;
    client_header_timeout 30;
    send_timeout 30;

    # バッファ
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    client_max_body_size 50m;
    large_client_header_buffers 4 8k;

    # gzip 圧縮
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_types
        application/atom+xml
        application/geo+json
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rdf+xml
        application/rss+xml
        application/xhtml+xml
        application/xml
        font/eot
        font/otf
        font/ttf
        image/svg+xml
        text/css
        text/javascript
        text/plain
        text/xml;

    # Brotli（対応版のみ）
    # brotli on;
    # brotli_comp_level 4;
    # brotli_types application/json application/javascript text/css text/plain;

    # レート制限ゾーン
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=3r/m;
    limit_req_status 429;

    # 接続数制限
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    # アップストリーム
    upstream app_backend {
        least_conn;
        server 127.0.0.1:3000 weight=5 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:3001 weight=5 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # conf.d 読み込み
    include /etc/nginx/conf.d/*.conf;
}
```

### Step 2: SSL/TLS 設定

```nginx
# /etc/nginx/conf.d/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

# TLS 1.3 優先の暗号スイート
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

# SSL セッションキャッシュ
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# DH パラメータ（TLS 1.2 用）
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
```

### Step 3: セキュリティヘッダ

```nginx
# /etc/nginx/conf.d/security-headers.conf

# クリックジャッキング防止
add_header X-Frame-Options "SAMEORIGIN" always;

# XSS フィルタ
add_header X-Content-Type-Options "nosniff" always;

# リファラポリシー
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# HSTS（Strict Transport Security）
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# Permissions Policy
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(self), payment=()" always;

# Content Security Policy
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.example.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://api.example.com; frame-ancestors 'self'" always;

# Cross-Origin ポリシー
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
```

### Step 4: リバースプロキシ（本番サイト設定）

```nginx
# /etc/nginx/conf.d/myapp.conf

# HTTP -> HTTPS リダイレクト
server {
    listen 80;
    listen [::]:80;
    server_name myapp.example.com;

    # ACME チャレンジ用
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS サーバー
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.example.com;

    # SSL 証明書
    ssl_certificate     /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;

    # セキュリティ設定読み込み
    include /etc/nginx/conf.d/ssl-params.conf;
    include /etc/nginx/conf.d/security-headers.conf;

    # リクエスト ID（トレーシング用）
    add_header X-Request-ID $request_id always;

    # API エンドポイント
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_conn addr 50;

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header Connection "";

        # タイムアウト
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # バッファリング
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # ログイン（厳格なレート制限）
    location /api/auth/login {
        limit_req zone=login burst=5 nodelay;

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400s;
    }

    # 静的ファイル（Next.js ビルド出力など）
    location /_next/static/ {
        alias /var/www/myapp/.next/static/;
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # 静的アセット
    location /static/ {
        alias /var/www/myapp/public/static/;
        expires 30d;
        add_header Cache-Control "public, no-transform";
        access_log off;

        # 画像最適化
        location ~* \.(jpg|jpeg|png|gif|ico|webp|avif)$ {
            expires 90d;
        }
    }

    # フロントエンド（SPA フォールバック）
    location / {
        limit_req zone=general burst=30 nodelay;

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ヘルスチェック（ロードバランサー用）
    location /nginx-health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # セキュリティ: 隠しファイルへのアクセス拒否
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # エラーページ
    error_page 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
        internal;
    }
}
```

### Step 5: キャッシュ設定

```nginx
# プロキシキャッシュ設定
proxy_cache_path /var/cache/nginx levels=1:2
    keys_zone=app_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;

server {
    # ...

    # キャッシュ対象 API
    location /api/public/ {
        proxy_pass http://app_backend;
        proxy_cache app_cache;
        proxy_cache_valid 200 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;

        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Step 6: 設定テストとリロード

```bash
# 構文テスト
nginx -t

# 設定リロード（ダウンタイムなし）
nginx -s reload

# 詳細テスト
nginx -T  # 全設定を表示

# パフォーマンステスト
ab -n 10000 -c 100 https://myapp.example.com/
wrk -t4 -c100 -d30s https://myapp.example.com/
```

## ベストプラクティス

1. **server_tokens off**: Nginx バージョン情報を隠す
2. **TLS 1.2+**: TLS 1.0/1.1 は無効化
3. **レート制限の階層化**: エンドポイント種別に応じた制限を設定
4. **構造化ログ**: JSON 形式でログを出力し、集約ツールと連携
5. **keepalive**: upstream への keepalive 接続でオーバーヘッド削減
6. **proxy_buffering**: レスポンスをバッファリングしてバックエンド解放を早める
7. **静的ファイルの分離**: Nginx から直接配信し、バックエンドの負荷を軽減
8. **Cache-Control**: immutable な静的アセットには長期キャッシュを設定

## トラブルシューティング

```bash
# アクセスログ確認
tail -f /var/log/nginx/access.log | jq .

# エラーログ確認
tail -f /var/log/nginx/error.log

# 接続数確認
ss -s
ss -tlnp | grep nginx

# ワーカープロセス確認
ps aux | grep nginx

# レート制限のテスト
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" https://myapp.example.com/api/test
done
```

## Docker での運用

```dockerfile
FROM nginx:1.25-alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

RUN nginx -t

EXPOSE 80 443

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost/nginx-health || exit 1
```

```yaml
# docker-compose.yml
services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    depends_on:
      - app
    restart: unless-stopped

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```
