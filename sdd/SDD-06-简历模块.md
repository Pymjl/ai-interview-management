# SDD-06: 简历模块

> **版本**: v1.0
> **日期**: 2026-04-04
> **前置依赖**:
> - [SDD-02: 数据库基础架构](SDD-02-数据库基础架构.md)
> - [SDD-03: 公共工具层](SDD-03-公共工具层.md)
> - [SDD-04: AI服务集成](SDD-04-AI服务集成.md)
> **目标**: 创建简历上传、解析、AI评分、PDF导出功能

---

## 1. 概述

本任务负责创建简历模块，包括：
- 简历 Schema 定义
- 文件处理服务 (FileService)
- 简历服务 (ResumeService)
- PDF 生成服务 (PdfService)
- API 路由

---

## 2. 文件结构

```
interview_guide/
├── app/
│   ├── schemas/
│   │   ├── resume.py          # 新增
│   ├── services/
│   │   ├── resume_service.py  # 新增
│   │   ├── file_service.py    # 新增
│   │   └── pdf_service.py     # 新增
│   ├── api/
│   │   ├── resume.py          # 新增
│   └── models/
│       ├── resume.py          # 来自 SDD-02
└── ...
```

---

## 3. Schema (app/schemas/resume.py)

```python
# app/schemas/resume.py

"""简历相关 Schema"""

from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime


class ResumeUploadResponse(BaseModel):
    """简历上传响应"""
    id: int
    filename: str
    analyze_status: str


class ResumeListItem(BaseModel):
    """简历列表项"""
    id: int
    original_filename: str
    file_size: int
    uploaded_at: datetime
    analyze_status: str
    overall_score: Optional[int] = None

    model_config = {"from_attributes": True}


class ResumeListResponse(BaseModel):
    """简历列表响应"""
    items: List[ResumeListItem]
    total: int
    page: int
    page_size: int


class ResumeDetailResponse(BaseModel):
    """简历详情响应"""
    id: int
    original_filename: str
    file_size: int
    content_type: str
    uploaded_at: datetime
    analyze_status: str
    resume_text: Optional[str] = None
    analysis: Optional["ResumeAnalysisResponse"] = None

    model_config = {"from_attributes": True}


class ScoreDetail(BaseModel):
    """评分详情"""
    content_score: int
    structure_score: int
    skill_match_score: int
    expression_score: int
    project_score: int


class ResumeAnalysisResponse(BaseModel):
    """简历分析结果"""
    id: int
    overall_score: int
    content_score: int
    structure_score: int
    skill_match_score: int
    expression_score: int
    project_score: int
    summary: str
    strengths: List[str]
    suggestions: List[dict]
    analyzed_at: datetime

    model_config = {"from_attributes": True}


class ResumeHealthResponse(BaseModel):
    """健康检查响应"""
    status: str
    version: str
```

---

## 4. 文件服务 (app/services/file_service.py)

```python
# app/services/file_service.py

"""文件处理服务

处理简历文件的上传、解析、存储
"""

import hashlib
from typing import Optional
import io
from fastapi import UploadFile

from app.services.minio_service import get_minio_service, MiniOService
from app.exceptions import FileProcessingException


class FileService:
    """文件处理服务"""

    # 支持的文件类型
    SUPPORTED_TYPES = {
        "application/pdf": "parse_pdf",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document": "parse_docx",
        "text/plain": "parse_txt",
        "text/markdown": "parse_txt",
    }

    # 最大文件大小 (10MB)
    MAX_FILE_SIZE = 10 * 1024 * 1024

    def __init__(self, minio_service: Optional[MiniOService] = None):
        self.minio_service = minio_service or get_minio_service()

    def calculate_hash(self, file_content: bytes) -> str:
        """计算 SHA-256 哈希"""
        return hashlib.sha256(file_content).hexdigest()

    async def parse_document(self, file_content: bytes, content_type: str) -> str:
        """解析文档为纯文本

        Args:
            file_content: 文件内容
            content_type: MIME类型

        Returns:
            提取的文本内容
        """
        parser_method = self.SUPPORTED_TYPES.get(content_type)

        if not parser_method:
            raise FileProcessingException(
                f"不支持的文件类型: {content_type}",
                code="UNSUPPORTED_FILE_TYPE"
            )

        parser = getattr(self, parser_method)
        return parser(file_content)

    def parse_pdf(self, file_content: bytes) -> str:
        """解析 PDF"""
        import pypdf
        from io import BytesIO

        try:
            reader = pypdf.PdfReader(BytesIO(file_content))
            text_parts = []
            for page in reader.pages:
                text = page.extract_text()
                if text:
                    text_parts.append(text)
            return "\n".join(text_parts)
        except Exception as e:
            raise FileProcessingException(f"PDF解析失败: {str(e)}")

    def parse_docx(self, file_content: bytes) -> str:
        """解析 DOCX"""
        from docx import Document
        from io import BytesIO

        try:
            doc = Document(BytesIO(file_content))
            return "\n".join([para.text for para in doc.paragraphs])
        except Exception as e:
            raise FileProcessingException(f"DOCX解析失败: {str(e)}")

    def parse_txt(self, file_content: bytes) -> str:
        """解析 TXT"""
        return file_content.decode("utf-8", errors="ignore")

    def validate_file(self, file: UploadFile) -> bytes:
        """验证并读取文件

        Args:
            file: 上传的文件

        Returns:
            文件内容
        """
        # 检查文件类型
        if file.content_type not in self.SUPPORTED_TYPES:
            raise FileProcessingException(
                f"不支持的文件类型: {file.content_type}",
                code="UNSUPPORTED_FILE_TYPE"
            )

        # 读取文件内容
        import asyncio
        content = asyncio.get_event_loop().run_until_complete(file.read())

        # 检查文件大小
        if len(content) > self.MAX_FILE_SIZE:
            raise FileProcessingException(
                f"文件大小超过限制: {len(content)} > {self.MAX_FILE_SIZE}",
                code="FILE_TOO_LARGE"
            )

        return content

    def generate_storage_key(self, user_id: int, filename: str, file_hash: str) -> str:
        """生成存储路径

        格式: resumes/{user_id}/{file_hash}.{ext}
        """
        ext = filename.split(".")[-1] if "." in filename else "bin"
        return f"resumes/{user_id}/{file_hash}.{ext}"
```

---

## 5. 简历服务 (app/services/resume_service.py)

```python
# app/services/resume_service.py

"""简历服务

处理简历的上传、分析、查询等业务逻辑
"""

from typing import Optional, Tuple
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from fastapi import UploadFile

from app.models.resume import Resume, ResumeAnalysis
from app.models.base import AsyncTaskStatus
from app.schemas.resume import (
    ResumeUploadResponse,
    ResumeListItem,
    ResumeDetailResponse,
    ResumeAnalysisResponse,
)
from app.services.file_service import FileService
from app.services.minio_service import get_minio_service
from app.exceptions import (
    BusinessException,
    DuplicateResourceException,
    ResourceNotFoundException,
)


class ResumeService:
    """简历服务"""

    def __init__(
        self,
        db: AsyncSession,
        file_service: Optional[FileService] = None,
    ):
        self.db = db
        self.file_service = file_service or FileService()

    async def upload_resume(
        self,
        file: UploadFile,
        user_id: int,
    ) -> ResumeUploadResponse:
        """上传并解析简历

        Args:
            file: 上传的文件
            user_id: 用户ID

        Returns:
            上传响应
        """
        # 1. 验证并读取文件
        file_content = self.file_service.validate_file(file)

        # 2. 计算文件哈希
        file_hash = self.file_service.calculate_hash(file_content)

        # 3. 检查是否重复上传
        existing = await self._get_by_hash(file_hash)
        if existing:
            raise DuplicateResourceException("简历")

        # 4. 解析文档
        resume_text = await self.file_service.parse_document(
            file_content, file.content_type
        )

        # 5. 生成存储路径
        storage_key = self.file_service.generate_storage_key(
            user_id, file.filename, file_hash
        )

        # 6. 上传到 MinIO
        minio_service = get_minio_service()
        storage_url = minio_service.upload_file(
            file_content,
            storage_key,
            file.content_type,
        )

        # 7. 保存到数据库
        resume = Resume(
            user_id=user_id,
            file_hash=file_hash,
            original_filename=file.filename,
            file_size=len(file_content),
            content_type=file.content_type,
            storage_key=storage_key,
            storage_url=storage_url,
            resume_text=resume_text,
            uploaded_at=datetime.utcnow(),
            analyze_status=AsyncTaskStatus.PENDING,
        )
        self.db.add(resume)
        await self.db.commit()
        await self.db.refresh(resume)

        # 8. 发送异步分析任务（后续SDD实现）

        return ResumeUploadResponse(
            id=resume.id,
            filename=resume.original_filename,
            analyze_status=resume.analyze_status,
        )

    async def get_user_resumes(
        self,
        user_id: int,
        page: int = 1,
        page_size: int = 10,
    ) -> Tuple[list[ResumeListItem], int]:
        """获取用户简历列表

        Args:
            user_id: 用户ID
            page: 页码
            page_size: 每页数量

        Returns:
            (简历列表, 总数)
        """
        # 查询总数
        count_query = select(func.count(Resume.id)).where(
            Resume.user_id == user_id
        )
        total_result = await self.db.execute(count_query)
        total = total_result.scalar()

        # 查询列表
        query = (
            select(Resume)
            .where(Resume.user_id == user_id)
            .order_by(Resume.uploaded_at.desc())
            .offset((page - 1) * page_size)
            .limit(page_size)
        )
        result = await self.db.execute(query)
        resumes = result.scalars().all()

        # 构建响应
        items = []
        for resume in resumes:
            score = None
            if resume.analysis:
                score = resume.analysis.overall_score

            items.append(ResumeListItem(
                id=resume.id,
                original_filename=resume.original_filename,
                file_size=resume.file_size,
                uploaded_at=resume.uploaded_at,
                analyze_status=resume.analyze_status,
                overall_score=score,
            ))

        return items, total

    async def get_resume_detail(
        self,
        resume_id: int,
        user_id: int,
    ) -> ResumeDetailResponse:
        """获取简历详情

        Args:
            resume_id: 简历ID
            user_id: 用户ID（用于权限校验）

        Returns:
            简历详情
        """
        resume = await self._get_by_id_and_user(resume_id, user_id)

        # 增加访问计数
        resume.access_count += 1
        resume.last_accessed_at = datetime.utcnow()
        await self.db.commit()

        analysis_resp = None
        if resume.analysis:
            analysis_resp = ResumeAnalysisResponse(
                id=resume.analysis.id,
                overall_score=resume.analysis.overall_score,
                content_score=resume.analysis.content_score,
                structure_score=resume.analysis.structure_score,
                skill_match_score=resume.analysis.skill_match_score,
                expression_score=resume.analysis.expression_score,
                project_score=resume.analysis.project_score,
                summary=resume.analysis.summary,
                strengths=[],  # 从 JSON 解析
                suggestions=[],  # 从 JSON 解析
                analyzed_at=resume.analysis.analyzed_at,
            )

        return ResumeDetailResponse(
            id=resume.id,
            original_filename=resume.original_filename,
            file_size=resume.file_size,
            content_type=resume.content_type,
            uploaded_at=resume.uploaded_at,
            analyze_status=resume.analyze_status,
            resume_text=resume.resume_text,
            analysis=analysis_resp,
        )

    async def delete_resume(self, resume_id: int, user_id: int) -> None:
        """删除简历

        Args:
            resume_id: 简历ID
            user_id: 用户ID
        """
        resume = await self._get_by_id_and_user(resume_id, user_id)

        # 从 MinIO 删除文件
        minio_service = get_minio_service()
        minio_service.delete_file(resume.storage_key)

        # 从数据库删除
        await self.db.delete(resume)
        await self.db.commit()

    async def reanalyze_resume(self, resume_id: int, user_id: int) -> None:
        """重新分析简历

        Args:
            resume_id: 简历ID
            user_id: 用户ID
        """
        resume = await self._get_by_id_and_user(resume_id, user_id)

        # 重置状态
        resume.analyze_status = AsyncTaskStatus.PENDING
        resume.analyze_error = None
        await self.db.commit()

        # 发送异步分析任务（后续SDD实现）

    async def update_status(
        self,
        resume_id: int,
        status: str,
        error: Optional[str] = None,
    ) -> None:
        """更新分析状态"""
        resume = await self._get_by_id(resume_id)
        if resume:
            resume.analyze_status = status
            if error:
                resume.analyze_error = error
            await self.db.commit()

    async def save_analysis(self, resume_id: int, analysis_data: dict) -> None:
        """保存分析结果"""
        resume = await self._get_by_id(resume_id)
        if not resume:
            return

        # 创建或更新分析结果
        if resume.analysis:
            analysis = resume.analysis
        else:
            analysis = ResumeAnalysis(resume_id=resume_id)
            self.db.add(analysis)

        analysis.overall_score = analysis_data.get("overallScore")
        analysis.content_score = analysis_data.get("scoreDetail", {}).get("content")
        analysis.structure_score = analysis_data.get("scoreDetail", {}).get("structure")
        analysis.skill_match_score = analysis_data.get("scoreDetail", {}).get("skillMatch")
        analysis.expression_score = analysis_data.get("scoreDetail", {}).get("expression")
        analysis.project_score = analysis_data.get("scoreDetail", {}).get("project")
        analysis.summary = analysis_data.get("summary")
        analysis.strengths_json = str(analysis_data.get("strengths", []))
        analysis.suggestions_json = str(analysis_data.get("suggestions", []))
        analysis.analyzed_at = datetime.utcnow()

        resume.analyze_status = AsyncTaskStatus.COMPLETED
        await self.db.commit()

    async def _get_by_id(self, resume_id: int) -> Optional[Resume]:
        """根据ID获取简历"""
        result = await self.db.execute(
            select(Resume).where(Resume.id == resume_id)
        )
        return result.scalar_one_or_none()

    async def _get_by_hash(self, file_hash: str) -> Optional[Resume]:
        """根据哈希获取简历"""
        result = await self.db.execute(
            select(Resume).where(Resume.file_hash == file_hash)
        )
        return result.scalar_one_or_none()

    async def _get_by_id_and_user(
        self,
        resume_id: int,
        user_id: int,
    ) -> Resume:
        """根据ID和用户获取简历"""
        result = await self.db.execute(
            select(Resume).where(
                Resume.id == resume_id,
                Resume.user_id == user_id
            )
        )
        resume = result.scalar_one_or_none()
        if not resume:
            raise ResourceNotFoundException("简历")
        return resume
```

---

## 6. PDF服务 (app/services/pdf_service.py)

```python
# app/services/pdf_service.py

"""PDF 生成服务

生成简历分析报告 PDF
"""

import io
from typing import Optional
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import cm
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Table, TableStyle
from reportlab.lib import colors

from app.models.resume import Resume, ResumeAnalysis


class PdfService:
    """PDF 服务"""

    def __init__(self):
        self.styles = getSampleStyleSheet()

    def generate_resume_report(
        self,
        resume: Resume,
        analysis: Optional[ResumeAnalysis] = None,
    ) -> bytes:
        """生成简历分析报告 PDF

        Args:
            resume: 简历对象
            analysis: 分析结果

        Returns:
            PDF 字节数据
        """
        buffer = io.BytesIO()
        doc = SimpleDocTemplate(
            buffer,
            pagesize=A4,
            rightMargin=2*cm,
            leftMargin=2*cm,
            topMargin=2*cm,
            bottomMargin=2*cm,
        )

        story = []

        # 标题
        title_style = ParagraphStyle(
            'CustomTitle',
            parent=self.styles['Heading1'],
            fontSize=24,
            spaceAfter=30,
        )
        story.append(Paragraph("简历分析报告", title_style))
        story.append(Spacer(1, 0.5*cm))

        # 基本信息
        story.append(Paragraph("基本信息", self.styles['Heading2']))
        info_data = [
            ["文件名", resume.original_filename],
            ["上传时间", resume.uploaded_at.strftime("%Y-%m-%d %H:%M:%S")],
            ["文件大小", f"{resume.file_size / 1024:.1f} KB"],
        ]
        info_table = Table(info_data, colWidths=[5*cm, 10*cm])
        info_table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (0, -1), colors.lightgrey),
            ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
            ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
            ('LEFTPADDING', (0, 0), (-1, -1), 8),
        ]))
        story.append(info_table)
        story.append(Spacer(1, 0.5*cm))

        # 分析结果
        if analysis:
            story.append(Paragraph("分析结果", self.styles['Heading2']))

            # 总分
            story.append(Paragraph(
                f"<b>总分: {analysis.overall_score}/100</b>",
                self.styles['Heading3']
            ))
            story.append(Spacer(1, 0.3*cm))

            # 各项得分
            scores_data = [
                ["维度", "得分"],
                ["内容完整度", f"{analysis.content_score}/100"],
                ["结构规范性", f"{analysis.structure_score}/100"],
                ["技能匹配度", f"{analysis.skill_match_score}/100"],
                ["表达清晰度", f"{analysis.expression_score}/100"],
                ["项目深度", f"{analysis.project_score}/100"],
            ]
            scores_table = Table(scores_data, colWidths=[5*cm, 3*cm])
            scores_table.setStyle(TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
                ('GRID', (0, 0), (-1, -1), 0.5, colors.black),
                ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
                ('ALIGN', (1, 0), (1, -1), 'CENTER'),
            ]))
            story.append(scores_table)
            story.append(Spacer(1, 0.5*cm))

            # 摘要
            if analysis.summary:
                story.append(Paragraph("简历摘要", self.styles['Heading3']))
                story.append(Paragraph(analysis.summary, self.styles['Normal']))
                story.append(Spacer(1, 0.3*cm))

            # 优点
            if analysis.strengths_json:
                story.append(Paragraph("简历亮点", self.styles['Heading3']))
                strengths = eval(analysis.strengths_json) if isinstance(analysis.strengths_json, str) else []
                for strength in strengths:
                    story.append(Paragraph(f"• {strength}", self.styles['Normal']))
                story.append(Spacer(1, 0.3*cm))

            # 建议
            if analysis.suggestions_json:
                story.append(Paragraph("改进建议", self.styles['Heading3']))
                suggestions = eval(analysis.suggestions_json) if isinstance(analysis.suggestions_json, str) else []
                for suggestion in suggestions:
                    if isinstance(suggestion, dict):
                        story.append(Paragraph(
                            f"• [{suggestion.get('category', '')}] {suggestion.get('recommendation', '')}",
                            self.styles['Normal']
                        ))
                story.append(Spacer(1, 0.3*cm))

        # 构建 PDF
        doc.build(story)
        buffer.seek(0)
        return buffer.read()
```

---

## 7. API路由 (app/api/resume.py)

```python
# app/api/resume.py

"""简历 API 路由"""

from fastapi import APIRouter, Depends, UploadFile, File, HTTPException, status, Query
from typing import Optional

from app.schemas.resume import (
    ResumeUploadResponse,
    ResumeListResponse,
    ResumeDetailResponse,
    ResumeHealthResponse,
)
from app.services.resume_service import ResumeService
from app.services.pdf_service import PdfService
from app.database import get_db_session
from app.models.user import User
from app.deps import get_current_active_user
from app.exceptions import DuplicateResourceException, FileProcessingException

router = APIRouter(prefix="/api/resumes", tags=["resume"])


@router.post("/upload", response_model=ResumeUploadResponse)
async def upload_resume(
    file: UploadFile = File(...),
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """上传简历"""
    resume_service = ResumeService(db)

    try:
        result = await resume_service.upload_resume(file, current_user.id)
    except DuplicateResourceException:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="该简历已上传"
        )
    except FileProcessingException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=e.message
        )

    return result


@router.get("/", response_model=ResumeListResponse)
async def list_resumes(
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=10, ge=1, le=100),
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """获取简历列表"""
    resume_service = ResumeService(db)

    items, total = await resume_service.get_user_resumes(
        current_user.id, page, page_size
    )

    return ResumeListResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
    )


@router.get("/{resume_id}/detail", response_model=ResumeDetailResponse)
async def get_resume_detail(
    resume_id: int,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """获取简历详情"""
    resume_service = ResumeService(db)

    try:
        result = await resume_service.get_resume_detail(resume_id, current_user.id)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e)
        )

    return result


@router.delete("/{resume_id}")
async def delete_resume(
    resume_id: int,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """删除简历"""
    resume_service = ResumeService(db)

    try:
        await resume_service.delete_resume(resume_id, current_user.id)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e)
        )

    return {"message": "删除成功"}


@router.post("/{resume_id}/reanalyze")
async def reanalyze_resume(
    resume_id: int,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """重新分析简历"""
    resume_service = ResumeService(db)

    try:
        await resume_service.reanalyze_resume(resume_id, current_user.id)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e)
        )

    return {"message": "已触发重新分析"}


@router.get("/{resume_id}/export")
async def export_resume_pdf(
    resume_id: int,
    current_user: User = Depends(get_current_active_user),
    db=Depends(get_db_session),
):
    """导出简历分析报告 PDF"""
    from fastapi.responses import StreamingResponse

    resume_service = ResumeService(db)
    pdf_service = PdfService()

    try:
        detail = await resume_service.get_resume_detail(resume_id, current_user.id)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e)
        )

    # 获取完整的 resume 对象
    from sqlalchemy import select
    from app.models.resume import Resume

    result = await db.execute(select(Resume).where(Resume.id == resume_id))
    resume = result.scalar_one()

    analysis = None
    if resume.analysis:
        analysis = resume.analysis

    pdf_bytes = pdf_service.generate_resume_report(resume, analysis)

    return StreamingResponse(
        iter([pdf_bytes]),
        media_type="application/pdf",
        headers={
            "Content-Disposition": f"attachment; filename=resume_{resume_id}_report.pdf"
        }
    )


@router.get("/health", response_model=ResumeHealthResponse)
async def health_check():
    """健康检查"""
    return ResumeHealthResponse(status="healthy", version="1.0.0")
```

---

## 8. 单元测试

```python
# tests/test_file_service.py

import pytest
from unittest.mock import MagicMock, patch, AsyncMock
from fastapi import UploadFile

from app.services.file_service import FileService
from app.exceptions import FileProcessingException


class TestFileService:
    """文件服务测试"""

    @pytest.fixture
    def file_service(self):
        """创建文件服务"""
        with patch('app.services.file_service.get_minio_service'):
            return FileService()

    def test_calculate_hash(self, file_service):
        """测试哈希计算"""
        content = b"test content"
        hash1 = file_service.calculate_hash(content)
        hash2 = file_service.calculate_hash(content)

        assert hash1 == hash2
        assert len(hash1) == 64  # SHA-256 hex length

    def test_calculate_hash_different_content(self, file_service):
        """测试不同内容哈希不同"""
        hash1 = file_service.calculate_hash(b"content1")
        hash2 = file_service.calculate_hash(b"content2")

        assert hash1 != hash2

    def test_parse_txt(self, file_service):
        """测试解析 TXT"""
        content = b"Hello World"
        result = file_service.parse_txt(content)

        assert result == "Hello World"

    def test_parse_txt_with_encoding(self, file_service):
        """测试解析带编码的 TXT"""
        content = "你好世界".encode("utf-8")
        result = file_service.parse_txt(content)

        assert result == "你好世界"

    def test_generate_storage_key(self, file_service):
        """测试生成存储路径"""
        key = file_service.generate_storage_key(1, "resume.pdf", "abc123")

        assert key == "resumes/1/abc123.pdf"

    def test_generate_storage_key_no_ext(self, file_service):
        """测试无扩展名文件"""
        key = file_service.generate_storage_key(1, "resume", "abc123")

        assert key == "resumes/1/abc123.bin"
```

```python
# tests/test_pdf_service.py

import pytest
from unittest.mock import MagicMock
from datetime import datetime

from app.services.pdf_service import PdfService


class TestPdfService:
    """PDF服务测试"""

    @pytest.fixture
    def pdf_service(self):
        """创建PDF服务"""
        return PdfService()

    def test_generate_resume_report_without_analysis(self, pdf_service):
        """测试生成无分析结果的报告"""
        mock_resume = MagicMock()
        mock_resume.original_filename = "test.pdf"
        mock_resume.uploaded_at = datetime(2024, 1, 1, 12, 0, 0)
        mock_resume.file_size = 1024

        result = pdf_service.generate_resume_report(mock_resume, None)

        assert result is not None
        assert len(result) > 0
        # PDF 文件以 %PDF 开头
        assert result.startswith(b'%PDF')

    def test_generate_resume_report_with_analysis(self, pdf_service):
        """测试生成带分析结果的报告"""
        mock_resume = MagicMock()
        mock_resume.original_filename = "test.pdf"
        mock_resume.uploaded_at = datetime(2024, 1, 1, 12, 0, 0)
        mock_resume.file_size = 1024

        mock_analysis = MagicMock()
        mock_analysis.overall_score = 85
        mock_analysis.content_score = 90
        mock_analysis.structure_score = 80
        mock_analysis.skill_match_score = 85
        mock_analysis.expression_score = 82
        mock_analysis.project_score = 88
        mock_analysis.summary = "这是一份优秀的简历"
        mock_analysis.strengths_json = "['技术栈全面', '项目经验丰富']"
        mock_analysis.suggestions_json = "[]"

        result = pdf_service.generate_resume_report(mock_resume, mock_analysis)

        assert result is not None
        assert result.startswith(b'%PDF')
```

---

## 9. 验收标准

1. ✅ 支持 PDF、DOCX、TXT 文件上传和解析
2. ✅ 简历哈希去重检测
3. ✅ 简历文本提取存储
4. ✅ 简历列表和详情查询
5. ✅ 简历删除功能（文件+数据库）
6. ✅ 简历分析状态管理
7. ✅ PDF 报告生成
8. ✅ 用户数据隔离（user_id 过滤）
9. ✅ 单元测试覆盖

---

## 10. 后续任务

- [SDD-07: 面试模块](SDD-07-面试模块.md) - 依赖本任务的简历模型
- [SDD-09: 异步任务处理](SDD-09-异步任务处理.md) - 依赖本任务的简历分析状态管理
- [SDD-10: LangGraphAgents](SDD-10-LangGraphAgents.md) - 依赖本任务的简历分析逻辑
