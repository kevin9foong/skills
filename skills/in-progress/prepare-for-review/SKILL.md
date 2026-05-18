---
name: prepare-for-review
description: Author-side counterpart to the reviewer-side `review` skill. Assemble a review-ready PR — body, inline review comments, and (for frontend changes) screenshots — from breadcrumbs the implementation loop captured, ADRs in the touched area, and the originating PRD/issue. Open the PR via `gh` only after the user confirms the preview. Use when the user has finished implementation on a branch and wants to ship for review.
---

# Prepare for Review

The author-side companion to `review`. `review` is what a reviewer runs over your branch; `prepare-for-review` is what you run before handing it to them.

The skill is a **pure transport layer**. It moves rationale from where it already exists (breadcrumbs, ADRs, the originating PRD/issue, the diff itself) into where reviewers look (PR body, inline review comments). It does not generate rationale. If a source is empty, the corresponding output section is omitted — nothing is invented. See [`docs/adr/0002-breadcrumbs-for-review-rationale.md`](../../../docs/adr/0002-breadcrumbs-for-review-rationale.md) for why this matters.

The issue tracker, breadcrumb convention, and commit style should have been provided to you — run `/setup-matt-pocock-skills` if `docs/agents/issue-tracker.md` and `docs/agents/decisions-breadcrumb.md` are missing.

## Disambiguation

- **`prepare-for-review`** (this skill, author side) — build a PR body and inline comments from the author's captured rationale; open the PR.
- **`review`** (reviewer side) — read a diff against documented standards and the originating spec; report findings.

Don't confuse the two. They are complements, not alternatives.

## Process

### 1. Pin the fixed point and gather context

Capture the comparison once. Use the user-supplied fixed point if given; otherwise default to `main`. Record:

- `git diff <fixed-point>...HEAD` — the three-dot diff against the merge-base.
- `git log <fixed-point>..HEAD --oneline` — the commit list, in order.
- The branch name and the current upstream tracking state (will we need to push?).

Identify the **feature slug** from the branch name or `.scratch/` layout — `.scratch/<feature>/` is the convention `to-issues` and `triage` already use.

### 2. Collect rationale sources

Read everything the skill will transport. Do not synthesise; just collect:

- **Breadcrumbs** at `.scratch/<feature>/decisions.md`. If absent, that's fine — the file is optional.
- **ADRs** in `docs/adr/` (or per-context `docs/adr/` directories if `CONTEXT-MAP.md` exists). Only the ones whose subject touches files in the diff are relevant.
- **Originating PRD/issue** — look up issue references in the commit messages (`#123`, `Closes #45`, etc.) via the workflow in `docs/agents/issue-tracker.md`. If none, look under `.scratch/<feature>/` for a `prd.md` or `issue-*.md`. If still nothing, ask the user.

If any of the three is missing entirely, note it and continue — the corresponding section of the PR body is omitted, not stubbed.

### 3. Assemble the PR body

The house template — sections always in this order; any sub-section whose source is empty is omitted entirely:

```md
## Problem
<from PRD/issue's Problem Statement, lifted verbatim or lightly trimmed>

## Solution
<one-paragraph synthesis of the approach, drawn from the ADRs you collected
in step 2 and the high-level shape of the diff. Don't invent rationale —
if there are no ADRs in the touched area and no breadcrumbs, keep this
paragraph factual and short.>

**Alternatives considered**
<bulleted list. Each bullet is one breadcrumb with kind: pr-body,
or one ADR's "considered options" section. Omit this sub-section
entirely if both sources are empty.>

**Breaking Changes**
<Yes or No. If yes, list the breaking changes and any migration steps.
Take this from the PRD's Breaking Changes section if it has one,
otherwise infer cautiously from the diff and flag uncertainty.>

## Tests
<test plan checklist from PRD/issue. If the PRD has a Tests section,
copy its checklist. If not, omit this section.>

## Review guide
1. `<short-sha>` — <commit subject>
2. `<short-sha>` — <commit subject>
...
```

(The Screenshots sub-section under Solution is added by the frontend branch of this skill — out of scope for this section. When that branch is implemented, it slots between **Alternatives considered** and **Breaking Changes**.)

### 4. Build the inline-comment list

Walk the breadcrumbs file. For each entry with `kind: inline`, produce one pending inline comment:

```
{
  file: "<path>",
  line: <number>,
  body: "<why>\n\n**Alternatives:**\n- <option>: <why rejected>"
}
```

Only `kind: inline` breadcrumbs produce inline comments. `kind: pr-body` breadcrumbs go in the PR body's "Alternatives considered" sub-section, not as line comments. If a breadcrumb is malformed (missing `file` or `line` when `kind: inline`), surface it to the user as a fix-up rather than silently dropping it.

### 5. Preview to the user

Before any shared-state action, show the user:

- The assembled PR body (rendered markdown, exactly as it will be posted).
- The inline-comment list — file, line, and body of each pending comment.
- The push plan — current branch, remote tracking status, whether `git push -u` is needed.

Invite edits. The user may rewrite the body inline, drop or revise any inline comment, or cancel entirely. Capture their final approval before proceeding.

### 6. Open the PR

Only after explicit confirmation:

1. Push the branch with `git push -u origin <branch>` if it has no upstream; otherwise plain `git push`.
2. `gh pr create --title "<title>" --body-file <tmp>` where the title is taken from the originating issue/PRD if present, else the first commit subject.
3. For each pending inline comment, `gh api repos/{owner}/{repo}/pulls/{n}/comments` posting the comment on `file:line`. Use `commit_id` of the PR head and `side: "RIGHT"` so the comment anchors to the new code.
4. Print the PR URL.

No commit-discipline gate. If the commits are wide, the Review guide will reflect that — but the skill does not block.

## What this skill does NOT do

- It does not generate rationale by reading the diff and inventing alternatives. If breadcrumbs and ADRs are silent, the PR body is silent on those points too.
- It does not rewrite or re-slice commits. Commit discipline lives upstream in `docs/agents/commit-style.md` and is followed by the implementation loop.
- It does not auto-invoke after `tdd` finishes. `tdd` nudges the user; the user runs this skill when ready.
- It does not handle frontend screenshots yet — that branch is tracked separately and slots into step 3 once implemented.
