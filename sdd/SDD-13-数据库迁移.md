# SDD-13: 数据库迁移

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](SDD-02-数据库基础架构.md)
> **目标**: 创建 Alembic 迁移脚本

---

## 1. 概述

本任务负责创建 Alembic 数据库迁移配置和初始迁移脚本。

---

## 2. Alembic 配置 (alembic.ini)

```ini
# alembic.ini

[alembic]
script_location = migrations
prepend_sys_path = .
version_path_separator = os

[post_write_hooks]

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

---

## 3. 环境配置 (migrations/env.py)

```python
# migrations/env.py

import asyncio
from logging.config import fileConfig

from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from alembic import context

from app.config import get_settings
from app.models.base import Base
from app.models.user import User
from app.models.resume import Resume, ResumeAnalysis
from app.models.interview import InterviewSession, InterviewAnswer
from app.models.knowledgebase import (
    KnowledgeBase, RagChatSession, RagChatMessage, Document
)

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

settings = get_settings()


def get_url():
    return settings.database.DATABASE_URL


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode."""
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)

    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Run migrations in 'online' mode."""
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()

    connectable = async_engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode."""
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

## 4. 初始迁移 (migrations/versions/001_initial_schema.py)

```python
# migrations/versions/001_initial_schema.py

"""初始数据库 schema

Revision ID: 001
Revises:
Create Date: 2026-04-04
"""

from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision: str = '001'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # 创建 user 表
    op.create_table(
        'user',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('email', sa.String(length=255), nullable=False),
        sa.Column('username', sa.String(length=100), nullable=False),
        sa.Column('name', sa.String(length=100), nullable=True),
        sa.Column('password_hash', sa.String(length=255), nullable=False),
        sa.Column('phone', sa.String(length=20), nullable=True),
        sa.Column('gender', sa.String(length=10), nullable=True),
        sa.Column('age', sa.Integer(), nullable=True),
        sa.Column('avatar_url', sa.String(length=500), nullable=True),
        sa.Column('school', sa.String(length=255), nullable=True),
        sa.Column('address', sa.String(length=500), nullable=True),
        sa.Column('homepage', sa.String(length=500), nullable=True),
        sa.Column('is_active', sa.Boolean(), default=True, nullable=False),
        sa.Column('last_login_at', sa.DateTime(), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('created_by', sa.BigInteger(), nullable=True),
        sa.Column('last_updated_by', sa.BigInteger(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email'),
    )
    op.create_index('ix_user_email', 'user', ['email'], unique=True)

    # 创建 resume 表
    op.create_table(
        'resume',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('user_id', sa.BigInteger(), nullable=False),
        sa.Column('file_hash', sa.String(length=64), nullable=False),
        sa.Column('original_filename', sa.String(length=255), nullable=False),
        sa.Column('file_size', sa.BigInteger(), nullable=False),
        sa.Column('content_type', sa.String(length=100), nullable=False),
        sa.Column('storage_key', sa.String(length=500), nullable=False),
        sa.Column('storage_url', sa.String(length=1000), nullable=True),
        sa.Column('resume_text', sa.Text(), nullable=True),
        sa.Column('uploaded_at', sa.DateTime(), nullable=True),
        sa.Column('last_accessed_at', sa.DateTime(), nullable=True),
        sa.Column('access_count', sa.Integer(), default=0, nullable=False),
        sa.Column('analyze_status', sa.String(length=20), default='PENDING', nullable=False),
        sa.Column('analyze_error', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('created_by', sa.BigInteger(), nullable=True),
        sa.Column('last_updated_by', sa.BigInteger(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('file_hash'),
    )
    op.create_index('ix_resume_file_hash', 'resume', ['file_hash'], unique=True)
    op.create_index('ix_resume_user_id', 'resume', ['user_id'], unique=False)

    # 创建 interview_session 表
    op.create_table(
        'interview_session',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('user_id', sa.BigInteger(), nullable=False),
        sa.Column('session_id', sa.String(length=36), nullable=False),
        sa.Column('resume_id', sa.BigInteger(), nullable=False),
        sa.Column('total_questions', sa.Integer(), nullable=False),
        sa.Column('current_question_index', sa.Integer(), default=0, nullable=False),
        sa.Column('status', sa.String(length=20), default='CREATED', nullable=False),
        sa.Column('questions_json', sa.Text(), nullable=True),
        sa.Column('overall_score', sa.Integer(), nullable=True),
        sa.Column('overall_feedback', sa.Text(), nullable=True),
        sa.Column('strengths_json', sa.Text(), nullable=True),
        sa.Column('improvements_json', sa.Text(), nullable=True),
        sa.Column('reference_answers_json', sa.Text(), nullable=True),
        sa.Column('evaluate_status', sa.String(length=20), default='PENDING', nullable=False),
        sa.Column('evaluate_error', sa.Text(), nullable=True),
        sa.Column('completed_at', sa.DateTime(), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('created_by', sa.BigInteger(), nullable=True),
        sa.Column('last_updated_by', sa.BigInteger(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('session_id'),
    )
    op.create_index('ix_interview_session_session_id', 'interview_session', ['session_id'], unique=True)
    op.create_index('ix_interview_session_user_id', 'interview_session', ['user_id'], unique=False)

    # 创建 interview_answer 表
    op.create_table(
        'interview_answer',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('session_id', sa.BigInteger(), nullable=False),
        sa.Column('question_index', sa.Integer(), nullable=False),
        sa.Column('question', sa.Text(), nullable=False),
        sa.Column('category', sa.String(length=50), nullable=True),
        sa.Column('user_answer', sa.Text(), nullable=True),
        sa.Column('score', sa.Integer(), nullable=True),
        sa.Column('feedback', sa.Text(), nullable=True),
        sa.Column('reference_answer', sa.Text(), nullable=True),
        sa.Column('key_points_json', sa.Text(), nullable=True),
        sa.Column('answered_at', sa.DateTime(), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.PrimaryKeyConstraint('id'),
    )

    # 创建 knowledge_base 表
    op.create_table(
        'knowledge_base',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('user_id', sa.BigInteger(), nullable=False),
        sa.Column('file_hash', sa.String(length=64), nullable=False),
        sa.Column('name', sa.String(length=255), nullable=False),
        sa.Column('category', sa.String(length=100), nullable=True),
        sa.Column('original_filename', sa.String(length=255), nullable=True),
        sa.Column('file_size', sa.BigInteger(), nullable=True),
        sa.Column('content_type', sa.String(length=100), nullable=True),
        sa.Column('storage_key', sa.String(length=500), nullable=True),
        sa.Column('storage_url', sa.String(length=1000), nullable=True),
        sa.Column('uploaded_at', sa.DateTime(), nullable=True),
        sa.Column('last_accessed_at', sa.DateTime(), nullable=True),
        sa.Column('access_count', sa.Integer(), default=0, nullable=False),
        sa.Column('question_count', sa.Integer(), default=0, nullable=False),
        sa.Column('vector_status', sa.String(length=20), default='PENDING', nullable=False),
        sa.Column('vector_error', sa.Text(), nullable=True),
        sa.Column('chunk_count', sa.Integer(), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('created_by', sa.BigInteger(), nullable=True),
        sa.Column('last_updated_by', sa.BigInteger(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('file_hash'),
    )
    op.create_index('ix_knowledge_base_file_hash', 'knowledge_base', ['file_hash'], unique=True)
    op.create_index('ix_knowledge_base_user_id', 'knowledge_base', ['user_id'], unique=False)

    # 创建 document 向量表
    op.create_table(
        'document',
        sa.Column('id', sa.Text(), nullable=False),
        sa.Column('user_id', sa.BigInteger(), nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('metadata', postgresql.JSONB(astext_type=sa.Text()), nullable=True),
        sa.Column('embedding', postgresql.JSONB(astext_type=sa.Text()), nullable=True),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.PrimaryKeyConstraint('id'),
    )
    op.create_index('ix_document_user_id', 'document', ['user_id'], unique=False)

    # 创建 rag_chat_session 表
    op.create_table(
        'rag_chat_session',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('user_id', sa.BigInteger(), nullable=False),
        sa.Column('title', sa.String(length=255), default='新会话', nullable=False),
        sa.Column('status', sa.String(length=20), default='ACTIVE', nullable=False),
        sa.Column('is_pinned', sa.Boolean(), default=False, nullable=False),
        sa.Column('message_count', sa.Integer(), default=0, nullable=False),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.PrimaryKeyConstraint('id'),
    )
    op.create_index('ix_rag_chat_session_user_id', 'rag_chat_session', ['user_id'], unique=False)

    # 创建 rag_chat_message 表
    op.create_table(
        'rag_chat_message',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('session_id', sa.BigInteger(), nullable=False),
        sa.Column('type', sa.String(length=20), default='USER', nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('message_order', sa.Integer(), nullable=False),
        sa.Column('completed', sa.Boolean(), default=True, nullable=False),
        sa.Column('created_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(), default=sa.text('now()'), nullable=False),
        sa.PrimaryKeyConstraint('id'),
    )

    # 创建关联表
    op.create_table(
        'rag_chat_kb_association',
        sa.Column('session_id', sa.BigInteger(), nullable=True),
        sa.Column('kb_id', sa.BigInteger(), nullable=True),
    )


def downgrade() -> None:
    op.drop_table('rag_chat_kb_association')
    op.drop_table('rag_chat_message')
    op.drop_table('rag_chat_session')
    op.drop_table('document')
    op.drop_table('knowledge_base')
    op.drop_table('interview_answer')
    op.drop_table('interview_session')
    op.drop_table('resume')
    op.drop_table('user')
```

---

## 5. 迁移命令

```bash
# 生成迁移
alembic revision --autogenerate -m "Initial schema"

# 执行迁移
alembic upgrade head

# 回滚
alembic downgrade -1

# 检查当前版本
alembic current

# 历史记录
alembic history
```

---

## 6. 验收标准

1. ✅ Alembic 配置正确
2. ✅ 初始迁移脚本创建所有表
3. ✅ 索引正确创建
4. ✅ 外键关系正确
5. ✅ 迁移可回滚
