# PARIKSHAN_AI

> An AI-powered interview practice platform that generates dynamic questions, evaluates answers, and tracks student performance.

---

## Project Overview

**PARIKSHAN_AI** simulates realistic technical interviews by combining a structured FastAPI backend with a React frontend and an LLM-based AI evaluation engine.

Users configure an interview session by selecting a subject, difficulty, and Bloom's taxonomy level. The system generates targeted questions via an AI pipeline, collects answers from the user, and then batch-evaluates the entire session to provide a detailed performance breakdown.

---

## Tech Stack

### Frontend

- **React** (Vite)
- **React Router** — client-side routing
- **Axios** — HTTP client for API calls
- **Google OAuth** (`@react-oauth/google`) — social login
- **TailwindCSS** — utility-first styling

### Backend

- **FastAPI** — async Python web framework
- **SQLAlchemy** — ORM for database interaction
- **Alembic** — database migration management
- **PostgreSQL** — relational database
- **JWT (python-jose)** — stateless authentication
- **Slowapi** — rate limiting
- **Google Auth Library** — OAuth2 token verification

### AI Services

- **Groq API** — LLM inference for question generation and answer evaluation
- **Pinecone** — vector embeddings for semantic similarity
- `AI_MOCK_MODE=true` — local development mode (bypasses real AI calls)

---

## System Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      FRONTEND (React)                    │
│  InterviewConfig → InterviewScreen → Results             │
└──────────────────────┬───────────────────────────────────┘
                       │ HTTP (JSON / JWT)
┌──────────────────────▼───────────────────────────────────┐
│                    BACKEND (FastAPI)                     │
│  Routes → Controllers → Services → Repositories → DB    │
└───────────┬──────────────────────────────────────────────┘
            │ LLM API calls
┌───────────▼──────────────────────────────────────────────┐
│              AI MODULE (Groq / Pinecone)                 │
│  Question Generator | Answer Evaluator | Embeddings      │
└──────────────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────┐
│                POSTGRESQL DATABASE                       │
│  Users | Sessions | Questions | Answers                  │
└──────────────────────────────────────────────────────────┘
```

---

## Repository Structure

The repository is split into four top-level folders, each owned by a different team. Every folder is a self-contained Python package or Node project.

```
central-submissions/
├── frontend/               # React (Vite) single-page application
├── backend/                # FastAPI REST API server
├── ai/                     # AI services module (Groq + Pinecone)
└── database/               # SQLAlchemy models, repositories, and Alembic migrations
```

---

### `frontend/` — React SPA

The user-facing application built with React + Vite.

```
frontend/
├── index.html                  # Vite HTML entry point
├── vite.config.js              # Vite bundler configuration
├── package.json                # npm dependencies and scripts
├── eslint.config.js            # ESLint rules
├── .env                        # Frontend environment variables (VITE_GOOGLE_CLIENT_ID)
└── src/
    ├── main.jsx                # React tree root — renders <App /> into the DOM
    ├── App.jsx                 # Router root — defines all client-side routes:
    │                           #   /          → LoginPage
    │                           #   /signup    → SignupPage
    │                           #   /config    → InterviewConfig  (protected)
    │                           #   /interview → InterviewScreen  (protected)
    ├── App.css / index.css     # Global and component-level styles (Tailwind base)
    │
    ├── pages/                  # Full-screen page components
    │   ├── LoginPage.jsx       # Email/password login form + Google OAuth button
    │   ├── SignupPage.jsx       # New-account registration form
    │   ├── InterviewConfig.jsx # Pre-interview setup form (subject, difficulty,
    │   │                       #   Bloom level, question count); calls POST /interview/start
    │   └── InterviewScreen.jsx # Live interview UI — shows one question at a time,
    │                           #   accepts typed answers, and displays the final results
    │                           #   panel with score ring, performance level, and breakdown
    │
    ├── Components/             # Shared, reusable UI components
    │   ├── ProtectedRoute.jsx  # HOC that redirects unauthenticated users to /
    │   └── QuestionCard.jsx    # Renders a single interview question with its timer
    │
    ├── hooks/                  # Custom React hooks (stateful logic, no JSX)
    │   ├── useAuth.js          # Convenience wrapper — reads user/token from AuthContext
    │   └── useInterview.js     # Full interview state machine:
    │                           #   start(), fetchNextQuestion(), answer(),
    │                           #   fetchSummary(), fetchResult(), reset()
    │
    ├── api/
    │   └── api.js              # Central Axios client (baseURL = http://127.0.0.1:8000/api/v1)
    │                           # Intercepts every request to attach Authorization: Bearer <token>
    │                           # Intercepts 401 responses to clear storage and redirect to /
    │                           # Exports: signup, login, googleAuth,
    │                           #          startInterview, getNextQuestion,
    │                           #          submitAnswer, getSummary, getResult
    │
    ├── context/
    │   └── AuthContext.jsx     # React Context that holds user, token, loading, error
    │                           # Persists token to localStorage across page reloads
    │                           # Exposes: login(), signup(), googleAuth(), logout()
    │
    └── assets/                 # Static images (logo.png, bg.png, react.svg)
```

---

### `backend/` — FastAPI Server

The Python REST API. Follows a strict four-layer architecture:
**Route → Controller → Service → Repository**.

```
backend/
├── requirements.txt            # Python dependencies
├── .env.example                # Template for environment variables
├── test_ai.py                  # Manual smoke test for the AI module
├── tests/                      # Pytest test suite (package)
├── docs/
│   └── API.md                  # Detailed HTTP contract reference
└── app/
    ├── main.py                 # FastAPI app factory:
    │                           #   • Creates the FastAPI app instance
    │                           #   • Registers CORS middleware (localhost:3000 / 5173)
    │                           #   • Attaches SlowAPI rate-limit middleware
    │                           #   • Registers the global validation error handler
    │                           #   • Mounts api_router under /api/v1
    │                           #   • Runs Base.metadata.create_all() on startup
    │
    ├── api/
    │   ├── router.py           # Aggregates all sub-routers into one APIRouter:
    │   │                       #   /auth       → auth_routes
    │   │                       #   /interview  → interview_routes
    │   │                       #   /users      → user_routes
    │   ├── routes/             # HTTP layer only — no business logic
    │   │   ├── auth_routes.py      # POST /signup, POST /login, POST /google
    │   │   ├── interview_routes.py # POST /start, GET /{id}/next,
    │   │   │                       # POST /{id}/answer, GET /{id}/summary,
    │   │   │                       # GET /{id}/result
    │   │   └── user_routes.py      # GET /users/me  (profile endpoints)
    │   └── v1/                 # Reserved namespace for future API versioning
    │
    ├── controllers/            # Orchestration layer — calls a service, wraps result
    │   ├── auth_controller.py      # handle_signup / handle_login / handle_google_login
    │   ├── interview_controller.py # handle_start_interview / handle_get_next_question /
    │   │                           # handle_submit_answer / handle_get_summary / handle_get_result
    │   └── user_controller.py      # handle_get_me
    │
    ├── services/               # Business logic — validation, orchestration, AI calls
    │   ├── auth_service.py         # signup: hash password, create user, issue JWT
    │   │                           # login: verify password, issue JWT
    │   │                           # google_login: verify Google ID token, upsert user, issue JWT
    │   ├── interview_service.py    # start_interview: create session → call AI → persist questions
    │   │                           # get_next_question: find first unanswered InterviewQuestion
    │   │                           # submit_answer: save answer, auto-complete session if last
    │   │                           # get_summary: batch-evaluate all answers via AI, return breakdown
    │   │                           # get_result: return simple score/total/percentage
    │   └── user_service.py         # get_current_user_profile
    │
    ├── schemas/                # Pydantic models for request/response validation
    │   ├── auth_schema.py          # SignupRequest, LoginRequest, GoogleLoginRequest
    │   ├── interview_schema.py     # InterviewStartRequest, SubmitAnswerRequest
    │   ├── interview.py            # Shared interview response shapes
    │   ├── user_schema.py          # UserResponse
    │   └── user.py                 # Shared user shapes
    │
    ├── core/                   # Cross-cutting concerns
    │   ├── config.py           # Settings loaded from .env via Pydantic BaseSettings
    │   │                       #   (PROJECT_NAME, DATABASE_URL, SECRET_KEY, etc.)
    │   ├── security.py         # Password hashing (bcrypt) and JWT creation/decoding
    │   ├── dependencies.py     # get_current_user() — FastAPI Dependency that decodes
    │   │                       #   the Bearer token and returns the user_id (int)
    │   ├── auth.py             # Backward-compat shim — re-exports get_current_user
    │   ├── google_auth.py      # Verifies Google ID tokens via google-auth library
    │   ├── rate_limit.py       # Shared SlowAPI Limiter instance (key = client IP)
    │   ├── logging.py          # Configures structured JSON logging at app startup
    │   └── responses.py        # Typed response model helpers
    │
    └── utils/
        └── response.py         # success_response(data) — wraps any payload in the
                                #   standard { success, data } envelope
```

---

### `ai/` — AI Services Module

An independent Python package. The backend imports from here; it has no knowledge of FastAPI or SQLAlchemy.

```
ai/
├── __init__.py
├── config.py               # AISettings — reads GROQ_API_KEY and PINECONE_API_KEY
│                           # from backend/.env (or project-root .env) via pydantic-settings
├── schemas/
│   └── ai_schema.py        # Pydantic models for AI input/output shapes
└── services/
    ├── llm_service.py          # Thin wrapper around the Groq SDK — sends a chat
    │                           # completion request to llama-3.3-70b-versatile and
    │                           # returns the raw text response
    ├── question_generator.py   # generate_questions(payload, student_id):
    │                           #   1. Queries Pinecone for the student's past interactions
    │                           #   2. Builds a system + user prompt including subject,
    │                           #      difficulty, Bloom level, and memory context
    │                           #   3. Calls Groq to get a JSON list of questions
    │                           #   4. Upserts each question into Pinecone memory
    ├── check_answers.py        # check_answer_correctness(question, answer, student_id):
    │                           #   1. Queries Pinecone for related memory
    │                           #   2. Sends question + answer to Groq for evaluation
    │                           #   3. Parses the JSON score (0-100), explanation, feedback
    │                           #   4. Classifies level: Weak / Average / Strong / Excellent
    │                           #   5. Upserts the interaction into Pinecone memory
    ├── embedding.py            # Converts text to vector embeddings (used by Pinecone)
    ├── pinecone_service.py     # query_embeddings() / upsert_embeddings():
    │                           #   manages the student's long-term vector memory
    │                           #   (one namespace per student_id)
    └── mock_ai.py              # Returns hardcoded questions/scores when
                                #   AI_MOCK_MODE=true (for local dev without API keys)
```

---

### `database/` — Data Layer

Contains all SQLAlchemy models, repositories, and Alembic migration tooling. Shared by both `backend/` and `ai/`.

```
database/
├── __init__.py
├── base.py                 # Imports all models so that Base.metadata knows about
│                           # every table (required for Alembic auto-generation)
├── base_class.py           # Declares the SQLAlchemy DeclarativeBase used by all models
├── config.py               # Reads DATABASE_URL from environment
├── session.py              # Creates the SQLAlchemy engine and SessionLocal factory
│                           # Also exposes get_db() — the FastAPI dependency that
│                           # yields a DB session per request and closes it afterwards
│
├── models/                 # ORM table definitions
│   ├── user_model.py       # User — id, name, email, password_hash, role,
│   │                       #   auth_provider, google_id, profile_picture, is_active
│   ├── interview_model.py  # InterviewSession — user_id, subject_id, mode, difficulty,
│   │                       #   bloom_strategy, status, num_questions_requested/generated
│   │                       # InterviewQuestion (junction) — links a session to a Question,
│   │                       #   stores sequence_number and bloom_level_at_time
│   │                       # Answer — links to InterviewQuestion, stores answer_text,
│   │                       #   evaluation_score, feedback, ai_evaluation_metadata (JSONB)
│   └── question_model.py   # Question — question_text, bloom_level, difficulty,
│                           #   topic_tags (JSONB array), source_type (AI_GENERATED)
│
├── repositories/           # All SQL queries live here — no business logic
│   ├── interview_repository.py # InterviewRepository singleton:
│   │                           #   create_session / update_session_status / get_session_by_id
│   │                           #   create_question / create_session_question_link
│   │                           #   get_next_unanswered_question / get_question_link
│   │                           #   save_answer / get_answers_for_session / get_answer_for_question
│   └── user_repository.py      # UserRepository singleton:
│                               #   create_user / get_user_by_email / get_user_by_id /
│                               #   get_user_by_google_id
│
└── migrations/             # Alembic migration tooling
    ├── alembic.ini         # Alembic configuration (script_location, DB URL pointer)
    ├── env.py              # Migration environment — imports Base metadata, runs migrations
    └── script.py.mako      # Template for auto-generated migration scripts
```

---

## Detailed Request Flow

This section traces every step a request takes through the system, from user action in the browser to database read/write and back.

### 1. Authentication Flow

```
Browser (LoginPage.jsx)
  │  user fills email + password, clicks "Login"
  │
  ▼
useAuth() / AuthContext.login()
  │  calls api.js → POST /api/v1/auth/login { email, password }
  │  Axios interceptor attaches no token (unauthenticated endpoint)
  │
  ▼
FastAPI  auth_routes.py  POST /auth/login
  │  rate-limited: 5 requests / minute per IP
  │  validates LoginRequest schema (Pydantic)
  │
  ▼
auth_controller.handle_login(db, payload)
  │  delegates to auth_service.login()
  │  wraps result in success_response({ token, user })
  │
  ▼
auth_service.login(db, payload)
  │  calls user_repository.get_user_by_email()
  │  verifies bcrypt password hash (security.py)
  │  calls security.create_access_token() → signs JWT with SECRET_KEY
  │  returns { token, user }
  │
  ▼
HTTP 200 { success: true, data: { token: "...", user: {...} } }
  │
  ▼
AuthContext.login() stores token in localStorage
App.jsx navigates to /config
```

> Google OAuth follows the same path except the frontend sends the Google ID token to
> `POST /auth/google` and `google_auth.py` verifies it before upserting the user.

---

### 2. Start Interview Flow

```
Browser (InterviewConfig.jsx)
  │  user selects subject, difficulty, Bloom level, question count
  │  clicks "Start Interview"
  │
  ▼
useInterview.start(payload)
  │  calls api.js → POST /api/v1/interview/start { subject, mode, difficulty, … }
  │  Axios interceptor attaches Authorization: Bearer <token>
  │
  ▼
FastAPI  interview_routes.py  POST /interview/start
  │  rate-limited: 5 requests / minute per IP
  │  validates InterviewStartRequest schema
  │  get_current_user() dependency decodes JWT → extracts user_id (int)
  │
  ▼
interview_controller.handle_start_interview(db, payload, user_id)
  │
  ▼
interview_service.start_interview(db, payload, user_id)
  │  1. user_repository.get_user_by_id() — verifies user exists
  │  2. interview_repository.create_session() — inserts InterviewSession row (status="initializing")
  │  3. Runs generate_questions() in a threadpool (non-blocking):
  │     │
  │     ▼
  │    ai/services/question_generator.generate_questions(payload, student_id)
  │     │  a. pinecone_service.query_embeddings() — fetches student's past interaction memory
  │     │  b. Builds system + user prompt (subject, difficulty, Bloom, memory context)
  │     │  c. groq_client.chat.completions.create() — LLM generates N questions as JSON
  │     │  d. pinecone_service.upsert_embeddings() — stores each question in student memory
  │     │  returns { questions: [ { question_text, bloom_level, difficulty, … } ] }
  │     │
  │  4. For each generated question:
  │     a. interview_repository.create_question() — inserts Question row
  │     b. interview_repository.create_session_question_link() — inserts InterviewQuestion row
  │  5. interview_repository.update_session_status() — sets status="active"
  │  6. db.commit()
  │  returns { session_id, questions: [...] }
  │
  ▼
HTTP 201 { session_id: 42, questions: [...] }
  │
  ▼
useInterview stores session_id; App.jsx navigates to /interview
```

---

### 3. Interview Q&A Loop

```
Browser (InterviewScreen.jsx)
  │  on mount, calls useInterview.fetchNextQuestion(sessionId)
  │
  ▼
api.js → GET /api/v1/interview/{id}/next
  │
  ▼
interview_service.get_next_question(db, session_id, user_id)
  │  interview_repository.get_next_unanswered_question()
  │    — SQL: JOIN Question + InterviewQuestion LEFT JOIN Answer
  │           WHERE Answer.id IS NULL  ORDER BY sequence_number ASC
  │  returns { status: "in_progress", question_text, bloom_level, sequence, time_limit }
  │  (or { status: "completed" } when all answered)
  │
  ▼
QuestionCard.jsx renders the question + countdown timer
User types their answer and clicks "Submit"
  │
  ▼
useInterview.answer(interview_question_id, user_answer, sessionId)
  │  calls api.js → POST /api/v1/interview/{id}/answer
  │
  ▼
interview_service.submit_answer(db, session_id, body, user_id)
  │  1. Validates session ownership and active status
  │  2. interview_repository.get_question_link() — verifies question belongs to this session
  │  3. interview_repository.get_answer_for_question() — guards duplicate submissions
  │  4. interview_repository.save_answer() — inserts Answer row
  │     (evaluation_score=NULL — AI scoring deferred to summary step)
  │  5. get_next_unanswered_question() — if None, marks session "completed"
  │  returns { score: null, feedback: null, is_complete: bool }
  │
  ▼
If is_complete=false → fetchNextQuestion() loop repeats
If is_complete=true  → fetchSummary() is called
```

---

### 4. Results / Summary Flow

```
useInterview.fetchSummary(sessionId)
  │  calls api.js → GET /api/v1/interview/{id}/summary
  │
  ▼
interview_service.get_summary(db, session_id, user_id)
  │  1. interview_repository.get_answers_for_session() — all Answer rows for this session
  │  2. For each answer where evaluation_score IS NULL:
  │     │
  │     ▼
  │    ai/services/check_answers.check_answer_correctness(question, answer, student_id)
  │     │  a. pinecone_service.query_embeddings() — retrieves relevant past memory
  │     │  b. Builds evaluation prompt (question + student answer)
  │     │  c. llm_service.generate_response() → Groq returns JSON { score, explanation, feedback }
  │     │  d. Classifies level: Weak / Average / Strong / Excellent
  │     │  e. pinecone_service.upsert_embeddings() — stores this interaction in memory
  │     │  returns { score, level, explanation, feedback }
  │     │
  │     Updates answer_record.evaluation_score / feedback / ai_evaluation_metadata
  │  3. db.commit() — all scores persisted
  │  4. Calculates average_score across all answers
  │  5. Maps average to performance_level (Excellent / Strong / Average / Weak)
  │  6. Marks session "completed" if not already
  │  returns { average_score, performance_level, total_answered, breakdown: [...] }
  │
  ▼
InterviewScreen.jsx renders score ring, performance badge, and per-question breakdown
```

---

### 5. Data Flow Summary (Tables)

```
users ──────────────────────────────────────────────────────┐
  id, name, email, password_hash, role, auth_provider, …    │
                                                             │ user_id FK
interview_sessions ──────────────────────────────────────── ▼
  id, user_id, mode, difficulty, bloom_strategy,
  num_questions_requested, num_questions_generated, status

questions ──────────────────────────────────────────────────┐
  id, question_text, bloom_level, difficulty, topic_tags     │
                                                   │         │ question_id FK
                                                   │         ▼
interview_questions (junction) ─────────────────── ▼ ──────►  id, interview_session_id, question_id,
                                                             sequence_number, bloom_level_at_time
                                                                      │
                                                                      │ interview_question_id FK
                                                                      ▼
answers ──────────────────────────────────────────────────────────────
  id, interview_question_id, answer_text,
  evaluation_score, feedback, ai_evaluation_metadata (JSONB)
```

---

## Setup Instructions

### Prerequisites

- Python 3.10+
- Node.js 18+
- PostgreSQL running locally or remotely

---

## Running the Backend

All commands below must be run from the `backend/` directory.

### 1. Create a Virtual Environment

```bash
python3 -m venv .venv
```

Activate it:

```bash
# macOS / Linux
source .venv/bin/activate

# Windows (Command Prompt)
.venv\Scripts\activate.bat

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure Environment Variables

```bash
cp .env.example .env
```

Then open `.env` and fill in your values:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
SECRET_KEY=your-secret-key-here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
GOOGLE_CLIENT_ID=your-google-client-id
PINECONE_API_KEY=your-pinecone-api-key
GROQ_API_KEY=your-groq-api-key
```

### 4. Run Database Migrations

```bash
alembic upgrade head
```

### 5. Start the Server

```bash
uvicorn app.main:app --reload
```

The API will be available at `http://127.0.0.1:8000`.  
Interactive Swagger docs: `http://127.0.0.1:8000/docs`

---

## Running the Frontend

All commands below must be run from the `frontend/` directory.

### 1. Install Dependencies

```bash
npm install
```

### 2. Configure Environment Variables

Create a `.env` file in `frontend/`:

```env
VITE_GOOGLE_CLIENT_ID=your-google-client-id
```

### 3. Start the Dev Server

```bash
npm run dev
```

The app will be available at `http://localhost:5173`.

---

## Interview Flow

```
User fills InterviewConfig
        ↓
POST /interview/start  →  AI generates questions  →  session created
        ↓
GET /interview/{id}/next  →  first question returned
        ↓
User reads question + types answer
        ↓
POST /interview/{id}/answer  →  answer saved (no scoring yet)
        ↓
GET /interview/{id}/next  →  next question (repeat until done)
        ↓
Backend detects all answers submitted → session marked "completed"
        ↓
GET /interview/{id}/summary  →  AI batch-evaluates all answers
        ↓
Frontend displays score ring, performance level, question breakdown
```

---

## API Endpoints

All endpoints are prefixed with `/api/v1`.

### Authentication

| Method | Endpoint       | Description                  |
| ------ | -------------- | ---------------------------- |
| POST   | `/auth/signup` | Register a new user          |
| POST   | `/auth/login`  | Authenticate and receive JWT |
| POST   | `/auth/google` | Google OAuth2 login          |

### Interview Engine

| Method | Endpoint                  | Description                             |
| ------ | ------------------------- | --------------------------------------- |
| POST   | `/interview/start`        | Start a new interview session           |
| GET    | `/interview/{id}/next`    | Fetch the next unanswered question      |
| POST   | `/interview/{id}/answer`  | Submit an answer (no immediate scoring) |
| GET    | `/interview/{id}/summary` | Get AI-evaluated performance summary    |
| GET    | `/interview/{id}/result`  | Get simplified score and percentage     |

> All interview endpoints require a valid `Authorization: Bearer <token>` header.

---

## Environment Variables Reference

| Variable                      | Description                                     |
| ----------------------------- | ----------------------------------------------- |
| `DATABASE_URL`                | PostgreSQL connection string                    |
| `SECRET_KEY`                  | JWT signing secret                              |
| `ALGORITHM`                   | JWT signing algorithm (default: `HS256`)        |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Token validity period in minutes                |
| `GOOGLE_CLIENT_ID`            | Google OAuth2 client ID (must be a single line) |
| `PINECONE_API_KEY`            | API key for Pinecone vector database            |
| `GROQ_API_KEY`                | API key for Groq LLM inference                  |
| `AI_MOCK_MODE`                | Set to `true` to bypass real AI calls locally   |

---

## Pull Request Policy

- **Direct push to `main` is strictly prohibited.**
- All changes must be submitted via Pull Requests.
- Only Team Leads are authorized to open PRs against `main`.
- Each team may only modify their assigned folder (`frontend/`, `backend/`, `ai/`).

---

## Purpose

PARIKSHAN_AI is designed to help students practice technical interviews at any time. The AI-driven architecture dynamically tailors questions to the selected subject and cognitive level, ensuring every session is meaningful and personalized.
