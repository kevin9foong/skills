# Frontend Screenshots

Run only if the diff looks frontend.

## Detection

Heuristic: any file in the diff with extension `.tsx`, `.ts`, `.jsx`, `.js`, `.vue`, `.svelte`, `.css`, `.scss`, `.html` *inside a directory that looks frontend* (`app/`, `src/components/`, `src/pages/`, `web/`, `frontend/`, `client/`, etc.). A backend TypeScript service does not count.

If ambiguous (monorepo, unclear role), ask the user: "I see changes to `<files>`. Should I capture screenshots?".

## Capture scope

- Before/after of every distinct page touched by the diff.
- If the PRD has a "User journey" section, additionally capture each step.

## Playwright script

Write or update `.scratch/<feature>/screenshots.spec.ts` in the consuming repo. Keep after the run — re-used on follow-up PRs. If the file already exists, merge new captures rather than overwriting.

The script:

1. For each touched page: screenshot on previous-commit build as `before-<slug>.png`, then on `HEAD` build as `after-<slug>.png`. If running both builds is impractical, capture only `after` and note the limitation in the PR body.
2. For each journey step: perform and screenshot as `journey-<n>-<slug>.png`.

## Running

`npx playwright test`. If `npx playwright --version` returns nonzero, **do not auto-install** — emit `npm i -D @playwright/test && npx playwright install` to the user and skip the Screenshots sub-section. Note the skip in the preview.

## Committing images

Write to `.github/screenshots/<feature>/` on the working branch. One dedicated commit (`chore(screenshots): capture <feature> before/after`) so binary churn does not pollute code commits.

## Embedding

Raw GitHub URLs of the form:

```
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/.github/screenshots/<feature>/<file>.png
```

Use the **branch name**, not a SHA — so embeds update when the author amends the branch.

## PR body sub-section format

Slot between **Alternatives considered** and **Breaking Changes**:

```md
**Screenshots**

| Page | Before | After |
| --- | --- | --- |
| `/path` | ![before](<raw-url>) | ![after](<raw-url>) |

**User journey: `<journey name>`**  <!-- only if PRD has one -->

1. ![step 1 — <caption>](<raw-url>)
2. ![step 2 — <caption>](<raw-url>)
```

No frontend changes → sub-section omitted entirely.
