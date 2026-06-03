# False-positive filter

Adapted from Anthropic's `/code-review`. Findings the user dismisses cost more credibility than findings you miss. Every finding from every axis passes through this before aggregation.

## How to run it

**One verification sub-agent per finding, run on Haiku, in parallel.** Each verifier gets only the diff, the one finding, and the standards-file list — **not** the originating axis's reasoning. The finder is the worst judge of its own finding; independence is the point. This fan-out is cheap precisely because it's Haiku + single-finding context + parallel — keep it that way (don't promote it to the review model, don't hand one agent all findings).

Spawn each with `Agent(..., model: "haiku")` (or, in a Workflow, `agent(prompt, {model: "haiku"})`). Each scores confidence **0–100** that the finding is a *real, in-scope* issue, and returns the score + one line of evidence.

The verifier must first **locate the finding in the diff** using its cited `file:line` + quoted code. If the line doesn't exist, the quote doesn't match, or the finding has no anchor at all, score it **0** — an unlocatable finding is unfalsifiable and is exactly how hallucinated findings slip the gate.

For findings flagged against a documented standard, the verifier must double-check the cited doc actually calls out that specific issue — don't let "CLAUDE.md says so" stand in for a rule that isn't there.

### Dedup (the one thing the per-finding pass can't do)

Per-finding verifiers can't see each other, so the same line flagged by two axes (commonly Architecture **and** Divergent) survives twice. After scoring, the orchestrator merges findings that target the same file+line into one, keeping the highest score and citing both axes. This is a cheap text-merge, not another agent.

### Rubric (give to the scoring agent verbatim)

- **0** — Not confident. False positive that doesn't survive light scrutiny, or a pre-existing issue.
- **25** — Somewhat confident. Might be real, might not; couldn't verify. If stylistic, not explicitly called out in the relevant standards doc.
- **50** — Moderately confident. Verified real, but a nitpick or rare in practice; low importance relative to the PR.
- **75** — Highly confident. Double-checked; very likely hit in practice, the PR's approach is insufficient, or it's directly named in a standards doc.
- **100** — Certain. Evidence directly confirms a real issue that will happen frequently.

### Threshold

Drop everything below **80**. Architecture and Divergent findings are inherently judgement calls — score them on whether the *reasoning* holds and the alternative is genuinely better, not on certainty of a bug; surface the strongest few rather than padding. If an axis has no finding ≥ 80, report that axis as clean.

## Always false positives — drop without scoring

- Pre-existing issues on lines the PR didn't touch.
- Looks like a bug but isn't.
- Pedantic nitpicks a senior engineer wouldn't raise.
- Anything a linter, typechecker, formatter, or compiler catches (imports, type errors, formatting, style nits). Assume CI runs these — don't run builds yourself.
- General code-quality gripes (test coverage, docs, broad "security") unless a standards doc explicitly requires it.
- Issues a standards doc names but the code explicitly silences (e.g. a lint-ignore with reason).
- Functionality changes that are clearly intentional and part of the broader change.

## No silent truncation

If you cap a list or drop borderline findings, say so in one line during aggregation — "dropped 3 findings below threshold" — so the user knows the filter ran, not that the code was spotless.
