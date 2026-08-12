# Multi-AI V2 — Post-Merge Production Baseline

Recorded by the Release Manager immediately after PR #1 merged into `main`, and updated after PR #2
(Risk C fix) merged. This document is the official record of what `main` contains; superseded
sections are updated in place with the new fact and the prior value preserved inline as history,
per `.ai-company/WORKFLOW.md`'s append-not-hide rule, rather than silently deleted.

## Current official baseline (updated after PR #2)

- **`main` HEAD (merge commit):** `0b60ba29951bb799c3fd3d8e30230e85636a19f0` (merge of PR #2, "Fix
  re-entrant analyze() race condition," parents `db4dd35d5ff08c422343981fd870ecc935f7f8f5` +
  `196fac857d35a6d1c59526e55c2ba46191c1d6dc`).
- **`index.html` SHA-256 (current, as of PR #2):**
  `d590bb41694e8769744ca6118fc03d5e019e1e7370dd9d23f12132d2daa70f66`
- **`LATEST_GOLD_MASTER_NEXT.html` SHA-256:** unchanged —
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba` (reconfirmed identical to every
  prior recording of this checksum; this file has never been modified since it was first committed).
- **Fix commit:** `196fac857d35a6d1c59526e55c2ba46191c1d6dc` — "fix: prevent re-entrant analyze()
  runs (Risk C)," on branch `fix/analyze-reentry` (preserved on `origin`, not deleted).
- **Source PR:** [`ohcha1/sat-study-2026#2`](https://github.com/ohcha1/sat-study-2026/pull/2) —
  merged via a normal (non-squash, non-rebase) merge commit.

## Original baseline (PR #1, preserved for history)

- **`main` commit (merge commit):** `dc7dc9c42710c02e3d7e3beeaaa762c63cfa8b80`
  (`Merge pull request #1 from ohcha1/multi-ai-v2-latest-dev`, parents
  `d490e1aadfe55b0314a4179f024b5734b0d8abee` + `5c9e90a105675daeeb5ca37b0a57dd61d07e22a5`).
- **`index.html` SHA-256 (as merged into `main` at that point, since superseded above):**
  `e2337ddf8adaff0bc79bf356d09e2d841c257c710ad5990ae9519af35583357b`
- **Source PR:** [`ohcha1/sat-study-2026#1`](https://github.com/ohcha1/sat-study-2026/pull/1) — "SAT
  Studio Multi-AI V2 — Gold Master Adoption + Router + Reliability," merged via a normal (non-squash,
  non-rebase) merge commit, preserving full commit history.
- **Development branch preserved:** `multi-ai-v2-latest-dev` still exists on `origin` at
  `5c9e90a105675daeeb5ca37b0a57dd61d07e22a5` (not deleted by the merge).

## Verification performed at merge time (this session)

- `index.html` content identical between `origin/main` and `multi-ai-v2-latest-dev` tip — confirmed
  by direct diff; the merge commit did not alter file content.
- Multi-AI router present: `providerRegistry`/`aiRouter` definitions found.
- `legacy-translation` adapter present (translation routing).
- `gemini` adapter present (summary routing, dormant by default).
- No API key or credential literal found anywhere in `index.html` by pattern search.
- Risk A fix present: Chrome Translator initialization is raced against a timeout and fails open to
  MyMemory (`"chrome translator init timeout"` found at the expected call site).
- Risk B fix present: save-ID collision avoidance (`while(list.some(x=>x.id===id))id++`) found at
  the expected call site.

## Feature status

**Multi-AI V2 stabilization: COMPLETE** — all three risks identified during Milestone 3 QA are now
resolved; no known open defect remains against this baseline.

- **QA result:** PASS (`docs/milestones/milestone-03/04-QA-REPORT.md`, plus a focused
  re-verification pass after the hot-fix, also PASS).
- **Gemini adapter:** live-tested successfully — provider `gemini`, model `gemini-3.1-flash-lite`,
  `success=true`, `error=null`, ~1261ms observed latency. The API key used for that test was
  browser-session-only and was never committed to this repository. The adapter remains dormant by
  default for end users (no UI call site invokes it; it still requires a
  `window.SAT_STUDIO_DEV_KEYS.gemini` value to activate).
- **Risk A (Chrome Translator init hang):** **RESOLVED**, independently re-verified.
- **Risk B (save-ID collision / data loss):** **RESOLVED**, independently re-verified.
- **Risk C (re-entrant `analyze()`):** **RESOLVED** (previously deferred as MEDIUM backlog — see
  the preserved note below for the original deferral rationale). Fixed via a synchronous in-flight
  guard on `analyze()` wrapped in `try/finally`; a second call while one is already running is now
  a silent no-op. Fix commit `196fac857d35a6d1c59526e55c2ba46191c1d6dc`, merged via PR #2. Verified
  with 5 focused runtime tests (normal run, rapid double-call with no overlap, no translation
  mismatch, no unhandled rejection, existing tabs/save/retry unaffected) — see PR #2's description
  and `.ai-company/DEFINITION_OF_DONE.md` for the verification standard applied.
  *(Original deferral note, preserved for history: "not fixed — deferred. Severity: MEDIUM. Logged
  as a backlog item for a future milestone; does not block this baseline... see
  `docs/milestones/milestone-03/04-QA-REPORT.md`/`05-REVIEW-REPORT.md` for the full severity
  rationale.")*
- **Preserved from the Gold Master, confirmed unmodified:** IndexedDB schema (6 object stores),
  photo/PDF/HEIC import, all 7 responsive `@media` blocks (see
  `docs/milestones/milestone-03/05-REVIEW-REPORT.md` §3 for the direct-diff verification this claim
  is based on).

## Full history

For the complete PM-Spec, Architecture, Implementation Log, QA Report, Review Report, and Release
Notes behind this baseline, see `docs/milestones/milestone-03/`.
