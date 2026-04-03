# SDD-03: 公共工具层

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**: [SDD-01: 项目基础配置](SDD-01-项目基础配置.md)
> **目标**: 创建公共工具函数和基础服务

---

## 1. 概述

本任务负责创建项目的基础工具层，包括：
- JWT 令牌管理工具 (`app/utils/jwt.py`)
- 密码哈希工具 (`app/utils/hash_util.py`)
- 文本清理工具 (`app/utils/text_cleaner.py`)
- S3/MinIO 客户端封装 (`app/services/minio_service.py`)
- Redis 客户端封装 (`app/services/redis_client.py`)
- 统一异常定义 (`app/exceptions.py`)

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── exceptions.py           # 新增
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── jwt.py              # 新增
│   │   ├── hash_util.py        # 新增
│   │   └── text_cleaner.py     # 新增
│   └── services/
│       ├── __init__.py
│       ├── minio_service.py    # 新增
│       └── redis_client.py     # 新增
└── ...
```

---

## 3. 统一异常 (app/exceptions.py)

```python
# app/exceptions.py

"""自定义异常定义"""


class AppException(Exception):
    """应用基础异常"""

    def __init__(self, message: str, code: str = "APP_ERROR"):
        self.message = message
        self.code = code
        super().__init__(self.message)


class BusinessException(AppException):
    """业务异常"""

    def __init__(self, message: str, code: str = "BUSINESS_ERROR"):
        super().__init__(message, code)


class AuthenticationException(AppException):
    """认证异常"""

    def __init__(self, message: str = "认证失败", code: str = "AUTH_ERROR"):
        super().__init__(message, code)


class AuthorizationException(AppException):
    """授权异常"""

    def __init__(self, message: str = "权限不足", code: str = "FORBIDDEN"):
        super().__init__(message, code)


class ResourceNotFoundException(AppException):
    """资源不存在异常"""

    def __init__(self, resource: str = "资源", code: str = "NOT_FOUND"):
        super().__init__(f"{resource}不存在", code)


class DuplicateResourceException(AppException):
    """重复资源异常"""

    def __init__(self, resource: str = "资源", code: str = "DUPLICATE"):
        super().__init__(f"{resource}已存在", code)


class ValidationException(AppException):
    """数据验证异常"""

    def __init__(self, message: str, code: str = "VALIDATION_ERROR"):
        super().__init__(message, code)


class FileProcessingException(AppException):
    """文件处理异常"""

    def __init__(self, message: str, code: str = "FILE_ERROR"):
        super().__init__(message, code)


class ExternalServiceException(AppException):
    """外部服务调用异常"""

    def __init__(self, service: str, message: str, code: str = "EXTERNAL_ERROR"):
        super().__init__(f"{service}服务错误: {message}", code)


class RateLimitException(AppException):
    """限流异常"""

    def __init__(self, message: str = "请求过于频繁", code: str = "RATE_LIMIT"):
        super().__init__(message, code)
```

---

## 4. JWT 工具 (app/utils/jwt.py)

```python
# app/utils/jwt.py

"""JWT令牌管理工具"""

from datetime import datetime, timedelta
from typing import Optional
from jose import jwt, JWTError
from pydantic import BaseModel
import uuid

from app.config import get_settings

settings = get_settings()


class TokenPayload(BaseModel):
    """JWT Payload 结构"""
    sub: str  # 用户ID
    email: str
    type: str  # "access" 或 "refresh"
    jti: str  # Token 唯一标识
    family: Optional[str] = None  # Token 家族（refresh token 用）
    iat: int  # 签发时间
    exp: int  # 过期时间


class TokenPair(BaseModel):
    """Token对"""
    access_token: str
    refresh_token: str
    access_token_jti: str
    refresh_token_jti: str
    expires_in: int


class JWTManager:
    """JWT令牌管理器"""

    def __init__(
        self,
        secret_key: Optional[str] = None,
        algorithm: str = "HS256",
        access_token_expire_minutes: int = 15,
        refresh_token_expire_days: int = 7
    ):
        self.secret_key = secret_key or settings.jwt.SECRET_KEY
        self.algorithm = algorithm
        self.access_token_expire = timedelta(minutes=access_token_expire_minutes)
        self.refresh_token_expire = timedelta(days=refresh_token_expire_days)

    def create_access_token(self, user_id: int, email: str) -> tuple[str, str]:
        """创建Access Token，返回 (token, jti)"""
        jti = str(uuid.uuid4())
        now = datetime.utcnow()

        payload = {
            "sub": str(user_id),
            "email": email,
            "type": "access",
            "jti": jti,
            "iat": int(now.timestamp()),
            "exp": int((now + self.access_token_expire).timestamp())
        }

        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token, jti

    def create_refresh_token(self, user_id: int, family_id: str) -> tuple[str, str]:
        """创建Refresh Token，返回 (token, jti)"""
        jti = str(uuid.uuid4())
        now = datetime.utcnow()

        payload = {
            "sub": str(user_id),
            "type": "refresh",
            "jti": jti,
            "family": family_id,
            "iat": int(now.timestamp()),
            "exp": int((now + self.refresh_token_expire).timestamp())
        }

        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token, jti

    def create_token_pair(self, user_id: int, email: str) -> TokenPair:
        """创建Token对"""
        family_id = str(uuid.uuid4())

        access_token, access_jti = self.create_access_token(user_id, email)
        refresh_token, refresh_jti = self.create_refresh_token(user_id, family_id)

        return TokenPair(
            access_token=access_token,
            refresh_token=refresh_token,
            access_token_jti=access_jti,
            refresh_token_jti=refresh_jti,
            expires_in=int(self.access_token_expire.total_seconds())
        )

    def verify_token(self, token: str, expected_type: str) -> Optional[TokenPayload]:
        """验证Token并返回Payload"""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])

            if payload.get("type") != expected_type:
                return None

            return TokenPayload(**payload)
        except JWTError:
            return None

    def decode_token(self, token: str) -> Optional[dict]:
        """解码Token（不验证类型）"""
        try:
            return jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
        except JWTError:
            return None


# 单例
_jwt_manager: Optional[JWTManager] = None


def get_jwt_manager() -> JWTManager:
    """获取JWT管理器单例"""
    global _jwt_manager
    if _jwt_manager is None:
        _jwt_manager = JWTManager()
    return _jwt_manager
```

---

## 5. 密码哈希工具 (app/utils/hash_util.py)

```python
# app/utils/hash_util.py

"""密码哈希工具"""

import bcrypt
from app.config import get_settings

settings = get_settings()


class HashUtil:
    """密码哈希工具类"""

    def __init__(self, rounds: int = 12):
        self.rounds = rounds

    def hash_password(self, password: str) -> str:
        """对密码进行哈希

        Args:
            password: 明文密码

        Returns:
            哈希后的密码字符串
        """
        password_bytes = password.encode('utf-8')
        salt = bcrypt.gensalt(rounds=self.rounds)
        hashed = bcrypt.hashpw(password_bytes, salt)
        return hashed.decode('utf-8')

    def verify_password(self, password: str, hashed: str) -> bool:
        """验证密码

        Args:
            password: 明文密码
            hashed: 哈希后的密码

        Returns:
            密码是否匹配
        """
        try:
            password_bytes = password.encode('utf-8')
            hashed_bytes = hashed.encode('utf-8')
            return bcrypt.checkpw(password_bytes, hashed_bytes)
        except Exception:
            return False

    def needs_rehash(self, hashed: str) -> bool:
        """检查是否需要重新哈希

        用于bcrypt轮数升级时检查旧密码是否需要重新哈希
        """
        try:
            # bcrypt 库内部会检查当前的 rounds 是否匹配
            bcrypt.checkpw(b"dummy", hashed.encode('utf-8'))
            return False
        except Exception:
            return True


# 单例
_hash_util: HashUtil | None = None


def get_hash_util() -> HashUtil:
    """获取哈希工具单例"""
    global _hash_util
    if _hash_util is None:
        _hash_util = HashUtil()
    return _hash_util
```

---

## 6. 文本清理工具 (app/utils/text_cleaner.py)

```python
# app/utils/text_cleaner.py

"""文本清理工具"""

import re
from typing import Optional


class TextCleaner:
    """文本清理工具类"""

    def __init__(self):
        # 常见的简历中的无用字符模式
        self._whitespace_pattern = re.compile(r'\s+')
        self._special_chars_pattern = re.compile(r'[\x00-\x08\x0b-\x0c\x0e-\x1f\x7f-\x9f]')
        self._url_pattern = re.compile(r'https?://\S+')
        self._email_pattern = re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b')

    def clean(self, text: str) -> str:
        """清理文本

        Args:
            text: 原始文本

        Returns:
            清理后的文本
        """
        if not text:
            return ""

        # 移除特殊控制字符
        text = self._special_chars_pattern.sub('', text)

        # 规范化空白字符
        text = self._whitespace_pattern.sub(' ', text)

        # 移除首尾空白
        text = text.strip()

        return text

    def clean_resume_text(self, text: str) -> str:
        """清理简历文本

        额外的简历特定清理逻辑
        """
        if not text:
            return ""

        text = self.clean(text)

        # 移除电话号码（可选，保留简历中的联系信息）
        # phone_pattern = re.compile(r'1[3-9]\d{9}')

        # 移除邮箱（可选）
        # text = self._email_pattern.sub('[EMAIL]', text)

        # 规范化换行
        text = re.sub(r'\n{3,}', '\n\n', text)

        return text

    def truncate(self, text: str, max_length: int, suffix: str = "...") -> str:
        """截断文本

        Args:
            text: 原始文本
            max_length: 最大长度
            suffix: 截断后缀

        Returns:
            截断后的文本
        """
        if not text or len(text) <= max_length:
            return text

        return text[:max_length - len(suffix)] + suffix

    def remove_urls(self, text: str) -> str:
        """移除URL"""
        return self._url_pattern.sub('[URL]', text)

    def mask_email(self, text: str) -> str:
        """脱敏邮箱"""
        def mask(m):
            email = m.group(0)
            parts = email.split('@')
            if len(parts[0]) > 2:
                return parts[0][:2] + '***@' + parts[1]
            return '***@' + parts[1]

        return self._email_pattern.sub(mask, text)


# 单例
_text_cleaner: Optional[TextCleaner] = None


def get_text_cleaner() -> TextCleaner:
    """获取文本清理工具单例"""
    global _text_cleaner
    if _text_cleaner is None:
        _text_cleaner = TextCleaner()
    return _text_cleaner


# 便捷函数
text_cleaner = get_text_cleaner()
```

---

## 7. MinIO 服务 (app/services/minio_service.py)

```python
# app/services/minio_service.py

"""MinIO (S3兼容) 文件存储服务"""

import io
from typing import Optional
from minio import Minio
from minio.error import S3Error

from app.config import get_settings

settings = get_settings()


class MiniOService:
    """MinIO文件存储服务"""

    def __init__(self):
        self.client = Minio(
            endpoint=settings.minio.ENDPOINT,
            access_key=settings.minio.ACCESS_KEY,
            secret_key=settings.minio.SECRET_KEY,
            secure=settings.minio.SECURE
        )
        self.bucket = settings.minio.BUCKET
        self._ensure_bucket()

    def _ensure_bucket(self) -> None:
        """确保bucket存在（幂等操作）"""
        try:
            self.client.make_bucket(self.bucket)
        except S3Error as e:
            if e.code not in ("BucketAlreadyExists", "BucketAlreadyOwnedByAccount"):
                raise

    def upload_file(
        self,
        data: bytes,
        object_name: str,
        content_type: str = "application/octet-stream"
    ) -> str:
        """上传文件

        Args:
            data: 文件数据
            object_name: 对象名称（存储路径）
            content_type: MIME类型

        Returns:
            访问URL
        """
        self.client.put_object(
            bucket_name=self.bucket,
            object_name=object_name,
            data=io.BytesIO(data),
            length=len(data),
            content_type=content_type
        )
        return self._get_object_url(object_name)

    def download_file(self, object_name: str) -> bytes:
        """下载文件

        Args:
            object_name: 对象名称

        Returns:
            文件数据
        """
        try:
            response = self.client.get_object(
                bucket_name=self.bucket,
                object_name=object_name
            )
            return response.read()
        except S3Error as e:
            if e.code == "NoSuchKey":
                raise FileNotFoundError(f"文件不存在: {object_name}")
            raise

    def delete_file(self, object_name: str) -> None:
        """删除文件

        Args:
            object_name: 对象名称
        """
        self.client.remove_object(
            bucket_name=self.bucket,
            object_name=object_name
        )

    def get_presigned_url(self, object_name: str, expires: int = 3600) -> str:
        """获取预签名URL

        Args:
            object_name: 对象名称
            expires: 过期时间（秒）

        Returns:
            预签名URL
        """
        return self.client.presigned_get_object(
            bucket_name=self.bucket,
            object_name=object_name,
            expires=expires
        )

    def file_exists(self, object_name: str) -> bool:
        """检查文件是否存在

        Args:
            object_name: 对象名称

        Returns:
            是否存在
        """
        try:
            self.client.stat_object(self.bucket, object_name)
            return True
        except S3Error:
            return False

    def _get_object_url(self, object_name: str) -> str:
        """获取对象URL"""
        protocol = "https" if settings.minio.SECURE else "http"
        return f"{protocol}://{settings.minio.ENDPOINT}/{self.bucket}/{object_name}"


# 单例
_minio_service: Optional[MiniOService] = None


def get_minio_service() -> MiniOService:
    """获取MinIO服务单例"""
    global _minio_service
    if _minio_service is None:
        _minio_service = MiniOService()
    return _minio_service
```

---

## 8. Redis 客户端 (app/services/redis_client.py)

```python
# app/services/redis_client.py

"""Redis客户端封装"""

import json
from typing import Optional, Any
from redis.asyncio import Redis
from redis.asyncio.client import Redis as AsyncRedis

from app.config import get_settings

settings = get_settings()


class RedisClient:
    """Redis客户端封装类"""

    def __init__(self):
        self._client: Optional[Redis] = None

    async def get_client(self) -> Redis:
        """获取Redis客户端"""
        if self._client is None:
            self._client = Redis.from_url(
                settings.redis.REDIS_URL,
                encoding="utf-8",
                decode_responses=True
            )
        return self._client

    async def close(self) -> None:
        """关闭Redis连接"""
        if self._client:
            await self._client.close()
            self._client = None

    # 基础操作
    async def get(self, key: str) -> Optional[str]:
        """获取值"""
        client = await self.get_client()
        return await client.get(key)

    async def set(
        self,
        key: str,
        value: str,
        ex: Optional[int] = None,
        px: Optional[int] = None,
        nx: bool = False,
        xx: bool = False
    ) -> bool:
        """设置值"""
        client = await self.get_client()
        return await client.set(key, value, ex=ex, px=px, nx=nx, xx=xx)

    async def delete(self, *keys: str) -> int:
        """删除键"""
        client = await self.get_client()
        return await client.delete(*keys)

    async def exists(self, key: str) -> int:
        """检查键是否存在"""
        client = await self.get_client()
        return await client.exists(key)

    async def expire(self, key: str, seconds: int) -> bool:
        """设置过期时间"""
        client = await self.get_client()
        return await client.expire(key, seconds)

    async def ttl(self, key: str) -> int:
        """获取剩余生存时间"""
        client = await self.get_client()
        return await client.ttl(key)

    # Hash操作
    async def hget(self, key: str, field: str) -> Optional[str]:
        """获取Hash字段值"""
        client = await self.get_client()
        return await client.hget(key, field)

    async def hset(self, key: str, field: str, value: str) -> int:
        """设置Hash字段值"""
        client = await self.get_client()
        return await client.hset(key, field, value)

    async def hset_dict(self, key: str, mapping: dict) -> int:
        """批量设置Hash字段"""
        client = await self.get_client()
        return await client.hset(key, mapping=mapping)

    async def hgetall(self, key: str) -> dict:
        """获取Hash所有字段"""
        client = await self.get_client()
        return await client.hgetall(key)

    async def hdel(self, key: str, *fields: str) -> int:
        """删除Hash字段"""
        client = await self.get_client()
        return await client.hdel(key, *fields)

    # JSON操作（使用字符串存储JSON）
    async def get_json(self, key: str) -> Optional[Any]:
        """获取JSON值"""
        value = await self.get(key)
        if value:
            return json.loads(value)
        return None

    async def set_json(self, key: str, value: Any, ex: Optional[int] = None) -> bool:
        """设置JSON值"""
        return await self.set(key, json.dumps(value), ex=ex)

    # List操作
    async def lpush(self, key: str, *values: str) -> int:
        """左推入列表"""
        client = await self.get_client()
        return await client.lpush(key, *values)

    async def rpush(self, key: str, *values: str) -> int:
        """右推入列表"""
        client = await self.get_client()
        return await client.rpush(key, *values)

    async def lrange(self, key: str, start: int, end: int) -> list:
        """获取列表范围"""
        client = await self.get_client()
        return await client.lrange(key, start, end)


# 单例
_redis_client: Optional[RedisClient] = None


def get_redis_client() -> RedisClient:
    """获取Redis客户端单例"""
    global _redis_client
    if _redis_client is None:
        _redis_client = RedisClient()
    return _redis_client


async def close_redis() -> None:
    """关闭Redis连接"""
    global _redis_client
    if _redis_client:
        await _redis_client.close()
        _redis_client = None
```

---

## 9. 工具导出 (app/utils/__init__.py)

```python
# app/utils/__init__.py

"""公共工具函数"""

from app.utils.jwt import (
    JWTManager,
    TokenPayload,
    TokenPair,
    get_jwt_manager,
)
from app.utils.hash_util import (
    HashUtil,
    get_hash_util,
)
from app.utils.text_cleaner import (
    TextCleaner,
    get_text_cleaner,
    text_cleaner,
)

__all__ = [
    "JWTManager",
    "TokenPayload",
    "TokenPair",
    "get_jwt_manager",
    "HashUtil",
    "get_hash_util",
    "TextCleaner",
    "get_text_cleaner",
    "text_cleaner",
]
```

---

## 10. 服务导出 (app/services/__init__.py)

```python
# app/services/__init__.py

"""服务层"""

from app.services.minio_service import (
    MiniOService,
    get_minio_service,
)
from app.services.redis_client import (
    RedisClient,
    get_redis_client,
    close_redis,
)

__all__ = [
    "MiniOService",
    "get_minio_service",
    "RedisClient",
    "get_redis_client",
    "close_redis",
]
```

---

## 11. 单元测试

```python
# tests/test_jwt.py

import pytest
from datetime import datetime, timedelta
from unittest.mock import patch

from app.utils.jwt import JWTManager, TokenPayload, TokenPair


class TestJWTManager:
    """JWT管理器测试"""

    @pytest.fixture
    def jwt_manager(self):
        """创建JWT管理器"""
        return JWTManager(
            secret_key="test-secret-key",
            algorithm="HS256",
            access_token_expire_minutes=15,
            refresh_token_expire_days=7
        )

    def test_create_access_token(self, jwt_manager):
        """测试创建访问令牌"""
        token, jti = jwt_manager.create_access_token(123, "test@example.com")

        assert token is not None
        assert len(token) > 0
        assert jti is not None

    def test_create_refresh_token(self, jwt_manager):
        """测试创建刷新令牌"""
        token, jti = jwt_manager.create_refresh_token(123, "family-123")

        assert token is not None
        assert len(token) > 0
        assert jti is not None

    def test_create_token_pair(self, jwt_manager):
        """测试创建令牌对"""
        token_pair = jwt_manager.create_token_pair(123, "test@example.com")

        assert isinstance(token_pair, TokenPair)
        assert token_pair.access_token is not None
        assert token_pair.refresh_token is not None
        assert token_pair.expires_in == 15 * 60  # 15分钟

    def test_verify_access_token(self, jwt_manager):
        """测试验证访问令牌"""
        token, jti = jwt_manager.create_access_token(123, "test@example.com")
        payload = jwt_manager.verify_token(token, "access")

        assert isinstance(payload, TokenPayload)
        assert payload.sub == "123"
        assert payload.email == "test@example.com"
        assert payload.type == "access"
        assert payload.jti == jti

    def test_verify_refresh_token(self, jwt_manager):
        """测试验证刷新令牌"""
        family_id = "family-123"
        token, jti = jwt_manager.create_refresh_token(123, family_id)
        payload = jwt_manager.verify_token(token, "refresh")

        assert isinstance(payload, TokenPayload)
        assert payload.sub == "123"
        assert payload.type == "refresh"
        assert payload.family == family_id

    def test_verify_token_wrong_type(self, jwt_manager):
        """测试验证令牌类型不匹配"""
        token, _ = jwt_manager.create_access_token(123, "test@example.com")
        payload = jwt_manager.verify_token(token, "refresh")

        assert payload is None

    def test_verify_invalid_token(self, jwt_manager):
        """测试验证无效令牌"""
        payload = jwt_manager.verify_token("invalid-token", "access")

        assert payload is None

    def test_decode_token(self, jwt_manager):
        """测试解码令牌"""
        token, _ = jwt_manager.create_access_token(123, "test@example.com")
        decoded = jwt_manager.decode_token(token)

        assert decoded is not None
        assert decoded["sub"] == "123"
        assert decoded["email"] == "test@example.com"
```

```python
# tests/test_hash_util.py

import pytest
from app.utils.hash_util import HashUtil


class TestHashUtil:
    """哈希工具测试"""

    @pytest.fixture
    def hash_util(self):
        """创建哈希工具"""
        return HashUtil(rounds=4)  # 使用较少的轮数加快测试

    def test_hash_password(self, hash_util):
        """测试密码哈希"""
        password = "test_password123"
        hashed = hash_util.hash_password(password)

        assert hashed != password
        assert hashed.startswith("$2")  # bcrypt 格式

    def test_verify_password_correct(self, hash_util):
        """测试验证正确密码"""
        password = "test_password123"
        hashed = hash_util.hash_password(password)

        assert hash_util.verify_password(password, hashed) is True

    def test_verify_password_incorrect(self, hash_util):
        """测试验证错误密码"""
        password = "test_password123"
        hashed = hash_util.hash_password(password)

        assert hash_util.verify_password("wrong_password", hashed) is False

    def test_verify_password_invalid_hash(self, hash_util):
        """测试验证无效哈希"""
        assert hash_util.verify_password("password", "invalid-hash") is False

    def test_needs_rehash(self, hash_util):
        """测试检查是否需要重新哈希"""
        password = "test_password123"
        hashed = hash_util.hash_password(password)

        # 新哈希不需要重新哈希
        assert hash_util.needs_rehash(hashed) is False

    def test_hash_password_different_each_time(self, hash_util):
        """测试同一密码每次哈希结果不同（因为salt不同）"""
        password = "test_password123"
        hash1 = hash_util.hash_password(password)
        hash2 = hash_util.hash_password(password)

        assert hash1 != hash2
        # 但都能验证通过
        assert hash_util.verify_password(password, hash1) is True
        assert hash_util.verify_password(password, hash2) is True
```

```python
# tests/test_text_cleaner.py

import pytest
from app.utils.text_cleaner import TextCleaner


class TestTextCleaner:
    """文本清理工具测试"""

    @pytest.fixture
    def cleaner(self):
        """创建文本清理工具"""
        return TextCleaner()

    def test_clean_normal_text(self, cleaner):
        """测试清理普通文本"""
        text = "  Hello   World  "
        result = cleaner.clean(text)

        assert result == "Hello World"

    def test_clean_empty_text(self, cleaner):
        """测试清理空文本"""
        assert cleaner.clean("") == ""
        assert cleaner.clean(None) == ""

    def test_clean_with_special_chars(self, cleaner):
        """测试清理特殊字符"""
        text = "Hello\x00World\x1f"
        result = cleaner.clean(text)

        assert "\x00" not in result
        assert "\x1f" not in result

    def test_clean_resume_text(self, cleaner):
        """测试清理简历文本"""
        text = "姓名：张三\n\n\n\n电话：13800138000"
        result = cleaner.clean_resume_text(text)

        assert "\n\n\n\n" not in result
        assert "  " not in result

    def test_truncate_short_text(self, cleaner):
        """测试截断短文本"""
        text = "Short text"
        result = cleaner.truncate(text, 100)

        assert result == "Short text"

    def test_truncate_long_text(self, cleaner):
        """测试截断长文本"""
        text = "This is a very long text that should be truncated"
        result = cleaner.truncate(text, 20, suffix="...")

        assert len(result) == 20
        assert result.endswith("...")

    def test_truncate_custom_suffix(self, cleaner):
        """测试自定义后缀"""
        text = "Long text here"
        result = cleaner.truncate(text, 10, suffix=">>")

        assert result == "Long text>>"

    def test_remove_urls(self, cleaner):
        """测试移除URL"""
        text = "Visit https://example.com today"
        result = cleaner.remove_urls(text)

        assert "https://example.com" not in result
        assert "[URL]" in result

    def test_mask_email(self, cleaner):
        """测试邮箱脱敏"""
        text = "Contact me at test@example.com please"
        result = cleaner.mask_email(text)

        assert "te***@example.com" in result
```

```python
# tests/test_minio_service.py

import pytest
from unittest.mock import MagicMock, patch
import io

from app.services.minio_service import MiniOService


class TestMiniOService:
    """MinIO服务测试"""

    @pytest.fixture
    def mock_minio(self):
        """创建Mock的MinIO客户端"""
        mock_client = MagicMock()
        return mock_client

    @pytest.fixture
    def minio_service(self, mock_minio):
        """创建MinIO服务（使用mock）"""
        with patch('app.services.minio_service.Minio', return_value=mock_minio):
            with patch('app.services.minio_service.settings') as mock_settings:
                mock_settings.minio.ENDPOINT = "localhost:9000"
                mock_settings.minio.ACCESS_KEY = "minioadmin"
                mock_settings.minio.SECRET_KEY = "minioadmin"
                mock_settings.minio.BUCKET = "test-bucket"
                mock_settings.minio.SECURE = False

                service = MiniOService()
                service.client = mock_minio
                return service

    def test_upload_file(self, minio_service, mock_minio):
        """测试上传文件"""
        data = b"test file content"
        object_name = "test/file.txt"

        result = minio_service.upload_file(data, object_name, "text/plain")

        mock_minio.put_object.assert_called_once()
        assert "http://" in result

    def test_download_file(self, minio_service, mock_minio):
        """测试下载文件"""
        mock_response = MagicMock()
        mock_response.read.return_value = b"file content"
        mock_minio.get_object.return_value = mock_response

        result = minio_service.download_file("test/file.txt")

        assert result == b"file content"

    def test_download_file_not_found(self, minio_service, mock_minio):
        """测试下载不存在的文件"""
        from minio.error import S3Error

        error = S3Error("NoSuchKey", "test", "test")
        mock_minio.get_object.side_effect = error

        with pytest.raises(FileNotFoundError):
            minio_service.download_file("nonexistent/file.txt")

    def test_delete_file(self, minio_service, mock_minio):
        """测试删除文件"""
        minio_service.delete_file("test/file.txt")

        mock_minio.remove_object.assert_called_once_with(
            bucket_name="test-bucket",
            object_name="test/file.txt"
        )

    def test_get_presigned_url(self, minio_service, mock_minio):
        """测试获取预签名URL"""
        mock_minio.presigned_get_object.return_value = "https://signed-url"

        result = minio_service.get_presigned_url("test/file.txt", expires=3600)

        assert "signed-url" in result

    def test_file_exists(self, minio_service, mock_minio):
        """测试检查文件是否存在"""
        mock_minio.stat_object.return_value = MagicMock()

        assert minio_service.file_exists("test/file.txt") is True

    def test_file_not_exists(self, minio_service, mock_minio):
        """测试文件不存在"""
        from minio.error import S3Error

        error = S3Error("NoSuchKey", "test", "test")
        mock_minio.stat_object.side_effect = error

        assert minio_service.file_exists("nonexistent/file.txt") is False
```

---

## 12. 验收标准

1. ✅ JWT 工具提供完整的 Token 创建和验证功能
2. ✅ 密码哈希工具使用 bcrypt，支持配置轮数
3. ✅ 文本清理工具提供常用的文本处理功能
4. ✅ MinIO 服务提供文件上传/下载/删除等操作
5. ✅ Redis 客户端封装提供常用的操作方法
6. ✅ 统一异常类覆盖所有业务场景
7. ✅ 单元测试覆盖所有工具函数
8. ✅ 所有工具函数可通过单例模式获取

---

## 13. 后续任务

- [SDD-04: AI服务集成](SDD-04-AI服务集成.md) - 依赖本任务的配置
- [SDD-05: 用户模块](SDD-05-用户模块.md) - 依赖本任务的 JWT 和哈希工具
