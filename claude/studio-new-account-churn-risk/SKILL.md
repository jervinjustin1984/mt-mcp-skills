---
name: studio-new-account-churn-risk
description: Generates a live new-account churn risk report for a MarianaTek fitness studio, focused exclusively on members who joined in the last 90 days. Use this skill whenever the user asks about new member churn, new account engagement, early retention, new member risk, recently joined members, or whether new members are coming back. Triggers on phrases like "new account churn", "new member report", "new members at risk", "early retention", "recently joined", "90 day members", "are new members coming back", or any request to analyze engagement or churn risk specifically for new or recent members. Always use this skill for these requests — do not attempt to generate the report from memory or without pulling live MCP data.
---

# Studio New Account Churn Risk Report

Generates a live, interactive churn risk dashboard scoped to members who joined in the last 90 days, for any MarianaTek studio using the connected MCP server.

## Step 1: Identify the MCP server

Check which MarianaTek studio MCP is connected and available. An example includes:
- `Cousteau` / `Cousteau Admin`

Use whichever studio MCP the user has connected and is asking about. If multiple are connected, ask the user which studio they want the report for.

## Step 2: Pull live member data

**Paginate through ALL members** — studios can have thousands of accounts. Fetch in batches of 10 pages at a time (page_size: 100 per page), continuing until a page returns fewer than 100 results (indicating the last page).

Batch loop:
1. Fetch pages 1–10 concurrently
2. Check if any page returned fewer than 100 results → that's the last page, stop
3. If all 10 returned 100 results, fetch pages 11–20, then 21–30, etc.
4. Continue until the last page is found

Extract for each member:
- `id`
- `first_name` + `last_initial` (note: some MCPs use these separate fields instead of `full_name`)
- `completed_class_count`
- `date_joined`
- `archived_at`

Filter to:
1. Active members only (`archived_at` is null)
2. **Joined within the last 90 days** — compute `today - 90 days` and exclude anyone whose `date_joined` is older than that cutoff

## Step 3: Classify risk tiers

Apply these rules **in priority order** — check from top to bottom and assign the first matching category:

1. **New account** — `date_joined` is within the last 7 days (recency always wins, regardless of class count)
2. **Never attended** — 0 classes completed
3. **Critical** — 1 class completed
4. **High risk** — 2–3 classes completed
5. **Watch** — 4–5 classes completed
6. **Engaged** — 6+ classes completed

Compute:
- Total new accounts (all members in the 90-day window)
- Count per tier
- % per tier (of total new accounts)

## Step 4: Render the report widget

Use `visualize:read_me` with modules `["chart", "data_viz"]` then `visualize:show_widget` to render the report.

### Report layout (match this exactly)

**Metric cards row** (4 cards):
1. Total new accounts (last 90 days)
2. Never attended — count of Never attended tier — muted/secondary color
3. Critical — count of Critical tier — danger color
4. High risk — count of High risk tier — warning color

**Two charts side by side**:

Left — Doughnut chart (6 tiers):
- New account = #378ADD (blue)
- Never attended = #888780 (gray)
- Critical = #E24B4A (red)
- High risk = #EF9F27 (amber)
- Watch = #F5C842 (yellow)
- Engaged = #97C459 (green)
- Custom HTML legend above with tier name, count, and percentage

Right — Bar chart (member count by class bracket):
- Brackets: [0, 1, 2–3, 4–5, 6–10, 11+]
- Colors: gray for 0, red for 1, amber for 2–3, yellow for 4–5, blue/teal for 6+
- X axis label: "Classes attended"

**Members table** — show Critical, High risk, and Never attended only (exclude Watch, Engaged, and New account):
- Columns: Name | Joined | Classes | Risk level
- Sort order: Critical first, then High risk, then Never attended; within each tier sort by days_ago descending (longest since joining first)
- Risk level column width: 140px (wide enough so "Never attended" fits on one line without wrapping)
- No max-height cap — show ALL rows, do not truncate or scroll
- Row backgrounds:
  - Critical rows: `background: var(--color-background-danger)`
  - Never attended rows: `background: var(--color-background-secondary)`
  - High risk rows: alternating transparent / `var(--color-background-secondary)`
- Risk badges:
  - Critical: bg=#E24B4A, text=#FCEBEB
  - High risk: bg=#EF9F27, text=#412402
  - Never attended: bg=#888780, text=#F1EFE8
- "Joined" column: show as a relative date, e.g. "14 days ago" or "52 days ago"

**Action buttons row** (`.no-print`):
1. "Draft re-engagement email ↗" → `sendPrompt('Draft a re-engagement email for my at-risk new members')`
2. "Download PDF" → client-side via `window.print()`

Print stylesheet:
```html
<style>
  @media print {
    .no-print { display: none !important; }
    body { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
  }
</style>
```

### Chart.js config notes
- Load from: `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js`
- Every canvas needs `role="img"`, `aria-label`, and fallback text
- Disable default legend: `plugins: { legend: { display: false } }`
- Use `responsive: true, maintainAspectRatio: false`
- Canvas wrapper: `position: relative; width: 100%; height: 220px`
- All chart colors are hardcoded hex (canvas cannot resolve CSS variables)

### Design system
- Metric cards: `background: var(--color-background-secondary)`, `border-radius: var(--border-radius-md)`, padding 1rem
- 4-card grid: `display: grid; grid-template-columns: repeat(4, 1fr); gap: 12px`
- Table: `border: 0.5px solid var(--color-border-tertiary)`, `border-radius: var(--border-radius-lg)`
- No max-height or overflow-y on the table wrapper — all rows must be visible
- All text uses CSS variables for dark mode compatibility — never hardcode `color: #333` etc.
- Buttons: plain `<button>` tags (pre-styled by the host)
- No emoji, no gradients, no drop shadows
- Sentence case everywhere

## Step 5: Follow-up

After rendering, be ready to:
- Draft a re-engagement email targeting new at-risk members
- Drill into a specific member's profile
- Adjust the window (e.g. "show me just the last 30 days")
- Re-run with updated live data
