# 🔬 AI Research Agent

An autonomous research assistant built with **FastAPI** and a local **Ollama** LLM. Give it a topic and the agent searches the web, reads pages, records findings, verifies each finding against its evidence, and writes a structured Markdown report. Its progress streams live to a browser UI.

---

## 🚀 Features

### 🧭 Tool-Calling Research Agent
- Runs a step-limited loop (`search_web` → `read_webpage` → `add_research_note`) through Ollama tool calling
- Stops early once enough verified findings are collected (`TARGET_VERIFIED_NOTES`)
- If the step limit is reached, it writes the report from whatever verified findings it has

### 🔎 Web Search & Reading
- Searches DuckDuckGo's HTML endpoint, so no API key is needed
- Unwraps DuckDuckGo redirect links to get the real URLs
- Cuts page text down so it fits in the model's context window

### ✅ Evidence Verification
- Each new note goes to a separate LLM check that returns JSON: `{ "supported": bool, "explanation": str }`
- Only verified notes are used in the final report

### 📝 Report Generation
- Turns the verified findings into a short, professional Markdown report

### 📡 Live Progress Streaming
- `GET /research` streams Server-Sent Events: `plan`, `step`, `tool`, `tool_complete`, `verification`, `complete` and `error`

### 🖥 Web Frontend
- A single-page UI (`frontend/`) served by the API, showing live agent activity and the finished report

---

## 🛠 Tech Stack

### Core
- Python 3.10+
- [Ollama](https://ollama.com), running locally (default model: `qwen3:4b`)

### Libraries
- `fastapi`: API and static file serving
- `uvicorn`: ASGI server
- `pydantic`: request/response schemas
- `ollama`: LLM client with tool calling
- `requests` + `beautifulsoup4`: web search and page scraping

---

## 📂 Project Structure

```
ai-research-agent/
├── main.py            # FastAPI app: routes, SSE streaming, static frontend
├── agent.py           # Research loop, tool dispatch, report generation
├── tools.py           # Web search, page reader, note store, verification
├── schemas.py         # Pydantic request/response/event models
├── requirements.txt   # Python dependencies
└── frontend/
    ├── index.html     # UI layout
    ├── app.js         # Calls /research and renders streamed events
    └── style.css      # Styling
```

---

## ⚙ Installation & Setup

### 1️⃣ Clone the repository

```
git clone https://github.com/mishraabhishek11/ai-research-agent.git
cd ai-research-agent
```

### 2️⃣ Create a virtual environment

```
python -m venv .venv
.venv\Scripts\activate        # Windows
source .venv/bin/activate     # macOS / Linux
```

### 3️⃣ Install dependencies

```
pip install -r requirements.txt
```

### 4️⃣ Install Ollama and pull the model

```
ollama pull qwen3:4b
```

To use a different model, change `MODEL` in both `agent.py` and `tools.py`. Pick a model that supports tool calling.

### 5️⃣ Run the server

```
uvicorn main:app --reload
```

Open http://127.0.0.1:8000 in your browser.

---

## 🧑‍💻 Usage

### Web UI
1. Open http://127.0.0.1:8000
2. Enter a research topic
3. Watch the agent's steps stream in, then read the final Markdown report

### API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Serves the frontend |
| `GET` | `/health` | Health check: `{ "status": "healthy" }` |
| `POST` | `/chat` | Body `{ "message": "<topic>" }`; returns `{ "response": "<markdown report>" }` once finished |
| `GET` | `/research?topic=<topic>` | Streams progress as Server-Sent Events, ending with a `complete` event that holds the report |

```
curl -X POST http://127.0.0.1:8000/chat -H "Content-Type: application/json" -d "{\"message\": \"solid-state batteries\"}"

curl -N "http://127.0.0.1:8000/research?topic=solid-state%20batteries"
```

Interactive API docs are at http://127.0.0.1:8000/docs.

---

## 🔁 How It Works

1. Research notes from any previous run are cleared
2. The LLM is prompted to call tools in a fixed order: search, read, note
3. Each tool call runs, and its result goes back to the model (page text is truncated)
4. Every new note is checked by a verification prompt
5. The loop ends when enough notes are verified, the step limit is reached, or `max_steps` runs out
6. Verified notes are combined into the final Markdown report

```
Topic
  │
  ▼
┌──────────────┐   tool calls   ┌──────────────────────────────┐
│  LLM Agent   │ ─────────────▶ │ search_web / read_webpage /  │
│  (Ollama)    │ ◀───────────── │ add_research_note            │
└──────┬───────┘    results     └──────────────┬───────────────┘
       │                                       │ new note
       │                                       ▼
       │                              ┌─────────────────┐
       │        enough verified? ◀─── │ verify_finding  │
       │                              └─────────────────┘
       ▼
┌──────────────────┐
│ generate_report  │ ──▶ Markdown report (SSE `complete` event)
└──────────────────┘
```

---

## ➕ Adding Your Own Tool

1. Write a typed Python function with a docstring in `tools.py` (Ollama builds the tool schema from the function signature)
2. Import it in `agent.py`
3. Add it to both `AVAILABLE_TOOLS` and `TOOLS`
4. Update the system prompt in `research_agent` so the model knows when to use it

---

## 🩺 Troubleshooting

| Error | Cause / Fix |
|---|---|
| `[Errno 10048] ... only one usage of each socket address` | Port 8000 is already in use, usually by another `uvicorn` instance. Stop it (`netstat -ano \| findstr :8000`, then `Stop-Process -Id <PID>`) or run with `--port 8001` |
| `Execution error: ... connection refused` | Ollama isn't running. Start the Ollama app or run `ollama serve` |
| `model "qwen3:4b" not found` | Run `ollama pull qwen3:4b` |
| `No verified findings collected during research.` | The model gathered no usable notes. Try a more specific topic or a larger model |
| `Failed to perform search` | Network problem or DuckDuckGo rate limiting. Wait and try again |

---

## ⚠ Known Limitations

- **Single user only:** research notes live in a module-level list in `tools.py`, so two research requests running at once will overwrite each other's notes
- **Weak verification:** if the verification response can't be parsed, the note is marked verified anyway, and the note matching falls back to verifying the first unverified note
- **Open CORS:** `allow_origins=["*"]` is fine locally; restrict it before deploying anywhere

---

## 🎯 Learning Objectives

- Building a tool-calling agent loop with a local LLM
- Streaming long-running work to a browser with Server-Sent Events
- Running blocking LLM calls off the async event loop (`run_in_threadpool`, `asyncio.to_thread`)
- Keeping a small model on task with prompts, step limits and stop conditions

---

## 🔮 Future Enhancements

- Store notes per request so several users can research at once
- Configure the model and limits through environment variables (`.env`)
- Stricter verification with no auto-pass fallback
- Source citations and links in the final report
- Unit tests for the tools and integration tests for the endpoints

---

## 🤝 Contributing

1. Fork the repository
2. Create a branch
   ```
   git checkout -b feat/your-feature
   ```
3. Commit your changes
   ```
   git commit -m "feat(agent): add your feature"
   ```
4. Push the branch
   ```
   git push origin feat/your-feature
   ```
5. Open a Pull Request

---

## 👨‍💻 Author

[@mishraabhishek11](https://github.com/mishraabhishek11)

---

## ⭐ Support

If you find this project useful, please give it a ⭐ on GitHub!
