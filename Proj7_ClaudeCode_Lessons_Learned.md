# n8n and Claude Code — Lessons Learned (Proj7 Outreach)

This document captures issues encountered and resolved during the build of the customer outreach automation workflow. Add to this file as new issues are discovered.

---

## Google Sheets Node

### `getAll` does not exist — use `getRows`
- The operation name in the UI is **Get Row(s)**, which maps to `"operation": "getRows"` in JSON.
- `"operation": "getAll"` is not a valid value and causes the node to fail on import.

### Never add a `resource` field to a Google Sheets read node
- Adding `"resource": "sheetWithinDocument"` (or any resource value) to a Sheets getRows node causes the node to render only the Resource selector in the UI — all other parameters disappear and the node becomes unconfigurable.
- The correct config for a working getRows node is `operation`, `documentId`, `sheetName`, and `options` only. No `resource` field.

### Sheet name must match the actual tab name exactly
- The `sheetName` parameter is case-sensitive and must match the Google Sheets tab name character-for-character.
- Verify tab names in the actual spreadsheet before setting them in JSON. Do not assume `Sheet1`, `Customers`, or any other default.
- The customer sheet and the activity log sheet will have different tab names — confirm both before exporting.

### Update node requires `matchingColumns` to contain a column that uniquely identifies the row
- For the **Update Last Contacted** nodes, the `Email` column was used as the matching key.
- The matching column must exist in the sheet and contain unique values, otherwise the update will hit the wrong row or fail silently.

---

## IF Node

### Boolean conditions require `={{ }}` expression syntax on the left side
- Using `"leftValue": "{{ $json.isSuppressed }}"` (without the `=` prefix) causes a type error at runtime: `Wrong type: '{{ $json.isSuppressed }}' is a string but was expecting a boolean`.
- Correct syntax: `"leftValue": "={{ $json.isSuppressed }}"`
- This applies to every boolean field used in IF node conditions.
- IF node conditions are the **most import-fragile** part of any n8n workflow. Always verify them manually after import — the left side must show as an expression (not a static string) in the UI.

### Use `"operation": "true"` operator for boolean checks
- Operator object: `{ "type": "boolean", "operation": "true" }`
- Do not use `"equal to true"` — it causes type mismatch errors.

---

## Loop Over Items

### Loop Over Items node is not needed for row-by-row processing
- n8n handles per-item iteration natively when Code nodes use `runOnceForEachItem` mode.
- Adding a Loop Over Items node creates confusion: its `done` output (not `loop`) connects to the next node, and the `loop` output goes nowhere.
- For a workflow that reads all rows from a sheet and processes each one, omit the Loop Over Items node entirely.

---

## Gmail Node

### Gmail terminates its branch — wire logging from the node before Gmail, not after
- Gmail does not pass data through to downstream nodes.
- The correct sequence is: **LLM Chain → Sanitize Text → Gmail → Log → Update Last Contacted**.
  - Gmail fires first to confirm delivery.
  - The Log node receives output from Gmail (even though it carries no useful data — use cross-node refs to pull the email body from the Sanitize Text node).
- Never wire a log or update node as a parallel sibling of Gmail expecting to receive the email body from Gmail's output.

### Always reference email address dynamically — never hardcode
- `sendTo` must always be an expression: `={{ $('Category Router').item.json.email }}`
- A hardcoded address in `sendTo` is always a production bug.

---

## LLM / Groq Nodes

### LLM prompt expressions must use `={{ }}` wrapper
- All prompt `text` fields must be set to expression mode with `={{ ... }}` syntax.
- Using backtick template literals (`` `${...}` ``) or bare string concatenation without `={{ }}` causes silent failures — the expression is not evaluated.

### Add explicit formatting rules to every LLM prompt
- Without formatting instructions, Groq models insert hard line breaks mid-sentence and produce fragmented email text.
- Include these rules in every email-generation prompt:
  ```
  Rules:
  - Write in flowing prose. Do not insert line breaks within or between sentences.
  - Use a blank line only to separate distinct paragraphs.
  - Do not include a greeting, salutation, closing, or signature.
  - Output ONLY the requested content. Begin directly with the first line of output.
    Do not include any introductory text, preamble, or closing remarks.
  ```

### Always add a Sanitize Text Code node after every LLM Chain node
- Even with formatting rules in the prompt, models frequently emit hard line breaks mid-sentence.
- A Code node immediately after the LLM Chain (before Gmail) must collapse single newlines into spaces while preserving paragraph breaks:
  ```javascript
  let text = $json.text || '';
  text = text.replace(/\\n/g, ' ');        // literal backslash-n (two chars)
  text = text.replace(/\n(?!\n)/g, ' ');  // single newlines mid-prose
  text = text.replace(/ {2,}/g, ' ');     // collapse extra spaces
  text = text.trim();
  return { json: { text } };
  ```
- The sequence is: **LLM Chain → Sanitize Text → Gmail**. Never wire Gmail directly from LLM Chain.
- Log nodes' Message Preview cross-node refs must point to the Sanitize Text node, not the LLM Chain node, so the preview shows cleaned text.

### `$json` is dead downstream of any `chainLlm` node
- The LangChain `chainLlm` node strips all upstream item context.
- In any node that runs after a `chainLlm` node, use cross-node references to retrieve data from earlier nodes:
  ```
  $('Category Router').item.json.customerName
  $('Sanitize Text Ticket').item.json.text
  ```
- Never rely on `$json.fieldName` after a chain node — the field will be undefined.

### Groq sub-node connects via `ai_languageModel`, not `main`
- The `lmChatGroq` sub-node connects to its parent `chainLlm` node using connection type `ai_languageModel`, not `main`.
- In the connections object, this appears under the Groq node's name with key `"ai_languageModel"` instead of `"main"`.

---

## Date and Timestamp Handling

### Use `$now.toFormat('yyyy-MM-dd')` not `$now.format()`
- Luxon (n8n's date library) uses `.toFormat()`, not `.format()`.
- `.format()` throws a runtime error with no clear signal of the root cause.

### All date comparisons must use `Date.now()` and `new Date(value).getTime()`
- Do not rely on n8n expression date helpers for threshold logic.
- Perform all arithmetic in Code nodes in milliseconds.

---

## Code Nodes

### Always pull fields off `$json` before referencing them
- Every Code node must open by explicitly assigning all needed fields:
  ```javascript
  let customerName = $json['Customer Name'] || '';
  let email = $json.email || '';
  ```
- Referencing a field name directly without first assigning it causes a `ReferenceError` with no clear root cause message.

### Use `runOnceForEachItem` for row-by-row processing
- In this mode, `$json` refers to the current item.
- Return a single object: `return { json: { ... } }` — not an array.

---

## GitHub MCP Constraint

### All file changes go through `mcp__github__push_files` only
- Do not run local git commands.
- Do not write or copy files to the local machine.
- Do not read local files to verify push results — trust the MCP push response.
- All commits go to branch `claude/plan-n8n-outreach-PKA07` on `mdunn83/proj7_outreach_claude`.

---

## General Build Process

### Build in phases to manage token limits
- Large workflow builds should be broken into 3–4 phases, each ending with a GitHub push.
- Do not attempt to generate the full workflow JSON in one response.
- After each phase, wait for explicit user confirmation before proceeding.

### Stop and ask rather than experiment
- When a node config is uncertain, ask before trying something. Experimenting on a live workflow that is partially working causes regressions that are harder to undo than the original problem.
- The user's working state is the ground truth. Revert first, investigate second.
