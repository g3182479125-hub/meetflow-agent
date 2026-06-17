# MeetFlow Agent

MeetFlow Agent is a Python-based multi-agent meeting assistant built with FastAPI, LangGraph, WhisperX, and LLM APIs. It turns meeting audio into structured transcripts, meeting summaries, action items, insights, and follow-up reports.

The project focuses on enterprise meeting automation: from meeting recording to task tracking, report delivery, and knowledge reuse.

## Features

- Audio upload through REST API and real-time meeting connection through WebSocket.
- Speech-to-text transcription with WhisperX.
- Optional speaker diarization with pyannote-audio.
- Structured meeting summary generation with an LLM.
- Action item extraction with assignee, task, deadline, priority, and context.
- Meeting insight analysis, including speaker statistics, sentiment, keywords, highlights, and suggestions.
- Follow-up report generation for post-meeting delivery.
- Optional Jira issue creation and Feishu task/message integration.
- Optional SMTP email delivery for meeting reports.
- ChromaDB-based meeting knowledge storage for future retrieval and reuse.

## Architecture

MeetFlow Agent uses LangGraph to split the meeting workflow into five agent nodes.

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

Workflow overview:

1. Transcription Agent converts meeting audio into timestamped transcript segments.
2. Summary Agent, Action Agent, and Insight Agent run after transcription and handle different analysis tasks.
3. Follow-up Agent merges the outputs and generates a meeting report.
4. Optional integrations push action items and reports to Jira, Feishu, email, and ChromaDB.

This design avoids putting all logic into a single prompt. Each node has a clear responsibility, which makes the system easier to debug, extend, and evaluate.

## Tech Stack

| Area | Technology |
| --- | --- |
| Web service | FastAPI, Uvicorn, WebSocket |
| Agent orchestration | LangGraph, LangChain |
| Speech-to-text | WhisperX, faster-whisper, pyannote-audio, torch |
| LLM client | MiniMax API / OpenAI-compatible API |
| Data models | Pydantic |
| Vector storage | ChromaDB |
| Enterprise integrations | Jira Cloud API, Feishu Open API, SMTP Email |
| Engineering | Docker, Docker Compose, pytest, loguru, tenacity |

## Project Structure

```text
.
|-- src/
|   |-- agents/
|   |   |-- transcription_agent.py
|   |   |-- summary_agent.py
|   |   |-- action_agent.py
|   |   |-- insight_agent.py
|   |   `-- followup_agent.py
|   |-- graph/
|   |   `-- meeting_graph.py
|   |-- integrations/
|   |   |-- minimax_client.py
|   |   |-- jira_client.py
|   |   |-- feishu_client.py
|   |   |-- email_client.py
|   |   `-- chroma_client.py
|   |-- models/
|   |   `-- schemas.py
|   |-- websocket/
|   |   `-- server.py
|   `-- main.py
|-- requirements.txt
|-- Dockerfile
|-- docker-compose.yml
|-- .env.example
`-- README.md
```

## Local Setup

### 1. Requirements

Recommended environment:

- Python 3.11+
- FFmpeg
- Optional: Docker and Docker Compose
- Optional: NVIDIA GPU for faster WhisperX inference

Speaker diarization requires a Hugging Face token and access to the related pyannote models.

### 2. Install Dependencies

Windows PowerShell:

```powershell
cd <your-project-path>
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

Linux / macOS:

```bash
cd /path/to/meetflow-agent
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Configure Environment Variables

Copy the template:

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Common configuration:

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

For demo mode, external integrations are optional. If Jira, Feishu, email, or Hugging Face are not configured, the core demo workflow can still run.

### 4. Start the Server

```bash
python -m src.main
```

Or start with Uvicorn:

```bash
uvicorn src.websocket.server:app --host 0.0.0.0 --port 8000 --reload
```

Open the API documentation:

```text
http://localhost:8000/docs
```

Health check:

```text
http://localhost:8000/
```

## Docker Setup

```bash
docker compose up -d --build
```

The Docker Compose file includes:

- `meeting-assistant`: FastAPI service
- `postgres`: optional relational database service
- `redis`: optional cache / queue service

The current core workflow can run without PostgreSQL and Redis. In production, meeting records, task status, and execution logs should be persisted to a database.

## API Examples

### Health Check

```bash
curl http://localhost:8000/
```

### Create a Meeting

```bash
curl -X POST http://localhost:8000/api/v1/meeting/start \
  -H "Content-Type: application/json" \
  -d '{"title":"Q3 Budget Review","participants":["Alice","Bob"],"language":"zh"}'
```

### Run Demo Mode

Demo mode uses a built-in transcript and does not require an audio file.

```bash
curl -X POST http://localhost:8000/api/v1/meeting/demo/demo
```

### Upload Audio

```bash
curl -X POST "http://localhost:8000/api/v1/meeting/{meeting_id}/upload" \
  -F "file=@meeting.wav" \
  -F "language=zh"
```

Supported audio formats:

- `.wav`
- `.mp3`
- `.m4a`
- `.flac`
- `.ogg`

### Query Results

```bash
curl http://localhost:8000/api/v1/meeting/demo/summary
curl http://localhost:8000/api/v1/meeting/demo/actions
curl http://localhost:8000/api/v1/meeting/demo/insights
curl http://localhost:8000/api/v1/meeting/demo/report
```

## WebSocket Protocol

Endpoint:

```text
ws://localhost:8000/ws/meeting/{meeting_id}
```

Client messages:

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

Server event types:

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

## Testing

```bash
pytest src/tests -q
```

Current tests cover:

- ChromaDB meeting storage
- Email client behavior
- Follow-up agent integration with email and vector storage

## Engineering Highlights

- LangGraph-based multi-agent workflow instead of a single prompt chain.
- Fan-out/Fan-in design for summary, action extraction, and insight analysis.
- Pydantic models for stable state exchange between agents.
- Retry support for external API calls through Tenacity.
- Graceful degradation when optional enterprise integrations are not configured.
- ChromaDB storage for meeting knowledge reuse.

## Notes

- WhisperX and pyannote-audio are heavy dependencies. The first installation and model download may take time.
- CPU inference works but can be slow. GPU is recommended for real meeting audio.
- Jira, Feishu, SMTP, and Hugging Face are optional for demo usage.
- `.env`, virtual environments, local cache, runtime data, and audio files are ignored by Git.

## Repository

GitHub: https://github.com/g3182479125-hub/meetflow-agent

## License

For learning, research, and portfolio demonstration.
