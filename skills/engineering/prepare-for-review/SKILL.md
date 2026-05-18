---
name: prepare-for-review
description: Assemble a review-ready PR body and inline comments from the breadcrumbs the implementation loop captured, the ADRs in the touched area, and the originating PRD. Use when the user has finished implementation on a branch and wants to ship for review.
---

# Prepare for Review

The author-side companion to the reviewer-side `review` skill. `review` reads someone else's diff; `prepare-for-review` ships yours.

The skill is a **pure transport layer**. It moves rationale from where it already exists (breadcrumbs, ADRs, the originating PRD) into where reviewers look (PR body, inline review comments). It does not generate rationale — if a source is empty, the matching section is omitted. See [ADR 0002](../../../docs/adr/0002-breadcrumbs-for-review-rationale.md).

The issue tracker, breadcrumb convention, and commit style should have been provided to you — run `/setup-matt-pocock-skills` if `docs/agents/issue-tracker.md` and `docs/agents/decisions-breadcrumb.md` are missing.

## Process

### 1. Pin the fixed point

Use the user-supplied fixed point, else `main`. Capture once:

- `git diff <fixed-point>...HEAD` — three-dot diff.
- `git log <fixed-point>..HEAD --oneline` — commit list.
- Branch name + upstream tracking state.

Feature slug = branch name, or the `.scratch/<feature>/` directory if one exists.

### 2. Collect rationale sources

Read; do not synthesise:

- **Breadcrumbs** — `.scratch/<feature>/decisions.md`. Optional.
- **ADRs** — `docs/adr/` (or per-context paths if `CONTEXT-MAP.md` exists). Only those touching files in the diff.
- **Originating PRD/issue** — issue refs in commit messages via `docs/agents/issue-tracker.md`, else `.scratch/<feature>/prd.md` or `issue-*.md`. If nothing, ask the user.

Missing sources produce omitted sections, not stubs.

### 3. Assemble the PR body

Use the template + sourcing rules in [TEMPLATE.md](TEMPLATE.md). Sections in fixed order; sub-sections with empty sources are omitted entirely.

### 3a. Screenshots (frontend only)

If the diff looks frontend, follow [SCREENSHOTS.md](SCREENSHOTS.md) — detection heuristic, Playwright capture, image commit, raw-URL embed. Otherwise skip.

### 4. Build the inline-comment list

For each breadcrumb with `kind: inline`, produce one pending comment:

```
{ file, line, body: "<why>\n\n**Alternatives:**\n- <option>: <why rejected>" }
```

Only `kind: inline` produces inline comments. Malformed entries (missing `file`/`line`) surface to the user — don't silently drop.

### 5. Preview

Before any shared-state action, show the user:

- The assembled PR body, exactly as it will be posted.
- The inline-comment list with file, line, body.
- The push plan — branch, tracking state, push variant.

Invite edits. Capture explicit approval.

### 6. Open the PR

Only after approval:

1. `git push -u origin <branch>` if no upstream, else `git push`.
2. `gh pr create --title "<title>" --body-file <tmp>`. Title from PRD/issue if present, else first commit subject.
3. For each inline comment, `gh api repos/{owner}/{repo}/pulls/{n}/comments` with `commit_id` = PR head, `side: "RIGHT"`.
4. Print the PR URL.

No commit-discipline gate — trust whatever commits exist.

## What this skill does NOT do

- Generate rationale from the diff. Empty sources = omitted sections.
- Rewrite commits. Commit discipline lives in `docs/agents/commit-style.md` and is followed during implementation.
- Auto-invoke after `tdd`. The user runs this when ready.
- Auto-install Playwright. See [SCREENSHOTS.md](SCREENSHOTS.md).
