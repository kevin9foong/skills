---
name: prepare-for-review
description: Umbrella skill that prepares a branch for Review-by-commit. Creates the PR if none exists, then invokes walkthrough-commits and screenshot-views to populate the PR description with a commit walkthrough and frontend screenshots. Idempotent — re-runs refresh both sections without touching content outside the marker regions. Use when a branch is ready to ship and the developer wants a fully-formed PR description in one command.
---

# Prepare for Review

One command from ready-branch to reviewer-ready PR. Creates the PR if needed, then populates it with a commit-by-commit walkthrough and frontend screenshots.

This skill is a **renderer, not a gatekeeper** — it never rewrites git history and never blocks on commit quality. See `docs/adr/0002-prepare-for-review-is-a-renderer-not-a-gatekeeper.md`.

## Process

### 1. Ensure the branch is pushed

```
git rev-parse @{u}
```

If no upstream is set: tell the user to push the branch first (`git push -u origin <branch>`), then re-run. Exit.

### 2. Check for an existing PR

```
gh pr view --json number,url
```

- **No PR found**: run `gh pr create --title "<branch-name>" --body "<!-- placeholder -->"` to open the PR with a minimal body. Note the PR URL.
- **PR exists**: note the PR URL. No action needed here.

### 3. Invoke walkthrough-commits

Spawn a sub-agent with the `walkthrough-commits` skill. Pass the PR number so the sub-agent updates the correct PR's marker section.

### 4. Invoke screenshot-views

Spawn a sub-agent with the `screenshot-views` skill. Pass the PR number. If the sub-agent reports "No frontend changes detected", that is expected — continue.

### 5. Report

Print the PR URL and a one-line summary: "Walkthrough: ✓ | Screenshots: ✓ (or skipped)".

## Re-runs

Safe to run multiple times. Each sub-skill replaces only its own marker region. Content the developer writes outside the markers is never touched.

## Dependencies

- `gh` CLI authenticated
- `playwright` CLI available (for screenshot-views; gracefully skipped if absent)
- Branch pushed to remote with upstream set
