# 🎯 50 High-Impact Interview Cross-Questions & Technical Answers: NidhiVani AI Voice Banking Platform

This document contains **50 technical cross-examination questions** and **bullet-proof candidate answers** covering architecture, security, multi-agent AI, biometrics, multilingual processing, database design, and production scaling for **NidhiVani**.

---

## 📑 Table of Contents
1. [System Architecture & FastAPI Backend (Q1 – Q10)](#1-system-architecture--fastapi-backend)
2. [Multi-Agent Systems & Generative AI - LangGraph & Gemini (Q11 – Q20)](#2-multi-agent-systems--generative-ai---langgraph--gemini)
3. [Computer Vision & Facial Recognition Inference (Q21 – Q28)](#3-computer-vision--facial-recognition-inference)
4. [Voice Biometrics, Audio & Multilingual NLP (Q29 – Q35)](#4-voice-biometrics-audio--multilingual-nlp)
5. [Security, MPIN Safeguards & Financial State Management (Q36 – Q43)](#5-security-mpin-safeguards--financial-state-management)
6. [Database Design, Logging, Performance & Production Scaling (Q44 – Q50)](#6-database-design-logging-performance--production-scaling)

---

## 1. System Architecture & FastAPI Backend

### Q1: Why did you choose FastAPI instead of Django or Flask for NidhiVani?
**Answer:**  
FastAPI was chosen for three primary reasons:
1. **Asynchronous (async/await) Native I/O**: High-concurrency async capabilities are critical when orchestrating non-blocking generative AI streaming (Gemini API) and heavy biometric model inferences.
2. **Pydantic Validation & Data Integrity**: Incoming request payloads (e.g., 4-digit MPINs, base64 images) are automatically validated against Pydantic schemas (`schemas.py`), eliminating manual validation boilerplate.
3. **Low Overhead & Speed**: Built on Starlette and ASGI, FastAPI delivers performance comparable to NodeJS and Go.

---

### Q2: How is the application modularized across directory layers?
**Answer:**  
The codebase follows a clean domain-driven architecture:
- **`main.py`**: Entry point handling FastAPI initialization, CORS middleware, static asset mounting, and router registration.
- **`routers/`**: Decoupled domain routes (`auth.py`, `chat.py`, `accounts.py`, `payments.py`, `document.py`).
- **`agents_graph.py`**: LangGraph state graph defining agent nodes and intent routing.
- **`inference.py`**: Isolated computer vision module handling YOLOv8 and PyTorch model loading.
- **`models.py` & `database.py`**: SQLModel schemas and database sessions.

---

### Q3: How do you handle static assets and web clients in FastAPI?
**Answer:**  
We use FastAPI's `StaticFiles` mounting (`app.mount("/static", StaticFiles(directory="static"), name="static")`). The frontend is served as a Single-Page Application (SPA) where `index.html` interacts with backend endpoints via REST calls and JSON payloads.

---

### Q4: How does CORS middleware protect your backend endpoints?
**Answer:**  
We configure FastAPI's `CORSMiddleware` in `main.py` allowing specified origin hosts (`allow_origins=["*"]` during dev, restrictive domain origins in production), methods (`GET`, `POST`), and headers, preventing unauthorized cross-origin requests from browser environments.

---

### Q5: What happens if an unhandled exception occurs inside a route handler?
**Answer:**  
FastAPI catches unhandled exceptions and returns an HTTP 500 status code. In critical routes like `routers/chat.py`, we implement explicit `try-except` blocks that capture exceptions (such as LLM API rate limits) and gracefully trigger a local fallback response rather than returning raw stack traces to the client.

---

### Q6: How do you handle heavy blocking I/O operations (like model inference) without freezing FastAPI's event loop?
**Answer:**  
Synchronous heavy functions (such as deep learning inference in `inference.py` or PDF generation via ReportLab in `routers/document.py`) are either executed within async thread pools (`starlette.concurrency.run_in_threadpool`) or handled within standard sync endpoint definitions that FastAPI automatically runs in external worker threads.

---

### Q7: Why use Pydantic `BaseModel` for request validation instead of raw dicts?
**Answer:**  
Pydantic guarantees strict type safety, automatic type casting, custom regex constraints (e.g., matching `^\d{4}$` for MPINs), detailed automatic HTTP 422 error responses on bad data, and seamless OpenAPI schema generation.

---

### Q8: How is PDF statement generation implemented in `routers/document.py`?
**Answer:**  
`document.py` uses the **ReportLab** library to dynamically generate PDF statement documents in memory using `BytesIO`. It queries transaction history from `TransactionTable`, formats a visual tabular statement with header logos, and streams the PDF buffer directly to the browser as an attachment (`media_type="application/pdf"`).

---

### Q9: How do you handle environment configurations safely?
**Answer:**  
API keys (`GEMINI_API_KEY`, `GROQ_API_KEY`) and database URLs are loaded from a `.env` file using `python-dotenv`. `.env` is explicitly included in `.gitignore` to prevent secret leakage into version control. `.env.template` is committed as a developer blueprint.

---

### Q10: How do you serve swagger API documentation in production?
**Answer:**  
FastAPI automatically generates interactive OpenAPI documentation at `/docs` (Swagger UI) and `/redoc`. In production, these endpoints can be disabled by setting `docs_url=None` and `redoc_url=None` during `FastAPI()` instantiation.

---

## 2. Multi-Agent Systems & Generative AI - LangGraph & Gemini

### Q11: Why did you choose a Multi-Agent architecture with LangGraph instead of a single prompt?
**Answer:**  
Single prompts suffer from **prompt pollution**, **tool misfires**, and **context bloat** when dealing with complex domains. LangGraph allows us to split responsibilities into specialized agent nodes (*Account Specialist*, *FD Specialist*, *Support Specialist*). Each node has a targeted system prompt and a specific subset of tools, improving precision and reducing token usage.

---

### Q12: Walk me through the state flow of your LangGraph system in `agents_graph.py`.
**Answer:**  
1. **User Request**: User sends input to `/api/chat`.
2. **Intent Classification**: Input hits `classify_intent_heuristically` (local regex check). If recognized, it routes directly to the target Specialist Node; otherwise, it hits the `router_llm` node.
3. **Agent Execution**: The assigned Specialist Node evaluates the message. If data is needed, it returns a `tool_calls` payload.
4. **Tool Execution Node**: Executes the Python tool against SQLite, wraps the result in a `ToolMessage`, and returns control to the agent.
5. **Final Response**: Agent synthesizes the tool output into a localized natural language response.

---

### Q13: How did you optimize routing latency in your LangGraph workflow?
**Answer:**  
Calling an LLM router on every turn adds 500ms–1.5s of latency. We implemented a **Local Heuristic Intent Classifier** (`classify_intent_heuristically`). Fast regex patterns check incoming queries for obvious domain keywords (e.g. *"balance"*, *"transfer"* $\rightarrow$ Account Agent; *"fixed deposit"* $\rightarrow$ FD Agent). Matching queries bypass the Router LLM entirely, reducing latency by **up to 60%**.

---

### Q14: How are Python functions bound to LLM agents as tools?
**Answer:**  
In `assistant.py`, tools like `get_balance`, `get_transactions`, `send_money`, and `get_fd_details` are decorated with `@tool` and equipped with detailed docstrings and Pydantic schemas. They are passed to the Gemini model via `.bind_tools()`, allowing Gemini to return structured tool invocation parameters.

---

### Q15: What happens if an LLM tool invocation fails due to bad arguments?
**Answer:**  
`tool_execution_node` catches exceptions during function execution, converts the exception message into an error string within a `ToolMessage`, and returns it to the agent. The agent then politely informs the user of the missing or invalid parameters.

---

### Q16: How do you prevent LLM hallucination during financial tool execution?
**Answer:**  
1. Tools do not take loose string instructions; they require strict typed arguments validated by Pydantic.
2. Tools query the relational database directly (`models.py`) to fetch verified balances and transactions.
3. The LLM only formats the retrieved database records into text; it never performs mathematical calculations itself.

---

### Q17: What is the purpose of the `chat_histories` dictionary in `routers/chat.py`?
**Answer:**  
`chat_histories` is an in-memory session store mapping `username` to conversation history lists. This maintains multi-turn context (e.g., remembering previous questions) across REST API calls without sending full conversation state from the frontend client.

---

### Q18: How does NidhiVani handle Gemini API rate limits or quota errors (HTTP 429)?
**Answer:**  
We implemented a **Resilient Simulation Fallback System** in `routers/chat.py`. If Gemini raises a `ResourceExhausted` or connection exception, the catch block activates `simulate_fallback_chat()`. This uses local intent matching and direct database SQL queries to return localized text without crashing the app.

---

### Q19: Why use Groq as a secondary option alongside Google Gemini?
**Answer:**  
Groq provides ultra-fast LLM inference (using Llama models) with near-instantaneous token generation speeds (~500 tokens/sec). Having Groq configured allows seamless switching between Gemini and Groq depending on quota limits and speed requirements.

---

### Q20: How do you pass user identity (session context) securely to LangGraph tools?
**Answer:**  
The authenticated `username` is extracted from the backend session/token state and injected directly into the LangGraph state input payload (`{"messages": [...], "username": current_user}`). Tools access `state["username"]` to ensure users can only query their own account data.

---

## 3. Computer Vision & Facial Recognition Inference

### Q21: Explain the two-stage Face Authentication pipeline in `inference.py`.
**Answer:**  
1. **Face Detection (YOLOv8)**: An ONNX-quantized `YOLOv8-Face` model (`YOLOFaceDetector`) detects faces in live webcam snapshots, crops the facial bounding box, and removes background clutter.
2. **Embedding Extraction (Wide ResNet-101)**: The cropped face is passed through a `Wide_ResNet101_2` deep convolutional network, projecting facial geometry into a **512-dimensional normalized embedding vector**.

---

### Q22: How is similarity computed between live scans and enrolled faces?
**Answer:**  
We compute **Cosine Similarity** between the live 512-D vector ($V_{live}$) and stored base64 enrolled vector ($V_{stored}$):
$$\text{Similarity} = \frac{V_{live} \cdot V_{stored}}{\|V_{live}\| \|V_{stored}\|}$$
If the score exceeds our calibrated threshold (`0.35`), identity is verified.

---

### Q23: What is the difference between 1:1 Verification and 1:N Identification in your app?
**Answer:**  
- **1:1 Verification**: Triggered when a username is provided. The system fetches the stored facial vector for *that specific user* and compares it directly against the live scan.
- **1:N Identification (Touchless Login)**: Triggered when no username is specified. The backend iterates through *all registered user face profiles*, computes similarity scores, and automatically logs in the highest matching profile above the threshold.

---

### Q24: Why did you use ONNX Runtime instead of raw PyTorch for YOLOv8 face detection?
**Answer:**  
ONNX (Open Neural Network Exchange) Runtime optimizes model graphs, reduces memory footprint, and executes CPU/GPU inference significantly faster than running raw PyTorch `.pt` models in production.

---

### Q25: How are facial images stored in the database?
**Answer:**  
Facial enrollment snapshots are converted to base64-encoded strings and stored in the `UserTable.face_image` field in SQLite. This eliminates external file dependency issues and keeps user profiles self-contained.

---

### Q26: How do you handle lighting or minor pose variations in face recognition?
**Answer:**  
Wide ResNet-101 is pre-trained on diverse facial datasets, learning deep structural features (eye spacing, nose bridge geometry) invariant to minor lighting variations. Cropping via YOLOv8 ensures the model only processes normalized facial crops.

---

### Q27: How do you prevent face matching errors when multiple faces appear in a frame?
**Answer:**  
The YOLOv8 face detector returns bounding boxes sorted by confidence and area. The system isolates the largest central face in the frame, ignoring background faces.

---

### Q28: What is the computational complexity of your current 1:N face identification, and how would you scale it?
**Answer:**  
Current complexity is $O(N \cdot D)$ where $N$ is total users and $D=512$ is vector dimensions. At scale, this would be migrated to a **Vector DB (e.g., Milvus / pgvector)** using HNSW indexing to reduce search time to $O(\log N)$.

---

## 4. Voice Biometrics, Audio & Multilingual NLP

### Q29: How does Voice Authentication work in NidhiVani?
**Answer:**  
Voice Authentication operates on a hybrid biometric & passphrase strategy:
1. **Passphrase Verification**: Transcribed speech is validated using a tri-tier check (exact match, word set overlap $\ge 70\%$, character sequence similarity $\ge 70\%$).
2. **Generative Speaker Verification**: If an enrolled voice audio file exists in `voices/`, the raw audio blob is sent to Gemini asking: *"Are these two voice samples from the same person?"*

---

### Q30: How does the frontend capture and process voice input in `static/app.js`?
**Answer:**  
`app.js` uses the native browser **Web Speech API** (`SpeechRecognition` / `webkitSpeechRecognition`) for real-time speech-to-text. Simultaneously, it uses `MediaRecorder` to record raw audio blobs (`.webm`/`.wav`) for voice biometric evaluation.

---

### Q31: How do you support three languages (English, Hindi, Marathi)?
**Answer:**  
- **Frontend STT/TTS**: `app.js` updates speech recognition language (`en-IN`, `hi-IN`, `mr-IN`) dynamically based on user toggle.
- **Backend UI Localization**: System notifications use localized strings from `translations.py`.
- **Agent Prompts**: Agent nodes inject instructions (e.g., *"Respond strictly in Hindi/Devanagari script"*).

---

### Q32: How do you handle recipient names spoken in Hindi/Marathi (e.g., "बॉब") when database usernames are in English ("bob")?
**Answer:**  
In `assistant.py`, when a transfer target cannot be found by exact string matching:
1. `utils.py` applies Devanagari transliteration cleanups.
2. The system calls Gemini for **Dynamic Phonetic Recipient Resolution**, asking it to map the spoken regional name (e.g., "बॉब" or "आलिस") to candidate English database usernames (`['bob', 'alice']`).

---

### Q33: How does browser Speech Synthesis (TTS) speak Hindi/Marathi back to the user?
**Answer:**  
`app.js` configures `window.speechSynthesis` with `SpeechSynthesisUtterance`. It queries available system voices (`speechSynthesis.getVoices()`) and assigns matching regional voice profiles (e.g., `hi-IN` or `mr-IN` voices).

---

### Q34: What happens if the browser Web Speech API is unsupported (e.g., older browsers)?
**Answer:**  
`app.js` detects Web Speech API support on initialization. If unsupported, it gracefully degrades to text input mode while notifying the user to use a modern browser (Chrome/Edge).

---

### Q35: Why store user voice samples locally in `voices/` instead of database blobs?
**Answer:**  
Audio recordings can be multi-megabyte binary files. Storing them in a dedicated local directory (`voices/{username}_enrolled.webm`) keeps database queries lightweight while allowing fast filesystem streaming to audio models.

---

## 5. Security, MPIN Safeguards & Financial State Management

### Q36: How do you prevent accidental transfers caused by voice AI hallucinations?
**Answer:**  
We enforced a **Two-Phase Transaction State Machine**:
1. **Initiation Phase (`send_money` tool)**: Validates balances and recipient existence, but **does not execute transfer**. It stashes details in `pending_transfers` and requests the user's 4-digit MPIN.
2. **Execution Phase (`routers/chat.py`)**: Intercepts the subsequent user response. Only upon exact MPIN validation against `UserTable.mpin` is the database transaction committed.

---

### Q37: What happens after multiple incorrect MPIN attempts during a transfer?
**Answer:**  
To prevent brute-force attacks, the controller tracks failed attempts in state. After **3 failed MPIN attempts**, the pending transaction is forcefully erased, and the user must start over.

---

### Q38: How are passwords and MPINs stored securely in `models.py`?
**Answer:**  
Passwords and MPINs are hashed using cryptographic hashing algorithms (e.g. `passlib` / `bcrypt` or salted hashes) before storage in `UserTable`, ensuring plain-text credentials are never stored.

---

### Q39: How is identity recovery handled when users forget their credentials?
**Answer:**  
`routers/auth.py` provides a reset workflow where users must answer their registered **Security Question** (e.g., *"What is your favorite pet's name?"*). Reset is permitted only upon exact answer validation.

---

### Q40: How do you protect endpoints against SQL Injection?
**Answer:**  
All database queries use **SQLModel / SQLAlchemy ORM** parameterized queries (e.g., `session.exec(select(UserTable).where(UserTable.username == username))`). User inputs are never concatenated into raw SQL strings.

---

### Q41: How do you prevent overdrawing account balances during transfers?
**Answer:**  
The `send_money` function in `assistant.py` checks `sender.savings_balance < amount` inside a database transaction block before creating pending states or executing transfers.

---

### Q42: Are banking transactions atomic? What happens if a server crashes mid-transfer?
**Answer:**  
Yes. Database balance updates and transaction log insertions are executed within an explicit SQLModel `session.commit()` block. If an error occurs midway, `session.rollback()` reverts all changes.

---

### Q43: How do you ensure users cannot inspect or transfer money from another user's account?
**Answer:**  
All account endpoints enforce session authentication. The authenticated user identity is retrieved from session state and passed directly to queries (`WHERE username = session_user`), preventing unauthorized cross-account access.

---

## 6. Database Design, Logging, Performance & Production Scaling

### Q44: Describe the database schema and table relationships in `models.py`.
**Answer:**  
SQLModel manages three core relational tables:
1. **`UserTable`**: Primary key `username`. Stores credentials, hashed MPIN, balances, voice passphrase, security Q&A, and base64 face snapshot.
2. **`TransactionTable`**: Audit log storing `id`, `username`, `date`, `type` (Credit/Debit), `amount`, `description`, `category`.
3. **`FixedDepositTable`**: Records active deposits with `principal`, `interest_rate`, `tenure_months`, `start_date`, `maturity_date`, and `maturity_amount`.

---

### Q45: How do you track LLM API token consumption and costs?
**Answer:**  
`utils.py` contains `log_token_usage()`. Every generative call intercepts `response.usage_metadata`, extracting `input_tokens`, `output_tokens`, and `total_tokens`. Metrics are logged to console and written as JSON lines to `token_usage.log`.

---

### Q46: Why choose SQLModel over raw SQLAlchemy or Peewee?
**Answer:**  
SQLModel combines **SQLAlchemy** (power and ORM maturity) with **Pydantic** (data validation and serialization). This eliminates duplicate class definitions for database models and API schemas.

---

### Q47: How does database seeding work in `database.py`?
**Answer:**  
`database.py` contains `init_db()`. On app startup, it creates database tables if missing and seeds default test users (`alice`, `bob`, `charlie`) with initial balances, sample transactions, fixed deposits, and hashed MPINs.

---

### Q48: How would you scale the database tier from SQLite to Enterprise level?
**Answer:**  
1. Replace SQLite with **PostgreSQL**.
2. Implement **connection pooling** using pgBouncer.
3. Add **read-replicas** to offload read operations (balance checks, transaction history).
4. Use database migrations (Alembic) for schema updates.

---

### Q49: How would you transition in-memory chat session storage to cloud scale?
**Answer:**  
Move in-memory `chat_histories` and `pending_transfers` dictionaries to a **Redis** cluster. Redis provides sub-millisecond key-value lookup, distributed locking, and automatic key expiration (TTL) for session management across load-balanced API containers.

---

### Q50: If you had 3 more months to work on NidhiVani, what features would you add?
**Answer:**  
1. **Passive Liveness Detection**: Anti-spoofing facial liveness checks (eye blink / depth estimation).
2. **WebSocket Audio Streaming**: Real-time duplex audio streaming using WebSockets and Gemini Multimodal Live API.
3. **On-Device Biometrics**: Edge facial embedding generation via WASM/ONNX Web inside the browser for client-side privacy.
