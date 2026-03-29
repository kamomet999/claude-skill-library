---
name: rag-patterns
description: RAG（検索拡張生成）設計パターン。チャンク戦略、ベクトルDB、リランキング、ハイブリッド検索
category: AI・ML
command: /rag
version: 1.0.0
tags:
  - rag
  - vector
  - embedding
  - retrieval
  - llm
---

# RAG (Retrieval-Augmented Generation) Patterns

RAGパイプラインの設計・実装パターン集。検索精度と生成品質を最大化する。

## When to Activate

- RAGシステムを新規設計するとき
- チャンク戦略を決定するとき
- ベクトルDBを選定・設定するとき
- 検索精度を改善するとき
- ハイブリッド検索やリランキングを導入するとき

## Steps

### Step 1: アーキテクチャ選定

```
[Naive RAG]
Document → Chunk → Embed → VectorDB → Retrieve → Generate

[Advanced RAG]
Document → Chunk → Embed → VectorDB
                                ↓
Query → Rewrite → Hybrid Search → Rerank → Generate

[Modular RAG]
Query → Router → [Search Strategy] → Merge → Rerank → Generate
                    ├── Vector Search
                    ├── Keyword Search
                    └── SQL/Graph Query
```

### Step 2: チャンキング戦略

```typescript
// 固定サイズチャンク（最もシンプル）
function fixedSizeChunk(text: string, size: number, overlap: number): string[] {
  const chunks: string[] = []
  for (let i = 0; i < text.length; i += size - overlap) {
    chunks.push(text.slice(i, i + size))
  }
  return chunks
}

// セマンティックチャンク（文境界で分割）
function semanticChunk(text: string, maxTokens: number): string[] {
  const sentences = text.split(/(?<=[。.!?])\s*/)
  const chunks: string[] = []
  let current = ''

  for (const sentence of sentences) {
    if (countTokens(current + sentence) > maxTokens && current) {
      chunks.push(current.trim())
      current = sentence
    } else {
      current += sentence
    }
  }
  if (current.trim()) {
    chunks.push(current.trim())
  }
  return chunks
}

// 再帰的チャンク（LangChain方式）
// セパレータの優先順位: \n\n → \n → 。→ スペース
function recursiveChunk(
  text: string,
  maxSize: number,
  separators: string[] = ['\n\n', '\n', '。', ' ']
): string[] {
  if (text.length <= maxSize) return [text]

  const sep = separators.find((s) => text.includes(s)) ?? ''
  const parts = text.split(sep)
  const chunks: string[] = []
  let current = ''

  for (const part of parts) {
    const candidate = current ? current + sep + part : part
    if (candidate.length > maxSize && current) {
      chunks.push(current)
      current = part
    } else {
      current = candidate
    }
  }
  if (current) chunks.push(current)
  return chunks
}
```

### Step 3: Embedding生成

```typescript
import OpenAI from 'openai'

const openai = new OpenAI()

async function generateEmbeddings(texts: string[]): Promise<number[][]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',  // 1536次元, 安価
    // model: 'text-embedding-3-large', // 3072次元, 高精度
    input: texts,
  })

  return response.data.map((d) => d.embedding)
}

// バッチ処理（APIレート制限対応）
async function batchEmbed(
  texts: string[],
  batchSize: number = 100
): Promise<number[][]> {
  const results: number[][] = []

  for (let i = 0; i < texts.length; i += batchSize) {
    const batch = texts.slice(i, i + batchSize)
    const embeddings = await generateEmbeddings(batch)
    results.push(...embeddings)

    if (i + batchSize < texts.length) {
      await new Promise((r) => setTimeout(r, 200))
    }
  }

  return results
}
```

### Step 4: ベクトルDB選定

| DB | Use Case | Managed | OSS |
|----|----------|---------|-----|
| Supabase pgvector | 既存Supabaseプロジェクト | Yes | Yes |
| Pinecone | 大規模、低レイテンシ | Yes | No |
| Qdrant | セルフホスト、フィルタリング | Yes | Yes |
| Chroma | プロトタイプ、ローカル開発 | No | Yes |
| Weaviate | マルチモーダル | Yes | Yes |

### Step 5: Supabase pgvectorでの実装

```sql
-- 拡張有効化
CREATE EXTENSION IF NOT EXISTS vector;

-- ドキュメントテーブル
CREATE TABLE documents (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  content text NOT NULL,
  metadata jsonb DEFAULT '{}'::jsonb,
  embedding vector(1536),
  created_at timestamptz DEFAULT now()
);

-- IVFFlatインデックス（大量データ向け）
CREATE INDEX ON documents
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- HNSWインデックス（高精度、メモリ多め）
CREATE INDEX ON documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- 検索関数
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding vector(1536),
  match_threshold float DEFAULT 0.7,
  match_count int DEFAULT 10,
  filter_metadata jsonb DEFAULT '{}'::jsonb
)
RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    d.id,
    d.content,
    d.metadata,
    1 - (d.embedding <=> query_embedding) AS similarity
  FROM documents d
  WHERE 1 - (d.embedding <=> query_embedding) > match_threshold
    AND (filter_metadata = '{}'::jsonb OR d.metadata @> filter_metadata)
  ORDER BY d.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

## 検索品質改善

### Query Rewriting

```typescript
// クエリを検索に適した形に変換
async function rewriteQuery(originalQuery: string): Promise<string[]> {
  const response = await llm.generate({
    system: `ユーザーの質問を、ベクトル検索に最適化された3つの異なる検索クエリに書き換えてください。
JSON配列で返してください。`,
    user: originalQuery,
    temperature: 0.3,
  })

  return JSON.parse(response)
}

// HyDE: 仮想回答を生成してから検索
async function hypotheticalDocumentEmbedding(query: string): Promise<number[]> {
  const hypotheticalAnswer = await llm.generate({
    system: '以下の質問に対する詳細な回答を、あたかも参考文書から抜粋したかのように書いてください。',
    user: query,
  })

  const [embedding] = await generateEmbeddings([hypotheticalAnswer])
  return embedding
}
```

### ハイブリッド検索（Vector + Full-Text）

```sql
-- フルテキスト検索カラム追加
ALTER TABLE documents ADD COLUMN fts tsvector
  GENERATED ALWAYS AS (to_tsvector('japanese', content)) STORED;
CREATE INDEX ON documents USING gin (fts);

-- ハイブリッド検索関数
CREATE OR REPLACE FUNCTION hybrid_search(
  query_text text,
  query_embedding vector(1536),
  match_count int DEFAULT 10,
  vector_weight float DEFAULT 0.7,
  text_weight float DEFAULT 0.3
)
RETURNS TABLE (id bigint, content text, score float)
LANGUAGE sql
AS $$
  WITH vector_results AS (
    SELECT d.id, d.content,
      1 - (d.embedding <=> query_embedding) AS similarity
    FROM documents d
    ORDER BY d.embedding <=> query_embedding
    LIMIT match_count * 2
  ),
  text_results AS (
    SELECT d.id, d.content,
      ts_rank(d.fts, plainto_tsquery('japanese', query_text)) AS rank
    FROM documents d
    WHERE d.fts @@ plainto_tsquery('japanese', query_text)
    LIMIT match_count * 2
  )
  SELECT
    COALESCE(v.id, t.id) AS id,
    COALESCE(v.content, t.content) AS content,
    (COALESCE(v.similarity, 0) * vector_weight +
     COALESCE(t.rank, 0) * text_weight) AS score
  FROM vector_results v
  FULL OUTER JOIN text_results t ON v.id = t.id
  ORDER BY score DESC
  LIMIT match_count;
$$;
```

### Reranking

```typescript
// Cross-Encoder Reranking（Cohere API例）
async function rerankResults(
  query: string,
  documents: { content: string; id: string }[],
  topK: number = 5
): Promise<{ content: string; id: string; score: number }[]> {
  const response = await fetch('https://api.cohere.ai/v1/rerank', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.COHERE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'rerank-v3.5',
      query,
      documents: documents.map((d) => d.content),
      top_n: topK,
    }),
  })

  const data = await response.json()
  return data.results.map((r: { index: number; relevance_score: number }) => ({
    ...documents[r.index],
    score: r.relevance_score,
  }))
}
```

## RAGパイプライン全体

```typescript
async function ragPipeline(userQuery: string): Promise<string> {
  // 1. Query Rewriting
  const queries = await rewriteQuery(userQuery)

  // 2. Parallel Retrieval
  const allResults = await Promise.all(
    queries.map((q) => hybridSearch(q))
  )

  // 3. Deduplicate
  const unique = deduplicateByContent(allResults.flat())

  // 4. Rerank
  const ranked = await rerankResults(userQuery, unique, 5)

  // 5. Generate
  const context = ranked.map((r) => r.content).join('\n\n---\n\n')
  const answer = await llm.generate({
    system: `以下のコンテキストに基づいて質問に回答してください。
コンテキストに情報がない場合は「情報が見つかりませんでした」と回答してください。
推測で回答しないでください。

コンテキスト:
${context}`,
    user: userQuery,
  })

  return answer
}
```

## チャンクサイズガイド

| Content Type | Chunk Size | Overlap | Strategy |
|-------------|-----------|---------|----------|
| 技術文書 | 500-1000 tokens | 100 | セマンティック |
| FAQ | 1 QA per chunk | 0 | QAペア単位 |
| 法律文書 | 300-500 tokens | 50 | 段落単位 |
| コード | 関数単位 | 0 | AST解析 |
| チャット履歴 | 会話単位 | 1-2ターン | ターン境界 |

## Best Practices

- チャンクにメタデータ（ソース、ページ番号、日付）を付与する
- 検索結果に出典を表示し、ハルシネーションを検証可能にする
- Embedding modelとチャンクサイズの組み合わせを評価してから本番投入
- 定期的にインデックスを再構築する（データ追加に伴う精度劣化対策）
- 検索ログを収集し、検索品質を継続的に改善する

## Related

- Skill: `prompt-engineering` - プロンプト設計
- Skill: `postgres-patterns` - PostgreSQLクエリ最適化
- Skill: `supabase-rls` - Supabase Row Level Security
