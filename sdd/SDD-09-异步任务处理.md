# SDD-09: 异步任务处理

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](./SDD-02-数据库基础架构.md)
> - [SDD-04: AI服务集成](./SDD-04-AI服务集成.md)
> **目标**: 创建 Redis Stream 异步任务处理框架

---

## 1. 概述

本任务负责创建异步任务处理框架，包括：
- Redis Stream 生产者
- Redis Stream 消费者
- 简历分析 Worker
- 知识库向量化 Worker
- 面试评估 Worker

---

## 2. 生产者 (app/tasks/producer.py)

```python
# app/tasks/producer.py

"""Redis Stream 生产者

发送异步任务到 Redis Stream
"""

import json
from datetime import datetime
from typing import Optional, Dict, Any
from redis.asyncio import Redis

from app.config import get_settings

settings = get_settings()


class StreamProducer:
    """Redis Stream 生产者"""

    STREAM_CONFIGS = {
        "resume_analyze": {
            "stream_key": "resume:analyze:stream",
            "consumer_group": "analyze-group",
        },
        "vectorize": {
            "stream_key": "knowledgebase:vectorize:stream",
            "consumer_group": "vectorize-group",
        },
        "evaluate": {
            "stream_key": "interview:evaluate:stream",
            "consumer_group": "evaluate-group",
        },
    }

    def __init__(self, redis: Optional[Redis] = None):
        self._redis = redis

    async def get_redis(self) -> Redis:
        if self._redis is None:
            self._redis = Redis.from_url(settings.redis.REDIS_URL, decode_responses=True)
        return self._redis

    async def send_task(
        self,
        task_type: str,
        payload: Dict[str, Any],
        task_id: Optional[str] = None,
    ) -> bool:
        """发送任务到 Stream

        Args:
            task_type: 任务类型 (resume_analyze/vectorize/evaluate)
            payload: 任务数据
            task_id: 可选的任务ID

        Returns:
            是否发送成功
        """
        config = self.STREAM_CONFIGS.get(task_type)
        if not config:
            raise ValueError(f"Unknown task type: {task_type}")

        redis = await self.get_redis()

        message = {
            "payload": json.dumps(payload, ensure_ascii=False),
            "timestamp": datetime.utcnow().isoformat(),
            "retry_count": "0",
        }

        if task_id:
            message["task_id"] = task_id

        try:
            await redis.xadd(config["stream_key"], message)
            return True
        except Exception as e:
            print(f"Failed to send task: {e}")
            return False

    async def send_resume_analyze_task(self, resume_id: int, resume_text: str, file_hash: str) -> bool:
        """发送简历分析任务"""
        return await self.send_task(
            "resume_analyze",
            {
                "resume_id": resume_id,
                "resume_text": resume_text,
                "file_hash": file_hash,
                "action": "analyze",
            }
        )

    async def send_vectorize_task(self, kb_id: int, content: str, user_id: int) -> bool:
        """发送向量化任务"""
        return await self.send_task(
            "vectorize",
            {
                "kb_id": kb_id,
                "content": content,
                "user_id": user_id,
                "action": "vectorize",
            }
        )

    async def send_evaluate_task(self, session_id: int, answers: list) -> bool:
        """发送评估任务"""
        return await self.send_task(
            "evaluate",
            {
                "session_id": session_id,
                "answers": answers,
                "action": "evaluate",
            }
        )

    async def close(self) -> None:
        if self._redis:
            await self._redis.close()
            self._redis = None
```

---

## 3. 消费者 (app/tasks/consumer.py)

```python
# app/tasks/consumer.py

"""Redis Stream 消费者

从 Redis Stream 消费任务并处理
"""

import json
import asyncio
from typing import Callable, Optional, Dict, Any
from redis.asyncio import Redis

from app.config import get_settings

settings = get_settings()


class StreamConsumer:
    """Redis Stream 消费者"""

    def __init__(
        self,
        redis: Optional[Redis] = None,
        stream_key: str = "",
        consumer_group: str = "",
        consumer_name: str = "consumer-1",
    ):
        self._redis = redis
        self.stream_key = stream_key
        self.consumer_group = consumer_group
        self.consumer_name = consumer_name
        self.running = True

    async def get_redis(self) -> Redis:
        if self._redis is None:
            self._redis = Redis.from_url(settings.redis.REDIS_URL, decode_responses=True)
        return self._redis

    async def init_group(self) -> None:
        """初始化消费者组"""
        redis = await self.get_redis()
        try:
            await redis.xgroup_create(
                self.stream_key,
                self.consumer_group,
                id="0",
                mkstream=True,
            )
        except Exception:
            pass  # 组已存在

    async def consume_loop(self, processor: Callable) -> None:
        """消费循环"""
        await self.init_group()
        redis = await self.get_redis()

        while self.running:
            try:
                messages = await redis.xreadgroup(
                    self.consumer_group,
                    self.consumer_name,
                    {self.stream_key: ">"},
                    count=1,
                    block=5000,
                )

                if not messages:
                    continue

                for stream, msgs in messages:
                    for message_id, fields in msgs:
                        await self._process_message(
                            message_id, fields, processor
                        )

            except Exception as e:
                print(f"Consumer error: {e}")
                await asyncio.sleep(1)

    async def _process_message(
        self,
        message_id: str,
        fields: Dict[str, str],
        processor: Callable,
    ) -> None:
        """处理单条消息"""
        redis = await self.get_redis()

        try:
            payload = json.loads(fields["payload"])
            retry_count = int(fields.get("retry_count", 0))

            # 调用处理器
            await processor(payload)

            # 确认消息
            await redis.xack(self.stream_key, self.consumer_group, message_id)

        except Exception as e:
            print(f"Failed to process message {message_id}: {e}")

            retry_count = int(fields.get("retry_count", 0))
            if retry_count < 3:
                await self._retry_message(message_id, fields, retry_count)
            else:
                await redis.xack(self.stream_key, self.consumer_group, message_id)

    async def _retry_message(
        self,
        message_id: str,
        fields: Dict[str, str],
        retry_count: int,
    ) -> None:
        """重试消息"""
        redis = await self.get_redis()
        fields["retry_count"] = str(retry_count + 1)
        await redis.xadd(self.stream_key, fields)

    def stop(self) -> None:
        """停止消费"""
        self.running = False
```

---

## 4. Workers (app/tasks/workers.py)

```python
# app/tasks/workers.py

"""异步任务 Workers

每个任务处理时从 db_factory 获取新的 session，避免 session 过期问题
"""

import asyncio
from typing import Optional, Callable
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from app.tasks.producer import StreamProducer
from app.tasks.consumer import StreamConsumer
from app.services.resume_service import ResumeService
from app.services.interview_service import InterviewSessionService
from app.models.base import AsyncTaskStatus


class ResumeAnalyzeWorker:
    """简历分析 Worker"""

    def __init__(self, db_factory: async_sessionmaker, redis=None):
        """初始化 Worker

        Args:
            db_factory: 数据库 session 工厂，每次处理任务时从此工厂获取新 session
            redis: Redis 连接（可选）
        """
        self._db_factory = db_factory
        self.producer = StreamProducer(redis)
        self.consumer = StreamConsumer(
            redis=redis,
            stream_key="resume:analyze:stream",
            consumer_group="analyze-group",
            consumer_name="analyze-worker-1",
        )

    async def _get_db(self) -> AsyncSession:
        """获取新的数据库 session"""
        return self._db_factory()

    async def process(self, payload: dict) -> None:
        """处理简历分析任务

        每次处理时从 factory 获取新的 session，确保数据库连接有效
        """
        db = await self._get_db()
        resume_service = ResumeService(db)

        resume_id = payload["resume_id"]
        resume_text = payload["resume_text"]

        try:
            # 更新状态为处理中
            await resume_service.update_status(
                resume_id, AsyncTaskStatus.PROCESSING
            )

            # TODO: 调用 LangGraph Agent 进行分析
            # result = await resume_analyzer_agent.ainvoke(resume_text, payload["file_hash"])

            # 模拟分析结果
            result = {
                "overallScore": 85,
                "scoreDetail": {
                    "content": 88,
                    "structure": 82,
                    "skillMatch": 85,
                    "expression": 80,
                    "project": 90,
                },
                "summary": "简历分析完成",
                "strengths": ["技术栈全面", "项目经验丰富"],
                "suggestions": [],
            }

            # 保存结果
            await resume_service.save_analysis(resume_id, result)

            # 更新状态为完成
            await resume_service.update_status(
                resume_id, AsyncTaskStatus.COMPLETED
            )

        except Exception as e:
            await resume_service.update_status(
                resume_id, AsyncTaskStatus.FAILED, error=str(e)
            )
        finally:
            await db.close()

    async def run(self) -> None:
        """启动 Worker"""
        await self.consumer.consume_loop(self.process)

    def stop(self) -> None:
        """停止 Worker"""
        self.consumer.stop()


class EvaluateWorker:
    """面试评估 Worker"""

    def __init__(self, db_factory: async_sessionmaker, redis=None):
        """初始化 Worker

        Args:
            db_factory: 数据库 session 工厂，每次处理任务时从此工厂获取新 session
            redis: Redis 连接（可选）
        """
        self._db_factory = db_factory
        self.consumer = StreamConsumer(
            redis=redis,
            stream_key="interview:evaluate:stream",
            consumer_group="evaluate-group",
            consumer_name="evaluate-worker-1",
        )

    async def _get_db(self) -> AsyncSession:
        """获取新的数据库 session"""
        return self._db_factory()

    async def process(self, payload: dict) -> None:
        """处理面试评估任务

        每次处理时从 factory 获取新的 session，确保数据库连接有效
        """
        db = await self._get_db()
        session_service = InterviewSessionService(db)

        session_id = payload["session_id"]
        answers = payload["answers"]

        try:
            # TODO: 调用 LangGraph Agent 进行评估
            # evaluations = await evaluator_agent.ainvoke(answers)

            # 模拟评估结果
            evaluations = [
                {
                    "score": 85,
                    "feedback": "回答较好",
                    "referenceAnswer": "参考答案",
                    "keyPoints": []
                }
                for _ in answers
            ]

            # 保存评估结果
            await session_service.save_answer_evaluation(session_id, evaluations)

        except Exception as e:
            print(f"评估任务失败: {e}")
        finally:
            await db.close()

    async def run(self) -> None:
        """启动 Worker"""
        await self.consumer.consume_loop(self.process)

    def stop(self) -> None:
        """停止 Worker"""
        self.consumer.stop()
```

---

## 5. Worker 启动脚本 (app/tasks/run_workers.py)

```python
# app/tasks/run_workers.py

"""Worker 启动脚本"""

import asyncio
import signal
import os
from app.database import get_session_factory


async def main():
    """启动所有 Worker"""
    from app.tasks.workers import ResumeAnalyzeWorker, EvaluateWorker

    db_factory = get_session_factory()

    workers = [
        ResumeAnalyzeWorker(db_factory),
        EvaluateWorker(db_factory),
    ]

    def signal_handler(sig, frame):
        for w in workers:
            w.stop()

    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    print("Workers started, waiting for tasks...")

    await asyncio.gather(*[w.run() for w in workers])


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. 单元测试

```python
# tests/test_producer.py

import pytest
from unittest.mock import AsyncMock, patch, MagicMock

from app.tasks.producer import StreamProducer


class TestStreamProducer:
    """Stream 生产者测试"""

    @pytest.fixture
    def mock_redis(self):
        """Mock Redis"""
        redis = AsyncMock()
        redis.xadd = AsyncMock(return_value="msg-id")
        return redis

    @pytest.fixture
    def producer(self, mock_redis):
        """创建生产者"""
        return StreamProducer(redis=mock_redis)

    @pytest.mark.asyncio
    async def test_send_task(self, producer, mock_redis):
        """测试发送任务"""
        result = await producer.send_task(
            "resume_analyze",
            {"resume_id": 1, "text": "test"},
        )

        assert result is True
        mock_redis.xadd.assert_called_once()

    @pytest.mark.asyncio
    async def test_send_resume_analyze_task(self, producer, mock_redis):
        """测试发送简历分析任务"""
        result = await producer.send_resume_analyze_task(
            resume_id=1,
            resume_text="test resume",
            file_hash="abc123",
        )

        assert result is True

    @pytest.mark.asyncio
    async def test_unknown_task_type(self, producer, mock_redis):
        """测试未知任务类型"""
        with pytest.raises(ValueError):
            await producer.send_task("unknown", {})


# tests/test_consumer.py

import pytest
from unittest.mock import AsyncMock, MagicMock

from app.tasks.consumer import StreamConsumer


class TestStreamConsumer:
    """Stream 消费者测试"""

    @pytest.fixture
    def mock_redis(self):
        """Mock Redis"""
        redis = AsyncMock()
        redis.xgroup_create = AsyncMock()
        redis.xreadgroup = AsyncMock(return_value=[])
        redis.xack = AsyncMock()
        return redis

    @pytest.fixture
    def consumer(self, mock_redis):
        """创建消费者"""
        return StreamConsumer(
            redis=mock_redis,
            stream_key="test:stream",
            consumer_group="test-group",
            consumer_name="test-consumer",
        )

    @pytest.mark.asyncio
    async def test_init_group(self, consumer, mock_redis):
        """测试初始化消费者组"""
        await consumer.init_group()
        mock_redis.xgroup_create.assert_called_once()

    def test_stop(self, consumer):
        """测试停止消费"""
        consumer.stop()
        assert consumer.running is False
```

---

## 7. 验收标准

1. ✅ StreamProducer 正确发送任务
2. ✅ StreamConsumer 正确消费任务
3. ✅ 消息确认机制
4. ✅ 重试逻辑
5. ✅ Worker 正确处理任务
6. ✅ 单元测试覆盖

---

## 8. 后续任务

- [SDD-10: LangGraphAgents](./SDD-10-LangGraphAgents.md) - 依赖本任务的 Worker 处理逻辑
