# GitHub Backlog Management

> A Claude Code skill that turns GitHub Issues + Projects v2 into a disciplined, AI-assisted backlog — no extra tools, no databases, no webhooks.

---

## Motivation

Backlogs rot. Items accumulate without acceptance criteria, blockers go unrecorded, priorities drift from execution order, and eventually the backlog stops reflecting reality — so people stop trusting it.

This skill keeps a GitHub backlog honest. Every item is INVEST-validated before it lands in the queue. Blockers are tracked with GitHub's native dependency API, not buried in comments. `/execute-backlog-item` picks the topmost unblocked work automatically, so "what do I do next?" has a deterministic answer.

Everything stays in GitHub — Issues, Projects v2, Milestones, Labels. No extra tools, no database, no webhooks.

### What Claude does (and doesn't do)

Claude enforces structure; it doesn't set your priorities. Specifically:

- **INVEST gate** — flags items with vague scope or missing acceptance criteria before they enter the queue
- **Dependency inference** — reads prose ("depends on X"), surfaces candidates for you to confirm
- **Next-item selection** — picks the topmost unblocked work; you decide whether to execute it
- **Audit** — `/validate-backlog` is read-only and surfaces problems as ready-to-run `gh` commands

### Already have a `TODO.md`, `BACKLOG.md`, or some other list?

Run `/migrate-backlog`, point Claude at the file, and it imports everything into GitHub Issues — skipping done items, inferring dependencies from your own prose, and letting you review before anything is applied.

---

## Requirements

- [Claude Code](https://claude.ai/code) (CLI or IDE extension)
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated (`gh auth login`)
- A GitHub repository with Issues enabled
- Git `origin` remote pointing to that repository

---

## Installation

### Native plugin install (recommended)

Add the marketplace and install the plugin with two Claude Code commands:

```text
/plugin marketplace add gringolito/github-backlog-management-skill
/plugin install github-backlog-management@gringolito
```

Restart Claude Code if it was already running. All commands below are then available in any repository you open with Claude Code.

> **Note:** If plugin installation fails with an SSH authentication error, see [Plugin install fails with SSH authentication error](#plugin-install-fails-with-ssh-authentication-error).

---

## Authentication & Permissions

### 1. Authenticate the GitHub CLI

If `gh auth status` reports that you are not logged in, run:

```bash
gh auth login
```

Choose **GitHub.com**, then **Login with a web browser** (recommended — it handles all scope grants in one step). If you prefer a token, select **Paste an authentication token** instead.

### 2. Verify required token scopes

This skill calls GitHub's Issues, Projects v2, and Dependencies APIs. Your token must include all four scopes:

| Scope | Required for |
|---|---|
| `repo` | Create and read Issues, Labels, Milestones, and PRs |
| `project` | Read and write GitHub Projects v2 |
| `read:user` | Resolve the authenticated user for metadata |
| `read:org` | Required when the repo lives under a GitHub organization |

Check your current scopes:

```bash
gh auth status
```

If `project` or `read:user` are missing, add them without re-authenticating:

```bash
gh auth refresh --scopes project,read:user
```

> **Note:** `gh auth refresh` works for OAuth (web browser) logins. If you authenticated with a Personal Access Token, generate a new classic PAT on GitHub with the scopes listed above.

### 3. Configure Claude Code permissions

This skill runs multi-step workflows that call `gh` and `git` many times in sequence. Without a pre-approved allowlist, Claude Code prompts for permission on every call and interrupts the workflow mid-run.

When you enable the plugin, Claude Code asks for your preferred mode (`yolo`, `safe`, or `off`). To apply that choice, run:

```text
/setup-permissions
```

The command asks which settings file to write (per-project gitignored, per-project shared, or user-global) and merges the allowlist block idempotently — re-running it is safe.

<details>
<summary>Manual fallback — copy the JSON block directly</summary>

Add one of the blocks below to `.claude/settings.json` in any repo where you use this skill, or to `~/.claude/settings.json` for a global default.

**YOLO mode — no prompts during any multi-step command:**

```json
{
  "permissions": {
    "allow": ["Bash(gh *)", "Bash(git *)"]
  }
}
```

**Safe / read-only mode — `/validate-backlog` and read queries run silently; write commands still ask for confirmation:**

```json
{
  "permissions": {
    "allow": [
      "Bash(gh auth status *)",
      "Bash(gh repo view *)",
      "Bash(gh issue list *)",
      "Bash(gh issue view *)",
      "Bash(gh label list *)",
      "Bash(gh project list *)",
      "Bash(gh project view *)",
      "Bash(gh project item-list *)",
      "Bash(gh project field-list *)",
      "Bash(gh release list *)"
    ]
  }
}
```

> **Note:** `gh api` calls used for reading milestones and issue dependencies are not listed above — the same command prefix covers both reads and writes, so they cannot be cleanly separated by pattern. In safe mode these calls will still prompt; approve them when the command starts with `gh api "repos/..."` and contains no `-X POST` or `-X DELETE` flag.

</details>

---

## Features

| Command | What it does |
|---|---|
| `/initialize-backlog` | One-time bootstrap: provisions the GitHub Project v2, the full label catalog, and the Issue Forms template. Idempotent — safe to re-run. |
| `/plan-release` | Creates a Milestone with a due date. Tie-breaks and active-milestone resolution are automatic. |
| `/add-backlog-item` | Interactively authors a single backlog item. Enforces INVEST, recommends rank and priority, wires up native GitHub dependencies. |
| `/migrate-backlog` | Bulk-imports an existing `BACKLOG.md`. Skips Done items. Dependency inference is opt-in — candidates are reviewed before anything is applied. |
| `/refine-backlog` | Lists all `needs-clarification` candidates, lets you select which to refine, then loops through them one by one — asking continue/stop after each. |
| `/refine-backlog-item` | Refines a single `needs-clarification` item: discovery dialogue, body rewrite, INVEST gate, label/rank/dep re-evaluation, and label removal after a final validation pass. |
| `/release-status` | Read-only milestone health dashboard — issue counts by Project Status, % complete, blocked items, and unestimated items. Accepts an optional milestone argument; defaults to the active milestone. |
| `/backlog-health` | Read-only strategic portfolio health report — open-issue distribution by type, priority, and effort; age cohorts; overdue P0/P1 items; stale In-Progress items; metadata debt. Suitable for leadership updates and retrospectives. |
| `/validate-backlog` | Read-only audit. Emits actionable `gh issue edit ...` snippets. Never mutates anything. |
| `/execute-backlog-item` | Picks the topmost unblocked Todo item, respects active milestone scope, skips blocked items, and walks you through to a PR. |
| `/setup-permissions` | Writes the `gh`/`git` allowlist block into your chosen Claude Code settings file. Idempotent — safe to re-run. |

### INVEST — the quality bar every backlog item must meet

INVEST is a checklist for deciding whether a backlog item is ready to be worked on. An item passes when it is:

| Letter | Criterion | Why it matters |
| --- | --- | --- |
| **I** | Independent | Can be built and shipped without waiting on another item in the same batch. Avoids invisible coupling that breaks your execution order. |
| **N** | Negotiable | The *what* is agreed; the *how* is still open. If an item already dictates the implementation, you've skipped the design conversation. |
| **V** | Valuable | Delivers something real to a user or the system. If you can't articulate the value, the item probably isn't ready. |
| **E** | Estimable | The team can roughly size it. Items that can't be estimated are usually under-specified or secretly multiple items. |
| **S** | Small | Fits inside a single cycle of work. Large items hide risk and delay feedback. |
| **T** | Testable | Has acceptance criteria concrete enough to write a test or a manual check against. "Works correctly" doesn't count. |

This skill enforces INVEST at creation time (`/add-backlog-item`) and during refinement (`/refine-backlog-item`). Items that don't pass get the `needs-clarification` label instead of landing in the queue — because a vague item at the top of your backlog is just a polite way of not knowing what you're doing next.

### Backlog structure

Every issue created by this skill follows a consistent body shape:

```
### What
### Why
### In Scope
### Out of Scope
### Acceptance Criteria
### INVEST Notes
```

Every backlog item carries three label groups:

- **Type** — `type:feature` `type:bug` `type:security` `type:performance` `type:dx` `type:tech-debt` `type:reliability` `type:compliance` `type:spike`
- **Priority** — `priority:P0` through `priority:P3`
- **Effort** — `effort:XS` `effort:S` `effort:M` `effort:L` `effort:XL`

Priority is severity classification. Execution order is the manual Project rank — the two are independent concepts that should stay consistent but are never conflated.

#### External blocker stubs

`type:external-blocker` is a special infrastructure label for lightweight stub issues that represent external constraints (API limitations, vendor issues, regulatory holds, etc.) blocking one or more backlog items. Stubs carry **only** the `type:external-blocker` label — no priority, no effort, no rank. They are created by `/add-external-blocker`, never appear as executable work in `/execute-backlog-item`, and are excluded from all milestone counts and planning scope. Close a stub with `/resolve-external-blocker` when the external constraint is lifted.

### Workflow

```
/initialize-backlog ──► /plan-release ──► /add-backlog-item
                                          /migrate-backlog
                                                │
                                                ├──► /refine-backlog ──► /refine-backlog-item
                                                ├──► /release-status    (read-only)
                                                ├──► /validate-backlog  (read-only)
                                                └──► /execute-backlog-item
```

Run `/initialize-backlog` once. Every other command preflights for the linked Project and stops with a clear error if it is missing.

---

## Usage

### Starting from scratch

```
/initialize-backlog
```

This provisions the GitHub Project v2, creates all labels, opens a PR with the Issue Forms template, and writes `.claude/backlog-project.json`. Run it once per repo.

### Planning a release

```
/plan-release
```

Claude presents three release modes — **Maintenance** (patch an existing milestone), **Regular** (you select scope interactively), or **Automated** (Claude proposes scope from unassigned items). It infers a semver version from the scope and creates a Milestone with a due date.

### Adding a backlog item

```
/add-backlog-item
```

Claude asks clarifying questions, authors the issue body, recommends a rank in the Project, and links any declared blockers using GitHub's native dependency API.

### Importing an existing backlog

```
/migrate-backlog
```

Point Claude at your existing `BACKLOG.md`. Done items are skipped. Dependency hints in prose (`"depends on"`, `"blocked by"`) are surfaced for your review before anything is applied.

### Refining unclear items

```
/refine-backlog
```

Lists all `needs-clarification` items sorted by priority, lets you select which ones to work on, then calls `/refine-backlog-item` for each — asking whether to continue after every iteration.

```
/refine-backlog-item 42
```

Refines a single item directly (useful when you know exactly which issue needs attention). Guides a discovery dialogue, rewrites the body, re-evaluates labels and rank, runs a validation gate, and removes `needs-clarification` only when everything checks out.

### Checking release health

```
/release-status
```

Produces a Markdown dashboard for the active milestone: issue counts by Project Status (Done / In Progress / Todo), % complete, blocked items (requires GitHub's native dependency API), and open items missing an `effort:*` label. Pass a milestone title, number, or version string to target a specific release:

```
/release-status v1.3.0
```

The output is valid GitHub-Flavored Markdown — paste it directly into a standup document, Slack message, or GitHub comment.

### Checking portfolio health

```
/backlog-health
```

Produces a Markdown strategic health report across all open Project issues: distribution tables by type, priority, and effort; age cohorts (<7d, 7–30d, 30–90d, >90d); overdue P0 (>14 days) and P1 (>30 days) items; stale In-Progress items (no update in 7+ days); and a metadata debt list of issues missing any label group. Useful for weekly leadership updates or retrospectives.

### Auditing backlog health

```
/validate-backlog
```

A read-only pass that surfaces missing labels, malformed issue bodies, dangling blockers, and cross-Project dependency smells. Outputs copy-pasteable `gh` commands — never applies fixes itself.

### Executing next work

```
/execute-backlog-item
```

Picks the topmost unblocked Todo item (active milestone first, unmilestoned fallback), reports which items were skipped and why, and walks you through implementation to a PR.

---

## Troubleshooting

### Plugin install fails with SSH authentication error

**Symptom:** Plugin installation fails with one of these errors:

```text
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

```text
No ED25519 host key is known for github.com and you have requested strict checking.
Host key verification failed.
fatal: Could not read from remote repository.
```

**Root cause:** Claude Code clones marketplace plugins using SSH URLs (`git@github.com:...`), even though marketplace repositories are public and read-only. This requires SSH to be configured for GitHub — including authentication keys and a trusted host entry — regardless of whether you use HTTPS with `gh auth login`. This is a known upstream issue tracked at [anthropics/claude-code#26588](https://github.com/anthropics/claude-code/issues/26588).

**Workaround A — rewrite SSH URLs to HTTPS (recommended if you don't rely on SSH for GitHub):**

```bash
git config --global url."https://github.com/".insteadOf git@github.com:
```

This tells Git to silently use HTTPS whenever it encounters a GitHub SSH URL, bypassing the SSH requirement entirely.

**Workaround B — configure SSH for GitHub (if you already use or need SSH):**

1. Add GitHub's host key to your known hosts:

```bash
ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts
```

2. Make sure your SSH key is added to your GitHub account and to your local SSH agent (`ssh-add`).

---

## Contributing

Issues, improvement suggestions, and pull requests are all welcome.

**Found a bug or unexpected behavior?**
Open a GitHub Issue describing what command you ran, what you expected, and what actually happened. Include the relevant `gh` output if you have it.

**Have an idea for a new feature or command?**
Open an Issue with the `type:feature` label and describe the problem it solves. The best feature requests explain the workflow gap, not just the proposed solution.

**Want to contribute a fix or improvement?**

1. Fork the repository
2. Create a branch
3. Make your changes — each command spec lives in `commands/*.md`
4. Verify cross-command invariants still hold (see the consistency greps in [CLAUDE.md](CLAUDE.md))
5. Open a PR with a clear description of what changed and why

This repository uses [Conventional Commits](https://www.conventionalcommits.org/). Commit messages must follow the `<type>: <description>` format. Common types: `feat` for new behavior, `fix` for corrections, `docs` for README/comment changes, `refactor` for rewrites that don't change behavior, `chore` for maintenance. Example: `feat: add needs-refinement label to validate-backlog report`.

When editing command specs, preserve the existing style: numbered workflow sections with `(MANDATORY)` / `(STRICT)` / `(RELATIVE)` flags, opening prose in the `You are an AI agent acting as...` form, and the standard preflight block. Label catalog, preflight stop string, and issue body section headings must stay consistent across all command files.

---

## License

MIT — see [LICENSE](LICENSE).

---

## On Beer-ware and the spirit that lives on

There is a beautiful license called the Beerware License. It was written by Poul-Henning Kamp sometime in the 1990s and it says, more or less: *if you think this software is worth it, and we ever meet in person, you can buy me a beer.*

It is one of the most honest licenses ever written. It captures exactly the spirit of open source: share freely, ask for nothing, and if someone's work genuinely helped you, buy them a drink and tell them about it.

Sadly, the Beerware License is not OSI-approved. It lacks the formal language needed for corporate legal teams to wave it through, which means — in a cruel twist — the most human license ever written is the one least likely to be used by humans working inside institutions.

So this project is MIT. Lawyers can sleep soundly.

But the spirit is still here. If this skill saved you an afternoon of backlog wrangling, helped you ship something that mattered, or simply made your GitHub a little less of a mess — and if we ever happen to meet in person — you can buy me a beer.
