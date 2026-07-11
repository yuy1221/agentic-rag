# deep — 基于 Multi-Agent 架构的 RAG 智能问答系统

一个融合了 **LangGraph Multi-Agent**、**混合检索（稠密+稀疏）**、**多级文档分块** 与 **实时思考追踪** 的智能知识库问答平台。前端采用类 DeepSeek 风格的对话交互界面。

[![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688.svg)](https://fastapi.tiangolo.com/)
[![Vue](https://img.shields.io/badge/Vue-3.3+-4FC08D.svg)](https://vuejs.org/)
[![Milvus](https://img.shields.io/badge/Milvus-2.5+-00BFFF.svg)](https://milvus.io/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 系统架构

```
┌──────────────────────────────────────────────────────────────┐
│                        前端 (Vue 3)                          │
│              Vite + TypeScript + Pinia + SSE                  │
│              类 DeepSeek 风格的对话交互界面                    │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTP/SSE
┌──────────────────────────▼───────────────────────────────────┐
│                    后端 (FastAPI)                             │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐  ┌────────────┐  │
│  │  Auth   │  │   Chat   │  │  Documents │  │  Sessions  │  │
│  │  JWT    │  │ Service  │  │   Upload   │  │  History   │  │
│  │  RBAC   │  │  SSE     │  │   Parse    │  │  Summary   │  │
│  └─────────┘  └────┬─────┘  └─────┬──────┘  └────────────┘  │
│                    │               │                          │
│  ┌─────────────────▼───────────────▼──────────────────────┐  │
│  │              LangGraph Agent 引擎                       │  │
│  │  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐  │  │
│  │  │ Main Agent   │  │ Sub-Agent│  │ Sub-Agent  ...   │  │  │
│  │  │ (路由/编排)  │  │  (并行)  │  │  (并行)          │  │  │
│  │  └──────┬───────┘  └────┬─────┘  └────────┬─────────┘  │  │
│  │         │               │                 │             │  │
│  │         ▼               ▼                 ▼             │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │              RAG Pipeline                         │   │  │
│  │  │   召回 → 自动合并 → 语义重排 → 生成               │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────┬──────────┬──────────┬────────────────────────────┘
           │          │          │
┌──────────▼──┐ ┌────▼────┐ ┌──▼──────────┐
│   Milvus    │ │PostgreSQL│ │   Redis     │
│  向量数据库  │ │ 关系存储 │ │  缓存       │
│ (稠密+BM25) │ │ 会话/文档│ │ 热点数据    │
└─────────────┘ └─────────┘ └─────────────┘
```

---

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **LLM** | 豆包 Seed 2.0 (Ark API) | 主模型 / 评估模型 / 快速模型 |
| **嵌入模型** | BAAI/bge-m3 | 本地 HuggingFace 部署，1024 维稠密向量 |
| **Agent 框架** | LangGraph + LangChain | Multi-Agent 编排、工具调用、条件路由 |
| **后端框架** | FastAPI + Uvicorn | RESTful API + SSE 流式响应 |
| **向量数据库** | Milvus 2.5 | 稠密向量检索 + BM25 稀疏向量混合检索 |
| **关系数据库** | PostgreSQL 15 | 会话历史、用户数据、父级文档分块 |
| **缓存** | Redis 7 | 热点会话缓存、父文档缓存 |
| **前端** | Vue 3 + Vite + TypeScript | Pinia 状态管理、SSE 实时推送 |
| **文档解析** | PyPDF / docx2txt / Unstructured | 多格式文档解析与向量化 |

---

## 核心特性

### 🧠 智能 RAG 流水线
- **三级滑动窗口分块**：L1（粗粒度）→ L2（中粒度）→ L3（细粒度叶子块），叶子块入库 Milvus
- **混合检索**：稠密向量 (bge-m3) + BM25 稀疏向量，RRF (Reciprocal Rank Fusion) 融合排序
- **Auto-Merging**：检索后自动从 L3 叶子向上合并 L2/L1 父级上下文，保证语义完整性
- **语义重排**：精排模型二次过滤，`RERANK_MIN_SCORE` 阈值拦截噪声
- **Step-back 重写**：精排无结果时自动扩展查询改写，长尾问题兜底

### 🤖 Multi-Agent 协同
- **LangGraph 状态图编排**：主 Agent 负责问题分解与路由
- **并行子 Agent**：利用 `Send` API 将子问题并行分发到独立 `rag_sub_agent`
- **条件路由**：根据问题复杂度自适应选择检索策略
- **工具集成**：知识库检索工具 + 天气查询工具（高德 API）

### 💬 实时交互体验
- **SSE 流式生成**：逐字输出 LLM 回复，低延迟感知
- **思考追踪**：实时展示检索步骤、文档评估、合并重排过程
- **子 Agent 分组**：并行流程可视化，各子问题独立展示

### 🔐 用户系统
- JWT Token 鉴权 + PBKDF2-SHA256 密码哈希
- RBAC 权限控制（admin / user）
- 管理员：文档知识库管理、用户管理
- 普通用户：对话交互、会话历史

### 📚 文档管理
- 多格式支持：PDF / DOCX / XLSX / HTML
- 异步上传 + 进度轮询 + 向量化入库
- 引用溯源：回答中标明参考文档片段

---

## 项目结构

```
SuperMew-main/
├── backend/                    # Python 后端
│   ├── app.py                  # FastAPI 应用入口
│   ├── env.py                  # 环境变量加载
│   ├── api/                    # API 路由层
│   │   ├── router.py           # 路由注册
│   │   ├── resources.py        # 静态资源托管
│   │   └── routes/             # 业务路由
│   │       ├── auth.py         # 认证接口
│   │       ├── chat.py         # 聊天接口 (SSE)
│   │       ├── documents.py    # 文档管理
│   │       └── sessions.py     # 会话管理
│   ├── chat/                   # 对话核心
│   │   ├── service.py          # 对话业务逻辑
│   │   ├── runtime.py          # LangGraph Agent 运行时
│   │   ├── streaming.py        # SSE 流式输出
│   │   ├── rag_context.py      # RAG 上下文管理
│   │   └── storage.py          # 消息持久化
│   ├── rag/                    # RAG 流水线
│   │   ├── pipeline.py         # 召回→合并→重排序
│   │   └── utils.py            # 检索工具函数
│   ├── indexing/               # 文档索引
│   │   ├── document_loader.py  # 文档加载解析
│   │   ├── embedding.py        # 向量化服务 (bge-m3 + BM25)
│   │   ├── milvus_client.py    # Milvus 客户端
│   │   ├── milvus_writer.py    # 向量写入
│   │   ├── parent_chunk_store.py # 父级分块存储
│   │   └── html_processor.py   # HTML 预处理
│   ├── tools/                  # Agent 工具
│   │   ├── knowledge.py        # 知识库检索工具
│   │   └── weather.py          # 天气查询工具
│   ├── infra/                  # 基础设施
│   │   ├── auth.py             # JWT 鉴权
│   │   ├── cache.py            # Redis 缓存
│   │   └── database.py         # PostgreSQL 连接
│   ├── db/                     # 数据模型
│   │   └── models.py           # SQLAlchemy ORM
│   ├── schemas/                # Pydantic 数据校验
│   └── jobs/                   # 异步任务
│       └── upload_jobs.py      # 文档上传处理 Job
├── frontend/                   # Vue 3 前端
│   ├── index.html              # HTML 入口
│   ├── vite.config.ts          # Vite 配置
│   ├── package.json            # 前端依赖
│   └── src/
│       ├── App.vue             # 根组件
│       ├── main.ts             # 入口
│       ├── assets/styles/      # 全局样式 (CSS 变量)
│       ├── components/
│       │   ├── Sidebar.vue     # 侧边栏导航
│       │   ├── AuthPanel.vue   # 登录/注册
│       │   ├── HistorySidebar.vue # 历史会话
│       │   ├── Chat/           # 聊天组件
│       │   │   ├── ChatArea.vue       # 聊天区域
│       │   │   ├── ChatInput.vue      # 输入框
│       │   │   ├── MessageItem.vue    # 消息项
│       │   │   ├── MessageContent.vue # 消息内容 (Markdown)
│       │   │   ├── WelcomeScreen.vue   # 欢迎页
│       │   │   ├── ThinkingTrace.vue  # 思考追踪
│       │   │   ├── References.vue     # 引用溯源
│       │   │   └── RetrievalTraceDetails.vue # 检索详情
│       │   └── Documents/      # 文档管理
│       ├── stores/             # Pinia 状态管理
│       │   ├── auth.ts         # 认证状态
│       │   ├── chat.ts         # 聊天状态 (SSE)
│       │   ├── sessions.ts     # 会话状态
│       │   └── documents.ts    # 文档状态
│       ├── types/              # TypeScript 类型定义
│       └── utils/              # 工具函数
│           ├── api.ts          # Axios 封装
│           └── markdown.ts     # Markdown 渲染
├── docker-compose.yml          # Docker 编排
├── pyproject.toml              # Python 项目配置
├── .env.example                # 环境变量模板
└── data/                       # 运行时数据 (BM25 状态等)
```

---

## 快速开始

### 前置要求

- **Python** `3.12+`
- **Node.js** `18+`
- **Docker** & **Docker Compose**
- 包管理工具推荐使用 [`uv`](https://docs.astral.sh/uv/)

### 1. 克隆项目

```bash
git clone https://github.com/yuy1221/agentic-rag.git
cd agentic-rag

### 2. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env` 文件，填入你的 LLM API Key 和其他配置：

```ini
# 必填
ARK_API_KEY=your_api_key_here
MODEL=your_model_name

# 可选（使用默认值即可运行）
JWT_SECRET_KEY=your-secret-key
ADMIN_INVITE_CODE=your-admin-code
```

### 3. 启动基础设施

```bash
# 启动 PostgreSQL + Redis + Milvus (含 etcd/minio)
docker compose up -d

# 确认所有服务就绪
docker compose ps
```

### 4. 安装后端依赖

```bash
# 使用 uv（推荐）
uv sync

# 或使用 pip
python -m venv .venv
.venv\Scripts\activate   # Windows
pip install -e .
```

### 5. 编译前端

```bash
cd frontend
npm install
npm run build
cd ..
```

### 6. 启动应用

```bash
# 启动后端（托管前端静态资源）
cd backend
python app.py（无科学上网启动代理：$env:HF_ENDPOINT="https://hf-mirror.com"; python app.py

#或者
uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000 --reload
```

浏览器访问：
- 前端页面：http://127.0.0.1:8000/
- API 文档 (Swagger)：http://127.0.0.1:8000/docs

### 前端开发模式（可选）

```bash
cd frontend
npm run dev      # 启动开发服务器 http://localhost:3000
                 # 内置反向代理至后端 8000 端口
```

---

## 端口说明

| 服务 | 端口 | 用途 |
|------|------|------|
| FastAPI 后端 | `8000` | API + 前端静态托管 |
| Vite 开发服务器 | `3000` | 前端热更新开发 |
| PostgreSQL | `5432` | 关系数据库 |
| Redis | `6379` | 缓存 |
| Milvus | `19530` | 向量数据库 |
| Milvus 健康检查 | `9091` | 健康监控 |
| MinIO API | `9000` | 对象存储 (Milvus 依赖) |
| MinIO Console | `9001` | MinIO 管理面板 |
| Attu | `8080` | Milvus 可视化管理工具 |

---

## RAG 工作流

```
用户提问
    │
    ▼
主 Agent 分析 ──→ 简单问题 ──→ 直接检索
    │                            │
    │              复杂问题      │
    ├──→ 问题分解 ──→ Send API ──┤
    │         │                  │
    │    ┌────┴────┐             │
    │    ▼         ▼             ▼
    │  Sub 1    Sub 2      召回 (Dense + BM25)
    │    │         │             │
    │    ▼         ▼             ▼
    │  各自 RAG  各自 RAG    Auto-Merge (L3 → L2/L1)
    │    │         │             │
    │    └────┬────┘             ▼
    │         ▼             Semantic Rerank
    │      结果汇聚              │
    │         │           ┌──────┴──────┐
    │         ▼           ▼             ▼
    └──→ 最终合成 ←── 通过阈值     Step-back 重写
              │
              ▼
         SSE 流式输出
```

---

## 环境变量要点

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ARK_API_KEY` | LLM API Key | 必填 |
| `MODEL` | 主模型名称 | 必填 |
| `EMBEDDING_MODEL` | 嵌入模型 | `BAAI/bge-m3` |
| `EMBEDDING_DEVICE` | 嵌入设备 | `cpu` |
| `DENSE_EMBEDDING_DIM` | 稠密向量维度 | `1024` |
| `MILVUS_HOST` | Milvus 地址 | `127.0.0.1` |
| `MILVUS_PORT` | Milvus 端口 | `19530` |
| `DATABASE_URL` | PostgreSQL 连接串 | `postgresql+psycopg2://...` |
| `REDIS_URL` | Redis 连接串 | `redis://127.0.0.1:6379/0` |
| `RETRIEVAL_CANDIDATE_MULTIPLIER` | 召回候选倍数 | `3` |
| `AUTO_MERGE_THRESHOLD` | 自动合并阈值 | `2` |
| `RERANK_MIN_SCORE` | 精排最低分 | `0.0` |
| `HF_ENDPOINT` | HuggingFace 镜像 | `https://hf-mirror.com` |

完整变量列表见 [`.env.example`](.env.example)。

---

## 开发日志

- **2026-07-01** — Multi-Agent RAG 架构：LangGraph 并行子 Agent + Step-back 重写 + 精排阈值
- **2026-06-02** — 通用 RAG 增强：思考模式切换 + 会话摘要 + 智能标题生成
- **2026-06-01** — Rerank 流水线重构：召回-合并-精排模块化
- **2026-04-08** — 本地嵌入模型 (bge-m3) + BM25 统计持久化
- **2026-03-21** — 后端升级：JWT 鉴权 + PostgreSQL + Redis + RBAC
- **2026-03-13** — 三级滑动窗口分块 + Auto-Merging 父子检索
- **2026-02-19** — RAG 实时思考链路修复 (SSE 事件跨线程调度)

---

## License

MIT License