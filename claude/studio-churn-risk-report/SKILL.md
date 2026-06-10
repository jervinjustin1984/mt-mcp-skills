---
name: studio-churn-risk-report
description: Generates a live member churn risk report for a MarianaTek fitness studio using the connected MCP. Use this skill whenever the user asks for a churn report, member risk analysis, at-risk members, member engagement report, or wants to know which members might cancel or stop coming. Triggers on phrases like "churn report", "at-risk members", "member engagement", "who's not coming in", "members at risk", "re-engagement", or any request to analyze member attendance patterns. Always use this skill for these requests — do not attempt to generate the report from memory or without pulling live MCP data.
---

# Studio Churn Risk Report

Generates a live, interactive churn risk dashboard for any MarianaTek studio using the connected MCP server.

## Step 1: Identify the MCP server

Check which MarianaTek studio MCP is connected and available. Common ones include:
- `Cousteau` / `Cousteau Admin`
- `Spin Unlimited`
- `Ra Yoga`
- `Zenergy`
- `OneSweat`

Use whichever studio MCP the user has connected and is asking about. If multiple are connected, ask the user which studio they want the report for.

## Step 2: Pull live member data

Call `studio_list_users` (with `page_size: 100` or more) on the appropriate MCP to get all members. Extract for each member:
- `id`
- `full_name`
- `completed_class_count`
- `date_joined`
- `archived_at` (to filter out churned/archived members)

Filter to active members only (`archived_at` is null).

## Step 3: Classify risk tiers

Use these definitions:
- **Critical risk**: 1–2 classes completed
- **High risk**: 3–5 classes completed
- **Engaged**: 6–20 classes completed
- **Power users**: 21+ classes completed

Compute:
- Total active members
- Count and % per tier
- Average classes per member
- At-risk count (critical + high risk combined)

## Step 4: Render the report widget

Use `visualize:read_me` with modules `["chart", "data_viz"]` then `visualize:show_widget` to render the report.

### Report layout (match this exactly)

**Metric cards row** (4 cards):
1. Total members
2. Avg classes per member (1 decimal)
3. At risk count (≤5 classes) — danger color
4. Critical risk count (1–2 classes) — danger color

**Two charts side by side**:
- Left: Doughnut chart — 4 tiers, colors: Critical=#E24B4A, High=#EF9F27, Engaged=#97C459, Power=#1D9E75. Custom HTML legend above with counts and percentages.
- Right: Bar chart — member count by class bracket: [1–2, 3–5, 6–10, 11–20, 21–50, 51+]. Same color scheme: red for 1–2, amber for 3–5, blue for 6–20, teal for 21+.

**At-risk members table** (members with ≤5 classes):
- Columns: Name | Classes | Risk level
- Critical rows get `background: var(--color-background-danger)`
- Risk badge: Critical = #E24B4A bg / #FCEBEB text, High = #EF9F27 bg / #412402 text
- Sort by classes ascending

**Action buttons row**:
1. "Draft re-engagement email ↗" → `sendPrompt('Draft a re-engagement email for my at-risk members')`
2. "Download PDF" → client-side PDF using `window.print()` with a print stylesheet that hides the buttons. Inject a `<style>` block `@media print { .no-print { display: none !important; } body { -webkit-print-color-adjust: exact; } }` and wrap buttons in `<div class="no-print">`. The download button calls `window.print()`.

### Chart.js config notes
- Load from: `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js`
- Every canvas needs `role="img"`, `aria-label`, and fallback text
- Disable default legend: `plugins: { legend: { display: false } }`
- Use `responsive: true, maintainAspectRatio: false`
- Canvas wrapper: `position: relative; width: 100%; height: 220px`
- All chart colors are hardcoded hex (canvas cannot resolve CSS variables)

### Design system
- Metric cards: `background: var(--color-background-secondary)`, `border-radius: var(--border-radius-md)`, padding 1rem
- Table: `border: 0.5px solid var(--color-border-tertiary)`, `border-radius: var(--border-radius-lg)`
- All text uses CSS variables for dark mode compatibility — never hardcode `color: #333` etc.
- Buttons: plain `<button>` tags (pre-styled by the host)
- No emoji, no gradients, no drop shadows
- Sentence case everywhere

## Step 5: Follow-up

After rendering, be ready to:
- Draft a re-engagement email if the user clicks the button or asks
- Drill into a specific member's profile
- Filter the list differently (e.g. "show only members who joined in the last 60 days")
- Re-run with updated data
