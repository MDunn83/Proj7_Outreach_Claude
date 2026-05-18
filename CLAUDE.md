# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## MANDATORY: Read Before Building

Read `n8n_SKILL(1).md` completely before writing any node JSON. It encodes hard-won runtime lessons — violating its rules produces workflows that import but fail silently.

> Note: `n8n_and_ClaudeCode_Lessons_Learned.md` is referenced in the starter template but does not yet exist in this repo. Skip that step until the file is added.

---

## Constraints

- **GitHub MCP tools only.** All file pushes go through `mcp__github__push_files`. Do not run `git` commands locally or modify files on the local machine.
- **Target branch:** `claude/plan-n8n-outreach-PKA07` on `mdunn83/proj7_outreach_claude`
- The workflow JSON must be valid and importable into n8n without modification.
- After completing each build phase, push to GitHub and stop. Wait for explicit confirmation before proceeding to the next phase.
- If unsure how to implement any node or connection, stop and ask rather than guessing.

---

## Platform and Output

- **Automation platform:** n8n
- **Output:** Single JSON file importable into n8n (`outreach_workflow.json`)
- **LLM provider:** Groq (free tier)
- **Model:** `llama-3.3-70b-versatile`
- **Rate limiting:** Add a 3-second Wait node between LLM calls if processing multiple rows in rapid succession on Groq free tier

---

## n8n Credentials

| Service | Credential Name in n8n |
|---|---|
| Gmail | `Gmail OAuth2 API` |
| Google Sheets | `Google Sheets OAuth2 API` |
| Groq | `Groq account` |

---

## Global LLM Prompt Rule

Every LLM prompt must include this instruction verbatim:

```
Output ONLY the requested content. Begin directly with the first line of output.
Do not include any introductory text, preamble, or closing remarks.
```

---

## Google Sheets Structure

### Customer Sheet (trigger + update source)
| Column | Notes |
|---|---|
| Customer Name | |
| Company | |
| Plan Tier | |
| Last Activity Date | Used for inactivity check (14-day threshold) |
| Renewal Date | Used for renewal check (30-day window) |
| Last Contacted Date | Used for suppression check (7-day window); updated at end of run |
| Milestone Reached? | Values: `Yes` or blank/other |
| Support Ticket Closed Date | Used for ticket check (24-hour threshold) |
| Email | Recipient address for Gmail node — always pull dynamically, never hardcode |

### Activity Log Sheet (separate sheet, same Google Spreadsheet)
| Column | Notes |
|---|---|
| Timestamp | Use `$now.toISO()` |
| Customer Name | |
| Company | |
| Trigger Type | `Suppressed` / `Ticket` / `Inactivity` / `Renewal` / `Milestone` / `No Action` |
| Email Sent | `Yes` or `No` |
| Message Preview | First 100 characters of email body; blank for Suppressed and No Action |

---

## Workflow Architecture

### Trigger
- **Testing:** Manual Trigger + Google Sheets getAll node (reads all rows at once)
- **Production:** Swap Manual Trigger for a daily Schedule Trigger — no other nodes change

### Node Sequence

1. **Manual Trigger** → fans out to:
   - **Get Customer Rows** (Google Sheets getAll on customer sheet)
2. **Loop Over Items** — iterates one customer row at a time
3. **Suppression Check** (Code node, `runOnceForEachItem`) — IF Last Contacted Date is within the last 7 days:
   - **YES →** Append `Suppressed` row to Activity Log (Email Sent = `No`, Message Preview = blank). Skip to next item.
   - **NO →** Continue to category checks
4. **Category Check — Ticket** (Code node) — Support Ticket Closed Date within last 24 hours?
5. **Category Check — Inactivity** — Last Activity Date 14+ days ago?
6. **Category Check — Renewal** — Renewal Date within next 30 days? (all entries treated as auto-renewal)
7. **Category Check — Milestone** — Milestone Reached? = `Yes`?
8. **No Action branch** — none of the above matched → Append `No Action` row to Activity Log (Email Sent = `No`, Message Preview = blank). Skip email.
9. **Groq LLM node** (one per active category branch) — generates email body using customer fields
10. **Gmail node** — sends email; Gmail does not pass data through, so logging must be wired from the node *before* Gmail
11. **Append to Activity Log** — Timestamp, Customer Name, Company, Trigger Type, Email Sent = `Yes`, first 100 chars of LLM body
12. **Update Customer Sheet** — write today's date to Last Contacted Date column for this row

### Category Hierarchy
One email per customer per run. Checks run in this order; first match wins:

| Priority | Category | Criterion |
|---|---|---|
| 1 | Ticket | Support Ticket Closed Date within last 24 hours |
| 2 | Inactivity | Last Activity Date more than 14 days ago |
| 3 | Renewal | Renewal Date within next 30 days |
| 4 | Milestone | Milestone Reached? = `Yes` |
| — | No Action | None of the above |

### Email Subject Lines (fixed per category)
| Trigger Type | Subject |
|---|---|
| Ticket | `We're here if you need us, [Customer Name]` |
| Inactivity | `We miss you, [Customer Name]` |
| Renewal | `Your renewal is coming up, [Customer Name]` |
| Milestone | `Congratulations on your milestone, [Customer Name]!` |

### LLM Prompt Guidance per Category
- **Ticket:** Warm follow-up; glad to have resolved the issue; invite further questions
- **Inactivity:** State how many days since last activity; politely encourage return
- **Renewal:** State exact number of days until auto-renewal date; no action required from customer
- **Milestone:** Congratulatory; acknowledge the achievement; keep it brief

---

## Key Architectural Decisions

- **Gmail + Logging pattern:** Gmail node terminates its branch — wire the Activity Log append from the node *before* Gmail, not after. See `n8n_SKILL(1).md` Gmail and Logging Pattern section.
- **LLM context preservation:** After any `chainLlm` node, `$json` is dead. Use `$('NodeName').item.json` cross-node references in all downstream Code nodes.
- **Suppressed/No Action rows:** Email Sent = `No`, Message Preview = blank string (not null).
- **Last Contacted Date update:** Runs after logging — only executes for rows where an email was actually sent (Ticket / Inactivity / Renewal / Milestone branches).
- **Date comparisons:** Perform all date arithmetic in Code nodes using `Date.now()` and `new Date(value).getTime()`. Do not rely on n8n expression date helpers for threshold logic.
- **Sheet IDs:** Use placeholder `YOUR_GOOGLE_SHEET_ID` in exported JSON. Fill in manually after import.
