---
name: ecrm-retro-insights
description: Generates AI-powered retrospective insights for the ECRM sprint at Sonos — pulls the last closed sprint, compares velocity and carry-over trends across the previous 3 sprints, and proposes structured retro items (what went well, what didn't, patterns, team health signals, and concrete improvement suggestions). Use this skill whenever the user asks for retro prep, retrospective insights, sprint retrospective, retro analysis, or how the last ECRM sprint went overall.
---

# ECRM Retrospective Insights

Generate a structured retrospective analysis for the most recently closed ECRM sprint. The output covers velocity trends, delivery patterns, team health signals, and LLM-generated improvement proposals — ready to paste into a retro board or Confluence page.

---

## Step 1 — Find the Atlassian cloud ID

Call `mcp__atlassian-rovo-mcp__getAccessibleAtlassianResources` to get the Sonos Jira cloud ID. Use it for all subsequent `fetchAtlassian` calls.

---

## Step 2 — Get the last 4 closed sprints

Call `mcp__atlassian-rovo-mcp__fetchAtlassian` with:
- url: `/rest/agile/1.0/board/2557/sprint?state=closed&maxResults=4`
- method: GET

Sort by `endDate` descending. The **first result** is the sprint being reviewed (T). The next three (T-1, T-2, T-3) are used for trend comparison.

Extract for each sprint: `id`, `name`, `startDate`, `endDate`, `goal` (if set).

---

## Step 3 — Fetch issues for the reviewed sprint (T)

Call `mcp__atlassian-rovo-mcp__searchJiraIssuesUsingJql` with:
- jql: `project = ECRM AND sprint = "<sprint_id_T>" ORDER BY status ASC`
- fields: `summary, status, assignee, customfield_10016, issuetype, priority, labels, updated, created, resolutiondate`
- limit: 50

Paginate if total > 50.

`customfield_10016` = story points. If null, treat as 1 point.

---

## Step 4 — Fetch issues for the previous 3 sprints (T-1, T-2, T-3)

Repeat Step 3 for each of the 3 previous sprint IDs. You only need:
- Total issues committed
- Total story points committed
- Completed issues (status = Done)
- Completed story points
- Count of carry-over issues (issues that appeared in a prior sprint too)

Use JQL shortcut for carry-overs:
`project = ECRM AND sprint = "<sprint_id>" AND sprint in closedSprints() AND issueFunction in lastSprints("ECRM Scrum - Daily Standup", 2)`

> If the carry-over JQL is not supported, use: `project = ECRM AND sprint = "<sprint_id>"` and flag issues whose `created` date is before the sprint `startDate` as likely carry-overs.

---

## Step 5 — Compute sprint T metrics

### Delivery metrics
- **Committed**: total issues and story points at sprint start (all issues in sprint)
- **Completed**: issues and points with status = Done
- **Completion rate**: completed / committed × 100
- **Carry-overs out**: issues that were NOT completed and likely roll into next sprint

### Velocity trend
Build a 4-sprint table:

| Sprint | Committed pts | Completed pts | Completion % | Carry-overs |
|--------|--------------|---------------|--------------|-------------|
| T-3    | ...          | ...           | ...          | ...         |
| T-2    | ...          | ...           | ...          | ...         |
| T-1    | ...          | ...           | ...          | ...         |
| T      | ...          | ...           | ...          | ...         |

Calculate:
- **Velocity trend**: is completion % going up, down, or flat across T-3 → T?
- **Carry-over trend**: are carry-overs increasing or decreasing?
- **Average velocity**: mean completed pts across T-3 → T-1 (baseline), compare to T

### Team health signals (sprint T only)
- **Workload distribution**: count issues per assignee. Flag if any one person has > 40% of issues.
- **Unassigned work**: count issues with no assignee that were Not Done.
- **Unpointed issues**: count issues where story points were null/missing.
- **Blocked issues**: count issues with status = Blocked OR label "blocked" at any point (proxy: still not Done at sprint end).
- **Late completions**: issues completed in the last 2 days of the sprint (resolutiondate within 2 days of sprint endDate) — signals last-minute rushing.
- **Issue type mix**: count Stories vs Bugs vs Tasks vs Sub-tasks. Flag if Bugs > 30% of total.

---

## Step 6 — Generate retrospective insights

Using all the data above, generate the following sections. This is the core LLM reasoning step — be specific, reference actual issue keys and assignees where relevant, and prioritize actionable observations over generic advice.

---

### ✅ What went well

Identify 2–4 genuine positives from the sprint data. Examples of things to look for:
- Completion rate above 80% → strong delivery sprint
- Carry-overs lower than T-1 → improvement in scope management
- No blocked issues → smooth flow
- Balanced workload across team
- Sprint goal met (if goal was set and all goal-related issues are Done)
- Velocity above the 3-sprint average

Be specific: "Completed X% of committed points, above the 3-sprint average of Y%."

---

### ⚠️ What didn't go well

Identify 2–4 friction points. Look for:
- Completion rate below 70% → delivery gap
- Carry-overs increased vs T-1 → scope creep or poor estimation
- One person carrying > 40% of the work → team imbalance
- High % of late completions (> 30% of Done issues finished in last 2 days)
- Blocked issues that weren't resolved mid-sprint
- High bug ratio suggesting reactive work crowding out planned work
- Large unpointed backlog entering the sprint

Be specific: "X issues were carried over, up from Y in the previous sprint."

---

### 🔍 Patterns & observations

Identify 2–3 structural patterns that span multiple sprints or reveal systemic issues. Examples:
- Carry-overs have been increasing for 3 consecutive sprints → backlog grooming or estimation problem
- Velocity dropping despite similar commitment → something is slowing the team down mid-sprint
- Same assignee repeatedly has unfinished work → capacity or dependency issue
- Bug ratio growing sprint-over-sprint → technical debt accumulating
- Unpointed issues appearing every sprint → stories entering sprint without refinement

For each pattern, name it clearly and back it up with numbers from the trend table.

---

### 💡 Improvement proposals

Generate 3–5 concrete, actionable improvement proposals tailored to the issues found above. Each proposal should follow this format:

**[Proposal title]**
- **Problem it solves**: [What issue/pattern this addresses]
- **Suggested action**: [Specific, concrete action the team can take]
- **Owner suggestion**: [Who on the team is best positioned to drive this — based on data]
- **Success metric**: [How you'd know it worked in the next sprint]

Examples of proposals (only include if relevant to the data):
- Introduce a Definition of Ready checklist to block unpointed stories from entering sprint
- Add a mid-sprint check-in (Wed) to surface blocked items earlier
- Cap any single assignee at X issues to balance workload
- Set a sprint bug budget (e.g. max 25% bugs) to protect planned work
- Run a quick backlog grooming session before sprint planning to reduce carry-overs
- Establish a carry-over limit (e.g. max 2 issues) as a team agreement

---

### 🎯 Suggested retro prompts

Generate 3 open-ended questions the Agile Lead can use to facilitate the retrospective discussion, based specifically on what the data revealed. These should spark honest conversation, not just repeat the observations.

Examples:
- "We completed X% of our points — what got in the way of the remaining Y%?"
- "ECRM-123 was carried over for the third time — what would it take to finally close it?"
- "One person handled 45% of the sprint work — is that sustainable, and what can we do about it?"

---

## Step 7 — Publish to Confluence

After generating all insights, create a new Confluence page as a child of the Sprint Review 2026 parent page.

Call `mcp__atlassian-rovo-mcp__createConfluencePage` with:
- **spaceKey**: `it`
- **parentId**: `2065662133`
- **title**: `Sprint Retro — [Sprint Name]` (e.g. "Sprint Retro — ECRM Sprint 42")
- **content**: the full retrospective formatted as Confluence storage format (see template below)

### Confluence storage format template

Build the page body as HTML using Confluence storage format macros:

```html
<p><strong>Sprint:</strong> [Sprint Name] &nbsp;|&nbsp; <strong>Period:</strong> [Start Date] → [End Date] &nbsp;|&nbsp; <strong>Generated:</strong> [datetime BRT]</p>

<h2>📈 Velocity Snapshot (last 4 sprints)</h2>
<table>
  <tbody>
    <tr><th>Sprint</th><th>Committed pts</th><th>Completed pts</th><th>Completion %</th><th>Carry-overs</th></tr>
    <tr><td>[T-3 name]</td><td>[X]</td><td>[X]</td><td>[X]%</td><td>[X]</td></tr>
    <tr><td>[T-2 name]</td><td>[X]</td><td>[X]</td><td>[X]%</td><td>[X]</td></tr>
    <tr><td>[T-1 name]</td><td>[X]</td><td>[X]</td><td>[X]%</td><td>[X]</td></tr>
    <tr><td><strong>[T name]</strong></td><td><strong>[X]</strong></td><td><strong>[X]</strong></td><td><strong>[X]%</strong></td><td><strong>[X]</strong></td></tr>
  </tbody>
</table>
<p><strong>Trend:</strong> [↑ Improving / ↓ Declining / → Stable] &nbsp;|&nbsp; Avg velocity (baseline): [X] pts &nbsp;|&nbsp; This sprint: [Y] pts</p>

<h2>✅ What Went Well</h2>
<ul>
  <li>[Insight 1]</li>
  <li>[Insight 2]</li>
</ul>

<h2>⚠️ What Didn't Go Well</h2>
<ul>
  <li>[Insight 1]</li>
  <li>[Insight 2]</li>
</ul>

<h2>🔍 Patterns &amp; Observations</h2>
<ul>
  <li>[Pattern 1]</li>
  <li>[Pattern 2]</li>
</ul>

<h2>💡 Improvement Proposals</h2>
<ol>
  <li>
    <strong>[Proposal Title]</strong><br/>
    <strong>Problem:</strong> [...]<br/>
    <strong>Action:</strong> [...]<br/>
    <strong>Owner:</strong> [...]<br/>
    <strong>Metric:</strong> [...]
  </li>
</ol>

<h2>🎯 Retro Discussion Prompts</h2>
<ol>
  <li>[Question 1]</li>
  <li>[Question 2]</li>
  <li>[Question 3]</li>
</ol>
```

After the page is created, output the Confluence page URL:
`https://sonosinc.atlassian.net/wiki/spaces/it/pages/[new_page_id]`

If a page with the same title already exists under parent `2065662133`, call `mcp__atlassian-rovo-mcp__updateConfluencePage` instead to update it with the latest content (increment version number).

---

## Step 8 — Output the full retrospective document

Print the complete retrospective in this format:

```
═══════════════════════════════════════════════════════
🔄 ECRM RETROSPECTIVE INSIGHTS
Sprint: [Sprint Name] | [Start Date] → [End Date]
Generated: [current datetime BRT]
═══════════════════════════════════════════════════════

📈 VELOCITY SNAPSHOT (last 4 sprints)
──────────────────────────────────────
[4-sprint trend table]

Trend: [↑ Improving / ↓ Declining / → Stable] | Avg velocity (T-3→T-1): [X] pts | This sprint: [Y] pts

──────────────────────────────────────
✅ WHAT WENT WELL
──────────────────────────────────────
• [Insight 1]
• [Insight 2]
• [Insight 3]

──────────────────────────────────────
⚠️  WHAT DIDN'T GO WELL
──────────────────────────────────────
• [Insight 1]
• [Insight 2]
• [Insight 3]

──────────────────────────────────────
🔍 PATTERNS & OBSERVATIONS
──────────────────────────────────────
• [Pattern 1]
• [Pattern 2]

──────────────────────────────────────
💡 IMPROVEMENT PROPOSALS
──────────────────────────────────────
1. [Proposal Title]
   Problem: [...]
   Action:  [...]
   Owner:   [...]
   Metric:  [...]

2. [...]

──────────────────────────────────────
🎯 RETRO DISCUSSION PROMPTS
──────────────────────────────────────
1. "[Question 1]"
2. "[Question 2]"
3. "[Question 3]"

══════════════════════════════════════════════════════
```

---

## Tips for good output

- Ground every observation in actual data — don't generate generic retro advice.
- If carry-over count is 0, celebrate it explicitly in "What went well."
- If the sprint goal was set (non-empty), check whether goal-tagged issues were completed and call it out.
- When proposing an owner, use the assignee name from Jira — don't say "the team lead" generically.
- If data for T-2 or T-3 is unavailable (new board, no history), note it and work with what you have.
- Avoid hedging language like "it seems" or "possibly" — if the data shows it, state it directly.
- Keep each bullet under 2 lines. Retros are time-boxed; the output should be scannable in 2 minutes.
