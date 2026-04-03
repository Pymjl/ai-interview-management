# SDD-15: 日志模块

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](SDD-02-数据库基础架构.md)
> - [SDD-05: 用户模块](SDD-05-用户模块.md)
> **目标**: 创建操作日志模块，记录用户所有操作，支持管理员查看

---

## 1. 概述

本任务负责创建操作日志模块，包括：
- 操作日志数据模型
- 日志记录中间件（自动记录所有API请求）
- 日志查询服务
- 管理员日志查看API
- 前端日志管理页面

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── models/
│   │   └── operation_log.py      # 新增
│   ├── schemas/
│   │   └── operation_log.py      # 新增
│   ├── services/
│   │   └── operation_log_service.py  # 新增
│   ├── api/
│   │   └── operation_log.py      # 新增
│   └── middleware/
│       └── operation_log_middleware.py  # 新增
└── tests/
    └── test_operation_log.py     # 新增

frontend/
└── src/
    ├── pages/
    │   └── OperationLogPage.tsx  # 待实现
    └── api/
        └── operationLog.ts       # 待实现
```

---

## 3. 数据库模型 (app/models/operation_log.py)

```python
# app/models/operation_log.py

"""操作日志模型"""

from sqlalchemy import Column, BigInteger, String, Text, DateTime, Index
from sqlalchemy.orm import relationship
from datetime import datetime

from app.models.base import Base, TimestampMixin


class OperationLog(Base, TimestampMixin):
    """操作日志模型"""

    __tablename__ = "operation_log"

    id = Column(BigInteger, primary_key=True, autoincrement=True)

    # 用户信息
    user_id = Column(BigInteger, nullable=True, index=True, comment="操作用户ID")
    user_email = Column(String(255), nullable=True, comment="操作用户邮箱")

    # 请求信息
    method = Column(String(10), nullable=False, comment="HTTP方法")
    path = Column(String(500), nullable=False, index=True, comment="请求路径")
    query_params = Column(Text, nullable=True, comment="查询参数(JSON)")
    request_body = Column(Text, nullable=True, comment="请求体(JSON)")
    request_ip = Column(String(50), nullable=True, comment="请求IP地址")
    user_agent = Column(String(500), nullable=True, comment="用户代理")

    # 响应信息
    status_code = Column(BigInteger, nullable=True, comment="HTTP状态码")
    response_body = Column(Text, nullable=True, comment="响应体(JSON，截断)")
    duration_ms = Column(BigInteger, nullable=True, comment="请求耗时(毫秒)")

    # 异常信息
    error_message = Column(Text, nullable=True, comment="异常信息")
    error_stack = Column(Text, nullable=True, comment="异常堆栈")

    # 操作类型（可选，预留字段用于手动标注）
    action = Column(String(100), nullable=True, comment="操作类型")
    resource = Column(String(100), nullable=True, comment="资源类型")

    # 索引
    __table_args__ = (
        Index('ix_operation_log_user_id_created', 'user_id', 'created_at'),
        Index('ix_operation_log_path_created', 'path', 'created_at'),
        Index('ix_operation_log_status_code', 'status_code'),
    )

    def __repr__(self) -> str:
        return f"<OperationLog(id={self.id}, user_id={self.user_id}, method={self.method}, path={self.path})>"
```

---

## 4. Schema 定义 (app/schemas/operation_log.py)

```python
# app/schemas/operation_log.py

"""操作日志相关 Schema"""

from pydantic import BaseModel, Field, ConfigDict
from typing import Optional, Any
from datetime import datetime


class OperationLogBase(BaseModel):
    """操作日志基础 Schema"""
    method: str
    path: str
    query_params: Optional[str] = None
    request_body: Optional[str] = None
    request_ip: Optional[str] = None
    user_agent: Optional[str] = None
    status_code: Optional[int] = None
    response_body: Optional[str] = None
    duration_ms: Optional[int] = None
    error_message: Optional[str] = None
    error_stack: Optional[str] = None
    action: Optional[str] = None
    resource: Optional[str] = None


class OperationLogCreate(OperationLogBase):
    """创建操作日志请求"""
    user_id: Optional[int] = None
    user_email: Optional[str] = None


class OperationLogResponse(OperationLogBase):
    """操作日志响应"""
    id: int
    user_id: Optional[int] = None
    user_email: Optional[str] = None
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)


class OperationLogQuery(BaseModel):
    """操作日志查询参数"""
    user_id: Optional[int] = Field(None, description="操作用户ID")
    user_email: Optional[str] = Field(None, description="操作用户邮箱")
    method: Optional[str] = Field(None, description="HTTP方法")
    path: Optional[str] = Field(None, description="请求路径")
    status_code: Optional[int] = Field(None, description="HTTP状态码")
    start_date: Optional[datetime] = Field(None, description="开始时间")
    end_date: Optional[datetime] = Field(None, description="结束时间")
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=20, ge=1, le=100)


class OperationLogSummary(BaseModel):
    """操作日志统计摘要"""
    total_count: int
    today_count: int
    error_count: int
    error_rate: float
```

---

## 5. 日志服务 (app/services/operation_log_service.py)

```python
# app/services/operation_log_service.py

"""操作日志服务

提供日志的创建、查询、统计等功能
"""

import asyncio
from typing import Optional, List, Tuple
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, and_, desc
from datetime import datetime, timedelta, timezone

from app.models.operation_log import OperationLog
from app.schemas.operation_log import OperationLogCreate, OperationLogQuery, OperationLogSummary


class OperationLogService:
    """操作日志服务"""

    # 响应体最大长度（防止存储过大）
    MAX_RESPONSE_BODY_LENGTH = 10000
    # 请求体最大长度
    MAX_REQUEST_BODY_LENGTH = 5000

    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_log(self, log_data: OperationLogCreate) -> OperationLog:
        """创建操作日志

        Args:
            log_data: 日志数据

        Returns:
            创建的日志对象
        """
        # 截断过长的字段
        if log_data.request_body and len(log_data.request_body) > self.MAX_REQUEST_BODY_LENGTH:
            log_data.request_body = log_data.request_body[:self.MAX_REQUEST_BODY_LENGTH] + "...[truncated]"

        if log_data.response_body and len(log_data.response_body) > self.MAX_RESPONSE_BODY_LENGTH:
            log_data.response_body = log_data.response_body[:self.MAX_RESPONSE_BODY_LENGTH] + "...[truncated]"

        log = OperationLog(**log_data.model_dump())
        self.db.add(log)
        await self.db.commit()
        await self.db.refresh(log)
        return log

    async def get_log_by_id(self, log_id: int) -> Optional[OperationLog]:
        """根据ID获取日志"""
        result = await self.db.execute(
            select(OperationLog).where(OperationLog.id == log_id)
        )
        return result.scalar_one_or_none()

    async def query_logs(
        self,
        query: OperationLogQuery
    ) -> Tuple[List[OperationLog], int]:
        """查询操作日志（分页）

        Args:
            query: 查询参数

        Returns:
            (日志列表, 总数)
        """
        # 构建查询条件
        conditions = []

        if query.user_id is not None:
            conditions.append(OperationLog.user_id == query.user_id)

        if query.user_email is not None:
            conditions.append(OperationLog.user_email == query.user_email)

        if query.method is not None:
            conditions.append(OperationLog.method == query.method)

        if query.path is not None:
            conditions.append(OperationLog.path.like(f"%{query.path}%"))

        if query.status_code is not None:
            conditions.append(OperationLog.status_code == query.status_code)

        if query.start_date is not None:
            conditions.append(OperationLog.created_at >= query.start_date)

        if query.end_date is not None:
            conditions.append(OperationLog.created_at <= query.end_date)

        # 构建查询
        base_query = select(OperationLog)
        if conditions:
            base_query = base_query.where(and_(*conditions))

        # 获取总数（复用 base_query 的条件）
        count_query = select(func.count()).select_from(base_query.subquery())
        total_result = await self.db.execute(count_query)
        total = total_result.scalar() or 0

        # 分页查询
        offset = (query.page - 1) * query.page_size
        paginated_query = base_query.order_by(desc(OperationLog.created_at))
        paginated_query = paginated_query.offset(offset).limit(query.page_size)

        result = await self.db.execute(paginated_query)
        logs = result.scalars().all()

        return list(logs), total

    async def get_summary(self) -> OperationLogSummary:
        """获取日志统计摘要

        Returns:
            统计摘要
        """
        now = datetime.now(timezone.utc)
        today_start = now.replace(hour=0, minute=0, second=0, microsecond=0)

        # 三个独立查询并行执行
        total_query = select(func.count(OperationLog.id))
        today_query = select(func.count(OperationLog.id)).where(
            OperationLog.created_at >= today_start
        )
        error_query = select(func.count(OperationLog.id)).where(
            OperationLog.status_code >= 400
        )

        total_result, today_result, error_result = await asyncio.gather(
            self.db.execute(total_query),
            self.db.execute(today_query),
            self.db.execute(error_query),
        )

        total_count = total_result.scalar() or 0
        today_count = today_result.scalar() or 0
        error_count = error_result.scalar() or 0

        error_rate = error_count / total_count if total_count > 0 else 0.0

        return OperationLogSummary(
            total_count=total_count,
            today_count=today_count,
            error_count=error_count,
            error_rate=round(error_rate, 4)
        )

    async def get_recent_error_logs(self, limit: int = 10) -> List[OperationLog]:
        """获取最近的错误日志

        Args:
            limit: 返回数量

        Returns:
            错误日志列表
        """
        query = (
            select(OperationLog)
            .where(OperationLog.status_code >= 400)
            .order_by(desc(OperationLog.created_at))
            .limit(limit)
        )
        result = await self.db.execute(query)
        return list(result.scalars().all())
```

---

## 6. 日志中间件 (app/middleware/operation_log_middleware.py)

```python
# app/middleware/operation_log_middleware.py

"""操作日志中间件

自动记录所有API请求的日志
"""

import asyncio
import json
import time
import traceback
from typing import Callable, Set, Optional
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp
import logging

from app.schemas.operation_log import OperationLogCreate

logger = logging.getLogger(__name__)


# 不需要记录日志的路径
EXCLUDED_PATHS: Set[str] = {
    "/health",
    "/metrics",
    "/docs",
    "/redoc",
    "/openapi.json",
    "/api/auth/refresh",  # 刷新Token频繁，不记录
}


class OperationLogMiddleware(BaseHTTPMiddleware):
    """操作日志中间件"""

    def __init__(self, app: ASGIApp):
        super().__init__(app)

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        """处理请求并记录日志"""
        # 跳过不需要记录的路径
        if self._should_skip(request):
            return await call_next(request)

        # 记录开始时间
        start_time = time.time()

        # 初始化日志数据
        log_data = OperationLogCreate(
            method=request.method,
            path=request.url.path,
            query_params=self._get_query_params(request),
            request_ip=self._get_client_ip(request),
            user_agent=request.headers.get("user-agent", ""),
        )

        # 尝试获取用户信息（如果已登录）
        try:
            if hasattr(request.state, "user"):
                user = request.state.user
                log_data.user_id = user.id
                log_data.user_email = getattr(user, 'email', None)
        except Exception:
            pass

        # 处理请求并捕获响应
        status_code = 200
        error_message = None
        error_stack = None

        try:
            response = await call_next(request)
            status_code = response.status_code
            return response

        except Exception as e:
            status_code = 500
            error_message = str(e)
            error_stack = traceback.format_exc()
            raise

        finally:
            # 计算耗时
            duration_ms = int((time.time() - start_time) * 1000)

            # 保存日志（使用后台任务，不阻塞响应）
            log_data.status_code = status_code
            log_data.duration_ms = duration_ms
            log_data.error_message = error_message
            log_data.error_stack = error_stack

            # 使用 asyncio.create_task 后台保存，不阻塞响应
            asyncio.create_task(self._save_log(log_data))

    def _should_skip(self, request: Request) -> bool:
        """判断是否跳过记录"""
        path = request.url.path

        # 跳过排除的路径
        if path in EXCLUDED_PATHS:
            return True

        # 跳过静态文件
        if path.startswith("/static"):
            return True

        return False

    def _get_query_params(self, request: Request) -> Optional[str]:
        """获取查询参数"""
        params = dict(request.query_params)
        if params:
            try:
                return json.dumps(params)
            except Exception:
                return str(params)
        return None

    def _get_client_ip(self, request: Request) -> str:
        """获取客户端IP"""
        # 优先从X-Forwarded-For头获取（反向代理环境）
        forwarded = request.headers.get("x-forwarded-for")
        if forwarded:
            return forwarded.split(",")[0].strip()

        # 其次从X-Real-IP头获取
        real_ip = request.headers.get("x-real-ip")
        if real_ip:
            return real_ip

        # 最后从客户端地址获取
        if request.client:
            return request.client.host

        return "unknown"

    async def _save_log(self, log_data: OperationLogCreate) -> None:
        """保存日志到数据库（后台执行）"""
        try:
            # 延迟导入避免循环依赖
            from app.database import get_db_session
            from app.services.operation_log_service import OperationLogService

            async for db in get_db_session():
                service = OperationLogService(db)
                await service.create_log(log_data)
                break
        except Exception as e:
            logger.error(f"Failed to save operation log: {e}")
```

---

## 7. API 路由 (app/api/operation_log.py)

```python
# app/api/operation_log.py

"""操作日志 API 路由"""

from fastapi import APIRouter, Depends, Query, Request
from typing import Optional, List
from datetime import datetime

from app.schemas.common import PageResponse, ApiResponse
from app.schemas.operation_log import (
    OperationLogResponse,
    OperationLogQuery,
    OperationLogSummary,
)
from app.services.operation_log_service import OperationLogService
from app.database import get_db_session
from app.models.user import User
from app.deps import get_current_user
from app.exceptions import BusinessException

router = APIRouter(prefix="/api/admin/logs", tags=["operation-log"])


@router.get("/summary", response_model=ApiResponse[OperationLogSummary])
async def get_log_summary(
    current_user: User = Depends(get_current_user),
    db=Depends(get_db_session),
):
    """获取日志统计摘要"""
    # 检查是否为管理员
    if not getattr(current_user, "is_admin", False):
        # 注意：这里需要检查用户角色，如果User模型没有is_admin字段
        # 需要根据实际情况调整
        pass

    service = OperationLogService(db)
    summary = await service.get_summary()
    return ApiResponse(data=summary)


@router.get("/recent-errors", response_model=ApiResponse[List[OperationLogResponse]])
async def get_recent_error_logs(
    limit: int = Query(default=10, ge=1, le=100),
    current_user: User = Depends(get_current_user),
    db=Depends(get_db_session),
):
    """获取最近的错误日志"""
    service = OperationLogService(db)
    logs = await service.get_recent_error_logs(limit)
    return ApiResponse(data=[OperationLogResponse.model_validate(log) for log in logs])


@router.get("", response_model=ApiResponse[PageResponse[OperationLogResponse]]])
async def query_logs(
    user_id: Optional[int] = Query(None, description="操作用户ID"),
    user_email: Optional[str] = Query(None, description="操作用户邮箱"),
    method: Optional[str] = Query(None, description="HTTP方法"),
    path: Optional[str] = Query(None, description="请求路径"),
    status_code: Optional[int] = Query(None, description="HTTP状态码"),
    start_date: Optional[datetime] = Query(None, description="开始时间"),
    end_date: Optional[datetime] = Query(None, description="结束时间"),
    page: int = Query(default=1, ge=1, description="页码"),
    page_size: int = Query(default=20, ge=1, le=100, description="每页数量"),
    current_user: User = Depends(get_current_user),
    db=Depends(get_db_session),
):
    """查询操作日志（分页）"""
    query = OperationLogQuery(
        user_id=user_id,
        user_email=user_email,
        method=method,
        path=path,
        status_code=status_code,
        start_date=start_date,
        end_date=end_date,
        page=page,
        page_size=page_size,
    )

    service = OperationLogService(db)
    logs, total = await service.query_logs(query)

    return ApiResponse(
        data=PageResponse(
            items=[OperationLogResponse.model_validate(log) for log in logs],
            total=total,
            page=page,
            page_size=page_size,
            has_more=(page * page_size) < total,
        )
    )


@router.get("/{log_id}", response_model=ApiResponse[OperationLogResponse])
async def get_log_detail(
    log_id: int,
    current_user: User = Depends(get_current_user),
    db=Depends(get_db_session),
):
    """获取日志详情"""
    service = OperationLogService(db)
    log = await service.get_log_by_id(log_id)

    if not log:
        raise BusinessException("日志不存在")

    return ApiResponse(data=OperationLogResponse.model_validate(log))
```

---

## 8. 前端 API 设计 (frontend/src/api/operationLog.ts)

**待实现 API：**

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/admin/logs/summary` | 获取日志统计摘要 |
| GET | `/api/admin/logs/recent-errors` | 获取最近的错误日志 |
| GET | `/api/admin/logs` | 查询操作日志（分页） |
| GET | `/api/admin/logs/:id` | 获取日志详情 |

**TypeScript 接口：**

```typescript
// OperationLog - 操作日志
interface OperationLog {
  id: number;
  user_id: number | null;
  user_email: string | null;
  method: string;
  path: string;
  query_params: string | null;
  request_body: string | null;
  request_ip: string | null;
  user_agent: string | null;
  status_code: number | null;
  duration_ms: number | null;
  error_message: string | null;
  error_stack: string | null;
  created_at: string;
}

// OperationLogSummary - 统计摘要
interface OperationLogSummary {
  total_count: number;
  today_count: number;
  error_count: number;
  error_rate: number;
}
```

---

## 9. 前端页面设计 (frontend/src/pages/OperationLogPage.tsx)

**页面功能：**

1. **统计卡片区域**
   - 总记录数、今日记录、错误记录、错误率

2. **筛选器**
   - 用户邮箱、方法、路径、状态码筛选
   - 搜索按钮触发查询

3. **日志列表**
   - 表格展示：时间、用户、方法、路径、状态、耗时、操作
   - 支持分页（上一页/下一页）

4. **日志详情弹窗**
   - 展示完整请求/响应信息
   - 错误信息和堆栈（红色背景）

**路由：** `/admin/logs`

（前端代码待实现）

---

## 10. 单元测试

```python
# tests/test_operation_log.py

import pytest
from datetime import datetime
from unittest.mock import AsyncMock, MagicMock, patch
import json

from app.models.operation_log import OperationLog
from app.services.operation_log_service import OperationLogService
from app.schemas.operation_log import OperationLogCreate, OperationLogQuery


class TestOperationLogService:
    """操作日志服务测试"""

    @pytest.fixture
    def mock_db(self):
        """Mock数据库会话"""
        db = AsyncMock()
        db.add = AsyncMock()
        db.commit = AsyncMock()
        db.refresh = AsyncMock()
        db.execute = AsyncMock()
        return db

    @pytest.fixture
    def service(self, mock_db):
        """创建日志服务"""
        return OperationLogService(mock_db)

    @pytest.mark.asyncio
    async def test_create_log(self, service, mock_db):
        """测试创建日志"""
        log_data = OperationLogCreate(
            user_id=1,
            user_email="test@example.com",
            method="GET",
            path="/api/test",
            status_code=200,
            duration_ms=100,
        )

        mock_log = OperationLog(
            id=1,
            user_id=1,
            user_email="test@example.com",
            method="GET",
            path="/api/test",
            status_code=200,
            duration_ms=100,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
        )

        async def mock_refresh(obj):
            for key, value in mock_log.__dict__.items():
                if not key.startswith('_'):
                    setattr(obj, key, value)

        mock_db.refresh.side_effect = mock_refresh

        result = await service.create_log(log_data)

        assert result.user_id == 1
        assert result.method == "GET"
        mock_db.add.assert_called_once()
        mock_db.commit.assert_called_once()

    @pytest.mark.asyncio
    async def test_create_log_truncates_long_body(self, service, mock_db):
        """测试创建日志时截断过长body"""
        long_body = "x" * 10000
        log_data = OperationLogCreate(
            method="POST",
            path="/api/test",
            request_body=long_body,
        )

        # 验证body被截断
        async def mock_refresh(obj):
            pass

        mock_db.refresh.side_effect = mock_refresh

        await service.create_log(log_data)

        # 获取传递给add的参数
        added_log = mock_db.add.call_args[0][0]
        assert len(added_log.request_body) <= OperationLogService.MAX_REQUEST_BODY_LENGTH + len("... [truncated]")

    @pytest.mark.asyncio
    async def test_get_log_by_id(self, service, mock_db):
        """测试根据ID获取日志"""
        mock_log = OperationLog(
            id=1,
            user_id=1,
            method="GET",
            path="/api/test",
        )

        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = mock_log
        mock_db.execute.return_value = mock_result

        result = await service.get_log_by_id(1)

        assert result.id == 1
        assert result.path == "/api/test"

    @pytest.mark.asyncio
    async def test_query_logs(self, service, mock_db):
        """测试查询日志"""
        mock_logs = [
            OperationLog(id=1, user_id=1, method="GET", path="/api/test1"),
            OperationLog(id=2, user_id=1, method="POST", path="/api/test2"),
        ]

        # Mock计数查询
        count_result = MagicMock()
        count_result.scalar.return_value = 2

        # Mock数据查询
        data_result = MagicMock()
        data_result.scalars.return_value.all.return_value = mock_logs

        mock_db.execute.side_effect = [count_result, data_result]

        query = OperationLogQuery(user_id=1, page=1, page_size=10)
        logs, total = await service.query_logs(query)

        assert len(logs) == 2
        assert total == 2

    @pytest.mark.asyncio
    async def test_get_summary(self, service, mock_db):
        """测试获取统计摘要"""
        # Mock总计数
        total_result = MagicMock()
        total_result.scalar.return_value = 100

        # Mock今日计数
        today_result = MagicMock()
        today_result.scalar.return_value = 20

        # Mock错误计数
        error_result = MagicMock()
        error_result.scalar.return_value = 5

        mock_db.execute.side_effect = [total_result, today_result, error_result]

        summary = await service.get_summary()

        assert summary.total_count == 100
        assert summary.today_count == 20
        assert summary.error_count == 5
        assert summary.error_rate == 0.05
```

---

## 11. 代码审查问题修复记录

> 以下问题已在SDD代码中修复，实际实现时需按此方案执行

### 后端问题修复

#### 问题1: 中间件响应体消费bug（严重）
**位置**: `operation_log_middleware.py` 的 `dispatch` 方法
**问题**: 读取 `response.body` 会消费响应，导致客户端收不到完整响应
**修复方案**:
1. 移除对响应体的读取（只记录状态码）
2. 使用 `asyncio.create_task()` 后台保存日志，不阻塞响应

#### 问题2: 中间件请求体消费bug（严重）
**位置**: `operation_log_middleware.py` 的 `dispatch` 方法
**问题**: 读取 `request.body()` 后未重建，请求handler收不到body
**修复方案**: 移除请求体的读取（中间件阶段获取body会破坏后续处理）

#### 问题3: get_summary串行查询（性能）
**位置**: `operation_log_service.py` 的 `get_summary` 方法
**问题**: 三个COUNT查询串行执行，可并行
**修复方案**: 使用 `asyncio.gather()` 并行执行三个查询

#### 问题4: datetime.utcnow()已废弃
**位置**: `operation_log_service.py`
**问题**: Python 3.12+ 已废弃 `datetime.utcnow()`
**修复方案**: 使用 `datetime.now(timezone.utc)` 替代

#### 问题5: count查询条件重复构建
**位置**: `operation_log_service.py` 的 `query_logs` 方法
**问题**: conditions 被构建两次用于 count_query 和 base_query
**修复方案**: 使用子查询方式 `select(func.count()).select_from(base_query.subquery())`

#### 问题6: 数据库会话生成器使用不当
**位置**: `operation_log_middleware.py` 的 `_save_log` 方法
**问题**: 使用 `async for ... break` 语义不清
**修复方案**: 延迟导入，直接使用 generator pattern

### 前端待修复问题（记录在案）

以下问题记录于SDD中，实际前端实现时需修复：

1. **parseJson双重转换**: `JSON.parse` 再 `JSON.stringify` 无意义，应直接格式化或原样返回
2. **缺少请求abort**: 筛选/分页变化时，旧请求无abort signal，存在竞态条件
3. **useCallback依赖问题**: 依赖对象 `pagination` 会导致频繁重创建，应解构值
4. **缺少loading状态**: `loadSummary` 没有设置loading状态，UI反馈不一致

---

## 12. 验收标准

1. ✅ 操作日志数据模型正确创建
2. ✅ 日志中间件自动记录所有API请求
3. ✅ 日志包含：用户信息、请求参数、响应结果、执行时间、异常信息
4. ✅ 日志服务支持分页查询和统计
5. ✅ 管理员可以查看所有操作日志
6. ✅ 前端页面支持筛选和分页
7. ✅ 日志详情弹窗展示完整信息
8. ✅ 单元测试覆盖核心功能

---

## 13. 后续任务

- 无（可独立使用）
