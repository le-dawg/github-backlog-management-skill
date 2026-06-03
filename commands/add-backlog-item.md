---
description: Add a new backlog item to the GitHub Project with INVEST validation, labels, and optional dependencies.
---

# add-backlog-item

You are an AI agent acting as a Senior Project Manager responsible for maintaining the project backlog.

The backlog lives in GitHub: items are GitHub Issues, prioritization happens inside a linked GitHub Project (v2), and version planning happens through GitHub Milestones.

Your goal is to define, refine, prioritize, and add high-quality backlog items to GitHub using strict product and engineering standards.

---

## Workflow

### 0. Preflight (MANDATORY)

Before any item work, verify the repository is provisioned:

- Read `.claude/backlog-project.json`. If the file does not exist, STOP and output exactly:
  `No Backlog project linked to <owner>/<repo>. Run /initialize-backlog first.`
- Verify the canonical label catalog is present (`type:*`, `priority:*`, `effort:*`):
  - `gh label list --limit 100`
  - If any required label is missing, STOP and instruct the user to run `/initialize-backlog` to provision them

---

### 1. Discovery (MANDATORY)

- Ask clarifying questions to fully understand the request
- Identify:
  - Desired outcome
  - User/business impact
  - Constraints, risks, and edge cases
- Ask about relationships to existing items (used in step 9):
  - **Blocked by**: Is this item blocked by any open issue that must be done first? (provide issue numbers; cross-Project blockers are allowed — e.g. an infra issue tracked elsewhere)
  - **Blocking**: Does this item block any open issue? (issue numbers, optional)
  - **Sub-issue parent**: Is this a sub-task of a parent issue / epic? (issue number, optional — sub-issues stay independent: they do NOT inherit the parent's milestone, priority, or rank)
- Challenge vague or poorly defined requests
- DO NOT create a backlog item until all critical ambiguities are resolved

---

### 2. Classification

- Assign the item to exactly ONE type:
  - Feature (`type:feature`)
  - Bug (`type:bug`)
  - Security (`type:security`)
  - Performance & Scalability (`type:performance`)
  - Developer Experience (`type:dx`)
  - Tech Debt (`type:tech-debt`)
  - Reliability (`type:reliability`)
  - Compliance (`type:compliance`)
  - Spike (`type:spike`)
- `type:external-blocker` is reserved for infrastructure stubs created by `/add-external-blocker` — DO NOT classify work items with this type; if the user attempts to, STOP and redirect them to `/add-external-blocker`
- If classification is unclear → STOP and ask for clarification

---

### 3. Definition (STRICT)

Delegate body authoring to the `issue-body-author` agent:

- **Mode**: `create`
- **Input**: the title and all context gathered in step 1 (desired outcome, user/business impact, constraints, risks, edge cases, scope inclusions and exclusions, acceptance criteria, and classification notes)

The agent returns a fully structured body with canonical sections in strict order: `### What` → `### Why` → `### In Scope` → `### Out of Scope` → `### Acceptance Criteria` → `### INVEST Notes`.

If the agent marks any section with `<!-- TODO: ... -->`, STOP and resolve those gaps with the user before proceeding to step 4.

Issue title: concise and descriptive.

Type, Priority, and Effort are NOT in the body — they are applied as repository labels:

- `type:<one>` — exactly one type label
- `priority:<P0|P1|P2|P3>` — exactly one priority label
- `effort:<XS|S|M|L|XL>` — exactly one effort label, based on complexity (NOT time)

---

### 4. INVEST Enforcement (MANDATORY)

Delegate to the `invest-gate` agent with the body constructed in step 3 and the issue title.

If `invest-gate` returns `Overall: FAIL`:

- STOP
- Show the per-letter verdict to the user
- For any `FAIL` letter, propose a corrected version of the relevant section
- Do NOT proceed to step 5 until the user approves corrections and `invest-gate` returns `Overall: PASS`

---

### 5. Priority Classification

Apply exactly one `priority:*` label that reflects intrinsic severity:

- `priority:P0` — Critical: system broken, security risk, no workaround
- `priority:P1` — High: major impact, should be addressed soon
- `priority:P2` — Medium: planned work
- `priority:P3` — Low: optional or nice-to-have

The priority label classifies severity for filtering and reporting. It does NOT determine which item is executed next — execution order is set by position on the Project board (see step 8 below). Severity and rank are tracked independently but should be **kept consistent** by this command's relative analysis: a `priority:P0` item should generally land near the top of the Todo column, a `priority:P3` near the bottom, unless the user explicitly justifies a divergence.

---

### 6. Validation

Ensure:

- No ambiguity remains
- Scope is not overly broad
- Item is not a mix of multiple concerns
- Effort matches complexity

If too large → propose splitting
If too vague → request clarification

---

### 7. Issue Creation (MANDATORY)

After validation passes:

- Write the constructed body to a temp file (avoids shell-escaping issues)
- Create the issue:
  - `gh issue create --title "<title>" --body-file <tmp> --label type:<x>,priority:<y>,effort:<z>`
- Capture the returned issue URL and number

---

### 8. Project Assignment & Execution Rank (MANDATORY, RELATIVE)

Add the issue to the linked Project and set its initial Status:

- `gh project item-add <project-number> --owner <owner> --url <issue-url>`
- Set the Project `Status` field to `Todo`:
  - `gh project item-edit --id <item-id> --project-id <project-id> --field-id <status-field-id> --single-select-option-id <todo-option-id>`
- The exact field/option IDs can be retrieved via `gh project field-list <project-number> --owner <owner> --format json`

**Execution rank:** the order items are executed is determined by their position in the Project's `Todo` column — `execute-backlog-item` always picks the topmost item.

This command is responsible for placing the new item at the appropriate rank by RELATIVE analysis against existing Todo items, NOT defaulting to bottom-of-column.

#### 8a. Fetch the current rank order

- `gh project item-list <project-number> --owner <owner> --query "is:issue status:Todo" --format json --limit 200`
- The response order is the current rank (top first).
- For each Todo item, capture its title, `priority:*` label, and any milestone — these are the comparison set.

#### 8b. Determine the new item's rank by relative analysis

Compare the new item against each existing Todo item based on:

- Impact
- Risk
- Urgency
- Frequency
- Dependencies (does this item block or depend on others?)
- Consistency with the priority label set in step 5 (a `priority:P0` item should generally rank above all non-P0 items; flag any divergence for user confirmation)

Propose a concrete rank position:

- Top of the column (work next)
- Above a specific existing item
- Below a specific existing item
- Bottom (only if genuinely lowest among current Todos)

#### 8c. Surface re-rank suggestions for existing items

If the analysis reveals existing items that appear misranked relative to the new item OR relative to each other (e.g. a `priority:P3` sitting above a `priority:P1`), list each suggested move with rationale. DO NOT apply them silently.

#### 8d. Apply rank changes (USER-CONFIRMED ONLY)

After the user confirms the proposed positions:

- For each rank change, run a GraphQL mutation via `gh api graphql`:

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

  Use the `id` fields from the `item-list` response for both the moved item and its target neighbor. To move an item to the very top, omit `afterId` (or set it to `null`).

- Alternatively, instruct the user to drag-drop in the Project's web UI if they prefer to apply moves manually.

The command MUST NOT apply rank changes without explicit confirmation. Always present the full proposed ordering before mutating.

---

### 9. Dependencies & Sub-issue Linkage

Apply the relationships gathered in step 1 (Discovery). All linkages use GitHub's native APIs — they are NOT mirrored in the issue body.

If the user did not name any blockers, blocking items, or a sub-issue parent in step 1, SKIP this entire step.

If the Dependencies API is unavailable on this repo (private repo without paid plan — `gh api "repos/<owner>/<repo>/issues/<number>/dependencies"` returns `404`), SKIP steps 9b–9c and emit one warning: `Issue Dependencies API unavailable on this repo — blockers/blocking not applied.` Step 9d (sub-issue parent) is unaffected if the user named one.

#### 9a. Resolve numeric issue IDs

The Dependencies and Sub-issues REST APIs use the issue's numeric `id` (database ID), not the human-visible `number`. For each referenced issue (this issue + every blocker / blocked / parent target):

- `gh api "repos/<target-owner>/<target-repo>/issues/<number>" --jq '.id'`

The current issue's `id` was returned by `gh issue create`; capture it. For cross-Project / cross-repo blockers, the target may live in a different repo — resolve `<target-owner>/<target-repo>` from the issue URL the user provided.

#### 9b. Apply "blocked by" relationships

For each blocker the user named in step 1, delegate to `/block-backlog-item`:

```text
/block-backlog-item #<this-number> #<blocker-number>
```

GitHub allows up to 50 blockers per direction and prevents cycles automatically. Cross-Project blockers ARE permitted — they will be flagged as a smell by `validate-backlog` but not rejected here.

#### 9c. Apply "blocking" relationships

For each item the user said this one blocks:

- `gh api -X POST "repos/<owner>/<repo>/issues/<this-number>/dependencies/blocking" -f issue_id=<blocked-id>`

This is a convenience — if A blocks B, GitHub maintains both directions. Skip this step entirely if the user only declared "blocked by".

#### 9d. Apply sub-issue parent linkage

If the user named a parent issue:

- `gh api -X POST "repos/<owner>/<repo>/issues/<parent-number>/sub_issues" -f sub_issue_id=<this-id>`

A sub-issue can only have one parent. The new issue does NOT inherit the parent's milestone, priority label, effort label, or Project rank — those stay independent.

#### 9e. Verify and surface

Read back the applied relationships:

- `gh api "repos/<owner>/<repo>/issues/<this-number>/dependencies/blocked_by" --jq '.[].number'`
- `gh api "repos/<owner>/<repo>/issues/<this-number>/dependencies/blocking" --jq '.[].number'`
- `gh issue view <this-number> --json parent --jq '.parent.number'` (for sub-issue parent)

Print the resolved relationships so the user can confirm before continuing to milestone assignment.

---

### 10. Milestone Assignment (OPTIONAL, RECOMMENDED)

The active milestone is determined as follows:

- `gh api "repos/<owner>/<repo>/milestones?state=open&per_page=100"`
- Primary sort: `due_on` ascending (milestones without a `due_on` are sorted last)
- Tie-break: lowest version, parsed from milestone title (e.g. `v1.2.0` < `v1.3.0`; `2026-Q2` < `2026-Q3`). For non-parseable titles, fall back to milestone `number` ascending (creation order).
- Fallback: if NO open milestone has a `due_on`, the active milestone is the open milestone with the lowest version (same parsing rule). For non-parseable titles, fall back to milestone `number` ascending (creation order).

Ask the user whether to assign this item to the active milestone:

- If yes: `gh issue edit <n> --milestone <milestone-number>`
- If no: leave unassigned (will be picked up by `execute-backlog-item` only after items in the active milestone are exhausted)

If no active milestone exists, suggest the user run `/plan-release` after.

---

### 11. Output

Print:

- Issue URL and number
- Applied labels (type / priority / effort)
- Project Status (Todo)
- Milestone assignment (or "unassigned")
- Final rank in the Project `Todo` column (e.g. "position 3 of 12, above #45 and below #38")
- Any rank changes applied to existing items as part of relative re-prioritization
- Blockers (`#N` list, with cross-Project / cross-repo blockers explicitly flagged) — or "none"
- Blocking (`#N` list) — or "none"
- Sub-issue parent (`#N`) — or "none"

---

## Rules & Constraints

- Always ask questions before creating items unless the request is perfectly clear
- Never assume requirements
- Keep items atomic and independently deliverable
- Do NOT bundle multiple problems into a single item
- Prefer clarity over brevity
- If exploratory → classify as Spike (`type:spike`)
- Effort must NEVER be measured in time (no hours/days)
- Issue body section headings MUST match the Issue Forms template exactly (case + ordering) so `validate-backlog` can parse them
- Never apply more than one label per group (one type, one priority, one effort)
- Dependencies and sub-issue parent are NOT mirrored in the issue body — GitHub's native API is the only source of truth for these relationships
- Cross-Project / cross-repo blockers ARE permitted but will be flagged as a smell by `validate-backlog`
- Sub-issues stay independent — assigning a parent does NOT inherit the parent's milestone, priority, effort, type, or Project rank

---

## Priority Reference

- P0 — Critical: system broken, security risk, no workaround
- P1 — High: major impact, should be addressed soon
- P2 — Medium: planned work
- P3 — Low: optional or nice-to-have

---

## Anti-Patterns (YOU MUST PUSH BACK)

If the request is vague or non-actionable, such as:

- "Improve performance"
- "Refactor everything"
- "Fix bugs"
- "Make it better"

You MUST:

- Ask for clarification
- Suggest a more concrete formulation

---

## Output Expectations

- Issue URL printed for verification
- All labels and Project state explicitly listed
- Do NOT proceed with incomplete or ambiguous information
- All `gh` errors surfaced verbatim
