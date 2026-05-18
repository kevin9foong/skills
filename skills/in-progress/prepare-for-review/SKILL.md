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

The body must be **concise**. Reviewers skim — long bodies get ignored. Prefer plain language over jargon: write for someone who knows the product but not this specific change. If you need a domain term, the project's `CONTEXT.md` glossary defines the canonical ones; anything outside that list is suspect.

Section order is fixed. Any sub-section whose source is empty is **omitted entirely** — no "N/A", no placeholder.

```md
## Problem

<2–4 sentences from the PRD/issue. End with "Closes <issue-ref>" if there is one.>

## Solution

<short prose paragraph — what changed and why, at the level a teammate
would explain it in standup. Use ### sub-headings only if the change
has genuinely separate facets (e.g. ### Feature flagging, ### Migration).
Don't pad with restated context.>

**Alternatives considered**
<bulleted list. Each bullet = one breadcrumb with kind: pr-body, or one
ADR's rejected option. Keep each bullet to one sentence. Omit the
sub-section entirely if both sources are empty.>

**Breaking Changes**

<One line. "No - backwards compatible." or "Yes - <what breaks> + <migration step>".>

## Tests

<Grouped manual test cases. Each group has a bold title summarising
the scenario; the steps are a nested checklist. Pre-tick boxes the
author has already verified locally.>

**TC1: <scenario>**

- [ ] <step>
- [ ] <step>

**TC2: <scenario>**

- [ ] <step>

## Review guide

1. `<short-sha>` — <commit subject>
2. `<short-sha>` — <commit subject>
...
```

**Sourcing the Tests section.** Prefer the PRD's own Tests block if it is already in this TC-grouped format. Otherwise build TCs from the user-journey or acceptance-criteria sections of the PRD/issue, one TC per distinct scenario the change enables. If the PRD has no testable scenarios at all (rare — only for pure docs PRs), omit the section.

**Length budget.** Aim for the whole body to fit on one screen without scrolling for a reader on a 13" laptop. If you're over, the Solution paragraph is the first thing to cut — most reviewers will read the diff for the "how" anyway.

For frontend changes (detection + capture covered in step 3a below), the body gains a **Screenshots** sub-section slotted between **Alternatives considered** and **Breaking Changes**:

```md
**Screenshots**

| Page | Before | After |
| --- | --- | --- |
| `/path` | ![before](<raw-url>) | ![after](<raw-url>) |

<journey section only if the PRD has one>

**User journey: `<journey name>`**

1. ![step 1 — <caption>](<raw-url>)
2. ![step 2 — <caption>](<raw-url>)
```

When no frontend changes are detected, the Screenshots sub-section is omitted entirely — no empty table, no "N/A".

### 3a. Frontend detection and screenshot capture

Run this between steps 3 and 4 only if the diff looks frontend.

**Detection.** Apply a heuristic over the diff: does it touch any file with extension `.tsx`, `.ts`, `.jsx`, `.js`, `.vue`, `.svelte`, `.css`, `.scss`, or `.html` *inside a directory that looks frontend* (e.g. `app/`, `src/components/`, `src/pages/`, `web/`, `frontend/`, `client/`, depending on project layout). A backend TypeScript service that happens to have `.ts` files does NOT count as frontend.

When the heuristic is ambiguous (e.g. a monorepo where the changed file's role is unclear), do not guess — ask the user "I see changes to `<files>`. Should I capture screenshots?".

**Capture scope.** Before/after of every distinct page touched by the diff. If the originating PRD/issue has a "User journey" or equivalent step-by-step section, additionally capture each journey step in sequence.

**Playwright script.** Write or update a Playwright spec at `.scratch/<feature>/screenshots.spec.ts`. Keep the file after the run — it can be re-used or extended on follow-up PRs to the same feature. If the file already exists from a previous run, merge new page captures into it rather than overwriting.

The script:

1. For each touched page: navigate to it on the previous commit's build, screenshot it as `before-<slug>.png`; then on `HEAD`'s build, screenshot it as `after-<slug>.png`. (If running both builds is impractical in the project, capture only the "after" state and note the limitation in the PR body.)
2. For each journey step from the PRD: perform the step and screenshot as `journey-<n>-<slug>.png`.

**Running.** Invoke via the project's local Playwright (`npx playwright test`). If `npx playwright --version` returns nonzero, do not auto-install — emit a one-line instruction (`npm i -D @playwright/test && npx playwright install`) and skip the Screenshots sub-section. Note the skip in the preview so the user knows what they're missing.

**Committing images.** Write images to `.github/screenshots/<feature>/` on the working branch. Commit them in a single dedicated commit so the rest of the Review guide stays clean (e.g. `chore(screenshots): capture <feature> before/after`).

**Embedding.** The PR body references each image via a raw GitHub URL of the form `https://raw.githubusercontent.com/<owner>/<repo>/<branch>/.github/screenshots/<feature>/<file>.png`. Use the branch name (not a SHA) so the images update if the branch is amended before merge.

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
- It does not auto-install Playwright. If the project does not already have it, the Screenshots sub-section is skipped and the user gets a one-line install instruction.
- It does not rewrite image URLs to pin to a SHA. The raw GitHub URL points at the branch name, so amending the branch updates the embeds.
