---
name: dns-ssl-config
description: DNS・SSL/TLS設定。レコード設計、Let's Encrypt自動更新、CDN連携、DNSSEC
category: DevOps
command: /dns-ssl-config
version: 1.0.0
tags: [dns, ssl, tls, lets-encrypt, certificate]
---

# DNS・SSL/TLS 設定スキル

## 概要

DNS レコード設計と SSL/TLS 証明書管理のスキル。Let's Encrypt 自動更新、
CDN 連携、DNSSEC、メール認証（SPF/DKIM/DMARC）をカバーする。

## Steps

### Step 1: DNS レコード設計

```
# 基本レコード設計
example.com.           300  IN  A      203.0.113.10
example.com.           300  IN  AAAA   2001:db8::10
www.example.com.       300  IN  CNAME  example.com.

# サブドメイン
api.example.com.       300  IN  A      203.0.113.11
staging.example.com.   300  IN  A      203.0.113.20
admin.example.com.     300  IN  CNAME  example.com.

# ワイルドカード（開発環境向け）
*.dev.example.com.     300  IN  CNAME  dev-lb.example.com.
```

**レコードタイプ早見表:**

| レコード | 用途 | 例 |
|---------|------|-----|
| A | IPv4 アドレス | `example.com -> 203.0.113.10` |
| AAAA | IPv6 アドレス | `example.com -> 2001:db8::10` |
| CNAME | エイリアス | `www -> example.com` |
| MX | メールサーバー | `10 mail.example.com` |
| TXT | テキスト情報 | SPF, DKIM, ドメイン検証 |
| NS | ネームサーバー | `ns1.example.com` |
| CAA | 証明書発行制限 | `0 issue "letsencrypt.org"` |
| SRV | サービス検出 | `_http._tcp 443 server.example.com` |

### Step 2: TTL 戦略

```
# TTL ガイドライン
# 安定した本番レコード: 3600（1時間）〜 86400（1日）
example.com.           3600  IN  A      203.0.113.10

# 変更が予想されるレコード: 300（5分）
api.example.com.       300   IN  A      203.0.113.11

# フェイルオーバー用: 60（1分）
primary.example.com.   60    IN  A      203.0.113.10

# 移行前に TTL を下げておく手順:
# 1. 移行の24時間前に TTL を 60 に下げる
# 2. 旧 TTL 分の時間が経過したら移行実行
# 3. 新しいレコードに変更
# 4. 安定後に TTL を元に戻す
```

### Step 3: Let's Encrypt 証明書（Certbot）

```bash
# 初回発行（Nginx プラグイン）
certbot --nginx \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email

# DNS チャレンジ（ワイルドカード証明書）
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.example.com" \
  -d example.com \
  --email admin@example.com \
  --agree-tos

# 自動更新の設定
# /etc/cron.d/certbot
0 3 * * * root certbot renew --quiet --deploy-hook "systemctl reload nginx"
```

**Cloudflare DNS 認証情報:**

```ini
# /etc/letsencrypt/cloudflare.ini（chmod 600）
dns_cloudflare_api_token = YOUR_API_TOKEN_HERE
```

### Step 4: Docker での自動証明書管理

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot-conf:/etc/letsencrypt:ro
      - certbot-www:/var/www/certbot:ro
    restart: unless-stopped

  certbot:
    image: certbot/certbot
    volumes:
      - certbot-conf:/etc/letsencrypt
      - certbot-www:/var/www/certbot
    # 12時間ごとに更新チェック
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew --webroot -w /var/www/certbot; sleep 12h & wait $${!}; done;'"

volumes:
  certbot-conf:
  certbot-www:
```

**初回証明書取得スクリプト:**

```bash
#!/bin/bash
# init-letsencrypt.sh

DOMAINS=(example.com www.example.com api.example.com)
EMAIL="admin@example.com"
STAGING=0  # 1 でステージング環境（テスト用）

# Nginx を一時設定で起動
docker compose up -d nginx

# 証明書取得
docker compose run --rm certbot certonly \
  --webroot \
  -w /var/www/certbot \
  ${DOMAINS[@]/#/-d } \
  --email "$EMAIL" \
  --agree-tos \
  --no-eff-email \
  $([ "$STAGING" = "1" ] && echo "--staging")

# Nginx リロード
docker compose exec nginx nginx -s reload
```

### Step 5: Kubernetes での証明書管理（cert-manager）

```yaml
# cert-manager のインストール
# helm install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true

# ClusterIssuer（Let's Encrypt 本番）
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      # HTTP-01 チャレンジ
      - http01:
          ingress:
            class: nginx
      # DNS-01 チャレンジ（ワイルドカード用）
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - "example.com"
---
# Certificate リソース
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: production
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - "*.example.com"
  renewBefore: 720h  # 30日前に更新
---
# Ingress で自動証明書取得
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - api.example.com
      secretName: example-com-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

### Step 6: CDN 連携（Cloudflare）

```
# Cloudflare Proxy 有効時の DNS 設定
# Cloudflare ダッシュボード上でオレンジクラウド（Proxied）にする

example.com        A     203.0.113.10  (Proxied)
www.example.com    CNAME example.com   (Proxied)
api.example.com    A     203.0.113.11  (Proxied)

# プロキシを通さないレコード（DNS Only）
mail.example.com   A     203.0.113.30  (DNS only)
ssh.example.com    A     203.0.113.10  (DNS only)
```

**Cloudflare SSL モード:**

| モード | 説明 | 推奨 |
|--------|------|------|
| Off | SSL なし | 非推奨 |
| Flexible | CF-ユーザー間のみ暗号化 | 非推奨 |
| Full | オリジンも自己署名証明書で暗号化 | 開発用 |
| Full (Strict) | オリジンも有効な証明書必須 | 本番推奨 |

```
# Cloudflare Origin Certificate（15年有効）
# CF ダッシュボード > SSL/TLS > Origin Server > Create Certificate
# Nginx に配置:
ssl_certificate     /etc/ssl/cloudflare/origin.pem;
ssl_certificate_key /etc/ssl/cloudflare/origin-key.pem;
```

### Step 7: メール認証（SPF / DKIM / DMARC）

```
# SPF（送信元 IP 認証）
example.com.  TXT  "v=spf1 include:_spf.google.com include:amazonses.com ~all"

# DKIM（電子署名）
google._domainkey.example.com.  TXT  "v=DKIM1; k=rsa; p=MIIBIjAN..."

# DMARC（ポリシー）
_dmarc.example.com.  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-forensic@example.com; pct=100; adkim=s; aspf=s"
```

**DMARC ポリシーの段階的導入:**

```
# Phase 1: 監視のみ（2-4週間）
_dmarc.example.com.  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com"

# Phase 2: 隔離（2-4週間）
_dmarc.example.com.  TXT  "v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc@example.com"

# Phase 3: 拒否
_dmarc.example.com.  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

### Step 8: CAA レコードと DNSSEC

```
# CAA レコード（証明書発行元の制限）
example.com.  CAA  0 issue "letsencrypt.org"
example.com.  CAA  0 issue "amazonaws.com"
example.com.  CAA  0 issuewild "letsencrypt.org"
example.com.  CAA  0 iodef "mailto:security@example.com"
```

**DNSSEC:**

```
# DNSSEC 有効化（レジストラ側で DS レコードを登録）
# Cloudflare の場合: ダッシュボード > DNS > DNSSEC > Enable

# DS レコード例（レジストラに登録）
# example.com. DS 12345 13 2 49FD46E6C4B45C55D4AC49...
```

## Terraform での DNS 管理

```hcl
# Cloudflare DNS
resource "cloudflare_record" "root" {
  zone_id = var.cloudflare_zone_id
  name    = "@"
  content = "203.0.113.10"
  type    = "A"
  proxied = true
  ttl     = 1  # auto (proxied)
}

resource "cloudflare_record" "www" {
  zone_id = var.cloudflare_zone_id
  name    = "www"
  content = "example.com"
  type    = "CNAME"
  proxied = true
  ttl     = 1
}

resource "cloudflare_record" "api" {
  zone_id = var.cloudflare_zone_id
  name    = "api"
  content = "203.0.113.11"
  type    = "A"
  proxied = true
  ttl     = 1
}

resource "cloudflare_record" "spf" {
  zone_id = var.cloudflare_zone_id
  name    = "@"
  content = "v=spf1 include:_spf.google.com ~all"
  type    = "TXT"
  ttl     = 3600
}
```

## ベストプラクティス

1. **HTTPS 必須**: 全サイトで HTTPS を強制（HSTS 含む）
2. **CAA レコード**: 証明書発行元を制限し、不正発行を防止
3. **自動更新**: 証明書の手動更新は事故のもと。必ず自動化する
4. **TTL 設計**: 平常時は長め、移行前に短くする
5. **DNSSEC**: 可能であれば有効化し、DNS 応答の改ざんを防止
6. **メール認証**: SPF + DKIM + DMARC を段階的に導入
7. **監視**: 証明書の有効期限を監視し、更新失敗をアラート
8. **CDN 活用**: Cloudflare 等で DDoS 防御とパフォーマンス改善

## 証明書監視

```bash
# 証明書の有効期限確認
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# DNS レコード確認
dig example.com A +short
dig example.com MX +short
dig _dmarc.example.com TXT +short

# SSL Labs テスト
# https://www.ssllabs.com/ssltest/analyze.html?d=example.com

# Prometheus による証明書監視
# blackbox_exporter の probe_ssl_earliest_cert_expiry メトリクス
```
