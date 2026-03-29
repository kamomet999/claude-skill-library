---
name: websocket-realtime
description: WebSocket/Socket.ioリアルタイム通信設計、ルーム管理、再接続、スケーリング
category: バックエンド
command: /websocket-realtime
version: 1.0.0
tags: [websocket, socket.io, realtime, pub-sub]
---

# WebSocket リアルタイム通信設計パターン

## 概要

Socket.io / WebSocketを使ったリアルタイム通信システムの設計パターン集。
ルーム管理、認証、再接続戦略、複数サーバーでのスケーリング、メッセージ設計を網羅する。

## Steps

1. Socket.ioサーバーを設定する
2. 認証ミドルウェアを実装する
3. イベント設計とメッセージ型を定義する
4. ルーム管理を実装する
5. 再接続とエラーハンドリングを設定する
6. Redis Adapterで水平スケーリングに対応する
7. クライアント側の接続管理を実装する
8. モニタリングとデバッグツールを追加する

## プロジェクト構造

```
src/
  socket/
    server.ts              # Socket.ioサーバー設定
    auth.middleware.ts      # 認証ミドルウェア
    events.ts              # イベント名定数
    types.ts               # メッセージ型定義
    handlers/
      chat.handler.ts      # チャットイベントハンドラー
      notification.handler.ts
      presence.handler.ts  # オンライン状態管理
    rooms/
      room-manager.ts      # ルーム管理
    adapters/
      redis-adapter.ts     # Redis Adapter設定
  client/
    socket-client.ts       # クライアント側SDK
```

## サーバー設定

```typescript
// src/socket/server.ts
import { Server as HttpServer } from 'http'
import { Server, Socket } from 'socket.io'
import { createAdapter } from '@socket.io/redis-adapter'
import { createClient } from 'redis'
import { authMiddleware } from './auth.middleware'
import { registerChatHandlers } from './handlers/chat.handler'
import { registerPresenceHandlers } from './handlers/presence.handler'
import { registerNotificationHandlers } from './handlers/notification.handler'
import { ServerEvents, ClientEvents } from './types'

export async function createSocketServer(httpServer: HttpServer): Promise<Server> {
  const io = new Server<ClientEvents, ServerEvents>(httpServer, {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
      credentials: true,
    },
    pingInterval: 25000,       // 25秒ごとにping
    pingTimeout: 20000,        // 20秒でタイムアウト
    maxHttpBufferSize: 1e6,    // 1MB（DoS防止）
    transports: ['websocket', 'polling'],  // WebSocket優先
    connectionStateRecovery: {
      maxDisconnectionDuration: 2 * 60 * 1000,  // 2分間のセッション復旧
      skipMiddlewares: true,
    },
  })

  // Redis Adapter（複数サーバー対応）
  const pubClient = createClient({ url: process.env.REDIS_URL })
  const subClient = pubClient.duplicate()
  await Promise.all([pubClient.connect(), subClient.connect()])
  io.adapter(createAdapter(pubClient, subClient))

  // 認証ミドルウェア
  io.use(authMiddleware)

  // 接続ハンドリング
  io.on('connection', (socket: Socket) => {
    console.info(`Client connected: ${socket.id}, user: ${socket.data.userId}`)

    registerChatHandlers(io, socket)
    registerPresenceHandlers(io, socket)
    registerNotificationHandlers(io, socket)

    socket.on('disconnect', (reason) => {
      console.info(`Client disconnected: ${socket.id}, reason: ${reason}`)
    })

    socket.on('error', (error) => {
      console.error(`Socket error: ${socket.id}`, error)
    })
  })

  return io
}
```

## 認証ミドルウェア

```typescript
// src/socket/auth.middleware.ts
import { Socket } from 'socket.io'
import { verifyToken } from '../auth/jwt'

export async function authMiddleware(
  socket: Socket,
  next: (err?: Error) => void
): Promise<void> {
  try {
    const token = socket.handshake.auth.token
      ?? socket.handshake.headers.authorization?.replace('Bearer ', '')

    if (!token) {
      return next(new Error('認証トークンが必要です'))
    }

    const payload = await verifyToken(token)
    socket.data.userId = payload.sub
    socket.data.userName = payload.name
    socket.data.roles = payload.roles

    next()
  } catch (error) {
    next(new Error('認証に失敗しました'))
  }
}
```

## イベント・メッセージ型定義

```typescript
// src/socket/types.ts

// サーバー -> クライアント
export interface ServerEvents {
  'chat:message': (message: ChatMessage) => void
  'chat:typing': (data: { userId: string; roomId: string }) => void
  'chat:stop-typing': (data: { userId: string; roomId: string }) => void
  'presence:online': (data: { userId: string }) => void
  'presence:offline': (data: { userId: string }) => void
  'presence:list': (data: { users: PresenceUser[] }) => void
  'notification:new': (notification: Notification) => void
  'error': (data: { code: string; message: string }) => void
}

// クライアント -> サーバー
export interface ClientEvents {
  'chat:send': (data: SendMessageData, ack: (response: AckResponse) => void) => void
  'chat:typing': (data: { roomId: string }) => void
  'chat:stop-typing': (data: { roomId: string }) => void
  'room:join': (data: { roomId: string }, ack: (response: AckResponse) => void) => void
  'room:leave': (data: { roomId: string }) => void
  'presence:heartbeat': () => void
}

export interface ChatMessage {
  id: string
  roomId: string
  userId: string
  userName: string
  content: string
  type: 'text' | 'image' | 'system'
  createdAt: string
}

export interface SendMessageData {
  roomId: string
  content: string
  type: 'text' | 'image'
}

export interface AckResponse {
  success: boolean
  error?: string
  data?: unknown
}

export interface PresenceUser {
  userId: string
  userName: string
  status: 'online' | 'away' | 'busy'
  lastSeen: string
}
```

## チャットハンドラー

```typescript
// src/socket/handlers/chat.handler.ts
import { Server, Socket } from 'socket.io'
import { z } from 'zod'
import { chatService } from '../../services/chat.service'

const sendMessageSchema = z.object({
  roomId: z.string().uuid(),
  content: z.string().min(1).max(5000),
  type: z.enum(['text', 'image']),
})

export function registerChatHandlers(io: Server, socket: Socket): void {
  // メッセージ送信
  socket.on('chat:send', async (data, ack) => {
    try {
      const validated = sendMessageSchema.parse(data)

      // ルームに参加しているか確認
      if (!socket.rooms.has(validated.roomId)) {
        return ack({ success: false, error: 'ルームに参加していません' })
      }

      const message = await chatService.createMessage({
        ...validated,
        userId: socket.data.userId,
        userName: socket.data.userName,
      })

      // ルーム内の全員にブロードキャスト（送信者含む）
      io.to(validated.roomId).emit('chat:message', message)

      ack({ success: true, data: { messageId: message.id } })
    } catch (error) {
      if (error instanceof z.ZodError) {
        return ack({ success: false, error: 'バリデーションエラー' })
      }
      ack({ success: false, error: 'メッセージ送信に失敗しました' })
    }
  })

  // タイピングインジケーター（debounce付き）
  let typingTimeout: NodeJS.Timeout | null = null

  socket.on('chat:typing', ({ roomId }) => {
    socket.to(roomId).emit('chat:typing', {
      userId: socket.data.userId,
      roomId,
    })

    // 3秒後に自動で停止
    if (typingTimeout) clearTimeout(typingTimeout)
    typingTimeout = setTimeout(() => {
      socket.to(roomId).emit('chat:stop-typing', {
        userId: socket.data.userId,
        roomId,
      })
    }, 3000)
  })
}
```

## ルーム管理

```typescript
// src/socket/rooms/room-manager.ts
import { Server, Socket } from 'socket.io'
import { roomService } from '../../services/room.service'

export function registerRoomHandlers(io: Server, socket: Socket): void {
  socket.on('room:join', async ({ roomId }, ack) => {
    try {
      // 権限チェック
      const hasAccess = await roomService.checkAccess(socket.data.userId, roomId)
      if (!hasAccess) {
        return ack({ success: false, error: 'アクセス権限がありません' })
      }

      await socket.join(roomId)

      // ルーム内にシステムメッセージ
      socket.to(roomId).emit('chat:message', {
        id: crypto.randomUUID(),
        roomId,
        userId: 'system',
        userName: 'System',
        content: `${socket.data.userName} が参加しました`,
        type: 'system',
        createdAt: new Date().toISOString(),
      })

      // 参加中のユーザー一覧を返す
      const sockets = await io.in(roomId).fetchSockets()
      const users = sockets.map((s) => ({
        userId: s.data.userId,
        userName: s.data.userName,
        status: 'online' as const,
        lastSeen: new Date().toISOString(),
      }))

      ack({ success: true, data: { users } })
    } catch (error) {
      ack({ success: false, error: 'ルーム参加に失敗しました' })
    }
  })

  socket.on('room:leave', async ({ roomId }) => {
    await socket.leave(roomId)

    socket.to(roomId).emit('chat:message', {
      id: crypto.randomUUID(),
      roomId,
      userId: 'system',
      userName: 'System',
      content: `${socket.data.userName} が退出しました`,
      type: 'system',
      createdAt: new Date().toISOString(),
    })
  })
}
```

## プレゼンス（オンライン状態）管理

```typescript
// src/socket/handlers/presence.handler.ts
import { Server, Socket } from 'socket.io'
import { Redis } from 'ioredis'

const redis = new Redis(process.env.REDIS_URL ?? '')
const PRESENCE_TTL = 60  // 60秒で期限切れ
const PRESENCE_KEY = 'presence'

export function registerPresenceHandlers(io: Server, socket: Socket): void {
  const userId = socket.data.userId

  // 接続時: オンラインに設定
  redis.hset(PRESENCE_KEY, userId, JSON.stringify({
    status: 'online',
    socketId: socket.id,
    lastSeen: new Date().toISOString(),
  }))
  socket.broadcast.emit('presence:online', { userId })

  // ハートビート
  socket.on('presence:heartbeat', () => {
    redis.hset(PRESENCE_KEY, userId, JSON.stringify({
      status: 'online',
      socketId: socket.id,
      lastSeen: new Date().toISOString(),
    }))
  })

  // 切断時: オフラインに設定（遅延あり: 一時的な切断対策）
  socket.on('disconnect', () => {
    setTimeout(async () => {
      const data = await redis.hget(PRESENCE_KEY, userId)
      if (data) {
        const parsed = JSON.parse(data)
        // 同じユーザーが別のソケットで再接続していなければオフラインに
        if (parsed.socketId === socket.id) {
          await redis.hdel(PRESENCE_KEY, userId)
          socket.broadcast.emit('presence:offline', { userId })
        }
      }
    }, 5000)  // 5秒待機
  })
}
```

## クライアント側実装

```typescript
// src/client/socket-client.ts
import { io, Socket } from 'socket.io-client'
import { ServerEvents, ClientEvents } from '../socket/types'

interface SocketClientOptions {
  url: string
  token: string
  onConnect?: () => void
  onDisconnect?: (reason: string) => void
  onError?: (error: Error) => void
}

export function createSocketClient(options: SocketClientOptions): Socket<ServerEvents, ClientEvents> {
  const socket: Socket<ServerEvents, ClientEvents> = io(options.url, {
    auth: { token: options.token },
    transports: ['websocket', 'polling'],
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 30000,
    randomizationFactor: 0.5,
    timeout: 20000,
  })

  socket.on('connect', () => {
    console.info('Socket connected:', socket.id)
    options.onConnect?.()
  })

  socket.on('disconnect', (reason) => {
    console.warn('Socket disconnected:', reason)
    options.onDisconnect?.(reason)

    // サーバー側からの切断は自動再接続されない
    if (reason === 'io server disconnect') {
      socket.connect()
    }
  })

  socket.on('connect_error', (error) => {
    console.error('Connection error:', error.message)
    options.onError?.(error)

    // 認証エラーの場合はトークンをリフレッシュ
    if (error.message === '認証に失敗しました') {
      // トークンリフレッシュロジック
      // socket.auth = { token: newToken }
      // socket.connect()
    }
  })

  // 接続状態復旧
  socket.on('connect', () => {
    if (socket.recovered) {
      console.info('Session recovered - missed events were received')
    } else {
      console.info('New session - fetch missed data from API')
    }
  })

  return socket
}
```

## React Hookパターン

```typescript
// hooks/useSocket.ts
import { useEffect, useRef, useCallback } from 'react'
import { Socket } from 'socket.io-client'
import { createSocketClient } from '../lib/socket-client'

export function useSocket(url: string, token: string) {
  const socketRef = useRef<Socket | null>(null)

  useEffect(() => {
    const socket = createSocketClient({
      url,
      token,
      onConnect: () => console.info('Connected'),
      onDisconnect: (reason) => console.warn('Disconnected:', reason),
    })
    socketRef.current = socket

    return () => {
      socket.disconnect()
    }
  }, [url, token])

  const emit = useCallback(<E extends keyof ClientEvents>(
    event: E,
    ...args: Parameters<ClientEvents[E]>
  ) => {
    socketRef.current?.emit(event, ...args)
  }, [])

  const on = useCallback(<E extends keyof ServerEvents>(
    event: E,
    handler: ServerEvents[E]
  ) => {
    socketRef.current?.on(event as string, handler as (...args: unknown[]) => void)
    return () => {
      socketRef.current?.off(event as string, handler as (...args: unknown[]) => void)
    }
  }, [])

  return { emit, on, socket: socketRef }
}
```

## ベストプラクティス

1. **型安全なイベント**: ServerEvents / ClientEvents で全イベントを型定義する
2. **ACKパターン**: 重要な操作にはコールバックで成否を返す
3. **バリデーション**: 全受信データをZodで検証する（クライアントは信用しない）
4. **プレゼンス管理**: Redis + TTLで正確なオンライン状態を管理する
5. **Redis Adapter**: 複数サーバー構成では必須
6. **再接続戦略**: 指数バックオフ + jitter + 最大試行回数
7. **メッセージサイズ制限**: `maxHttpBufferSize` でDoSを防ぐ
8. **Connection State Recovery**: 短時間の切断なら自動復旧する

## アンチパターン

- 認証なしでWebSocket接続を許可する
- 全データをブロードキャストする（ルームで分離せよ）
- クライアントからのデータをバリデーションなしで使う
- 再接続ロジックを実装しない（ネットワークは不安定）
- WebSocketだけに依存する（ポーリングフォールバック必須）
- 大きなペイロードをWebSocketで送る（ファイルはREST API経由で）
