---
name: walkthrough-commits
description: Render a commit-by-commit walkthrough section into the PR description to enable Review-by-commit. Each commit's subject and body are rendered oldest-first so reviewers can follow the refactor-then-feature sequence. Owns a marker region in the PR body — idempotent on re-run. Use when preparing a branch for review, after amending a commit message, or when invoked by prepare-for-review.
---

# Walkthrough Commits

Render each commit's full message into the PR description so reviewers can step through the branch commit-by-commit. **Design rationale** (commit body text) appears automatically — no separate step needed.

## Process

### 1. Determine base

Use the merge-base with `main` or `master` unless the user specifies a different base:

```
git merge-base HEAD $(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')
```

### 2. Collect commits

```
git log --reverse <base>...HEAD --format="%H %s%n%b%n---COMMIT-END---"
```

Collect: short SHA, subject, body (may be empty).

### 3. Render the section

Build a markdown fragment between the marker delimiters:

```
<!-- prepare-for-review:walkthrough-start -->
## Walkthrough

Review commit-by-commit in the order below (oldest → newest). Each commit is a [Scoped commit](../CONTEXT.md) in refactor-then-feature order — behaviour-preserving changes come first, behaviour changes last.

### `<short-sha>` <subject>

<body — omit block if empty>

---
<!-- prepare-for-review:walkthrough-end -->
```

### 4. Update or print

- **PR exists** (`gh pr view` succeeds): read current body, replace content between markers, write back with `gh pr edit --body "..."`.
- **No PR**: print the rendered section to stdout. Tell the user to paste it into their PR description or run `/prepare-for-review` to create the PR automatically.

## Marker contract

This skill owns exactly the region between:
```
<!-- prepare-for-review:walkthrough-start -->
<!-- prepare-for-review:walkthrough-end -->
```

Never modify content outside these markers. Re-runs are safe — the section is fully replaced each time.
