---
name: review
description: Review the changes since a fixed point (commit, branch, tag, or merge-base) along four independent axes — Standards, Spec, Architecture, and Divergent design — each run as a parallel sub-agent so contexts stay clean, then verified against false positives and reported side by side. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to "review since X".
---

# Review

Multi-axis review of the diff between `HEAD` and a fixed point the user supplies. Each axis runs as its own **parallel sub-agent** so they don't pollute each other's context. Findings are then verified against false positives and aggregated **without merging the axes** — the user evaluates each independently.

The four axes:

- **Standards** — does the code follow the repo's documented standards? And are those standards themselves sound, or tech debt worth flagging?
- **Spec** — does the code faithfully implement the originating issue / PRD / spec?
- **Architecture** — are there deepening opportunities in the changed code? (lens of `/improve-codebase-architecture`)
- **Divergent** — for each change, was this the best option among the alternatives, given the PR's scope?

The issue tracker should have been provided to you — run `/setup-matt-pocock-skills` if `docs/agents/issue-tracker.md` is missing.

## Process

### 1. Pin the fixed point

Whatever the user said is the fixed point — a commit SHA, branch name, tag, `main`, `HEAD~5`, etc. Don't be opinionated; pass it through. If they didn't specify one, ask: "Review against what — a branch, a commit, or `main`?" Don't proceed until you have it.

Capture the diff command once: `git diff <fixed-point>...HEAD` (three-dot, against the merge-base). Note the commit list via `git log <fixed-point>..HEAD --oneline`.

### 2. Identify the spec source

Look for the originating spec, in this order:

1. Issue references in commit messages (`#123`, `Closes #45`, GitLab `!67`) — fetch via `docs/agents/issue-tracker.md`.
2. A path the user passed as an argument.
3. A PRD/spec file under `docs/`, `specs/`, or `.scratch/` matching the branch or feature.
4. If nothing is found, ask. If there's no spec, the **Spec** axis skips and reports "no spec available".

### 3. Identify the standards sources

Anything documenting how code should be written: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `CONTEXT.md` / `CONTEXT-MAP.md`, `docs/adr/`, `STYLE.md` / `STANDARDS.md`, and config files (`eslint.config.*`, `biome.json`, `tsconfig.json` — note them but don't re-check what tooling enforces). Collect the file list; the **Standards** sub-agent reads them.

### 4. Triage, then spawn the axis sub-agents in parallel

**Triage first (one judgment call, no formulas):** is there anything substantive to reason about?

- Unambiguously cosmetic — docs/comments/formatting only, a version bump, a generated-file change → run **Standards + Spec** only; skip Architecture and Divergent. Note that you did.
- Clearly automated or no-op (e.g. a dependabot bump that CI already gates) → skip the review with a one-line "nothing substantive to review."
- Anything else, or any doubt → run **all four**. Bias toward running more: a one-line change can be load-bearing (a tweak to a security check is exactly the `isV4` case). Collapse only when it's obviously just cosmetic.

Then send a single message with one `Agent` call per applicable axis (`general-purpose` subagent for all). **Don't read the diff or docs yourself** — stay a thin dispatcher. Tell each sub-agent to read its own brief file and follow it, passing only the diff command, commit list, and the source paths it needs:

- **Standards** → `axes/standards.md` (+ the standards-source paths from step 3)
- **Spec** → `axes/spec.md` (+ the spec path) — skip if no spec exists; note it in the report
- **Architecture** → `axes/architecture.md`
- **Divergent** → `axes/divergent.md`

Each brief caps its report at ~400 words and ends by asking for a confidence score + evidence per finding, so the filter in step 5 has what it needs.

### 5. Verify each finding — parallel, on Haiku

Collect the findings the axis agents returned. Spawn **one verification sub-agent per finding, on Haiku, in a single parallel batch** (`Agent(..., model: "haiku")`). Each gets only the diff, its one finding, and the standards-file list — never the finder's reasoning. It returns a 0–100 confidence score + one line of evidence. Then **dedup**: merge findings hitting the same file+line into one (keep the highest score, cite both axes), and **drop everything below 80**. Full rubric and the always-drop list: [FILTER.md](FILTER.md). This gate is what makes autonomous posting safe — a finding the author dismisses costs more credibility than a missed nit.

### 6. Aggregate

Present each surviving finding under its axis heading (`## Standards`, `## Spec`, `## Architecture`, `## Divergent`), lightly cleaned. Do **not** merge or rerank *across* axes — the separation is the point (the file+line dedup in step 5 is the only cross-axis merge). End with a one-line summary: surviving findings per axis, the single worst issue, and how many were dropped below threshold (no silent truncation).

### 7. (Optional) Post as PR comments

Only if the user asked to post. One comment per surviving finding, anchored inline on its line(s) via `gh`; add a summary comment only if it adds value. **If nothing survived, say so — don't manufacture nits.** **If the PR was already reviewed by this tool, re-evaluate against the prior comments, the author's replies, and commits pushed since — post only the delta, never a repost.** Skip closed/merged/draft PRs. Format, labels, AI disclaimer, empty-review path, and re-review logic: [COMMENTS.md](COMMENTS.md).

## Why separate axes

A change can pass one axis and fail another:

- Follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Does exactly what the issue asked but breaks conventions → **Spec pass, Standards fail.**
- Correct and conventional, but a shallow module or a worse-than-the-alternative choice → caught only by **Architecture** / **Divergent.**

Reporting them separately stops one axis from masking another.
