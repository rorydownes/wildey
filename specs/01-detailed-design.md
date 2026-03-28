# Job Application Tracker — Detailed Design Specification

## Document Purpose

This document describes every screen, component, and user interaction in the Job Application Tracker dashboard. It is intended for consumption by a designer (or design-generation tool) to produce complete UI mockups. Pair this with `00-project-overview.md` for full architectural and technical context.

The application is a single-page web app that runs locally in the browser. There is no user authentication on the dashboard itself — it is a single-user local tool.

---

## Screen Map

The application consists of two primary screens and one one-time setup flow:

```
┌─────────────────────────────────────────────┐
│  Setup Screen (one-time OAuth flow)         │
│  └─► redirects to Dashboard on completion   │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Dashboard Screen                           │
│  ├── Summary Stats Bar                      │
│  ├── Filter & Sort Controls                 │
│  ├── Company Table                          │
│  │   └── Expanded Row: Conversation View    │
│  └── Sync Status Footer                     │
└─────────────────────────────────────────────┘
```

---

## Screen 1: Setup Screen

### When It Appears

This screen appears only on first launch, before Gmail OAuth credentials have been authorized. Once setup is complete, the user never sees this screen again unless they manually reset credentials.

### Layout

A centered card on a neutral background. The card contains:

1. **App title and tagline**: "Job Application Tracker" with a short subtitle like "Connect your Gmail to get started."
2. **Explanation text**: A brief sentence explaining that the app will read emails from their connected Gmail account to track job applications. Emphasize that it runs locally and no data leaves their machine.
3. **Connect Gmail button**: A prominent primary-action button labeled "Connect Gmail Account". Clicking this initiates the OAuth2 flow, which opens a Google consent screen in a new browser tab/window.
4. **Status indicator**: After the user completes the OAuth flow in the Google tab, this area updates to show a success state with a checkmark and the connected email address.
5. **Continue to Dashboard button**: Appears only after successful connection. Takes the user to the Dashboard screen.

### User Actions

| Action | Behavior |
|---|---|
| Click "Connect Gmail Account" | Opens Google OAuth consent screen in new tab. User authorizes access. On completion, the setup screen updates to show success. |
| Click "Continue to Dashboard" | Navigates to the Dashboard screen. The ingestion worker begins its initial 3-month backfill in the background. |

### Edge Cases

- If the user closes the Google tab without authorizing, the setup screen remains in its initial state with the Connect button still available.
- If OAuth token is later revoked or expires, the Dashboard screen should show a reconnection prompt (a banner at the top of the Dashboard, not a redirect back to this screen).

---

## Screen 2: Dashboard Screen

This is the primary and only ongoing screen of the application. It consists of four vertically stacked sections: Summary Stats Bar, Filter & Sort Controls, Company Table, and Sync Status Footer.

---

### Section 2A: Summary Stats Bar

**Position**: Top of the dashboard, full width. Displayed as a horizontal row of stat cards.

**Stat Cards** (each card shows a label and a prominent number):

| Card | Label | Value | Notes |
|---|---|---|---|
| 1 | Active Applications | Count | Applications that are not rejected and not explicitly closed. |
| 2 | Awaiting Response | Count | Applications where the user sent the most recent email and has not received a reply. |
| 3 | Interviews Scheduled | Count | Applications where the LLM-determined status indicates an upcoming interview stage. |
| 4 | Stale (7+ days) | Count | Applications with no email activity in 7+ days. Display in a warning color (amber/orange). |
| 5 | Rejections | Count | Total rejected applications. Shown in a muted/neutral color — this is informational, not alarming. |
| 6 | Ready to Re-apply | Count | Companies where rejection cooldown has elapsed (immediate for pre-interview rejections, 3 months for post-interview). |

**Visual Treatment**: Cards should be compact, scannable at a glance. Use a consistent card style with the label in small secondary text above a large bold number. The "Stale" card should have an amber/orange accent to draw attention. All other cards use neutral styling.

**Interactions**: Clicking a stat card filters the Company Table below to show only the relevant subset. Clicking the same card again clears the filter. Only one stat filter can be active at a time. When a filter is active, the clicked card should have a visually distinct selected/active state (e.g., highlighted border or background).

---

### Section 2B: Filter & Sort Controls

**Position**: Directly below the Summary Stats Bar, above the Company Table. Displayed as a single horizontal toolbar row.

**Controls**:

1. **Search input**: A text field with placeholder text "Search companies...". Filters the table in real time as the user types. Matches against company name and status summary text.

2. **Status filter dropdown**: A dropdown (or segmented control) with the following options:
   - All (default)
   - Active
   - Interviewing
   - Offered
   - Rejected — Pre-Interview
   - Rejected — Post-Interview
   - Stale

3. **Sort dropdown**: Controls table sort order. Options:
   - Last Activity (newest first) — default
   - Last Activity (oldest first)
   - Company Name (A–Z)
   - Company Name (Z–A)

**Interactions**: All filters and sort controls apply immediately. They combine with any active stat card filter (Section 2A). If a stat card filter is active, show a dismissible chip/pill next to the controls indicating the active filter (e.g., "Showing: Stale" with an × to clear).

---

### Section 2C: Company Table

**Position**: The main body of the dashboard, below the Filter & Sort Controls. Takes up the majority of the viewport.

**Table Structure — Collapsed Row (default state)**:

Each row represents one company and displays these columns:

| Column | Content | Width | Notes |
|---|---|---|---|
| Company Name | Text | ~25% | Primary identifier. Bold text. |
| Status | Text | ~35% | The 1-sentence LLM-generated status summary. Truncate with ellipsis if it exceeds the available width. Full text visible on hover via tooltip. |
| Last Activity | Relative date | ~15% | e.g., "2 hours ago", "3 days ago", "Jan 14". Use relative time for recent dates (<7 days), absolute date for older. |
| Stale Indicator | Icon or badge | ~10% | Visible only when the application has had no activity in 7+ days. Display as a small amber/orange badge or warning icon with the text "Stale". Hidden (blank cell) when not stale. |
| Rejection / Re-apply | Badge or text | ~15% | Blank for active applications. For rejections, show a badge: "Rejected — Pre-Interview" or "Rejected — Post-Interview". If the cooldown has elapsed, show "Ready to Re-apply" in a green badge instead of the rejection badge. |

**Visual Treatment for Rows**:

- Default rows have a clean white background with subtle bottom borders.
- Stale rows: A very subtle amber/orange left-border accent (2–3px) in addition to the stale badge in the column, so stale applications are visible even when scanning quickly.
- Rejected rows: Slightly muted text color (not fully grayed out, but visually recessed compared to active applications).
- "Ready to Re-apply" rows: The green badge in the last column provides sufficient visual distinction.
- Hover state: Light background highlight on the entire row, with the cursor changing to pointer to indicate the row is clickable.

**Empty States**:

- If the table has no data at all (initial backfill still running): Show a centered message: "Syncing your emails... This may take a few minutes on first run." with a subtle loading spinner.
- If filters produce no results: Show "No applications match your filters." with a link/button to clear all filters.

---

### Section 2D: Expanded Row — Conversation View

**Trigger**: Clicking any row in the Company Table expands it inline, pushing subsequent rows down. Only one row can be expanded at a time — clicking a different row collapses the currently expanded one and expands the new one. Clicking the same expanded row collapses it.

**Layout**: The expanded area appears directly below the clicked row, contained within the table's visual boundary. It has a slightly different background color (e.g., very light gray) to visually distinguish it from the table rows.

**Expanded Area Structure** (top to bottom):

#### 2D-1: Company Header

A compact header within the expanded area showing:

- **Company name** (large, bold)
- **Current status summary** (full text, not truncated — the LLM-generated 1-sentence status)
- **Application stage badge**: A small colored badge showing the general stage (e.g., "Applied", "Phone Screen", "Interviewing", "Offer", "Rejected — Pre-Interview", "Rejected — Post-Interview", "Ready to Re-apply")
- **Last activity date**: Full timestamp of most recent email

#### 2D-2: Conversation Thread

A vertically scrollable area (with a max height, e.g., 500px, before internal scrolling activates) displaying all emails associated with this company in chronological order (oldest at top, newest at bottom).

**Message Bubble Design**:

Each email is rendered as a chat-style message bubble:

- **Received emails** (from the company/recruiter): Left-aligned bubbles with a light gray background.
- **Sent emails** (from the user): Right-aligned bubbles with a blue/brand-colored background and white text.

**Each bubble contains**:

1. **Sender name or label**: Small secondary text above the bubble. For received emails, show the sender's name (parsed from the From header). For sent emails, show "You".
2. **Email subject line**: Displayed as a small bold line at the top of the bubble content, only if the subject differs from the previous message in the thread (to avoid repeating the same subject on every reply).
3. **Email body text**: The plain text content of the email. Render line breaks faithfully. Do not render HTML markup — show the plain text version. If the email body is very long (more than ~300 words), show the first ~200 words with a "Show more" toggle that expands the bubble to reveal the full content.
4. **Timestamp**: Small secondary text below the bubble showing the date and time the email was sent (e.g., "Mar 15, 2026 at 2:34 PM").

**Thread Separators**: If the conversation spans multiple Gmail threads (different subject lines), insert a subtle horizontal divider with the new thread's subject line centered on it, similar to how chat apps show date separators.

**Scroll Behavior**: When the expanded area first opens, auto-scroll to the bottom (most recent message) so the user immediately sees the latest interaction.

#### 2D-3: Collapse Control

A subtle "Collapse" link or up-arrow icon at the bottom of the expanded area. Clicking it collapses the row back to its default state. The user can also click the row header itself to collapse.

---

### Section 2E: Sync Status Footer

**Position**: Fixed to the bottom of the viewport, full width. A thin, unobtrusive bar.

**Content**:

- **Left side**: "Last synced: [relative time]" (e.g., "Last synced: 3 minutes ago"). Updates automatically.
- **Right side**: If the ingestion worker is currently running a sync cycle, show a small spinner with "Syncing..." text. Otherwise, show nothing on the right side.

**Visual Treatment**: This footer should be minimal and non-distracting — small text, muted colors, thin height (30–40px). It provides confidence that data is fresh without demanding attention.

---

## Interaction Summary

| User Action | Result |
|---|---|
| Click stat card | Filters the Company Table to the relevant subset. Clicking again clears the filter. |
| Type in search box | Real-time filtering of the Company Table by company name or status text. |
| Change status filter dropdown | Filters table to show only applications matching the selected status. |
| Change sort dropdown | Re-sorts the table by the selected criterion. |
| Click a company row | Expands the row to show the conversation view. Collapses any other expanded row. |
| Click an expanded row or "Collapse" | Collapses the expanded conversation view. |
| Click "Show more" in a long message | Expands the message bubble to show the full email body. |
| Hover over a truncated status cell | Shows a tooltip with the full status summary text. |
| Click "Clear filters" in empty state | Resets all filters and shows the full table. |
| Dismiss filter chip | Removes the stat card filter, returning to the full table view. |

---

## Visual Design Notes

**Overall Aesthetic**: Clean, functional, and data-dense without feeling cluttered. Think "productivity tool" — closer to Linear or Notion than a consumer app. Neutral color palette with selective use of color for status indicators.

**Color Usage**:

- Primary/brand color (blue): Used for sent message bubbles, active stat card highlights, and primary action buttons.
- Amber/orange: Reserved exclusively for "stale" indicators (badge + row accent).
- Green: Used for "Ready to Re-apply" badges.
- Red/muted red: Used for rejection badges. Keep it subtle — muted, not alarming.
- Gray scale: All other UI elements (backgrounds, borders, secondary text).

**Typography**:

- One font family throughout (system font stack or a clean sans-serif like Inter).
- Clear hierarchy: Large bold for company names and stat numbers, regular weight for body text and status summaries, small muted text for timestamps and labels.

**Responsive Behavior**: The application is designed for desktop-width browser windows (1200px+). It does not need to be mobile-responsive for the MVP — it will only be used locally on a laptop/desktop. However, the layout should handle window resizes gracefully down to ~1000px width without breaking.

---

## State Considerations for Design

The designer should account for these distinct states when creating mockups:

1. **First run / backfill in progress**: Dashboard is visible but the table shows a loading/syncing message. Summary stats show zeros or dashes. The sync footer shows active syncing status.

2. **Normal state with data**: All sections populated. Mix of active, stale, and rejected applications in the table. At least one row expanded to show the conversation view.

3. **Filtered state**: A stat card is selected (highlighted), the table shows a subset of applications, and a filter chip is visible near the filter controls.

4. **Empty filter results**: Filters applied but no matching applications. Table area shows the "no results" empty state message.

5. **All applications rejected / no active applications**: Summary stats reflect this (Active = 0). Table still shows all companies but all rows have rejection badges. "Ready to Re-apply" badges appear on eligible ones.

6. **OAuth reconnection needed**: A dismissible warning banner appears at the very top of the Dashboard (above the Summary Stats Bar) with text like "Gmail connection expired. Reconnect to resume syncing." and a "Reconnect" button that re-initiates the OAuth flow.
