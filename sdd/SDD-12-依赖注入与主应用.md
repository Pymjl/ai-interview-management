# SDD-12: 依赖注入与主应用

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-05 ~ SDD-11](./)
> **目标**: 创建主应用入口、路由注册、依赖注入整合

---

## 1. 概述

本任务负责创建主应用入口和整合所有模块，包括：
- 主应用入口 (app/main.py)
- 路由注册
- 启动和关闭事件处理

---

## 2. 主应用入口 (app/main.py)

```python
# app/main.py

"""FastAPI 主应用入口"""

import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.config import get_settings
from app.database import init_db, close_db
from app.services.redis_client import get_redis_client, close_redis
from app.middleware.cors import setup_cors
from app.middleware.rate_limiter import RateLimitMiddleware
from app.middleware.security import SecurityHeadersMiddleware, RequestLoggingMiddleware
from app.middleware.exception_handler import (
    app_exception_handler,
    http_exception_handler,
    validation_exception_handler,
    generic_exception_handler,
)
from app.exceptions import AppException

# 路由
from app.api.auth import router as auth_router
from app.api.resume import router as resume_router
from app.api.interview import router as interview_router
from app.api.knowledgebase import router as knowledgebase_router

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理"""
    # 启动
    logger.info("Starting up...")

    # 初始化数据库
    try:
        await init_db()
        logger.info("Database initialized")
    except Exception as e:
        logger.error(f"Database initialization failed: {e}")

    # 初始化 Redis
    try:
        redis = await get_redis_client()
        await redis.get_client()
        logger.info("Redis connected")
    except Exception as e:
        logger.error(f"Redis connection failed: {e}")

    yield

    # 关闭
    logger.info("Shutting down...")
    await close_db()
    await close_redis()
    logger.info("Cleanup completed")


def create_app() -> FastAPI:
    """创建 FastAPI 应用"""
    app = FastAPI(
        title=settings.app.APP_NAME,
        version=settings.app.APP_VERSION,
        lifespan=lifespan,
    )

    # 添加中间件（按顺序）
    app.add_middleware(SecurityHeadersMiddleware)
    app.add_middleware(RequestLoggingMiddleware)
    app.add_middleware(RateLimitMiddleware)

    # 配置 CORS
    setup_cors(app)

    # 注册异常处理器
    app.add_exception_handler(AppException, app_exception_handler)
    app.add_exception_handler(StarletteHTTPException, http_exception_handler)
    app.add_exception_handler(RequestValidationError, validation_exception_handler)
    app.add_exception_handler(Exception, generic_exception_handler)

    # 注册路由
    app.include_router(auth_router)
    app.include_router(resume_router)
    app.include_router(interview_router)
    app.include_router(knowledgebase_router)

    # 健康检查端点
    @app.get("/health")
    async def health_check():
        return {"status": "healthy", "version": settings.app.APP_VERSION}

    @app.get("/")
    async def root():
        return {
            "name": settings.app.APP_NAME,
            "version": settings.app.APP_VERSION,
            "docs": "/docs",
        }

    return app


# 创建应用实例
app = create_app()


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        "app.main:app",
        host="0.0.0.0",
        port=8080,
        reload=settings.app.DEBUG,
    )
```

---

## 3. 应用导出 (app/__init__.py)

```python
# app/__init__.py

"""Interview Guide - AI驱动的面试平台后端服务"""

__version__ = "1.0.0"
__app_name__ = "interview-guide"

from app.main import app, create_app

__all__ = ["app", "create_app", "__version__", "__app_name__"]
```

---

## 4. 依赖注入整合 (app/deps.py)

```python
# app/deps.py

"""依赖注入

整合所有模块的依赖注入函数
"""

from typing import AsyncGenerator
from functools import lru_cache
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db_session
from app.models.user import User
from app.services.auth_service import AuthService
from app.services.user_service import UserService
from app.services.session_service import SessionService
from app.services.resume_service import ResumeService
from app.services.interview_service import InterviewSessionService
from app.services.knowledgebase_service import KnowledgeBaseService
from app.services.minio_service import get_minio_service
from app.utils.jwt import get_jwt_manager, JWTManager
from app.utils.hash_util import get_hash_util, HashUtil

security = HTTPBearer(auto_error=False)


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db_session),
    jwt_manager: JWTManager = Depends(get_jwt_manager),
) -> User:
    """获取当前登录用户"""
    if not credentials:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="未提供认证凭证",
            headers={"WWW-Authenticate": "Bearer"},
        )

    token = credentials.credentials
    payload = jwt_manager.verify_token(token, expected_type="access")

    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token 无效或已过期",
        )

    session_service = SessionService()
    if await session_service.is_blacklisted(payload.jti):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token 已失效",
        )

    user_service = UserService(db)
    user = await user_service.get_user_by_id(int(payload.sub))

    if not user or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="用户不存在或已禁用",
        )

    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user),
) -> User:
    """获取当前活跃用户"""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="用户账号已被禁用",
        )
    return current_user


# 服务工厂函数

def get_auth_service(db: AsyncSession = Depends(get_db_session)) -> AuthService:
    return AuthService(db)


def get_user_service(db: AsyncSession = Depends(get_db_session)) -> UserService:
    return UserService(db)


def get_resume_service(db: AsyncSession = Depends(get_db_session)) -> ResumeService:
    return ResumeService(db)


def get_interview_service(db: AsyncSession = Depends(get_db_session)) -> InterviewSessionService:
    return InterviewSessionService(db)


def get_knowledgebase_service(db: AsyncSession = Depends(get_db_session)) -> KnowledgeBaseService:
    return KnowledgeBaseService(db)


def get_session_service() -> SessionService:
    return SessionService()
```

---

## 5. 验收标准

1. ✅ 主应用正确创建和配置
2. ✅ 所有中间件正确注册
3. ✅ 所有路由正确注册
4. ✅ 生命周期管理正确（启动/关闭）
5. ✅ 全局异常处理正确
6. ✅ 健康检查端点可用
