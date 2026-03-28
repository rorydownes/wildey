# Milestone 3 — LLM-Powered Company Identification & Analysis

## Objective

Integrate Claude CLI into the ingestion pipeline to: identify which company each email belongs to, group emails into applications, generate 1-sentence status summaries, classify rejections into pre-interview and post-interview categories, compute staleness flags, and write all derived data to the `companies`, `applications`, and `email_application_map` tables. After this milestone, the database contains everything needed to power the dashboard.

---

## Step 1: Claude CLI Integration Layer

Implement `ingestion/llm_analyzer.py`:

### CLI Wrapper

```python
import subprocess
import json

class ClaudeAnalyzer:
    def __init__(self, cli_path: str, model: str, timeout: int, max_retries: int):
        self.cli_path = cli_path
        self.model = model
        self.timeout = timeout
        self.max_retries = max_retries

    def analyze(self, prompt: str) -> str:
        """
        Call Claude CLI with a prompt and return the response text.
        Retries on failure up to max_retries times.
        """
        for attempt in range(self.max_retries):
            try:
                result = subprocess.run(
                    [self.cli_path, "--model", self.model, "--print", "--prompt", prompt],
                    capture_output=True,
                    text=True,
                    timeout=self.timeout
                )

                if result.returncode != 0:
                    log(f"Claude CLI error (attempt {attempt+1}): {result.stderr}")
                    continue

                return result.stdout.strip()

            except subprocess.TimeoutExpired:
                log(f"Claude CLI timeout (attempt {attempt+1})")
                continue

        raise LLMAnalysisError(f"Claude CLI failed after {self.max_retries} attempts")
```

### Structured Output Parsing

All prompts should instruct Claude to return JSON. The wrapper should parse and validate the response:

```python
def analyze_json(self, prompt: str) -> dict:
    """Call Claude CLI and parse the response as JSON."""
    raw = self.analyze(prompt)

    # Handle cases where Claude wraps JSON in markdown code blocks
    cleaned = raw.strip()
    if cleaned.startswith("```"):
        cleaned = cleaned.split("\n", 1)[1]  # Remove first line
        cleaned = cleaned.rsplit("```", 1)[0]  # Remove last fence

    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        log(f"Failed to parse Claude response as JSON: {raw[:200]}")
        raise LLMAnalysisError("Invalid JSON response from Claude")
```

---

## Step 2: Company Identification

### Strategy

When new emails arrive (after backfill or incremental sync), the pipeline needs to determine which company each email is associated with. The process works in two stages:

1. **Domain-based pre-grouping**: Group unprocessed emails by sender domain (for inbound) or recipient domain (for outbound). This reduces LLM calls by batching emails likely from the same company.
2. **LLM identification**: For each domain group, send a representative sample to Claude to identify the company name.

### Implementation: `ingestion/company_resolver.py`

```python
def resolve_companies_for_new_emails(db_conn, analyzer: ClaudeAnalyzer):
    """
    Process all emails not yet mapped to an application.
    Group by domain, identify companies via LLM, create/match company records.
    """
    unmapped_emails = get_unmapped_emails(db_conn)

    # Group by the non-user email domain
    domain_groups = group_by_domain(unmapped_emails)

    for domain, emails in domain_groups.items():
        company_name = identify_company(analyzer, domain, emails)
        company_id = get_or_create_company(db_conn, company_name, domain)
        application_id = get_or_create_application(db_conn, company_id)

        for email in emails:
            map_email_to_application(db_conn, email['id'], application_id)
```

### Company Identification Prompt

```python
COMPANY_ID_PROMPT = """
You are analyzing job application emails. Given the following email metadata and content,
identify the company the applicant is interacting with.

Email domain: {domain}
Sample emails (up to 5):
{email_samples}

Respond with ONLY a JSON object:
{{
    "company_name": "The official company name (e.g., 'Google', 'Stripe', 'Acme Corp')",
    "confidence": "high" | "medium" | "low",
    "reasoning": "Brief explanation of how you identified the company"
}}

Rules:
- If the emails are from a recruiting platform (Greenhouse, Lever, Ashby, etc.),
  identify the HIRING company, not the platform.
- If the emails are from a third-party recruiter, identify the recruiting agency name.
- Use the most commonly recognized form of the company name (e.g., "Google" not "Alphabet Inc.").
- If you cannot determine the company with reasonable confidence, set company_name to
  "Unknown ({domain})" and confidence to "low".
"""
```

### Email Sample Selection

When preparing email samples for the prompt, select up to 5 emails from the domain group prioritizing:
1. The earliest email (likely the initial application or first recruiter outreach)
2. The most recent email
3. Emails with unique subject lines (to capture different threads)

For each sample, include: subject, sender name, first 300 characters of body text. Truncate aggressively — the LLM needs context, not full email bodies.

### Company Matching

When the LLM returns a company name, check for existing companies before creating a new one:

```python
def get_or_create_company(db_conn, name: str, domain: str) -> int:
    """
    Find an existing company by name (case-insensitive) or domain.
    Create a new one if no match found.
    Returns company_id.
    """
    # Try exact name match (case-insensitive)
    existing = db_conn.execute(
        "SELECT id FROM companies WHERE LOWER(name) = LOWER(?)", (name,)
    ).fetchone()
    if existing:
        return existing['id']

    # Try domain match
    if domain:
        existing = db_conn.execute(
            "SELECT id FROM companies WHERE domain = ?", (domain,)
        ).fetchone()
        if existing:
            return existing['id']

    # Create new company
    cursor = db_conn.execute(
        "INSERT INTO companies (name, domain) VALUES (?, ?)",
        (name, domain)
    )
    return cursor.lastrowid
```

---

## Step 3: Application Status Analysis

After emails are mapped to applications, analyze each application's full conversation to determine its current status.

### When to Run Analysis

Run status analysis for an application when:
- It has new emails since the last analysis (compare email timestamps against `applications.updated_at`).
- It was just created (new company identified).

Do NOT re-analyze applications with no new emails — this wastes LLM calls.

### Status Analysis Prompt

```python
STATUS_ANALYSIS_PROMPT = """
You are analyzing a job application email thread to determine its current status.
This could be for any type of job (engineering, sales, marketing, executive, etc.).

Applicant's email: {user_email}
Company: {company_name}

Email thread (chronological order):
{conversation_thread}

Analyze this conversation and respond with ONLY a JSON object:
{{
    "status_summary": "A single sentence describing the current state or most recent completed step.
                       Examples: 'Completed phone screen with recruiter, awaiting next steps',
                       'Received rejection after final round interview',
                       'Submitted application, no response yet',
                       'Offer extended, discussing compensation details'",
    "stage": One of: "applied" | "recruiter_screen" | "interviewing" | "offer" |
             "accepted" | "rejected_pre_interview" | "rejected_post_interview" |
             "withdrawn" | "unknown",
    "rejection_details": {{
        "is_rejected": true | false,
        "rejection_type": "pre_interview" | "post_interview" | null,
        "reasoning": "Explain why this is classified as pre or post interview rejection, or null"
    }},
    "last_sender": "user" | "company"
}}

Stage classification rules:
- "applied": Application submitted but no substantive response beyond auto-acknowledgment.
- "recruiter_screen": Interaction with a recruiter (scheduling, phone screen, initial chat)
  but NO technical or hiring-manager interviews yet.
- "interviewing": At least one non-recruiter interview has been scheduled or completed
  (technical screen, hiring manager, panel, onsite, etc.).
- "offer": An offer has been extended.
- "accepted": The applicant has accepted an offer.
- "rejected_pre_interview": The company rejected the applicant BEFORE any non-recruiter
  interview occurred. This includes rejections after recruiter screens.
- "rejected_post_interview": The company rejected the applicant AFTER at least one
  non-recruiter interview occurred.
- "withdrawn": The applicant withdrew from consideration.
- "unknown": Cannot determine status from available emails.

The distinction between pre-interview and post-interview rejection is critical.
A recruiter phone screen does NOT count as an interview for this classification.
Only interviews with engineers, hiring managers, or structured interview panels count.
"""
```

### Conversation Thread Formatting

When building the `{conversation_thread}` for the prompt, format each email as:

```
[2026-03-15 14:30 UTC] FROM: Jane Smith <jane@company.com>
SUBJECT: Re: Software Engineer Position
---
Hi Rory, thanks for your application. We'd love to schedule a phone screen...
---

[2026-03-16 09:15 UTC] FROM: You
SUBJECT: Re: Software Engineer Position
---
Thanks Jane, I'm available Thursday afternoon...
---
```

**Token management**: If the full conversation exceeds approximately 3000 words, truncate older emails to their first 100 words each and keep the 5 most recent emails in full. Always preserve the first email and last 5 emails in full — the beginning and end of the conversation are most informative.

---

## Step 4: Writing Analysis Results

After the LLM returns the status analysis, update the database:

```python
def update_application_from_analysis(db_conn, application_id: int, analysis: dict, config: dict):
    """
    Write LLM analysis results to the applications table.
    Compute derived fields (staleness, re-apply eligibility).
    """
    # Get last interaction timestamp from mapped emails
    last_email = db_conn.execute("""
        SELECT MAX(e.sent_at) as last_sent, e.direction
        FROM emails e
        JOIN email_application_map m ON m.email_id = e.id
        WHERE m.application_id = ?
        ORDER BY e.sent_at DESC LIMIT 1
    """, (application_id,)).fetchone()

    last_interaction_at = last_email['last_sent']

    # Compute staleness
    stale_threshold = datetime.utcnow() - timedelta(days=config['thresholds']['stale_days'])
    is_stale = 1 if last_interaction_at < stale_threshold else 0

    # Don't flag rejected/accepted/withdrawn applications as stale
    terminal_stages = ('rejected_pre_interview', 'rejected_post_interview',
                       'accepted', 'withdrawn')
    if analysis['stage'] in terminal_stages:
        is_stale = 0

    # Compute re-apply eligible date
    reapply_eligible_date = None
    rejection_date = None
    if analysis['rejection_details']['is_rejected']:
        rejection_date = last_interaction_at  # Approximate: use last email date

        if analysis['rejection_details']['rejection_type'] == 'pre_interview':
            reapply_eligible_date = rejection_date  # Immediately eligible
        elif analysis['rejection_details']['rejection_type'] == 'post_interview':
            cooldown = config['thresholds']['rejection_cooldown_months']
            reapply_eligible_date = rejection_date + timedelta(days=cooldown * 30)

    db_conn.execute("""
        UPDATE applications SET
            status_summary = ?,
            stage = ?,
            rejection_date = ?,
            reapply_eligible_date = ?,
            last_interaction_at = ?,
            last_sender = ?,
            is_stale = ?,
            updated_at = datetime('now')
        WHERE id = ?
    """, (
        analysis['status_summary'],
        analysis['stage'],
        rejection_date,
        reapply_eligible_date,
        last_interaction_at,
        analysis['last_sender'],
        is_stale,
        application_id
    ))
```

---

## Step 5: LLM Analysis Audit Logging

Every LLM call should be logged to `llm_analysis_log` for debugging:

```python
def log_analysis(db_conn, application_id: int, analysis_type: str,
                 input_summary: str, output_raw: str, model: str):
    db_conn.execute("""
        INSERT INTO llm_analysis_log
            (application_id, analysis_type, input_summary, output_raw, model_used)
        VALUES (?, ?, ?, ?, ?)
    """, (application_id, analysis_type, input_summary[:500], output_raw, model))
```

Log both company identification calls and status analysis calls. The `input_summary` should be a truncated version of what was sent (first 500 chars) — enough for debugging, not a full copy of the email content.

---

## Step 6: Integration into the Ingestion Pipeline

Modify `ingestion/worker.py` to add the LLM analysis step after email fetching:

```python
# Inside the main loop, after email fetch completes:

def run_sync_cycle(gmail, db_conn, analyzer, config, user_email):
    # Step 1: Fetch new emails (backfill or incremental — from Milestone 2)
    new_emails = fetch_new_emails(gmail, db_conn, config, user_email)

    if not new_emails:
        log("No new emails found.")
        return

    log(f"Processing {len(new_emails)} new emails through analysis pipeline...")

    # Step 2: Company identification for unmapped emails
    resolve_companies_for_new_emails(db_conn, analyzer)

    # Step 3: Status analysis for applications with new emails
    applications_to_analyze = get_applications_needing_analysis(db_conn)
    for app in applications_to_analyze:
        try:
            analyze_application_status(db_conn, analyzer, app, config, user_email)
        except LLMAnalysisError as e:
            log(f"Failed to analyze application {app['id']} ({app['company_name']}): {e}")
            # Continue with other applications — don't let one failure block all

    db_conn.commit()
    log("Analysis pipeline complete.")
```

### Analysis Ordering

The pipeline must run in this order:
1. **Email fetch** (Milestone 2 logic)
2. **Company identification** — assigns emails to companies/applications
3. **Status analysis** — analyzes complete conversations per application

This ordering ensures that all new emails are mapped before status analysis runs, so the analysis sees the complete conversation.

---

## Step 7: Handling Edge Cases

### Multiple Applications to Same Company

If a user applies to the same company months apart, emails may arrive from the same domain. Handle this by:
- If an existing application to a company is in a terminal state (`rejected_*`, `accepted`, `withdrawn`) AND the new email is more than 30 days after the last email in that application, create a new application record.
- Otherwise, map the email to the existing active application.

### Recruiter Platforms (Greenhouse, Lever, Ashby)

Emails from `no-reply@greenhouse.io` or similar platforms are common. The company identification prompt already handles this (instructing Claude to identify the hiring company, not the platform). However, the domain-based pre-grouping will group all Greenhouse emails together initially. To handle this:

1. If the domain is a known recruiting platform (maintain a small list: `greenhouse.io`, `lever.co`, `ashbyhq.com`, `jobs.workday.com`, etc.), skip domain-based grouping and instead group by Gmail thread ID.
2. Each thread from a recruiting platform likely represents a separate company.

```python
KNOWN_RECRUITING_PLATFORMS = {
    'greenhouse.io', 'lever.co', 'ashbyhq.com',
    'jobs.workday.com', 'myworkdayjobs.com',
    'smartrecruiters.com', 'icims.com',
    'jobvite.com', 'breezy.hr', 'hire.jazz.co'
}

def group_by_domain(emails: list, user_email: str) -> dict:
    groups = defaultdict(list)
    for email in emails:
        domain = extract_domain(email, user_email)
        if domain in KNOWN_RECRUITING_PLATFORMS:
            # Group by thread instead of domain
            key = f"__thread__{email['gmail_thread_id']}"
        else:
            key = domain
        groups[key].append(email)
    return groups
```

### Low-Confidence Company Identification

If the LLM returns `confidence: "low"`, still create the company record (with the `"Unknown (domain)"` name) and map the emails. The status analysis will still run and produce useful output. These can be improved in future milestones with manual override UI.

### Newsletters, Automated Notifications, Non-Application Emails

Not every email to the job application address will be a real application conversation. The user may receive job board newsletters, LinkedIn notifications, or spam. The company identification prompt should handle this, but add a post-processing filter:

- If Claude identifies the "company" as a job board, newsletter, or notification service, do not create an application for it. Create the company record but skip the application mapping.
- Add a stage value of `'unknown'` to catch these cases and filter them out in the dashboard.

---

## Step 8: Staleness Recomputation

Staleness is time-dependent and must be recomputed periodically, not just when new emails arrive. Add a step to the beginning of each sync cycle:

```python
def recompute_staleness(db_conn, config):
    """
    Update is_stale flag for all non-terminal applications.
    Run this every sync cycle regardless of whether new emails arrived.
    """
    stale_days = config['thresholds']['stale_days']

    db_conn.execute("""
        UPDATE applications
        SET is_stale = CASE
            WHEN last_interaction_at < datetime('now', ? || ' days')
            THEN 1 ELSE 0
        END,
        updated_at = datetime('now')
        WHERE stage NOT IN (
            'rejected_pre_interview', 'rejected_post_interview',
            'accepted', 'withdrawn'
        )
    """, (f"-{stale_days}",))

    # Ensure terminal applications are never stale
    db_conn.execute("""
        UPDATE applications SET is_stale = 0
        WHERE stage IN (
            'rejected_pre_interview', 'rejected_post_interview',
            'accepted', 'withdrawn'
        ) AND is_stale = 1
    """)
```

---

## Acceptance Criteria

This milestone is complete when:

1. After the ingestion worker runs with real Gmail data, the `companies` table contains correctly identified company names (not email domains or platform names).
2. Every email in the `emails` table is mapped to exactly one application via `email_application_map`.
3. Every application has a `status_summary` (1-sentence text) and a valid `stage` value.
4. Rejections are correctly classified as `rejected_pre_interview` or `rejected_post_interview`, with appropriate `reapply_eligible_date` values.
5. Applications with no email activity in the configured stale period are flagged with `is_stale = 1`.
6. Terminal applications (rejected, accepted, withdrawn) are never flagged as stale.
7. The `llm_analysis_log` table contains a record of every LLM call made.
8. Emails from known recruiting platforms (Greenhouse, Lever, etc.) are correctly attributed to the hiring company.
9. The analysis pipeline is resilient — a single failed LLM call does not block processing of other applications.
10. Running the worker multiple times does not create duplicate companies, applications, or mappings.
