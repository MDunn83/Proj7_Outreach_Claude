# CLAUDE.md

## MANDATORY: Read n8n_SKILL.md before doing anything else.
## MANDATORY: Read [PROJECT]_ClaudeCode_Lessons_Learned.md before doing anything else.
## Do not build anything until you confirm you have read both files.
## GitHub MCP only. Do not run any local git commands. Do not touch the local machine.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repository Purpose

This repo produces a single n8n workflow JSON file (`FILENAME.json`) importable directly into n8n. There is no build system, no tests, and no dependencies -- the output artifact is the JSON file itself.

---

## Key Constraints

- **GitHub MCP tools only.** All file pushes go through `mcp__github__push_files`. Do not run `git` commands locally or modify files on the local machine.
- **Target branch:** `BRANCH_NAME` on `mdunn83/REPO_NAME`.
- The workflow JSON must be valid and importable into n8n without modification.
- After completing each phase, push to GitHub and stop. Wait for explicit confirmation before proceeding to the next phase.
- If you are unsure how to implement any node or connection, stop and ask rather than attempting it silently.
- Do not initialize local git repos. Do not create or modify stop hooks or any files under ~/.claude/. GitHub MCP only.

---

## n8n JSON Structure Notes

- Each node requires a unique `id` (UUID v4), a `name`, a `type`, and `typeVersion`.
- Connections are declared separately in the top-level `"connections"` object, keyed by source node name.
- Credentials are referenced by name (not ID) -- use the exact credential names listed below.
- The `"Wait"` node type is `n8n-nodes-base.wait`; set `resume: "timeInterval"` with `amount` and `unit` as needed.
- Retry settings live inside each node's `"onError"` field: `{ "maxTries": 3, "waitBetweenTries": 2000 }`.
- Google Sheets read uses `"operation": "getRows"` -- never `"getAll"` (does not exist). Never add a `resource` field to a Sheets read node; doing so hides all other parameters in the UI.
- Google Sheets append uses `"operation": "append"` on `n8n-nodes-base.googleSheets`.
- Google Tasks create uses `n8n-nodes-base.googleTasks` with `resource: "task"`, `operation: "create"`.
- Groq LLM calls use `@n8n/n8n-nodes-langchain.lmChatGroq` -- prefer this over the OpenAI node pointed at Groq.
- Google Tasks due dates must use full ISO 8601 format: `"2026-07-01T00:00:00.000Z"` -- bare date strings return a 400 error.
- Merge node (typeVersion 3) defaults to 2 inputs -- always set `"numberInputs"` explicitly.
- Merge combineByPosition parameter: `"combineBy": "combineByPosition"` (not `"combinationMode"`).
- **Sanitize Text Code node:** For any workflow that sends LLM output to Gmail or another output node, always insert a Code node between the LLM Chain and the output node. Prompt-level formatting rules alone do not prevent mid-sentence line breaks. Pattern:
  ```javascript
  let text = $json.text || '';
  text = text.replace(/\\n/g, ' ');        // literal backslash-n (two chars)
  text = text.replace(/\n(?!\n)/g, ' ');  // single newlines mid-prose
  text = text.replace(/ {2,}/g, ' ');     // collapse extra spaces
  text = text.trim();
  return { json: { text } };
  ```
  Sequence: **LLM Chain → Sanitize Text → Gmail → Log → Update**.

---

## Global LLM Prompt Rule

Every LLM prompt in every node must include this instruction:

```
Output ONLY the requested content. Begin directly with the first line of output.
Do not include any introductory text, preamble, or closing remarks.
```

---

## n8n Credentials (as configured in n8n)

| Service | Credential Name in n8n |
|---|---|
| Gmail | `Gmail OAuth2 API` |
| Google Sheets | `Google Sheets OAuth2 API` |
| Google Tasks | `Google Tasks OAuth2 API` |
| Groq | `Groq account` |
| Gemini | `Google Gemini(PaLM) Api account` |

---

## Platform and Output

- **Automation platform:** n8n
- **Output format:** Single `.json` file importable into n8n
- **LLM provider:** [SPECIFY -- Groq / Gemini / other]
- **Model:** [SPECIFY]
- **Rate limiting:** [SPECIFY if needed -- e.g. 3-second wait node between LLM calls for Groq free tier]

---

## Workflow Architecture

[FILL IN -- describe the workflow purpose, input source, output targets, and high-level node sequence before starting the build]

---

## Google Sheets Structure

[FILL IN -- list sheet names and column schemas. Verify all tab names against the actual spreadsheet before exporting -- `sheetName` is case-sensitive and must match exactly.]

---

## Key Assumptions and Decisions

[FILL IN -- document any architectural decisions, edge cases, or constraints specific to this workflow before building]
