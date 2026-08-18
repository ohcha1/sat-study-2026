<!--
  Worked example of an archived LIGHT_CONTEXT.md — this is what .ai-company/LIGHT_CONTEXT.md
  looks like snapshotted into docs/milestones/milestone-XX/ once a LIGHT-mode milestone closes.
  Not a placeholder file — read it as a filled-in illustration.
-->

# LIGHT_CONTEXT — Milestone 0 — Fix pagination off-by-one on last page

Mode: LIGHT
Status: RELEASED

## Goal

The results list drops the last item when the total count isn't a multiple of the page size,
because `pageOffset()` computes the final page's slice end one index short.

## Acceptance criteria

1. Given a result set of 23 items and a page size of 10, page 3 shows items 21–23 (3 items), not
   21–22.
2. Given a result set that is an exact multiple of the page size (e.g. 20 items, page size 10),
   pagination is unaffected (2 pages of 10, no empty trailing page).
3. Given a result set smaller than one page, pagination is unaffected (1 page, no controls shown).

## Files allowed to change

- src/pagination.js

## Files prohibited from changing

- Anything in src/api/ — this is a pure display-logic bug, no data-fetching change needed.

## Architecture constraints

- Match existing function style in pagination.js (small pure functions, no new dependencies).
- No change to the page-size configuration or API contract.

## Latest relevant commit hashes

- a1b2c3d — Fix: pagination off-by-one in pageOffset() for final page

## Known open defects / risks

- None going in. Pre-existing and unrelated: page-size selector has no keyboard focus state
  (tracked separately, out of scope here).

## Test command

npm test -- pagination

## Handoff status

- Current stage: Released
- Next role: none — milestone closed
- Explicit stop point: N/A, closed

---

## Change log (append-only — one line per stage transition)

- 2026-08-01 — CEO — scope drafted and approved
- 2026-08-01 — Developer — fixed pageOffset(), commit a1b2c3d, handed off to QA
- 2026-08-01 — QA — all 3 acceptance criteria pass, no regressions in adjacent list/sort features,
  handed off to CEO
- 2026-08-01 — CEO — release approved
- 2026-08-01 — Release Manager — pushed to main, milestone closed
