# Architecture axis brief

You are one axis of a multi-axis code review. Apply the lens of the `/improve-codebase-architecture` skill to the **changed code only**.

- Diff: `{diff command}`
- Commits: `{commit list}`

Read `CONTEXT.md` and any ADRs in the touched area first (use that vocabulary). Then look at the diff for **deepening opportunities**:

- **Shallow modules** introduced by the change — interface nearly as complex as the implementation.
- **Duplicated source of truth** — the same concept decided two (or more) different ways across the diff.
- **Leaky seams** the change introduces or widens.
- Apply the **deletion test** to anything you suspect is a pass-through.

For each: name the files, the friction, and a plain-English fix described in terms of **locality** and **leverage**. Respect PR scope — prefer fixes proportionate to the change; flag larger refactors as non-blocking. Don't re-litigate decisions recorded in ADRs unless the friction is real enough to reopen one. Under 400 words.

*Example of the kind of finding this axis catches:* a security check that decides "is this the new format?" two ways at once — peeking at a provenance key (`isFieldResponsesV4`) **and** reading a saved version number (`mrfVersion`), joined with `&&`, while the same number is read four other ways elsewhere (`=== 1`, `=== 2`, `!= null`, `!mrfVersion`). The fix is one source of truth (the saved version) behind one tiny helper (`isV4(submission)`), used everywhere — concentrating the decision in one place instead of forcing every reader to re-derive it.

For each finding: cite the exact `file:line` and quote the line(s) of code it concerns; state your confidence (0–100) that it's a real, in-scope issue, and one line of evidence. Cross-cutting findings cite a primary `file:line` plus the others; a finding about an *absence* (a pattern that should exist but doesn't) anchors to the nearest relevant line, noted as such. A finding with no locatable anchor will be dropped. Do not pad the list — a short list of verified, located findings beats a long list of maybes.
