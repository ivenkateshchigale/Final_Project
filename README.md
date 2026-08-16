# 🎙️ NidhiVani - Voice-Powered Multi-Lingual AI Banking Assistant

**NidhiVani** (निधिवाणी) is a next-generation voice-first, multi-lingual AI banking platform designed to make digital banking accessible to everyone. Built with **FastAPI**, **LangGraph**, **Google Gemini**, and modern web tech, NidhiVani supports voice biometrics, multi-agent conversational routing, MPIN-secured transfers, and natural voice interaction in **English**, **Hindi (हिंदी)**, and **Marathi (मराठी)**.

---

## ✨ Key Features

- 🗣️ **Voice-First Banking Interface**: Complete voice control for checking account balances, fixed deposits, transaction history, and fund transfers.
- 🌐 **Multi-Lingual Support**: Seamless localization and speech handling for English (`en-IN`), Hindi (`hi-IN`), and Marathi (`mr-IN`), including Devanagari transliteration.
- 🔐 **Biometric Voice Authentication**: Speaker verification powered by Gemini API matching voice prints recorded during enrollment.
- 🤖 **LangGraph Multi-Agent Architecture**: Intelligent request routing between specialized agent nodes (Account Specialist, FD Specialist, Transfer Specialist, Support Specialist).
- 🔒 **MPIN Secured Transfers**: Multi-step transaction workflow with MPIN verification and pending state confirmations.
- 📊 **Real-Time Banking Dashboard**: Beautiful dark glassmorphic web dashboard with live activity feeds, visual bank statements, and PDF export capabilities.
- 📈 **Token & Usage Metrics**: Built-in LLM token tracking and usage log analysis (`token_usage.log`).

---

## 🛠️ Tech Stack

- **Backend Framework**: [FastAPI](https://fastapi.tiangolo.com/) + [Uvicorn](https://www.uvicorn.org/)
- **Database & ORM**: [SQLModel](https://sqlmodel.tiangolo.com/) (SQLite / PostgreSQL support)
- **AI & Agent Orchestration**: [LangGraph](https://www.langchain.com/langgraph), [Google Gemini API](https://ai.google.dev/), [Groq](https://groq.com/)
- **Voice & Biometrics**: Web Speech API (Recognition & Synthesis), Audio Blob Recording, Gemini Audio Comparison
- **Frontend**: Vanilla HTML5, CSS3 (Glassmorphism & Micro-animations), JavaScript ES6+
- **Security & Utilities**: PyMySQL, Cryptography, Pydantic, Aiofiles, reportlab / PDF generation

---

## 📂 Project Architecture

```text
.
├── main.py                 # FastAPI application entry point, CORS, static mounting
├── database.py             # SQLModel engine setup, migrations, and database seeding
├── models.py               # SQLModel database schemas (UserTable, TransactionTable, FDTable)
├── schemas.py              # Pydantic schemas for requests and responses
├── assistant.py            # Gemini API client, LLM tool definitions, session state
├── agents_graph.py         # LangGraph multi-agent routing workflow
├── utils.py                # Devanagari transliteration, phonetic matching, logging helpers
├── translations.py         # Localization dictionaries (en-IN, hi-IN, mr-IN)
├── routers/                # API route controllers
│   ├── auth.py             # Login, register, voice enrollment & verification
│   ├── accounts.py         # Account balances, fixed deposits, user directory
│   ├── payments.py         # MPIN secured direct fund transfers
│   ├── chat.py             # Conversational agent endpoint & pending transfer state handler
│   └── document.py         # PDF bank statement generation
├── static/                 # Web Application Frontend
│   ├── index.html          # Banking dashboard UI layout
│   ├── style.css           # Glassmorphic UI design system
│   └── app.js              # Speech recognition, audio recording, API integration
├── ARCHITECTURE.md         # Detailed technical architecture document
└── FUNCTION_GUIDE.md       # Comprehensive function-level guide
```

---

## 🚀 Getting Started

### 1. Prerequisites

- **Python**: `3.10+` recommended
- **Git**
- **Google Gemini API Key** (or Groq API key depending on configured model)

---

### 2. Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/ivenkateshchigale/Final_Project.git
   cd Final_Project
   ```

2. **Create and activate a Virtual Environment:**
   - **Windows (PowerShell):**
     ```powershell
     python -m venv venv
     .\venv\Scripts\Activate.ps1
     ```
   - **Linux / macOS:**
     ```bash
     python3 -m venv venv
     source venv/bin/activate
     ```

3. **Install Dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

---

### 3. Environment Configuration

Create a `.env` file in the root directory (you can copy `.env.template` if available):

```env
GEMINI_API_KEY=your_gemini_api_key_here
GROQ_API_KEY=your_groq_api_key_here
DATABASE_URL=sqlite:///./banking.db
SECRET_KEY=your_secret_key
```

---

### 4. Running the Application

Start the FastAPI server using `main.py` or `uvicorn`:

```bash
python main.py
```
*or*
```bash
uvicorn main:app --reload --port 8001
```

Open your browser and navigate to:
- **Web App Dashboard**: `http://localhost:8001/`
- **Interactive API Docs (Swagger UI)**: `http://localhost:8001/docs`

---

## 🔒 Security & Privacy Notice

- **`.env`** file containing secret keys is strictly ignored in `.gitignore`.
- Database files (`*.db`) and recorded user voice prints (`voices/`) are kept local and excluded from version control.

---

## 📜 License

This project is released under the **MIT License**.
