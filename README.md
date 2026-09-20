# 基于混合检索的图书馆智能办事助手

把 **混合检索（BM25 + 向量 + RRF + 重排）** 与 **LLM Tool Calling**
组合成"咨询 + 办事"一体的图书馆智能助手，实现从"找到信息"到"完成办事"的闭环。

```
用户问题 → 意图识别（LLM / 规则）
              ├── 咨询类 → BM25 Top-K + Vector Top-K → RRF 融合 → BGE-Reranker 精排
              │            → LLM 生成（严格基于证据 + [n] 来源引用）
              └── 办事类 → Tool Calling（白名单 + Pydantic 校验）→ Service → SQLite
                          → 查询借阅 / 查询图书 / 续借 / 预约研讨室 / 取消预约
```

## 目录

| 路径 | 说明 |
| --- | --- |
| `backend/` | **后端**：FastAPI + SQLAlchemy + 混合检索 + RAG + Tool Calling，见 [`backend/README.md`](backend/README.md) |
| `frontend/` | **前端**：React 19 + TypeScript + Vite + Tailwind + three.js（作者星系可视化、办事中心、流式问答） |
| `启动演示.bat` | **一键启动**：拉起后端（同时托管前端界面）并自动打开浏览器 |
| `backend/app/static/` | 原版 Swagger UI / ReDoc 静态资源（本地托管，代理开关都能用） |
| `backend/knowledge/` | 知识库文档（借阅规则、服务指南、研讨室预约办法） |
| `backend/scripts/` | 自检 / 初始化 / 建索引 / 评测 / 演示 / 文档资源 / OpenAPI 校验 / LLM 自检 |
| `backend/tests/` | 单元检查 16 项 + 端到端冒烟 64 项（不依赖 pytest） |
| `extract_docx.py` | 两份 docx 方案文档的纯文本抽取脚本 |
| `final_design.txt`、`design_phase.txt` | 两份文档的纯文本版（段落结构，便于检索与 diff） |
| `基于混合检索的图书馆智能办事助手-*.docx` | 方案设计文档 / 设计阶段文档（原件） |
| `TODO.md` | 开发进度与待办清单 |

## 快速开始

### 方式一：一键启动（演示/答辩推荐）

**双击项目根目录的 `启动演示.bat`** —— 它会启动后端并自动打开浏览器。

后端会**直接托管前端构建产物**（`frontend/dist`），因此：

| 地址 | 内容 |
| --- | --- |
| <http://127.0.0.1:8000/> | 前端界面（星海书库） |
| <http://127.0.0.1:8000/docs> | Swagger UI 接口文档 |
| <http://127.0.0.1:8000/api/v1/system/status> | 运行状态（数据库 / 检索组件 / LLM 后端） |

**一个进程同时提供 API + UI**，不需要再单独起前端服务，也不存在跨域问题。
（如果 `frontend/dist` 不存在，脚本会提示先在 `frontend/` 执行 `npm install && npm run build`。）

### 方式二：开发模式（改前端代码时用）

```powershell
# 终端 A：后端
cd backend
python -m uvicorn app.main:app --port 8000 --reload

# 终端 B：前端（Vite 热更新）
cd frontend
npm run dev            # http://127.0.0.1:5173
```

### 两种方式怎么选（重要）

| | 方式一：一键启动（演示模式） | 方式二：开发模式 |
| --- | --- | --- |
| 进程数 | **1 个**（后端同时提供 API + 界面） | 2 个（后端 + Vite） |
| 访问地址 | <http://127.0.0.1:8000/> | <http://127.0.0.1:5173/> |
| 前端来源 | `frontend/dist`（构建产物，静态文件） | 源码实时编译（改代码秒级生效） |
| 是否需要 build | **需要**：改过前端源码后要 `npm run build` | 不需要，Vite 自动热更新 |
| 跨域 | 不需要（同源） | 需要 CORS（后端已放行本机端口） |
| 适用 | **答辩/演示**、给别人跑 | 开发前端时 |

> 原理：前端 `npm run build` 之后就是一堆静态文件（`dist/index.html` + `assets/*.js|css`），
> 后端用 `StaticFiles` 把它们挂在 `/` 上（配置项 `SERVE_FRONTEND=true`），
> 所以"启动后端"就等于"前端也起来了"——浏览器拿到的是同一份构建产物。
> 这不是把前端"编译进后端"，后端只是**托管**了这些文件；如果 `frontend/dist` 不存在，
> 后端会退化为只在 `/` 返回服务信息 JSON。
>
> 这种"前端构建产物由后端/服务端托管"是业界很常见的部署方式（真实线上通常换成 Nginx/CDN），
> 你以前"先起后端再起前端"是**开发模式**，两者不冲突：开发用方式二，演示用方式一。

### 后端常用命令

```bash
cd backend
python scripts/check_stack.py            # 环境自检
python scripts/check_llm.py              # 在线大模型自检（Key / 连通性）

## 5 个核心演示场景

| 场景 | 提问示例 | 展示的技术点 |
| --- | --- | --- |
| 规则咨询 | 图书馆周末几点开门？ | 混合检索 + 来源引用 |
| 同义改写召回 | 书还没看完可以再借一段时间吗 | 向量召回 + RRF + 重排 |
| 馆藏查询 | 有没有人工智能方面的书？/ 查 ISBN 978-7-111-42876-3 | Tool Calling + SQL 事实查询 |
| 借阅与续借 | 我现在借了哪些书？/ 帮我续借 | Tool 白名单 + 制度校验（限续借一次） |
| 空间预约 | 帮我预约明天下午3点的研讨室A02 | 中文时间解析 + 冲突检测 + 受控写库 |

## 交付物清单

- [x] 可运行后端源码（`backend/app`，分层：api / services / repositories / models / schemas / rag / llm / core）
- [x] 可运行前端源码（`frontend/`，React 19 + TypeScript + Vite + Tailwind + three.js 星系可视化）
- [x] 模拟馆藏数据：**312 种图书 / 927 册副本 / 272 位作者 / 12 个分类**（`scripts/seed_books.py`）
- [x] 知识库文档与索引构建脚本（`backend/knowledge`、`scripts/build_index.py`）
- [x] 数据库初始化脚本与模拟数据（`scripts/init_data.py`、`scripts/seed_books.py`）
- [x] 检索对比实验（`scripts/run_eval.py` → `data/eval_results.json`，含 Recall@K / Precision@K / MRR）
- [x] 单元检查与端到端冒烟测试（`tests/unit_checks.py`、`tests/smoke_test.py`）
- [x] 8 个完整演示案例（`scripts/demo_scenarios.py`）
- [x] README（环境搭建、运行方式、演示步骤、降级说明、修复清单）
- [ ] 答辩 PPT / 演示讲稿（按你的计划不做，答辩 5 分钟口头讲）

## 当前实现状态

| 模块 | 状态 |
| --- | --- |
| 业务数据库（9 张表 + 外键关系） | ✅ 完成 |
| BM25 / 向量 / RRF / 重排 四模式检索 | ✅ 完成（可自动降级） |
| RAG 问答 + 来源引用 + 证据不足拒答 | ✅ 完成 |
| 意图识别（LLM + 规则兜底 + 中文时间解析） | ✅ 完成 |
| Tool Calling（6 个工具，白名单 + Pydantic 校验） | ✅ 完成 |
| SSE 流式输出 | ✅ 完成 |
| 鉴权（JWT + bcrypt/PBKDF2 分层哈希）、管理端接口 | ✅ 完成 |
| 原版 Swagger UI（本地资源 / CDN+本地兜底两种模式、Authorize、OpenAPI 3.1） | ✅ 完成 |
| 前端：星海星系 / 作者书籍详情 / 办事中心（借阅·续借·归还·预约） | ✅ 完成 |
| 前端：SSE 流式问答 + 来源卡片 + 工具调用卡片 + 登录注册 | ✅ 完成 |
| **DeepSeek 在线大模型**（Key 已配置，意图识别 / 生成 / 工具总结） | ✅ 完成 |
| **检索查询扩展**（口语 → 知识库用词，解决"同义问法答不上来"） | ✅ 完成 |
| 一键启动（后端同时托管前端，一个地址） | ✅ 完成 |
| 单元检查 16 项 + 端到端冒烟 64 项 + OpenAPI 校验 + 文档页自检 | ✅ 全部通过 |

