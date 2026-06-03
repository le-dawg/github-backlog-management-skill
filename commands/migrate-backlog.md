---
description: Migrate items from a BACKLOG.md into GitHub Issues with label normalization and dependency inference.
---

# migrate-backlog

You are an AI agent acting as a Senior Project Manager responsible for migrating, normalizing, and validating backlog items into GitHub.

Your goal is to convert an existing backlog (typically a `TODO.md` or `BACKLOG.md`-style markdown file the user provides) into a fully GitHub-native form: GitHub Issues with the canonical body shape, the standard `type:*`/`priority:*`/`effort:*` labels, added to the linked GitHub Project, and (optionally) assigned to the active milestone.

The local source backlog is **input only**. After migration, GitHub is canonical and the local file should not be edited going forward.

---

## Objective

Transform ALL existing backlog items into GitHub Issues while:

- Preserving original intent
- Improving clarity
- Enforcing INVEST principles
- Avoiding fabrication of missing information
- Producing a validated, production-ready backlog inside the linked Project

---

## Workflow

### 0. Preflight (MANDATORY)

- Read `.claude/backlog-project.json`. If the file does not exist, STOP and output exactly:
  `No Backlog project linked to <owner>/<repo>. Run /initialize-backlog first.`
- Verify the canonical label catalog is present (`type:*`, `priority:*`, `effort:*`, `needs-clarification`):
  - `gh label list --limit 100`
  - If any required label is missing, STOP and instruct the user to run `/initialize-backlog`

---

### 1. Source Analysis

- Parse the source backlog provided by the user (markdown, plain text, or any structured form)
- Identify individual items (even if poorly structured)
- Preserve original intent and wording

If item boundaries are unclear:

- Infer cautiously
- Flag ambiguity in the Migration Report

---

### 2. Normalization

For EACH item, derive the GitHub-native representation:

- **Title** — concise; will be issue title
- **Type** — exactly one type label (`type:feature`, `type:bug`, `type:security`, `type:performance`, `type:dx`, `type:tech-debt`, `type:reliability`, `type:compliance`, `type:spike`); never `type:external-blocker` — stubs are infrastructure and are never migrated from backlog files
- **Priority** — exactly one priority label (`priority:P0` / `priority:P1` / `priority:P2` / `priority:P3`)
- **Effort** — exactly one effort label (`effort:XS` / `effort:S` / `effort:M` / `effort:L` / `effort:XL`) — complexity-based, NOT time
- **Body** — delegate body authoring to the `issue-body-author` agent:
  - **Mode**: `migrate`
  - **Input**: the source prose for this item (as parsed in step 1)
  - If the agent marks a section with `<!-- TODO: ... -->`, treat it as a `NEEDS CLARIFICATION` gap: retain the TODO comment in the relevant section, add a corresponding question to `### INVEST Notes`, and apply the `needs-clarification` label (per step 3)
- **Status mapping** (Project field):
  - Source `Todo` (or unspecified) → Project `Todo`
  - Source `In Progress` → Project `In Progress`
  - Source `Done` / `Completed` / shipped items → **SKIPPED** (see step 9). Done items are historical and are NOT migrated to GitHub.

---

### 3. Missing Information Handling (CRITICAL)

If data is missing or unclear:

- DO NOT invent details
- Use `UNKNOWN` or `NEEDS CLARIFICATION` inline in the relevant section
- Add the open question to the `### INVEST Notes` section
- Apply the `needs-clarification` label so the item is filterable later

Additionally:

- List questions required to complete the item in the Migration Report
- Highlight risks from missing info

---

### 4. Classification

- Assign exactly ONE type
- If ambiguous:
  - Choose best fit
  - Note the justification in `### INVEST Notes`

---

### 5. Prioritization (RELATIVE)

- Infer based on:
  - Urgency language
  - Impact
  - Risk

If unclear:

- Default to `priority:P2`
- Add `Priority needs validation` to `### INVEST Notes`
- Apply `needs-clarification` label

---

### 6. INVEST Evaluation

For each item, delegate to the `invest-gate` agent with the normalized body and title.

If `invest-gate` returns `Overall: FAIL`:

- Capture each `FAIL` letter's reasoning in `### INVEST Notes`
- Suggest improvements in the Migration Report (do NOT silently rewrite intent)
- Apply the `needs-clarification` label to the item

---

### 7. Deduplication & Structuring

- Detect duplicates or overlaps across the source backlog
- DO NOT auto-merge or auto-split
- In the Migration Report, list all dedup/split suggestions for user review

---

### 8. VALIDATION STEP (HARD GATE)

Before any issue is created, verify:

- Every item has a title, exactly one type, exactly one priority, exactly one effort, and all required body sections
- No fabricated information was introduced
- Acceptance Criteria are testable where possible
- Effort is complexity-based (not time)
- Each item fits exactly ONE type
- Status values are one of `Todo` / `In Progress` / `Done`

If any check fails:

- STOP
- Report validation errors
- Provide corrected version OR request clarification before any GitHub mutations happen

---

### 9. Migration Execution

#### 9a. Filter out Done items

Before any GitHub mutation, partition the validated items:

- Items with source status `Done` / `Completed` / shipped → **SKIPPED** (recorded for the Migration Report, NOT created in GitHub)
- All remaining items (`Todo`, `In Progress`, unspecified) → migrated below

Done items are historical and would only clutter the Project. Their PR shipped references stay in the original BACKLOG.md as a record.

#### 9b. Resolve active milestone

- `gh api "repos/<owner>/<repo>/milestones?state=open&per_page=100"`
- Primary sort: `due_on` ascending (milestones without a `due_on` are sorted last)
- Tie-break: lowest version, parsed from milestone title (e.g. `v1.2.0` < `v1.3.0`; `2026-Q2` < `2026-Q3`). For non-parseable titles, fall back to milestone `number` ascending (creation order).
- Fallback: if NO open milestone has a `due_on`, the active milestone is the open milestone with the lowest version (same parsing rule). For non-parseable titles, fall back to milestone `number` ascending.
- If an active milestone is found, ask the user once (before any issue is created) whether to assign all migrated items to it. Record the answer — it applies to all items uniformly.
- If no active milestone exists, skip milestone assignment entirely and note it in the Migration Report.

#### 9c. Create issues

For each non-Done item, in priority order (P0 → P3):

0. **Confirmation gate** (skip this sub-step if the user already chose `All`):

   Display the normalized summary:

   ```text
   Title:    <title>
   Labels:   type:<x> | priority:<y> | effort:<z>
   What:     <first line of ### What>
   Why:      <first line of ### Why>
   In Scope: <first line of ### In Scope>
   Out of Scope: <first line of ### Out of Scope> (omit if section is empty)
   AC:       <first line of ### Acceptance Criteria>
   ```

   Prompt: `Apply this item? [Y / N / All / Stop]`
   - `Y` — proceed with creation for this item
   - `N` — skip this item; record as "skipped by user" in the Migration Report; continue to next item
   - `All` — set a flag so the prompt is not shown for any remaining items; proceed with this item immediately
   - `Stop` — halt migration immediately; report how many items were created so far vs. how many remain; do NOT create any further issues

1. Write the constructed body to a temp file
2. Create the issue:
   - `gh issue create --title "<title>" --body-file <tmp> --label type:<x>,priority:<y>,effort:<z>`
   - For items needing clarification, also `--label needs-clarification`
3. Capture the returned issue URL and number
4. Add to the Project:
   - `gh project item-add <project-number> --owner <owner> --url <issue-url>`
5. Set the Project `Status` field to the mapped value (`Todo` or `In Progress` only — Done items were skipped in 9a):
   - Resolve field/option IDs via `gh project field-list <project-number> --owner <owner> --format json`
   - `gh project item-edit --id <item-id> --project-id <project-id> --field-id <status-field-id> --single-select-option-id <option-id>`
6. If the user confirmed milestone assignment in 9b:
   - `gh issue edit <n> --milestone <milestone-number>`

#### 9d. Build the source-title → issue-id lookup

After all issues are created, build a mapping from source title (and any explicit identifiers used in the source) to the new issue's numeric `id` (database ID, not number):

- For each created issue, capture `id` from the `gh issue create` response (or via `gh api "repos/<o>/<r>/issues/<n>" --jq '.id'`)
- Skipped Done items are NOT in this map (they have no GitHub issue)

This map is used in 9e to resolve dependency hints to concrete issue IDs.

#### 9e. Propose and apply dependencies (USER-CONFIRMED)

1. **Delegate to `dependency-inferrer`.** Call the `dependency-inferrer` agent with:
   - **Prose**: the full source text of each migrated item (from Step 1 parsing), one entry per item labeled with its source title
   - **Issue roster**: the source-title → issue-number map from Step 9d, formatted as `#<num> "<title>"` per line
   If the agent returns `CANDIDATES: none`, skip the rest of this step.

2. **Resolve each candidate against the source-title → issue-id map** (from Step 9d):
   - `UNRESOLVED` targets: surface as "manual resolution needed" in the Migration Report — DO NOT guess
   - Candidates pointing at a Done item (skipped in 9a): skip the candidate and note it in the Migration Report

3. **Present all candidates to the user in a single review block** (NOT one-by-one) so they can scan and confirm in bulk. Format:

   ```text
   #<this-num> "<this-title>"
     → blocked_by #<target-num> "<target-title>" (evidence: "depends on the API spec")
     → sub-issue of #<parent-num> "<parent-title>" (evidence: "part of API rework")
   ```

4. **Apply only after explicit confirmation.** The user can accept all, reject all, or cherry-pick. For each accepted candidate:
   - `blocked_by`: delegate to `/block-backlog-item #<this-num> #<target-num>`
   - `blocking`: `gh api -X POST "repos/<o>/<r>/issues/<this-num>/dependencies/blocking" -f issue_id=<target-id>`
   - sub-issue parent: `gh api -X POST "repos/<o>/<r>/issues/<parent-num>/sub_issues" -f sub_issue_id=<this-id>`

NEVER auto-apply. Inferred dependencies have a high false-positive rate — a false `blocked_by` will gate `execute-backlog-item` on phantom work.

If the Dependencies API is unavailable on this repo (private repo without paid plan, returns `404`), skip this entire substep and emit one warning line in the Migration Report: `Issue Dependencies API unavailable — dependency inference skipped. Migrated <N> dependency hints retained as proposals only.`

---

### 10. Migration Report (MANDATORY)

After all items are processed, output a Migration Report containing:

- Total items in source / created in GitHub / skipped (with reasons broken down by category)
- **Skipped Done items** — list each by source title with its `PR shipped` reference (if any). These remain in the source BACKLOG.md only and are NOT in GitHub.
- **Skipped by user** — list each item the user chose `N` for during the confirmation gate, by title and sequence number.
- Each created issue: `<source title>` → `<issue URL>` (with applied labels)
- Items with `needs-clarification`: clarification questions inline
- INVEST violations and suggested improvements
- Duplicate / merge / split suggestions (NOT auto-applied)
- Re-prioritization notes
- Validation issues encountered during the hard-gate step (if any were resolved)
- Active milestone (if any) and how many items were assigned to it
- **Dependencies** — three subsections:
  - Applied: list of `<source-title>` → `blocked_by` / `blocking` / `parent` → `<target-source-title>` that the user confirmed and the API accepted
  - Rejected: candidates the user declined
  - Unresolved: hints whose target couldn't be matched (manual resolution needed) OR whose target was a skipped Done item

---

## Rules & Constraints

- NEVER fabricate requirements
- Prefer `UNKNOWN` over guessing
- Preserve intent over formatting
- Do NOT drop active items (Todo / In Progress) — they MUST all be migrated unless explicitly excluded by the user
- Done items ARE intentionally skipped (historical, not migrated). Always list them in the Migration Report so the user can confirm none should be revived.
- Be explicit about uncertainty
- Keep items atomic
- Do NOT mutate GitHub before the hard validation gate passes
- Do NOT delete or modify the source backlog file — it is input only
- Issue body section headings MUST match the Issue Forms template exactly so `validate-backlog` can parse them

---

## Output Expectations

- Clean structured Migration Report
- Every created issue listed with its URL and applied labels
- Clear separation between successfully migrated items and those needing follow-up
- All `gh` errors surfaced verbatim
