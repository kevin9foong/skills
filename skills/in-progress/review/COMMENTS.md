# Posting findings as PR comments

Only when the user asks to post to the PR. A surviving finding (≥ 80, per [FILTER.md](FILTER.md)) becomes a comment. The goal is **mental alignment with the author** — they should finish reading and understand the change you'd make and *why*, without re-deriving it.

## Where to post

- **Inline, on the line(s).** Anchor each finding to the specific lines it concerns via `gh` (e.g. `gh pr comment` for general, or the PR review API for line-anchored comments). On-the-LoC comments are far easier for the author to act on than a wall of prose at the bottom.
- **One optional summary comment**, only if it adds value: what to focus on first, what the review covered, what it deliberately skipped. Skip it if the inline comments speak for themselves — don't post a summary that just restates them.

## When there's nothing to post (empty / low-signal)

A finding-free result is a valid result — never manufacture nits to look busy (the false-positive list in [FILTER.md](FILTER.md) exists to prevent exactly this). If nothing survives the filter, follow Anthropic's `/code-review` no-issues path: post a single brief summary naming what was checked — e.g. *"No blocking issues found. Checked standards, spec, architecture, and alternatives."* — and no inline comments. If only low-value `note:`/`praise:` items survive, post those and nothing more. The reviewer is allowed to find nothing.

## Re-reviewing a PR that already has a review

Before posting, check whether this tool already left a review (look for prior `🤖 ...AI code review` comments). If it did, **do not repost** — re-evaluate against what changed since:

1. **Read the prior comments and the author's replies.** If the author pushed back with a load-bearing reason (as in the subpath-export thread — *"happy to keep things consistent"*), treat that thread as resolved; don't re-raise it.
2. **Read the commits pushed since the last review.** A finding the author has since fixed is resolved — confirm the fix rather than re-flagging it; a brief `praise:` or resolving reply is enough.
3. **Post only the delta:** genuinely new findings introduced by the new commits, and follow-ups on prior comments that are still unaddressed *and* weren't reasonably declined.

Skip entirely if the PR is closed, merged, or draft. The goal is a conversation that moves forward, not a re-run that repeats itself.

## Conventional comments

Follow the [Conventional Comments](https://conventionalcomments.org/) spec. Format:

```
<label> [decorations]: <subject>

[optional discussion]
```

Lead every comment with a label so the author knows the weight before reading the subject. The nine strongly-suggested labels:

- **praise** — highlight something positive. Leave at least one sincere praise per review; never false praise.
- **nitpick** — trivial, preference-based. Non-blocking by nature — decorate `(non-blocking)`.
- **suggestion** — propose a concrete improvement. Be explicit about *what* and *why*; pair with the rationale.
- **issue** — a specific problem. Strongly pair it with a `suggestion`; if you're not sure a problem exists, use `question` instead.
- **todo** — small, trivial, but necessary change before acceptance.
- **question** — you have a potential concern but aren't sure it's relevant; ask the author to clarify or investigate.
- **thought** — a non-blocking idea the review surfaced (natural fit for Divergent/Architecture findings).
- **chore** — a process task needed before acceptance; link to the process so the author knows how to resolve it.
- **note** — non-blocking; simply highlights something the author should take note of.

More expressive optional labels: **typo** (a `todo` where the issue is a misspelling), **polish** (a `suggestion` where nothing is wrong but quality can improve), **quibble** (a `nitpick` without the connotations). Diverge from the list only when it genuinely reads better.

**Decorations** — parenthesised, comma-separated, after the label. Blocking semantics: `(non-blocking)` must not block acceptance, `(blocking)` must be resolved first, `(if-minor)` resolve only if the fix is trivial. Decorations can also **categorise**: `(security)`, `(ux)`, `(test)` — e.g. `suggestion (security):` or `issue (ux, non-blocking):`. Keep them minimal; a pile of decorations hurts readability.

## Write the rationale like a human (`/unslop`)

Reference the `/unslop` discipline: cut AI slop. No jargon the author has to decode (**don't** write "altitude", "leverage the synergy", "at a high level"). No throat-clearing, no restating the diff back at them, no hedging stacked on hedging. Say the thing, give the reason, propose the change. Plain words. Short.

Every comment needs the **why** — the cost the author avoids by taking the suggestion — not just the *what*.

## How the words land (Conventional Comments [communication](https://conventionalcomments.org/communication/))

`/unslop` makes a comment clear; these make it collaborative. An AI reviewer carries no rapport and its "tone" comes *only* through word choice — so follow the spec's five communication rules:

- **Be curious.** Don't assume you have all the context — the author may have already tried your idea or hit a hidden constraint. Unless you're certain, ask rather than assert. Prefer `question: Could we solve this in X?` over `suggestion: This should be solved in X.` This matters doubly for the Divergent/Architecture axes, where you're proposing the author *didn't* consider something — frame it as a question, not a verdict.
- **Patient mentoring pays off.** Explain kindly and completely; a reviewer who learns *why* writes better code next time. The `why` is the teaching.
- **Leave actionable comments.** Make it really clear how each comment is resolved. If there's no obvious path forward, say that explicitly too.
- **Combine similar comments.** Don't bury the author in dozens of tiny comments — batch the same issue across the diff into one comment (with a patch/snippet where it helps). One `polish:` covering every `m_x → x` rename beats twenty.
- **Replace "you" with "we".** "you should write tests" puts focus and blame on the person; `todo: We should add tests here` is equally direct but collaborative. The work is something *we* do.

## AI disclaimer

End each comment (or the summary, if findings are grouped under it) with:

> 🤖 This comment was generated by an AI code review. Please verify before acting on it.

## Length calibration — same finding, three ways

**Too long** (jargon, over-explained, restates the diff — what *not* to do):

> Architectural suggestion (altitude): expose the adapters via a subpath export instead of decorating the factory function. This loop copies named exports onto the default factory function so CJS consumers can reach adaptV3ToV4 / adaptV4ToV3. It works, but it's a workaround for overloading one entry point with two unrelated responsibilities… [six more sentences] …prefer explicit named assignments over the dynamic loop so the export surface is legible.

**Right** (label, the change, the why, context, scoped):

> **suggestion:** instead of a loop copying named exports onto the default factory function so CJS consumers (the backend) can reach `adaptV3ToV4` / `adaptV4ToV3`, shall we use the existing `packages/sdk/package.json` subpath-export approach to keep things consistent?
>
> For context, `"./dist/types"` faced the same issue and was resolved with subpath exports, as in our `formsg-shared` package.
>
> ```json
> "./adapters": {
>   "import":  { "types": "./dist/esm/adapters.d.ts", "default": "./dist/esm/adapters.js" },
>   "require": { "types": "./dist/cjs/adapters.d.ts", "default": "./dist/cjs/adapters.js" }
> }
> ```
>
> Two benefits: avoids a named export colliding with an intrinsic function property (`name`, `length`) being silently clobbered, and keeps one method in the codebase for resolving these exports.
>
> 🤖 This comment was generated by an AI code review. Please verify before acting on it.

**Also right** (a one-liner is fine when the finding is simple):

> **question:** since this is a one-way door, shall we remove this function too? We can recover it from git history, and keeping it makes it unclear which decryption pipeline is canonical.

The author's reply on the example above — *"agree the loop wasn't optimal, happy to keep things consistent"* — is the signal you're aiming for: the suggestion landed because it named the existing pattern and the concrete cost, not because it was long.
