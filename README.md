# longchain-python · 私人厨师 Agent

基于 **LangChain / LangGraph + FastAPI** 的多模态 AI Agent 示例项目。

用户上传冰箱食材照片或食材清单，Agent 自动识别食材 → 联网检索菜谱 → 按营养与难度多维打分排序 → 输出结构化的膳食建议报告。后端同时托管前端静态页面，单进程即可提供完整服务。

## 功能特性

- **多模态输入**：支持图片 URL 与纯文本食材清单
- **联网检索菜谱**：内置 Tavily Web Search 工具，检索不到时才由模型自行发挥
- **多轮会话记忆**：基于 LangGraph `SqliteSaver` 按 `thread_id` 持久化，支持查询与清空
- **流式输出**：`text/event-stream` 逐字返回，前端无需等待完整响应
- **图片直传 OSS**：服务端只签发预签名 URL，图片由前端直传阿里云 OSS
- **开箱即用**：FastAPI 直接挂载 Next.js 预构建静态资源，并带 SPA fallback

## 技术栈

| 分类 | 选型 |
| --- | --- |
| 语言 / 依赖管理 | Python 3.13 · [uv](https://github.com/astral-sh/uv) |
| Agent 框架 | LangChain · LangGraph（`langgraph-checkpoint-sqlite`） |
| 模型 | 阿里云百炼 DashScope `qwen3-omni-flash`（多模态，OpenAI 兼容接口） |
| 工具 | Tavily Search |
| Web 框架 | FastAPI · Uvicorn |
| 存储 | SQLite（会话 checkpoint） · 阿里云 OSS（图片） |
| 前端 | Next.js 预构建产物（`app/static`） |

## 目录结构

```
longchain-python/
├── app/
│   ├── main.py                  # FastAPI 入口：日志、CORS、路由挂载、静态资源与 SPA fallback
│   ├── agents/
│   │   └── personal_chief.py    # 核心 Agent：模型 / 工具 / checkpointer / 系统提示词 / 会话读写
│   ├── api/v1/
│   │   ├── chat.py              # 对话接口：流式对话、历史查询、清空
│   │   └── oss.py               # OSS 预签名上传接口
│   ├── common/
│   │   └── logger.py            # 统一日志配置
│   ├── models/
│   │   └── schemas.py           # Pydantic 请求模型
│   └── static/                  # Next.js 预构建前端，由 FastAPI 托管
├── resources/                   # 本地运行时数据（SQLite，已被 .gitignore 忽略）
├── langgraph.json               # LangGraph CLI（langgraph dev）配置
├── pyproject.toml / uv.lock     # 依赖声明与锁定
└── .env.example                 # 环境变量模板
```

## 快速开始

### 1. 环境准备

需要 Python 3.13+ 与 uv：

```bash
pip install uv          # 或参考 https://docs.astral.sh/uv/ 安装
```

### 2. 安装依赖

```bash
uv sync
```

### 3. 配置环境变量

```bash
cp .env.example .env    # Windows: copy .env.example .env
```

至少填写 `DASHSCOPE_API_KEY` 与 `TAVILY_API_KEY`，其余按需填写，详见下方配置表。

### 4. 启动服务

```bash
uv run python -m app.main     # 开发模式（热重载），默认 http://127.0.0.1:8001
```

也可以直接用 LangGraph CLI 调试 Agent 图：

```bash
uv run langgraph dev
```

### 5. 访问

浏览器打开 http://127.0.0.1:8001 即可使用内置前端。

## API 说明

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/api/v1/chat/stream` | 流式对话（SSE），Body: `{message, image_url?, thread_id}` |
| `GET` | `/api/v1/chat/messages?thread_id=` | 查询指定会话历史消息 |
| `DELETE` | `/api/v1/chat/messages?thread_id=` | 清空指定会话历史 |
| `GET` | `/api/v1/oss/presign?filename=` | 获取 OSS 预签名上传 URL 与访问地址 |

示例：

```bash
curl -N -X POST http://127.0.0.1:8001/api/v1/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message":"冰箱里有鸡蛋、西红柿、牛腩，今晚吃什么？","thread_id":"demo-1"}'
```

## 环境变量

| 变量 | 必填 | 说明 |
| --- | :---: | --- |
| `DASHSCOPE_API_KEY` | 是 | 阿里云百炼 API Key，多模态模型调用 |
| `DASHSCOPE_BASE_URL` | 是 | DashScope OpenAI 兼容地址 |
| `TAVILY_API_KEY` | 是 | Tavily 联网搜索 |
| `OSS_ACCESS_KEY_ID` / `OSS_ACCESS_KEY_SECRET` | 否 | 阿里云 OSS 凭证，图片直传用 |
| `OSS_BUCKET` | 否 | OSS Bucket 名称 |
| `OSS_ENDPOINT` | 否 | OSS Endpoint，默认 `oss-cn-beijing.aliyuncs.com` |
| `SQLITE_DB_PATH` | 否 | 会话记忆库路径，默认 `resources/personal_chief.db` |
| `DEEPSEEK_API_KEY` | 否 | 可选，本地实验笔记使用 |
| `LANGSMITH_API_KEY` / `LANGSMITH_TRACING` | 否 | LangSmith 链路追踪，调试用 |

> 安全提示：`.env` 已被 `.gitignore` 忽略，请勿提交任何真实密钥。若密钥曾经进入过提交历史，请立即到对应平台轮换。

## License

[MIT](LICENSE)