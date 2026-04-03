# SDD-11: 安全与限流

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-03: 公共工具层](SDD-03-公共工具层.md)
> - [SDD-05: 用户模块](SDD-05-用户模块.md)
> **目标**: 创建限流中间件、CORS 配置

---

## 1. 概述

本任务负责创建安全相关组件，包括：
- 限流中间件
- CORS 中间件
- 安全响应处理

---

## 2. 限流中间件 (app/middleware/rate_limiter.py)

```python
# app/middleware/rate_limiter.py

"""限流中间件"""

import os
from fastapi import Request, Response
from fastapi.responses import JSONResponse
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from starlette.middleware.base import BaseHTTPMiddleware

from app.config import get_settings

settings = get_settings()


def get_client_ip(request: Request) -> str:
    """获取客户端 IP"""
    forwarded = request.headers.get("X-Forwarded-For")
    if forwarded:
        return forwarded.split(",")[0].strip()
    return request.client.host if request.client else "unknown"


# 全局限流器
limiter = Limiter(
    key_func=get_client_ip,
    storage_uri=settings.redis.REDIS_URL if hasattr(settings, 'redis') else None,
    strategy="fixed-window"
)


class RateLimitMiddleware(BaseHTTPMiddleware):
    """自定义限流中间件"""

    async def dispatch(self, request: Request, call_next):
        # 不对认证端点限流
        if request.url.path in ["/api/auth/login", "/api/auth/register"]:
            return await call_next(request)

        try:
            response = await call_next(request)
            return response
        except RateLimitExceeded:
            return JSONResponse(
                status_code=429,
                content={
                    "code": 429,
                    "message": "请求过于频繁，请稍后再试",
                    "detail": "Rate limit exceeded"
                }
            )


def rate_limit_dependency(limit_string: str):
    """限流依赖装饰器

    用法:
        @router.post("/items")
        @limiter.limit("10/minute")
        async def create_item(request: Request):
            ...
    """
    def decorator(func):
        func.__rate_limit__ = limit_string
        return func
    return decorator
```

---

## 3. CORS 中间件 (app/middleware/cors.py)

```python
# app/middleware/cors.py

"""CORS 中间件配置"""

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings

settings = get_settings()


def setup_cors(app: FastAPI) -> None:
    """配置 CORS 中间件"""

    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.app.CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

---

## 4. 安全响应 (app/middleware/security.py)

```python
# app/middleware/security.py

"""安全相关中间件"""

from fastapi import Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
import time


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """安全响应头中间件"""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # 添加安全响应头
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        return response


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """请求日志中间件"""

    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        response = await call_next(request)

        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)

        # 记录日志
        print(f"{request.method} {request.url.path} - {response.status_code} - {process_time:.3f}s")

        return response
```

---

## 5. 异常处理 (app/middleware/exception_handler.py)

```python
# app/middleware/exception_handler.py

"""全局异常处理"""

from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.exceptions import (
    AppException,
    BusinessException,
    AuthenticationException,
    ResourceNotFoundException,
)


async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    """应用异常处理"""
    status_code = status.HTTP_400_BAD_REQUEST

    if isinstance(exc, AuthenticationException):
        status_code = status.HTTP_401_UNAUTHORIZED
    elif isinstance(exc, ResourceNotFoundException):
        status_code = status.HTTP_404_NOT_FOUND
    elif isinstance(exc, BusinessException):
        status_code = status.HTTP_400_BAD_REQUEST

    return JSONResponse(
        status_code=status_code,
        content={
            "code": exc.code,
            "message": exc.message,
        }
    )


async def http_exception_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
    """HTTP 异常处理"""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "code": exc.status_code,
            "message": exc.detail,
        }
    )


async def validation_exception_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    """请求验证异常处理"""
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "code": 422,
            "message": "请求参数验证失败",
            "detail": exc.errors(),
        }
    )


async def generic_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    """通用异常处理"""
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "code": 500,
            "message": "服务器内部错误",
        }
    )
```

---

## 6. 单元测试

```python
# tests/test_rate_limiter.py

import pytest
from unittest.mock import MagicMock, patch

from app.middleware.rate_limiter import get_client_ip


class TestRateLimiter:
    """限流测试"""

    def test_get_client_ip_from_forwarded(self):
        """测试从 X-Forwarded-For 获取 IP"""
        mock_request = MagicMock()
        mock_request.headers.get.return_value = "203.0.113.195, 70.41.3.18, 150.172.238.178"
        mock_request.client.host = "127.0.0.1"

        ip = get_client_ip(mock_request)

        assert ip == "203.0.113.195"

    def test_get_client_ip_from_client(self):
        """测试从 client.host 获取 IP"""
        mock_request = MagicMock()
        mock_request.headers.get.return_value = None
        mock_request.client.host = "192.168.1.1"

        ip = get_client_ip(mock_request)

        assert ip == "192.168.1.1"
```

---

## 7. 验收标准

1. ✅ 限流中间件正确实现
2. ✅ CORS 配置正确
3. ✅ 安全响应头添加
4. ✅ 全局异常处理
5. ✅ 单元测试覆盖
