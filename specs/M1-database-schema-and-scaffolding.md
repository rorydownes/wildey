# Milestone 1 — Database Schema & Project Scaffolding

## Objective

Establish the monorepo structure, initialize all three components (React frontend, FastAPI backend, ingestion worker), define the complete SQLite schema, implement the configuration system, and verify the dev environment runs end-to-end with seed data.

---

## Step 1: Monorepo Directory Structure

Create the following directory layout at the project root:

```
job-tracker/
├── frontend/                # React + Vite app
│   ├── src/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/             # API client functions
│   │   ├── types/           # TypeScript or JS type definitions
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── public/
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
├── backend/                 # FastAPI server
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py          # FastAPI app entry point
│   │   ├── routes/          # Route modules
│   │   ├── models.py        # Pydantic response models
│   │   └── database.py      # SQLite connection helpers
│   └── requirements.txt
├── ingestion/               # Standalone ingestion worker
│   ├── __init__.py
│   ├── worker.py            # Main polling loop entry point
│   ├── gmail_client.py      # Gmail API wrapper
│   ├── llm_analyzer.py      # Claude CLI integration
│   ├── company_resolver.py  # Company identification logic
│   └── requirements.txt
├── db/
│   ├── schema.sql           # Full DDL for all tables
│   └── seed.sql             # Seed data for development
├── config/
│   └── config.yaml          # All configurable parameters
├── scripts/
│   ├── init_db.py           # Creates the database from schema.sql
│   └── dev_start.sh         # Starts all three components
├── credentials/             # .gitignored — OAuth tokens stored here
│   └── .gitkeep
├── .gitignore
├── .env.example             # Documents required env vars
└── README.md
```

**Implementation notes**:
- The `credentials/` directory must be added to `.gitignore` immediately. It will hold `client_secret.json` and `token.json` from the Gmail OAuth flow.
- The `db/` directory holds the schema definition and seed data, but the actual SQLite `.db` file is created at the path specified in `config.yaml` (default: `db/tracker.db`).

---

## Step 2: Configuration System

Create `config/config.yaml` with the following structure:

```yaml
gmail:
  polling_interval_seconds: 600      # 10 minutes
  backfill_months: 3                 # Minimum months to backfill on first run
  credentials_dir: "./credentials"   # Path to OAuth credential files

database:
  path: "./db/tracker.db"

llm:
  claude_cli_path: "claude"          # Path to Claude CLI binary
  model: "sonnet"                    # Model to use for analysis
  max_retries: 3                     # Retries on CLI failure
  timeout_seconds: 30                # Timeout per CLI call

thresholds:
  stale_days: 7                      # Days without activity before flagging
  rejection_cooldown_months: 3       # Post-interview rejection re-apply cooldown

server:
  host: "127.0.0.1"
  port: 8000
  frontend_dist: "../frontend/dist"  # Path to built frontend assets
```

**Implementation notes**:
- Create a shared Python module (`backend/app/config.py` and symlinked or duplicated for `ingestion/`) that loads this YAML file at startup using `PyYAML`.
- Environment variables should override YAML values where set (e.g., `TRACKER_DB_PATH` overrides `database.path`). Use the convention `TRACKER_` prefix for all env vars.
- The config loader should validate that all required keys are present and raise clear errors on missing values at startup, not at first use.

---

## Step 3: SQLite Database Schema

Create `db/schema.sql` with the following DDL:

### Table: `emails`

Stores every individual email message fetched from Gmail.

```sql
CREATE TABLE IF NOT EXISTS emails (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    gmail_message_id TEXT UNIQUE NOT NULL,     -- Gmail's unique message ID
    gmail_thread_id TEXT NOT NULL,             -- Gmail's thread ID for grouping
    gmail_history_id TEXT,                     -- History ID for incremental sync
    subject TEXT,
    sender_address TEXT NOT NULL,
    sender_name TEXT,
    recipient_addresses TEXT NOT NULL,         -- JSON array of recipient emails
    body_text TEXT,                            -- Plain text body content
    sent_at TIMESTAMP NOT NULL,               -- Email send timestamp (UTC)
    direction TEXT NOT NULL CHECK(direction IN ('inbound', 'outbound')),  -- Relative to user
    raw_headers TEXT,                          -- JSON blob of select headers for debugging
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Indexes defined below
    UNIQUE(gmail_message_id)
);

CREATE INDEX IF NOT EXISTS idx_emails_thread_id ON emails(gmail_thread_id);
CREATE INDEX IF NOT EXISTS idx_emails_sent_at ON emails(sent_at);
CREATE INDEX IF NOT EXISTS idx_emails_sender ON emails(sender_address);
```

### Table: `companies`

One row per identified company.

```sql
CREATE TABLE IF NOT EXISTS companies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,                        -- LLM-inferred company name
    domain TEXT,                               -- Primary email domain if identifiable
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_companies_name ON companies(name);
```

### Table: `applications`

One row per job application (a user may have multiple applications to the same company over time, but typically one active at a time).

```sql
CREATE TABLE IF NOT EXISTS applications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    company_id INTEGER NOT NULL REFERENCES companies(id),
    status_summary TEXT,                       -- LLM-generated 1-sentence status
    stage TEXT CHECK(stage IN (
        'applied', 'recruiter_screen', 'interviewing',
        'offer', 'accepted', 'rejected_pre_interview',
        'rejected_post_interview', 'withdrawn', 'unknown'
    )),
    rejection_date TIMESTAMP,                  -- When rejection was received (if applicable)
    reapply_eligible_date TIMESTAMP,           -- Computed: rejection_date + cooldown (or immediate)
    last_interaction_at TIMESTAMP,             -- Timestamp of most recent email in this application
    last_sender TEXT,                          -- 'user' or 'company' — who sent the last email
    is_stale INTEGER DEFAULT 0,               -- 1 if last_interaction_at > stale_days threshold
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (company_id) REFERENCES companies(id)
);

CREATE INDEX IF NOT EXISTS idx_applications_company ON applications(company_id);
CREATE INDEX IF NOT EXISTS idx_applications_stage ON applications(stage);
CREATE INDEX IF NOT EXISTS idx_applications_stale ON applications(is_stale);
CREATE INDEX IF NOT EXISTS idx_applications_last_interaction ON applications(last_interaction_at);
```

### Table: `email_application_map`

Maps individual emails to applications. An email may map to exactly one application. This enables the conversation view.

```sql
CREATE TABLE IF NOT EXISTS email_application_map (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email_id INTEGER NOT NULL REFERENCES emails(id),
    application_id INTEGER NOT NULL REFERENCES applications(id),

    UNIQUE(email_id),
    FOREIGN KEY (email_id) REFERENCES emails(id),
    FOREIGN KEY (application_id) REFERENCES applications(id)
);

CREATE INDEX IF NOT EXISTS idx_email_app_map_application ON email_application_map(application_id);
```

### Table: `sync_state`

Tracks Gmail sync progress for incremental fetching.

```sql
CREATE TABLE IF NOT EXISTS sync_state (
    id INTEGER PRIMARY KEY CHECK(id = 1),      -- Singleton row
    last_history_id TEXT,                       -- Gmail history ID for incremental sync
    last_sync_at TIMESTAMP,                     -- When the last successful sync completed
    initial_backfill_complete INTEGER DEFAULT 0  -- 1 once the first full backfill finishes
);

-- Initialize the singleton row
INSERT OR IGNORE INTO sync_state (id, initial_backfill_complete) VALUES (1, 0);
```

### Table: `llm_analysis_log`

Audit trail of LLM calls made during ingestion. Useful for debugging and avoiding redundant calls.

```sql
CREATE TABLE IF NOT EXISTS llm_analysis_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    application_id INTEGER REFERENCES applications(id),
    analysis_type TEXT NOT NULL CHECK(analysis_type IN (
        'company_identification', 'status_update', 'rejection_classification'
    )),
    input_summary TEXT,                        -- Truncated input sent to Claude (for debugging)
    output_raw TEXT,                           -- Raw CLI output
    model_used TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (application_id) REFERENCES applications(id)
);
```

---

## Step 4: Database Initialization Script

Create `scripts/init_db.py`:

- Read `database.path` from `config/config.yaml`.
- Execute `db/schema.sql` against the SQLite database at that path.
- If the database file already exists, the `IF NOT EXISTS` clauses ensure idempotency.
- Print a summary of tables created.
- Optionally accept a `--seed` flag that also executes `db/seed.sql`.

---

## Step 5: Seed Data

Create `db/seed.sql` with realistic test data for development:

- 5–8 companies with varied names (mix of recognizable patterns like "TechCorp Recruiting", "hiring@startup.io", etc.).
- 15–25 emails spread across those companies, with realistic subject lines and body text that simulate recruiter conversations at different stages.
- Applications in varied states: 2 active (one stale), 1 in interviewing, 1 with an offer, 1 rejected pre-interview, 1 rejected post-interview with a recent rejection date, 1 rejected post-interview with a rejection date >3 months ago (ready to re-apply).
- Properly linked `email_application_map` entries.
- A `sync_state` row with `initial_backfill_complete = 1` so the dashboard doesn't show the syncing state when using seed data.

This seed data is critical for frontend development in Milestone 4 — the React app needs to be buildable and testable before the Gmail integration is working.

---

## Step 6: FastAPI Server Skeleton

Initialize the FastAPI server in `backend/`:

### `backend/app/database.py`
- Create a helper that returns a SQLite connection using the path from config.
- Use `sqlite3.Row` row factory so results are accessible by column name.
- Implement a context manager pattern for connection lifecycle.
- Enable WAL mode (`PRAGMA journal_mode=WAL`) on connection to allow concurrent reads from the API while the ingestion worker writes.

### `backend/app/main.py`
- Create the FastAPI app instance.
- Mount a static file handler for the built React frontend at `/` (serving from the path in `config.server.frontend_dist`).
- Include a health check endpoint: `GET /api/health` returning `{"status": "ok", "db_path": "...", "last_sync": "..."}`.
- Include route modules from `backend/app/routes/` (empty stubs for now, to be implemented in Milestone 4).

### `backend/requirements.txt`
```
fastapi>=0.100.0
uvicorn>=0.23.0
pyyaml>=6.0
```

---

## Step 7: React App Skeleton

Initialize the React app using Vite in `frontend/`:

```bash
npm create vite@latest frontend -- --template react
```

### Minimal setup:
- Clear the boilerplate content from `App.jsx`.
- Set up the project structure with empty directories: `components/`, `hooks/`, `api/`, `types/`.
- Create `api/client.js` with a base fetch wrapper configured to hit `http://localhost:8000/api/`.
- Create a placeholder `App.jsx` that renders "Job Application Tracker — Loading..." and calls the `/api/health` endpoint on mount to verify backend connectivity.
- Configure Vite proxy in `vite.config.js` to forward `/api/*` requests to `http://localhost:8000` during development.

### `frontend/package.json` dependencies:
```json
{
  "dependencies": {
    "react": "^18",
    "react-dom": "^18"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4",
    "vite": "^5"
  }
}
```

---

## Step 8: Ingestion Worker Skeleton

Initialize the ingestion worker in `ingestion/`:

### `ingestion/worker.py`
- Create the main loop structure: load config, connect to DB, enter polling loop.
- The loop should: check `sync_state` to determine if backfill is needed, then either run backfill or incremental sync, then sleep for `polling_interval_seconds`.
- For this milestone, the Gmail and LLM logic are stubs that log messages like "Would fetch emails here" and "Would analyze with Claude here".
- Implement graceful shutdown on SIGINT/SIGTERM.
- Log all actions to stdout with timestamps.

### `ingestion/requirements.txt`
```
google-auth>=2.0
google-auth-oauthlib>=1.0
google-api-python-client>=2.0
pyyaml>=6.0
```

---

## Step 9: Dev Startup Script

Create `scripts/dev_start.sh`:

```bash
#!/bin/bash
# Starts all three components for local development.
# Usage: ./scripts/dev_start.sh

# Initialize DB if it doesn't exist
python scripts/init_db.py

# Start FastAPI backend
cd backend && uvicorn app.main:app --reload --port 8000 &
BACKEND_PID=$!

# Start React dev server
cd frontend && npm run dev &
FRONTEND_PID=$!

# Start ingestion worker (optional flag to skip in dev)
if [ "$SKIP_INGESTION" != "1" ]; then
    cd ingestion && python worker.py &
    INGESTION_PID=$!
fi

# Trap Ctrl+C to kill all processes
trap "kill $BACKEND_PID $FRONTEND_PID $INGESTION_PID 2>/dev/null; exit" SIGINT SIGTERM

wait
```

**Implementation notes**:
- Support `SKIP_INGESTION=1` env var so frontend developers can work without the ingestion worker running (using seed data).
- All three processes should log to stdout with a prefix identifying the component (e.g., `[backend]`, `[frontend]`, `[ingestion]`).

---

## Step 10: .gitignore and Environment Setup

### `.gitignore`
```
# Database
db/*.db

# Credentials
credentials/
*.json
!package.json
!tsconfig.json

# Python
__pycache__/
*.pyc
.venv/
venv/

# Node
node_modules/
frontend/dist/

# Environment
.env

# OS
.DS_Store
```

### `.env.example`
```bash
# Optional overrides (values in config/config.yaml are used by default)
# TRACKER_DB_PATH=./db/tracker.db
# TRACKER_GMAIL_POLLING_INTERVAL=600
# TRACKER_STALE_DAYS=7
```

---

## Acceptance Criteria

This milestone is complete when:

1. Running `python scripts/init_db.py` creates the SQLite database with all tables.
2. Running `python scripts/init_db.py --seed` populates the database with realistic test data.
3. The FastAPI server starts and `GET /api/health` returns a success response.
4. The React dev server starts and renders the placeholder page.
5. The React app successfully calls the health endpoint through the Vite proxy.
6. The ingestion worker starts, logs its stub messages, and respects the polling interval from `config.yaml`.
7. `scripts/dev_start.sh` launches all three components and shuts them all down cleanly on Ctrl+C.
8. The `credentials/` directory and `db/*.db` files are git-ignored.
