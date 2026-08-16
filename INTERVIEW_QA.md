# NidhiVani Voice Banking Assistant - Comprehensive Interview Q&A Guide

This document contains a structured set of technical, architectural, and scenario-based interview questions and detailed answers about **NidhiVani**—a multilingual, multi-agent AI voice and biometric banking platform.

---

## 📑 Table of Contents
1. [Project Overview & Core Architecture](#1-project-overview--core-architecture)
2. [Multimodal Biometric Authentication (Face & Voice AI)](#2-multimodal-biometric-authentication-face--voice-ai)
3. [Multi-Agent System & Generative AI (LangGraph & Gemini)](#3-multi-agent-system--generative-ai-langgraph--gemini)
4. [Security, Transaction Safeguards & State Management](#4-security-transaction-safeguards--state-management)
5. [Multilingual Speech & Phonetic Resolution](#5-multilingual-speech--phonetic-resolution)
6. [Database Design, Logging & Performance](#6-database-design-logging--performance)
7. [System Scaling, Trade-offs & Challenging Scenarios](#7-system-scaling-trade-offs--challenging-scenarios)

---

## 1. Project Overview & Core Architecture

### Q1: Can you give an elevator pitch for NidhiVani? What problem does it solve?
**Answer:**  
NidhiVani is an end-to-end AI-powered voice and biometric banking assistant designed to make digital banking seamless and accessible, especially for users who prefer hands-free voice commands or regional languages (English, Hindi, and Marathi). It integrates **multimodal biometric authentication** (face recognition and voice print matching) with a **multi-agent LLM framework (LangGraph & Gemini)**. It enables users to check balances, review transactions, manage fixed deposits, and execute secure peer-to-peer transfers using natural voice in their native language, secured by 4-digit MPIN validation.

---

### Q2: Walk me through the overall system architecture of NidhiVani.
**Answer:**  
NidhiVani is built as a modern full-stack web application:
- **Frontend Layer**: Built with clean Vanilla HTML5, CSS3 (glassmorphic theme), and JS (`app.js`). It utilizes the native Browser **Web Speech API** (`SpeechRecognition` and `SpeechSynthesis`) for speech-to-text and text-to-speech, and HTML5 `<canvas>` for webcam capture.
- **API Router Layer (FastAPI)**: Modular backend controllers split into domain routes (`routers/auth.py`, `routers/chat.py`, `routers/accounts.py`, `routers/payments.py`, `routers/document.py`).
- **Multi-Agent Conversational Graph (`agents_graph.py`)**: Built on **LangGraph**. It consists of a Receptionist/Router Node, Specialist Agents (Account Specialist, Fixed Deposit Specialist, General Support Specialist), and a central Tool Execution Node.
- **Biometric Inference Engine (`inference.py`)**: A local deep learning pipeline combining **YOLOv8** (face detection) and a fine-tuned **Wide ResNet-101** (512-dim face embeddings) running via PyTorch and ONNX Runtime.
- **Database & ORM (`database.py`, `models.py`)**: Built using **SQLModel** (SQLAlchemy + Pydantic) on SQLite for structured relational storage (users, transactions, fixed deposits).

---

### Q3: Why did you choose FastAPI over traditional Python web frameworks like Django or Flask?
**Answer:**  
FastAPI was selected for three primary reasons:
1. **Asynchronous (async/await) Native Support**: Handles non-blocking asynchronous I/O efficiently, crucial when coordinating generative AI APIs (Gemini) and AI inference models.
2. **Pydantic Validation & Speed**: Automatic HTTP request body validation (`schemas.py`) and high performance built on Starlette and Pydantic.
3. **Automatic OpenAPI / Swagger Documentation**: Simplifies testing biometrics and multi-agent payload endpoints directly during development.

---

## 2. Multimodal Biometric Authentication (Face & Voice AI)

### Q4: How is Face Authentication implemented in NidhiVani?
**Answer:**  
Face Authentication ([routers/auth.py](file:///d:/final%20project%201/routers/auth.py#L286-L370) & [inference.py](file:///d:/final%20project%201/inference.py)) uses a two-stage computer vision deep learning pipeline:
1. **Face Detection (YOLOv8)**: We use an ONNX-quantized `YOLOv8-Face` model (`YOLOFaceDetector`) to locate faces in live frames, crop the facial region, and strip background noise.
2. **Embedding Extraction (Wide ResNet-101)**: The cropped face is fed into a `Wide_ResNet101_2` deep convolutional network that projects the facial geometry into a **512-dimensional normalized feature vector**.
3. **Similarity Verification**: We compute **Cosine Similarity** between the live embedding vector and the database enrolled embedding. If the cosine similarity score exceeds `0.35`, the face is verified.

---

### Q5: What is the difference between 1:1 Verification and 1:N Identification in your face authentication module?
**Answer:**  
- **1:1 Verification (Targeted Match)**: Triggered when a user provides their username along with a live snapshot. The system fetches the stored facial snapshot for that specific user and compares the live scan directly against it.
- **1:N Identification (Touchless Login)**: Triggered when no username is specified. The system queries all users with registered face scans from SQLite, computes cosine similarity against every profile, and automatically identifies and logs in the highest matching user profile above the threshold (`0.35`).

---

### Q6: How does Voice Authentication work in NidhiVani?
**Answer:**  
Voice Authentication ([routers/auth.py](file:///d:/final%20project%201/routers/auth.py#L122-L176)) operates on a hybrid biometric & passphrase strategy:
1. **Fuzzy Passphrase Similarity (`check_passphrase_similarity`)**: Cleans the transcribed spoken text and validates it against the user's stored passphrase using a tri-tier matching check:
   - Direct Substring match
   - Jaccard Word Set overlap ($\ge 70\%$)
   - Character Sequence similarity using `difflib.SequenceMatcher` ($\ge 70\%$)
2. **Generative Speaker Verification**: If an enrolled voice audio file (`.webm`/`.wav`) exists in `voices/`, the backend sends both the raw audio file and the live audio snapshot to Gemini (`gemini-3.6-flash`) with a speaker comparison prompt asking: *"Are these two voice samples from the same person?"*

---

## 3. Multi-Agent System & Generative AI (LangGraph & Gemini)

### Q7: Why did you use a Multi-Agent architecture with LangGraph instead of a single LLM system prompt?
**Answer:**  
Single LLMs with monolithic prompts often suffer from **prompt ambiguity, tool misfires, and context pollution** in complex domains like banking.  
By using **LangGraph**, we decoupled responsibilities:
- **Router Node**: Acts as a receptionist to classify user intent.
- **Account Specialist Agent**: Strictly handles balance checks, transactions, and transfers.
- **Fixed Deposit Specialist Agent**: Specialized in FD rates, calculator estimations, and maturity rules.
- **General Support Specialist**: Answers general banking queries and FAQs.

This division of labor improves precision, reduces prompt token usage, and isolates agent tools to prevent execution errors.

---

### Q8: How did you optimize conversational latency in your Multi-Agent Graph?
**Answer:**  
Calling an LLM router on every turn adds $500\text{ms} - 1.5\text{s}$ of latency. To solve this, we implemented a **Local Heuristic Intent Classifier** (`classify_intent_heuristically` in [agents_graph.py](file:///d:/final%20project%201/agents_graph.py)):
- Fast regular expressions and keyword checks inspect incoming user queries for obvious domain patterns (e.g. *"balance"*, *"statement"*, *"send money"* $\rightarrow$ Account Agent; *"fixed deposit"*, *"FD rate"* $\rightarrow$ FD Agent; *"hello"*, *"namaste"* $\rightarrow$ Support Agent).
- **LLM Bypass**: If a match is found locally, the system bypasses the Router LLM call entirely, cutting response latency by up to **$60\%$**. If uncertain, it falls back gracefully to the `router_llm` node.

---

### Q9: How do your LLM Agents invoke backend Python tools safely?
**Answer:**  
We define structured Python tools in `assistant.py` (`get_balance`, `get_transactions`, `send_money`, `get_fd_details`) bound to Gemini model schemas.  
When an agent determines a tool call is required:
1. The agent LLM returns a structured `tool_calls` payload instead of final text.
2. LangGraph routes the execution to `tool_execution_node` ([agents_graph.py](file:///d:/final%20project%201/agents_graph.py)).
3. The node executes the native Python function against the SQLite database using `sqlmodel`.
4. The output is wrapped in a `ToolMessage` and passed back to the agent node so the LLM can synthesize a natural language response in the user's requested language.

---

### Q10: How does NidhiVani handle LLM API rate limits, timeouts, or quota exceeded (HTTP 429) errors in production?
**Answer:**  
We implemented a **Resilient Simulation Fallback System** in [routers/chat.py](file:///d:/final%20project%201/routers/chat.py):
- If the Gemini API or LangGraph raises an exception (such as `ResourceExhausted` / `429` quota limits), the catch block activates `simulate_fallback_chat()`.
- The simulation engine uses local regex-based intent classification, directly queries the database via standard SQL helper functions, and constructs localized responses in English, Hindi, or Marathi without crashing or locking out the user.

---

## 4. Security, Transaction Safeguards & State Management

### Q11: Voice interfaces can accidentally trigger transfers. How did you secure money transfers against voice hallucination or unauthorized speech?
**Answer:**  
We implemented a **Two-Phase State Machine with MPIN Verification**:
1. **Initiation Phase (`send_money` tool)**: When a user says *"Send ₹500 to Bob"*, the tool validates account balances and recipient existence, but **does not transfer money**. Instead, it stashes transaction details in a server-side `pending_transfers` state map and responds: *"Please confirm transfer of ₹500 to Bob by stating your 4-digit MPIN"*.
2. **Execution Phase (`routers/chat.py`)**: On the subsequent turn, `chat_endpoint` checks if the user is in `pending_transfers`.
   - If the user says *"cancel"*, the state is cleared and aborted.
   - If a 4-digit MPIN is spoken or typed, it is verified against `user.mpin` in the database. Only upon an exact match is the database transaction committed.
   - **Lockout Mechanism**: After 3 failed MPIN attempts, the pending transfer is forcefully erased to prevent brute-force attacks.

---

### Q12: How are password, voice, and face resets handled securely?
**Answer:**  
For resetting credentials ([routers/auth.py](file:///d:/final%20project%201/routers/auth.py)):
- Users must answer their registered **Security Question** (e.g. *"What is your favorite pet's name?"*).
- For legacy accounts, fallback mappings ensure legacy seeded accounts remain recoverable.
- Only upon validating `entered_answer == db_security_answer` does the API update the password, re-enroll the base64 face image, or save a new speaker print file in `voices/`.

---

### Q13: How do you prevent SQL Injection and data leakage?
**Answer:**  
- **SQLModel / SQLAlchemy ORM**: All database interactions use parameterized queries via SQLModel sessions (`select(UserTable).where(...)`), completely insulating the system against SQL injection.
- **Pydantic Validation**: All incoming requests are validated against strict type constraints (e.g., regex validation matching `^\d{4}$` for 4-digit MPINs).

---

## 5. Multilingual Speech & Phonetic Resolution

### Q14: How does NidhiVani support multiple Indian languages (English, Hindi, Marathi)?
**Answer:**  
- **Frontend Speech Processing**: `app.js` sets the Web Speech API recognition language (`en-IN`, `hi-IN`, `mr-IN`) dynamically based on user selection.
- **Backend Localization (`translations.py`)**: Contains dictionary translations for all UI system notifications and fallback simulation prompts.
- **LLM System Prompt Conditioning**: Each agent node (`account_agent_node`, `fd_agent_node`) dynamically injects language rules into the system prompt (e.g., *"Respond strictly in Hindi / Devanagari script"*).

---

### Q15: How do you handle names spoken in Hindi or Marathi (e.g. "बॉब" or "आलिस") when searching for English database usernames?
**Answer:**  
In `assistant.py`, when a user requests a money transfer in Hindi/Marathi:
1. The system cleans and transliterates Devanagari characters using `utils.py` helpers.
2. If exact username lookup fails, it invokes Gemini to perform **Dynamic Phonetic Recipient Resolution**:
   - Prompt: *"Given the database users ['alice', 'bob', 'charlie'], match the spoken Hindi/Marathi recipient name 'बॉब'. Respond with the exact matching database username."*
3. The LLM resolves `"बॉब"` to `"bob"`, enabling seamless cross-lingual transactions.

---

## 6. Database Design, Logging & Performance

### Q16: Describe your Database Schema design.
**Answer:**  
We designed three relational tables in SQLModel ([models.py](file:///d:/final%20project%201/models.py)):
1. `UserTable`: Primary key `username` (string), stores hashed passwords, full names, 4-digit `mpin`, `savings_balance`, `checking_balance`, `voice_passphrase`, `security_question`, `security_answer`, and base64 encoded `face_image`.
2. `TransactionTable`: Stores audit trail records (`id`, `username`, `date`, `type` [Credit/Debit], `amount`, `description`, `category`).
3. `FixedDepositTable`: Tracks active deposits (`id`, `username`, `principal`, `interest_rate`, `tenure_months`, `start_date`, `maturity_date`, `maturity_amount`).

---

### Q17: How do you monitor LLM token usage and cost metrics?
**Answer:**  
We created a centralized tracking utility `log_token_usage` ([utils.py](file:///d:/final%20project%201/utils.py)):
- Every generative call across LangGraph nodes or recipient resolution functions intercepts `response.usage_metadata`.
- It records `input_tokens`, `output_tokens`, and `total_tokens`.
- It logs formatted metrics to the console and appends structured JSON lines to `token_usage.log` for cost analysis and auditability.

---

## 7. System Scaling, Trade-offs & Challenging Scenarios

### Q18: What was the most challenging technical hurdle in building NidhiVani, and how did you solve it?
**Answer:**  
*Sample Answer*:  
> "The biggest challenge was achieving **low-latency multilingual intent routing while ensuring strict transaction safety**. Initially, routing every chat query through an LLM router caused a noticeable delay of over 1.5 seconds. Additionally, early tests showed LLMs could occasionally trigger money transfer tools directly without user confirmation.  
> 
> We solved this with a two-part architectural pattern:
> 1. **Heuristic Latency Optimization**: Added a local regex heuristic classifier that bypassed the LLM router for clear queries, dropping routing latency to near $0\text{ms}$.
> 2. **Pending Transfer State Machine**: Decoupled transfer initiation from execution by requiring a mandatory 4-digit MPIN step managed directly by the FastAPI controller, guaranteeing zero accidental transfers."

---

### Q19: How would you scale NidhiVani to support 1 Million concurrent users in a production cloud environment?
**Answer:**  
To scale NidhiVani for production:
1. **Database Tier**: Migrate from SQLite to **PostgreSQL** with read-replicas and connection pooling (pgBouncer).
2. **Biometric Vector Indexing**: Move facial embedding matching from inline Python loops to a dedicated Vector Database like **Milvus** or **pgvector** using HNSW indexing for $O(\log N)$ 1:N face search.
3. **Async Task Queues**: Offload heavy AI inference (YOLOv8 + Wide ResNet face scans and Gemini audio evaluations) to worker nodes using **Celery / Redis / RabbitMQ**.
4. **Caching Layer**: Cache frequent user balances, fixed deposit rates, and conversation history in **Redis**.
5. **Stateful Chat Sessions**: Store multi-agent session state in Redis or DynamoDB instead of in-memory dictionaries (`chat_histories`).

---

### Q20: What are the key takeaways or future improvements planned for NidhiVani?
**Answer:**  
- **Passive Liveness Detection**: Adding anti-spoofing facial liveness checks (e.g. eye blink detection, depth estimation) to prevent photo/screen playback attacks.
- **Streaming Voice (WebSockets)**: Migrating from REST polling to WebSockets with streaming TTS/STT (e.g., ElevenLabs / Gemini Multimodal Live API) for real-time natural conversational cadence.
- **On-Device Edge Models**: Running face embeddings locally on client devices via WebAssembly (WASM) / ONNX Web for faster and privacy-centric biometric matching.

---

*Document compiled for NidhiVani Voice Banking Assistant project interviews.*
