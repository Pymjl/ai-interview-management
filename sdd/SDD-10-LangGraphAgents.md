# SDD-10: LangGraph Agents

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-04: AI服务集成](./SDD-04-AI服务集成.md)
> **目标**: 创建 LangGraph Agent 实现 AI 分析和问答

---

## 1. 概述

本任务负责创建 LangGraph Agents，包括：
- 简历分析 Agent
- 面试问题生成 Agent
- 答案评估 Agent
- RAG 问答 Agent

---

## 2. 简历分析 Agent (app/agents/resume_analyzer.py)

```python
# app/agents/resume_analyzer.py

"""简历分析 Agent

使用 LangGraph 实现简历分析工作流
"""

from typing import Optional, TypedDict
from langgraph.graph import StateGraph, END
from pydantic import BaseModel

from app.services.ai_provider import get_llm
from app.services.structured_output import ResumeAnalysisOutput
from app.agents.chains import create_resume_analysis_chain
from app.agents.prompts import load_prompt


class ResumeAnalyzerState(TypedDict):
    """简历分析状态"""
    resume_text: str
    file_hash: str
    overall_score: Optional[int]
    score_details: Optional[dict]
    summary: Optional[str]
    strengths: Optional[list]
    suggestions: Optional[list]
    error: Optional[str]


class ResumeAnalyzerAgent:
    """简历分析 Agent"""

    def __init__(self):
        self.llm = get_llm(temperature=0.1)
        self.chain = create_resume_analysis_chain(self.llm)
        self.graph = self._build_graph()

    def _build_graph(self) -> StateGraph:
        """构建状态机图"""
        workflow = StateGraph(ResumeAnalyzerState)

        workflow.add_node("parse_resume", self._parse_resume)
        workflow.add_node("ai_scoring", self._ai_scoring)
        workflow.add_node("generate_suggestions", self._generate_suggestions)
        workflow.add_node("finalize", self._finalize)

        workflow.set_entry_point("parse_resume")
        workflow.add_edge("parse_resume", "ai_scoring")
        workflow.add_edge("ai_scoring", "generate_suggestions")
        workflow.add_edge("generate_suggestions", "finalize")
        workflow.add_edge("finalize", END)

        return workflow.compile()

    def _parse_resume(self, state: ResumeAnalyzerState) -> dict:
        """解析简历文本"""
        from app.utils.text_cleaner import text_cleaner
        cleaned_text = text_cleaner.clean_resume_text(state["resume_text"])
        return {"resume_text": cleaned_text}

    async def _ai_scoring(self, state: ResumeAnalyzerState) -> dict:
        """AI 评分"""
        try:
            result = await self.chain.ainvoke({
                "resume_text": state["resume_text"]
            })

            return {
                "overall_score": result.get("overallScore"),
                "score_details": result.get("scoreDetail", {}),
                "summary": result.get("summary"),
            }
        except Exception as e:
            return {"error": str(e)}

    def _generate_suggestions(self, state: ResumeAnalyzerState) -> dict:
        """生成改进建议"""
        suggestions = []

        if state.get("score_details"):
            details = state["score_details"]
            if details.get("projectScore", 0) < 25:
                suggestions.append({
                    "category": "项目",
                    "priority": "高",
                    "issue": "项目深度不足",
                    "recommendation": "建议增加复杂问题排查或性能优化相关项目经历",
                })

        return {"suggestions": suggestions}

    def _finalize(self, state: ResumeAnalyzerState) -> dict:
        """最终处理"""
        return {
            "overall_score": state.get("overall_score"),
            "score_details": state.get("score_details"),
            "summary": state.get("summary"),
            "strengths": state.get("strengths", []),
            "suggestions": state.get("suggestions", []),
        }

    async def ainvoke(self, resume_text: str, file_hash: str) -> dict:
        """异步执行分析"""
        initial_state = ResumeAnalyzerState(
            resume_text=resume_text,
            file_hash=file_hash,
        )
        result = await self.graph.ainvoke(initial_state)
        return result
```

---

## 3. 面试问题生成 Agent (app/agents/interview_generator.py)

```python
# app/agents/interview_generator.py

"""面试问题生成 Agent"""

from typing import Optional, TypedDict, List
from langgraph.graph import StateGraph, END
from pydantic import BaseModel

from app.services.ai_provider import get_llm
from app.agents.chains import create_question_generation_chain


class InterviewQuestionState(TypedDict):
    """面试问题生成状态"""
    resume_text: str
    resume_id: int
    question_count: int
    existing_questions: List[str]
    questions: Optional[List[dict]]
    error: Optional[str]


class InterviewQuestionAgent:
    """面试问题生成 Agent"""

    QUESTION_TYPES = {
        "PROJECT": 0.20,
        "MYSQL": 0.20,
        "REDIS": 0.20,
        "JAVA_BASIC": 0.10,
        "JAVA_COLLECTION": 0.10,
        "JAVA_CONCURRENT": 0.10,
        "SPRING": 0.10,
    }

    def __init__(self):
        self.llm = get_llm(temperature=0.3)
        self.chain = create_question_generation_chain(self.llm)
        self.graph = self._build_graph()

    def _build_graph(self) -> StateGraph:
        workflow = StateGraph(InterviewQuestionState)

        workflow.add_node("calculate_distribution", self._calculate_distribution)
        workflow.add_node("generate_questions", self._generate_questions)
        workflow.add_node("finalize", self._finalize)

        workflow.set_entry_point("calculate_distribution")
        workflow.add_edge("calculate_distribution", "generate_questions")
        workflow.add_edge("generate_questions", "finalize")
        workflow.add_edge("finalize", END)

        return workflow.compile()

    def _calculate_distribution(self, state: InterviewQuestionState) -> dict:
        """计算问题分布"""
        distribution = {
            qtype: max(1, int(state["question_count"] * ratio))
            for qtype, ratio in self.QUESTION_TYPES.items()
        }
        return {"question_count": state["question_count"]}

    async def _generate_questions(self, state: InterviewQuestionState) -> dict:
        """生成面试问题"""
        try:
            result = await self.chain.ainvoke({
                "resume_text": state["resume_text"],
                "question_count": state["question_count"],
                "existing_questions": state.get("existing_questions", []),
            })

            questions = result.get("questions", [])
            return {"questions": questions}
        except Exception as e:
            return {"error": str(e)}

    def _finalize(self, state: InterviewQuestionState) -> dict:
        """最终处理"""
        return {"questions": state.get("questions", [])}

    async def ainvoke(self, resume_text: str, resume_id: int, question_count: int = 10) -> dict:
        """异步执行"""
        initial_state = InterviewQuestionState(
            resume_text=resume_text,
            resume_id=resume_id,
            question_count=question_count,
            existing_questions=[],
        )
        result = await self.graph.ainvoke(initial_state)
        return result
```

---

## 4. 答案评估 Agent (app/agents/answer_evaluator.py)

```python
# app/agents/answer_evaluator.py

"""答案评估 Agent"""

from typing import Optional, TypedDict, List
from langgraph.graph import StateGraph, END
from pydantic import BaseModel

from app.services.ai_provider import get_llm
from app.agents.chains import (
    create_answer_evaluation_chain,
    create_evaluation_summary_chain,
)


class AnswerEvaluatorState(TypedDict):
    """答案评估状态"""
    session_id: str
    answers: List[dict]
    batch_size: int
    evaluated_answers: List[dict]
    overall_score: Optional[int]
    overall_feedback: Optional[str]
    strengths: List[str]
    improvements: List[str]
    reference_answers: List[dict]
    error: Optional[str]


class AnswerEvaluatorAgent:
    """答案评估 Agent"""

    def __init__(self):
        self.llm = get_llm(temperature=0.2)
        self.eval_chain = create_answer_evaluation_chain(self.llm)
        self.summary_chain = create_evaluation_summary_chain(self.llm)
        self.graph = self._build_graph()

    def _build_graph(self) -> StateGraph:
        workflow = StateGraph(AnswerEvaluatorState)

        workflow.add_node("batch_evaluate", self._batch_evaluate)
        workflow.add_node("summarize", self._summarize_results)
        workflow.add_node("finalize", self._finalize)

        workflow.set_entry_point("batch_evaluate")
        workflow.add_edge("batch_evaluate", "summarize")
        workflow.add_edge("summarize", "finalize")
        workflow.add_edge("finalize", END)

        return workflow.compile()

    async def _batch_evaluate(self, state: AnswerEvaluatorState) -> dict:
        """批量评估答案"""
        evaluated = []
        batch_size = state.get("batch_size", 8)

        for i in range(0, len(state["answers"]), batch_size):
            batch = state["answers"][i:i + batch_size]

            answers_text = "\n\n".join([
                f"问题 {idx+1} ({a.get('category', 'N/A')}): {a.get('question', '')}\n答案: {a.get('answer', '')}"
                for idx, a in enumerate(batch)
            ])

            try:
                result = await self.eval_chain.ainvoke({"answers": answers_text})
                evaluated.extend(result if isinstance(result, list) else [result])
            except Exception:
                pass

        return {"evaluated_answers": evaluated}

    async def _summarize_results(self, state: AnswerEvaluatorState) -> dict:
        """汇总评估结果"""
        if not state["evaluated_answers"]:
            return {
                "overall_score": 0,
                "overall_feedback": "无法评估",
                "strengths": [],
                "improvements": [],
                "reference_answers": [],
            }

        avg_score = sum(
            e.get("score", 0) for e in state["evaluated_answers"]
        ) / len(state["evaluated_answers"])

        summary_input = {
            "total": len(state["evaluated_answers"]),
            "average_score": avg_score,
            "details": state["evaluated_answers"],
        }

        try:
            summary = await self.summary_chain.ainvoke(summary_input)

            return {
                "overall_score": summary.get("overallScore", int(avg_score)),
                "overall_feedback": summary.get("overallFeedback", ""),
                "strengths": summary.get("strengths", []),
                "improvements": summary.get("improvements", []),
                "reference_answers": summary.get("referenceAnswers", []),
            }
        except Exception as e:
            return {
                "overall_score": int(avg_score),
                "overall_feedback": "评估完成",
                "strengths": [],
                "improvements": [],
                "reference_answers": [],
            }

    def _finalize(self, state: AnswerEvaluatorState) -> dict:
        """最终处理"""
        return {
            "overall_score": state.get("overall_score"),
            "overall_feedback": state.get("overall_feedback"),
            "strengths": state.get("strengths", []),
            "improvements": state.get("improvements", []),
            "reference_answers": state.get("reference_answers", []),
        }

    async def ainvoke(self, session_id: str, answers: List[dict]) -> dict:
        """异步执行"""
        initial_state = AnswerEvaluatorState(
            session_id=session_id,
            answers=answers,
            batch_size=8,
            evaluated_answers=[],
            strengths=[],
            improvements=[],
            reference_answers=[],
        )
        result = await self.graph.ainvoke(initial_state)
        return result
```

---

## 5. RAG 问答 Agent (app/agents/rag_chat_agent.py)

```python
# app/agents/rag_chat_agent.py

"""RAG 问答 Agent"""

import asyncio
from typing import Optional, TypedDict, List, Any, AsyncIterator
from langgraph.graph import StateGraph, END
from langgraph.types import StreamMode
from langchain_core.callbacks import AsyncCallbackHandler

from app.services.ai_provider import get_llm
from app.agents.chains import create_rag_query_chain, create_query_rewrite_chain
from app.services.vector_service import VectorService


class RagChatState(TypedDict):
    """RAG 问答状态"""
    query: str
    chat_history: List[dict]
    kb_ids: List[int]
    user_id: int
    rewritten_query: Optional[str]
    top_k: int
    min_score: float
    retrieved_docs: List[dict]
    answer: Optional[str]


class StreamingCallback(AsyncCallbackHandler):
    """LangChain 流式回调处理器

    用于捕获 LLM 生成过程中的 token，实现真正的流式输出
    """

    def __init__(self, queue: asyncio.Queue):
        """初始化流式回调

        Args:
            queue: 异步队列，用于存储捕获的 token
        """
        self.queue = queue
        self._done = False

    async def on_llm_new_token(self, token: str, **kwargs) -> None:
        """LLM 生成新 token 时回调"""
        if token:  # 忽略空token
            await self.queue.put(token)

    async def on_llm_end(self, **kwargs) -> None:
        """LLM 生成结束时回调"""
        await self.queue.put(None)  # None 作为结束信号

    async def on_llm_error(self, error: Exception, **kwargs) -> None:
        """LLM 错误时回调"""
        await self.queue.put(None)
        raise error


class RagChatAgent:
    """RAG 问答 Agent"""

    def __init__(self, vector_service: VectorService):
        self.llm = get_llm(temperature=0.2)
        self.vector_service = vector_service
        self.query_chain = create_rag_query_chain(self.llm)
        self.rewrite_chain = create_query_rewrite_chain(self.llm)
        self.graph = self._build_graph()

    def _build_graph(self) -> StateGraph:
        workflow = StateGraph(RagChatState)

        workflow.add_node("rewrite_query", self._rewrite_query)
        workflow.add_node("retrieve", self._retrieve_docs)
        workflow.add_node("generate", self._generate_answer)

        workflow.set_entry_point("rewrite_query")
        workflow.add_edge("rewrite_query", "retrieve")
        workflow.add_edge("retrieve", "generate")
        workflow.add_edge("generate", END)

        return workflow.compile()

    def _rewrite_query(self, state: RagChatState) -> dict:
        """Query 改写"""
        query = state["query"]

        if len(query) <= 4:
            # 短查询需要改写
            try:
                result = self.rewrite_chain.invoke({"query": query})
                if isinstance(result, dict):
                    rewritten = result.get("rewrittenQuery", query)
                else:
                    rewritten = str(result)
            except Exception:
                rewritten = query

            return {
                "rewritten_query": rewritten,
                "top_k": 20,
                "min_score": 0.18,
            }

        return {
            "rewritten_query": query,
            "top_k": 8 if len(query) > 12 else 12,
            "min_score": 0.28,
        }

    async def _retrieve_docs(self, state: RagChatState) -> dict:
        """向量检索"""
        query = state.get("rewritten_query") or state["query"]

        docs = await self.vector_service.similarity_search(
            query=query,
            user_id=state["user_id"],
            kb_ids=state["kb_ids"],
            top_k=state["top_k"],
            min_score=state["min_score"],
        )

        return {"retrieved_docs": docs}

    async def _generate_answer(
        self,
        state: RagChatState,
        callbacks: Optional[List[AsyncCallbackHandler]] = None,
    ) -> dict:
        """生成答案

        Args:
            state: 当前状态
            callbacks: 回调处理器列表，用于流式输出
        """
        docs = state.get("retrieved_docs", [])

        context = "\n\n".join([
            f"【文档 {i+1}】\n{doc['content']}"
            for i, doc in enumerate(docs)
        ])

        history = "\n".join([
            f"{'用户' if msg['role'] == 'user' else '助手'}：{msg['content']}"
            for msg in state.get("chat_history", [])[-5:]
        ])

        try:
            # 如果有回调，使用带回调的异步流式调用
            if callbacks:
                answer = ""
                async for chunk in self.query_chain.astream(
                    {"history": history, "context": context, "query": state["query"]},
                    config={"callbacks": callbacks}
                ):
                    if chunk:
                        answer += chunk if isinstance(chunk, str) else str(chunk)
                return {"answer": answer}

            # 普通调用（非流式）
            result = await self.query_chain.ainvoke({
                "history": history,
                "context": context,
                "query": state["query"],
            })
            # 支持多种返回格式
            if isinstance(result, dict):
                answer = result.get("answer", str(result))
            else:
                answer = str(result)
        except Exception as e:
            answer = f"抱歉，生成答案时出错：{str(e)}"

        return {"answer": answer}

    async def ainvoke(self, query: str, user_id: int, kb_ids: List[int],
                      chat_history: List[dict] = None) -> dict:
        """异步执行"""
        initial_state = RagChatState(
            query=query,
            user_id=user_id,
            kb_ids=kb_ids,
            chat_history=chat_history or [],
        )
        result = await self.graph.ainvoke(initial_state)
        return result

    async def ainvoke_stream(
        self, query: str, user_id: int, kb_ids: List[int],
        chat_history: List[dict] = None,
    ) -> AsyncIterator[str]:
        """流式执行

        使用 LangChain 的 AsyncCallbackHandler 实现真正的流式输出

        Args:
            query: 查询文本
            user_id: 用户ID
            kb_ids: 知识库ID列表
            chat_history: 聊天历史

        Yields:
            生成的 token 字符串
        """
        import asyncio

        # 创建队列用于接收 token
        queue = asyncio.Queue()
        streaming_callback = StreamingCallback(queue)

        # 执行图，但只流式处理 "generate" 节点
        # 由于 LangGraph 的 ainvoke 不直接支持流式回调，
        # 我们使用 astream_events 或直接调用支持流的 LLM
        try:
            # 使用 astream_events 捕获 LLM 的流式事件
            async for event in self.graph.astream_events(
                {
                    "query": query,
                    "user_id": user_id,
                    "kb_ids": kb_ids,
                    "chat_history": chat_history or [],
                },
                version="v1",
                config={"callbacks": [streaming_callback]}
            ):
                # 事件类型: on_chat_model_start, on_chat_model_stream, on_chat_model_end
                if event["event"] == "on_chat_model_stream":
                    token = event["data"]["chunk"].content
                    if token:
                        yield token
        except Exception as e:
            # 如果 astream_events 不支持，回退到普通调用
            result = await self.ainvoke(
                query=query,
                user_id=user_id,
                kb_ids=kb_ids,
                chat_history=chat_history,
            )
            answer = result.get("answer", "")
            for char in answer:
                yield char
```

---

## 6. Agent 导出 (app/agents/__init__.py)

```python
# app/agents/__init__.py

"""LangGraph Agents"""

from app.agents.resume_analyzer import ResumeAnalyzerAgent
from app.agents.interview_generator import InterviewQuestionAgent
from app.agents.answer_evaluator import AnswerEvaluatorAgent
from app.agents.rag_chat_agent import RagChatAgent

__all__ = [
    "ResumeAnalyzerAgent",
    "InterviewQuestionAgent",
    "AnswerEvaluatorAgent",
    "RagChatAgent",
]
```

---

## 7. 单元测试

```python
# tests/test_agents/test_resume_analyzer.py

import pytest
from unittest.mock import patch, MagicMock, AsyncMock

from app.agents.resume_analyzer import ResumeAnalyzerAgent


class TestResumeAnalyzerAgent:
    """简历分析 Agent 测试"""

    @pytest.fixture
    def agent(self):
        """创建 Agent"""
        with patch('app.agents.resume_analyzer.get_llm') as mock_llm:
            mock_llm.return_value = MagicMock()
            return ResumeAnalyzerAgent()

    def test_build_graph(self, agent):
        """测试图构建"""
        assert agent.graph is not None

    def test_parse_resume(self, agent):
        """测试简历解析"""
        state = {
            "resume_text": "  Hello   World  ",
            "file_hash": "abc",
        }
        result = agent._parse_resume(state)

        assert "resume_text" in result
```

---

## 8. 验收标准

1. ✅ ResumeAnalyzerAgent 正确实现简历分析
2. ✅ InterviewQuestionAgent 正确生成面试问题
3. ✅ AnswerEvaluatorAgent 正确评估答案
4. ✅ RagChatAgent 正确实现 RAG 问答
5. ✅ 流式输出支持
6. ✅ 单元测试覆盖

---

## 9. 后续任务

- [SDD-11: 安全与限流](./SDD-11-安全与限流.md) - 独立的中间件功能
- [SDD-12: 依赖注入与主应用](./SDD-12-依赖注入与主应用.md) - 整合所有模块
