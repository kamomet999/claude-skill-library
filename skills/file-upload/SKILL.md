---
name: file-upload
description: ファイルアップロード設計。マルチパート、チャンク、S3直接アップロード、画像リサイズ、ウイルススキャン
category: バックエンド
command: /file-upload
version: 1.0.0
tags: [upload, s3, multipart, image, storage]
---

# ファイルアップロード設計パターン

## 概要

プロダクション品質のファイルアップロードシステムを構築するためのパターン集。
マルチパートアップロード、チャンクアップロード、S3 Presigned URL、画像処理、ウイルススキャン、ストレージ設計を網羅する。

## Steps

1. アップロード方式を選択する（サーバー経由 vs S3直接）
2. ファイルバリデーションを実装する（サイズ、MIME、拡張子）
3. ストレージ構造を設計する
4. マルチパートアップロードを実装する
5. 大容量ファイル用のチャンクアップロードを実装する
6. 画像処理パイプラインを構築する
7. ウイルススキャンを統合する
8. CDN配信を設定する

## アップロード方式の選択

| 方式 | 最大サイズ | メリット | デメリット |
|------|-----------|---------|-----------|
| サーバー経由（Multer） | ~50MB | 簡単、前処理可能 | サーバー負荷大 |
| S3 Presigned URL | 5GB | サーバー負荷なし | 前処理不可 |
| S3マルチパート | 5TB | 超大容量対応 | 実装が複雑 |
| tus プロトコル | 無制限 | 再開可能、標準化 | サーバー必要 |

## プロジェクト構造

```
src/
  upload/
    config.ts              # アップロード設定
    validators.ts          # ファイルバリデーション
    storage.ts             # ストレージ抽象化
    presigned.ts           # Presigned URL生成
    chunked.ts             # チャンクアップロード
  processing/
    image.processor.ts     # 画像リサイズ・変換
    video.processor.ts     # 動画トランスコード
    antivirus.ts           # ウイルススキャン
  middleware/
    upload.middleware.ts    # Multerミドルウェア
  routes/
    upload.routes.ts
```

## ファイルバリデーション

```typescript
// src/upload/validators.ts
import { z } from 'zod'
import fileType from 'file-type'

// 許可設定
const UPLOAD_CONFIG = {
  image: {
    maxSize: 10 * 1024 * 1024,  // 10MB
    allowedMimes: ['image/jpeg', 'image/png', 'image/webp', 'image/gif'],
    allowedExtensions: ['.jpg', '.jpeg', '.png', '.webp', '.gif'],
  },
  document: {
    maxSize: 50 * 1024 * 1024,  // 50MB
    allowedMimes: ['application/pdf', 'application/msword', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
    allowedExtensions: ['.pdf', '.doc', '.docx'],
  },
  video: {
    maxSize: 500 * 1024 * 1024,  // 500MB
    allowedMimes: ['video/mp4', 'video/webm'],
    allowedExtensions: ['.mp4', '.webm'],
  },
} as const

type FileCategory = keyof typeof UPLOAD_CONFIG

interface ValidatedFile {
  buffer: Buffer
  originalName: string
  mimeType: string
  size: number
  category: FileCategory
}

export async function validateFile(
  buffer: Buffer,
  originalName: string,
  category: FileCategory
): Promise<ValidatedFile> {
  const config = UPLOAD_CONFIG[category]

  // 1. サイズチェック
  if (buffer.length > config.maxSize) {
    const maxMB = config.maxSize / (1024 * 1024)
    throw new Error(`ファイルサイズが上限（${maxMB}MB）を超えています`)
  }

  // 2. マジックバイトでMIME型を検証（拡張子偽装防止）
  const detectedType = await fileType.fromBuffer(buffer)
  if (!detectedType) {
    throw new Error('ファイル形式を判定できません')
  }

  if (!config.allowedMimes.includes(detectedType.mime as typeof config.allowedMimes[number])) {
    throw new Error(`許可されていないファイル形式です: ${detectedType.mime}`)
  }

  // 3. 拡張子チェック
  const ext = '.' + originalName.split('.').pop()?.toLowerCase()
  if (!config.allowedExtensions.includes(ext as typeof config.allowedExtensions[number])) {
    throw new Error(`許可されていない拡張子です: ${ext}`)
  }

  // 4. ファイル名サニタイズ
  const sanitizedName = originalName
    .replace(/[^a-zA-Z0-9._-]/g, '_')
    .replace(/_{2,}/g, '_')

  return {
    buffer,
    originalName: sanitizedName,
    mimeType: detectedType.mime,
    size: buffer.length,
    category,
  }
}
```

## Multerミドルウェア（サーバー経由アップロード）

```typescript
// src/middleware/upload.middleware.ts
import multer from 'multer'
import crypto from 'crypto'
import path from 'path'

// メモリストレージ（S3にアップロードする場合）
const memoryStorage = multer.memoryStorage()

// ディスクストレージ（ローカル保存する場合）
const diskStorage = multer.diskStorage({
  destination: '/tmp/uploads',
  filename: (req, file, cb) => {
    const uniqueName = `${crypto.randomUUID()}${path.extname(file.originalname)}`
    cb(null, uniqueName)
  },
})

export const uploadImage = multer({
  storage: memoryStorage,
  limits: {
    fileSize: 10 * 1024 * 1024,  // 10MB
    files: 5,                     // 最大5ファイル
  },
  fileFilter: (req, file, cb) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/webp']
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true)
    } else {
      cb(new Error('許可されていないファイル形式です'))
    }
  },
})

export const uploadDocument = multer({
  storage: memoryStorage,
  limits: { fileSize: 50 * 1024 * 1024, files: 1 },
})
```

## S3ストレージ

```typescript
// src/upload/storage.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import crypto from 'crypto'

const s3 = new S3Client({
  region: process.env.AWS_REGION ?? 'ap-northeast-1',
})

const BUCKET = process.env.S3_BUCKET

if (!BUCKET) throw new Error('S3_BUCKET must be set')

// ストレージキー生成（年月でパーティション）
function generateKey(category: string, extension: string): string {
  const date = new Date()
  const partition = `${date.getFullYear()}/${String(date.getMonth() + 1).padStart(2, '0')}`
  const filename = `${crypto.randomUUID()}${extension}`
  return `${category}/${partition}/${filename}`
}

// ファイルアップロード
export async function uploadToS3(
  buffer: Buffer,
  category: string,
  mimeType: string,
  extension: string
): Promise<{ key: string; url: string }> {
  const key = generateKey(category, extension)

  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: buffer,
    ContentType: mimeType,
    // メタデータ
    Metadata: {
      'uploaded-at': new Date().toISOString(),
    },
    // サーバーサイド暗号化
    ServerSideEncryption: 'AES256',
  }))

  const url = `https://${BUCKET}.s3.amazonaws.com/${key}`
  return { key, url }
}

// ファイル削除
export async function deleteFromS3(key: string): Promise<void> {
  await s3.send(new DeleteObjectCommand({
    Bucket: BUCKET,
    Key: key,
  }))
}

// 署名付きURL生成（一時的なアクセス）
export async function getPresignedDownloadUrl(key: string, expiresIn = 3600): Promise<string> {
  return getSignedUrl(s3, new GetObjectCommand({
    Bucket: BUCKET,
    Key: key,
  }), { expiresIn })
}
```

## Presigned URL アップロード（推奨）

```typescript
// src/upload/presigned.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import crypto from 'crypto'

const s3 = new S3Client({ region: process.env.AWS_REGION })
const BUCKET = process.env.S3_BUCKET!

interface PresignedUploadResult {
  uploadUrl: string      // PUT先URL
  key: string            // S3キー
  publicUrl: string      // 公開URL（CloudFront経由）
  expiresAt: string      // URL有効期限
}

export async function createPresignedUpload(
  category: string,
  contentType: string,
  maxSize: number
): Promise<PresignedUploadResult> {
  const date = new Date()
  const partition = `${date.getFullYear()}/${String(date.getMonth() + 1).padStart(2, '0')}`
  const extension = contentType.split('/')[1] ?? 'bin'
  const key = `${category}/${partition}/${crypto.randomUUID()}.${extension}`

  const uploadUrl = await getSignedUrl(
    s3,
    new PutObjectCommand({
      Bucket: BUCKET,
      Key: key,
      ContentType: contentType,
      // Content-Lengthの上限を設定（サイズ制限）
      Metadata: {
        'max-size': String(maxSize),
      },
    }),
    { expiresIn: 600 }  // 10分有効
  )

  const cdnDomain = process.env.CDN_DOMAIN ?? `${BUCKET}.s3.amazonaws.com`

  return {
    uploadUrl,
    key,
    publicUrl: `https://${cdnDomain}/${key}`,
    expiresAt: new Date(Date.now() + 600_000).toISOString(),
  }
}

// APIエンドポイント
// POST /api/uploads/presign
// { category: "image", contentType: "image/jpeg", size: 1234567 }
// -> { uploadUrl: "https://s3.../...", key: "...", publicUrl: "..." }
```

## 画像処理パイプライン

```typescript
// src/processing/image.processor.ts
import sharp from 'sharp'

interface ImageVariant {
  suffix: string
  width: number
  height: number
  quality: number
  format: 'jpeg' | 'webp' | 'avif'
}

const IMAGE_VARIANTS: ImageVariant[] = [
  { suffix: 'thumb', width: 150, height: 150, quality: 80, format: 'webp' },
  { suffix: 'medium', width: 800, height: 600, quality: 85, format: 'webp' },
  { suffix: 'large', width: 1920, height: 1080, quality: 90, format: 'webp' },
  // 元画像のWebP変換
  { suffix: 'original', width: 4096, height: 4096, quality: 90, format: 'webp' },
]

interface ProcessedImage {
  suffix: string
  buffer: Buffer
  width: number
  height: number
  format: string
  size: number
}

export async function processImage(inputBuffer: Buffer): Promise<ProcessedImage[]> {
  const metadata = await sharp(inputBuffer).metadata()

  const results = await Promise.all(
    IMAGE_VARIANTS.map(async (variant) => {
      const processed = sharp(inputBuffer)
        .resize(variant.width, variant.height, {
          fit: 'inside',           // アスペクト比を維持
          withoutEnlargement: true, // 拡大しない
        })
        .rotate()  // EXIFに基づいて自動回転

      let formatted: sharp.Sharp
      switch (variant.format) {
        case 'webp':
          formatted = processed.webp({ quality: variant.quality })
          break
        case 'avif':
          formatted = processed.avif({ quality: variant.quality })
          break
        default:
          formatted = processed.jpeg({ quality: variant.quality, progressive: true })
      }

      const buffer = await formatted.toBuffer()
      const info = await sharp(buffer).metadata()

      return {
        suffix: variant.suffix,
        buffer,
        width: info.width ?? 0,
        height: info.height ?? 0,
        format: variant.format,
        size: buffer.length,
      }
    })
  )

  return results
}

// EXIF情報の除去（プライバシー保護）
export async function stripExif(buffer: Buffer): Promise<Buffer> {
  return sharp(buffer)
    .rotate()  // EXIF回転を適用してから
    .withMetadata({ orientation: undefined })  // EXIFを除去
    .toBuffer()
}
```

## チャンクアップロード

```typescript
// src/upload/chunked.ts
import { redis } from '../cache/redis-client'
import { uploadToS3 } from './storage'

interface ChunkUploadSession {
  id: string
  fileName: string
  fileSize: number
  chunkSize: number
  totalChunks: number
  uploadedChunks: number[]
  category: string
  mimeType: string
  createdAt: string
}

// アップロードセッション開始
export async function initChunkUpload(params: {
  fileName: string
  fileSize: number
  mimeType: string
  category: string
}): Promise<ChunkUploadSession> {
  const chunkSize = 5 * 1024 * 1024  // 5MB
  const totalChunks = Math.ceil(params.fileSize / chunkSize)
  const sessionId = crypto.randomUUID()

  const session: ChunkUploadSession = {
    id: sessionId,
    fileName: params.fileName,
    fileSize: params.fileSize,
    chunkSize,
    totalChunks,
    uploadedChunks: [],
    category: params.category,
    mimeType: params.mimeType,
    createdAt: new Date().toISOString(),
  }

  // セッションをRedisに保存（1時間有効）
  await redis.setex(
    `chunk-upload:${sessionId}`,
    3600,
    JSON.stringify(session)
  )

  return session
}

// チャンクアップロード
export async function uploadChunk(
  sessionId: string,
  chunkIndex: number,
  buffer: Buffer
): Promise<{ completed: boolean; progress: number }> {
  const sessionData = await redis.get(`chunk-upload:${sessionId}`)
  if (!sessionData) throw new Error('アップロードセッションが期限切れです')

  const session: ChunkUploadSession = JSON.parse(sessionData)

  // チャンクを一時保存
  await redis.setex(
    `chunk:${sessionId}:${chunkIndex}`,
    3600,
    buffer.toString('base64')
  )

  // アップロード済みチャンクを更新
  const updatedChunks = [...session.uploadedChunks, chunkIndex]
  const updatedSession = { ...session, uploadedChunks: updatedChunks }
  await redis.setex(`chunk-upload:${sessionId}`, 3600, JSON.stringify(updatedSession))

  const progress = updatedChunks.length / session.totalChunks

  // 全チャンク受信完了
  if (updatedChunks.length === session.totalChunks) {
    await assembleAndUpload(sessionId, updatedSession)
    return { completed: true, progress: 1 }
  }

  return { completed: false, progress }
}

// チャンクを結合してS3にアップロード
async function assembleAndUpload(
  sessionId: string,
  session: ChunkUploadSession
): Promise<void> {
  const chunks: Buffer[] = []

  for (let i = 0; i < session.totalChunks; i++) {
    const chunkData = await redis.get(`chunk:${sessionId}:${i}`)
    if (!chunkData) throw new Error(`チャンク ${i} が見つかりません`)
    chunks.push(Buffer.from(chunkData, 'base64'))
  }

  const assembledBuffer = Buffer.concat(chunks)
  const ext = '.' + session.fileName.split('.').pop()

  await uploadToS3(assembledBuffer, session.category, session.mimeType, ext)

  // クリーンアップ
  const pipeline = redis.pipeline()
  pipeline.del(`chunk-upload:${sessionId}`)
  for (let i = 0; i < session.totalChunks; i++) {
    pipeline.del(`chunk:${sessionId}:${i}`)
  }
  await pipeline.exec()
}
```

## ウイルススキャン

```typescript
// src/processing/antivirus.ts
import ClamScan from 'clamscan'

let clamav: ClamScan | null = null

async function getClamAV(): Promise<ClamScan> {
  if (!clamav) {
    clamav = await new ClamScan().init({
      clamdscan: {
        host: process.env.CLAMAV_HOST ?? 'localhost',
        port: Number(process.env.CLAMAV_PORT) ?? 3310,
        timeout: 30000,
      },
    })
  }
  return clamav
}

export async function scanFile(buffer: Buffer): Promise<{
  isClean: boolean
  viruses: string[]
}> {
  const scanner = await getClamAV()

  const { isInfected, viruses } = await scanner.scanStream(
    new (await import('stream')).Readable({
      read() {
        this.push(buffer)
        this.push(null)
      },
    })
  )

  return {
    isClean: !isInfected,
    viruses: viruses ?? [],
  }
}

// アップロードパイプラインに統合
export async function safeUpload(buffer: Buffer, category: string, mimeType: string) {
  // 1. ウイルススキャン
  const scanResult = await scanFile(buffer)
  if (!scanResult.isClean) {
    throw new Error(`ウイルスが検出されました: ${scanResult.viruses.join(', ')}`)
  }

  // 2. ファイル処理（画像の場合はリサイズ等）
  // 3. S3アップロード
  // ...
}
```

## アップロードAPIエンドポイント

```typescript
// src/routes/upload.routes.ts
import { Router } from 'express'
import { authenticate } from '../middleware/auth'
import { uploadImage } from '../middleware/upload.middleware'
import { asyncHandler } from '../utils/async-handler'
import { validateFile } from '../upload/validators'
import { uploadToS3 } from '../upload/storage'
import { processImage } from '../processing/image.processor'
import { createPresignedUpload } from '../upload/presigned'

const router = Router()

// 画像アップロード（サーバー経由）
router.post('/images', authenticate, uploadImage.array('files', 5), asyncHandler(async (req, res) => {
  const files = req.files as Express.Multer.File[]
  if (!files?.length) {
    return res.status(400).json({ success: false, error: 'ファイルが必要です' })
  }

  const results = await Promise.all(
    files.map(async (file) => {
      // バリデーション
      const validated = await validateFile(file.buffer, file.originalname, 'image')

      // 画像処理
      const variants = await processImage(validated.buffer)

      // 各バリアントをS3にアップロード
      const uploaded = await Promise.all(
        variants.map(async (v) => {
          const result = await uploadToS3(v.buffer, `images/${v.suffix}`, `image/${v.format}`, `.${v.format}`)
          return { variant: v.suffix, ...result, width: v.width, height: v.height, size: v.size }
        })
      )

      return {
        originalName: validated.originalName,
        variants: uploaded,
      }
    })
  )

  res.status(201).json({ success: true, data: results })
}))

// Presigned URLの取得
router.post('/presign', authenticate, asyncHandler(async (req, res) => {
  const { category, contentType, size } = req.body

  // サイズ検証
  const maxSizes: Record<string, number> = {
    image: 10 * 1024 * 1024,
    document: 50 * 1024 * 1024,
    video: 500 * 1024 * 1024,
  }

  const maxSize = maxSizes[category]
  if (!maxSize) {
    return res.status(400).json({ success: false, error: '不明なカテゴリです' })
  }

  if (size > maxSize) {
    return res.status(400).json({ success: false, error: 'ファイルサイズが上限を超えています' })
  }

  const result = await createPresignedUpload(category, contentType, maxSize)
  res.json({ success: true, data: result })
}))

export { router as uploadRoutes }
```

## クライアント側の実装例

```typescript
// client/upload-client.ts

// Presigned URLを使ったアップロード
async function uploadWithPresignedUrl(file: File, category: string): Promise<string> {
  // 1. Presigned URLを取得
  const presignRes = await fetch('/api/uploads/presign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({
      category,
      contentType: file.type,
      size: file.size,
    }),
  })
  const { data } = await presignRes.json()

  // 2. S3に直接PUT
  await fetch(data.uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
  })

  return data.publicUrl
}

// チャンクアップロード（大容量ファイル、進捗表示付き）
async function uploadLargeFile(
  file: File,
  category: string,
  onProgress: (progress: number) => void
): Promise<void> {
  // 1. セッション開始
  const initRes = await fetch('/api/uploads/chunk/init', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      fileName: file.name,
      fileSize: file.size,
      mimeType: file.type,
      category,
    }),
  })
  const { data: session } = await initRes.json()

  // 2. チャンクごとにアップロード
  const chunkSize = session.chunkSize
  for (let i = 0; i < session.totalChunks; i++) {
    const start = i * chunkSize
    const end = Math.min(start + chunkSize, file.size)
    const chunk = file.slice(start, end)

    const formData = new FormData()
    formData.append('chunk', chunk)
    formData.append('index', String(i))

    const res = await fetch(`/api/uploads/chunk/${session.id}`, {
      method: 'POST',
      body: formData,
    })
    const result = await res.json()
    onProgress(result.data.progress)
  }
}
```

## ベストプラクティス

1. **マジックバイト検証**: 拡張子だけでなくファイルの中身でMIMEを判定する
2. **Presigned URL推奨**: サーバーの帯域を節約し、スケーラブルにする
3. **画像は複数バリアント**: thumb / medium / large をアップロード時に生成する
4. **EXIFを除去**: GPS情報などのプライバシーデータを削除する
5. **ファイル名をUUID化**: ユーザー提供のファイル名を直接使わない
6. **サーバーサイド暗号化**: S3の `AES256` または `aws:kms` を有効にする
7. **ウイルススキャン**: ユーザーアップロードファイルは必ずスキャンする
8. **アップロードサイズ制限**: カテゴリごとに適切な上限を設定する

## アンチパターン

- ユーザーのファイル名をそのままストレージキーにする（パストラバーサル攻撃）
- 拡張子だけでファイルタイプを判定する（偽装可能）
- アップロードファイルをサーバーのローカルディスクに永続保存する
- 画像のリサイズをリクエスト中に同期で行う（大量ファイルで遅い）
- Content-Typeヘッダーだけを信用する（クライアントが設定する値）
- アップロードディレクトリをWebサーバーで直接公開する（実行可能ファイルの危険）
