# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **skill** — a collection of slash-command specs (one markdown file each in `commands/`) that drive a GitHub-native backlog management workflow.

There is **no source code, no build, no tests, no lint**. Each command file is an AI agent prompt. "Working in this repo" means editing these prompts — Claude Code is both the artifact's author and its eventual runtime.

## The commands and their workflow

```text
initialize-backlog   ─►  plan-release   ─►  add-backlog-item / migrate-backlog
                                                  │
                                                  ├─►  refine-backlog ─► refine-backlog-item   (needs-clarification)
                                                  ├─►  release-status                          (read-only dashboard)
                                                  ├─►  backlog-health                          (read-only portfolio report)
                                                  ├─►  validate-backlog                        (read-only audit)
                                                  └─►  execute-backlog-item                    (picks topmost unblocked)
```

`initialize-backlog` is the bootstrap. Every other command preflights for a Project linked to the repo and stops with the standard error if missing. `plan-release` creates Milestones (releases). `add-backlog-item` and `migrate-backlog` create Issues. `execute-backlog-item` picks work. `refine-backlog` orchestrates a session over items flagged `needs-clarification`, delegating each to `refine-backlog-item`. `release-status` produces a read-only milestone health dashboard. `backlog-health` produces a read-only strategic portfolio health report (distribution, age cohorts, overdue P0/P1, stale In-Progress, metadata debt). `validate-backlog` audits without mutating.

## Invariants that MUST be preserved across all commands

When editing any command, these must stay consistent across files. Most consistency violations are caught by:

```bash
grep -hoE "type:[a-z-]+" commands/*.md | sort -u           # 10 type labels
grep -hoE "priority:P[0-3]" commands/*.md | sort -u         # 4 priority labels
grep -hoE "effort:(XS|S|M|L|XL)\b" commands/*.md | sort -u  # 5 effort labels
grep -n "No Backlog project linked" commands/*.md           # identical preflight stop string in all consumer commands
grep -n ".claude/backlog-project.json" commands/*.md        # metadata file referenced everywhere
```

1. **Label catalog** — `type:{feature,bug,security,performance,dx,tech-debt,reliability,compliance,spike,external-blocker}`, `priority:{P0,P1,P2,P3}`, `effort:{XS,S,M,L,XL}`, plus `needs-clarification`. Any new value or rename must be applied in all command files. `type:external-blocker` is infrastructure-only — never assigned to work items.

2. **Standard preflight stop string** — every consumer command outputs *exactly* `No Backlog project linked to <owner>/<repo>. Run /initialize-backlog first.` when the metadata file is missing.

3. **Issue Forms template body shape** — section headings `### What`, `### Why`, `### In Scope`, `### Out of Scope`, `### Acceptance Criteria`, `### INVEST Notes` (in this order). `validate-backlog` parses these by exact match. `add-backlog-item`, `migrate-backlog`, `refine-backlog-item` all emit bodies that conform. The template itself is authored at runtime by `initialize-backlog` (NOT committed in this repo) and goes out via PR, never direct commit.

4. **Metadata file location** — `.claude/backlog-project.json`. `initialize-backlog` writes it; all other commands read it directly with no live-query fallback. Schema documented in `initialize-backlog.md`.

5. **Active milestone resolution** — earliest open `due_on`, tie-break by lowest version parsed from milestone title (`v1.2.0` < `v1.3.0`), fallback to milestone `number` for non-parseable titles. When no milestone has `due_on`, lowest version wins. This logic must match in `execute-backlog-item`, `add-backlog-item`, and `release-status`.

6. **Priority label vs Project rank** — these are independent concepts. `priority:*` is severity classification. Execution order is the manual rank in the Project's `Todo` column (top wins). `add-backlog-item` does relative analysis on BOTH and recommends keeping them consistent, but `execute-backlog-item` sorts by rank ONLY and ignores the priority label for ordering. Don't conflate.

7. **Native deps as source of truth** — Issue Dependencies (`/dependencies/blocked_by`, `/dependencies/blocking`) and Sub-issues (`/sub_issues`) are GitHub-native (GA Aug 2025). They are NOT mirrored into the issue body. `execute-backlog-item` strictly skips blocked items. All five commands that touch deps handle the `404`-on-private-repos case gracefully and emit a warning starting with `Issue Dependencies API unavailable`.

8. **Sub-issues stay independent** — assigning a sub-issue parent does NOT inherit milestone, priority, effort, type, or rank. Cross-Project / cross-repo blockers are permitted (flagged as a smell by `validate-backlog`, not rejected).

## Locked design decisions

These were made deliberately — don't undo without explicit user direction:

- **Single repo, single Project** — `Backlog` titled Project v2 linked to the active repo. No cross-repo Projects.
- **Status is the only Project v2 custom field** — bucket/priority/effort live as repo labels (visible on issue cards everywhere).
- **`migrate-backlog` skips Done items** — historical work is not migrated to GitHub. PR shipped references stay in the source `BACKLOG.md`.
- **Issue Forms template ships via PR** — `initialize-backlog` opens a `chore/backlog-item-issue-template` PR; it never commits the template directly to the default branch.
- **`validate-backlog` is strictly read-only** — never mutates issues. It surfaces `gh issue edit ...` snippets the user can run.
- **Migration dep inference is opt-in** — `migrate-backlog` scans source prose for hints like "depends on" / "blocked by" but presents all candidates in a single review block and applies only after user confirmation.
- **`plugin.json:version` is the update cache key** — Claude Code uses this field to decide whether `/plugin update` fetches new code from the remote. Pushing commits without bumping the version leaves all installed users on the previous version indefinitely. **Policy: version bumps happen at release closure (via `/close-release`), not per PR.** Do not bump `plugin.json:version` in feature PRs.

## Commit requirements

All commits to this repository MUST be both **signed** (`-S`, GPG/SSH signature) and **signed-off** (`-s`, Developer Certificate of Origin). Use:

```bash
git commit -S -s -m "..."
```

Never skip signing or the DCO sign-off (`--no-gpg-sign`, omitting `-s`).

## When editing commands

- Match the existing style: numbered workflow sections with `(MANDATORY)` / `(STRICT)` / `(RELATIVE)` flags, opening prose `You are an AI agent acting as...`, `Rules & Constraints` section, `Output Expectations` section.
- New commands MUST add the standard preflight block (auth → origin parse → metadata file read → standard stop string → label catalog check).
- After non-trivial edits, run the consistency greps above and verify no spurious bucket/Bucket leftovers (`grep -nE "[Bb]ucket" commands/*.md` should be empty — that term was renamed to `type` in this repo's history).
- All `gh` errors must be surfaced verbatim — never swallow.

## Verification (manual, in a scratch GitHub repo)

There is no automated test suite. The end-to-end smoke is:

1. `/initialize-backlog` in a fresh repo with `gh auth status` succeeding → verify Project, labels, Issue Forms PR, `.claude/backlog-project.json` written.
2. `/plan-release` → milestone created with `due_on`.
3. `/add-backlog-item` → issue with all three label groups, body matches template, item in Project at the chosen rank with Status=Todo, deps applied if declared.
4. `/migrate-backlog` against a small sample BACKLOG.md including a Done item → Done item skipped, others migrated, dep inference candidates reviewed.
5. `/execute-backlog-item` → picks topmost unblocked item, surfaces skipped-because-blocked items above it.
6. `/release-status` → confirm Markdown dashboard renders with correct counts by Status, blocked-items section (or graceful omission on `404`), and unestimated list; run again with an explicit milestone argument and verify it resolves correctly.
7. `/validate-backlog` → seed deliberate violations (missing `priority:*`, dangling blocker, cross-Project blocker) and confirm they appear in the Critical/Quality/Consistency report sections.
8. `/backlog-health` → confirm Markdown report renders all six sections (summary, distribution tables, age cohorts, overdue P0/P1, stale In-Progress, metadata debt); verify stubs excluded from counts; confirm zero mutations.
9. `/refine-backlog` → presents the `needs-clarification` queue, user selects items, loop delegates to `/refine-backlog-item`; label removed only after pre-removal validation gate passes.

## Agent skills

### Issue tracker

Issues live in GitHub Issues, managed via the `github-backlog-management-skill`. See `docs/agents/issue-tracker.md`.

### Triage labels

This repo uses the `github-backlog-management-skill`'s own label classification (`type:*`, `priority:*`, `effort:*`) rather than the canonical triage roles. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context repo: one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
