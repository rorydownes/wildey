# Milestone 2 — Gmail OAuth2 & Email Ingestion

## Objective

Implement the complete Gmail integration: OAuth2 authentication flow, the standalone ingestion worker that polls for new emails on a configurable interval, initial 3-month historical backfill, incremental sync via Gmail history API, and writing raw email data to SQLite. No LLM analysis in this milestone — emails are fetched, parsed, and stored only.

---

## Step 1: Google Cloud Project Setup Instructions

The ingestion worker needs a Google Cloud OAuth2 client. Document the following setup steps in a `credentials/SETUP.md` file that the user follows once:

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (e.g., "Job Application Tracker").
3. Enable the **Gmail API** under APIs & Services → Library.
4. Configure the **OAuth consent screen**: set to "External" user type, add the email address being tracked as a test user.
5. Create **OAuth 2.0 Client ID** credentials: application type = "Desktop app".
6. Download the credentials JSON file and save it as `credentials/client_secret.json`.

**Required OAuth scopes**:
```
https://www.googleapis.com/auth/gmail.readonly
https://www.googleapis.com/auth/gmail.metadata
```

Read-only access is sufficient. The application never sends or modifies emails.

---

## Step 2: OAuth2 Authentication Flow

Implement in `ingestion/gmail_client.py`:

### Token Management

```python
# Pseudocode structure
class GmailClient:
    def __init__(self, credentials_dir: str):
        self.credentials_dir = credentials_dir
        self.token_path = os.path.join(credentials_dir, "token.json")
        self.client_secret_path = os.path.join(credentials_dir, "client_secret.json")
        self.service = None

    def authenticate(self):
        """
        Load existing token or initiate OAuth flow.
        On first run, opens browser for user consent.
        On subsequent runs, uses/refreshes stored token.
        """
        ...

    def _build_service(self, creds):
        """Build the Gmail API service object."""
        ...
```

**Implementation details**:
- Use `google-auth-oauthlib` `InstalledAppFlow` for the initial browser-based consent flow.
- Store the resulting token at `credentials/token.json`.
- On subsequent runs, load the token, check `creds.expired`, and refresh using `creds.refresh(Request())` if needed.
- If the refresh fails (revoked token), delete `token.json` and re-run the full OAuth flow.
- The authentication step must happen before the polling loop starts. If auth fails, the worker should exit with a clear error message, not enter the loop.

---

## Step 3: Email Fetching — Initial Backfill

Implement the backfill logic that runs once on first startup (when `sync_state.initial_backfill_complete = 0`).

### Approach

Use the Gmail API `users.messages.list` endpoint with a date-based query:

```python
def backfill(self, months: int):
    """
    Fetch all messages from the last N months.
    """
    cutoff_date = datetime.utcnow() - timedelta(days=months * 30)
    query = f"after:{cutoff_date.strftime('%Y/%m/%d')}"

    messages = []
    page_token = None

    while True:
        result = self.service.users().messages().list(
            userId='me',
            q=query,
            pageToken=page_token,
            maxResults=100
        ).execute()

        messages.extend(result.get('messages', []))
        page_token = result.get('nextPageToken')

        if not page_token:
            break

    return messages  # List of {id, threadId} dicts
```

### Rate Limiting and Batching

- Gmail API has a quota of ~250 quota units per second per user.
- `messages.list` costs 5 units, `messages.get` costs 5 units.
- Implement batching: use `BatchHttpRequest` to fetch message details in batches of 50.
- Add a configurable delay between batches (default: 0.5 seconds) to stay well within limits.
- Log progress during backfill: "Fetched 150/423 messages..."

### After Backfill Completes

- Record the most recent `historyId` from any fetched message into `sync_state.last_history_id`.
- Set `sync_state.initial_backfill_complete = 1`.
- Set `sync_state.last_sync_at` to current UTC timestamp.

---

## Step 4: Email Fetching — Incremental Sync

After backfill is complete, all subsequent poll cycles use the Gmail History API for efficient delta fetching.

### Approach

```python
def incremental_sync(self, last_history_id: str):
    """
    Fetch only messages that changed since last_history_id.
    Returns list of new message IDs to fetch.
    """
    new_message_ids = []
    page_token = None

    while True:
        result = self.service.users().history().list(
            userId='me',
            startHistoryId=last_history_id,
            historyTypes=['messageAdded'],
            pageToken=page_token
        ).execute()

        for history_record in result.get('history', []):
            for msg_added in history_record.get('messagesAdded', []):
                new_message_ids.append(msg_added['message']['id'])

        page_token = result.get('nextPageToken')
        if not page_token:
            break

    # Update stored history ID
    new_history_id = result.get('historyId', last_history_id)
    return new_message_ids, new_history_id
```

### History ID Expiration

Gmail history IDs expire after approximately one week. If the API returns a `404` on the history request:

1. Log a warning: "History ID expired, falling back to full resync."
2. Re-run the backfill logic (it's idempotent due to `UNIQUE` constraint on `gmail_message_id`).
3. Store the new history ID.

---

## Step 5: Email Parsing and Storage

For each fetched message ID, retrieve the full message and parse it into a row for the `emails` table.

### Fetching Message Detail

```python
def get_message(self, message_id: str):
    """Fetch full message detail with plain text body."""
    return self.service.users().messages().get(
        userId='me',
        id=message_id,
        format='full'
    ).execute()
```

### Parsing Logic

Implement `ingestion/email_parser.py`:

```python
def parse_message(raw_message: dict, user_email: str) -> dict:
    """
    Extract structured fields from a Gmail API message resource.
    Returns a dict matching the `emails` table columns.
    """
    headers = {h['name'].lower(): h['value']
               for h in raw_message['payload']['headers']}

    return {
        'gmail_message_id': raw_message['id'],
        'gmail_thread_id': raw_message['threadId'],
        'gmail_history_id': raw_message.get('historyId'),
        'subject': headers.get('subject', ''),
        'sender_address': extract_email(headers.get('from', '')),
        'sender_name': extract_name(headers.get('from', '')),
        'recipient_addresses': json.dumps(parse_recipients(headers)),
        'body_text': extract_plain_text_body(raw_message['payload']),
        'sent_at': parse_gmail_timestamp(raw_message['internalDate']),
        'direction': determine_direction(headers, user_email),
        'raw_headers': json.dumps({
            'from': headers.get('from'),
            'to': headers.get('to'),
            'cc': headers.get('cc'),
            'date': headers.get('date'),
            'message-id': headers.get('message-id'),
            'in-reply-to': headers.get('in-reply-to'),
        }),
    }
```

### Body Text Extraction

Gmail messages have a nested MIME structure. Implement `extract_plain_text_body` to handle:

1. **Simple text/plain messages**: Return the body directly (base64url decoded).
2. **Multipart messages**: Walk the `parts` tree recursively, prefer `text/plain` over `text/html`.
3. **HTML-only messages**: If no `text/plain` part exists, extract text from `text/html` using a simple tag-stripping approach (regex or `html.parser`). Do not install heavy dependencies like BeautifulSoup for this.
4. **Empty body**: Return empty string, don't fail.

All body text should be decoded from base64url using `base64.urlsafe_b64decode`.

### Direction Detection

```python
def determine_direction(headers: dict, user_email: str) -> str:
    """
    Determine if this email was sent or received by the user.
    """
    sender = extract_email(headers.get('from', ''))
    if sender.lower() == user_email.lower():
        return 'outbound'
    return 'inbound'
```

The user's email address should be detected once at startup by calling `users.getProfile` and stored for the session.

### Recipient Parsing

```python
def parse_recipients(headers: dict) -> list[str]:
    """Parse To and CC headers into a list of email addresses."""
    recipients = []
    for field in ['to', 'cc']:
        if headers.get(field):
            recipients.extend(extract_all_emails(headers[field]))
    return recipients
```

---

## Step 6: Database Write Layer

Implement `ingestion/db_writer.py`:

### Upsert Emails

```python
def upsert_email(conn, email_data: dict):
    """
    Insert an email record. Skip silently if gmail_message_id already exists.
    """
    conn.execute("""
        INSERT OR IGNORE INTO emails (
            gmail_message_id, gmail_thread_id, gmail_history_id,
            subject, sender_address, sender_name, recipient_addresses,
            body_text, sent_at, direction, raw_headers
        ) VALUES (
            :gmail_message_id, :gmail_thread_id, :gmail_history_id,
            :subject, :sender_address, :sender_name, :recipient_addresses,
            :body_text, :sent_at, :direction, :raw_headers
        )
    """, email_data)
```

`INSERT OR IGNORE` ensures idempotency — re-processing the same email (during backfill recovery or history ID expiration) is safe.

### Update Sync State

```python
def update_sync_state(conn, history_id: str, backfill_complete: bool = None):
    """Update the singleton sync_state row."""
    updates = ["last_history_id = ?", "last_sync_at = datetime('now')"]
    params = [history_id]

    if backfill_complete is not None:
        updates.append("initial_backfill_complete = ?")
        params.append(1 if backfill_complete else 0)

    conn.execute(
        f"UPDATE sync_state SET {', '.join(updates)} WHERE id = 1",
        params
    )
```

### Transaction Boundaries

- Each poll cycle (backfill or incremental) should write all fetched emails within a single transaction.
- Commit after all emails in the batch are inserted.
- If any error occurs mid-batch, rollback the entire transaction and retry on the next poll cycle.
- The `sync_state` update should be part of the same transaction so the history ID is only advanced after emails are committed.

---

## Step 7: Main Worker Loop

Implement the complete `ingestion/worker.py`:

```python
def main():
    config = load_config()
    db_conn = connect_db(config)
    gmail = GmailClient(config['gmail']['credentials_dir'])
    gmail.authenticate()

    user_email = gmail.get_user_email()  # calls users.getProfile
    log(f"Authenticated as {user_email}")

    while True:
        try:
            sync_state = get_sync_state(db_conn)

            if not sync_state['initial_backfill_complete']:
                log("Starting initial backfill...")
                run_backfill(gmail, db_conn, config, user_email)
                log("Backfill complete.")
            else:
                log("Running incremental sync...")
                new_count = run_incremental_sync(gmail, db_conn, sync_state, user_email)
                log(f"Incremental sync complete. {new_count} new emails.")

        except HttpError as e:
            if e.resp.status == 404:
                log("History ID expired, will re-backfill on next cycle.")
                reset_backfill_state(db_conn)
            else:
                log(f"Gmail API error: {e}")

        except Exception as e:
            log(f"Unexpected error: {e}")

        interval = config['gmail']['polling_interval_seconds']
        log(f"Sleeping {interval} seconds until next sync...")
        time.sleep(interval)
```

### Graceful Shutdown

```python
shutdown_requested = False

def signal_handler(signum, frame):
    global shutdown_requested
    shutdown_requested = True
    log("Shutdown requested, will exit after current cycle.")

signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)

# In the loop, check: if shutdown_requested: break
```

---

## Step 8: Logging

All ingestion worker logging should use Python's `logging` module with a consistent format:

```python
logging.basicConfig(
    level=logging.INFO,
    format='[ingestion] %(asctime)s %(levelname)s: %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
```

Log at minimum:
- Authentication success/failure
- Backfill start, progress (every 50 messages), and completion with total count
- Each incremental sync cycle: start, number of new messages found, completion
- Any API errors with full details
- Sleep/wake transitions
- Shutdown signals

---

## Step 9: Error Handling and Resilience

### Gmail API Errors
- **401 Unauthorized**: Token expired and refresh failed. Log error, attempt re-auth, and if that fails, exit with instructions to re-run OAuth setup.
- **403 Rate Limit**: Back off exponentially. Start at 1 second, double up to 60 seconds, then log a warning and continue the polling loop.
- **404 History ID expired**: Reset `initial_backfill_complete` to 0 and re-run backfill on next cycle.
- **500/503 Server errors**: Log and retry on next poll cycle. Don't crash.

### Database Errors
- If SQLite is locked (from concurrent API server reads), retry the write up to 3 times with a 1-second delay. SQLite WAL mode should minimize this.
- If the database file is missing or corrupt, exit with a clear error directing the user to run `init_db.py`.

### Network Errors
- Catch `ConnectionError`, `TimeoutError`, etc. Log and continue to next poll cycle.

---

## Acceptance Criteria

This milestone is complete when:

1. Running the worker for the first time opens a browser for Gmail OAuth consent, successfully authenticates, and stores the token.
2. The worker performs a full backfill fetching at least 3 months of emails, writing them all to the `emails` table.
3. Subsequent poll cycles use the History API and only fetch genuinely new messages.
4. Re-running the worker after stopping it picks up where it left off (no duplicate emails, history ID preserved).
5. The `sync_state` table accurately reflects the last sync time and backfill status.
6. Email direction (inbound/outbound) is correctly determined for all messages.
7. Plain text body extraction works for simple text, multipart, and HTML-only emails.
8. The worker handles API errors gracefully (rate limits, expired history IDs, network issues) without crashing.
9. The worker shuts down cleanly on SIGINT/SIGTERM after finishing the current operation.
10. At this point, no `email_application_map`, `companies`, or `applications` rows exist — that mapping is Milestone 3.
