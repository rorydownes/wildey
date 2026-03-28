# Milestone 5 — Dashboard Summary Stats & Polish

## Objective

Wire up the summary stats bar at the top of the dashboard with live data, implement stat-card click-to-filter interaction, and perform a visual polish pass across the entire UI. This milestone is a nice-to-have — only start it after Milestones 1–4 are fully functional.

---

## Step 1: Stats Bar Component

The `GET /api/stats` endpoint was already implemented in Milestone 4. This step wires it to the React frontend.

### `StatsBar.jsx`

```jsx
function StatsBar({ activeStatFilter, onStatFilterChange }) {
    const [stats, setStats] = useState(null);

    useEffect(() => {
        async function load() {
            const data = await getStats();
            setStats(data);
        }
        load();
        // Refresh stats every 60 seconds
        const interval = setInterval(load, 60000);
        return () => clearInterval(interval);
    }, []);

    if (!stats) return <StatsBarSkeleton />;

    const cards = [
        { key: 'active', label: 'Active', value: stats.active_applications, filterValue: 'active' },
        { key: 'awaiting', label: 'Awaiting Response', value: stats.awaiting_response, filterValue: 'awaiting' },
        { key: 'interviewing', label: 'Interviews', value: stats.interviews_scheduled, filterValue: 'interviewing' },
        { key: 'stale', label: 'Stale (7+ days)', value: stats.stale_count, filterValue: 'stale', variant: 'warning' },
        { key: 'rejected', label: 'Rejections', value: stats.total_rejections, filterValue: 'rejected', variant: 'muted' },
        { key: 'reapply', label: 'Ready to Re-apply', value: stats.ready_to_reapply, filterValue: 'reapply', variant: 'success' },
    ];

    return (
        <div className="stats-bar">
            {cards.map(card => (
                <StatCard
                    key={card.key}
                    label={card.label}
                    value={card.value}
                    variant={card.variant}
                    isActive={activeStatFilter === card.filterValue}
                    onClick={() => onStatFilterChange(
                        activeStatFilter === card.filterValue ? null : card.filterValue
                    )}
                />
            ))}
        </div>
    );
}
```

### `StatCard.jsx`

```jsx
function StatCard({ label, value, variant, isActive, onClick }) {
    const variantClass = variant ? `stat-card--${variant}` : '';
    const activeClass = isActive ? 'stat-card--active' : '';

    return (
        <button
            className={`stat-card ${variantClass} ${activeClass}`}
            onClick={onClick}
        >
            <span className="stat-value">{value}</span>
            <span className="stat-label">{label}</span>
        </button>
    );
}
```

---

## Step 2: Stat Card Click-to-Filter Integration

### Filter Mapping

When a stat card is clicked, it sets a filter that maps to API query parameters. This mapping lives in `Dashboard.jsx`:

```javascript
function resolveFiltersFromStatCard(statFilter) {
    switch (statFilter) {
        case 'active':
            return {
                stage: 'applied,recruiter_screen,interviewing,offer',
            };
        case 'awaiting':
            return {
                stage: 'applied,recruiter_screen,interviewing,offer',
                // The API needs a dedicated param or the frontend post-filters.
                // Simplest: add an `awaiting_response=true` API param.
                awaiting_response: true,
            };
        case 'interviewing':
            return { stage: 'interviewing' };
        case 'stale':
            return { is_stale: true };
        case 'rejected':
            return { stage: 'rejected_pre_interview,rejected_post_interview' };
        case 'reapply':
            return { reapply_eligible: true };
        default:
            return {};
    }
}
```

### "Awaiting Response" API Support

The Milestone 4 API definition does not include an `awaiting_response` filter. Add it now:

**Backend change** — in `backend/app/routes/applications.py`, add query parameter `awaiting_response: bool = None`:

```python
if filters.get('awaiting_response'):
    query += """
        AND a.last_sender = 'user'
        AND a.stage NOT IN (
            'rejected_pre_interview', 'rejected_post_interview',
            'accepted', 'withdrawn'
        )
    """
```

### Active Filter Chip

When a stat card filter is active, display a dismissible chip in the `FilterControls` area:

```jsx
function FilterChip({ label, onDismiss }) {
    return (
        <span className="filter-chip">
            Showing: {label}
            <button className="chip-dismiss" onClick={onDismiss}>×</button>
        </span>
    );
}
```

The chip label maps from the stat filter key:
```javascript
const chipLabels = {
    active: 'Active Applications',
    awaiting: 'Awaiting Response',
    interviewing: 'Interviews',
    stale: 'Stale (7+ days)',
    rejected: 'Rejections',
    reapply: 'Ready to Re-apply',
};
```

### Filter Combination Behavior

Stat card filters combine with the search input and sort controls, but override the status dropdown filter. When a stat card is active:
- The status dropdown should be disabled or reset to "All" (since the stat card implies its own stage filter).
- Search still applies on top of the stat filter.
- Sort still applies.

Clicking a stat card while one is already active replaces the active filter (not additive).

---

## Step 3: Stats Bar Skeleton Loading State

While stats are loading, show placeholder cards to prevent layout shift:

```jsx
function StatsBarSkeleton() {
    return (
        <div className="stats-bar">
            {[...Array(6)].map((_, i) => (
                <div key={i} className="stat-card stat-card--skeleton">
                    <span className="stat-value skeleton-pulse">--</span>
                    <span className="stat-label skeleton-pulse">&nbsp;</span>
                </div>
            ))}
        </div>
    );
}
```

```css
.skeleton-pulse {
    background: linear-gradient(90deg, var(--color-bg-secondary) 25%, #e5e7eb 50%, var(--color-bg-secondary) 75%);
    background-size: 200% 100%;
    animation: pulse 1.5s ease-in-out infinite;
    border-radius: 4px;
    color: transparent;
}

@keyframes pulse {
    0% { background-position: 200% 0; }
    100% { background-position: -200% 0; }
}
```

---

## Step 4: Stats Auto-Refresh After Table Filter Changes

When the user interacts with the table (expanding rows, changing filters), the stats should remain accurate. Since the ingestion worker could update data at any time, refresh stats when:

1. The component mounts (initial load).
2. Every 60 seconds (background poll).
3. When the browser tab regains focus (user switches back to the dashboard):

```javascript
useEffect(() => {
    function handleVisibilityChange() {
        if (!document.hidden) {
            loadStats();
        }
    }
    document.addEventListener('visibilitychange', handleVisibilityChange);
    return () => document.removeEventListener('visibilitychange', handleVisibilityChange);
}, []);
```

---

## Step 5: Visual Polish — Stats Bar Styling

```css
.stats-bar {
    display: grid;
    grid-template-columns: repeat(6, 1fr);
    gap: 12px;
    padding: 20px 24px;
    border-bottom: 1px solid var(--color-border);
}

.stat-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 16px 12px;
    background: var(--color-bg);
    border: 1px solid var(--color-border);
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.15s ease;
    /* Reset button styles */
    font-family: inherit;
    text-align: center;
}

.stat-card:hover {
    border-color: var(--color-primary);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
}

.stat-card--active {
    border-color: var(--color-primary);
    background: rgba(37, 99, 235, 0.04);
    box-shadow: 0 0 0 1px var(--color-primary);
}

.stat-card--warning .stat-value { color: var(--color-stale); }
.stat-card--success .stat-value { color: var(--color-success); }
.stat-card--muted .stat-value { color: var(--color-text-secondary); }

.stat-value {
    font-size: 1.75rem;
    font-weight: 700;
    line-height: 1;
    color: var(--color-text);
}

.stat-label {
    font-size: 0.75rem;
    color: var(--color-text-secondary);
    margin-top: 6px;
    white-space: nowrap;
}
```

---

## Step 6: Visual Polish — Global Improvements

This step covers UI refinements across all components built in Milestone 4.

### Typography and Spacing Audit

- Verify consistent use of the CSS custom properties defined in Milestone 4.
- Ensure heading hierarchy is correct: page title (24px, bold), section labels (14px, semi-bold, secondary color), body text (14px, regular).
- Standardize padding: 24px horizontal padding on all full-width sections, 12px internal padding on table cells.

### Table Row Hover and Focus States

```css
.company-row {
    display: grid;
    grid-template-columns: 2fr 3fr 1fr 0.8fr 1.5fr;
    align-items: center;
    padding: 14px 24px;
    border-bottom: 1px solid var(--color-border);
    cursor: pointer;
    transition: background 0.1s ease;
}

.company-row:hover {
    background: var(--color-bg-secondary);
}

.company-row.expanded {
    background: var(--color-bg-secondary);
    border-bottom: none;  /* Merge visually with expanded area */
}

.company-row.stale {
    border-left: 3px solid var(--color-stale);
    padding-left: 21px;  /* Compensate for border to maintain alignment */
}

/* Mute rejected rows */
.company-row .col-company,
.company-row .col-status {
    /* Check stage via data attribute */
}
```

For rejected row muting, add a `data-stage` attribute to the row and use CSS:

```css
.company-row[data-stage^="rejected"] .col-company,
.company-row[data-stage^="rejected"] .col-status,
.company-row[data-stage^="rejected"] .col-activity {
    color: var(--color-text-secondary);
}
```

### Message Bubble Polish

```css
.message-bubble {
    max-width: 70%;
    padding: 12px 16px;
    border-radius: 16px;
    margin-bottom: 8px;
    line-height: 1.5;
    font-size: 0.875rem;
}

.message-bubble.inbound {
    align-self: flex-start;
    background: var(--color-bg-secondary);
    border-bottom-left-radius: 4px;  /* Chat bubble tail effect */
}

.message-bubble.outbound {
    align-self: flex-end;
    background: var(--color-primary);
    color: white;
    border-bottom-right-radius: 4px;
}

.message-sender {
    font-size: 0.75rem;
    font-weight: 600;
    margin-bottom: 4px;
    color: var(--color-text-secondary);
}

.message-bubble.outbound .message-sender {
    color: rgba(255, 255, 255, 0.8);
}

.message-subject {
    font-weight: 600;
    font-size: 0.8rem;
    margin-bottom: 6px;
}

.message-timestamp {
    font-size: 0.7rem;
    color: var(--color-text-secondary);
    margin-top: 6px;
    text-align: right;
}

.message-bubble.outbound .message-timestamp {
    color: rgba(255, 255, 255, 0.7);
}

.message-body {
    white-space: pre-wrap;       /* Preserve line breaks from email */
    word-break: break-word;      /* Prevent overflow on long URLs */
}
```

### Badge Polish

```css
.badge {
    display: inline-block;
    padding: 2px 10px;
    border-radius: 9999px;
    font-size: 0.7rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.03em;
    white-space: nowrap;
}

.badge-blue { background: rgba(37, 99, 235, 0.1); color: var(--color-primary); }
.badge-green { background: rgba(16, 185, 129, 0.1); color: var(--color-success); }
.badge-red-muted { background: rgba(107, 114, 128, 0.1); color: var(--color-danger-muted); }
.badge-gray { background: rgba(107, 114, 128, 0.1); color: var(--color-text-secondary); }
.badge-amber { background: rgba(245, 158, 11, 0.1); color: var(--color-stale); }
```

### Filter Controls Polish

```css
.filter-controls {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px 24px;
    border-bottom: 1px solid var(--color-border);
}

.search-input {
    flex: 1;
    max-width: 320px;
    padding: 8px 12px;
    border: 1px solid var(--color-border);
    border-radius: 6px;
    font-size: 0.875rem;
    outline: none;
    transition: border-color 0.15s;
}

.search-input:focus {
    border-color: var(--color-primary);
    box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.1);
}

.filter-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    background: rgba(37, 99, 235, 0.08);
    color: var(--color-primary);
    border-radius: 9999px;
    font-size: 0.75rem;
    font-weight: 500;
}

.chip-dismiss {
    background: none;
    border: none;
    color: var(--color-primary);
    cursor: pointer;
    font-size: 1rem;
    line-height: 1;
    padding: 0;
}
```

---

## Step 7: Responsive Handling (Desktop Only)

The app targets 1200px+ viewports but should handle resizes down to 1000px:

```css
/* At narrower widths, shrink the stats grid */
@media (max-width: 1100px) {
    .stats-bar {
        grid-template-columns: repeat(3, 1fr);
    }
}

/* At very narrow widths, stack to 2 columns */
@media (max-width: 800px) {
    .stats-bar {
        grid-template-columns: repeat(2, 1fr);
    }

    .company-row {
        grid-template-columns: 1.5fr 2fr 1fr 1fr;
    }

    /* Hide the stale column, rely on left border indicator */
    .col-stale { display: none; }
}
```

---

## Step 8: Smooth Transitions

Add transitions for expanding/collapsing conversation views:

```css
.conversation-view {
    overflow: hidden;
    animation: slideDown 0.2s ease-out;
}

@keyframes slideDown {
    from {
        opacity: 0;
        max-height: 0;
    }
    to {
        opacity: 1;
        max-height: 600px;
    }
}
```

For stat card selection:
```css
.stat-card {
    transition: border-color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease;
}
```

---

## Acceptance Criteria

This milestone is complete when:

1. The stats bar displays all 6 stat cards with correct counts from the database.
2. Clicking a stat card filters the company table to the matching subset; the card shows an active/selected state.
3. Clicking the same active stat card clears the filter and deselects the card.
4. A filter chip appears in the filter controls area when a stat card filter is active, and dismissing the chip clears the filter.
5. Stat card filters combine correctly with search and sort controls.
6. The status dropdown is disabled or reset when a stat card filter is active.
7. Stats refresh automatically every 60 seconds and on tab refocus.
8. The skeleton loading state shows while stats are being fetched.
9. All visual polish items are applied: hover states, transitions, badge styling, message bubble tails, stale row borders, rejected row muting.
10. The layout handles window widths from 1000px to 1920px+ without breaking.
11. The conversation view expand/collapse has a smooth animation.
