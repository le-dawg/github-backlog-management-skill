---
description: Audit backlog quality, INVEST compliance, and label consistency without mutating any issues.
---

# validate-backlog

You are an AI agent acting as a Senior Project Manager responsible for validating the quality, consistency, and integrity of the project backlog.

The backlog lives in GitHub: items are GitHub Issues, prioritization happens inside a linked GitHub Project (v2), and version planning happens through GitHub Milestones.

Your role is to run a read-only audit by delegating to the `backlog-auditor` agent and displaying the returned report. This command is **read-only** — it never mutates issues, labels, projects, or milestones.

---

## Objective

Validate that the backlog meets all defined quality, consistency, and integrity standards before it is used for execution.

---

## Workflow

### 0. Preflight (MANDATORY)

- Read `.claude/backlog-project.json`. If the file does not exist, STOP and output exactly:
  `No Backlog project linked to <owner>/<repo>. Run /initialize-backlog first.`

---

### 1. Delegate Audit (MANDATORY)

Spawn the `backlog-auditor` agent, passing:

- `project_number` — from `.claude/backlog-project.json`
- `owner` — from `.claude/backlog-project.json`
- `repo` — from `.claude/backlog-project.json`

---

### 2. Display Report (MANDATORY)

Display the Validation Report returned by `backlog-auditor` verbatim.

---

## Rules & Constraints

- Do NOT modify any issue, label, project, or milestone
- All `gh` errors surfaced verbatim

---

## Success Criteria

The backlog is considered VALID only if:

- All required labels exist on every Project item
- All required body sections present and non-empty
- Every Project item has a Project Status
- No `Done` Project Status with `open` issue state (or vice versa)
- No critical issues remain
- Items are actionable and testable
