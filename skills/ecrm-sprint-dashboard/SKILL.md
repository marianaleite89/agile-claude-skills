---
name: ecrm-sprint-dashboard
description: Generates a live sprint health dashboard for the ECRM Jira project — pulls the active sprint, builds a burndown chart, flags risks (blocked, flagged, stale), identifies carry-over issues and counts how many sprints each one has been dragged across, then outputs a self-contained HTML page and optionally deploys it to Vercel. Use this skill whenever the user asks about the ECRM sprint status, sprint burndown, sprint dashboard, carry-over issues, sprint risks, or how the ECRM sprint is going — even if they say things like "show me where we are this sprint", "any blockers in ECRM?", "what's being carried over?", or "build me a sprint page". Always invoke this before generating any sprint health response for ECRM.
---

# ECRM Sprint Dashboard

Generate a self-contained HTML dashboard for the current ECRM sprint, covering burndown, risks, and carry-over issues. The output is a single `sprint-dashboard.html` file that can be opened in any browser or deployed to Vercel instantly.

---

## Step 1 — Fetch active sprint data from Sonos Jira

Use the **`mcp__sonos-jira__`** tools — this is the correct Jira integration for Sonos.

**Step 1a — Find the ECRM scrum board:**
```
mcp__sonos-jira__get_agile_boards(project_key="ECRM", board_type="scrum")
```
The board you want is **"ECRM Scrum - Daily Standup"** (board ID `2557`).

**Step 1b — Get the active sprint:**
```
mcp__sonos-jira__get_sprints_from_board(board_id="2557", state="active")
```
This returns the sprint ID, name, start date, and end date.

**Step 1c — Get all sprint issues:**
```
mcp__sonos-jira__get_sprint_issues(
  sprint_id="<id from above>",
  limit=50,
  fields="summary,status,assignee,customfield_10016,issuetype,priority,labels,updated,created"
)
```
If total > 50, paginate with `start_at=50`, `start_at=100`, etc. until all issues are fetched.

The `customfield_10016` field is story points. If null for an issue, treat it as 1 point.

---

## Step 2 — Parse and compute metrics

From the data above, derive the following. If story points are missing for an issue, treat it as 1 point so it still contributes to the count.

### Sprint summary
- Sprint name, start date, end date (from sprint metadata)
- Total issues committed (count of Query A results)
- Total story points committed (sum of all story points in Query A)
- Completed issues (count of Query D results)
- Completed story points (sum of story points in Query D)
- Remaining story points = committed − completed
- Percent complete = completed / committed × 100

### Burndown data
Build a day-by-day burndown array:
- X axis: each calendar day from sprint start to today (inclusive)
- Ideal line: linear decline from total committed points on day 0 to 0 on the last day
- Actual line: for each day, calculate remaining points = total − sum of points completed on or before that day (use the `updated` date of Done issues as their completion date)

### Carry-over issues
For each issue in Query B:
- Count the number of distinct sprint entries in the `sprint` field — that is the number of sprints it has been carried through
- Flag it as carry-over with the sprint count

### Risk flags — classify each risky issue into one of:
| Risk type | Detection |
|-----------|-----------|
| **Blocked** | Status = "Blocked" OR label contains "blocked" OR flag = "Impediment" |
| **Sizing Risk** | Days in current status exceeds what the story size suggests (see formula below) |
| **Stale** | In-progress or To Do, and `updated` is more than 3 days ago (catch-all for unpointed issues) |
| **Carry-over** | Appears in Query B (in a closed sprint too) |
| **No assignee** | `assignee` is null and status is not Done |
| **High priority, not started** | Priority = Highest or High AND status = "To Do" |

### Sizing Risk formula (most important new signal)
An issue is a **Sizing Risk** when it has been stuck in the same non-Done status for longer than its size justifies:

```
days_threshold = max(2, story_points × 1.5)
days_in_status = today − status_last_changed_date (use `updated` as proxy)

If days_in_status > days_threshold → Sizing Risk
```

**Why this matters:** A 1-point story stuck In Progress for 5 days signals something is wrong (the work is probably bigger than estimated, or blocked unofficially). An 8-point story In Progress for 5 days is probably fine. The threshold scales with the size, so you catch the real mismatches.

Show in each Sizing Risk card:
- The issue key, summary, assignee
- Current status and how many days it has been there
- Story points and the threshold that was exceeded (e.g. "5 days in progress — expected ≤ 1.5 days for a 1-pt story")

---

## Step 3 — Build the HTML dashboard

Generate a single self-contained HTML string and write it to `sprint-dashboard.html` in the current working directory (use the `Write` tool).

Use this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>ECRM Sprint Dashboard — [Sprint Name]</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <style>
    /* dark-friendly, clean card layout — see Style Guide below */
  </style>
</head>
<body>
  <!-- 1. Sprint Header -->
  <!-- 2. Summary Stats Row -->
  <!-- 3. Burndown Chart -->
  <!-- 4. Risk Flags Section -->
  <!-- 5. Carry-Over Table -->
  <!-- 6. Full Issue Breakdown by Status -->
  <script>
    /* inline Chart.js burndown initialization */
  </script>
</body>
</html>
```

### Style Guide
- Background: `#0f1117`, cards: `#1e2130`, text: `#e0e0e0`
- Accent colors: green `#4caf50` (done), amber `#ff9800` (at risk), red `#f44336` (blocked), blue `#2196f3` (info)
- Font: system-ui, sans-serif
- Cards with `border-radius: 8px`, subtle `box-shadow`
- Responsive grid for the stats row (4 cards across on desktop)

### Section details

**1. Sprint Header**
Sprint name, date range, and a colored progress bar (green when ≥ 70% complete, amber 40–69%, red < 40%).

**2. Summary Stats Row** — 4 cards:
- Total Committed (points + issues)
- Completed (points + issues + %)
- Remaining (points + issues)
- Days left in sprint

**3. Burndown Chart**
Chart.js line chart, two datasets:
- "Ideal" — grey dashed line
- "Actual" — blue solid line
If the actual line is above the ideal, the chart header turns amber/red as a visual warning.

**4. Risk Flags**
One card per risky issue. Each card shows:
- Issue key (e.g., ECRM-1234) + summary
- Risk type badge (color-coded per the table above)
- Assignee or "Unassigned"
- Current status
Sort order: Blocked first, then Stale, then No Assignee, then High Priority.

**5. Carry-Over Table**
Table columns: Issue Key | Summary | Assignee | Status | Sprints Carried | Points
Highlight rows where Sprints Carried ≥ 3 in amber, ≥ 5 in red.
Show a note at the top: "X issues carried over from previous sprint(s)."

**6. Issue Breakdown by Status**
A grouped list: To Do / In Progress / In Review / Done — showing issue key, summary, assignee, and points for each. Collapsible sections if there are more than 8 issues in a group.

---

## Step 4 — Output the file

Write the complete HTML to `sprint-dashboard.html` using the `Write` tool. Then tell the user:

```
✅ Dashboard written to sprint-dashboard.html

Sprint: [Name] | [Start] → [End]
Committed: [X] pts across [N] issues
Completed: [Y] pts ([Z]%)
Carry-overs: [C] issues
Risks flagged: [R] issues

To open locally: double-click sprint-dashboard.html in Explorer, or open it in your browser.
To deploy to Vercel (one command):
  vercel --prod sprint-dashboard.html
```

If `vercel` CLI is available (check with `vercel --version 2>/dev/null`), deploy automatically and provide the live URL instead.

---

## Tips for good output

- Story points are commonly stored in `customfield_10016` in Jira. If that field is null, also check `story_points` or `customfield_10028` (next-gen projects). If all are null, count the issue as 1 point and note it in the summary.
- Carry-over detection is the most important signal for an Agile lead — make the carry-over table prominent, not buried at the bottom.
- When the sprint is nearly over (≤ 2 days left) and remaining points are high, add a prominent warning banner at the top of the page.
- The burndown ideal line should be based on *working days* if you can determine them; otherwise use calendar days and note the assumption.
- If Query A returns 0 results, there may be no open sprint — tell the user clearly and do not generate an empty dashboard.
- Avoid showing raw Jira field names (like `customfield_10016`) anywhere in the output — always use human-readable labels.
