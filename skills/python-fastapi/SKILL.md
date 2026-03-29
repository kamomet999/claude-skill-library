---
name: python-fastapi
description: FastAPI設計パターン。依存性注入、Pydanticモデル、非同期処理、ミドルウェア、OpenAPI活用
category: バックエンド
command: /fastapi
version: 1.0.0
tags:
  - python
  - fastapi
  - pydantic
  - async
  - openapi
---

# FastAPI Design Patterns

FastAPIを使ったPython Webアプリケーションの設計・実装パターン集。

## When to Activate

- FastAPIプロジェクトを新規セットアップするとき
- Pydanticモデルやバリデーションを設計するとき
- 依存性注入パターンを実装するとき
- 非同期処理やバックグラウンドタスクを構築するとき
- OpenAPI/Swagger文書を整備するとき

## Steps

### Step 1: プロジェクトセットアップ

```bash
# uv（推奨）
uv init my-api
cd my-api
uv add fastapi uvicorn[standard] pydantic-settings sqlalchemy[asyncio] asyncpg alembic

# or pip
pip install "fastapi[standard]" pydantic-settings sqlalchemy[asyncio] asyncpg alembic
```

### Step 2: ディレクトリ構成

```
src/
├── main.py                  # アプリケーションエントリポイント
├── config.py                # 設定管理
├── database.py              # DB接続
├── dependencies.py          # 共通依存性
├── middleware.py             # ミドルウェア
├── models/                  # SQLAlchemyモデル
│   ├── __init__.py
│   └── user.py
├── schemas/                 # Pydanticスキーマ
│   ├── __init__.py
│   ├── user.py
│   └── common.py
├── routers/                 # エンドポイント
│   ├── __init__.py
│   ├── users.py
│   └── auth.py
├── services/                # ビジネスロジック
│   ├── __init__.py
│   └── user_service.py
├── repositories/            # データアクセス
│   ├── __init__.py
│   └── user_repo.py
└── utils/                   # ユーティリティ
    ├── __init__.py
    └── security.py
alembic/                     # マイグレーション
tests/
pyproject.toml
```

### Step 3: 設定管理

```python
# src/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    app_name: str = "My API"
    debug: bool = False
    database_url: str
    redis_url: str = "redis://localhost:6379"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 30
    cors_origins: list[str] = ["http://localhost:3000"]

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

### Step 4: Pydanticスキーマ

```python
# src/schemas/common.py
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")


class ApiResponse(BaseModel, Generic[T]):
    success: bool = True
    data: T
    message: str | None = None


class PaginatedResponse(BaseModel, Generic[T]):
    success: bool = True
    data: list[T]
    meta: "PaginationMeta"


class PaginationMeta(BaseModel):
    total: int
    page: int
    limit: int
    pages: int


# src/schemas/user.py
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime
from uuid import UUID


class UserCreate(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    password: str = Field(min_length=8, max_length=128)


class UserUpdate(BaseModel):
    name: str | None = Field(None, min_length=1, max_length=100)
    email: EmailStr | None = None


class UserResponse(BaseModel):
    id: UUID
    email: str
    name: str
    role: str
    created_at: datetime

    model_config = {"from_attributes": True}


class UserLogin(BaseModel):
    email: EmailStr
    password: str
```

### Step 5: 依存性注入

```python
# src/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from jose import JWTError, jwt

from .config import get_settings, Settings
from .database import get_session

security = HTTPBearer()


async def get_db() -> AsyncSession:
    async with get_session() as session:
        yield session


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    settings: Settings = Depends(get_settings),
    db: AsyncSession = Depends(get_db),
) -> "User":
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.jwt_secret,
            algorithms=[settings.jwt_algorithm],
        )
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

    user = await UserRepository(db).find_by_id(user_id)
    if user is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return user


def require_role(*roles: str):
    async def role_checker(
        current_user=Depends(get_current_user),
    ):
        if current_user.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user

    return role_checker
```

### Step 6: ルーター

```python
# src/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, Query, status
from uuid import UUID

from ..schemas.user import UserCreate, UserUpdate, UserResponse
from ..schemas.common import ApiResponse, PaginatedResponse
from ..services.user_service import UserService
from ..dependencies import get_db, get_current_user, require_role

router = APIRouter(prefix="/users", tags=["Users"])


@router.post(
    "",
    response_model=ApiResponse[UserResponse],
    status_code=status.HTTP_201_CREATED,
    summary="ユーザー作成",
)
async def create_user(
    body: UserCreate,
    db=Depends(get_db),
):
    service = UserService(db)
    user = await service.create(body)
    return ApiResponse(data=UserResponse.model_validate(user))


@router.get(
    "",
    response_model=PaginatedResponse[UserResponse],
    summary="ユーザー一覧",
)
async def list_users(
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100),
    db=Depends(get_db),
    _=Depends(require_role("admin")),
):
    service = UserService(db)
    users, total = await service.list(page=page, limit=limit)
    return PaginatedResponse(
        data=[UserResponse.model_validate(u) for u in users],
        meta={
            "total": total,
            "page": page,
            "limit": limit,
            "pages": (total + limit - 1) // limit,
        },
    )


@router.get(
    "/{user_id}",
    response_model=ApiResponse[UserResponse],
    summary="ユーザー詳細",
)
async def get_user(
    user_id: UUID,
    db=Depends(get_db),
    current_user=Depends(get_current_user),
):
    service = UserService(db)
    user = await service.get_by_id(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return ApiResponse(data=UserResponse.model_validate(user))


@router.patch(
    "/{user_id}",
    response_model=ApiResponse[UserResponse],
    summary="ユーザー更新",
)
async def update_user(
    user_id: UUID,
    body: UserUpdate,
    db=Depends(get_db),
    current_user=Depends(get_current_user),
):
    if str(current_user.id) != str(user_id) and current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Cannot update other users")

    service = UserService(db)
    user = await service.update(user_id, body)
    return ApiResponse(data=UserResponse.model_validate(user))
```

### Step 7: アプリケーション組み立て

```python
# src/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .config import get_settings
from .routers import users, auth
from .middleware import RequestIdMiddleware, LoggingMiddleware


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    settings = get_settings()
    print(f"Starting {settings.app_name}")
    yield
    # Shutdown
    print("Shutting down")


def create_app() -> FastAPI:
    settings = get_settings()

    app = FastAPI(
        title=settings.app_name,
        lifespan=lifespan,
        docs_url="/docs" if settings.debug else None,
        redoc_url="/redoc" if settings.debug else None,
    )

    # Middleware（逆順に実行される）
    app.add_middleware(LoggingMiddleware)
    app.add_middleware(RequestIdMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Routers
    app.include_router(auth.router, prefix="/api")
    app.include_router(users.router, prefix="/api")

    @app.get("/health")
    async def health():
        return {"status": "healthy"}

    return app


app = create_app()
```

## ミドルウェア

```python
# src/middleware.py
import time
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request


class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response


class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start

        print(
            f"{request.method} {request.url.path} "
            f"status={response.status_code} "
            f"duration={duration:.3f}s"
        )
        return response
```

## エラーハンドリング

```python
# src/main.py に追加
from fastapi import Request
from fastapi.responses import JSONResponse


@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
            }
        },
    )


@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception):
    print(f"Unhandled error: {exc}")
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "Internal server error",
            }
        },
    )
```

## バックグラウンドタスク

```python
from fastapi import BackgroundTasks


async def send_welcome_email(email: str, name: str):
    # メール送信処理
    pass


@router.post("/users", status_code=201)
async def create_user(
    body: UserCreate,
    background_tasks: BackgroundTasks,
    db=Depends(get_db),
):
    service = UserService(db)
    user = await service.create(body)
    background_tasks.add_task(send_welcome_email, user.email, user.name)
    return ApiResponse(data=UserResponse.model_validate(user))
```

## テスト

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from src.main import create_app


@pytest.fixture
async def client():
    app = create_app()
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac


# tests/test_users.py
import pytest


@pytest.mark.anyio
async def test_create_user(client: AsyncClient):
    response = await client.post("/api/users", json={
        "email": "test@example.com",
        "name": "Test User",
        "password": "securepassword123",
    })
    assert response.status_code == 201
    data = response.json()
    assert data["success"] is True
    assert data["data"]["email"] == "test@example.com"


@pytest.mark.anyio
async def test_create_user_validation_error(client: AsyncClient):
    response = await client.post("/api/users", json={
        "email": "invalid",
        "name": "",
        "password": "short",
    })
    assert response.status_code == 422
```

## Best Practices

| Practice | Do | Don't |
|----------|-----|-------|
| バリデーション | Pydanticモデルで定義 | 手動バリデーション |
| 設定 | pydantic-settings + .env | ハードコード |
| DB接続 | asyncセッション + DI | グローバル変数 |
| テスト | httpx.AsyncClient | requests |
| 型ヒント | 全関数に付与 | Any多用 |
| ドキュメント | OpenAPIアノテーション | 別途Swagger作成 |

## Related

- Skill: `error-handling-patterns` - エラーハンドリング設計
- Skill: `api-testing` - APIテスト
- Skill: `drizzle-orm` - ORM設計（TypeScript版）
