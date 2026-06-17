# MeetFlow Agent - 多 Agent 智能会议助手

MeetFlow Agent 是一个基于 Python 的多 Agent 智能会议助手，使用 FastAPI、LangGraph、WhisperX 和大模型 API 构建。系统可以把会议音频转成结构化转写文本，并自动生成会议纪要、行动项、会议洞察和会后跟进报告。

这个项目面向企业会议自动化场景，目标是把“会议记录”进一步转化为“任务跟进、报告推送和知识沉淀”。

## 核心功能

- 音频接入：支持通过 REST API 上传会议音频，也支持通过 WebSocket 接入实时会议流。
- 语音转写：使用 WhisperX 完成语音转文字和时间戳对齐。
- 说话人识别：可接入 pyannote-audio，对不同发言人进行区分。
- 会议纪要：Summary Agent 根据转写文本生成议题、讨论要点、结论、决策和下一步计划。
- 行动项提取：Action Agent 自动提取负责人、任务内容、截止时间、优先级和上下文。
- 会议洞察：Insight Agent 输出发言统计、整体情绪、关键词、会议亮点和改进建议。
- 会后跟进：Follow-up Agent 汇总结果，生成会议报告，并支持推送到 Jira、飞书和邮件。
- 知识沉淀：通过 ChromaDB 存储会议摘要、行动项和洞察内容，方便后续检索和复用。

## 系统架构

系统使用 LangGraph 将会议处理流程拆成 5 个 Agent 节点：

```text
START
  |
  v
Transcription Agent
  |
  +--> Summary Agent
  +--> Action Agent
  +--> Insight Agent
          |
          v
Follow-up Agent
  |
  v
END
```

流程说明：

1. Transcription Agent 先把音频转换成带时间戳、发言人和置信度的转写结果。
2. Summary Agent、Action Agent 和 Insight Agent 在转写完成后并行执行，分别负责摘要、待办和洞察。
3. Follow-up Agent 汇总三个并行节点的结果，生成完整会议报告。
4. 如果配置了外部系统，系统会把待办和报告同步到 Jira、飞书、邮件和 ChromaDB。

这种设计避免把所有逻辑塞进一个大 Prompt，每个节点职责清晰，方便调试、扩展和评估。

## 技术栈

| 模块 | 技术 |
| --- | --- |
| Web 服务 | FastAPI、Uvicorn、WebSocket |
| Agent 编排 | LangGraph、LangChain |
| 语音转写 | WhisperX、faster-whisper、pyannote-audio、torch |
| 大模型调用 | MiniMax API / OpenAI-compatible API |
| 数据建模 | Pydantic |
| 向量存储 | ChromaDB |
| 企业集成 | Jira Cloud API、飞书 Open API、SMTP Email |
| 工程化 | Docker、Docker Compose、pytest、loguru、tenacity |

## 项目结构

```text
.
|-- src/
|   |-- agents/
|   |   |-- transcription_agent.py   # 语音转写 Agent
|   |   |-- summary_agent.py         # 会议纪要 Agent
|   |   |-- action_agent.py          # 行动项提取 Agent
|   |   |-- insight_agent.py         # 会议洞察 Agent
|   |   `-- followup_agent.py        # 会后跟进 Agent
|   |-- graph/
|   |   `-- meeting_graph.py         # LangGraph 编排核心
|   |-- integrations/
|   |   |-- minimax_client.py        # 大模型客户端
|   |   |-- jira_client.py           # Jira 集成
|   |   |-- feishu_client.py         # 飞书集成
|   |   |-- email_client.py          # 邮件发送
|   |   `-- chroma_client.py         # ChromaDB 向量库
|   |-- models/
|   |   `-- schemas.py               # Pydantic 数据结构
|   |-- websocket/
|   |   `-- server.py                # FastAPI / WebSocket 接口
|   `-- main.py                      # 服务入口
|-- requirements.txt
|-- Dockerfile
|-- docker-compose.yml
|-- .env.example
`-- README.md
```

## 本地部署

### 1. 环境要求

建议环境：

- Python 3.11+
- FFmpeg
- 可选：Docker 和 Docker Compose
- 可选：NVIDIA GPU，用于提升 WhisperX 推理速度

如果需要真实说话人识别，需要准备 Hugging Face Token，并确保可以访问 pyannote 相关模型。

### 2. 安装依赖

Windows PowerShell：

```powershell
cd <你的项目路径>
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

Linux / macOS：

```bash
cd /path/to/meetflow-agent
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. 配置环境变量

复制模板文件：

```bash
cp .env.example .env
```

Windows PowerShell：

```powershell
Copy-Item .env.example .env
```

常用配置如下：

```env
MINIMAX_API_KEY=your_minimax_api_key
MINIMAX_GROUP_ID=your_group_id
HF_TOKEN=your_huggingface_token
JIRA_SERVER=https://your-domain.atlassian.net
JIRA_EMAIL=your_email
JIRA_API_TOKEN=your_jira_token
FEISHU_APP_ID=your_feishu_app_id
FEISHU_APP_SECRET=your_feishu_app_secret
FEISHU_WEBHOOK_URL=your_feishu_webhook_url
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_email
SMTP_PASSWORD=your_password
SMTP_FROM=your_email
```

如果只跑 demo 模式，Jira、飞书、邮件和 Hugging Face 都可以先不配置。未配置的外部集成会自动跳过，不影响核心演示流程。

### 4. 启动服务

```bash
python -m src.main
```

也可以直接用 Uvicorn 启动：

```bash
uvicorn src.websocket.server:app --host 0.0.0.0 --port 8000 --reload
```

启动后访问：

```text
http://localhost:8000/docs
```

健康检查：

```text
http://localhost:8000/
```

## Docker 启动

```bash
docker compose up -d --build
```

Docker Compose 中包含：

- meeting-assistant：FastAPI 服务
- postgres：可选关系型数据库
- redis：可选缓存 / 队列服务

当前核心流程可以在不配置 PostgreSQL 和 Redis 的情况下运行。生产环境建议将会议记录、任务状态和执行日志持久化到数据库。

## API 示例

### 健康检查

```bash
curl http://localhost:8000/
```

### 创建会议

```bash
curl -X POST http://localhost:8000/api/v1/meeting/start \
  -H "Content-Type: application/json" \
  -d '{"title":"Q3预算评审会议","participants":["张总","李明"],"language":"zh"}'
```

### 运行演示模式

演示模式使用内置会议文本，不需要上传音频。

```bash
curl -X POST http://localhost:8000/api/v1/meeting/demo/demo
```

### 上传音频

```bash
curl -X POST "http://localhost:8000/api/v1/meeting/{meeting_id}/upload" \
  -F "file=@meeting.wav" \
  -F "language=zh"
```

支持的音频格式：

- `.wav`
- `.mp3`
- `.m4a`
- `.flac`
- `.ogg`

### 查询结果

```bash
curl http://localhost:8000/api/v1/meeting/demo/summary
curl http://localhost:8000/api/v1/meeting/demo/actions
curl http://localhost:8000/api/v1/meeting/demo/insights
curl http://localhost:8000/api/v1/meeting/demo/report
```

## WebSocket 协议

连接地址：

```text
ws://localhost:8000/ws/meeting/{meeting_id}
```

客户端消息：

```json
{"type":"start"}
```

```json
{"type":"stop"}
```

```json
{"type":"demo"}
```

```json
{"type":"ping"}
```

服务端事件类型：

- `connected`
- `recording`
- `processing`
- `transcript`
- `summary`
- `actions`
- `insights`
- `followup`
- `completed`
- `error`

## 测试

```bash
pytest src/tests -q
```

当前测试主要覆盖：

- ChromaDB 会议存储逻辑
- EmailClient 邮件发送逻辑
- Follow-up Agent 与邮件、向量库的集成逻辑

## 工程亮点

- 使用 LangGraph 编排多 Agent 流程，而不是单一 Prompt Chain。
- 使用 Fan-out/Fan-in 结构并行处理会议摘要、行动项和会议洞察。
- 使用 Pydantic 统一 Agent 之间传递的数据结构。
- 使用 Tenacity 为外部 API 调用提供重试机制。
- 外部系统未配置时自动降级，不影响核心会议分析流程。
- 使用 ChromaDB 沉淀会议知识，为后续历史会议检索和知识复用打基础。

## 注意事项

- WhisperX 和 pyannote-audio 依赖较重，首次安装和模型下载会比较慢。
- CPU 可以运行，但真实音频转写速度会比较慢，建议有 GPU 时使用 GPU。
- Jira、飞书、SMTP、Hugging Face 都是可选配置。
- `.env`、虚拟环境、本地缓存、运行数据和音频文件不会上传到 GitHub。

## 仓库地址

https://github.com/g3182479125-hub/meetflow-agent

## License

For learning, research, and portfolio demonstration.
