# SDD-04: AI服务集成

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**: [SDD-01: 项目基础配置](SDD-01-项目基础配置.md), [SDD-03: 公共工具层](SDD-03-公共工具层.md)
> **目标**: 创建 AI 服务提供商封装、LangChain/LangGraph 基础组件

---

## 1. 概述

本任务负责创建 AI 服务集成层，包括：
- AI Provider 封装（支持 OpenAI 兼容 API）
- Embedding 服务封装
- LangChain Chains 工厂函数
- LangGraph 状态机基础组件
- 结构化输出处理

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── ai_provider.py      # 新增
│   │   └── structured_output.py # 新增
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── chains.py           # 新增
│   │   └── prompts/            # 新增
│   │       ├── __init__.py
│   │       └── ...             # 提示词模板
│   └── models/
│       └── ...
└── ...
```

---

## 3. AI Provider (app/services/ai_provider.py)

```python
# app/services/ai_provider.py

"""AI 服务提供商封装

支持任意 OpenAI 兼容 API（Ollama、vLLM、LocalAI、OpenAI 等）
"""

from typing import Optional, Any, Callable
from functools import lru_cache
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.callbacks import AsyncCallbackHandler

from app.config import get_settings

settings = get_settings()


class LLMConfig:
    """LLM 配置"""

    def __init__(
        self,
        model: Optional[str] = None,
        base_url: Optional[str] = None,
        api_key: Optional[str] = None,
        temperature: float = 0.2,
        streaming: bool = True,
        max_retries: int = 3,
        request_timeout: int = 120,
    ):
        self.model = model or settings.llm.MODEL
        self.base_url = base_url or settings.llm.BASE_URL
        self.api_key = api_key or settings.llm.API_KEY
        self.temperature = temperature
        self.streaming = streaming
        self.max_retries = max_retries
        self.request_timeout = request_timeout

    def to_dict(self) -> dict:
        """转换为字典"""
        return {
            "model": self.model,
            "base_url": self.base_url,
            "api_key": self.api_key,
            "temperature": self.temperature,
            "streaming": self.streaming,
            "max_retries": self.max_retries,
            "request_timeout": self.request_timeout,
        }


class EmbeddingConfig:
    """Embedding 配置"""

    def __init__(
        self,
        model: Optional[str] = None,
        base_url: Optional[str] = None,
        api_key: Optional[str] = None,
        batch_size: int = 100,
    ):
        self.model = model or settings.embedding.MODEL
        self.base_url = base_url or settings.embedding.BASE_URL
        self.api_key = api_key or settings.embedding.API_KEY
        self.batch_size = batch_size

    def to_dict(self) -> dict:
        """转换为字典"""
        return {
            "model": self.model,
            "base_url": self.base_url,
            "api_key": self.api_key,
            "batch_size": self.batch_size,
        }


def get_llm(config: Optional[LLMConfig] = None) -> ChatOpenAI:
    """获取 LLM 实例

    Args:
        config: LLM 配置，默认使用环境变量配置

    Returns:
        ChatOpenAI 实例
    """
    if config is None:
        config = LLMConfig()

    return ChatOpenAI(
        model=config.model,
        base_url=config.base_url,
        api_key=config.api_key,
        temperature=config.temperature,
        streaming=config.streaming,
        max_retries=config.max_retries,
        request_timeout=config.request_timeout,
    )


def get_embeddings(config: Optional[EmbeddingConfig] = None) -> OpenAIEmbeddings:
    """获取 Embeddings 实例

    Args:
        config: Embedding 配置，默认使用环境变量配置

    Returns:
        OpenAIEmbeddings 实例
    """
    if config is None:
        config = EmbeddingConfig()

    return OpenAIEmbeddings(
        model=config.model,
        embedding = base_url=config.base_url,
        api_key=config.api_key,
    )


def create_llm(
    model: Optional[str] = None,
    temperature: float = 0.2,
    streaming: bool = True,
) -> ChatOpenAI:
    """快速创建 LLM 实例的便捷函数"""
    config = LLMConfig(
        model=model,
        temperature=temperature,
        streaming=streaming,
    )
    return get_llm(config)


def create_embeddings(
    model: Optional[str] = None,
) -> OpenAIEmbeddings:
    """快速创建 Embeddings 实例的便捷函数"""
    config = EmbeddingConfig(model=model)
    return get_embeddings(config)


# 向量存储接口（用于类型提示）
class VectorStoreInterface:
    """向量存储接口"""

    async def aadd_texts(self, texts: list[str], metadatas: list[dict]) -> list[str]:
        """添加文本到向量存储"""
        raise NotImplementedError

    async def a similarity_search_with_score(
        self, query: str, k: int = 4, filter: Optional[dict] = None
    ) -> list[tuple[Any, float]]:
        """相似度搜索"""
        raise NotImplementedError

    async def aget_relevant_documents(
        self, query: str, k: int = 4, filter: Optional[dict] = None
    ) -> list[Any]:
        """获取相关文档"""
        raise NotImplementedError
```

---

## 4. 结构化输出 (app/services/structured_output.py)

```python
# app/services/structured_output.py

"""结构化输出处理

提供结构化的 AI 输出解析能力
"""

from pydantic import BaseModel, Field
from typing import TypeVar, Generic, Type, Optional, Any
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.runnables import Runnable
import json
import logging

logger = logging.getLogger(__name__)

T = TypeVar("T", bound=BaseModel)


class OutputValidationError(Exception):
    """输出验证错误"""

    def __init__(self, message: str, raw_output: str, cause: Optional[Exception] = None):
        self.message = message
        self.raw_output = raw_output
        self.cause = cause
        super().__init__(self.message)


class StructuredOutputInvoker:
    """结构化输出调用器

    提供带重试的 AI 结构化输出调用
    """

    def __init__(
        self,
        max_retries: int = 3,
        validation_enabled: bool = True,
    ):
        """
        Args:
            max_retries: 最大重试次数
            validation_enabled: 是否启用验证
        """
        self.max_retries = max_retries
        self.validation_enabled = validation_enabled

    async def invoke(
        self,
        chain: Runnable,
        output_class: Type[T],
        input_data: dict,
    ) -> T:
        """调用链并返回结构化对象

        Args:
            chain: LangChain Runnable 链
            output_class: Pydantic 输出类
            input_data: 输入数据

        Returns:
            结构化的输出对象
        """
        output_parser = JsonOutputParser(pydantic_object=output_class)
        chain_with_parser = chain | output_parser

        last_error: Optional[Exception] = None

        for attempt in range(self.max_retries):
            try:
                result = await chain_with_parser.ainvoke(input_data)

                if self.validation_enabled:
                    return output_class.model_validate(result)

                return result

            except Exception as e:
                last_error = e
                logger.warning(
                    f"Attempt {attempt + 1}/{self.max_retries} failed: {e}"
                )

                if attempt < self.max_retries - 1:
                    continue

        raise OutputValidationError(
            message=f"Failed after {self.max_retries} attempts: {last_error}",
            raw_output="",
            cause=last_error,
        )

    def invoke_sync(
        self,
        chain: Runnable,
        output_class: Type[T],
        input_data: dict,
    ) -> T:
        """同步调用链并返回结构化对象"""
        output_parser = JsonOutputParser(pydantic_object=output_class)
        chain_with_parser = chain | output_parser

        last_error: Optional[Exception] = None

        for attempt in range(self.max_retries):
            try:
                result = chain_with_parser.invoke(input_data)

                if self.validation_enabled:
                    return output_class.model_validate(result)

                return result

            except Exception as e:
                last_error = e
                logger.warning(
                    f"Attempt {attempt + 1}/{self.max_retries} failed: {e}"
                )

                if attempt < self.max_retries - 1:
                    continue

        raise OutputValidationError(
            message=f"Failed after {self.max_retries} attempts: {last_error}",
            raw_output="",
            cause=last_error,
        )


# 常用输出模型

class ResumeAnalysisOutput(BaseModel):
    """简历分析输出"""

    overallScore: int = Field(description="总分 (0-100)")
    scoreDetail: dict = Field(description="各维度得分")
    summary: str = Field(description="简历摘要")
    strengths: list[str] = Field(description="简历亮点")
    suggestions: list[dict] = Field(description="改进建议")


class QuestionListOutput(BaseModel):
    """问题列表输出"""

    questions: list[dict] = Field(description="面试问题列表")


class AnswerEvaluationOutput(BaseModel):
    """答案评估输出"""

    score: int = Field(description="得分 (0-100)")
    feedback: str = Field(description="反馈意见")
    keyPoints: list[str] = Field(description="关键要点")
    referenceAnswer: str = Field(description="参考答案")


class EvaluationSummaryOutput(BaseModel):
    """评估汇总输出"""

    overallScore: int = Field(description="总体得分")
    overallFeedback: str = Field(description="总体反馈")
    strengths: list[str] = Field(description="优点")
    improvements: list[str] = Field(description="改进点")
    referenceAnswers: list[dict] = Field(description="参考答案列表")


class QueryRewriteOutput(BaseModel):
    """Query改写输出"""

    rewrittenQuery: str = Field(description="改写后的查询")
    alternativeQueries: list[str] = Field(description="备选查询")


class RagAnswerOutput(BaseModel):
    """RAG答案输出"""

    answer: str = Field(description="生成的答案")
    sources: list[dict] = Field(description="参考来源")
```

---

## 5. Chains 工厂 (app/agents/chains.py)

```python
# app/agents/chains.py

"""LangChain Chains 工厂函数

提供各种 AI 任务的 Chain 创建函数
"""

from typing import Optional, Callable
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser, StrOutputParser
from langchain_core.runnables import Runnable
from functools import partial

from app.services.ai_provider import get_llm, create_llm
from app.agents.prompts import (
    load_prompt,
    RESUME_ANALYSIS_PROMPT,
    INTERVIEW_QUESTION_PROMPT,
    INTERVIEW_EVALUATION_PROMPT,
    INTERVIEW_EVALUATION_SUMMARY_PROMPT,
    KNOWLEDGEBASE_QUERY_PROMPT,
    KNOWLEDGEBASE_QUERY_REWRITE_PROMPT,
)


def create_resume_analysis_chain(
    llm: Optional[any] = None,
) -> Runnable:
    """创建简历分析 Chain

    Args:
        llm: 语言模型实例，默认创建新实例

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=0.1)

    prompt = ChatPromptTemplate.from_messages([
        ("system", RESUME_ANALYSIS_PROMPT),
        ("human", "请分析以下简历：\n\n{resume_text}")
    ])

    output_parser = JsonOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_question_generation_chain(
    llm: Optional[any] = None,
    temperature: float = 0.3,
) -> Runnable:
    """创建面试问题生成 Chain

    Args:
        llm: 语言模型实例
        temperature: 温度参数

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    prompt = ChatPromptTemplate.from_messages([
        ("system", INTERVIEW_QUESTION_PROMPT),
        ("human", """基于以下简历生成面试问题：
简历内容：{resume_text}
问题数量：{question_count}
历史问题（需去重）：{existing_questions}""")
    ])

    output_parser = JsonOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_answer_evaluation_chain(
    llm: Optional[any] = None,
    temperature: float = 0.2,
) -> Runnable:
    """创建答案评估 Chain（批量）

    Args:
        llm: 语言模型实例
        temperature: 温度参数

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    prompt = ChatPromptTemplate.from_messages([
        ("system", INTERVIEW_EVALUATION_PROMPT),
        ("human", "请评估以下面试答案：\n\n{answers}")
    ])

    output_parser = JsonOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_evaluation_summary_chain(
    llm: Optional[any] = None,
    temperature: float = 0.2,
) -> Runnable:
    """创建评估汇总 Chain

    Args:
        llm: 语言模型实例
        temperature: 温度参数

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    prompt = ChatPromptTemplate.from_messages([
        ("system", INTERVIEW_EVALUATION_SUMMARY_PROMPT),
        ("human", """基于以下评估结果生成总体评价：
{evaluation_results}""")
    ])

    output_parser = JsonOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_rag_query_chain(
    llm: Optional[any] = None,
    temperature: float = 0.2,
) -> Runnable:
    """创建 RAG 查询 Chain

    Args:
        llm: 语言模型实例
        temperature: 温度参数

    Returns:
        LangChain Runnable（输出字符串）
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    prompt = ChatPromptTemplate.from_messages([
        ("system", KNOWLEDGEBASE_QUERY_PROMPT),
        ("human", """基于以下知识库内容回答用户问题。

历史对话：
{history}

知识库内容：
{context}

用户问题：{query}""")
    ])

    output_parser = StrOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_query_rewrite_chain(
    llm: Optional[any] = None,
    temperature: float = 0.5,
) -> Runnable:
    """创建 Query 改写 Chain

    Args:
        llm: 语言模型实例
        temperature: 温度参数

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    prompt = ChatPromptTemplate.from_messages([
        ("system", KNOWLEDGEBASE_QUERY_REWRITE_PROMPT),
        ("human", "原始查询：{query}")
    ])

    output_parser = JsonOutputParser()

    chain = prompt | llm | output_parser

    return chain


def create_chat_chain(
    llm: Optional[any] = None,
    system_prompt: Optional[str] = None,
    temperature: float = 0.7,
) -> Runnable:
    """创建通用聊天 Chain

    Args:
        llm: 语言模型实例
        system_prompt: 系统提示词
        temperature: 温度参数

    Returns:
        LangChain Runnable
    """
    if llm is None:
        llm = get_llm(temperature=temperature)

    if system_prompt:
        prompt = ChatPromptTemplate.from_messages([
            ("system", system_prompt),
            ("human", "{message}")
        ])
    else:
        prompt = ChatPromptTemplate.from_messages([
            ("human", "{message}")
        ])

    output_parser = StrOutputParser()

    chain = prompt | llm | output_parser

    return chain
```

---

## 6. 提示词模板 (app/agents/prompts/__init__.py)

```python
# app/agents/prompts/__init__.py

"""提示词模板

包含所有 AI 任务的系统提示词
"""

# ============ 简历分析提示词 ============

RESUME_ANALYSIS_PROMPT = """# Role
你是一位拥有 10 年以上经验的资深技术架构师、工程管理专家及高级技术人才顾问。你具备跨语言（Java, Go, Python, Rust, Frontend, Infrastructure 等）的深度技术视野，擅长从底层架构、工程效率和业务价值三个维度对简历进行"穿透式"审计。

# Task
请对用户提供的简历内容进行深度技术审计、多维度评分，并提供极具实操性的改进建议。

# Output Format
请直接输出一个 JSON 对象...

JSON 结构必须严格包含以下字段：
1. overallScore: 整数，总分（0-100）
2. scoreDetail: 对象，包含各维度得分...
3. summary: 字符串，简历摘要
4. strengths: 字符串数组，简历亮点
5. suggestions: 对象数组，改进建议

# 注意
- 必须输出有效的 JSON 格式
- 不要输出任何其他内容
"""


# ============ 面试问题生成提示词 ============

INTERVIEW_QUESTION_PROMPT = """# Role
你是一位拥有10年面试经验的技术面试官，精通各类技术主题和面试技巧。

# Task
根据给定的简历内容，生成针对性的面试问题。

# Question Types
- PROJECT: 项目经验问题 (20%)
- MYSQL: 数据库问题 (20%)
- REDIS: 缓存问题 (20%)
- JAVA_BASIC: Java基础 (10%)
- JAVA_COLLECTION: 集合框架 (10%)
- JAVA_CONCURRENT: 并发编程 (10%)
- SPRING: Spring框架 (10%)

# Output Format
请直接输出一个 JSON 对象，包含 questions 数组...
"""


# ============ 面试答案评估提示词 ============

INTERVIEW_EVALUATION_PROMPT = """# Role
你是一位资深技术面试官，擅长评估候选人的技术能力和项目经验。

# Task
评估候选人给出的面试答案，给出分数和改进建议。

# Scoring Rubric
- 深度 (40%): 答案的技术深度
- 广度 (30%): 答案的全面程度
- 表达 (20%): 表达的清晰程度
- 项目关联 (10%): 与简历项目的相关性

# Output Format
请直接输出一个 JSON 对象...
"""


# ============ 评估汇总提示词 ============

INTERVIEW_EVALUATION_SUMMARY_PROMPT = """# Role
你是一位技术总监，擅长综合评估候选人的整体表现。

# Task
根据候选人在多道题目上的表现，生成综合评价和改进建议。

# Output Format
请直接输出一个 JSON 对象，包含：
- overallScore: 总体得分
- overallFeedback: 总体反馈
- strengths: 优点列表
- improvements: 改进点列表
- referenceAnswers: 参考答案列表
"""


# ============ 知识库查询提示词 ============

KNOWLEDGEBASE_QUERY_PROMPT = """# Role
你是一个知识库问答助手，擅长根据提供的文档内容回答用户问题。

# Instructions
- 根据提供的知识库内容，准确回答用户问题
- 如果知识库中没有相关信息，请明确指出
- 保持回答简洁、有条理
- 引用相关文档来源

# Output Format
直接输出回答内容，不需要特殊格式
"""


# ============ Query 改写提示词 ============

KNOWLEDGEBASE_QUERY_REWRITE_PROMPT = """# Role
你是一个查询优化专家，擅长将短查询改写为更有效的搜索查询。

# Task
当用户输入过短的查询时，将其改写为更详细、更有效的查询。

# Rules
- 保持原查询的核心意图
- 扩展相关术语
- 补充上下文信息
- 输出 JSON 格式

# Output Format
请直接输出一个 JSON 对象：
{
    "rewrittenQuery": "改写后的查询",
    "alternativeQueries": ["备选查询1", "备选查询2"]
}
"""


def load_prompt(prompt_name: str) -> str:
    """加载提示词模板

    Args:
        prompt_name: 提示词标识

    Returns:
        提示词内容
    """
    prompts = {
        "resume-analysis": RESUME_ANALYSIS_PROMPT,
        "interview-question": INTERVIEW_QUESTION_PROMPT,
        "interview-evaluation": INTERVIEW_EVALUATION_PROMPT,
        "interview-evaluation-summary": INTERVIEW_EVALUATION_SUMMARY_PROMPT,
        "knowledgebase-query": KNOWLEDGEBASE_QUERY_PROMPT,
        "knowledgebase-query-rewrite": KNOWLEDGEBASE_QUERY_REWRITE_PROMPT,
    }

    if prompt_name not in prompts:
        raise ValueError(f"Unknown prompt: {prompt_name}")

    return prompts[prompt_name]
```

---

## 7. LangGraph 状态机基础 (app/agents/graph_base.py)

```python
# app/agents/graph_base.py

"""LangGraph 状态机基础组件"""

from typing import TypedDict, Optional, Any
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from abc import ABC, abstractmethod


class BaseState(TypedDict, ABC):
    """状态机基类"""

    error: Optional[str]


class BaseAgent(ABC):
    """Agent 基类

    所有 Agent 应继承此类
    """

    def __init__(self):
        self.graph: Optional[StateGraph] = None
        self.checkpointer = MemorySaver()

    @abstractmethod
    def _build_graph(self) -> StateGraph:
        """构建状态机图"""
        pass

    def get_graph(self) -> StateGraph:
        """获取状态机图"""
        if self.graph is None:
            self.graph = self._build_graph()
        return self.graph

    def compile(self) -> Any:
        """编译状态机"""
        return self.get_graph().compile(checkpointer=self.checkpointer)

    async def ainvoke(self, initial_state: dict) -> dict:
        """异步执行"""
        compiled = self.compile()
        return await compiled.ainvoke(initial_state)

    def invoke(self, initial_state: dict) -> dict:
        """同步执行"""
        compiled = self.compile()
        return compiled.invoke(initial_state)


def create_basic_graph(
    state_class: type[BaseState],
    nodes: dict[str, callable],
    initial_node: str,
) -> StateGraph:
    """创建基础状态机

    Args:
        state_class: 状态类
        nodes: 节点字典 {node_name: node_function}
        initial_node: 初始节点名称

    Returns:
        编译后的状态机
    """
    workflow = StateGraph(state_class)

    # 添加节点
    for node_name, node_func in nodes.items():
        workflow.add_node(node_name, node_func)

    # 设置入口点和边
    workflow.set_entry_point(initial_node)

    # 添加从初始节点到终点的边（简单版本）
    for node_name in nodes.keys():
        if node_name != initial_node:
            workflow.add_edge(node_name, END)

    return workflow.compile(checkpointer=MemorySaver())


class StreamingCallbackHandler:
    """流式回调处理器"""

    def __init__(self):
        self.tokens: list[str] = []

    async def on_llm_new_token(self, token: str, **kwargs):
        """处理新 token"""
        self.tokens.append(token)

    def get_full_output(self) -> str:
        """获取完整输出"""
        return "".join(self.tokens)

    def clear(self):
        """清空 tokens"""
        self.tokens.clear()
```

---

## 8. 服务导出 (app/services/__init__.py)

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
from app.services.ai_provider import (
    LLMConfig,
    EmbeddingConfig,
    get_llm,
    get_embeddings,
    create_llm,
    create_embeddings,
)
from app.services.structured_output import (
    StructuredOutputInvoker,
    OutputValidationError,
    ResumeAnalysisOutput,
    QuestionListOutput,
    AnswerEvaluationOutput,
    EvaluationSummaryOutput,
    QueryRewriteOutput,
    RagAnswerOutput,
)

__all__ = [
    # MinIO
    "MiniOService",
    "get_minio_service",
    # Redis
    "RedisClient",
    "get_redis_client",
    "close_redis",
    # AI Provider
    "LLMConfig",
    "EmbeddingConfig",
    "get_llm",
    "get_embeddings",
    "create_llm",
    "create_embeddings",
    # Structured Output
    "StructuredOutputInvoker",
    "OutputValidationError",
    "ResumeAnalysisOutput",
    "QuestionListOutput",
    "AnswerEvaluationOutput",
    "EvaluationSummaryOutput",
    "QueryRewriteOutput",
    "RagAnswerOutput",
]
```

---

## 9. 单元测试

```python
# tests/test_ai_provider.py

import pytest
from unittest.mock import patch, MagicMock

from app.services.ai_provider import (
    LLMConfig,
    EmbeddingConfig,
    get_llm,
    get_embeddings,
    create_llm,
    create_embeddings,
)


class TestLLMConfig:
    """LLM配置测试"""

    def test_default_values(self):
        """测试默认值"""
        config = LLMConfig()
        assert config.model is not None
        assert config.temperature == 0.2
        assert config.streaming is True

    def test_custom_values(self):
        """测试自定义值"""
        config = LLMConfig(
            model="gpt-4",
            temperature=0.5,
            streaming=False,
        )
        assert config.model == "gpt-4"
        assert config.temperature == 0.5
        assert config.streaming is False

    def test_to_dict(self):
        """测试转换为字典"""
        config = LLMConfig(model="test-model")
        d = config.to_dict()
        assert isinstance(d, dict)
        assert d["model"] == "test-model"


class TestEmbeddingConfig:
    """Embedding配置测试"""

    def test_default_values(self):
        """测试默认值"""
        config = EmbeddingConfig()
        assert config.model is not None
        assert config.batch_size == 100


class TestLLMCreation:
    """LLM创建测试"""

    @pytest.fixture
    def mock_settings(self):
        """Mock配置"""
        with patch('app.services.ai_provider.settings') as mock:
            mock.llm.MODEL = "test-model"
            mock.llm.BASE_URL = "http://test:11434/v1"
            mock.llm.API_KEY = "test-key"
            yield mock

    def test_get_llm(self, mock_settings):
        """测试获取LLM实例"""
        llm = get_llm()
        assert llm is not None

    def test_get_llm_with_config(self, mock_settings):
        """测试使用配置获取LLM"""
        config = LLMConfig(model="custom-model")
        llm = get_llm(config)
        assert llm is not None

    def test_create_llm(self, mock_settings):
        """测试便捷创建函数"""
        llm = create_llm(temperature=0.5)
        assert llm is not None
```

```python
# tests/test_structured_output.py

import pytest
from pydantic import BaseModel, Field
from unittest.mock import MagicMock, AsyncMock

from app.services.structured_output import (
    StructuredOutputInvoker,
    OutputValidationError,
    ResumeAnalysisOutput,
)


class MockChain:
    """Mock的LangChain Chain"""

    def __init__(self, result):
        self.result = result

    def __or__(self, other):
        return self

    async def ainvoke(self, input_data):
        return self.result


class TestStructuredOutputInvoker:
    """结构化输出调用器测试"""

    @pytest.fixture
    def invoker(self):
        """创建调用器"""
        return StructuredOutputInvoker(max_retries=3)

    @pytest.mark.asyncio
    async def test_invoke_success(self, invoker):
        """测试成功调用"""
        mock_result = {
            "overallScore": 85,
            "scoreDetail": {"content": 90, "structure": 80},
            "summary": "Test summary",
            "strengths": ["Good experience"],
            "suggestions": [],
        }

        chain = MagicMock()
        chain.ainvoke = AsyncMock(return_value=mock_result)

        result = await invoker.invoke(
            chain=chain,
            output_class=ResumeAnalysisOutput,
            input_data={"resume_text": "test"},
        )

        assert isinstance(result, ResumeAnalysisOutput)
        assert result.overallScore == 85
        assert result.summary == "Test summary"

    @pytest.mark.asyncio
    async def test_invoke_with_retry(self, invoker):
        """测试重试逻辑"""
        mock_result = {
            "overallScore": 85,
            "scoreDetail": {},
            "summary": "Test",
            "strengths": [],
            "suggestions": [],
        }

        chain = MagicMock()
        call_count = 0

        async def mock_invoke(input_data):
            nonlocal call_count
            call_count += 1
            return mock_result

        chain.ainvoke = mock_invoke

        result = await invoker.invoke(
            chain=chain,
            output_class=ResumeAnalysisOutput,
            input_data={},
        )

        assert call_count == 1  # 第一次就成功
        assert isinstance(result, ResumeAnalysisOutput)


class TestOutputModels:
    """输出模型测试"""

    def test_resume_analysis_output(self):
        """测试简历分析输出模型"""
        data = {
            "overallScore": 85,
            "scoreDetail": {"content": 90},
            "summary": "Test summary",
            "strengths": ["Good"],
            "suggestions": [{"issue": "Test issue"}],
        }

        output = ResumeAnalysisOutput(**data)

        assert output.overallScore == 85
        assert output.summary == "Test summary"
        assert len(output.strengths) == 1
```

```python
# tests/test_chains.py

import pytest
from unittest.mock import patch, MagicMock

from app.agents.chains import (
    create_resume_analysis_chain,
    create_question_generation_chain,
    create_rag_query_chain,
    create_chat_chain,
)


class TestChainCreation:
    """Chain创建测试"""

    @pytest.fixture
    def mock_llm(self):
        """Mock LLM"""
        with patch('app.agents.chains.get_llm') as mock_get_llm:
            mock_llm = MagicMock()
            mock_get_llm.return_value = mock_llm
            yield mock_llm

    def test_create_resume_analysis_chain(self, mock_llm):
        """测试创建简历分析链"""
        chain = create_resume_analysis_chain()
        assert chain is not None

    def test_create_question_generation_chain(self, mock_llm):
        """测试创建问题生成链"""
        chain = create_question_generation_chain()
        assert chain is not None

    def test_create_rag_query_chain(self, mock_llm):
        """测试创建RAG查询链"""
        chain = create_rag_query_chain()
        assert chain is not None

    def test_create_chat_chain(self, mock_llm):
        """测试创建聊天链"""
        chain = create_chat_chain()
        assert chain is not None

    def test_create_chat_chain_with_system_prompt(self, mock_llm):
        """测试带系统提示的聊天链"""
        chain = create_chat_chain(system_prompt="You are a helpful assistant.")
        assert chain is not None
```

---

## 10. 验收标准

1. ✅ AI Provider 支持配置不同的模型和端点
2. ✅ Embedding 服务正确封装
3. ✅ 所有 Chain 工厂函数正确实现
4. ✅ 提示词模板正确导出
5. ✅ 结构化输出支持 Pydantic 模型验证
6. ✅ LangGraph 基础组件可用
7. ✅ 单元测试覆盖核心功能
8. ✅ 支持 OpenAI 兼容 API 配置

---

## 11. 后续任务

- [SDD-05: 用户模块](SDD-05-用户模块.md) - 依赖本任务的配置
- [SDD-06: 简历模块](SDD-06-简历模块.md) - 依赖本任务的 Chain 和 Agent
- [SDD-07: 面试模块](SDD-07-面试模块.md) - 依赖本任务的 Chain 和 Agent
- [SDD-08: 知识库模块](SDD-08-知识库模块.md) - 依赖本任务的 Chain 和 Agent
- [SDD-10: LangGraphAgents](SDD-10-LangGraphAgents.md) - 依赖本任务的基础组件
