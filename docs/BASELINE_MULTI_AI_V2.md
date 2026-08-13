# Multi-AI V2 — Post-Merge Production Baseline

Recorded by the Release Manager immediately after PR #1 merged into `main`, updated after PR #2
(Risk C fix), updated again after PR #3 (Gemini Summary UI), and updated again after PR #5
(Milestone 5 — Practice Loop Closure, which also carried the Vocabulary UI integration onto `main`).
This document is the official record of what `main` contains; superseded sections are updated in
place with the new fact and the prior value preserved inline as history, per
`.ai-company/WORKFLOW.md`'s append-not-hide rule, rather than silently deleted.

## Current official baseline (updated after PR #5)

- **`main` HEAD (merge commit):** `3b5f57dce1dde2270aa0fb07ef778b7b9eb8ecf4` (merge of PR #5,
  "SAT Studio — Practice Loop Closure: Retry & Reinforcement," reviewed head
  `d3109c9a7a5b92f44e05863ce4c84b162c16b1e6` confirmed contained in the merge).
- **`index.html` SHA-256 (current, as of PR #5):**
  `4c998ef1df47d17405e586f448aa291924668d58a9932e61e8f666bb62900d44`
- **`LATEST_GOLD_MASTER_NEXT.html` SHA-256:** unchanged —
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba` (reconfirmed identical to every
  prior recording of this checksum; this file has never been modified since it was first committed).
- **Feature branch:** `feature/practice-loop-closure` (preserved on `origin`, not deleted), 10
  commits (an earlier packaging report in this session miscounted this as 12 — corrected here).
- **Source PR:** [`ohcha1/sat-study-2026#5`](https://github.com/ohcha1/sat-study-2026/pull/5) —
  merged via a normal (non-squash, non-rebase) merge commit.
- **Note on sequencing:** PR #4 ("Dictionary/Vocabulary Experience Upgrade," Milestone 4) merged
  into `main` before PR #5 but was not separately recorded in this document at the time — its content
  is included in the verification below since it's part of the current baseline regardless.

### Milestone 5 — Practice Loop Closure: Retry & Reinforcement (new in PR #5)

- **Status: MERGED / COMPLETE.**
- Closes the one previously-missing link in the student study loop (repeat practice after a wrong
  answer) via three additive, AI-independent pieces, plus a report-statistics safeguard:
  - **Retry missed questions** ("틀린 문제 다시 풀기") — only wrong questions reset and re-grade;
    originally-correct questions are protected at the data layer (`gradeRetry()` unconditionally
    skips any question already marked correct, independent of DOM state), not merely the UI.
  - **Evidence sentence** ("근거 문장") — surfaced in both the live quiz explanation and the
    wrong-answer report; `escapeHtml()`-protected at both render sites (this field is not escaped
    upstream in `makeQuestions()`, so this protection is load-bearing, not redundant).
  - **Weak-area reinforcement** — future quizzes lightly bias toward a student's weakest question
    type(s) (≥3 attempts, <60% accuracy, reusing the report tab's existing thresholds), capped at
    35% of a quiz; fails open with no history, insufficient history, guest sessions, or an
    unsupported type.
  - **Report-statistics filter** — retry attempts are excluded from aggregate accuracy/trend so a
    coached retry can't distort genuine unaided-performance history; still-wrong retries remain
    visible in the wrong-answer review list; pre-Milestone-5 records are unaffected (the new
    `attemptKind` field is simply absent on them).
- **Vocabulary UI now officially part of `main`:** the CEO-accepted-for-now Vocabulary Card state
  (from `feature/dictionary-visual-polish` @ `4725fdf`, which was never separately merged) was
  integrated into the Milestone 5 branch as one narrowly-scoped commit and rode along with this
  merge. `vocabCardTemplate()` on `main` is confirmed byte-identical to the `4725fdf` reference. Word
  Book, Review-state cycling, pronunciation, and the per-word concurrency hot-fix are all preserved.
- **No new AI provider, no IndexedDB schema change** — confirmed via direct diff (zero
  `createObjectStore`/`createIndex` lines in the PR's diff against its base) and via the schema still
  reporting exactly 6 object stores at every checkpoint.
- **Verification composition:** 11/11 PM-Spec acceptance criteria PASS; 20/20 CEO-required tests
  PASS; independent QA PASS with zero findings at any severity (including an adversarial
  DOM-manipulation test on the retry safeguard and an independently-chosen XSS payload); Principal
  Review verdict APPROVE with zero findings, based on direct re-inspection of the actual code rather
  than re-trusting prior reports.

### Verification performed at baseline-recording time (PR #5)

- `origin/main` HEAD confirmed `3b5f57dce1dde2270aa0fb07ef778b7b9eb8ecf4` via `git fetch`+
  `git rev-parse`.
- Reviewed head `d3109c9a7a5b92f44e05863ce4c84b162c16b1e6` confirmed an ancestor of the merge via
  `git merge-base --is-ancestor`.
- Vocabulary UI present on `main`: `word-card-left`/`word-card-right`/`level-chip`/
  `vocabCardTemplate` all found directly in `origin/main`'s `index.html`.
- Milestone 5 functions present on `main`: `gradeRetry`/`retryMissedQuestions`/
  `captureRetryAttempt`/`computeWeakTypes`/`biasForWeakTypes` all found directly in `origin/main`'s
  `index.html`.
- `LATEST_GOLD_MASTER_NEXT.html` SHA-256 read directly from `origin/main`, confirmed identical to
  every prior recording.
- IndexedDB schema: exactly 6 `createObjectStore` calls found in `origin/main`'s `index.html`,
  matching the schema's full historical count — no new store added.
- No API key or credential literal found anywhere in `origin/main`'s `index.html` by pattern search.

## Previous baseline (PR #3, preserved for history)

- **`main` HEAD (merge commit):** `12da7b4b6fc6d3d5e46fa188ced5a20e9af6e5d6` (merge of PR #3, "Add
  optional Gemini AI summary to SAT Studio," parents `cdd02dc698f504eb25f84d527353d6ed246d85ab` +
  `8ec8921c944fbe334f631945901621a43b275752`).
- **`index.html` SHA-256 (as merged into `main` at that point, since superseded above):**
  `7ab6e51243baf5151966d5b641c6cac711fb78dd800768f1d5cf7e9d929076ba`
- **`LATEST_GOLD_MASTER_NEXT.html` SHA-256:** unchanged —
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba` (reconfirmed identical to every
  prior recording of this checksum; this file has never been modified since it was first committed).
- **Feature commit:** `8ec8921c944fbe334f631945901621a43b275752` — "feat: expose optional Gemini
  summary in the Summary tab," on branch `feature/gemini-summary-ui` (preserved on `origin`, not
  deleted).
- **Source PR:** [`ohcha1/sat-study-2026#3`](https://github.com/ohcha1/sat-study-2026/pull/3) —
  merged via a normal (non-squash, non-rebase) merge commit.

### Gemini Summary UI (new in PR #3)

- **Status: MERGED / COMPLETE.**
- Adds an optional, user-triggered "AI 요약 (선택, Gemini)" card to the Summary tab. The existing
  local extractive summary (`summaryEn`/`summaryKo`) is **preserved and remains the default** —
  unchanged, always shown, never overwritten by the Gemini path.
- **Gemini behavior:** optional and dormant unless a dev key is configured. `overview()` checks the
  existing `getDevApiKey("gemini")` synchronously at render time — the action button only appears
  when a key is present; otherwise a disabled/informational note renders instead. No automatic
  request ever fires just from viewing the tab; only a button click triggers `loadGeminiSummary()`.
- **Privacy:** only `{text: state.analysis.text}` (the passage text) is sent to the adapter — no
  title, grade, date, id, or other metadata, matching the adapter's existing student-privacy
  comment. Independently confirmed by inspecting the captured request payload during testing (see
  PR #3's description).
- **Verification composition** (per the CEO's explicit direction during PR #3 review, since a
  second real-key browser test could not be performed without exposing the key to the controlled
  browser surface): **the real Gemini API was previously live-tested successfully at the adapter
  level** (see "Gemini adapter" under Feature status below — unchanged, this UI merely calls that
  same already-verified adapter). **The new UI path itself was verified end-to-end with a mocked
  Gemini response** — result renders correctly, local summary untouched, missing-key path fails
  closed at render time, a mocked failure response degrades gracefully without disturbing the local
  summary, and a double-click in-flight guard (same pattern as the Risk C fix) collapsed two
  overlapping calls into one request. A second live-key UI test was not performed for this reason,
  not because it was skipped without cause.
- Router/provider internals, translation logic, IndexedDB schema, and every other tab/function are
  unchanged — confirmed by direct diff (all changed hunks confined to `overview()` and the new
  `loadGeminiSummary()` function).

## Previous baseline (PR #2, preserved for history)

- **`main` HEAD (merge commit):** `0b60ba29951bb799c3fd3d8e30230e85636a19f0` (merge of PR #2, "Fix
  re-entrant analyze() race condition," parents `db4dd35d5ff08c422343981fd870ecc935f7f8f5` +
  `196fac857d35a6d1c59526e55c2ba46191c1d6dc`).
- **`index.html` SHA-256 (as merged into `main` at that point, since superseded above):**
  `d590bb41694e8769744ca6118fc03d5e019e1e7370dd9d23f12132d2daa70f66`
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

**Milestone 5 — Practice Loop Closure integration: COMPLETE** — see "Milestone 5 — Practice Loop
Closure: Retry & Reinforcement (new in PR #5)" above for full detail. Includes the Vocabulary UI
integration (Word Book/Review/pronunciation/concurrency hot-fix all preserved and confirmed on
`main`). No known open defect; QA and Principal Review both returned zero findings at any severity.

**Gemini Summary UI integration: COMPLETE** — see "Gemini Summary UI (new in PR #3)" above for full
detail.

- **QA result:** PASS (`docs/milestones/milestone-03/04-QA-REPORT.md`, plus a focused
  re-verification pass after the hot-fix, also PASS).
- **Gemini adapter:** live API smoke test previously PASS — provider `gemini`, model
  `gemini-3.1-flash-lite`, `success=true`, `error=null`, ~1261ms observed latency. The API key used
  for that test was browser-session-only and was never committed to this repository. As of PR #3,
  the adapter is reachable through an optional, user-triggered UI action (no automatic call site);
  it still requires a `window.SAT_STUDIO_DEV_KEYS.gemini` value to activate and fails closed
  without one.
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
Notes behind the current baseline (PR #5, Milestone 5 — Practice Loop Closure), see
`docs/milestones/milestone-05/`. For the Multi-AI V2 stabilization baseline (PR #1-#3), see
`docs/milestones/milestone-03/`. For the Vocabulary/Dictionary Experience Upgrade (PR #4, Milestone
4), see `docs/milestones/milestone-04/`.
