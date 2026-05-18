---
name: screenshot-views
description: Detect changed frontend views from the diff, capture single-shot screenshots via Playwright CLI, and embed them into the PR description. Owns a marker region in the PR body — idempotent on re-run. Skips silently if no frontend files changed. Use when preparing a branch for review, after a UI tweak, or when invoked by prepare-for-review. Requires Playwright CLI installed in the target repo.
---

# Screenshot Views

Capture single-shot screenshots of changed frontend views and embed them in the PR description so reviewers can assess UI changes without checking out the branch.

## Process

### 1. Detect frontend changes

```
git diff --name-only <base>...HEAD | grep -E '\.(tsx|jsx|vue|svelte)$'
```

If no matches: print "No frontend changes detected — skipping screenshots." and exit.

### 2. Resolve routes

Read `.scratch/prepare-for-review.json` if it exists:

```json
{
  "baseUrl": "http://localhost:3000",
  "routes": ["/dashboard", "/settings"]
}
```

If the file does not exist: ask the user for BASE_URL and the list of routes to screenshot. Write the answers to `.scratch/prepare-for-review.json` for future runs.

### 3. Capture screenshots

For each route, run:

```
playwright screenshot "<baseUrl><route>" "/tmp/pfr-screenshots/<slug>.png"
```

Where `<slug>` is the route with `/` replaced by `-` (e.g. `/settings/profile` → `settings-profile`).

Ensure Playwright is available: `npx playwright --version` or `playwright --version`. If neither works, tell the user to install Playwright and exit.

### 4. Render the section

Build a markdown fragment between the marker delimiters:

```
<!-- prepare-for-review:screenshots-start -->
## Screenshots

| Route | Preview |
|-------|---------|
| `/dashboard` | ![dashboard](/tmp/pfr-screenshots/dashboard.png) |
<!-- prepare-for-review:screenshots-end -->
```

### 5. Update or print

- **PR exists** (`gh pr view` succeeds): read current body, replace content between markers, write back with `gh pr edit --body "..."`.
- **No PR**: print the rendered section to stdout.

### 6. Clean up

Delete all files under `/tmp/pfr-screenshots/` after embedding.

## Marker contract

This skill owns exactly the region between:
```
<!-- prepare-for-review:screenshots-start -->
<!-- prepare-for-review:screenshots-end -->
```

Never modify content outside these markers. Re-runs are safe — the section is fully replaced each time.
