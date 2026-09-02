# 🎯 150 Technical Interview Cross-Questions & Candidate Answers: NidhiVani AI Voice Banking Platform

This document is a comprehensive interview preparation blueprint containing **150 technical cross-examination questions** and **bulletproof answers** for **NidhiVani AI**, a voice-enabled digital banking application powered by FastAPI, LangGraph multi-agent LLM systems, Google Gemini, Web Speech API, Voice & Face Biometrics, SQLModel, and multilingual NLP.

---

## 📑 Table of Contents
1. [System Architecture, Web Framework & High-Concurrency Async Design (Q1 – Q15)](#1-system-architecture-web-framework--high-concurrency-async-design)
2. [LangGraph Multi-Agent Architecture, State Graph & Agent Routing (Q16 – Q30)](#2-langgraph-multi-agent-architecture-state-graph--agent-routing)
3. [Google Gemini LLM Integration, Prompt Engineering & Function/Tool Calling (Q31 – Q45)](#3-google-gemini-llm-integration-prompt-engineering--functiontool-calling)
4. [Voice Processing, Web Speech API & HTML5 Audio MediaRecorder Pipeline (Q46 – Q60)](#4-voice-processing-web-speech-api--html5-audio-mediarecorder-pipeline)
5. [Computer Vision, Face Biometrics & Facial Recognition Pipeline (Q61 – Q75)](#5-computer-vision-face-biometrics--facial-recognition-pipeline)
6. [Voice Biometrics, Audio Analysis & Passphrase Authentication (Q76 – Q90)](#6-voice-biometrics-audio-analysis--passphrase-authentication)
7. [Multilingual Processing, Devanagari Transliteration & Spoken Digit Normalization (Q91 – Q105)](#7-multilingual-processing-devanagari-transliteration--spoken-digit-normalization)
8. [Financial Security, 4-Digit MPIN Logic, 2FA & Session Guards (Q106 – Q120)](#8-financial-security-4-digit-mpin-logic-2fa--session-guards)
9. [Database Schemas, SQLModel ORM, Financial Transactions & State Consistency (Q121 – Q135)](#9-database-schemas-sqlmodel-orm-financial-transactions--state-consistency)
10. [Frontend Architecture, PDF Generation, Unit Testing & Cloud Deployment (Q136 – Q150)](#10-frontend-architecture-pdf-generation-unit-testing--cloud-deployment)

---

## 1. System Architecture, Web Framework & High-Concurrency Async Design

### Q1: Why did you select FastAPI over traditional Python web frameworks like Django or Flask for NidhiVani?
**Answer:**  
FastAPI was selected for three primary architectural requirements:
1. **Native Asynchronous (async/await) Capabilities**: High-concurrency async non-blocking execution is essential when orchestrating external generative AI LLM requests (Gemini API), voice streams, and deep learning vision inference.
2. **Strict Pydantic Schema Validation**: Automatic request payload validation (e.g., MPIN regex, Base64 strings, language codes) eliminates manual validation boilerplate.
3. **High Performance**: Built on Starlette and Uvicorn (ASGI), FastAPI offers execution speed comparable to Node.js and Go.

### Q2: How is the codebase structured for modularity and separation of concerns?
**Answer:**  
The repository follows a clean domain-driven module structure:
- `main.py`: Application entry point, initializing FastAPI, mounting StaticFiles, CORS middleware, and routing modules.
- `routers/`: Decoupled APIRouter instances split by domain (`auth.py`, `chat.py`, `accounts.py`, `payments.py`, `document.py`).
- `agents_graph.py`: LangGraph state machine, intent classification heuristics, and agent nodes.
- `assistant.py`: LLM client initialization, prompt templates, Python database tools, and ContextVar state trackers.
- `inference.py`: Isolated computer vision module for OpenCV/PyTorch facial detection.
- `models.py` & `database.py`: SQLModel ORM schemas and thread-safe database session generators.

### Q3: What is the purpose of ContextVars in `assistant.py` (`current_user_var`, `last_queried_transactions`)?
**Answer:**  
`ContextVar` provides thread-local and task-local isolation across asynchronous execution contexts in FastAPI. Because multiple HTTP requests execute concurrently in the event loop, using simple global variables would cause race conditions (e.g., User A overwriting User B's session username). `ContextVar` ensures each request thread tracks its own context safely.

### Q4: How does FastAPI handle static asset serving and SPA routing for `index.html`?
**Answer:**  
FastAPI mounts the `static/` directory using `app.mount("/static", StaticFiles(directory="static"), name="static")`. For the root endpoint `/`, a custom HTMLResponse returns `static/index.html`. The frontend JavaScript SPA consumes JSON endpoints exposed under `/api/*`.

### Q5: How do you handle Cross-Origin Resource Sharing (CORS) in `main.py`?
**Answer:**  
We attach `CORSMiddleware` to the FastAPI instance, configuring `allow_origins`, `allow_credentials=True`, `allow_methods=["*"]`, and `allow_headers=["*"]`. This guarantees secure browser API requests while blocking unauthorized cross-site scripting attempts.

### Q6: How do heavy blocking CPU-bound tasks (like image processing or PDF parsing) avoid blocking FastAPI’s single-threaded event loop?
**Answer:**  
Synchronous CPU-heavy tasks are executed via standard `def` (synchronous) endpoint routes or offloaded using `starlette.concurrency.run_in_threadpool`. FastAPI automatically delegates synchronous endpoint functions to a background thread pool, keeping the main ASGI loop responsive.

### Q7: Why use SQLModel instead of traditional SQLAlchemy ORM or raw SQL queries?
**Answer:**  
SQLModel bridges Pydantic and SQLAlchemy into a unified type annotation system. It allows a single model class definition (in `models.py`) to function as both a Pydantic schema for API serialization and a database table model for ORM queries, preventing model duplication.

### Q8: What is the role of Uvicorn in running NidhiVani AI?
**Answer:**  
Uvicorn is a lightning-fast ASGI (Asynchronous Server Gateway Interface) web server built on `uvloop` and `httptools`. It serves as the HTTP web server interface that receives incoming requests, handles TLS/SSL, and routes async requests to FastAPI.

### Q9: How do you ensure environment variables and API keys remain secure in dev and prod?
**Answer:**  
We utilize `python-dotenv` to load `.env` variables at runtime. `.env` is listed in `.gitignore` to prevent committing secrets to source control. A `.env.template` is committed as a reference guide. In production, secrets are injected via container environment variables.

### Q10: How do you gracefully capture unhandled exceptions in API routes?
**Answer:**  
Each router uses standard `try-except` blocks wrapping critical domain logic. When an exception occurs, HTTP standard status codes (400 Bad Request, 401 Unauthorized, 404 Not Found, 500 Internal Server Error) are returned via `HTTPException(status_code=..., detail=...)`.

### Q11: Explain how `/api/chat` returns formatted chat state back to the UI.
**Answer:**  
When `/api/chat` finishes processing through LangGraph, it returns a unified JSON dictionary:
- `response`: The natural language string response from the agent.
- `state`: Freshly fetched account balances and transactions via `get_user_current_state()`.
- `pending_transfer`: Boolean flag indicating if an MPIN is required.
- `logout`: Optional boolean flag triggering client side session invalidation on security lockouts.

### Q12: How do you prevent thread starvation in SQLite when executing multiple concurrent write transactions?
**Answer:**  
SQLite uses file-level/database-level locking. To prevent database locks during concurrent FastAPI requests, we configure `connect_args={"check_same_thread": False}` and use short-lived `Session` scopes via FastAPI's `Depends(get_db_session)` dependency injector.

### Q13: How does `document.py` handle PDF banking statement generation?
**Answer:**  
`routers/document.py` uses `ReportLab` to construct PDF documents in-memory inside a `io.BytesIO` buffer. It formats transactions into visual auto-tables, adds custom corporate styling, and streams the PDF buffer back as `StreamingResponse(buffer, media_type="application/pdf")`.

### Q14: What is the advantage of using Pydantic’s `Field(default_factory=...)` in database models?
**Answer:**  
`Field(default_factory=...)` evaluates dynamic values (such as timestamps via `datetime.now`) at runtime when a new record instance is instantiated, avoiding hardcoded static class load time values.

### Q15: How can NidhiVani AI scale horizontally across multiple container instances?
**Answer:**  
Because FastAPI endpoints are stateless, multiple Uvicorn worker containers can sit behind a load balancer (Nginx/Traefik). Session state is stored in SQLite (or scalable PostgreSQL), and LLM/Biometric operations execute statelessly per request.

---

## 2. LangGraph Multi-Agent Architecture, State Graph & Agent Routing

### Q16: Why did you choose a Multi-Agent architecture using LangGraph instead of a single prompt monolithic LLM?
**Answer:**  
A single prompt LLM suffers from **prompt pollution**, **tool misfires**, and **context token bloat** when handling multi-domain logic. LangGraph decomposes the system into specialized agent nodes (*Account Specialist*, *FD Specialist*, *Support Specialist*). Each node carries a tailored prompt system and a minimal subset of relevant tools, increasing execution precision and lowering token consumption.

### Q17: Walk me through the exact state lifecycle in `agents_graph.py`.
**Answer:**  
1. **Input Payload**: The router accepts `messages`, `db_user_name`, and `language_name`.
2. **Heuristic Classification**: `classify_intent_heuristically()` scans the user message. If matching keywords/regex exist, it bypasses LLM classification and directly assigns `next_agent`.
3. **Router Node**: If heuristics fail, `router_llm` evaluates the intent and outputs the routing decision.
4. **Specialist Node**: The designated agent node (`account_agent`, `fd_agent`, `support_agent`) runs with its bound tools.
5. **Tool Execution Node**: If the LLM generates a tool call, `tools_node` executes the Python function against SQLite, appends a `ToolMessage`, and returns control to the specialist agent.
6. **END Node**: Returns final `AIMessage` content.

### Q18: What is `AgentState` in `agents_graph.py` and how is it defined?
**Answer:**  
`AgentState` is a `TypedDict` defining the graph state schema:
```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    next_agent: str
    active_agent_name: str
    db_user_name: str
    language_name: str
```
`Annotated[..., add_messages]` is a LangGraph reducer that automatically appends new messages to conversation history instead of overwriting the array.

### Q19: Why do you combine Heuristic Regex intent matching with LLM routing in `agents_graph.py`?
**Answer:**  
Deterministic heuristic matching handles common patterns (e.g., "check balance", "transfer money", "hi") with **zero LLM latency** and **zero API cost**. LLM routing acts as a fallback for complex or ambiguous natural language inputs.

### Q20: How does `prune_chat_history()` work in `routers/chat.py`?
**Answer:**  
`prune_chat_history()` prevents context window overflow by capping history length (default max 20 messages). It ensures that pruning cuts off at a `HumanMessage` boundary so conversation turn structure remains intact.

### Q21: How are tools bound to specific agents in LangGraph?
**Answer:**  
We use `.bind_tools()` on specific LLM instances:
- `account_llm_with_tools = account_llm.bind_tools([get_transaction_history, send_money, get_balance])`
- `fd_llm_with_tools = fd_llm.bind_tools([get_fixed_deposit_details])`
This ensures the Account Agent cannot accidentally call FD tools, and vice versa.

### Q22: What happens if an agent node produces an invalid tool call schema?
**Answer:**  
LangGraph's tool node catches the invocation schema error, formats an error string inside a `ToolMessage`, and returns it back to the agent node so the LLM can self-correct or notify the user.

### Q23: How does NidhiVani ensure agent responses remain in the user's requested spoken language?
**Answer:**  
Each specialist agent node dynamically injects a language instruction into its system prompt: `"IMPORTANT: Respond ONLY in {language_name}."`.

### Q24: What is the purpose of the conditional edge `should_continue` in `agents_graph.py`?
**Answer:**  
`should_continue` checks if the last message in `AgentState` contains `tool_calls`. If yes, it routes to `"tools"`; if no, it routes to `END`.

### Q25: How does the system prevent infinite tool execution loops in LangGraph?
**Answer:**  
LangGraph includes a built-in `recursion_limit` (default 25 steps). Additionally, our tool design ensures functions return concrete data payloads in a single turn.

### Q26: How does `send_money` tool handle ambiguous recipient names?
**Answer:**  
If a user asks to "send ₹200 to John" and multiple users named John exist, `send_money` returns a JSON structure containing `pending_recipient_resolution: True` and a list of `possible_recipients`. The API router then prompts the user to clarify.

### Q27: How is context passed between multiple sub-agent transitions?
**Answer:**  
Because all agents read from and write to the unified `AgentState["messages"]` list, any node can inspect prior messages and tool outputs from preceding steps.

### Q28: Why do you set `temperature=0.0` for the Router LLM and `temperature=0.2` for Specialist Agents?
**Answer:**  
- `temperature=0.0` ensures strictly deterministic, repeatable routing decisions.
- `temperature=0.2` provides natural linguistic variation in responses while remaining factual and non-hallucinatory.

### Q29: How does NidhiVani handle offline/simulation fallback when the LLM API is unavailable?
**Answer:**  
`routers/chat.py` wraps `agents_graph.invoke()` in a `try-except` block. If an API exception or rate limit error occurs, `fallback_to_simulation` triggers local rule-based intent parsing via `get_simulation_message()`.

### Q30: How do you log token consumption across agent invocations?
**Answer:**  
`utils.py` contains `log_token_usage(prompt_tokens, completion_tokens, model_name)`, which logs structured usage records to `token_usage.log` for cost tracking and performance monitoring.

---

## 3. Google Gemini LLM Integration, Prompt Engineering & Function/Tool Calling

### Q31: Which specific Google Gemini model version is utilized in NidhiVani AI and why?
**Answer:**  
We utilize **Gemini 3.6 Flash** (`gemini-3.6-flash`). It offers low latency, high throughput function calling capabilities, native multilingual understanding (English, Hindi, Marathi), and cost efficiency.

### Q32: How is the Gemini API client initialized in `assistant.py`?
**Answer:**  
We instantiate the client using Google's new unified SDK:
```python
from google import genai
client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
```
For LangGraph integration, we use `ChatGoogleGenerativeAI(model="gemini-3.6-flash", google_api_key=api_key)`.

### Q33: How does Gemini handle Python functions as schema-defined tools?
**Answer:**  
LangChain automatically inspects Python function signatures, docstrings, and type hints using `pydantic` reflection, transforming them into OpenAPI-compliant JSON function schemas sent to Gemini's `tools` API parameter.

### Q34: What techniques are used in system prompts to minimize LLM hallucinations in financial operations?
**Answer:**  
1. **Strict Context Grounding**: Prompts explicitly state: *"You must ONLY answer using information provided by tool outputs. Do NOT fabricate transaction records or account balances."*
2. **Explicit Fallback Directives**: *"If data is unavailable, state clearly that you cannot find the record."*
3. **Structured System Instruction Injections**.

### Q35: How does the `send_money` tool intercept financial transfers for MPIN validation?
**Answer:**  
`send_money` does **not** instantly execute the DB transaction. Instead, it staging-logs the request into `pending_transfers[username] = {...}` and returns a prompt instructing the user to supply their 4-digit MPIN.

### Q36: How does `get_transaction_history` filter transactions dynamically?
**Answer:**  
`get_transaction_history(limit, category, transaction_type)` accepts optional filtering parameters. It executes a SQLModel `select(TransactionTable)` query with dynamic `.where()` clauses and stores the resulting objects in `last_queried_transactions.set(tx_list)`.

### Q37: How do you ensure Gemini parses Indian numbering formats (e.g., Lakhs/Crores) correctly?
**Answer:**  
The system prompts instruct Gemini to format monetary values using Indian numbering formats (`₹50,000.00`). In Python, strings are parsed using regex to extract clean floating-point numbers.

### Q38: What happens if Gemini returns a tool call with missing required arguments?
**Answer:**  
LangGraph's tool executor node returns a validation error `ToolMessage` specifying missing arguments. Gemini receives this message in the next turn and re-issues the tool call with corrected parameters.

### Q39: How is multi-turn conversation memory maintained across HTTP requests?
**Answer:**  
Conversation histories are persisted per user in `chat_histories` dictionary in `assistant.py`. On each request, history is passed into `agents_graph.invoke()`, updated with new turns, and pruned.

### Q40: How do you handle Gemini API quota limits (HTTP 429)?
**Answer:**  
`ChatGoogleGenerativeAI` is instantiated with `max_retries=1`. If retries fail, `routers/chat.py` catches `GoogleAPIError` and seamlessly shifts the user session into local rule-based simulation mode.

### Q41: How do system prompts enforce security boundaries regarding user account switching?
**Answer:**  
`current_user_var` injects the verified session username into every tool call. Even if a user speaks *"Show Alice's balance"*, the system prompt and tool logic enforce filtering strictly by `current_user_var.get()`.

### Q42: What is the benefit of using structured JSON outputs in tool responses?
**Answer:**  
Structured JSON returns unambiguous key-value pairs (e.g., `{"status": "success", "new_balance": 45000.00}`), preventing LLM interpretation ambiguities.

### Q43: How does Gemini support Marathi and Hindi language queries?
**Answer:**  
Gemini 3.6 Flash is natively pre-trained on broad multilingual corpora. It understands queries written in Devanagari script (`हिंदी`, `मराठी`) as well as Latin script transliterations ("Hinglish" / "Marathi in Roman script").

### Q44: Why are banking tools defined in `assistant.py` decorated with `@tool`?
**Answer:**  
LangChain's `@tool` decorator transforms standard Python functions into runnable `BaseTool` objects containing metadata, argument definitions, and schemas required for LLM tool binding.

### Q45: How do you prevent prompt injection attacks where a user tries to override system rules?
**Answer:**  
System messages are prepended at the highest priority level in `AgentState["messages"]`. Furthermore, actual database operations require explicit server-side state confirmation (like MPIN check), isolating database integrity from prompt manipulations.

---

## 4. Voice Processing, Web Speech API & HTML5 Audio MediaRecorder Pipeline

### Q46: How is speech-to-text (STT) achieved in the frontend without external paid APIs?
**Answer:**  
We utilize the browser's native **Web Speech API** (`window.SpeechRecognition` / `webkitSpeechRecognition`). It streams audio directly to the browser's native speech engine and emits real-time text transcripts via the `onresult` event handler.

### Q47: How does `startAudioRecording()` in `static/app.js` handle cross-browser MIME type compatibility?
**Answer:**  
`startAudioRecording()` queries `MediaRecorder.isTypeSupported()` in order of preference:
1. `audio/webm;codecs=opus` (Chrome/Edge)
2. `audio/webm`
3. `audio/ogg;codecs=opus` (Firefox)
4. `audio/mp4` (Safari)
This guarantees smooth audio recording across all major web browsers.

### Q48: How is recorded audio converted from browser memory to the server payload?
**Answer:**  
Recorded PCM chunks are collected in an `audioChunks` array. On recording stop, a `Blob` is constructed and converted into a Base64-encoded string via `FileReader.readAsDataURL(audioBlob)` before being sent over JSON.

### Q49: How does `speakText()` implement Text-to-Speech (TTS) in `static/app.js`?
**Answer:**  
`speakText()` uses `window.speechSynthesis.speak(utterance)` with `SpeechSynthesisUtterance`. It dynamically filters out tabular markdown data (so raw tables are not read out loud) and selects regional voice accents based on `selectedLanguage`.

### Q50: How do you handle ambient background noise during speech recognition?
**Answer:**  
The browser Web Speech API employs native hardware acoustic echo cancellation (AEC) and noise suppression. Additionally, `recognition.continuous = false` ensures recognition closes automatically when the user finishes speaking a sentence.

### Q51: What happens if a browser does not support `webkitSpeechRecognition`?
**Answer:**  
`static/app.js` detects missing API support on load, displays a user banner (*"Speech recognition not supported in this browser"*), disables microphone buttons, and defaults the interface to typed text inputs.

### Q52: How do you handle audio recording stream cleanup to prevent memory leaks?
**Answer:**  
When `stopAudioRecording()` completes, it iterates over all tracks in `mediaRecorder.stream.getTracks()` and explicitly calls `track.stop()`, releasing the user's hardware microphone resource.

### Q53: Why do you capture BOTH text transcription AND raw audio binary during Voice Login?
**Answer:**  
- **Text Transcript**: Contains the spoken passphrase text evaluated for linguistic correctness.
- **Raw Audio Binary**: Contains acoustic spectral biometric features used for voice print verification and enrolled storage.

### Q54: How does `recognition.maxAlternatives = 1` optimize speech recognition?
**Answer:**  
It instructs the recognition engine to return only the top-ranked confidence transcript hypothesis, reducing memory consumption and parsing complexity.

### Q55: How do you handle speech recognition network disconnects mid-sentence?
**Answer:**  
The `recognition.onerror` handler catches errors (e.g., `network`, `no-speech`, `audio-capture`), updates UI status indicators, and provides visual tap-to-retry cues.

### Q56: How is user audio privacy protected during speech recording?
**Answer:**  
Microphone streams are activated strictly on explicit user tap gestures (`micBtn` click). Stream tracks are immediately terminated upon recording completion.

### Q57: How does `static/app.js` manage UI microphone states during active listening?
**Answer:**  
CSS classes (`recording`, `listening`) are dynamically added/removed from mic wrappers, toggling glowing pulse animations and status text ("Listening... Speak now").

### Q58: Why is `window.speechSynthesis.cancel()` called before starting new speech recognition?
**Answer:**  
Calling `.cancel()` stops active TTS audio output immediately so speaker output does not feedback into microphone input during recording.

### Q59: How is multi-language TTS voice selection implemented?
**Answer:**  
`window.speechSynthesis.getVoices()` is filtered by language locale code (`hi-IN`, `mr-IN`, `en-IN`), selecting Google or system native regional voice profiles.

### Q60: What is the maximum duration for recorded voice audio payloads?
**Answer:**  
Voice passphrases and commands are short phrases (typically 2 to 6 seconds), keeping Base64 payload sizes small (~50KB – 200KB) for lightweight network transmission.

---

## 5. Computer Vision, Face Biometrics & Facial Recognition Pipeline

### Q61: What computer vision libraries and models are integrated into `inference.py`?
**Answer:**  
`inference.py` utilizes **OpenCV (`cv2`)** for image decoding and image processing, alongside **PyTorch / Torchvision** deep learning models (or lightweight Haar-Cascade / HOG detectors) for facial detection and feature extraction.

### Q62: How are facial images transmitted from the web browser to `/api/face-login`?
**Answer:**  
The frontend captures a snapshot from an HTML5 `<video>` element to an HTML5 `<canvas>`, extracts a JPEG data URI (`canvas.toDataURL('image/jpeg')`), strips the Data URI prefix, and posts the Base64 string to FastAPI.

### Q63: How does `inference.py` decode Base64 image strings into OpenCV images?
**Answer:**  
```python
image_bytes = base64.b64decode(base64_string)
np_array = np.frombuffer(image_bytes, np.uint8)
cv_image = cv2.imdecode(np_array, cv2.IMREAD_COLOR)
```

### Q64: What preprocessing steps are applied to facial images before recognition matching?
**Answer:**  
1. Conversion to Grayscale (`cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)`).
2. Histogram Equalization (`cv2.equalizeHist()`) to normalize lighting variations.
3. Resizing to standard dimensions (e.g., 128x128 pixels).

### Q65: How does NidhiVani evaluate face verification matching in `test_face_login.py`?
**Answer:**  
`test_face_login.py` measures feature distance (e.g., Mean Squared Error or Cosine Distance over normalized pixel matrices/embeddings) between candidate login snapshots and stored user enrollment snapshots (`user.face_image`).

### Q66: How do you prevent face spoofing (e.g., holding up a static photo to the camera)?
**Answer:**  
1. **Liveness Detection**: Require dynamic micro-gestures (blinking, head turn).
2. **2FA Enforcement**: Face login acts as primary auth, followed by 4-digit MPIN validation for high-risk operations.

### Q67: What happens if no face is detected in the uploaded frame?
**Answer:**  
`inference.py` returns a failure result (`"No face detected in snapshot"`). The endpoint returns HTTP 400, prompting the user to center their face in lighting.

### Q68: How do you handle low-light or over-exposed camera conditions?
**Answer:**  
Histogram equalization enhances image contrast. If image variance falls below quality thresholds, the system requests camera adjustment.

### Q69: Why store face snapshots as Base64 strings in `UserTable.face_image` instead of raw files?
**Answer:**  
Storing Base64 strings directly in SQLite allows single-transaction atomicity and simple database migrations without managing separate file system assets.

### Q70: How is webcam video captured in `static/app.js`?
**Answer:**  
`navigator.mediaDevices.getUserMedia({ video: { width: 640, height: 480 } })` attaches the live stream to `#face-login-video.srcObject`.

### Q71: How do you release webcam resources when switching tabs or closing modals?
**Answer:**  
`stream.getTracks().forEach(track => track.stop())` terminates camera hardware activity, turning off camera indicator LEDs.

### Q72: How does facial recognition inference scale under high API concurrency?
**Answer:**  
Vision processing is CPU/GPU intensive. In production, inference routines are offloaded to dedicated worker processes (Celery/Redis Queue) or GPU inference containers.

### Q73: What is the face match threshold used in NidhiVani biometric verification?
**Answer:**  
The verification metric uses a normalized similarity threshold (e.g., Cosine Similarity > 0.85 or Euclidean Distance < 0.40) tuned to balance False Accept Rate (FAR) and False Reject Rate (FRR).

### Q74: How do you test facial login end-to-end?
**Answer:**  
`test_face_login.py` simulates API requests with synthetic test images, validating endpoint responses, success flags, and error boundary behaviors.

### Q75: Can a user register both face biometrics and voice passphrases?
**Answer:**  
Yes, `RegisterRequest` accepts both `face_image_base64` and `voice_audio_base64`, enabling Multi-Modal Biometric Authentication.

---

## 6. Voice Biometrics, Audio Analysis & Passphrase Authentication

### Q76: How does NidhiVani store enrolled voice audio prints?
**Answer:**  
When a user registers, `routers/auth.py` decodes `voice_audio_base64` and writes the audio file to `voices/{username}_enroll.{ext}` (supporting `.webm`, `.wav`, `.ogg`).

### Q77: How does `check_passphrase_similarity()` in `utils.py` verify spoken passphrases?
**Answer:**  
`check_passphrase_similarity()` normalizes string inputs (lowercasing, punctuation stripping) and computes similarity using character/word n-gram ratios (`difflib.SequenceMatcher`).

### Q78: Why is exact string matching insufficient for voice passphrase login?
**Answer:**  
Speech-to-text engines introduce slight transcription variances (e.g., "open sesame" vs "open sesami"). Fuzzy similarity matching allows valid logins while filtering incorrect passphrases.

### Q79: How does `voice_login_endpoint` handle username-less voice login?
**Answer:**  
If a user clicks voice login without typing a username, the system queries all registered users, evaluates passphrase similarity against every enrolled passphrase, and identifies the best match (`best_score > threshold`).

### Q80: What audio formats are accepted for voice enrollment and login?
**Answer:**  
The backend inspects `voice_audio_mime` and supports WebM Opus (`audio/webm`), Ogg Opus (`audio/ogg`), WAV (`audio/wav`), and MP4 AAC (`audio/mp4`).

### Q81: How do you test voice login programmatically?
**Answer:**  
`test_voice_login.py` sends synthetic audio requests and passphrase parameters to `/api/voice-login`, validating verification success logic.

### Q82: What is a voice passphrase enrollment badge in the UI?
**Answer:**  
In `static/index.html`, `#voice-enrollment-badge` provides visual confirmation ("Voice print captured successfully!") when dictation completes.

### Q83: How do you prevent replay attacks using recorded voice passphrase audio?
**Answer:**  
1. **Dynamic Passphrases**: Require users to speak dynamic challenge phrases.
2. **Biometric 2FA**: Mandatory MPIN entry for money transfers.

### Q84: What acoustic features can be extracted from enrolled voice files for deep voice biometrics?
**Answer:**  
Enrolled `.wav` files can be processed using `librosa` or `torchaudio` to extract Mel-Frequency Cepstral Coefficients (MFCCs), pitch contours, and speaker embeddings (e.g., ECAPA-TDNN).

### Q85: How do you handle ambient acoustic noise in recorded voice files?
**Answer:**  
Audio bytes can pass through bandpass filtering (80Hz – 3400Hz) and spectral subtraction algorithms to strip static noise.

### Q86: What happens if a user forgets their voice passphrase?
**Answer:**  
The UI provides a *"Reset Voice Passphrase"* workflow. The user answers security questions to clear their old passphrase and re-enroll a new voice print.

### Q87: How does `difflib.SequenceMatcher` calculate string similarity ratios?
**Answer:**  
It uses the Gestalt Pattern Matching algorithm, computing $2M / (T1 + T2)$ where $M$ is the number of matching characters and $T1, T2$ are total string lengths.

### Q88: Why is `re.sub(r"[^\w\s]", "", text)` used in passphrase normalization?
**Answer:**  
It removes punctuation marks (periods, commas, question marks) added by STT engines, leaving clean alphanumeric tokens for accurate comparison.

### Q89: How do you ensure high performance when matching a voice passphrase against thousands of database users?
**Answer:**  
Passphrase strings can be indexed in database tables or cached in memory (Redis), reducing matching lookup times.

### Q90: Can a user log in with traditional credentials if voice recognition fails?
**Answer:**  
Yes, the UI supports credential login (username + password), face login, and voice login simultaneously.

---

## 7. Multilingual Processing, Devanagari Transliteration & Spoken Digit Normalization

### Q91: What languages are natively supported by NidhiVani AI?
**Answer:**  
NidhiVani natively supports **English (`en-IN`)**, **Hindi (`hi-IN`)**, and **Marathi (`mr-IN`)**.

### Q92: How does `transliterate_devanagari()` in `utils.py` handle Indian language inputs?
**Answer:**  
`transliterate_devanagari()` maps Devanagari characters (`अ-ह`, numbers `०-९`) to Latin/English phonetic equivalents, enabling unified intent processing regardless of script used.

### Q93: How does `normalize_spoken_digits()` handle multi-script number speech?
**Answer:**  
`normalize_spoken_digits()` converts spoken digit words in English ("one two three four"), Hindi ("एक दो तीन चार"), and Marathi ("एक दोन तीन चार"), as well as Devanagari numerals (`१२३४`), into standard ASCII digits (`1234`).

### Q94: Why is spoken digit normalization critical for MPIN entry?
**Answer:**  
Users speaking their MPIN might trigger STT outputs like *"four five six seven"* or *"चार पांच छह सात"*. `normalize_spoken_digits()` converts these to `"4567"` for pattern matching.

### Q95: How is language switching implemented in `static/app.js`?
**Answer:**  
A language selector dropdown updates `selectedLanguage` in `localStorage`. `translateUI()` dynamically updates DOM labels and placeholders from a `translations` dictionary object.

### Q96: How does `translations.py` support server-side response localization?
**Answer:**  
`translations.py` contains dictionaries for application responses. `get_translated_message(key, language, **kwargs)` formats localized strings dynamically based on request locale.

### Q97: What is "Hinglish" / "Code-Mixing" and how does NidhiVani process it?
**Answer:**  
Code-mixing is mixing English and Indian words (e.g., *"Mera savings balance kitna hai"*). Intent heuristics and Gemini LLM prompts handle code-mixed inputs natively.

### Q98: How do you normalize regional dialect variations in spoken commands?
**Answer:**  
Heuristic phrase sets incorporate regional synonyms (e.g., Marathi *"पैसे पाठवा"*, *"पैसे ट्रान्सफर"*, *"पैसे पाठवायचे आहेत"*).

### Q99: How does the system ensure speech synthesis (TTS) speaks in the correct accent?
**Answer:**  
`speakText()` filters available browser voices (`window.speechSynthesis.getVoices()`) for language locale prefixes (`hi`, `mr`, `en`), ensuring appropriate pronunciation.

### Q100: How do you format currency displays across different locales?
**Answer:**  
JavaScript's `toLocaleString('en-IN')` formats currency values with appropriate comma placement (`₹1,50,000.00`).

### Q101: How do you handle cases where Devanagari transliteration yields ambiguous word spellings?
**Answer:**  
Fuzzy string matching algorithms and LLM context understanding resolve phonetic ambiguities.

### Q102: How are security question options localized in `static/app.js`?
**Answer:**  
Security question text elements are assigned translation dictionary keys mapped during `translateUI()` execution.

### Q103: What happens if a user submits a query in an unsupported language (e.g., French)?
**Answer:**  
Gemini LLM understands the query and defaults to responding politely in English, prompting the user to select a supported banking language.

### Q104: How do you test multilingual translation coverage?
**Answer:**  
`translations.py` includes validation tests asserting that all message keys exist across `en-IN`, `hi-IN`, and `mr-IN` dictionaries.

### Q105: Why use Unicode character ranges `\u0900-\u097F` in regex expressions?
**Answer:**  
`\u0900-\u097F` defines the official Unicode block for Devanagari script, enabling precise regex tokenization of Hindi and Marathi text.

---

## 8. Financial Security, 4-Digit MPIN Logic, 2FA & Session Guards

### Q106: What is an MPIN and how is it enforced in NidhiVani AI?
**Answer:**  
An MPIN is a mandatory 4-digit Personal Identification Number required for authorizing financial transactions (transfers, bill payments).

### Q107: Walk me through the multi-turn pending transfer authorization flow.
**Answer:**  
1. User requests money transfer ("Send ₹500 to Alice").
2. `send_money` tool creates a pending transfer state in `pending_transfers[username]`.
3. The server prompts the user for their 4-digit MPIN.
4. User submits MPIN via speech or UI input.
5. The server validates MPIN against `UserTable.mpin`. If correct, the transaction executes; if incorrect, an error is returned.

### Q108: What happens after 3 consecutive failed MPIN attempts?
**Answer:**  
1. Pending transfer state is cleared.
2. Server returns `logout: True`.
3. Frontend triggers a 5-second visual countdown warning and executes `logoutUser()`, clearing session state.

### Q109: How is the secure MPIN entry UI modal designed in `static/index.html`?
**Answer:**  
`#mpin-input-wrapper` provides a focused, masked input interface with explicit *Submit* and *Cancel* controls, preventing accidental speech misinterpretations during financial authorization.

### Q110: Why are financial transfer operations prohibited from completing via voice commands alone without MPIN?
**Answer:**  
Voice alone is vulnerable to background speech capture and speaker misinterpretations. Mandatory 2FA via MPIN ensures explicit user intent and regulatory compliance.

### Q111: How do you prevent double-spending when a user submits rapid transfer requests?
**Answer:**  
Database updates run inside atomic SQL transactions (`session.commit()`). Balances are verified immediately before debit execution within the database lock scope.

### Q112: How are user passwords and sensitive pins protected in `models.py`?
**Answer:**  
In production, plain passwords and MPINs are hashed using salted cryptographic hashing algorithms (e.g., `bcrypt` or `argon2`) via `passlib`.

### Q113: How does NidhiVani prevent unauthorized account access via parameter tampering in API endpoints?
**Answer:**  
Endpoints verify session identity against `current_user_var`. Requests targeting other accounts without valid authorization are rejected (HTTP 403 Forbidden).

### Q114: How does session storage work in the frontend SPA?
**Answer:**  
On successful login, user session data (`username`, `name`) is stored in `sessionStorage`. On logout or window close, `sessionStorage.clear()` purges session tokens.

### Q115: What is daily transfer limit enforcement?
**Answer:**  
`send_money` evaluates the sum of debits executed in the current 24-hour period. If the total exceeds configured limits (e.g., ₹50,000), the transfer is rejected.

### Q116: How do you handle security question verification during password resets?
**Answer:**  
`/api/forgot-password/reset` verifies that `security_answer.lower().strip()` matches `user.security_answer.lower().strip()` before updating credentials.

### Q117: What security protections prevent timing attacks on password/MPIN verification?
**Answer:**  
Cryptographic comparison functions (`hmac.compare_digest`) execute in constant time, preventing execution time analysis attacks.

### Q118: How is channel tracking recorded in `TransactionTable`?
**Answer:**  
Each transaction record includes a `channel` attribute (`"Voice"`, `"Web"`, `"Mobile"`), providing audit trails for compliance reporting.

### Q119: What mechanisms prevent cross-site request forgery (CSRF)?
**Answer:**  
API calls enforce JSON content-type headers, strict CORS origin policies, and session validation tokens.

### Q120: How do you secure database connection strings and sensitive configurations?
**Answer:**  
Database URIs and credentials are set as environment variables loaded securely at startup via `python-dotenv`.

---

## 9. Database Schemas, SQLModel ORM, Financial Transactions & State Consistency

### Q121: Describe the database schema design in `models.py`.
**Answer:**  
`models.py` defines three primary SQLModel tables:
- `UserTable`: Account credentials, balances, MPIN, voice/face biometrics, and security questions.
- `TransactionTable`: Ledger records (`id`, `username`, `date`, `type`, `amount`, `description`, `category`, `channel`).
- `FixedDepositTable`: Fixed deposit records (`id`, `username`, `principal_amount`, `tenure`, `interest_rate`, `maturity_date`, `status`).

### Q122: How is `database.py` configured for FastAPI dependency injection?
**Answer:**  
`database.py` creates a SQLAlchemy engine and defines a session generator:
```python
def get_db_session():
    with Session(engine) as session:
        yield session
```
This guarantees automatic session opening and closing per HTTP request.

### Q123: Walk me through the exact SQL database operations during a money transfer.
**Answer:**  
1. `db_sender` and `db_recipient` models are queried within a single transaction.
2. Sender balance is checked (`sender_balance >= amount`).
3. Sender balance is debited; recipient balance is credited.
4. Two `TransactionTable` records are inserted (Debit for sender, Credit for recipient).
5. `session.commit()` commits both updates atomically.

### Q124: What happens if an error occurs mid-way through a transfer transaction?
**Answer:**  
The `except` block executes `session.rollback()`. This undoes all partial updates, preserving balance integrity.

### Q125: How do you seed initial database data for development and testing?
**Answer:**  
`database.py` includes `create_db_and_tables()`. Migration scripts (`migrate_db.py`, `inspect_db.py`) initialize test user accounts (Alice, Bob) with initial balances.

### Q126: Why use `session.expire_all()` after committing financial transfers?
**Answer:**  
`session.expire_all()` clears cached ORM instance attributes, forcing subsequent queries to reload fresh state directly from the database.

### Q127: How are database relationships modeled between users and transactions?
**Answer:**  
`TransactionTable.username` acts as a Foreign Key referencing `UserTable.username`.

### Q128: How do you inspect SQLite database contents programmatically?
**Answer:**  
`inspect_db.py` executes SQLModel queries and prints formatted tables of users, transactions, and fixed deposits.

### Q129: How do you migrate database schemas when adding new columns?
**Answer:**  
`migrate_db.py` uses SQLModel metadata inspection or **Alembic** migration scripts to issue `ALTER TABLE` statements without losing existing user data.

### Q130: Why is `amount` stored as a `Float` or `Decimal` in `models.py`?
**Answer:**  
Financial monetary amounts require fractional decimal representation. In production database schemas, standard `Decimal` (Numeric) types prevent floating-point rounding errors.

### Q131: How does `get_user_current_state()` construct account dashboards?
**Answer:**  
`get_user_current_state(username, session)` queries user balances, recent transaction lists, and active fixed deposits, returning a formatted JSON dictionary for UI rendering.

### Q132: What is the purpose of `session.flush()` during user registration?
**Answer:**  
`session.flush()` pushes pending ORM objects to the database transaction buffer to assign generated primary keys before final `session.commit()`.

### Q133: How do you handle database connection pooling in high-traffic deployments?
**Answer:**  
SQLAlchemy's engine is configured with `pool_size=20`, `max_overflow=10`, and `pool_recycle=3600` to manage reusable database connection pools efficiently.

### Q134: How do you ensure index optimization for transaction queries?
**Answer:**  
Indexes are added to frequently queried columns like `TransactionTable.username` and `TransactionTable.date` (`index=True`), optimizing SQL filtering performance.

### Q135: Can SQLModel be switched from SQLite to PostgreSQL without modifying application logic?
**Answer:**  
Yes. Because SQLModel abstracts database operations, updating `DATABASE_URL` in `.env` to a PostgreSQL connection string (`postgresql://user:pass@host/db`) migrates the backend seamlessly.

---

## 10. Frontend Architecture, PDF Generation, Unit Testing & Cloud Deployment

### Q136: How is the frontend UI designed in `static/index.html` and `static/style.css`?
**Answer:**  
The frontend is a single-page web application (SPA) built with Semantic HTML5, Vanilla JavaScript (`app.js`), and custom CSS3 (`style.css`). It features a modern dark-mode aesthetic with glassmorphism, responsive flex/grid layouts, FontAwesome icons, and animated status widgets.

### Q137: How does `generatePDFStatement()` work in `static/app.js`?
**Answer:**  
`generatePDFStatement()` uses **jsPDF** and **jspdf-autotable**. It extracts transaction rows, builds a styled PDF with header branding, formats text colors (green for credit, red for debit), and triggers a browser file download.

### Q138: Why does `generatePDFStatement()` submit a form instead of opening a raw Blob URL?
**Answer:**  
Some mobile browsers block dynamic Blob URL downloads. Submitting Base64 PDF data via a hidden form (`#pdf-download-form`) to `/api/download-pdf` forces native browser file download attachments reliably.

### Q139: How is automated unit testing structured in the project?
**Answer:**  
The repository includes dedicated test scripts (`test_voice_login.py`, `test_face_login.py`, `test_gemini_match.py`). Tests use FastAPI's `TestClient` and `pytest` to validate endpoints, authentication rules, and model responses.

### Q140: How do you run automated test suites with `pytest`?
**Answer:**  
Executing `pytest -v` in the root directory runs all `test_*.py` files, reporting assertion passes, coverage metrics, and execution times.

### Q141: What containerization strategy is implemented in `Dockerfile`?
**Answer:**  
`Dockerfile` builds a multi-stage container:
1. Base image: `python:3.10-slim`.
2. Installs system dependencies (`ffmpeg`, OpenCV libs).
3. Copies `requirements.txt` and installs Python packages.
4. Copies application code and exposes port 8000.
5. Entry command: `cmd ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]`.

### Q142: What is configured in `.dockerignore`?
**Answer:**  
`.dockerignore` excludes `.git`, `venv`, `__pycache__`, `.env`, and test log files, keeping Docker build contexts lightweight.

### Q143: How would you deploy NidhiVani AI to AWS or Google Cloud Platform (GCP)?
**Answer:**  
- **AWS**: Deploy Docker container to AWS ECS / Fargate behind an Application Load Balancer (ALB), with Amazon RDS PostgreSQL for database storage.
- **GCP**: Deploy container to Google Cloud Run, utilizing Google Cloud SQL (PostgreSQL) and Secret Manager for API keys.

### Q144: How is real-time transaction search implemented in `static/app.js`?
**Answer:**  
`#tx-search-input` listens to `keyup` events. It filters `activeTransactions` in memory using string matching across description, category, and type fields, re-rendering `#transactions-body` dynamically.

### Q145: How is financial data privacy protected in the dashboard?
**Answer:**  
Balance cards initialize with masked values (`₹ ••••••`). Clicking the eye toggle icon updates element text content from `dataset.realValue`.

### Q146: How do you handle frontend UI state synchronization after a successful transaction?
**Answer:**  
When `/api/chat` returns updated `state`, `updateDashboard()` updates DOM balance numbers, appends new transaction rows, and redraws analytics charts automatically.

### Q147: How are analytics charts rendered in the dashboard?
**Answer:**  
`renderAnalytics()` uses **Chart.js** to render interactive canvas visual displays:
1. Spending trend line chart over time.
2. Category spending breakdown doughnut chart.
3. Income vs. Expense bar comparison chart.

### Q148: What is the purpose of `ARCHITECTURE.md` and `FUNCTION_GUIDE.md`?
**Answer:**  
They serve as project documentation, outlining system data flows, module dependencies, API route schemas, and developer onboarding instructions.

### Q149: How do you optimize production performance for static assets?
**Answer:**  
Static files (`style.css`, `app.js`) can be served via CDN or cached with HTTP `Cache-Control` headers, while CSS/JS files can be minified to reduce bundle sizes.

### Q150: What are the key technical innovations of NidhiVani AI?
**Answer:**  
1. **Multi-Agent LangGraph Routing**: Contextual intent routing across specialized Banking Agents.
2. **Multi-Modal Biometrics**: Seamless fusion of Web Speech STT, Voice Print Passphrase matching, and OpenCV Face Recognition.
3. **Multilingual Indian NLP**: Native multi-lingual execution across English, Hindi, and Marathi with Devanagari transliteration.
4. **End-to-End Voice Banking Workflow**: Voice command recognition, 2FA MPIN authorization, transactional execution, and TTS vocal feedback.
