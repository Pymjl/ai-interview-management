# SDD-01: 项目基础配置

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**: 无
> **目标**: 创建项目的基础配置文件和依赖管理

---

## 1. 概述

本任务负责创建项目的核心配置文件，包括：
- `pyproject.toml`: Python 项目配置和依赖管理
- `.env.example`: 环境变量示例文件
- `app/config.py`: 配置管理模块
- `app/__init__.py`: 应用初始化

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── __init__.py
│   └── config.py
├── pyproject.toml
├── uv.lock
├── .env.example
└── ...
```

---

## 3. pyproject.toml

```toml
# pyproject.toml
[project]
name = "interview-guide"
version = "1.0.0"
description = "AI驱动的面试平台后端服务"
requires-python = ">=3.12"
readme = "README.md"

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",
    "ruff>=0.6.0",
    "mypy>=1.10.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",
    "ruff>=0.6.0",
    "mypy>=1.10.0",
]

[tool.ruff]
line-length = 120
target-version = "py312"
select = ["E", "F", "I", "N", "W", "UP"]
ignore = ["E501"]

[tool.mypy]
python_version = "3.12"
warn_return_any = true
warn_unused_ignores = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### 详细依赖列表

```toml
dependencies = [
    # Web Framework
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "python-multipart>=0.0.12",

    # LangChain & LangGraph
    "langchain>=0.3.0",
    "langchain-community>=0.3.0",
    "langchain-core>=0.3.0",
    "langgraph>=0.2.0",
    "langgraph-sdk>=0.1.0",

    # AI Providers (OpenAI Compatible API)
    "langchain-openai>=0.2.0",

    # Database
    "sqlalchemy[asyncio]>=2.0.0",
    "asyncpg>=0.30.0",
    "pgvector>=0.3.0",
    "alembic>=1.14.0",

    # Redis
    "redis>=5.0.0",

    # MinIO (S3 Compatible Storage)
    "minio>=7.2.0",

    # Document Processing
    "tika>=2.9.0",
    "python-docx>=1.1.0",
    "pypdf>=5.0.0",

    # PDF
    "reportlab>=4.2.0",

    # Validation & Serialization
    "pydantic>=2.10.0",
    "pydantic-settings>=2.0.0",

    # Rate Limiting
    "slowapi>=0.1.9",
    "limits>=3.0.0",

    # Authentication & Security
    "python-jose[cryptography]>=3.3.0",
    "bcrypt>=4.1.0",
    "passlib[bcrypt]>=1.7.4",

    # Utilities
    "python-dotenv>=1.0.0",
    "httpx>=0.27.0",
]
```

---

## 4. 配置管理 (app/config.py)

```python
# app/config.py

from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache
from typing import Optional


class DatabaseSettings(BaseSettings):
    """数据库配置"""
    model_config = SettingsConfigDict(env_prefix="POSTGRES_")

    HOST: str = "localhost"
    PORT: int = 5432
    DB: str = "interview_guide"
    USER: str = "postgres"
    PASSWORD: str = "postgres"

    @property
    def DATABASE_URL(self) -> str:
        return f"postgresql+asyncpg://{self.USER}:{self.PASSWORD}@{self.HOST}:{self.PORT}/{self.DB}"

    @property
    def SYNC_DATABASE_URL(self) -> str:
        return f"postgresql://{self.USER}:{self.PASSWORD}@{self.HOST}:{self.PORT}/{self.DB}"


class RedisSettings(BaseSettings):
    """Redis配置"""
    model_config = SettingsConfigDict(env_prefix="REDIS_")

    HOST: str = "localhost"
    PORT: int = 6379
    DB: int = 0
    PASSWORD: Optional[str] = None

    @property
    def REDIS_URL(self) -> str:
        if self.PASSWORD:
            return f"redis://:{self.PASSWORD}@{self.HOST}:{self.PORT}/{self.DB}"
        return f"redis://{self.HOST}:{self.PORT}/{self.DB}"


class LLMSettings(BaseSettings):
    """LLM配置 - 支持任意OpenAI兼容API"""
    model_config = SettingsConfigDict(env_prefix="LLM_")

    MODEL: str = "gpt-oss:20b"
    BASE_URL: str = "http://localhost:11434/v1"
    API_KEY: str = "not-needed"


class EmbeddingSettings(BaseSettings):
    """Embedding配置 - 支持任意OpenAI兼容API"""
    model_config = SettingsConfigDict(env_prefix="EMBEDDING_")

    MODEL: str = "nomic-embed-text"
    BASE_URL: str = "http://localhost:11434/v1"
    API_KEY: str = "not-needed"


class JWTSettings(BaseSettings):
    """JWT配置"""
    model_config = SettingsConfigDict(env_prefix="JWT_")

    SECRET_KEY: str = "your-super-secret-key-change-in-production"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7


class MiniIOSettings(BaseSettings):
    """MinIO配置"""
    model_config = SettingsConfigDict(env_prefix="MINIO_")

    ENDPOINT: str = "localhost:9000"
    ACCESS_KEY: str = "minioadmin"
    SECRET_KEY: str = "minioadmin"
    BUCKET: str = "interview-guide"
    SECURE: bool = False


class AppSettings(BaseSettings):
    """应用配置"""
    model_config = SettingsConfigDict(env_file=".env")

    APP_NAME: str = "Interview Guide API"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False

    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:5173"]

    # 上传文件限制
    MAX_FILE_SIZE: int = 10 * 1024 * 1024  # 10MB


class Settings(BaseSettings):
    """全局配置汇总"""
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()
    llm: LLMSettings = LLMSettings()
    embedding: EmbeddingSettings = EmbeddingSettings()
    jwt: JWTSettings = JWTSettings()
    minio: MiniIOSettings = MiniIOSettings()
    app: AppSettings = AppSettings()


@lru_cache()
def get_settings() -> Settings:
    """获取全局配置单例"""
    return Settings()
```

---

## 5. 环境变量示例 (.env.example)

```bash
# =============================================
# Interview Guide 环境变量配置
# =============================================

# 应用配置
DEBUG=false

# PostgreSQL + pgvector
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=interview_guide
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
# REDIS_PASSWORD=your_redis_password

# LLM (OpenAI兼容API - 支持Ollama/vLLM/LocalAI/OpenAI等)
LLM_MODEL=gpt-oss:20b
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=not-needed

# Embedding (OpenAI兼容API)
EMBEDDING_MODEL=nomic-embed-text
EMBEDDING_BASE_URL=http://localhost:11434/v1
EMBEDDING_API_KEY=not-needed

# JWT
JWT_SECRET_KEY=your-super-secret-key-change-in-production
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7

# MinIO (S3 Compatible)
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin123
MINIO_BUCKET=interview-guide
MINIO_SECURE=false
```

---

## 6. 应用初始化 (app/__init__.py)

```python
# app/__init__.py

"""Interview Guide - AI驱动的面试平台后端服务"""

__version__ = "1.0.0"
__app_name__ = "interview-guide"
```

---

## 7. uv 常用命令

```bash
# 安装依赖 (根据 pyproject.toml)
uv sync

# 添加新依赖
uv add fastapi
uv add "fastapi>=0.115.0"

# 添加开发依赖
uv add --dev pytest pytest-asyncio

# 锁定依赖版本 (生成 uv.lock)
uv lock

# 更新依赖
uv lock --upgrade

# 运行脚本
uv run python script.py

# 虚拟环境管理
uv venv                      # 创建虚拟环境
source .venv/bin/activate    # 激活虚拟环境 (Linux/Mac)
.venv\Scripts\activate       # 激活虚拟环境 (Windows)
```

---

## 8. 单元测试

```python
# tests/test_config.py

import pytest
from app.config import (
    DatabaseSettings,
    RedisSettings,
    LLMSettings,
    Settings,
    get_settings,
)


class TestDatabaseSettings:
    """数据库配置测试"""

    def test_default_values(self):
        """测试默认值"""
        settings = DatabaseSettings()
        assert settings.HOST == "localhost"
        assert settings.PORT == 5432
        assert settings.DB == "interview_guide"

    def test_database_url(self):
        """测试数据库URL生成"""
        settings = DatabaseSettings()
        url = settings.DATABASE_URL
        assert "postgresql+asyncpg://" in url
        assert settings.DB in url

    def test_env_override(self):
        """测试环境变量覆盖"""
        settings = DatabaseSettings(
            HOST="db.example.com",
            PORT=5433,
            DB="testdb"
        )
        assert settings.HOST == "db.example.com"
        assert settings.PORT == 5433
        assert settings.DB == "testdb"


class TestRedisSettings:
    """Redis配置测试"""

    def test_default_values(self):
        """测试默认值"""
        settings = RedisSettings()
        assert settings.HOST == "localhost"
        assert settings.PORT == 6379

    def test_redis_url_without_password(self):
        """测试无密码的Redis URL"""
        settings = RedisSettings()
        url = settings.REDIS_URL
        assert "redis://" in url
        assert "localhost" in url

    def test_redis_url_with_password(self):
        """测试有密码的Redis URL"""
        settings = RedisSettings(PASSWORD="secret")
        url = settings.REDIS_URL
        assert "redis://:secret@" in url


class TestLLMSettings:
    """LLM配置测试"""

    def test_default_values(self):
        """测试默认值"""
        settings = LLMSettings()
        assert settings.MODEL == "gpt-oss:20b"
        assert "localhost" in settings.BASE_URL


class TestSettingsSingleton:
    """全局配置单例测试"""

    def test_get_settings_returns_same_instance(self):
        """测试配置单例"""
        settings1 = get_settings()
        settings2 = get_settings()
        assert settings1 is settings2
```

---

## 9. 验收标准

1. ✅ `pyproject.toml` 包含所有必要的依赖
2. ✅ `.env.example` 包含所有环境变量并有清晰注释
3. ✅ `app/config.py` 提供所有配置类的定义
4. ✅ 配置类支持从环境变量读取
5. ✅ 所有配置类有默认值
6. ✅ 单元测试覆盖配置加载逻辑
7. ✅ `uv sync` 能够成功安装所有依赖

---

## 10. 后续任务

- [SDD-02: 数据库基础架构](SDD-02-数据库基础架构.md) - 依赖本任务的配置
