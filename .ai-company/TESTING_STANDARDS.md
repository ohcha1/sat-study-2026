# TESTING_STANDARDS.md

Applies during the Development and QA stages of `.ai-company/WORKFLOW.md`. This repository has no
automated test framework installed as of Milestone 1. Until Milestone 8 (Deployment) or an earlier
milestone explicitly introduces one, testing is manual/scripted per the method below. This
document applies to both the Senior Developer (verifying own work before
handoff) and the QA Engineer (independent verification).

## Verification method (current, pre-framework)

Following the precedent set in Milestone 1:

1. Write a throwaway verification script (e.g., jsdom-based) that loads `index.html`, executes its
   scripts, and exercises the specific function(s)/flow(s) changed.
2. Assert on DOM output and/or in-memory state, not just "it ran without throwing."
3. Keep these scripts as scratch files — they are not checked into the repository unless a future
   milestone formally introduces a test suite.
4. Before final handoff, run a **combined regression sweep**: re-run all relevant prior milestone
   checks together, not just the current change in isolation.

## What to test

- Every acceptance criterion in the milestone's `01-PM-SPEC.md`, individually.
- Adjacent features that share state, DOM elements, or code paths with the change, even if not
  explicitly in scope — this codebase's features are tightly coupled in a single file, and
  regressions have historically been found this way (see the Milestone 1 follow-up inspection).
- Error/edge-case paths, not just the happy path: empty input, corrupted `localStorage`, API
  failure/timeout, rapid duplicate actions, XSS-shaped input where user or API content reaches the
  DOM.
- Cross-tab/cross-feature interactions where relevant (e.g., save/load/search/delete, all 8 result
  tabs).

## Defect severity definitions

- **Critical** — security vulnerability (e.g., XSS), data loss, or a crash that blocks core
  functionality.
- **High** — a feature is broken or produces incorrect results for realistic input, but doesn't
  crash the app or lose data.
- **Medium** — a feature works but with a notable usability or correctness gap under less common
  conditions.
- **Low** — cosmetic, edge-case, or a pre-existing/latent issue not reachable in practice.

See `.ai-company/DEFINITION_OF_DONE.md` for which severities block release.

## Reporting

QA reports go in `04-QA-REPORT.md` with: acceptance criterion, pass/fail, defect ID (if failed),
severity, reproduction steps, and which commit (if any) introduced or fixed it. Do not overwrite
prior loop iterations — append.
