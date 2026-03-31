---
name: sonos-quarter-review
description: Generates a slide-ready Q delivery summary for Sonos IT projects by pulling completed Jira issues from the quarter across ECRM, CAREDEV, HWOF (EAT), SBIZ, CRMPTNR, MET, WEBPLAT, WEBTECH, SUB, and PWM. Use this skill whenever the user asks for a quarterly review, Q delivery summary, what was shipped this quarter, slide deck content for a Sonos quarter, or any variation of "what did [project/team] deliver in Q[N] FY[YY]". Always invoke this skill before generating any response about Sonos quarterly deliveries — even if the user just says "Q3 review" or "what shipped last quarter".
---

# Sonos Quarter Review

Produce a slide-ready delivery summary for a given Sonos fiscal quarter across all IT projects.

## Sonos Fiscal Year Reference

| Quarter | Months        | Example date range (FY26) |
|---------|---------------|---------------------------|
| Q1      | Oct – Dec     | 2025-10-01 → 2025-12-31   |
| Q2      | Jan – Mar     | 2026-01-01 → 2026-03-31   |
| Q3      | Apr – Jun     | 2026-04-01 → 2026-06-30   |
| Q4      | Jul – Sep     | 2026-07-01 → 2026-09-30   |

If the user provides a quarter (e.g., "Q2 FY26"), derive the start/end dates yourself. If they say "this quarter" or "last quarter", infer from today's date.

## Projects and Jira Keys

| User label | Jira project key | Notes |
|------------|-----------------|-------|
| ECRM       | ECRM            | Agent Experience / Salesforce CX |
| CareDev    | CAREDEV         | Nirvana diagnostics platform |
| HWOF       | EAT             | HW Order Fulfillment — search with `text ~ "HWOF"` |
| SBIZ       | SBIZ            | Sonos Pro (B2B subscriptions) |
| CRMPTNR    | CRMPTNR         | CRM Partner / dealer portal |
| MET        | MET             | Marketing Enablement Technology |
| WEBPLAT    | WEBPLAT         | Web Platform |
| WEBTECH    | WEBTECH         | Web Technology / Sonos.com storefront |
| SUB        | SUB             | Consumer Subscriptions |
| PWM        | PWM             | Professional Web & Marketing |

## Step 1 — Gather completed issues from Jira

For each project, search for **non-epic issues** with `statusCategory = Done` updated within the quarter date range. Run searches in parallel where possible to save time.

Standard JQL pattern:
```
project = {KEY} AND issuetype != Epic AND statusCategory = Done
AND updated >= "{START_DATE}" AND updated <= "{END_DATE}"
ORDER BY updated DESC
```

For HWOF, use EAT with the text filter:
```
project = EAT AND issuetype != Epic AND text ~ "HWOF"
AND statusCategory = Done AND updated >= "{START_DATE}" AND updated <= "{END_DATE}"
ORDER BY updated DESC
```

Fetch up to 100 issues per project. You only need the `summary` and `id` fields — descriptions are not required at this stage.

## Step 2 — Identify closed epics with completed children

Also search for closed/done epics per project in the same date range, so you can reference them by name when grouping:
```
project = {KEY} AND issuetype = Epic AND statusCategory = Done
AND updated >= "{START_DATE}" ORDER BY updated DESC
```

This gives you the capability names to anchor your summary sentences.

## Step 3 — Group issues by capability theme

Read through the completed issues and cluster them into 3–6 capability themes per project. Each theme should represent a coherent piece of work a business stakeholder would recognize — not a technical sub-task category.

Good theme names: "Enhanced Chat (MIAW) global rollout", "DTC Mexico geo expansion", "Dealer notification email suite"
Poor theme names: "Bug fixes", "Sprint 3 work", "Miscellaneous"

If an epic name aligns cleanly with a cluster, use it as the theme name.

## Step 4 — Write the slide content

For each project, write **exactly 2 sentences** using this approach:

- **Sentence 1**: Name the most impactful 1–2 capabilities or epics delivered. Be specific — use the epic/feature name, not vague language. Describe what it enables for the business or end user.
- **Sentence 2**: Name the second wave of delivery — either a different capability cluster, platform/technical work that unblocks future quarters, or stabilization work that is meaningful at scale.

Tone: confident, business-facing, impact-led. Avoid jargon like "ticket", "story", or "sprint". Use words like "delivered", "launched", "shipped", "enabled", "completed".

If a project has very few completed issues (fewer than 5), note that explicitly rather than fabricating impact.

## Step 5 — Output format

Use this exact structure for the slide content:

```
## [Project Label] — [Full Project Name]

[Sentence 1]

[Sentence 2]

---
```

Repeat for all 10 projects. After all projects, add a brief **Notes** section flagging:
- Any project with fewer than 5 completed issues (low signal)
- Any Jira key mismatches or search errors encountered
- HWOF reminder: "HWOF work is tracked under the EAT (Enterprise Applications) project"

## Tips for good output

- Lead with capabilities and epics, not individual stories. A story like "Add Bambino pictures to PDP carousel" belongs to the "Bambino launch" theme — surface the theme, not the story.
- When multiple projects all delivered work for the same initiative (e.g., Mojave/Bambino launch, DTC Mexico, RMR), call that out in the relevant slides to show cross-team coordination.
- If SUB has minimal standalone output, note that consumer subscription feature work is often tracked across SBIZ and PWM.
- The HWOF team's FY26 epics span the full year (bucket epics) — completed child issues are the right signal, not epic status.
