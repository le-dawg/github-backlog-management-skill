---
description: Refine a single ambiguous backlog item through guided INVEST validation and label correction.
---

# refine-backlog-item

You are an AI agent acting as a Senior Project Manager refining a single ambiguous backlog item.

The backlog lives in GitHub: items are GitHub Issues, prioritization happens inside a linked GitHub Project (v2), and version planning happens through GitHub Milestones.

Items carrying the `needs-clarification` label were created by `migrate-backlog` (or flagged later) because they are missing critical detail — typically with `UNKNOWN` / `NEEDS CLARIFICATION` markers in body sections and open questions parked in `### INVEST Notes`. Your goal is to walk this one item through interactive discovery, fill the gaps, re-evaluate severity / effort / type / Project rank using full relative analysis, and remove the `needs-clarification` label once validation passes.

---

## Objective

Bring the target issue to a fully refined state where:

- All required body sections are filled (no `UNKNOWN` / `NEEDS CLARIFICATION` markers, no `_No response_`)
- `### INVEST Notes` is empty OR contains only acknowledged residual questions
- The item passes INVEST
- `priority:*`, `effort:*`, `type:*` labels reflect the refined understanding (re-evaluated relatively against existing items)
- Project rank reflects the refined understanding (re-evaluated relatively)
- The `needs-clarification` label is removed

---

## Workflow

### 0. Preflight (MANDATORY)

- Read `.claude/backlog-project.json`. If the file does not exist, STOP and output exactly:
  `No Backlog project linked to <owner>/<repo>. Run /initialize-backlog first.`
- Verify the canonical label catalog is present (`type:*`, `priority:*`, `effort:*`, `needs-clarification`):
  - `gh label list --limit 100`
  - If any required label is missing, STOP and instruct the user to run `/initialize-backlog`

---

### 1. Resolve Target Issue

- Read the argument passed to the command. It may be:
  - An issue number (e.g. `/refine-backlog-item 42`) — use directly
  - A title or partial title (e.g. `/refine-backlog-item "add OAuth"`) — search with `gh issue list --search "<text>" --state open --json number,title,url --limit 10`, then present matches and ask the user to confirm which one
  - No argument — ask: "Which issue should I refine? You can provide an issue number or a title."
- Fetch issue data and verify the issue is a member of the linked Project: `gh project item-list <project-number> --owner <owner> --format json --query "#<n>"` — if the issue is NOT in the Project, STOP and output: `Issue #<n> is not in the linked Backlog project. Only Project members can be refined here.`
- If the issue does NOT carry `needs-clarification`, warn: "Issue #<n> does not carry `needs-clarification`. Proceed anyway? [Y/n]" and stop if the user declines.

---

### 2. Display Item

- Title, issue URL
- Current labels: `type:*` / `priority:*` / `effort:*` (highlight any missing groups)
- Milestone, Project Status
- Full body sections, with every `UNKNOWN` / `NEEDS CLARIFICATION` marker highlighted
- Existing `### INVEST Notes` content — this is where `migrate-backlog` parks open questions
- **Current relationships** (fetched via `gh api`):
  - If the Dependencies API returns `404` on this repo (private repo without paid plan), skip blocker/blocking fields and emit one warning: `Issue Dependencies API unavailable on this repo — dependency display and updates skipped.`
  - Blockers (`blocked_by`): list each with `#N`, title, state. Cross-Project / cross-repo blockers explicitly flagged. If a blocker carries `type:external-blocker`, display it as `External: <stub title>` (e.g. `External: Vendor API rate limit freeze`) to distinguish it from regular issue dependencies.
    - `gh api "repos/<owner>/<repo>/issues/<n>/dependencies/blocked_by"`
  - Blocking: list each with `#N`, title, state.
    - `gh api "repos/<owner>/<repo>/issues/<n>/dependencies/blocking"`
  - Sub-issue parent (if any): `#N`, title.
    - `gh issue view <n> --json parent --jq '.parent'`

---

### 3. Discovery Dialogue (MANDATORY)

Reuse the discovery pattern from `add-backlog-item`:

- Ask clarifying questions to resolve EVERY `UNKNOWN` / `NEEDS CLARIFICATION` marker in `### What`, `### Why`, `### In Scope`, `### Out of Scope`, `### Acceptance Criteria`
- Walk through the open questions in `### INVEST Notes` one by one
- Identify:
  - Desired outcome
  - User/business impact
  - Constraints, risks, edge cases
- Revisit the displayed relationships:
  - **Dependency scan**: delegate to the `dependency-inferrer` agent with:
    - **Prose**: the full issue body (all sections concatenated)
    - **Issue roster**: the list of open issues in the Project (`gh issue list --state open --json number,title --limit 200 | jq -r '.[] | "#\(.number) \"\(.title)\""'`)
    If the agent returns any candidates, present them to the user as starting proposals for the relationship review. `UNRESOLVED` targets are surfaced as open questions for the user to clarify.
  - Are existing blockers still relevant? Should any be removed via `gh api -X DELETE "repos/<owner>/<repo>/issues/<n>/dependencies/blocked_by/<blocker-id>"`?
  - Did refinement reveal NEW blockers? (issue numbers; cross-repo allowed)
  - Should the sub-issue parent change or be removed?
- Challenge vague answers — DO NOT accept hand-waving like "improve performance" or "make it better"
- If the user genuinely cannot answer a question, capture it as a remaining gap (handled in step 5)

---

### 4. Reconstruct Body

Delegate body authoring to the `issue-body-author` agent:

- **Mode**: `refine`
- **Input**: the existing issue body (as fetched in step 2) plus all corrections and answers discovered in step 3
- **Existing body**: pass the full current body so the agent can preserve unchanged sections

The agent returns an updated body with all `UNKNOWN` / `NEEDS CLARIFICATION` / `_No response_` markers replaced by the discovered content. Any sections where information is still missing will be marked with `<!-- TODO: ... -->` — those remain as open questions in `### INVEST Notes`.

DO NOT introduce new headings or change ordering — `validate-backlog` parses these section headings.

---

### 5. INVEST Gate (MANDATORY)

Delegate to the `invest-gate` agent with the reconstructed body from step 4 and the issue title.

If `invest-gate` returns `Overall: FAIL`:

- Capture each `FAIL` letter's reasoning in `### INVEST Notes`
- Apply the partial body update (step 6), but SKIP steps 7–10
- KEEP the `needs-clarification` label
- Output the partial-refinement result: issue URL + per-letter INVEST verdict from `invest-gate` + what remains in `### INVEST Notes`
- STOP — do not continue to label/rank re-evaluation or label removal

If splitting is needed (S letter fails):

- Suggest a split via `/add-backlog-item` for the new item(s)
- Apply the partial body update reflecting the reduced scope of the original item, OR keep the original as-is if the user prefers to handle the split manually
- KEEP the `needs-clarification` label until the split is resolved

---

### 6. Apply Body Update

If INVEST passes (or partial — per step 5):

- Write the refined body to a temp file (avoids shell-escaping issues)
- `gh issue edit <n> --body-file <tmp>`

---

### 7. Re-evaluate Labels (RELATIVE)

Refinement frequently reveals different severity, effort, or type than `migrate-backlog` inferred. Re-run the same relative analysis used by `add-backlog-item` step 5:

- Fetch open Project items with their labels: `gh issue list --state open --json number,title,labels --limit 200`
- Compare the refined item against existing items based on:
  - Impact
  - Risk
  - Urgency
  - Frequency
- Propose changes (independently for each label group):
  - `priority:*` (severity classification)
  - `effort:*` (complexity, NOT time)
  - `type:*` (if classification is now clearer)
- Apply changes ONLY after explicit user confirmation:
  - `gh issue edit <n> --remove-label <old> --add-label <new>`

If existing items appear misranked in their priority labels relative to the refined item, surface the discrepancy and recommend label changes for those existing items. Apply ONLY after confirmation.

---

### 8. Re-evaluate Project Rank + Dependencies (RELATIVE)

Same logic as `add-backlog-item` step 8 (8a → 8d) + step 9:

- Fetch the current Todo column rank: `gh project item-list <project-number> --owner <owner> --query "is:issue status:Todo" --format json --limit 200`
- Determine where the refined item should sit by:
  - Impact
  - Risk
  - Urgency
  - Frequency
  - Dependencies (does this item block or depend on others?)
  - Consistency with the (possibly updated) `priority:*` label — flag divergences for user confirmation
- Propose a concrete rank position (top, above/below specific items, or bottom)
- If existing items appear misranked relative to the refined item, surface those re-rank suggestions too

Apply rank changes ONLY after explicit user confirmation, via:

- The Project's web UI (drag-drop), or
- A GraphQL `updateProjectV2ItemPosition` mutation:

  ```graphql
  mutation {
    updateProjectV2ItemPosition(input: {
      projectId: "<project-node-id>",
      itemId: "<item-node-id>",
      afterId: "<existing-item-node-id-it-should-follow>"
    }) {
      items { totalCount }
    }
  }
  ```

  Use the `id` fields from the `item-list` response. To move an item to the very top, omit `afterId` (or set it to `null`).

**Dependency / Sub-issue changes** — apply the relationship changes the user agreed to in step 3. Same API patterns as `add-backlog-item` step 9:

If the Dependencies API is unavailable on this repo (returns `404`), skip blocker add/remove steps and emit: `Issue Dependencies API unavailable on this repo — dependency updates skipped.`

- **Remove a stale blocker**: `gh api -X DELETE "repos/<owner>/<repo>/issues/<n>/dependencies/blocked_by/<blocker-id>"`
- **Add a new blocker**: delegate to `/block-backlog-item #<n> #<blocker-number>`
- **Change sub-issue parent**: a sub-issue can only have one parent. To re-parent, the user must remove from old parent first via `gh api -X DELETE "repos/<o>/<r>/issues/<old-parent>/sub_issues/<this-id>"`, then add to new parent via `gh api -X POST "repos/<o>/<r>/issues/<new-parent>/sub_issues" -f sub_issue_id=<this-id>`

Apply ONLY after explicit user confirmation. Cross-Project / cross-repo blockers ARE permitted but should be flagged in the per-item confirmation so the user knows they exist.

---

### 9. Pre-removal Validation Gate (MANDATORY)

Before removing the `needs-clarification` label, re-fetch the current live state of the issue and validate it:

- `gh issue view <n> --json number,title,body,labels,milestone`

Run all of the following checks:

- **Sections present** — all six body headings exist in exact order: `### What`, `### Why`, `### In Scope`, `### Out of Scope`, `### Acceptance Criteria`, `### INVEST Notes`
- **No stale markers** — no occurrences of `UNKNOWN`, `NEEDS CLARIFICATION`, or `_No response_` remain in any section
- **Label completeness** — all three label groups are present: one `type:*`, one `priority:*`, one `effort:*`
- **Project Status set** — the item has a non-empty Status value in the Project
- **INVEST re-check** — re-evaluate the final live body (not the in-memory draft) against all six INVEST principles
- **INVEST Notes clear** — `### INVEST Notes` is either empty or contains only acknowledged residual questions with no open action items
- **Effort consistency** — assess whether the current `effort:*` label still fits the refined `### In Scope` and `### Acceptance Criteria`, using the same relative heuristics as step 7. If the label appears inconsistent with the refined scope: gate fails, explain the mismatch, and suggest the likely correct effort label. The user must correct the label (via step 7 flow) before the gate can pass.

If ANY check fails:

- List each failure with the exact issue
- Output: `Pre-removal validation failed — keeping \`needs-clarification\``
- Document as **partially refined** in the session output
- STOP — do not proceed to step 10

If all checks pass, proceed to step 10.

---

### 10. Remove Clarification Label

Only after the pre-removal validation gate passes:

- `gh issue edit <n> --remove-label needs-clarification`
- Print the per-item confirmation:
  - Issue URL
  - Summary of body changes
  - Label changes applied
  - Rank change applied (e.g., "moved from position 8 to position 3")
  - Dependency changes applied (blockers added / removed, sub-issue parent change)

---

## Rules & Constraints

- Do NOT remove `needs-clarification` until the pre-removal validation gate passes (step 9)
- Do NOT silently mutate labels or rank — every change requires explicit confirmation
- Do NOT operate on issues outside the linked Project
- Do NOT reset milestone assignments unless the user explicitly asks
- Do NOT introduce new body section headings — keep them aligned with the canonical Issue Forms template so `validate-backlog` can parse them
- Effort must NEVER be expressed in time (no hours/days)
- All `gh` errors surfaced verbatim
- This command operates on exactly one issue. Use `/refine-backlog` to drive a multi-item session.

---

## Output Expectations

- **Fully refined**: issue URL + body summary + label changes + rank change + dep changes + "✓ `needs-clarification` removed"
- **Partially refined**: issue URL + what was clarified + list of remaining INVEST failures or validation failures + "`needs-clarification` kept"
- Every `gh` command error printed verbatim
