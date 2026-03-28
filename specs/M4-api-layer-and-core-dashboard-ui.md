# Milestone 4 — API Layer & Core Dashboard UI

## Objective

Build the FastAPI read-only REST endpoints that serve application data from SQLite, and the complete React frontend: company table with sortable/filterable columns, inline expandable conversation view with chat-style message bubbles, stale indicators, rejection badges, and re-apply eligibility display. After this milestone, the dashboard is fully functional end-to-end.

---

## Step 1: FastAPI Endpoint Design

All endpoints are read-only (`GET`) and serve pre-computed data from SQLite. No endpoint triggers LLM calls or Gmail fetches.

### Base URL: `/api`

### Endpoint: `GET /api/applications`

Returns the data powering the company table.

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `stage` | string | (none) | Filter by stage value. Accepts comma-separated list: `stage=interviewing,offer` |
| `is_stale` | boolean | (none) | Filter to stale (`true`) or non-stale (`false`) applications |
| `reapply_eligible` | boolean | (none) | If `true`, return only applications where `reapply_eligible_date <= now` |
| `search` | string | (none) | Case-insensitive search across company name and status_summary |
| `sort` | string | `last_interaction_at` | Sort field: `last_interaction_at`, `company_name` |
| `order` | string | `desc` | Sort direction: `asc` or `desc` |

**Response schema**:
```json
{
    "applications": [
        {
            "id": 1,
            "company_id": 1,
            "company_name": "Stripe",
            "company_domain": "stripe.com",
            "status_summary": "Completed phone screen with recruiter, awaiting technical interview scheduling",
            "stage": "recruiter_screen",
            "rejection_date": null,
            "reapply_eligible_date": null,
            "reapply_eligible": false,
            "last_interaction_at": "2026-03-25T14:30:00Z",
            "last_sender": "company",
            "is_stale": false
        }
    ],
    "total_count": 42
}
```

**SQL query structure**:
```python
def get_applications(db_conn, filters: dict) -> list:
    query = """
        SELECT
            a.id, a.company_id, c.name as company_name, c.domain as company_domain,
            a.status_summary, a.stage, a.rejection_date, a.reapply_eligible_date,
            CASE WHEN a.reapply_eligible_date IS NOT NULL
                 AND a.reapply_eligible_date <= datetime('now')
                 THEN 1 ELSE 0 END as reapply_eligible,
            a.last_interaction_at, a.last_sender, a.is_stale
        FROM applications a
        JOIN companies c ON c.id = a.company_id
        WHERE 1=1
    """
    params = []

    # Dynamic WHERE clause construction based on filters
    if filters.get('stage'):
        stages = filters['stage'].split(',')
        placeholders = ','.join('?' * len(stages))
        query += f" AND a.stage IN ({placeholders})"
        params.extend(stages)

    if filters.get('is_stale') is not None:
        query += " AND a.is_stale = ?"
        params.append(1 if filters['is_stale'] else 0)

    if filters.get('search'):
        query += " AND (LOWER(c.name) LIKE ? OR LOWER(a.status_summary) LIKE ?)"
        search_term = f"%{filters['search'].lower()}%"
        params.extend([search_term, search_term])

    if filters.get('reapply_eligible'):
        query += " AND a.reapply_eligible_date IS NOT NULL AND a.reapply_eligible_date <= datetime('now')"

    # Sorting
    sort_col = 'a.last_interaction_at' if filters.get('sort') != 'company_name' else 'c.name'
    sort_dir = 'ASC' if filters.get('order') == 'asc' else 'DESC'
    query += f" ORDER BY {sort_col} {sort_dir}"

    return db_conn.execute(query, params).fetchall()
```

---

### Endpoint: `GET /api/applications/{application_id}/conversation`

Returns the full email conversation for a single application, ordered chronologically.

**Response schema**:
```json
{
    "application_id": 1,
    "company_name": "Stripe",
    "status_summary": "Completed phone screen with recruiter, awaiting technical interview scheduling",
    "stage": "recruiter_screen",
    "messages": [
        {
            "id": 101,
            "direction": "inbound",
            "sender_name": "Jane Smith",
            "sender_address": "jane@stripe.com",
            "subject": "Software Engineering Manager - Application Received",
            "body_text": "Hi Rory, thank you for applying to the Software Engineering Manager position...",
            "sent_at": "2026-03-10T09:15:00Z",
            "gmail_thread_id": "thread_abc123"
        },
        {
            "id": 102,
            "direction": "outbound",
            "sender_name": null,
            "sender_address": "user@gmail.com",
            "subject": "Re: Software Engineering Manager - Application Received",
            "body_text": "Thanks Jane, I'm very excited about this opportunity...",
            "sent_at": "2026-03-10T14:22:00Z",
            "gmail_thread_id": "thread_abc123"
        }
    ]
}
```

**SQL query**:
```sql
SELECT
    e.id, e.direction, e.sender_name, e.sender_address,
    e.subject, e.body_text, e.sent_at, e.gmail_thread_id
FROM emails e
JOIN email_application_map m ON m.email_id = e.id
WHERE m.application_id = ?
ORDER BY e.sent_at ASC
```

---

### Endpoint: `GET /api/stats`

Returns aggregated counts for the summary stats bar. Used by Milestone 5, but implement it now since the query is simple.

**Response schema**:
```json
{
    "active_applications": 12,
    "awaiting_response": 5,
    "interviews_scheduled": 3,
    "stale_count": 4,
    "total_rejections": 8,
    "ready_to_reapply": 2
}
```

**SQL queries** (run as separate queries for clarity, or combine into one with CASE expressions):

```sql
-- active: not in a terminal stage
SELECT COUNT(*) FROM applications
WHERE stage NOT IN ('rejected_pre_interview', 'rejected_post_interview', 'accepted', 'withdrawn');

-- awaiting_response: last sender was user, not in terminal stage
SELECT COUNT(*) FROM applications
WHERE last_sender = 'user'
AND stage NOT IN ('rejected_pre_interview', 'rejected_post_interview', 'accepted', 'withdrawn');

-- interviews_scheduled: stage is 'interviewing'
SELECT COUNT(*) FROM applications WHERE stage = 'interviewing';

-- stale
SELECT COUNT(*) FROM applications WHERE is_stale = 1;

-- total rejections
SELECT COUNT(*) FROM applications
WHERE stage IN ('rejected_pre_interview', 'rejected_post_interview');

-- ready to reapply
SELECT COUNT(*) FROM applications
WHERE reapply_eligible_date IS NOT NULL AND reapply_eligible_date <= datetime('now');
```

---

### Endpoint: `GET /api/sync-status`

Returns the last sync time for the footer display.

**Response schema**:
```json
{
    "last_sync_at": "2026-03-28T14:30:00Z",
    "initial_backfill_complete": true
}
```

---

## Step 2: FastAPI Implementation Details

### Pydantic Response Models

Define response models in `backend/app/models.py` for all endpoints. Use Pydantic's `BaseModel` for serialization and automatic OpenAPI documentation.

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class ApplicationListItem(BaseModel):
    id: int
    company_id: int
    company_name: str
    company_domain: Optional[str]
    status_summary: Optional[str]
    stage: str
    rejection_date: Optional[datetime]
    reapply_eligible_date: Optional[datetime]
    reapply_eligible: bool
    last_interaction_at: Optional[datetime]
    last_sender: Optional[str]
    is_stale: bool

class ApplicationListResponse(BaseModel):
    applications: list[ApplicationListItem]
    total_count: int

class ConversationMessage(BaseModel):
    id: int
    direction: str
    sender_name: Optional[str]
    sender_address: str
    subject: Optional[str]
    body_text: Optional[str]
    sent_at: datetime
    gmail_thread_id: str

class ConversationResponse(BaseModel):
    application_id: int
    company_name: str
    status_summary: Optional[str]
    stage: str
    messages: list[ConversationMessage]

class StatsResponse(BaseModel):
    active_applications: int
    awaiting_response: int
    interviews_scheduled: int
    stale_count: int
    total_rejections: int
    ready_to_reapply: int

class SyncStatusResponse(BaseModel):
    last_sync_at: Optional[datetime]
    initial_backfill_complete: bool
```

### Route Organization

Create separate route files in `backend/app/routes/`:

- `applications.py` — `/api/applications` and `/api/applications/{id}/conversation`
- `stats.py` — `/api/stats`
- `sync.py` — `/api/sync-status`

Register all routes in `main.py` using `app.include_router(...)`.

### CORS Configuration

Since the React dev server runs on a different port (typically 5173) than FastAPI (8000), configure CORS:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://127.0.0.1:5173"],
    allow_methods=["GET"],
    allow_headers=["*"],
)
```

Only allow `GET` since all endpoints are read-only.

---

## Step 3: React Frontend — API Client

Create `frontend/src/api/client.js`:

```javascript
const API_BASE = '/api';

async function fetchApi(path, params = {}) {
    const url = new URL(path, window.location.origin);
    Object.entries(params).forEach(([key, value]) => {
        if (value !== null && value !== undefined && value !== '') {
            url.searchParams.set(key, value);
        }
    });

    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`API error: ${response.status} ${response.statusText}`);
    }
    return response.json();
}

export async function getApplications(filters = {}) {
    return fetchApi(`${API_BASE}/applications`, filters);
}

export async function getConversation(applicationId) {
    return fetchApi(`${API_BASE}/applications/${applicationId}/conversation`);
}

export async function getStats() {
    return fetchApi(`${API_BASE}/stats`);
}

export async function getSyncStatus() {
    return fetchApi(`${API_BASE}/sync-status`);
}
```

---

## Step 4: React Frontend — Component Architecture

```
App.jsx
├── SyncStatusFooter.jsx          # Fixed footer showing last sync time
└── Dashboard.jsx                 # Main dashboard layout
    ├── StatsBar.jsx              # Summary stat cards (renders placeholder in M4, wired in M5)
    ├── FilterControls.jsx        # Search, status filter, sort controls
    └── CompanyTable.jsx          # The main table
        ├── CompanyRow.jsx        # Collapsed row for a single application
        └── ConversationView.jsx  # Expanded inline panel
            ├── CompanyHeader.jsx # Company name, status, stage badge
            └── MessageBubble.jsx # Individual chat-style message
```

---

## Step 5: React Frontend — State Management

Use React hooks (`useState`, `useEffect`, `useCallback`) for all state management. No external state library needed for this app.

### Top-Level State in `Dashboard.jsx`

```javascript
function Dashboard() {
    // Application list data
    const [applications, setApplications] = useState([]);
    const [totalCount, setTotalCount] = useState(0);
    const [loading, setLoading] = useState(true);

    // Filter state
    const [searchQuery, setSearchQuery] = useState('');
    const [stageFilter, setStageFilter] = useState('');  // '' = all
    const [sortField, setSortField] = useState('last_interaction_at');
    const [sortOrder, setSortOrder] = useState('desc');
    const [statFilter, setStatFilter] = useState(null);  // From stat card clicks

    // Expanded row
    const [expandedApplicationId, setExpandedApplicationId] = useState(null);

    // Fetch applications when filters change
    useEffect(() => {
        fetchApplications();
    }, [searchQuery, stageFilter, sortField, sortOrder, statFilter]);

    async function fetchApplications() {
        setLoading(true);
        const params = {
            search: searchQuery || undefined,
            stage: resolveStageFilter(stageFilter, statFilter),
            is_stale: statFilter === 'stale' ? true : undefined,
            reapply_eligible: statFilter === 'reapply' ? true : undefined,
            sort: sortField,
            order: sortOrder,
        };
        const data = await getApplications(params);
        setApplications(data.applications);
        setTotalCount(data.total_count);
        setLoading(false);
    }

    function handleRowClick(applicationId) {
        setExpandedApplicationId(
            expandedApplicationId === applicationId ? null : applicationId
        );
    }

    // ... render
}
```

### Debounced Search

The search input should debounce API calls. Do not fire a request on every keystroke:

```javascript
import { useState, useEffect, useCallback } from 'react';

function useDebounce(value, delayMs) {
    const [debouncedValue, setDebouncedValue] = useState(value);
    useEffect(() => {
        const timer = setTimeout(() => setDebouncedValue(value), delayMs);
        return () => clearTimeout(timer);
    }, [value, delayMs]);
    return debouncedValue;
}

// In Dashboard:
const [rawSearchQuery, setRawSearchQuery] = useState('');
const searchQuery = useDebounce(rawSearchQuery, 300);
```

---

## Step 6: React Frontend — Company Table

### `CompanyTable.jsx`

Renders the table header and maps `applications` to `CompanyRow` components.

```jsx
function CompanyTable({ applications, expandedId, onRowClick, loading }) {
    if (loading) {
        return <LoadingState />;
    }

    if (applications.length === 0) {
        return <EmptyState />;  // "No applications match your filters."
    }

    return (
        <div className="company-table">
            <div className="table-header">
                <span className="col-company">Company</span>
                <span className="col-status">Status</span>
                <span className="col-activity">Last Activity</span>
                <span className="col-stale"></span>
                <span className="col-rejection">Stage</span>
            </div>
            {applications.map(app => (
                <React.Fragment key={app.id}>
                    <CompanyRow
                        application={app}
                        isExpanded={expandedId === app.id}
                        onClick={() => onRowClick(app.id)}
                    />
                    {expandedId === app.id && (
                        <ConversationView
                            applicationId={app.id}
                            onCollapse={() => onRowClick(app.id)}
                        />
                    )}
                </React.Fragment>
            ))}
        </div>
    );
}
```

### `CompanyRow.jsx`

```jsx
function CompanyRow({ application, isExpanded, onClick }) {
    const app = application;

    return (
        <div
            className={`company-row ${app.is_stale ? 'stale' : ''} ${isExpanded ? 'expanded' : ''}`}
            onClick={onClick}
        >
            <span className="col-company">{app.company_name}</span>
            <span className="col-status" title={app.status_summary}>
                {app.status_summary}
            </span>
            <span className="col-activity">
                {formatRelativeTime(app.last_interaction_at)}
            </span>
            <span className="col-stale">
                {app.is_stale && <StaleBadge />}
            </span>
            <span className="col-rejection">
                <StageBadge stage={app.stage} reapplyEligible={app.reapply_eligible} />
            </span>
        </div>
    );
}
```

### Stage Badge Logic

```jsx
function StageBadge({ stage, reapplyEligible }) {
    if (reapplyEligible) {
        return <span className="badge badge-green">Ready to Re-apply</span>;
    }

    const badgeMap = {
        'applied': null,  // No badge for default state
        'recruiter_screen': <span className="badge badge-blue">Recruiter Screen</span>,
        'interviewing': <span className="badge badge-blue">Interviewing</span>,
        'offer': <span className="badge badge-green">Offer</span>,
        'accepted': <span className="badge badge-green">Accepted</span>,
        'rejected_pre_interview': <span className="badge badge-red-muted">Rejected — Pre-Interview</span>,
        'rejected_post_interview': <span className="badge badge-red-muted">Rejected — Post-Interview</span>,
        'withdrawn': <span className="badge badge-gray">Withdrawn</span>,
        'unknown': null,
    };

    return badgeMap[stage] || null;
}
```

### Relative Time Formatting

Implement a `formatRelativeTime` utility:

```javascript
function formatRelativeTime(isoString) {
    const date = new Date(isoString);
    const now = new Date();
    const diffMs = now - date;
    const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));

    if (diffDays === 0) {
        const diffHours = Math.floor(diffMs / (1000 * 60 * 60));
        if (diffHours === 0) {
            const diffMins = Math.floor(diffMs / (1000 * 60));
            return `${diffMins}m ago`;
        }
        return `${diffHours}h ago`;
    }
    if (diffDays === 1) return 'Yesterday';
    if (diffDays < 7) return `${diffDays}d ago`;

    return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}
```

---

## Step 7: React Frontend — Conversation View

### `ConversationView.jsx`

This component fetches and renders the full email conversation when a row is expanded.

```jsx
function ConversationView({ applicationId, onCollapse }) {
    const [conversation, setConversation] = useState(null);
    const [loading, setLoading] = useState(true);
    const scrollRef = useRef(null);

    useEffect(() => {
        async function load() {
            setLoading(true);
            const data = await getConversation(applicationId);
            setConversation(data);
            setLoading(false);
        }
        load();
    }, [applicationId]);

    // Auto-scroll to bottom (most recent message) on load
    useEffect(() => {
        if (!loading && scrollRef.current) {
            scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
        }
    }, [loading]);

    if (loading) return <div className="conversation-loading">Loading conversation...</div>;

    const messages = conversation.messages;
    let lastThreadId = null;

    return (
        <div className="conversation-view">
            <CompanyHeader conversation={conversation} />
            <div className="message-list" ref={scrollRef}>
                {messages.map((msg, idx) => {
                    const showThreadSeparator =
                        msg.gmail_thread_id !== lastThreadId && lastThreadId !== null;
                    lastThreadId = msg.gmail_thread_id;

                    return (
                        <React.Fragment key={msg.id}>
                            {showThreadSeparator && (
                                <ThreadSeparator subject={msg.subject} />
                            )}
                            <MessageBubble
                                message={msg}
                                showSubject={shouldShowSubject(msg, messages, idx)}
                            />
                        </React.Fragment>
                    );
                })}
            </div>
            <button className="collapse-button" onClick={onCollapse}>
                Collapse
            </button>
        </div>
    );
}
```

### Subject Line Display Logic

Only show the subject line in a bubble if it differs from the previous message:

```javascript
function shouldShowSubject(message, allMessages, index) {
    if (index === 0) return true;
    return message.subject !== allMessages[index - 1].subject;
}
```

### `MessageBubble.jsx`

```jsx
function MessageBubble({ message, showSubject }) {
    const [expanded, setExpanded] = useState(false);
    const isLong = message.body_text && message.body_text.length > 1500;
    const displayText = isLong && !expanded
        ? message.body_text.substring(0, 1000) + '...'
        : message.body_text;

    const isOutbound = message.direction === 'outbound';

    return (
        <div className={`message-bubble ${isOutbound ? 'outbound' : 'inbound'}`}>
            <div className="message-sender">
                {isOutbound ? 'You' : message.sender_name || message.sender_address}
            </div>
            {showSubject && message.subject && (
                <div className="message-subject">{message.subject}</div>
            )}
            <div className="message-body">{displayText}</div>
            {isLong && !expanded && (
                <button
                    className="show-more-button"
                    onClick={(e) => { e.stopPropagation(); setExpanded(true); }}
                >
                    Show more
                </button>
            )}
            <div className="message-timestamp">
                {formatMessageTimestamp(message.sent_at)}
            </div>
        </div>
    );
}
```

### Message Timestamp Format

For the conversation view, use full timestamps (not relative):

```javascript
function formatMessageTimestamp(isoString) {
    const date = new Date(isoString);
    return date.toLocaleDateString('en-US', {
        month: 'short', day: 'numeric', year: 'numeric',
        hour: 'numeric', minute: '2-digit', hour12: true
    });
    // Output: "Mar 15, 2026 at 2:34 PM"
}
```

### Thread Separator

```jsx
function ThreadSeparator({ subject }) {
    return (
        <div className="thread-separator">
            <span className="separator-line" />
            <span className="separator-subject">{subject}</span>
            <span className="separator-line" />
        </div>
    );
}
```

---

## Step 8: React Frontend — Styling Approach

Use a single CSS file (`frontend/src/styles.css`) imported in `main.jsx`. CSS custom properties for theming:

```css
:root {
    --color-primary: #2563eb;       /* Blue — primary actions, outbound bubbles */
    --color-stale: #f59e0b;         /* Amber — stale indicators */
    --color-success: #10b981;       /* Green — re-apply, offers, accepted */
    --color-danger-muted: #9ca3af;  /* Muted gray-red — rejection badges */
    --color-bg: #ffffff;
    --color-bg-secondary: #f9fafb;
    --color-bg-expanded: #f3f4f6;
    --color-text: #111827;
    --color-text-secondary: #6b7280;
    --color-border: #e5e7eb;
    --font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Inter', sans-serif;
}
```

### Key Styling Rules

- **Company rows**: `cursor: pointer`, light hover background, `2px` left border on stale rows using `var(--color-stale)`.
- **Message bubbles**: `max-width: 70%`, `border-radius: 12px`, `padding: 12px 16px`. Inbound: left-aligned, `var(--color-bg-secondary)` background. Outbound: right-aligned, `var(--color-primary)` background, white text.
- **Conversation view container**: `max-height: 500px`, `overflow-y: auto` for internal scrolling.
- **Badges**: `display: inline-block`, `padding: 2px 8px`, `border-radius: 9999px`, `font-size: 0.75rem`, `font-weight: 500`. Color variants per type.

---

## Step 9: React Frontend — Sync Status Footer

### `SyncStatusFooter.jsx`

```jsx
function SyncStatusFooter() {
    const [syncStatus, setSyncStatus] = useState(null);

    useEffect(() => {
        async function poll() {
            const data = await getSyncStatus();
            setSyncStatus(data);
        }
        poll();
        const interval = setInterval(poll, 30000);  // Refresh every 30 seconds
        return () => clearInterval(interval);
    }, []);

    if (!syncStatus) return null;

    return (
        <div className="sync-footer">
            <span className="sync-time">
                {syncStatus.last_sync_at
                    ? `Last synced: ${formatRelativeTime(syncStatus.last_sync_at)}`
                    : 'Not yet synced'
                }
            </span>
            {!syncStatus.initial_backfill_complete && (
                <span className="sync-active">Syncing...</span>
            )}
        </div>
    );
}
```

### Footer Styling

```css
.sync-footer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    height: 36px;
    background: var(--color-bg-secondary);
    border-top: 1px solid var(--color-border);
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 24px;
    font-size: 0.75rem;
    color: var(--color-text-secondary);
    z-index: 100;
}
```

---

## Step 10: Empty and Loading States

### Initial Backfill In Progress

If `sync_status.initial_backfill_complete` is `false` and the applications list is empty, show a centered loading state in the table area:

```jsx
function BackfillLoadingState() {
    return (
        <div className="empty-state">
            <div className="spinner" />
            <p>Syncing your emails... This may take a few minutes on first run.</p>
        </div>
    );
}
```

### No Filter Results

```jsx
function EmptyFilterState({ onClearFilters }) {
    return (
        <div className="empty-state">
            <p>No applications match your filters.</p>
            <button onClick={onClearFilters}>Clear all filters</button>
        </div>
    );
}
```

### OAuth Reconnection Banner

If the API returns errors indicating auth issues (the API endpoint should surface this from `sync_state` or a flag set by the ingestion worker), show a banner at the top of the dashboard:

```jsx
function ReconnectionBanner() {
    return (
        <div className="reconnection-banner">
            Gmail connection expired. Reconnect to resume syncing.
            <a href="/auth/reconnect">Reconnect</a>
        </div>
    );
}
```

---

## Step 11: Vite Proxy Configuration

Update `frontend/vite.config.js` for development:

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    server: {
        port: 5173,
        proxy: {
            '/api': {
                target: 'http://localhost:8000',
                changeOrigin: true,
            },
        },
    },
});
```

---

## Acceptance Criteria

This milestone is complete when:

1. All four API endpoints return correct data from the SQLite database seeded in Milestone 1 or populated by the ingestion worker from Milestones 2–3.
2. The company table renders with all columns: company name, status summary (with tooltip on truncation), relative last activity time, stale badge, and stage/rejection/re-apply badge.
3. Clicking a table row expands the conversation view inline; clicking again (or clicking a different row) collapses it.
4. The conversation view displays messages in chronological order with correct left/right alignment for inbound/outbound emails.
5. Thread separators appear when the conversation spans multiple Gmail threads.
6. Long messages are truncated with a working "Show more" toggle.
7. The conversation view auto-scrolls to the most recent message on open.
8. The search input filters the table in real time (debounced).
9. The status filter dropdown and sort dropdown correctly filter and sort the table.
10. The sync status footer shows the last sync time and updates periodically.
11. Empty states render correctly for both "no data yet" and "no filter results" scenarios.
12. Stale rows have the amber left-border accent.
13. Rejected rows appear visually muted compared to active applications.
