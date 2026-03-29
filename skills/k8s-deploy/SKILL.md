---
name: k8s-deploy
description: Kubernetesデプロイ戦略。Deployment、Service、Ingress、HPA、ローリングアップデート、Blue-Green
category: DevOps
command: /k8s-deploy
version: 1.0.0
tags: [kubernetes, k8s, deploy, hpa, ingress]
---

# Kubernetes デプロイ戦略スキル

## 概要

Kubernetesへのアプリケーションデプロイを安全かつ効率的に行うためのスキル。
Deployment、Service、Ingress、HPA の設定からローリングアップデート、Blue-Green デプロイまでカバーする。

## Steps

### Step 1: Namespace とリソース設計

アプリケーションごとに Namespace を分離し、ResourceQuota で制限する。

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    app: myapp
    env: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp-production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

### Step 2: Deployment 定義

プロダクション品質の Deployment マニフェスト。

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-production
  labels:
    app: myapp
    version: v1.2.0
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        version: v1.2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: myapp-sa
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [myapp]
                topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: myregistry.io/myapp:v1.2.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          env:
            - name: NODE_ENV
              value: production
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-host
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
      imagePullSecrets:
        - name: registry-credentials
```

### Step 3: Service と Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: myapp-production
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  type: ClusterIP
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp-production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-svc
                port:
                  name: http
```

### Step 4: HPA（水平 Pod オートスケーリング）

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 15
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

### Step 5: Blue-Green デプロイ

```yaml
# blue-green: 新バージョンを green としてデプロイ
# deployment-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: myapp-production
  labels:
    app: myapp
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
    spec:
      containers:
        - name: myapp
          image: myregistry.io/myapp:v1.3.0
          # ... (同じリソース設定)
---
# Service の切り替え（blue -> green）
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: myapp-production
spec:
  selector:
    app: myapp
    slot: green  # blue から green に変更
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

**Blue-Green 切り替え手順:**

```bash
# 1. Green デプロイを作成
kubectl apply -f deployment-green.yaml

# 2. Green の準備完了を確認
kubectl rollout status deployment/myapp-green -n myapp-production

# 3. Green でスモークテスト
kubectl port-forward svc/myapp-green-test 8081:80 -n myapp-production

# 4. Service を Green に切り替え
kubectl patch svc myapp-svc -n myapp-production \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# 5. 問題があればロールバック
kubectl patch svc myapp-svc -n myapp-production \
  -p '{"spec":{"selector":{"slot":"blue"}}}'

# 6. 安定後に Blue を削除
kubectl delete deployment myapp-blue -n myapp-production
```

### Step 6: PodDisruptionBudget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: myapp-production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

## ベストプラクティス

1. **イメージタグ**: `latest` は使わない。必ずセマンティックバージョンまたは Git SHA を使う
2. **リソース制限**: requests と limits を必ず設定する。OOM Kill を防ぐ
3. **ヘルスチェック**: liveness / readiness / startup の3つのプローブを適切に設定
4. **Graceful Shutdown**: preStop フックで接続のドレインを待つ
5. **Pod Anti-Affinity**: 同一ノードへの集中を避け、可用性を高める
6. **RevisionHistoryLimit**: ロールバック用に履歴を保持する（デフォルト10、5程度が推奨）
7. **maxUnavailable: 0**: ゼロダウンタイムデプロイのために必須
8. **シークレット管理**: Secret は External Secrets Operator か Sealed Secrets で管理

## ロールバック手順

```bash
# デプロイ履歴確認
kubectl rollout history deployment/myapp -n myapp-production

# 直前にロールバック
kubectl rollout undo deployment/myapp -n myapp-production

# 特定リビジョンにロールバック
kubectl rollout undo deployment/myapp --to-revision=3 -n myapp-production

# ロールバック状況確認
kubectl rollout status deployment/myapp -n myapp-production
```

## トラブルシューティング

```bash
# Pod が起動しない
kubectl describe pod <pod-name> -n myapp-production
kubectl logs <pod-name> -n myapp-production --previous

# リソース使用量確認
kubectl top pods -n myapp-production

# イベント確認
kubectl get events -n myapp-production --sort-by='.lastTimestamp'

# HPA 状態確認
kubectl describe hpa myapp-hpa -n myapp-production
```
