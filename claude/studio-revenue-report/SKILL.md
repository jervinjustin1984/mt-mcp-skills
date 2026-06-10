---
name: studio-revenue-report
description: Generates a live membership revenue overview report for a MarianaTek fitness studio using the connected MCP. Use this skill whenever the user asks about revenue, membership income, MRR, monthly recurring revenue, financial performance, revenue trends, revenue at risk, or per-member value. Triggers on phrases like "revenue report", "how is revenue doing", "membership revenue", "MRR", "financial overview", "revenue trend", "how much are we making", or any request to analyze studio financial or membership revenue data. Always use this skill for these requests — do not attempt to generate the report from memory or without pulling live MCP data.
---

# Studio Revenue Report

Generates a live, interactive membership revenue dashboard for any MarianaTek studio using the connected MCP server.

## Step 1: Identify the MCP server

Check which MarianaTek studio MCP is connected and available. Common ones include:
- `Cousteau` / `Cousteau Admin`
- `Spin Unlimited`
- `Ra Yoga`
- `Zenergy`
- `OneSweat`

Use whichever studio MCP the user has connected and is asking about. If multiple are connected, ask which studio they want the report for.

## Step 2: Pull live data

Make the following MCP calls:

1. **`studio_list_users`** (page_size: 100+) — get all active members, their `completed_class_count`, `date_joined`, membership info
2. **`studio_list_user_memberships`** or **`studio_list_user_transactions`** for a sample of top members — to identify membership type name and pricing if available
3. If the MCP exposes a `table_reports` or insights endpoint (e.g. `insights_sales`, `insights_active_memberships`), call those for actual revenue figures

### Revenue estimation fallback
If actual billing amounts are not available from the MCP, estimate using:
- Count of active members × assumed monthly membership rate
- Note clearly in the report that figures are estimated and link to MarianaTek dashboard for exact numbers
- Use $149/month as the default assumption unless the data reveals a different membership price

## Step 3: Compute metrics

- **Estimated MRR**: active members × monthly rate
- **Active member count**
- **Avg revenue per member**: MRR / active members
- **Revenue at risk**: at-risk members (≤5 classes) × monthly rate
- **Top members by engagement**: top 10 by `completed_class_count`
- **Monthly trend**: if transaction/billing history is available use real data; otherwise project a plausible growth curve based on member join dates cohorted by month
- **Revenue by class type**: estimate share from class schedule data if available (cycling, barre, boot camp, etc.)

## Step 4: Render the report widget

Use `visualize:read_me` with modules `["chart", "data_viz"]` then `visualize:show_widget` to render the report.

### Report layout (match this exactly)

**Disclaimer banner** (if using estimated data):
- `border-left: 3px solid var(--color-border-warning)`, `border-radius: 0`
- Icon: `<i class="ti ti-info-circle">` + small muted text noting estimates
- Omit this banner if actual billing data was retrieved

**Metric cards row** (4 cards):
1. Est. MRR (or actual MRR if available) — format as `$X,XXX`
2. Active members
3. Avg rev / member — format as `$XXX`
4. Revenue at risk (danger color) — format as `$X,XXX` with sub-label "X at-risk members"

**Two charts side by side**:

Left — **Monthly revenue trend** (line chart):
- X axis: last 7 months labels (e.g. Nov → May) + 2 projected months
- Two datasets:
  - Actual: solid blue line `#378ADD` with fill `rgba(55,138,221,0.08)`
  - Projected: dashed amber line `#EF9F27`, no fill, `borderDash: [5,4]`
- Y axis: formatted as `$Xk`
- Custom HTML legend above with colored squares

Right — **Revenue by class type** (doughnut):
- Slices for top class types offered at the studio (pull from schedule data or use defaults: Cycling, Barre, Boot Camp, Hot Yoga, Pilates, Other)
- Colors: `#378ADD, #1D9E75, #EF9F27, #D4537E, #7F77DD, #888780`
- Custom HTML legend with % per type

**Top 10 members bar chart** (horizontal bar):
- Y axis: member names (top 10 by class count)
- X axis: estimated annual value ($)
- Bar colors: teal `#1D9E75` for top 5, blue `#378ADD` for next 5
- Wrapper height: at least `(10 * 40) + 80 = 480px`

**Action buttons row** (`.no-print` class):
1. "Check upcoming renewals ↗" → `sendPrompt('Which members are coming up for membership renewal soon?')`
2. "Drill into at-risk revenue ↗" → `sendPrompt('Show me a full breakdown of the at-risk members and what revenue is at stake')`
3. "Download PDF" → client-side via `window.print()`

Include print stylesheet:
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
- Horizontal bar chart: set `indexAxis: 'y'`, wrapper height ≥ `(numBars * 40) + 80`px
- Y-axis currency formatter: `callback: v => '$' + v.toLocaleString()`
- All chart colors are hardcoded hex (canvas cannot resolve CSS variables)

### Design system
- Metric cards: `background: var(--color-background-secondary)`, `border-radius: var(--border-radius-md)`, padding 1rem. Include a sub-label line in muted secondary color (13px).
- Grid layout: `display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 12px`
- Two-column chart area: `grid-template-columns: 2fr 1fr` for trend + donut
- All text uses CSS variables — never hardcode `color: #333`
- No emoji, no gradients, no drop shadows
- Sentence case everywhere

## Step 5: Follow-up

After rendering, be ready to:
- Drill into a specific member's billing history
- Pull actual transaction data for the top members
- Check upcoming renewals
- Run the churn risk report alongside this one
- Re-run with updated data
