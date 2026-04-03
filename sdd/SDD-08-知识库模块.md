# SDD-08: 知识库模块

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](./SDD-02-数据库基础架构.md)
> - [SDD-03: 公共工具层](./SDD-03-公共工具层.md)
> - [SDD-04: AI服务集成](./SDD-04-AI服务集成.md)
> **目标**: 创建知识库上传、RAG检索、问答功能

---

## 1. 概述

知识库模块负责文档上传、向量化存储、RAG问答。

## 2. Schema (app/schemas/knowledgebase.py)

```python
# app/schemas/knowledgebase.py

from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime


class KnowledgeBaseUploadResponse(BaseModel):
    id: int
    name: str
    vector_status: str


class KnowledgeBaseItem(BaseModel):
    id: int
    name: str
    category: Optional[str]
    file_size: int
    uploaded_at: datetime
    vector_status: str
    chunk_count: Optional[int]

    model_config = {"from_attributes": True}


class KnowledgeBaseListResponse(BaseModel):
    items: List[KnowledgeBaseItem]
    total: int


class KnowledgeBaseDetailResponse(BaseModel):
    id: int
    name: str
    category: Optional[str]
    original_filename: str
    file_size: int
    uploaded_at: datetime
    vector_status: str

    model_config = {"from_attributes": True}


class UpdateCategoryRequest(BaseModel):
    category: str


class RagQueryRequest(BaseModel):
    query: str
    knowledge_base_ids: List[int]


class RagQueryResponse(BaseModel):
    answer: str
    sources: List[dict]
```

## 3. 向量服务 (app/services/vector_service.py)

```python
# app/services/vector_service.py

"""向量服务

处理文本分块、向量化、相似度搜索
"""

from typing import List, Optional, Tuple
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json

from app.models.knowledgebase import Document
from app.services.ai_provider import get_embeddings
from app.exceptions import BusinessException


class VectorService:
    """向量服务"""

    # 向量维度（根据 embedding 模型）
    VECTOR_DIM = 1024

    def __init__(self, db: AsyncSession):
        self.db = db

    async def chunk_text(self, text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
        """将文本分块

        Args:
            text: 原始文本
            chunk_size: 块大小（字符数）
            overlap: 重叠大小

        Returns:
            文本块列表
        """
        if not text:
            return []

        chunks = []
        start = 0

        while start < len(text):
            end = start + chunk_size
            chunk = text[start:end]
            chunks.append(chunk)
            start = end - overlap

        return chunks

    async def create_embeddings(self, texts: List[str]) -> List[List[float]]:
        """为文本列表创建 embeddings"""
        embedder = get_embeddings()
        embeddings = await embedder.aembed_documents(texts)
        return embeddings

    async def add_documents(
        self,
        user_id: int,
        kb_id: int,
        chunks: List[str],
        metadata_list: List[dict],
    ) -> int:
        """添加文档块到向量存储

        使用 pgvector 存储向量，支持高效的向量相似度搜索
        Returns:
            添加的块数量
        """
        from sqlalchemy import text
        from pgvector.psycopg2 import Vector

        embeddings = await self.create_embeddings(chunks)

        # 使用原生SQL插入向量（pgvector需要特殊处理）
        for i, (chunk, embedding, metadata) in enumerate(zip(chunks, embeddings, metadata_list)):
            doc_id = f"{kb_id}_{i}_{metadata.get('chunk_index', i)}"
            meta = json.dumps({**metadata, "kb_id": kb_id})

            # 使用 text() 执行原生SQL以正确处理 Vector 类型
            await self.db.execute(
                text("""
                    INSERT INTO document (id, user_id, content, metadata, embedding)
                    VALUES (:id, :user_id, :content, :metadata::jsonb, :embedding::vector)
                """),
                {
                    "id": doc_id,
                    "user_id": user_id,
                    "content": chunk,
                    "metadata": meta,
                    "embedding": embedding,  # list类型会被psycopg2转为vector
                }
            )

        await self.db.commit()
        return len(chunks)

    async def similarity_search(
        self,
        query: str,
        user_id: int,
        kb_ids: List[int],
        top_k: int = 8,
        min_score: float = 0.28,
    ) -> List[dict]:
        """向量相似度搜索（使用 pgvector）

        使用 pgvector 的余弦距离操作符 (<=>) 进行高效的向量相似度搜索

        Args:
            query: 查询文本
            user_id: 用户ID
            kb_ids: 知识库ID列表（为空则搜索全部）
            top_k: 返回数量
            min_score: 最低相似度分数（0-1，余弦相似度）

        Returns:
            相似文档列表，按相似度降序排列
        """
        from sqlalchemy import text

        # 创建查询 embedding
        embedder = get_embeddings()
        query_embedding = await embedder.aembed_query(query)

        # 构建 kb_ids 过滤条件
        kb_filter = ""
        if kb_ids:
            kb_filter = f"AND metadata->>'kb_id' = ANY(:kb_ids)"

        # 使用 pgvector 的余弦距离操作符 (<=>) 进行向量搜索
        # 距离越小相似度越高，因此用 ORDER BY distance ASC
        # 距离转换为相似度: similarity = 1 - distance
        sql = text(f"""
            SELECT
                id,
                content,
                metadata,
                (embedding <=> :query_embedding::vector) AS distance
            FROM document
            WHERE user_id = :user_id
            {kb_filter}
            ORDER BY embedding <=> :query_embedding::vector
            LIMIT :top_k
        """)

        # 转换 kb_ids 为整数数组
        kb_ids_param = kb_ids if kb_ids else None

        result = await self.db.execute(
            sql,
            {
                "query_embedding": query_embedding,
                "user_id": user_id,
                "kb_ids": kb_ids_param,
                "top_k": top_k * 2,  # 多取一些，后面过滤
            }
        )
        rows = result.fetchall()

        # 转换距离为相似度并过滤
        scored_docs = []
        for row in rows:
            distance = row.distance
            # pgvector 余弦距离范围 [0, 2]，转换为相似度 [1, -1]
            # similarity = 1 - distance (对于余弦距离)
            similarity = 1 - distance

            if similarity >= min_score:
                scored_docs.append({
                    "content": row.content,
                    "metadata": row.metadata if isinstance(row.metadata, dict) else json.loads(row.metadata),
                    "score": round(similarity, 4),
                })

            if len(scored_docs) >= top_k:
                break

        return scored_docs

    async def delete_by_kb_id(self, kb_id: int) -> int:
        """删除知识库关联的文档"""
        from sqlalchemy import text

        # 使用 jsonb 操作符删除关联文档
        result = await self.db.execute(
            text("""
                DELETE FROM document
                WHERE metadata->>'kb_id' = :kb_id
                RETURNING id
            """),
            {"kb_id": str(kb_id)}
        )
        deleted = result.fetchall()
        return len(deleted)
```

## 4. 知识库服务 (app/services/knowledgebase_service.py)

```python
# app/services/knowledgebase_service.py

"""知识库服务"""

from typing import Optional, List, Tuple
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from fastapi import UploadFile

from app.models.knowledgebase import KnowledgeBase
from app.models.base import AsyncTaskStatus
from app.schemas.knowledgebase import (
    KnowledgeBaseUploadResponse,
    KnowledgeBaseItem,
    KnowledgeBaseListResponse,
    UpdateCategoryRequest,
)
from app.services.file_service import FileService
from app.services.vector_service import VectorService
from app.services.minio_service import get_minio_service
from app.exceptions import DuplicateResourceException, ResourceNotFoundException


class KnowledgeBaseService:
    """知识库服务"""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.file_service = FileService()
        self.vector_service = VectorService(db)

    async def upload(
        self,
        file: UploadFile,
        user_id: int,
        name: str,
        category: Optional[str] = None,
    ) -> KnowledgeBaseUploadResponse:
        """上传知识库文件"""
        # 1. 读取并验证文件
        file_content = self.file_service.validate_file(file)

        # 2. 计算哈希
        file_hash = self.file_service.calculate_hash(file_content)

        # 3. 检查重复
        existing = await self._get_by_hash(file_hash)
        if existing:
            raise DuplicateResourceException("知识库文件")

        # 4. 解析文档
        content = await self.file_service.parse_document(file_content, file.content_type)

        # 5. 上传到 MinIO
        storage_key = f"knowledgebase/{user_id}/{file_hash}.{file.filename.split('.')[-1]}"
        minio_service = get_minio_service()
        storage_url = minio_service.upload_file(file_content, storage_key, file.content_type)

        # 6. 保存到数据库
        kb = KnowledgeBase(
            user_id=user_id,
            file_hash=file_hash,
            name=name,
            category=category,
            original_filename=file.filename,
            file_size=len(file_content),
            content_type=file.content_type,
            storage_key=storage_key,
            storage_url=storage_url,
            vector_status=AsyncTaskStatus.PENDING,
        )
        self.db.add(kb)
        await self.db.commit()
        await self.db.refresh(kb)

        # 7. 触发向量化（后续异步任务）
        await self._trigger_vectorize(kb.id, content, user_id)

        return KnowledgeBaseUploadResponse(
            id=kb.id,
            name=kb.name,
            vector_status=kb.vector_status,
        )

    async def _trigger_vectorize(self, kb_id: int, content: str, user_id: int) -> None:
        """触发向量化"""
        # 分块
        chunks = await self.vector_service.chunk_text(content)

        # 创建元数据
        metadata_list = [{"chunk_index": i} for i in range(len(chunks))]

        # 添加到向量存储
        chunk_count = await self.vector_service.add_documents(
            user_id=user_id,
            kb_id=kb_id,
            chunks=chunks,
            metadata_list=metadata_list,
        )

        # 更新状态
        kb = await self._get_by_id(kb_id)
        if kb:
            kb.chunk_count = chunk_count
            kb.vector_status = AsyncTaskStatus.COMPLETED.value
            await self.db.commit()

    async def get_list(
        self,
        user_id: int,
        page: int = 1,
        page_size: int = 10,
        category: Optional[str] = None,
    ) -> Tuple[List[KnowledgeBaseItem], int]:
        """获取知识库列表"""
        query = select(KnowledgeBase).where(KnowledgeBase.user_id == user_id)

        if category:
            query = query.where(KnowledgeBase.category == category)

        # 总数
        count_query = select(func.count()).select_from(query.subquery())
        total_result = await self.db.execute(count_query)
        total = total_result.scalar()

        # 分页
        query = query.order_by(KnowledgeBase.uploaded_at.desc())
        query = query.offset((page - 1) * page_size).limit(page_size)

        result = await self.db.execute(query)
        items = result.scalars().all()

        return [KnowledgeBaseItem.model_validate(kb) for kb in items], total

    async def update_category(
        self,
        kb_id: int,
        user_id: int,
        request: UpdateCategoryRequest,
    ) -> KnowledgeBase:
        """更新分类"""
        kb = await self._get_by_id_and_user(kb_id, user_id)
        kb.category = request.category
        await self.db.commit()
        return kb

    async def delete(self, kb_id: int, user_id: int) -> None:
        """删除知识库"""
        kb = await self._get_by_id_and_user(kb_id, user_id)

        # 删除向量文档
        await self.vector_service.delete_by_kb_id(kb_id)

        # 删除 MinIO 文件
        minio_service = get_minio_service()
        minio_service.delete_file(kb.storage_key)

        # 删除数据库记录
        await self.db.delete(kb)
        await self.db.commit()

    async def get_stats(self, user_id: int) -> dict:
        """获取统计信息"""
        result = await self.db.execute(
            select(KnowledgeBase).where(KnowledgeBase.user_id == user_id)
        )
        items = result.scalars().all()

        total = len(items)
        completed = sum(1 for kb in items if kb.vector_status == AsyncTaskStatus.COMPLETED.value)
        pending = sum(1 for kb in items if kb.vector_status == AsyncTaskStatus.PENDING.value)

        categories = list(set(kb.category for kb in items if kb.category))

        return {
            "total": total,
            "completed": completed,
            "pending": pending,
            "categories": categories,
        }

    async def _get_by_id(self, kb_id: int) -> Optional[KnowledgeBase]:
        result = await self.db.execute(select(KnowledgeBase).where(KnowledgeBase.id == kb_id))
        return result.scalar_one_or_none()

    async def _get_by_hash(self, file_hash: str) -> Optional[KnowledgeBase]:
        result = await self.db.execute(
            select(KnowledgeBase).where(KnowledgeBase.file_hash == file_hash)
        )
        return result.scalar_one_or_none()

    async def _get_by_id_and_user(self, kb_id: int, user_id: int) -> KnowledgeBase:
        result = await self.db.execute(
            select(KnowledgeBase).where(
                KnowledgeBase.id == kb_id,
                KnowledgeBase.user_id == user_id,
            )
        )
        kb = result.scalar_one_or_none()
        if not kb:
            raise ResourceNotFoundException("知识库")
        return kb
```

## 5. API路由 (app/api/knowledgebase.py)

```python
# app/api/knowledgebase.py

"""知识库 API 路由"""

from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, Form, Query
from typing import Optional

from app.schemas.knowledgebase import (
    KnowledgeBaseUploadResponse,
    KnowledgeBaseListResponse,
    UpdateCategoryRequest,
)
from app.services.knowledgebase_service import KnowledgeBaseService
from app.database import get_db_session
from app.models.user import User
from app.deps import get_current_active_user

router = APIRouter(prefix="/api/knowledgebase", tags=["knowledgebase"])


@router.post("/upload", response_model=KnowledgeBaseUploadResponse)
async def upload_knowledge_base(
    file: UploadFile = File(...),
    name: str = Form(...),
    category: Optional[str] = Form(None),
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """上传知识库文件"""
    service = KnowledgeBaseService(db)

    try:
        return await service.upload(file, current_user.id, name, category)
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/list", response_model=KnowledgeBaseListResponse)
async def list_knowledge_bases(
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    category: Optional[str] = Query(None),
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """获取知识库列表"""
    service = KnowledgeBaseService(db)
    items, total = await service.get_list(current_user.id, page, page_size, category)

    return KnowledgeBaseListResponse(items=items, total=total)


@router.put("/{kb_id}/category")
async def update_category(
    kb_id: int,
    request: UpdateCategoryRequest,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """更新分类"""
    service = KnowledgeBaseService(db)

    try:
        await service.update_category(kb_id, current_user.id, request)
    except Exception as e:
        raise HTTPException(status_code=404, detail=str(e))

    return {"message": "更新成功"}


@router.delete("/{kb_id}")
async def delete_knowledge_base(
    kb_id: int,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """删除知识库"""
    service = KnowledgeBaseService(db)

    try:
        await service.delete(kb_id, current_user.id)
    except Exception as e:
        raise HTTPException(status_code=404, detail=str(e))

    return {"message": "删除成功"}


@router.get("/stats")
async def get_stats(
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """获取统计信息"""
    service = KnowledgeBaseService(db)
    return await service.get_stats(current_user.id)
```

## 6. 验收标准

1. ✅ 知识库文件上传
2. ✅ 文档分块和向量化
3. ✅ 知识库列表查询
4. ✅ 分类管理
5. ✅ 删除功能（文件+向量）
6. ✅ 用户数据隔离

## 7. 后续任务

- [SDD-09: 异步任务处理](./SDD-09-异步任务处理.md) - 依赖本模块的向量化任务
- [SDD-10: LangGraphAgents](./SDD-10-LangGraphAgents.md) - 依赖本模块的 RAG 问答
