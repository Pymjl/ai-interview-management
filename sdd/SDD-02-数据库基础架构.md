# SDD-02: 数据库基础架构

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**: [SDD-01: 项目基础配置](SDD-01-项目基础配置.md)
> **目标**: 创建 SQLAlchemy 基础架构、Mixins、数据库连接管理

---

## 1. 概述

本任务负责创建数据库层的基础架构，包括：
- SQLAlchemy Base 基类
- Mixin 类（TimestampMixin、UserMixin、CreatorMixin）
- 数据库连接管理（async session）
- 所有业务模型的基类 UserBaseModel

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── __init__.py
│   ├── config.py              # 来自 SDD-01
│   ├── database.py            # 新增
│   └── models/
│       ├── __init__.py
│       ├── base.py            # 新增
│       └── ...
└── ...
```

---

## 3. 数据库连接 (app/database.py)

```python
# app/database.py

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    AsyncEngine,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy.pool import NullPool
from typing import AsyncGenerator
import logging

from app.config import get_settings

logger = logging.getLogger(__name__)

settings = get_settings()


def create_engine() -> AsyncEngine:
    """创建异步数据库引擎"""
    return create_async_engine(
        settings.database.DATABASE_URL,
        echo=settings.app.DEBUG,
        poolclass=NullPool,  # 生产环境使用连接池
        pool_size=10,
        max_overflow=20,
    )


_engine: AsyncEngine | None = None
_async_session_factory: async_sessionmaker[AsyncSession] | None = None


def get_engine() -> AsyncEngine:
    """获取数据库引擎（单例）"""
    global _engine
    if _engine is None:
        _engine = create_engine()
    return _engine


def get_session_factory() -> async_sessionmaker[AsyncSession]:
    """获取会话工厂（单例）"""
    global _async_session_factory
    if _async_session_factory is None:
        engine = get_engine()
        _async_session_factory = async_sessionmaker(
            bind=engine,
            class_=AsyncSession,
            expire_on_commit=False,
            autocommit=False,
            autoflush=False,
        )
    return _async_session_factory


async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """获取数据库会话（依赖注入用）

    用法:
        @router.get("/items")
        async def get_items(db: AsyncSession = Depends(get_db_session)):
            ...
    """
    factory = get_session_factory()
    async with factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


async def init_db() -> None:
    """初始化数据库（创建表等）"""
    engine = get_engine()
    # 导入所有模型以确保它们被注册
    from app.models import base, user, resume, interview, knowledgebase  # noqa

    async with engine.begin() as conn:
        # 可以在这里执行表创建或其他初始化
        # 注意：生产环境应使用 Alembic 迁移
        pass
    logger.info("Database initialized successfully")


async def close_db() -> None:
    """关闭数据库连接"""
    global _engine, _async_session_factory
    if _engine:
        await _engine.dispose()
        _engine = None
        _async_session_factory = None
    logger.info("Database connections closed")
```

---

## 4. 基础模型 (app/models/base.py)

```python
# app/models/base.py

from sqlalchemy import Column, Integer, DateTime, BigInteger, Boolean
from sqlalchemy.orm import DeclarativeBase
from datetime import datetime
from typing import Optional


class Base(DeclarativeBase):
    """所有SQLAlchemy模型的基类"""
    pass


class TimestampMixin:
    """时间戳混入 - 包含 created_at 和 updated_at 字段"""

    created_at = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False,
        comment="创建时间"
    )
    updated_at = Column(
        DateTime,
        default=datetime.utcnow,
        onupdate=datetime.utcnow,
        nullable=False,
        comment="最后更新时间"
    )


class UserMixin:
    """用户关联混入 - 包含 user_id 字段用于数据隔离"""

    user_id = Column(
        BigInteger,
        nullable=False,
        index=True,
        comment="关联用户ID"
    )


class CreatorMixin:
    """创建者混入 - 用于审计字段"""

    created_by = Column(
        BigInteger,
        nullable=True,
        comment="创建者用户ID"
    )
    last_updated_by = Column(
        BigInteger,
        nullable=True,
        comment="最后更新者用户ID"
    )


class SoftDeleteMixin:
    """软删除混入"""

    is_deleted = Column(
        Boolean,
        default=False,
        nullable=False,
        index=True,
        comment="是否已删除"
    )
    deleted_at = Column(
        DateTime,
        nullable=True,
        comment="删除时间"
    )


class UserBaseModel(Base, TimestampMixin, CreatorMixin):
    """用户相关业务模型基类

    所有与用户关联的业务模型应继承此类
    自动包含: created_at, updated_at, created_by, last_updated_by
    子类需要添加 user_id 字段
    """

    __abstract__ = True

    def set_created_by(self, user_id: int) -> None:
        """设置创建者"""
        self.created_by = user_id
        self.last_updated_by = user_id

    def set_updated_by(self, user_id: int) -> None:
        """设置更新者"""
        self.last_updated_by = user_id


class AsyncTaskStatus:
    """异步任务状态枚举"""

    PENDING = "PENDING"
    PROCESSING = "PROCESSING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
```

---

## 5. 模型导出 (app/models/__init__.py)

```python
# app/models/__init__.py

"""数据库模型"""

from app.models.base import (
    Base,
    TimestampMixin,
    UserMixin,
    CreatorMixin,
    SoftDeleteMixin,
    UserBaseModel,
    AsyncTaskStatus,
)

# 导入所有模型以确保它们被注册到 Base
from app.models.user import User
from app.models.resume import Resume, ResumeAnalysis
from app.models.interview import InterviewSession, InterviewAnswer
from app.models.knowledgebase import (
    KnowledgeBase,
    RagChatSession,
    RagChatMessage,
    Document,
)

__all__ = [
    "Base",
    "TimestampMixin",
    "UserMixin",
    "CreatorMixin",
    "SoftDeleteMixin",
    "UserBaseModel",
    "AsyncTaskStatus",
    "User",
    "Resume",
    "ResumeAnalysis",
    "InterviewSession",
    "InterviewAnswer",
    "KnowledgeBase",
    "RagChatSession",
    "RagChatMessage",
    "Document",
]
```

---

## 6. 模型占位文件

为了确保导入正常工作，需要创建以下占位文件：

### app/models/user.py
```python
# app/models/user.py
"""用户模型"""

from sqlalchemy import Column, BigInteger, String, Boolean, DateTime, Text
from sqlalchemy.orm import relationship
from datetime import datetime

from app.models.base import Base, TimestampMixin, CreatorMixin


class User(Base, TimestampMixin, CreatorMixin):
    """用户模型"""
    __tablename__ = "user"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(100), nullable=False)
    name = Column(String(100), nullable=True)
    password_hash = Column(String(255), nullable=False)

    # 可选字段
    phone = Column(String(20), nullable=True)
    gender = Column(String(10), nullable=True)
    age = Column(Integer, nullable=True)
    avatar_url = Column(String(500), nullable=True)
    school = Column(String(255), nullable=True)
    address = Column(String(500), nullable=True)
    homepage = Column(String(500), nullable=True)

    # 状态
    is_active = Column(Boolean, default=True)
    last_login_at = Column(DateTime, nullable=True)

    # 关系
    resumes = relationship("Resume", back_populates="user")
    interview_sessions = relationship("InterviewSession", back_populates="user")
    knowledge_bases = relationship("KnowledgeBase", back_populates="user")
    rag_chat_sessions = relationship("RagChatSession", back_populates="user")

    def __repr__(self) -> str:
        return f"<User(id={self.id}, email={self.email})>"
```

### app/models/resume.py
```python
# app/models/resume.py
"""简历模型"""

from sqlalchemy import Column, BigInteger, String, Text, DateTime, Integer, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime

from app.models.base import Base, TimestampMixin, UserMixin, CreatorMixin, AsyncTaskStatus


class Resume(Base, TimestampMixin, UserMixin, CreatorMixin):
    """简历模型"""
    __tablename__ = "resume"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    file_hash = Column(String(64), unique=True, nullable=False, index=True)
    original_filename = Column(String(255), nullable=False)
    file_size = Column(BigInteger, nullable=False)
    content_type = Column(String(100), nullable=False)
    storage_key = Column(String(500), nullable=False)
    storage_url = Column(String(1000), nullable=True)
    resume_text = Column(Text, nullable=True)
    uploaded_at = Column(DateTime, default=datetime.utcnow)
    last_accessed_at = Column(DateTime, nullable=True)
    access_count = Column(Integer, default=0)
    analyze_status = Column(String(20), default=AsyncTaskStatus.PENDING)
    analyze_error = Column(Text, nullable=True)

    # 关系
    user = relationship("User", back_populates="resumes")
    analysis = relationship("ResumeAnalysis", back_populates="resume", uselist=False)
    interview_sessions = relationship("InterviewSession", back_populates="resume")

    def __repr__(self) -> str:
        return f"<Resume(id={self.id}, filename={self.original_filename})>"


class ResumeAnalysis(Base, TimestampMixin):
    """简历分析结果模型"""
    __tablename__ = "resume_analysis"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    resume_id = Column(BigInteger, ForeignKey("resume.id"), unique=True, nullable=False)
    overall_score = Column(Integer, nullable=True)
    content_score = Column(Integer, nullable=True)
    structure_score = Column(Integer, nullable=True)
    skill_match_score = Column(Integer, nullable=True)
    expression_score = Column(Integer, nullable=True)
    project_score = Column(Integer, nullable=True)
    summary = Column(Text, nullable=True)
    strengths_json = Column(Text, nullable=True)
    suggestions_json = Column(Text, nullable=True)
    analyzed_at = Column(DateTime, default=datetime.utcnow)

    # 关系
    resume = relationship("Resume", back_populates="analysis")

    def __repr__(self) -> str:
        return f"<ResumeAnalysis(resume_id={self.resume_id}, score={self.overall_score})>"
```

### app/models/interview.py
```python
# app/models/interview.py
"""面试模块模型"""

from sqlalchemy import Column, BigInteger, String, Text, DateTime, Integer, ForeignKey, Enum as SQLEnum
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

from app.models.base import Base, TimestampMixin, UserMixin, CreatorMixin, AsyncTaskStatus


class SessionStatus(str, enum.Enum):
    """面试会话状态"""
    CREATED = "CREATED"
    IN_PROGRESS = "IN_PROGRESS"
    COMPLETED = "COMPLETED"
    EVALUATED = "EVALUATED"


class InterviewSession(Base, TimestampMixin, UserMixin, CreatorMixin):
    """面试会话模型"""
    __tablename__ = "interview_session"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    session_id = Column(String(36), unique=True, nullable=False, index=True)
    resume_id = Column(BigInteger, ForeignKey("resume.id"), nullable=False)
    total_questions = Column(Integer, nullable=False)
    current_question_index = Column(Integer, default=0)
    status = Column(String(20), default=SessionStatus.CREATED.value)
    questions_json = Column(Text, nullable=True)
    overall_score = Column(Integer, nullable=True)
    overall_feedback = Column(Text, nullable=True)
    strengths_json = Column(Text, nullable=True)
    improvements_json = Column(Text, nullable=True)
    reference_answers_json = Column(Text, nullable=True)
    evaluate_status = Column(String(20), default=AsyncTaskStatus.PENDING)
    evaluate_error = Column(Text, nullable=True)
    completed_at = Column(DateTime, nullable=True)

    # 关系
    user = relationship("User", back_populates="interview_sessions")
    resume = relationship("Resume", back_populates="interview_sessions")
    answers = relationship("InterviewAnswer", back_populates="session", cascade="all, delete-orphan")

    def __repr__(self) -> str:
        return f"<InterviewSession(id={self.id}, session_id={self.session_id})>"


class InterviewAnswer(Base, TimestampMixin):
    """面试答案模型"""
    __tablename__ = "interview_answer"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    session_id = Column(BigInteger, ForeignKey("interview_session.id"), nullable=False)
    question_index = Column(Integer, nullable=False)
    question = Column(Text, nullable=False)
    category = Column(String(50), nullable=True)
    user_answer = Column(Text, nullable=True)
    score = Column(Integer, nullable=True)
    feedback = Column(Text, nullable=True)
    reference_answer = Column(Text, nullable=True)
    key_points_json = Column(Text, nullable=True)
    answered_at = Column(DateTime, nullable=True)

    # 关系
    session = relationship("InterviewSession", back_populates="answers")

    def __repr__(self) -> str:
        return f"<InterviewAnswer(id={self.id}, question_index={self.question_index})>"
```

### app/models/knowledgebase.py
```python
# app/models/knowledgebase.py
"""知识库模块模型"""

from sqlalchemy import Column, BigInteger, String, Text, DateTime, Integer, ForeignKey, Table, Boolean, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

from app.models.base import Base, TimestampMixin, UserMixin, CreatorMixin, AsyncTaskStatus


class ChatSessionStatus(str, enum.Enum):
    """RAG聊天会话状态"""
    ACTIVE = "ACTIVE"
    ARCHIVED = "ARCHIVED"


class MessageType(str, enum.Enum):
    """RAG消息类型"""
    USER = "USER"
    ASSISTANT = "ASSISTANT"


# 关联表
rag_chat_kb_association = Table(
    "rag_chat_kb_association",
    Base.metadata,
    Column("session_id", BigInteger, ForeignKey("rag_chat_session.id")),
    Column("kb_id", BigInteger, ForeignKey("knowledge_base.id"))
)


class KnowledgeBase(Base, TimestampMixin, UserMixin, CreatorMixin):
    """知识库模型"""
    __tablename__ = "knowledge_base"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    file_hash = Column(String(64), unique=True, nullable=False, index=True)
    name = Column(String(255), nullable=False)
    category = Column(String(100), index=True, nullable=True)
    original_filename = Column(String(255), nullable=True)
    file_size = Column(BigInteger, nullable=True)
    content_type = Column(String(100), nullable=True)
    storage_key = Column(String(500), nullable=True)
    storage_url = Column(String(1000), nullable=True)
    uploaded_at = Column(DateTime, default=datetime.utcnow)
    last_accessed_at = Column(DateTime, nullable=True)
    access_count = Column(Integer, default=0)
    question_count = Column(Integer, default=0)
    vector_status = Column(String(20), default=AsyncTaskStatus.PENDING)
    vector_error = Column(Text, nullable=True)
    chunk_count = Column(Integer, nullable=True)

    # 关系
    user = relationship("User", back_populates="knowledge_bases")
    chat_sessions = relationship("RagChatSession", secondary=rag_chat_kb_association)

    def __repr__(self) -> str:
        return f"<KnowledgeBase(id={self.id}, name={self.name})>"


class RagChatSession(Base, TimestampMixin, UserMixin):
    """RAG聊天会话模型"""
    __tablename__ = "rag_chat_session"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    title = Column(String(255), default="新会话")
    status = Column(String(20), default=ChatSessionStatus.ACTIVE.value)
    is_pinned = Column(Boolean, default=False)
    message_count = Column(Integer, default=0)

    # 关系
    user = relationship("User", back_populates="rag_chat_sessions")
    messages = relationship("RagChatMessage", back_populates="session", cascade="all, delete-orphan")
    knowledge_bases = relationship("KnowledgeBase", secondary=rag_chat_kb_association)

    def __repr__(self) -> str:
        return f"<RagChatSession(id={self.id}, title={self.title})>"


class RagChatMessage(Base, TimestampMixin):
    """RAG聊天消息模型"""
    __tablename__ = "rag_chat_message"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    session_id = Column(BigInteger, ForeignKey("rag_chat_session.id"), nullable=False)
    type = Column(String(20), default=MessageType.USER.value)
    content = Column(Text, nullable=False)
    message_order = Column(Integer, nullable=False)
    completed = Column(Boolean, default=True)

    # 关系
    session = relationship("RagChatSession", back_populates="messages")

    def __repr__(self) -> str:
        return f"<RagChatMessage(id={self.id}, type={self.type})>"


class Document(Base, TimestampMixin):
    """向量存储表 (pgvector)"""
    __tablename__ = "document"

    id = Column(Text, primary_key=True)
    user_id = Column(BigInteger, nullable=False, index=True)
    content = Column(Text, nullable=False)
    metadata = Column(JSON, nullable=True)
    embedding = Column(JSON, nullable=True)  # pgvector 使用 JSON 存储

    def __repr__(self) -> str:
        return f"<Document(id={self.id[:20]}...)>"
```

---

## 7. 单元测试

```python
# tests/test_database.py

import pytest
from sqlalchemy import text
from unittest.mock import AsyncMock, patch, MagicMock

from app.database import (
    get_engine,
    get_session_factory,
    get_db_session,
    init_db,
    close_db,
)


class TestDatabaseConnection:
    """数据库连接测试"""

    def test_get_engine_creates_singleton(self):
        """测试引擎单例创建"""
        # 重置全局状态
        import app.database as db_module
        db_module._engine = None
        db_module._async_session_factory = None

        engine1 = get_engine()
        engine2 = get_engine()
        assert engine1 is engine2

    def test_get_session_factory_creates_singleton(self):
        """测试会话工厂单例创建"""
        import app.database as db_module
        db_module._engine = None
        db_module._async_session_factory = None

        factory1 = get_session_factory()
        factory2 = get_session_factory()
        assert factory1 is factory2

    @pytest.mark.asyncio
    async def test_get_db_session_yields_session(self):
        """测试获取数据库会话"""
        mock_session = AsyncMock()
        mock_factory = MagicMock()
        mock_factory.return_value.__aenter__ = AsyncMock(return_value=mock_session)
        mock_factory.return_value.__aexit__ = AsyncMock(return_value=None)

        import app.database as db_module
        db_module._async_session_factory = mock_factory

        sessions = []
        async for session in get_db_session():
            sessions.append(session)

        assert len(sessions) == 1
        assert sessions[0] is mock_session

    @pytest.mark.asyncio
    async def test_get_db_session_commits_on_success(self):
        """测试会话成功时提交"""
        mock_session = AsyncMock()
        mock_factory = MagicMock()
        mock_factory.return_value.__aenter__ = AsyncMock(return_value=mock_session)
        mock_factory.return_value.__aexit__ = AsyncMock(return_value=None)

        import app.database as db_module
        db_module._async_session_factory = mock_factory

        async for _ in get_db_session():
            pass

        mock_session.commit.assert_called_once()
        mock_session.close.assert_called_once()

    @pytest.mark.asyncio
    async def test_get_db_session_rollbacks_on_error(self):
        """测试会话失败时回滚"""
        mock_session = AsyncMock()
        mock_factory = MagicMock()
        mock_factory.return_value.__aenter__ = AsyncMock(return_value=mock_session)
        mock_factory.return_value.__aexit__ = AsyncMock(side_effect=Exception("DB Error"))

        import app.database as db_module
        db_module._async_session_factory = mock_factory

        with pytest.raises(Exception):
            async for _ in get_db_session():
                raise Exception("Test Error")

        mock_session.rollback.assert_called_once()


class TestDatabaseInit:
    """数据库初始化测试"""

    @pytest.mark.asyncio
    async def test_init_db_logs_success(self):
        """测试初始化日志"""
        import app.database as db_module
        db_module._engine = None
        db_module._async_session_factory = None

        mock_engine = AsyncMock()
        mock_engine.begin.return_value.__aenter__ = AsyncMock()
        mock_engine.begin.return_value.__aexit__ = AsyncMock()

        with patch.object(db_module, 'get_engine', return_value=mock_engine):
            await init_db()

    @pytest.mark.asyncio
    async def test_close_db_disposes_engine(self):
        """测试关闭数据库连接"""
        import app.database as db_module

        mock_engine = AsyncMock()
        db_module._engine = mock_engine
        db_module._async_session_factory = None

        await close_db()

        mock_engine.dispose.assert_called_once()
        assert db_module._engine is None
```

```python
# tests/test_models_base.py

import pytest
from datetime import datetime
from sqlalchemy import Column, Integer, String
from unittest.mock import MagicMock

from app.models.base import (
    Base,
    TimestampMixin,
    UserMixin,
    CreatorMixin,
    SoftDeleteMixin,
    UserBaseModel,
    AsyncTaskStatus,
)


class TestTimestampMixin:
    """时间戳混入测试"""

    def test_created_at_default(self):
        """测试创建时间默认值"""
        mixin = TimestampMixin()
        assert mixin.created_at is not None
        assert isinstance(mixin.created_at, datetime)

    def test_updated_at_default(self):
        """测试更新时间默认值"""
        mixin = TimestampMixin()
        assert mixin.updated_at is not None


class TestUserMixin:
    """用户关联混入测试"""

    def test_user_id_column(self):
        """测试user_id字段"""
        mixin = UserMixin()
        assert hasattr(mixin, 'user_id')
        assert mixin.user_id is None or isinstance(mixin.user_id, int)


class TestCreatorMixin:
    """创建者混入测试"""

    def test_created_by_default(self):
        """测试created_by默认值"""
        mixin = CreatorMixin()
        assert hasattr(mixin, 'created_by')
        assert mixin.created_by is None

    def test_last_updated_by_default(self):
        """测试last_updated_by默认值"""
        mixin = CreatorMixin()
        assert hasattr(mixin, 'last_updated_by')
        assert mixin.last_updated_by is None


class TestSoftDeleteMixin:
    """软删除混入测试"""

    def test_is_deleted_default(self):
        """测试is_deleted默认值"""
        mixin = SoftDeleteMixin()
        assert hasattr(mixin, 'is_deleted')
        assert mixin.is_deleted is False

    def test_deleted_at_default(self):
        """测试deleted_at默认值"""
        mixin = SoftDeleteMixin()
        assert hasattr(mixin, 'deleted_at')
        assert mixin.deleted_at is None


class TestAsyncTaskStatus:
    """异步任务状态枚举测试"""

    def test_status_values(self):
        """测试状态值"""
        assert AsyncTaskStatus.PENDING == "PENDING"
        assert AsyncTaskStatus.PROCESSING == "PROCESSING"
        assert AsyncTaskStatus.COMPLETED == "COMPLETED"
        assert AsyncTaskStatus.FAILED == "FAILED"


class TestUserBaseModel:
    """用户基模型测试"""

    def test_set_created_by(self):
        """测试设置创建者"""
        class TestModel(UserBaseModel):
            __tablename__ = "test"

            id = Column(Integer, primary_key=True)
            created_by = Column(Integer, nullable=True)
            last_updated_by = Column(Integer, nullable=True)

        model = TestModel()
        model.set_created_by(123)
        assert model.created_by == 123
        assert model.last_updated_by == 123

    def test_set_updated_by(self):
        """测试设置更新者"""
        class TestModel(UserBaseModel):
            __tablename__ = "test"

            id = Column(Integer, primary_key=True)
            created_by = Column(Integer, nullable=True)
            last_updated_by = Column(Integer, nullable=True)

        model = TestModel()
        model.set_updated_by(456)
        assert model.last_updated_by == 456
        assert model.created_by is None  # 未设置
```

---

## 8. 验收标准

1. ✅ SQLAlchemy Base 基类正确定义
2. ✅ 所有 Mixin 类（TimestampMixin、UserMixin、CreatorMixin、SoftDeleteMixin）正确定义
3. ✅ UserBaseModel 包含所有必要的审计字段
4. ✅ 异步数据库引擎创建函数
5. ✅ 异步会话工厂和依赖注入函数
6. ✅ 所有业务模型已定义（User、Resume、InterviewSession、KnowledgeBase等）
7. ✅ 模型间关系正确定义（relationships）
8. ✅ 单元测试覆盖核心功能
9. ✅ 模型导入不报错

---

## 9. 后续任务

- [SDD-03: 公共工具层](SDD-03-公共工具层.md) - 依赖本任务的配置
- [SDD-13: 数据库迁移](SDD-13-数据库迁移.md) - 依赖本任务的模型定义
