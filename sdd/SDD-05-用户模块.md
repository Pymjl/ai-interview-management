# SDD-05: 用户模块

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](SDD-02-数据库基础架构.md)
> - [SDD-03: 公共工具层](SDD-03-公共工具层.md)
> **目标**: 创建用户认证模块（注册、登录、Token管理）

---

## 1. 概述

本任务负责创建用户认证模块，包括：
- 用户 Schema 定义
- 认证服务 (AuthService)
- 用户服务 (UserService)
- 会话服务 (SessionService)
- API 路由
- 依赖注入

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── __init__.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py           # 新增
│   │   └── common.py         # 新增
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth_service.py   # 新增
│   │   ├── user_service.py   # 新增
│   │   └── session_service.py # 新增
│   ├── api/
│   │   ├── __init__.py
│   │   └── auth.py           # 新增
│   ├── models/
│   │   ├── user.py           # 来自 SDD-02
│   │   └── base.py
│   └── utils/
│       ├── jwt.py            # 来自 SDD-03
│       └── hash_util.py      # 来自 SDD-03
└── ...
```

---

## 3. Schema 定义 (app/schemas/user.py)

```python
# app/schemas/user.py

"""用户相关 Schema"""

from pydantic import BaseModel, EmailStr, Field, ConfigDict
from typing import Optional
from datetime import datetime


class UserBase(BaseModel):
    """用户基础 Schema"""
    email: EmailStr
    username: str = Field(..., min_length=2, max_length=100)
    name: Optional[str] = None
    phone: Optional[str] = None
    gender: Optional[str] = None
    age: Optional[int] = None
    avatar_url: Optional[str] = None
    school: Optional[str] = None
    address: Optional[str] = None
    homepage: Optional[str] = None


class UserCreate(UserBase):
    """用户注册请求"""
    password: str = Field(..., min_length=8, max_length=128)


class UserUpdate(BaseModel):
    """用户信息更新"""
    username: Optional[str] = Field(None, min_length=2, max_length=100)
    name: Optional[str] = None
    phone: Optional[str] = None
    gender: Optional[str] = None
    age: Optional[int] = None
    avatar_url: Optional[str] = None
    school: Optional[str] = None
    address: Optional[str] = None
    homepage: Optional[str] = None


class UserResponse(UserBase):
    """用户响应（不包含敏感信息）"""
    id: int
    is_active: bool
    last_login_at: Optional[datetime] = None
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)


class ChangePasswordRequest(BaseModel):
    """修改密码请求"""
    old_password: str
    new_password: str = Field(..., min_length=8, max_length=128)


# ============ 认证相关 Schema ============

class LoginRequest(BaseModel):
    """登录请求"""
    email: EmailStr
    password: str


class LoginResponse(BaseModel):
    """登录响应"""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int
    user: UserResponse


class RefreshTokenRequest(BaseModel):
    """刷新 Token 请求"""
    refresh_token: str


class RefreshTokenResponse(BaseModel):
    """刷新 Token 响应"""
    access_token: str
    expires_in: int


class TokenResponse(BaseModel):
    """Token 响应"""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int


class RegisterResponse(BaseModel):
    """注册响应"""
    user: UserResponse
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int
```

---

## 4. 公共 Schema (app/schemas/common.py)

```python
# app/schemas/common.py

"""通用 Schema"""

from pydantic import BaseModel, Field
from typing import Generic, TypeVar, Optional, Any

T = TypeVar("T")


class ApiResponse(BaseModel, Generic[T]):
    """统一 API 响应格式"""
    code: int = 200
    message: str = "success"
    data: Optional[T] = None


class PageParams(BaseModel):
    """分页参数"""
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=10, ge=1, le=100)


class PageResponse(BaseModel, Generic[T]):
    """分页响应"""
    items: list[T]
    total: int
    page: int
    page_size: int
    has_more: bool


class MessageResponse(BaseModel):
    """消息响应"""
    message: str
    code: int = 200


class ErrorResponse(BaseModel):
    """错误响应"""
    code: int
    message: str
    detail: Optional[str] = None
```

---

## 5. 会话服务 (app/services/session_service.py)

```python
# app/services/session_service.py

"""会话管理服务

管理用户会话和 Token 黑名单
"""

import json
from typing import Optional
from datetime import timedelta

from app.services.redis_client import get_redis_client, RedisClient
from app.utils.jwt import TokenPair


class SessionService:
    """会话管理服务"""

    # Redis Key 模式
    SESSION_KEY = "user:session:{user_id}"
    TOKEN_BLACKLIST_KEY = "token:blacklist:{jti}"
    REFRESH_FAMILY_KEY = "refresh_token:family:{family_id}"

    def __init__(self, redis_client: Optional[RedisClient] = None):
        self._redis = redis_client or get_redis_client()

    async def create_session(
        self,
        user_id: int,
        access_token_jti: str,
        refresh_token_jti: str,
        family_id: str,
        device_info: Optional[str] = None,
        ttl_days: int = 7,
    ) -> None:
        """创建用户会话

        Args:
            user_id: 用户ID
            access_token_jti: Access Token 的 JTI
            refresh_token_jti: Refresh Token 的 JTI
            family_id: Token 家族ID
            device_info: 设备信息
            ttl_days: 会话有效期（天）
        """
        session_key = self.SESSION_KEY.format(user_id=user_id)
        ttl_seconds = int(timedelta(days=ttl_days).total_seconds())

        session_data = {
            "access_token_jti": access_token_jti,
            "refresh_token_jti": refresh_token_jti,
            "family_id": family_id,
            "device_info": device_info or "unknown",
        }

        await self._redis.hset_dict(session_key, session_data)
        await self._redis.expire(session_key, ttl_seconds)

        # 存储 Token 家族信息
        family_key = self.REFRESH_FAMILY_KEY.format(family_id=family_id)
        family_data = {
            "user_id": str(user_id),
            "rotated_count": 0,
        }
        await self._redis.set_json(family_key, family_data, ex=ttl_seconds)

    async def get_session(self, user_id: int) -> Optional[dict]:
        """获取用户会话"""
        session_key = self.SESSION_KEY.format(user_id=user_id)
        return await self._redis.hgetall(session_key)

    async def invalidate_session(self, user_id: int) -> None:
        """使会话失效"""
        session_key = self.SESSION_KEY.format(user_id=user_id)
        session = await self._redis.hgetall(session_key)

        if session:
            # 将 Access Token 加入黑名单
            await self.add_to_blacklist(
                session.get("access_token_jti", ""),
                int(timedelta(minutes=15).total_seconds())
            )

            # 删除 Refresh Token 家族
            family_id = session.get("family_id")
            if family_id:
                await self._redis.delete(
                    self.REFRESH_FAMILY_KEY.format(family_id=family_id)
                )

        await self._redis.delete(session_key)

    async def add_to_blacklist(self, jti: str, ttl_seconds: int) -> None:
        """将 Token JTI 加入黑名单"""
        if not jti:
            return
        blacklist_key = self.TOKEN_BLACKLIST_KEY.format(jti=jti)
        await self._redis.set(blacklist_key, "1", ex=ttl_seconds)

    async def is_blacklisted(self, jti: str) -> bool:
        """检查 Token 是否在黑名单"""
        if not jti:
            return False
        blacklist_key = self.TOKEN_BLACKLIST_KEY.format(jti=jti)
        result = await self._redis.exists(blacklist_key)
        return result > 0

    async def rotate_refresh_token(
        self,
        old_jti: str,
        family_id: str,
    ) -> Optional[tuple[str, int]]:
        """轮换 Refresh Token

        Returns:
            (new_jti, rotated_count) 或 None
        """
        family_key = self.REFRESH_FAMILY_KEY.format(family_id=family_id)
        family_data = await self._redis.get_json(family_key)

        if not family_data:
            return None

        # 将旧 Token 加入黑名单（保守设为1小时）
        await self.add_to_blacklist(old_jti, ttl_seconds=3600)

        # 更新家族计数
        rotated_count = family_data.get("rotated_count", 0) + 1
        family_data["rotated_count"] = rotated_count

        # 保持原有 TTL
        ttl = await self._redis.ttl(family_key)
        await self._redis.set_json(
            family_key,
            family_data,
            ex=ttl if ttl > 0 else None
        )

        return old_jti, rotated_count

    async def extend_session(self, user_id: int, ttl_days: int = 7) -> None:
        """延长会话有效期"""
        session_key = self.SESSION_KEY.format(user_id=user_id)
        ttl_seconds = int(timedelta(days=ttl_days).total_seconds())
        await self._redis.expire(session_key, ttl_seconds)
```

---

## 6. 认证服务 (app/services/auth_service.py)

```python
# app/services/auth_service.py

"""认证服务

处理用户注册、登录、登出、Token刷新等
"""

from typing import Optional, Tuple
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models.user import User
from app.schemas.user import (
    UserCreate,
    UserResponse,
    LoginRequest,
    TokenPair,
)
from app.services.session_service import SessionService
from app.utils.jwt import JWTManager, TokenPair as JWTTokenPair
from app.utils.hash_util import HashUtil
from app.exceptions import (
    AuthenticationException,
    BusinessException,
    DuplicateResourceException,
)


class AuthService:
    """认证服务"""

    def __init__(
        self,
        db: AsyncSession,
        jwt_manager: Optional[JWTManager] = None,
        session_service: Optional[SessionService] = None,
        hash_util: Optional[HashUtil] = None,
    ):
        self.db = db
        self.jwt_manager = jwt_manager or JWTManager()
        self.session_service = session_service or SessionService()
        self.hash_util = hash_util or HashUtil()

    async def register(self, request: UserCreate) -> Tuple[User, JWTTokenPair]:
        """用户注册

        Args:
            request: 注册请求

        Returns:
            (用户对象, Token对)

        Raises:
            DuplicateResourceException: 邮箱已被注册
            BusinessException: 密码不符合要求
        """
        # 1. 检查邮箱唯一性
        existing = await self.db.execute(
            select(User).where(User.email == request.email)
        )
        if existing.scalar_one_or_none():
            raise DuplicateResourceException("邮箱")

        # 2. 密码强度校验
        if len(request.password) < 8:
            raise BusinessException("密码长度至少8位")

        # 3. 哈希密码
        password_hash = self.hash_util.hash_password(request.password)

        # 4. 创建用户
        user = User(
            email=request.email,
            username=request.username,
            password_hash=password_hash,
            name=request.name,
            phone=request.phone,
            gender=request.gender,
            age=request.age,
            school=request.school,
            address=request.address,
            homepage=request.homepage,
        )
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)

        # 5. 生成 Token
        token_pair = self.jwt_manager.create_token_pair(user.id, user.email)

        # 6. 存储会话
        await self.session_service.create_session(
            user_id=user.id,
            access_token_jti=token_pair.access_token_jti,
            refresh_token_jti=token_pair.refresh_token_jti,
            family_id=token_pair.refresh_token_jti,
        )

        return user, token_pair

    async def login(self, request: LoginRequest) -> Tuple[User, JWTTokenPair]:
        """用户登录

        Args:
            request: 登录请求

        Returns:
            (用户对象, Token对)

        Raises:
            AuthenticationException: 邮箱或密码错误
            BusinessException: 账号已被禁用
        """
        # 1. 查找用户
        result = await self.db.execute(
            select(User).where(User.email == request.email)
        )
        user = result.scalar_one_or_none()

        if not user:
            raise AuthenticationException("邮箱或密码错误")

        # 2. 验证密码
        if not self.hash_util.verify_password(request.password, user.password_hash):
            raise AuthenticationException("邮箱或密码错误")

        # 3. 检查账号状态
        if not user.is_active:
            raise BusinessException("账号已被禁用")

        # 4. 生成 Token
        token_pair = self.jwt_manager.create_token_pair(user.id, user.email)

        # 5. 更新最后登录时间
        user.last_login_at = datetime.utcnow()
        await self.db.commit()

        # 6. 存储会话
        await self.session_service.create_session(
            user_id=user.id,
            access_token_jti=token_pair.access_token_jti,
            refresh_token_jti=token_pair.refresh_token_jti,
            family_id=token_pair.refresh_token_jti,
        )

        return user, token_pair

    async def refresh_token(self, refresh_token: str) -> Tuple[str, int]:
        """刷新 Access Token

        Args:
            refresh_token: Refresh Token

        Returns:
            (新的Access Token, 过期时间秒数)

        Raises:
            AuthenticationException: Token无效或已失效
        """
        # 1. 验证 Refresh Token
        payload = self.jwt_manager.verify_token(refresh_token, expected_type="refresh")
        if not payload:
            raise AuthenticationException("Refresh Token 无效")

        # 2. 检查是否在黑名单
        if await self.session_service.is_blacklisted(payload.jti):
            raise AuthenticationException("Refresh Token 已失效")

        # 3. 轮换 Refresh Token
        result = await self.session_service.rotate_refresh_token(
            payload.jti, payload.family
        )
        if not result:
            raise AuthenticationException("Refresh Token 家族无效")

        # 4. 创建新的 Access Token
        new_access_token, access_jti = self.jwt_manager.create_access_token(
            int(payload.sub), payload.email
        )

        expires_in = int(self.jwt_manager.access_token_expire.total_seconds())

        # 5. 将旧 Access Token 加入黑名单
        await self.session_service.add_to_blacklist(
            payload.jti,
            int(self.jwt_manager.access_token_expire.total_seconds())
        )

        return new_access_token, expires_in

    async def logout(self, user_id: int, access_token_jti: str) -> None:
        """用户登出

        Args:
            user_id: 用户ID
            access_token_jti: Access Token 的 JTI
        """
        # 1. 使会话失效
        await self.session_service.invalidate_session(user_id)

        # 2. 将 Access Token 加入黑名单
        await self.session_service.add_to_blacklist(
            access_token_jti,
            int(self.jwt_manager.access_token_expire.total_seconds())
        )

    async def change_password(
        self,
        user_id: int,
        old_password: str,
        new_password: str,
    ) -> None:
        """修改密码

        Args:
            user_id: 用户ID
            old_password: 旧密码
            new_password: 新密码

        Raises:
            AuthenticationException: 原密码错误
            BusinessException: 新密码不符合要求
        """
        # 1. 获取用户
        result = await self.db.execute(select(User).where(User.id == user_id))
        user = result.scalar_one_or_none()

        if not user:
            raise BusinessException("用户不存在")

        # 2. 验证旧密码
        if not self.hash_util.verify_password(old_password, user.password_hash):
            raise AuthenticationException("原密码错误")

        # 3. 验证新密码
        if len(new_password) < 8:
            raise BusinessException("新密码长度至少8位")

        # 4. 更新密码
        user.password_hash = self.hash_util.hash_password(new_password)
        await self.db.commit()

        # 5. 使所有现有会话失效（可选，安全性增强）
        # await self.session_service.invalidate_session(user_id)
```

---

## 7. 用户服务 (app/services/user_service.py)

```python
# app/services/user_service.py

"""用户服务

处理用户信息查询和更新
"""

from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from fastapi import UploadFile

from app.models.user import User
from app.schemas.user import UserUpdate, UserResponse
from app.services.minio_service import get_minio_service, MiniOService
from app.exceptions import BusinessException, ResourceNotFoundException


class UserService:
    """用户服务"""

    def __init__(
        self,
        db: AsyncSession,
        minio_service: Optional[MiniOService] = None,
    ):
        self.db = db
        self.minio_service = minio_service or get_minio_service()

    async def get_user_by_id(self, user_id: int) -> Optional[User]:
        """根据ID获取用户"""
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_user_by_email(self, email: str) -> Optional[User]:
        """根据邮箱获取用户"""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def update_user(self, user_id: int, update_data: UserUpdate) -> User:
        """更新用户信息

        Args:
            user_id: 用户ID
            update_data: 更新数据

        Returns:
            更新后的用户

        Raises:
            ResourceNotFoundException: 用户不存在
        """
        user = await self.get_user_by_id(user_id)
        if not user:
            raise ResourceNotFoundException("用户")

        # 更新非空字段
        update_dict = update_data.model_dump(exclude_unset=True)
        for key, value in update_dict.items():
            if hasattr(user, key):
                setattr(user, key, value)

        await self.db.commit()
        await self.db.refresh(user)

        return user

    async def upload_avatar(self, user_id: int, file: UploadFile) -> str:
        """上传头像

        Args:
            user_id: 用户ID
            file: 上传的文件

        Returns:
            头像URL

        Raises:
            ResourceNotFoundException: 用户不存在
            BusinessException: 文件类型不支持
        """
        user = await self.get_user_by_id(user_id)
        if not user:
            raise ResourceNotFoundException("用户")

        # 验证文件类型
        allowed_types = ["image/jpeg", "image/png", "image/gif", "image/webp"]
        if file.content_type not in allowed_types:
            raise BusinessException("不支持的图片格式")

        # 读取文件内容
        content = await file.read()

        # 生成存储路径
        import uuid
        ext = file.filename.split(".")[-1] if file.filename else "jpg"
        object_name = f"avatars/{user_id}/{uuid.uuid4()}.{ext}"

        # 上传到 MinIO
        avatar_url = self.minio_service.upload_file(
            data=content,
            object_name=object_name,
            content_type=file.content_type or "image/jpeg",
        )

        # 更新用户头像
        user.avatar_url = avatar_url
        await self.db.commit()

        return avatar_url

    async def deactivate_user(self, user_id: int) -> None:
        """禁用用户账号

        Args:
            user_id: 用户ID
        """
        user = await self.get_user_by_id(user_id)
        if not user:
            raise ResourceNotFoundException("用户")

        user.is_active = False
        await self.db.commit()

    async def activate_user(self, user_id: int) -> None:
        """激活用户账号

        Args:
            user_id: 用户ID
        """
        user = await self.get_user_by_id(user_id)
        if not user:
            raise ResourceNotFoundException("用户")

        user.is_active = True
        await self.db.commit()
```

---

## 8. API 路由 (app/api/auth.py)

```python
# app/api/auth.py

"""认证 API 路由"""

from fastapi import APIRouter, Depends, HTTPException, status, UploadFile, File, Request, Cookie
from typing import Optional
from fastapi.responses import JSONResponse

from app.schemas.user import (
    UserCreate,
    UserResponse,
    LoginRequest,
    LoginResponse,
    RefreshTokenRequest,
    RefreshTokenResponse,
    ChangePasswordRequest,
    RegisterResponse,
)
from app.services.auth_service import AuthService
from app.services.user_service import UserService
from app.database import get_db_session
from app.exceptions import (
    AuthenticationException,
    BusinessException,
    DuplicateResourceException,
)
from app.models.user import User
from app.deps import get_current_user, get_current_active_user

router = APIRouter(prefix="/api/auth", tags=["auth"])


@router.post("/register", response_model=RegisterResponse, status_code=status.HTTP_201_CREATED)
async def register(
    request: UserCreate,
    db=Depends(get_db_session),
):
    """用户注册"""
    auth_service = AuthService(db)

    try:
        user, token_pair = await auth_service.register(request)
    except DuplicateResourceException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=e.message
        )
    except BusinessException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=e.message
        )

    return RegisterResponse(
        user=UserResponse.model_validate(user),
        access_token=token_pair.access_token,
        refresh_token=token_pair.refresh_token,
        token_type="bearer",
        expires_in=token_pair.expires_in,
    )


@router.post("/login", response_model=LoginResponse)
async def login(
    request: LoginRequest,
    db=Depends(get_db_session),
):
    """用户登录"""
    auth_service = AuthService(db)

    try:
        user, token_pair = await auth_service.login(request)
    except AuthenticationException as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=e.message
        )
    except BusinessException as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=e.message
        )

    return LoginResponse(
        access_token=token_pair.access_token,
        refresh_token=token_pair.refresh_token,
        token_type="bearer",
        expires_in=token_pair.expires_in,
        user=UserResponse.model_validate(user),
    )


@router.post("/logout")
async def logout(
    request: Request,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """用户登出"""
    from app.services.auth_service import AuthService

    auth_service = AuthService(db)

    access_token = request.headers.get("Authorization", "").replace("Bearer ", "")
    await auth_service.logout(current_user.id, access_token)

    response = JSONResponse({"message": "登出成功"})
    response.delete_cookie("refresh_token")
    return response


@router.post("/refresh", response_model=RefreshTokenResponse)
async def refresh_token(
    request: Request,
    response: JSONResponse,
    db=Depends(get_db_session),
):
    """刷新 Access Token"""
    from app.services.auth_service import AuthService

    # 优先从 Cookie 获取，否则从请求体获取
    refresh_token = request.cookies.get("refresh_token")
    if not refresh_token:
        body = await request.json()
        refresh_token = body.get("refresh_token")

    if not refresh_token:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="缺少 refresh_token"
        )

    auth_service = AuthService(db)

    try:
        new_access_token, expires_in = await auth_service.refresh_token(refresh_token)
    except AuthenticationException as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=e.message
        )

    return {
        "access_token": new_access_token,
        "expires_in": expires_in,
    }


@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_active_user),
):
    """获取当前用户信息"""
    return current_user


@router.put("/me", response_model=UserResponse)
async def update_current_user_info(
    request: UserUpdate,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """更新当前用户信息"""
    from app.services.user_service import UserService

    user_service = UserService(db)
    updated_user = await user_service.update_user(current_user.id, request)
    return updated_user


@router.post("/change-password")
async def change_password(
    request: ChangePasswordRequest,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """修改密码"""
    from app.services.auth_service import AuthService

    auth_service = AuthService(db)

    try:
        await auth_service.change_password(
            current_user.id,
            request.old_password,
            request.new_password
        )
    except AuthenticationException as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=e.message
        )
    except BusinessException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=e.message
        )

    return {"message": "密码修改成功"}


@router.post("/upload-avatar")
async def upload_avatar(
    file: UploadFile = File(...),
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """上传头像"""
    from app.services.user_service import UserService

    user_service = UserService(db)

    try:
        avatar_url = await user_service.upload_avatar(current_user.id, file)
    except BusinessException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=e.message
        )

    return {"avatar_url": avatar_url}
```

---

## 9. 依赖注入 (app/deps.py)

```python
# app/deps.py

"""依赖注入

提供各种服务的依赖获取函数
"""

from typing import Optional
from functools import lru_cache
from fastapi import Depends, HTTPException, status, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db_session
from app.models.user import User
from app.services.auth_service import AuthService
from app.services.user_service import UserService
from app.services.session_service import SessionService
from app.services.minio_service import get_minio_service
from app.utils.jwt import get_jwt_manager, JWTManager
from app.utils.hash_util import get_hash_util, HashUtil

# 安全方案
security = HTTPBearer(auto_error=False)


async def get_current_user(
    request: Request,
    credentials: Optional[HTTPAuthorizationCredentials] = Depends(security),
    db: AsyncSession = Depends(get_db_session),
    jwt_manager: JWTManager = Depends(get_jwt_manager),
) -> User:
    """获取当前登录用户（依赖注入）

    如果用户未登录或Token无效，抛出401异常
    """
    if not credentials:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="未提供认证凭证",
            headers={"WWW-Authenticate": "Bearer"},
        )

    token = credentials.credentials

    # 1. 解析并验证 JWT
    payload = jwt_manager.verify_token(token, expected_type="access")
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token 无效或已过期",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # 2. 检查 Token 是否在黑名单
    session_service = SessionService()
    if await session_service.is_blacklisted(payload.jti):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token 已失效",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # 3. 从数据库加载用户
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


def get_current_access_token(
    credentials: Optional[HTTPAuthorizationCredentials] = Depends(security),
) -> str:
    """获取当前 Access Token"""
    if not credentials:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="未提供认证凭证",
        )
    return credentials.credentials


# 服务工厂函数

def get_auth_service(db: AsyncSession = Depends(get_db_session)) -> AuthService:
    """获取认证服务"""
    return AuthService(db)


def get_user_service(db: AsyncSession = Depends(get_db_session)) -> UserService:
    """获取用户服务"""
    return UserService(db)


def get_session_service() -> SessionService:
    """获取会话服务"""
    return SessionService()
```

---

## 10. Schema 导出 (app/schemas/__init__.py)

```python
# app/schemas/__init__.py

"""Schema 定义"""

from app.schemas.user import (
    UserBase,
    UserCreate,
    UserUpdate,
    UserResponse,
    LoginRequest,
    LoginResponse,
    RefreshTokenRequest,
    RefreshTokenResponse,
    ChangePasswordRequest,
    TokenResponse,
    RegisterResponse,
)
from app.schemas.common import (
    ApiResponse,
    PageParams,
    PageResponse,
    MessageResponse,
    ErrorResponse,
)

__all__ = [
    # User schemas
    "UserBase",
    "UserCreate",
    "UserUpdate",
    "UserResponse",
    "LoginRequest",
    "LoginResponse",
    "RefreshTokenRequest",
    "RefreshTokenResponse",
    "ChangePasswordRequest",
    "TokenResponse",
    "RegisterResponse",
    # Common schemas
    "ApiResponse",
    "PageParams",
    "PageResponse",
    "MessageResponse",
    "ErrorResponse",
]
```

---

## 11. 单元测试

```python
# tests/test_auth_service.py

import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from datetime import datetime

from app.services.auth_service import AuthService
from app.schemas.user import UserCreate, LoginRequest
from app.exceptions import AuthenticationException, DuplicateResourceException, BusinessException


class TestAuthService:
    """认证服务测试"""

    @pytest.fixture
    def mock_db(self):
        """Mock数据库会话"""
        db = AsyncMock()
        db.execute = AsyncMock()
        db.commit = AsyncMock()
        db.refresh = AsyncMock()
        return db

    @pytest.fixture
    def mock_jwt_manager(self):
        """Mock JWT管理器"""
        from app.utils.jwt import TokenPair
        manager = MagicMock()
        manager.create_token_pair.return_value = TokenPair(
            access_token="access_token",
            refresh_token="refresh_token",
            access_token_jti="access_jti",
            refresh_token_jti="refresh_jti",
            expires_in=900,
        )
        return manager

    @pytest.fixture
    def mock_session_service(self):
        """Mock会话服务"""
        service = AsyncMock()
        service.create_session = AsyncMock()
        service.is_blacklisted = AsyncMock(return_value=False)
        service.rotate_refresh_token = AsyncMock(return_value=("old_jti", 1))
        service.add_to_blacklist = AsyncMock()
        return service

    @pytest.fixture
    def auth_service(self, mock_db, mock_jwt_manager, mock_session_service):
        """创建认证服务"""
        return AuthService(
            db=mock_db,
            jwt_manager=mock_jwt_manager,
            session_service=mock_session_service,
        )

    @pytest.mark.asyncio
    async def test_register_success(self, auth_service, mock_db):
        """测试成功注册"""
        from app.models.user import User

        # Mock查询结果（邮箱不存在）
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = None
        mock_db.execute.return_value = mock_result

        # Mock用户对象
        mock_user = User(
            id=1,
            email="test@example.com",
            username="testuser",
            password_hash="hashed",
        )

        # 模拟add和refresh
        async def mock_add(user):
            user.id = 1

        mock_db.add.side_effect = mock_add
        mock_db.refresh = AsyncMock()

        with patch('app.services.auth_service.HashUtil') as MockHashUtil:
            mock_hash = MagicMock()
            mock_hash.hash_password.return_value = "hashed"
            MockHashUtil.return_value = mock_hash

            request = UserCreate(
                email="test@example.com",
                username="testuser",
                password="password123",
            )

            user, token_pair = await auth_service.register(request)

            assert user.email == "test@example.com"
            assert token_pair.access_token == "access_token"

    @pytest.mark.asyncio
    async def test_register_duplicate_email(self, auth_service, mock_db):
        """测试注册重复邮箱"""
        from app.models.user import User

        # Mock查询结果（邮箱已存在）
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = User(
            id=1,
            email="test@example.com",
            username="existing",
            password_hash="hash",
        )
        mock_db.execute.return_value = mock_result

        request = UserCreate(
            email="test@example.com",
            username="testuser",
            password="password123",
        )

        with pytest.raises(DuplicateResourceException):
            await auth_service.register(request)

    @pytest.mark.asyncio
    async def test_login_success(self, auth_service, mock_db, mock_session_service):
        """测试成功登录"""
        from app.models.user import User

        mock_user = User(
            id=1,
            email="test@example.com",
            username="testuser",
            password_hash="$2b$12$hashedpassword",
            is_active=True,
        )

        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = mock_user
        mock_db.execute.return_value = mock_result

        with patch('app.services.auth_service.HashUtil') as MockHashUtil:
            mock_hash = MagicMock()
            mock_hash.verify_password.return_value = True
            MockHashUtil.return_value = mock_hash

            request = LoginRequest(
                email="test@example.com",
                password="password123",
            )

            user, token_pair = await auth_service.login(request)

            assert user.email == "test@example.com"
            assert token_pair.access_token == "access_token"

    @pytest.mark.asyncio
    async def test_login_wrong_password(self, auth_service, mock_db):
        """测试密码错误"""
        from app.models.user import User

        mock_user = User(
            id=1,
            email="test@example.com",
            username="testuser",
            password_hash="$2b$12$hashedpassword",
            is_active=True,
        )

        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = mock_user
        mock_db.execute.return_value = mock_result

        with patch('app.services.auth_service.HashUtil') as MockHashUtil:
            mock_hash = MagicMock()
            mock_hash.verify_password.return_value = False
            MockHashUtil.return_value = mock_hash

            request = LoginRequest(
                email="test@example.com",
                password="wrongpassword",
            )

            with pytest.raises(AuthenticationException):
                await auth_service.login(request)

    @pytest.mark.asyncio
    async def test_login_user_not_found(self, auth_service, mock_db):
        """测试用户不存在"""
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = None
        mock_db.execute.return_value = mock_result

        request = LoginRequest(
            email="notexist@example.com",
            password="password123",
        )

        with pytest.raises(AuthenticationException):
            await auth_service.login(request)

    @pytest.mark.asyncio
    async def test_logout(self, auth_service, mock_session_service):
        """测试登出"""
        await auth_service.logout(user_id=1, access_token_jti="test_jti")

        mock_session_service.invalidate_session.assert_called_once_with(1)
        mock_session_service.add_to_blacklist.assert_called_once()
```

```python
# tests/test_session_service.py

import pytest
from unittest.mock import AsyncMock, patch
from datetime import timedelta

from app.services.session_service import SessionService


class TestSessionService:
    """会话服务测试"""

    @pytest.fixture
    def mock_redis(self):
        """Mock Redis客户端"""
        redis = AsyncMock()
        redis.hset_dict = AsyncMock()
        redis.expire = AsyncMock()
        redis.set_json = AsyncMock()
        redis.get_json = AsyncMock()
        redis.hgetall = AsyncMock()
        redis.delete = AsyncMock()
        redis.set = AsyncMock()
        redis.exists = AsyncMock(return_value=0)
        redis.ttl = AsyncMock(return_value=3600)
        return redis

    @pytest.fixture
    def session_service(self, mock_redis):
        """创建会话服务"""
        service = SessionService(redis_client=mock_redis)
        service._redis = mock_redis
        return service

    @pytest.mark.asyncio
    async def test_create_session(self, session_service, mock_redis):
        """测试创建会话"""
        await session_service.create_session(
            user_id=1,
            access_token_jti="access_jti",
            refresh_token_jti="refresh_jti",
            family_id="family_123",
        )

        mock_redis.hset_dict.assert_called_once()
        mock_redis.expire.assert_called_once()
        mock_redis.set_json.assert_called_once()

    @pytest.mark.asyncio
    async def test_is_blacklisted_true(self, session_service, mock_redis):
        """测试Token在黑名单"""
        mock_redis.exists = AsyncMock(return_value=1)

        result = await session_service.is_blacklisted("test_jti")

        assert result is True

    @pytest.mark.asyncio
    async def test_is_blacklisted_false(self, session_service, mock_redis):
        """测试Token不在黑名单"""
        mock_redis.exists = AsyncMock(return_value=0)

        result = await session_service.is_blacklisted("test_jti")

        assert result is False

    @pytest.mark.asyncio
    async def test_rotate_refresh_token(self, session_service, mock_redis):
        """测试轮换Refresh Token"""
        mock_redis.get_json = AsyncMock(return_value={
            "user_id": "1",
            "rotated_count": 0,
        })
        mock_redis.ttl = AsyncMock(return_value=3600)

        result = await session_service.rotate_refresh_token(
            old_jti="old_jti",
            family_id="family_123",
        )

        assert result is not None
        old_jti, count = result
        assert old_jti == "old_jti"
        assert count == 1
```

---

## 12. 验收标准

1. ✅ 用户注册功能（邮箱唯一性校验、密码哈希）
2. ✅ 用户登录功能（密码验证、Token生成）
3. ✅ Token刷新功能（Refresh Token轮换）
4. ✅ 用户登出功能（会话失效、Token黑名单）
5. ✅ 修改密码功能
6. ✅ 上传头像功能
7. ✅ 用户信息更新功能
8. ✅ 所有API路由符合原Java接口规范
9. ✅ 单元测试覆盖核心功能

---

## 13. 后续任务

- [SDD-06: 简历模块](SDD-06-简历模块.md) - 依赖本任务的用户模型和认证
- [SDD-11: 安全与限流](SDD-11-安全与限流.md) - 依赖本任务的认证依赖注入
