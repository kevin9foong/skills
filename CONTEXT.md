# Matt Pocock Skills

A collection of agent skills (slash commands and behaviors) loaded by Claude Code. Skills are organized into buckets and consumed by per-repo configuration emitted by `/setup-matt-pocock-skills`.

## Language

**Issue tracker**:
The tool that hosts a repo's issues. Use a local `.scratch/` markdown convention. Skills like `to-issues`, `to-prd`, `triage`, and `qa` read from and write to it.
_Avoid_: backlog manager, backlog backend, issue host

**Issue**:
A single tracked unit of work inside an **Issue tracker** — a bug, task, PRD, or slice produced by `to-issues`.
_Avoid_: ticket (use only when quoting external systems that call them tickets)

**Triage role**:
A canonical state-machine label applied to an **Issue** during triage (e.g. `needs-triage`, `ready-for-afk`). Each role maps to a real label string in the **Issue tracker** via `docs/agents/triage-labels.md`.

**Review-by-commit**:
A reviewer practice where the reviewer steps through each commit in a PR individually rather than reading the squashed diff. Skills that *prepare* a branch for review (e.g. `prepare-for-review`) curate the commit sequence and write a PR description that explicitly directs the reviewer to consume it commit-by-commit.
_Avoid_: "commit-by-commit review" (use the canonical form)

**Scoped commit**:
A commit that satisfies the reviewer's contract: (a) what does this commit do, (b) why is it here, (c) does it build and pass tests on its own. A branch prepared for **Review-by-commit** consists of scoped commits arranged in **refactor-then-feature** order — behaviour-preserving refactors first, then the final commit(s) that make the behaviour change.
_Avoid_: "logical commit", "clean commit" (both too vague)

**Design rationale**:
The explanation of *why* a behaviour-changing commit was made the way it was — the alternatives considered and why they were rejected. Lives in the **commit message of the commit that introduces the decision** (typically a behaviour-changing commit; not in PR descriptions, not in ADRs). A single PR may carry multiple design rationales, one per design-decision commit. The PR description for a **Review-by-commit** PR mirrors each rationale back from its commit message so the reviewer doesn't have to dig, but the commit messages remain the source of truth.
_Avoid_: "design notes", "PR notes" (imply the wrong home); "the rationale" in the singular when discussing PRs (assume plural)

## Relationships

- An **Issue tracker** holds many **Issues**
- An **Issue** carries one **Triage role** at a time

## Flagged ambiguities

- "backlog" was previously used to mean both the *tool* hosting issues and the *body of work* inside it — resolved: the tool is the **Issue tracker**; "backlog" is no longer used as a domain term.
- "backlog backend" / "backlog manager" — resolved: collapsed into **Issue tracker**.
