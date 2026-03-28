# Job Application Tracker — Project Overview

## Purpose

A locally-run web dashboard that connects to a dedicated Gmail account used for job applications, ingests all email correspondence, and presents a clean interface for tracking application status across companies. The system uses LLM analysis (Claude via CLI) during ingestion to automatically categorize conversations, determine application status, and flag items needing attention.

This is a proof-of-concept scoped to local development. Deployment, multi-user support, and hosted infrastructure are explicitly out of scope.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Frontend | React (Vite) |
| Backend API | Python / FastAPI |
| Database | SQLite |
| Email Source | Gmail API (OAuth2) |
| LLM Analysis | Claude CLI (subprocess calls during ingestion) |
| Ingestion | Standalone Python process (separate from the API server) |

---

## Architecture at a Glance

The system has three independent components that share a single SQLite database:

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────┐
│  React App   │──────▶│  FastAPI Server   │──────▶│              │
│  (browser)   │ HTTP  │  (read-only API)  │  SQL  │              │
└──────────────┘       └──────────────────┘       │    SQLite    │
                                                   │      DB      │
┌──────────────────────────────────┐               │              │
│  Ingestion Worker (standalone)   │──────────────▶│              │
│  - Gmail polling                 │  SQL + Claude │              │
│  - LLM analysis via Claude CLI   │     CLI       │              │
└──────────────────────────────────┘               └──────────────┘
```

**FastAPI Server**: Serves the React frontend and exposes read-only REST endpoints. All data comes from SQLite — no live Gmail calls from the dashboard.

**Ingestion Worker**: A separate long-running Python process that polls Gmail on a configurable interval (default: 10 minutes), fetches new emails, groups them by company, runs Claude CLI analysis, and writes results to SQLite. On first run, it backfills at minimum 3 months of historical email data.

**SQLite Database**: Single source of truth. Stores raw email data, company-level aggregations, and LLM-generated metadata. Shared by the API server (reads) and ingestion worker (writes).

---

## Core Data Model (Conceptual)

**emails** — Raw email records pulled from Gmail. Each row is a single email message with sender, recipients, subject, body text, Gmail thread ID, and timestamp.

**companies** — One row per identified company. Company identification is LLM-inferred from email content during ingestion. Stores the company name and any metadata.

**applications** — Represents a job application to a specific company. Links to a company and contains LLM-generated fields: a 1-sentence status summary, application stage, rejection type (if applicable), and last interaction timestamp.

**email_company_map** — Maps individual emails to companies, enabling a single company view to pull conversation history across multiple Gmail threads.

**aggregations** — Pre-computed company-level data written during ingestion: current status, last interaction date, stale flag (no interaction in 7+ days), rejection classification, and re-apply eligibility.

---

## Key Features

### Company Table View
The primary interface is a table of companies with key columns: company name, current status summary, last interaction date, and staleness indicator. Each row is expandable.

### Chat-Style Conversation View
Expanding a company row reveals a chronological feed of all email interactions with that company, rendered in a chat-bubble style (similar to iMessage/WhatsApp). Messages are ordered by send time across all related email threads. Sent emails appear on one side, received emails on the other.

### Application Status Tracking
During ingestion, Claude CLI analyzes each company's conversation and produces a 1-sentence status summary describing the current state or most recent completed step (e.g., "Completed second round technical interview", "Awaiting response after recruiter screen", "Received offer letter"). These summaries are generic to any job type — not specific to engineering roles.

### Rejection Classification
Rejections are classified into two categories:

- **Pre-interview rejection**: Rejected before any non-recruiter interview was scheduled. These companies are immediately eligible for re-application.
- **Post-interview rejection**: Rejected after substantive interviews began. A 3-month cooldown period applies before re-application is recommended.

The dashboard surfaces re-apply eligibility clearly per company.

### Stale Application Flagging
Applications with no email activity in the last 7 days are flagged as stale in the table view, signaling the user should follow up.

### LLM Analysis Boundary
All Claude CLI calls happen exclusively in the ingestion worker. The dashboard and API layer perform zero LLM calls — every read is serviced from pre-computed data in SQLite. This keeps the UI fast and predictable.

---

## Gmail Integration

- **Authentication**: OAuth2 via Google Cloud project. Credentials and tokens stored locally.
- **Polling**: Configurable interval via external config file or environment variable (default: 10 minutes). The ingestion worker uses Gmail API's history-based sync to efficiently fetch only new messages after initial backfill.
- **Backfill**: On first run, fetches at minimum 3 months of email history from the connected account.
- **Scope**: Email body text only for MVP. Attachments and calendar invites are out of scope.

---

## Configuration

Key parameters are managed outside application code (e.g., via a `.env` file or `config.yaml`):

- Gmail polling interval
- Backfill depth (number of months on first run)
- Stale threshold (days without interaction)
- Rejection cooldown period (months)
- Claude CLI path / model preferences
- SQLite database path

---

## Milestones

### Milestone 1 — Database Schema & Project Scaffolding
Set up the monorepo structure, initialize the React app (Vite), FastAPI server, and SQLite database. Define and create all tables. Establish the config system. Get the dev environment running end-to-end with placeholder data.

### Milestone 2 — Gmail OAuth2 & Email Ingestion
Implement OAuth2 authentication flow for Gmail. Build the standalone ingestion worker that polls for new emails and writes raw email data to SQLite. Implement the initial 3-month backfill. Handle Gmail thread grouping and incremental sync via history API.

### Milestone 3 — LLM-Powered Company Identification & Analysis
Integrate Claude CLI calls into the ingestion pipeline. Implement company identification from email content. Generate application status summaries, rejection classifications, and staleness flags. Write all aggregated data to the appropriate tables.

### Milestone 4 — API Layer & Core Dashboard UI
Build FastAPI read-only endpoints for the company table, conversation history, and application metadata. Build the React frontend: company table with expandable rows, chat-style conversation view, status display, stale flags, and rejection/re-apply indicators.

### Milestone 5 — Dashboard Summary Stats & Polish
Add summary statistics to the top of the dashboard (active application count, interviews this week, rejection rate, stale count, etc.). Visual polish pass on the UI. This milestone is a nice-to-have and should only be started after Milestones 1–4 are solid.

---

## Out of Scope (Future Work)

- Hosted deployment / multi-device access
- Email attachment surfacing
- Calendar invite detection and interview scheduling
- Multi-user support / authentication on the dashboard
- Manual company tagging or override UI
- Email sending or reply functionality from the dashboard
