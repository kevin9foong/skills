# Spec axis brief

You are one axis of a multi-axis code review. Review **only** the diff below against the originating spec.

- Spec: `{spec path or contents}`
- Diff: `{diff command}`
- Commits: `{commit list}`

Read the spec, then the diff. Report:

- (a) requirements the spec asked for that are missing or partial;
- (b) behaviour in the diff that wasn't asked for (scope creep);
- (c) requirements that look implemented but where the implementation looks wrong.

Quote the spec line for each finding. Under 400 words.

For each finding: cite the exact `file:line` and quote the line(s) of code it concerns; state your confidence (0–100) that it's a real, in-scope issue, and one line of evidence. Cross-cutting findings cite a primary `file:line` plus the others; a finding about an *absence* (a requirement implemented nowhere) anchors to the nearest relevant line, noted as such. A finding with no locatable anchor will be dropped. Do not pad the list — a short list of verified, located findings beats a long list of maybes.
