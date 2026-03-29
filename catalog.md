# Skill Catalog

> skill-library が参照する辞書ファイル。スキル名・説明・タグのみ。本体は `skills/` ディレクトリ。
> **Total: 72 skills**

## バックエンド
| Skill | Description | Tags |
|-------|-------------|------|
| backend-patterns | Node.js/Express/Next.jsのバックエンドアーキテクチャパターン、API設計、DB最適化 | node, express, nextjs, api, backend |
| express-api | Express REST API設計パターン。ミドルウェア構成、エラーハンドリング、認証、レート制限 | node, express, api, rest, middleware |
| graphql-patterns | GraphQLスキーマ設計、N+1問題解決、DataLoader、サブスクリプション | graphql, api, schema, dataloader |
| queue-worker | Bull/BullMQジョブキュー設計、リトライ戦略、デッドレター、並行制御 | queue, bull, redis, async, worker |
| websocket-realtime | WebSocket/Socket.ioリアルタイム通信、ルーム管理、再接続、スケーリング | websocket, socket.io, realtime, pub-sub |
| auth-patterns | 認証・認可パターン集。JWT、OAuth2.0、RBAC、セッション管理、MFA、パスキー | auth, jwt, oauth, rbac, session, mfa |
| api-versioning | APIバージョニング戦略、後方互換性、マイグレーション、deprecation | api, versioning, migration, backward-compatibility |
| caching-strategy | キャッシュ戦略設計。Redis、CDN、Cache-Aside、Write-Through、TTL | cache, redis, cdn, performance, ttl |
| file-upload | ファイルアップロード設計。マルチパート、チャンク、S3直接アップロード、画像リサイズ | upload, s3, multipart, image, storage |
| python-fastapi | FastAPI設計パターン。依存性注入、Pydantic、非同期、ミドルウェア、OpenAPI | python, fastapi, pydantic, async, openapi |

## フロントエンド
| Skill | Description | Tags |
|-------|-------------|------|
| frontend-patterns | React/Next.jsのフロントエンド開発パターン、状態管理、パフォーマンス最適化 | react, nextjs, frontend, state, performance |
| react-performance | Reactパフォーマンス最適化。memo、useMemo、useCallback、仮想化、コード分割 | react, performance, memo, virtualization, bundle |
| tailwind-design | Tailwind CSSデザインシステム構築。カスタムテーマ、レスポンシブ、ダークモード | tailwind, css, design-system, responsive, dark-mode |
| form-validation | フォームバリデーション設計。React Hook Form、Zod、エラーUX、多段階フォーム | form, validation, zod, react-hook-form, ux |
| state-management | 状態管理パターン選択ガイド。Zustand、Jotai、Redux Toolkit、React Query | state, zustand, jotai, redux, react-query |
| nextjs-app-router | Next.js App Router完全ガイド。Server Components、Streaming、並列ルート、キャッシュ | nextjs, app-router, rsc, streaming, cache |
| animation-patterns | Webアニメーション設計。Framer Motion、CSS Transitions、スクロールアニメーション | animation, framer-motion, css, scroll, ux |
| accessibility-audit | アクセシビリティ監査。WCAG 2.1、ARIA、キーボードナビ、スクリーンリーダー | a11y, wcag, aria, keyboard, screen-reader |
| i18n-localization | 国際化・多言語対応。next-intl、ICUメッセージ、RTL、日付・通貨フォーマット | i18n, l10n, next-intl, translation, rtl |

## データベース
| Skill | Description | Tags |
|-------|-------------|------|
| postgres-patterns | PostgreSQLクエリ最適化、スキーマ設計、インデックス、Supabase準拠 | postgres, sql, supabase, index, schema |
| clickhouse-io | ClickHouseのクエリ最適化、分析パターン、データエンジニアリング | clickhouse, analytics, sql, data |
| drizzle-orm | Drizzle ORMスキーマ定義、マイグレーション、クエリビルダー、型安全 | drizzle, orm, typescript, migration, schema |
| supabase-rls | Supabase Row Level Security設計パターン、マルチテナント、パフォーマンス | supabase, rls, security, policy, multi-tenant |
| redis-patterns | Redisデータ構造活用。セッション、ランキング、Pub/Sub、Lua、Streams | redis, cache, pub-sub, session, streams |

## DevOps
| Skill | Description | Tags |
|-------|-------------|------|
| docker-optimize | Dockerイメージ最適化、ビルド高速化、マルチステージビルド | docker, container, optimization |
| k8s-deploy | Kubernetesデプロイ戦略。Deployment、Service、Ingress、HPA、Blue-Green | kubernetes, k8s, deploy, hpa, ingress |
| ci-cd-github-actions | GitHub Actions CI/CD設計。マトリックスビルド、キャッシュ、自動リリース | github-actions, ci, cd, automation, release |
| terraform-patterns | Terraform IaCパターン。モジュール設計、状態管理、環境分離、ドリフト検知 | terraform, iac, infrastructure, modules, state |
| monitoring-alerting | 監視・アラート設計。Prometheus、Grafana、SLI/SLO/SLA、アラート疲れ防止 | monitoring, prometheus, grafana, slo, alerting |
| nginx-config | Nginx設定最適化。リバースプロキシ、SSL終端、レート制限、セキュリティヘッダ | nginx, reverse-proxy, ssl, rate-limit, config |
| log-management | ログ管理設計。構造化ログ、ELK/Loki、ログレベル戦略、コスト管理 | logging, elk, loki, structured-log, observability |
| dns-ssl-config | DNS・SSL/TLS設定。レコード設計、Let's Encrypt自動更新、CDN連携 | dns, ssl, tls, lets-encrypt, certificate |
| disaster-recovery | 災害復旧計画。バックアップ戦略、RTO/RPO、フェイルオーバー、カオスエンジニアリング | disaster-recovery, backup, failover, rto, chaos |

## アーキテクチャ
| Skill | Description | Tags |
|-------|-------------|------|
| microservices-design | マイクロサービス設計。サービス分割、Saga、CQRS、イベント駆動 | microservices, saga, cqrs, event-driven, ddd |
| monorepo-management | モノレポ管理。Turborepo/Nx、ワークスペース、ビルドキャッシュ、変更検知 | monorepo, turborepo, nx, workspace, build-cache |
| error-handling-patterns | エラーハンドリング設計。Result型、カスタムエラー階層、リトライ、サーキットブレーカー | error-handling, result, retry, circuit-breaker, resilience |

## テスト
| Skill | Description | Tags |
|-------|-------------|------|
| tdd-workflow | テスト駆動開発ワークフロー、80%+カバレッジ | tdd, testing, coverage, workflow |
| golang-testing | Goテストパターン。テーブル駆動、サブテスト、ベンチマーク、ファジング | go, golang, testing, tdd, benchmark |
| playwright-testing | Playwright E2Eテスト設計。Page Object Model、フィクスチャ、並列実行 | playwright, e2e, testing, pom, visual |
| api-testing | APIテスト設計。Supertest、コントラクトテスト、負荷テスト、モック | api-testing, supertest, contract, load-test, mock |

## セキュリティ
| Skill | Description | Tags |
|-------|-------------|------|
| security-review | セキュリティチェックリスト。認証、入力検証、シークレット管理 | security, owasp, auth, validation |
| coding-standards | TypeScript/JavaScript/React/Node.jsの汎用コーディング規約 | typescript, javascript, react, standards |

## AI・ML
| Skill | Description | Tags |
|-------|-------------|------|
| prompt-engineering | プロンプトエンジニアリング。Chain-of-Thought、Few-Shot、システムプロンプト設計 | prompt, llm, chain-of-thought, few-shot, evaluation |
| rag-patterns | RAG設計パターン。チャンク戦略、ベクトルDB、リランキング、ハイブリッド検索 | rag, vector, embedding, retrieval, llm |

## Go言語
| Skill | Description | Tags |
|-------|-------------|------|
| golang-patterns | Go言語のイディオムパターン、堅牢なアプリケーション構築 | go, golang, patterns, idiom |

## X運用
| Skill | Description | Tags |
|-------|-------------|------|
| x-daily | X直近投稿の日次パフォーマンスレポート生成 | x, twitter, analytics, daily |
| x-image | X投稿用画像のプロンプト生成・仕様設計 | x, twitter, image, prompt |
| x-manager | X運用スキル統括マネージャー | x, twitter, manager, orchestration |
| x-trend | X競合アカウント・トレンド分析、市場動向レポート | x, twitter, trend, competitor |
| x-twitter-growth | Xグロースエンジン。オーディエンス構築、バイラル、アルゴリズム攻略 | x, twitter, growth, viral, audience |
| x-writing | X投稿用文章生成 | x, twitter, writing, content |

## リサーチ・分析
| Skill | Description | Tags |
|-------|-------------|------|
| multi-stage-research | 並列サブエージェント調査→統合→ファクトチェック→レポート | research, parallel, agent, report |
| news-insight | ニュースURL深掘りリサーチ、インサイト・実装プラン生成 | news, research, insight, url |
| survey-reader | Supabaseアンケート結果読み込み・マーケティング分析 | survey, supabase, marketing, analytics |

## エージェント・学習
| Skill | Description | Tags |
|-------|-------------|------|
| continuous-learning | セッションから再利用可能パターン自動抽出・スキル保存 | learning, pattern, auto-extract |
| continuous-learning-v2 | hooks観察→インスティンクト生成→スキル進化する学習システム | learning, instinct, hooks, evolution |
| iterative-retrieval | サブエージェントのコンテキスト問題を解決する段階的検索 | context, retrieval, subagent |
| strategic-compact | タスクフェーズ区切りでコンテキスト圧縮提案 | context, compression, phase |

## 開発ワークフロー
| Skill | Description | Tags |
|-------|-------------|------|
| eval-harness | Claude Codeセッション評価フレームワーク、EDD原則 | eval, quality, metrics |
| verification-loop | コード変更後の品質ゲート自動検証 | verification, ci, quality-gate |
| worker-assignment | タスクへの作業者自動割り当て、コスト見積もり | task, assignment, cost, planning |

## ユーティリティ
| Skill | Description | Tags |
|-------|-------------|------|
| creator-marketing-system | クリエイター向け対話型マーケティング戦略、事業計画書生成 | marketing, creator, strategy, plan |
| diary | 今日の日付で日記テンプレート作成 | diary, template, daily |
| pdf | PDF操作全般。読取、抽出、結合、分割、作成、暗号化、OCR | pdf, extract, merge, ocr |
| shell-scripting | シェルスクリプトベストプラクティス。Bash、エラー処理、並列実行 | bash, shell, script, automation, cli |
| project-guidelines-example | プロジェクト固有スキルのテンプレート例 | template, example, guidelines |
| skill-finder | 会話文脈から最適スキル推論・ランク付け提案 | skill, search, recommend, context |
| skill-library | GitHubライブラリのcatalog.md辞書検索、ローカルにないスキル発見 | skill, library, catalog, search |
| skill-pull | ライブラリからローカルへスキル取得 | skill, install, pull |
| skill-unload | 使い終わったスキルをローカルから削除、コンテキスト節約 | skill, cleanup, unload |
| skill-manager | スキル1軍/2軍/3軍/育成を管理するGM、週次昇降格 | skill, management, roster, gm |
| skill-evaluate | 5軸評価（KPI+360度+コンピテンシー+ポテンシャル+市場価値）で査定 | skill, evaluation, scoring, hr |
| skill-retro | セッション完了時にスキル有効性検証、改善提案レポート出力 | skill, retro, review, report |
