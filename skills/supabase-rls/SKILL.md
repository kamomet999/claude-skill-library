---
name: supabase-rls
description: Supabase Row Level Security設計パターン。ポリシー設計、マルチテナント、パフォーマンス最適化
category: データベース
command: /rls
version: 1.0.0
tags:
  - supabase
  - rls
  - security
  - policy
  - multi-tenant
---

# Supabase Row Level Security Patterns

SupabaseのRLSを使った行レベルセキュリティ設計パターン集。

## When to Activate

- Supabaseプロジェクトでテーブルのアクセス制御を設計するとき
- マルチテナントアプリケーションを構築するとき
- RLSポリシーのパフォーマンスを改善するとき
- 既存RLSポリシーのセキュリティレビューを行うとき

## Steps

### Step 1: RLS有効化（必須）

```sql
-- RLSを有効にしないとすべてのデータが公開される
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- サービスロールを除外しない（デフォルトで除外済み）
-- supabase_admin, service_role はRLSをバイパスする
```

### Step 2: 基本ポリシーパターン

```sql
-- ユーザーは自分のデータのみアクセス可能
CREATE POLICY "Users can view own data"
  ON public.profiles FOR SELECT
  USING ((SELECT auth.uid()) = user_id);

-- CRITICAL: auth.uid()をSELECTでラップする
-- ラップしないと行ごとに関数が評価されてパフォーマンス劣化
CREATE POLICY "Users can update own data"
  ON public.profiles FOR UPDATE
  USING ((SELECT auth.uid()) = user_id)
  WITH CHECK ((SELECT auth.uid()) = user_id);

CREATE POLICY "Users can insert own data"
  ON public.profiles FOR INSERT
  WITH CHECK ((SELECT auth.uid()) = id);

CREATE POLICY "Users can delete own data"
  ON public.profiles FOR DELETE
  USING ((SELECT auth.uid()) = user_id);
```

### Step 3: ロールベースアクセス制御

```sql
-- ユーザーメタデータにroleを格納
-- JWTから直接参照（DBクエリ不要で高速）
CREATE POLICY "Admins can view all"
  ON public.profiles FOR SELECT
  USING (
    (SELECT auth.jwt() ->> 'role') = 'admin'
    OR (SELECT auth.uid()) = user_id
  );

-- カスタムクレーム活用パターン
-- app_metadata.roleはサーバーサイドでのみ設定可能
CREATE POLICY "Based on app_metadata"
  ON public.profiles FOR SELECT
  USING (
    (SELECT (auth.jwt() -> 'app_metadata' ->> 'role')) = 'admin'
  );
```

### Step 4: マルチテナント設計

```sql
-- テナントテーブル
CREATE TABLE public.tenants (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  name text NOT NULL,
  created_at timestamptz DEFAULT now() NOT NULL
);

-- テナントメンバーシップ
CREATE TABLE public.tenant_members (
  tenant_id uuid REFERENCES public.tenants(id) ON DELETE CASCADE,
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  role text NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  PRIMARY KEY (tenant_id, user_id)
);

-- セキュリティ関数（SELECTラップ + SECURITY DEFINER）
CREATE OR REPLACE FUNCTION public.get_user_tenant_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT tenant_id FROM public.tenant_members
  WHERE user_id = (SELECT auth.uid())
$$;

CREATE OR REPLACE FUNCTION public.get_user_tenant_role(p_tenant_id uuid)
RETURNS text
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT role FROM public.tenant_members
  WHERE user_id = (SELECT auth.uid()) AND tenant_id = p_tenant_id
$$;

-- テナントスコープのポリシー
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Tenant members can view projects"
  ON public.projects FOR SELECT
  USING (tenant_id IN (SELECT public.get_user_tenant_ids()));

CREATE POLICY "Tenant admins can create projects"
  ON public.projects FOR INSERT
  WITH CHECK (
    public.get_user_tenant_role(tenant_id) IN ('owner', 'admin')
  );

CREATE POLICY "Tenant admins can update projects"
  ON public.projects FOR UPDATE
  USING (public.get_user_tenant_role(tenant_id) IN ('owner', 'admin'))
  WITH CHECK (public.get_user_tenant_role(tenant_id) IN ('owner', 'admin'));
```

## パフォーマンス最適化

### 必須インデックス

```sql
-- RLSポリシーで使うカラムには必ずインデックスを張る
CREATE INDEX idx_profiles_user_id ON public.profiles (user_id);
CREATE INDEX idx_tenant_members_user_id ON public.tenant_members (user_id);
CREATE INDEX idx_tenant_members_tenant_user ON public.tenant_members (tenant_id, user_id);
CREATE INDEX idx_projects_tenant_id ON public.projects (tenant_id);
```

### パフォーマンスチェックリスト

| Pattern | Good | Bad |
|---------|------|-----|
| auth.uid()呼び出し | `(SELECT auth.uid())` | `auth.uid()` |
| テナントチェック | SECURITY DEFINER関数 | サブクエリ直書き |
| ロールチェック | JWT claims参照 | DBルックアップ毎回 |
| EXISTS | `EXISTS (SELECT 1 ...)` | `IN (SELECT id ...)` |

### EXPLAIN ANALYZEでポリシー確認

```sql
-- RLSが有効な状態でクエリプランを確認
SET LOCAL role = 'authenticated';
SET LOCAL request.jwt.claims = '{"sub": "user-uuid-here"}';

EXPLAIN ANALYZE SELECT * FROM public.projects;
```

## 共通パターン集

### 公開 + 所有者パターン

```sql
-- 公開データは誰でも閲覧、編集は所有者のみ
CREATE POLICY "Public read"
  ON public.articles FOR SELECT
  USING (published = true OR (SELECT auth.uid()) = author_id);

CREATE POLICY "Owner write"
  ON public.articles FOR UPDATE
  USING ((SELECT auth.uid()) = author_id);
```

### 階層アクセスパターン

```sql
-- org -> team -> project の階層
CREATE POLICY "Hierarchical access"
  ON public.tasks FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.project_members pm
      JOIN public.projects p ON p.id = pm.project_id
      WHERE pm.user_id = (SELECT auth.uid())
        AND p.id = tasks.project_id
    )
  );
```

### サービスロール専用操作

```sql
-- Webhookやバッチ処理はサービスロールで実行
-- RLSはサービスロールをバイパスするため、ポリシー不要
-- クライアントからは絶対にサービスキーを使わない
```

## セキュリティチェックリスト

- [ ] 全テーブルでRLSが有効になっている
- [ ] `auth.uid()`は必ず`(SELECT auth.uid())`でラップ
- [ ] SECURITY DEFINER関数には`SET search_path = ''`を設定
- [ ] INSERT/UPDATEには`WITH CHECK`を設定
- [ ] UPDATEには`USING`と`WITH CHECK`の両方を設定
- [ ] テナントIDの偽装を防止（JWTまたはSECURITY DEFINER経由）
- [ ] RLSポリシーで使うカラムにインデックスがある
- [ ] ストレージバケットにもRLSポリシーを設定

## アンチパターン

```sql
-- BAD: auth.uid()をラップしない（行ごとに評価される）
CREATE POLICY "bad" ON t USING (auth.uid() = user_id);

-- BAD: search_pathを設定しないSECURITY DEFINER
CREATE FUNCTION bad_fn() RETURNS void LANGUAGE sql SECURITY DEFINER AS $$...$$;

-- BAD: USINGなしのUPDATEポリシー（全行が更新対象になる）
CREATE POLICY "bad" ON t FOR UPDATE WITH CHECK (user_id = (SELECT auth.uid()));

-- BAD: サービスキーをクライアントに露出
const supabase = createClient(url, SERVICE_ROLE_KEY) // クライアントで使うな！
```

## Related

- Skill: `postgres-patterns` - PostgreSQLクエリ最適化
- Skill: `drizzle-orm` - Drizzle ORMパターン
- Agent: `security-reviewer` - セキュリティレビュー
- Agent: `database-reviewer` - データベースレビュー
