# Milestone 5 — Release Notes: Practice Loop Closure (Retry & Reinforcement)

Release Manager packaging pass. Branch `feature/practice-loop-closure`, packaged at HEAD `a40d973`
(the Principal Review commit — the last change on this branch). No application code modified during
this packaging session — confirmed via `git status`/diff-scope check before and after (see §12).

**Purpose:** package this milestone's already-complete Development/QA/Principal-Review work for CEO
push approval. This document summarizes what ships; it does not re-run or re-judge any prior phase's
verdict.

## 1. Main feature

**Practice Loop Closure: Retry & Reinforcement.** Closes the one missing link in the student study
loop — passage → analysis → vocabulary → SAT questions → answer → explanation → wrong-answer review
→ **repeat practice** — every other stage already existed in the product; repeat practice did not.

## 2. Student-visible improvements

- **틀린 문제 다시 풀기** — retry only the questions a student got wrong, without re-analyzing the
  passage.
- **근거 문장 표시** — the passage sentence that grounds each question's correct answer, shown in
  both the live quiz and the wrong-answer report.
- **Original score + retry score separation** — the retry toast names both results explicitly
  (e.g. "원래 점수: 2/5 · 다시 풀기 결과: 2/3"), never conflating the two.
- **Lightweight weak-area reinforcement** — future quizzes lightly favor a student's weakest
  question type(s) when the passage supports them.
- **Preserved question diversity** — the weak-area bias is capped; the large majority of every quiz
  always follows the existing, unbiased selection.

## 3. Retry behavior

- Only missed questions are reset and re-attempted; originally-correct questions are locked (radios
  disabled) and structurally protected — `gradeRetry()` unconditionally skips any question already
  marked correct, independent of UI/DOM state, confirmed via an adversarial direct-manipulation test
  in QA.
- The original attempt's `satAttempts`/`wrongAnswers` records are never read or written by the retry
  path — confirmed via before/after byte-comparison of all original records.
- The retry attempt is stored as new, separate records, tagged `attemptKind:"retry"` with
  `retryOfAttemptId` linking back to the original question's attempt — additive fields only, no
  schema change.

## 4. Evidence Sentence

- Displayed in the live quiz's explanation panel and in the wrong-answer report card.
- `escapeHtml()`-protected at both render sites — `evidenceSentence` is the one question field that
  is not escaped upstream in `makeQuestions()`, so this render-time escaping is load-bearing, not
  redundant; verified against an XSS payload independently by QA with no live injection.
- No internal field name (e.g. "evidenceSentence") is ever exposed — always labeled "근거 문장" in
  the student-facing UI.

## 5. Weak-area reinforcement

- Reuses existing `satAttempts` history — no new tracking mechanism.
- Detection threshold: **≥3 attempts** in a question type, considered weak if **<60% accuracy** —
  both numbers reused from the report tab's own pre-existing "weak" convention, not invented.
- Bias capped at **35%** of a given quiz's question count (within the approved 30–40% range).
- **Guest / no-history:** fails open — no bias applied, no error, identical to pre-milestone
  behavior.
- **Unsupported weak type:** if the current passage's generators don't produce that type, the bias
  is a no-op by construction — confirmed via both unit tests and a live fabricated-type test.

## 6. Statistics

- Retry attempts are excluded from aggregate accuracy/monthly-trend/per-type/per-skill statistics —
  confirmed via before/after report comparison, including a maximal-improvement retry scenario (all
  retried questions answered correctly) that still left the aggregate numbers unchanged.
- A still-wrong retry answer remains visible in the wrong-answer review list — a genuine,
  unresolved gap is never hidden.
- Backward compatibility: `attemptKind` is absent on every pre-Milestone-5 record; every new read
  site treats an absent value as "original," so historical data and its statistics are unaffected.

## 7. Vocabulary UI integration

- The CEO-accepted-for-now Vocabulary Card state (`feature/dictionary-visual-polish` @ `4725fdf`)
  was integrated into this branch, since it had never been merged to `main`.
- `vocabCardTemplate()` confirmed **byte-identical** to the `4725fdf` reference via direct diff — the
  integration carried over only the final accepted presentation state, not the branch's four
  intermediate, superseded iteration commits.
- Word Book (save/unsave/reconstruct), Review-state cycling, pronunciation, and the per-word
  concurrency hot-fix all reconfirmed functional after integration.

## 8. QA

- **All 11 acceptance criteria (`01-PM-SPEC.md` §6): PASS.**
- **All 20 CEO-required tests: PASS** (`03-IMPLEMENTATION-LOG.md`).
- **Independent QA: PASS**, with zero findings at any severity — several checks run under stronger
  or adversarial conditions than originally specified (a direct DOM-manipulation attack on the retry
  safeguard, an independently-chosen XSS payload, weak-area history built from genuine non-seeded
  quiz attempts across 4 separate passages) (`04-QA-REPORT.md`).
- **End-to-end student-flow improvement confirmed:** a student can now retry what they got wrong,
  see the evidence behind each answer, and have future practice lightly adapt to their weak areas —
  all within the existing, already-verified analyze → quiz → report flow.

## 9. Principal Review

- **Final verdict: APPROVE**, no conditions.
- **No findings at any severity.** Every safety-critical property (retry's data-layer lock,
  evidence-sentence escaping, the weak-area cap, the report-statistics filter, the Vocabulary UI
  integration's scope) was independently re-verified by direct code inspection, not by re-trusting
  prior reports (`05-REVIEW-REPORT.md`).

## 10. Preservation

Confirmed unmodified throughout Development, QA, Principal Review, and this packaging pass:
- All 11 SAT question generators (`QG_GENERATORS`/`QG_PRIORITY`), scoring, and answer-position
  randomization.
- Vocabulary / Word Book (frozen per CEO decision, then correctly integrated — see §7).
- Gemini Summary UI and Gemini vocab-context — both dormant without a key, unchanged.
- Translation architecture (Multi-AI router, `legacy-translation` adapter).
- PDF/photo/OCR/HEIC import.
- Risk A (Chrome Translator timeout), Risk B (save-ID uniqueness), Risk C (`analyzeInFlight` guard).
- IndexedDB schema — still exactly 6 object stores, zero new stores or indexes across this entire
  milestone's diff against `origin/main`.
- Responsive/mobile behavior — the SAT tab's new controls reuse existing, already-touch-target-
  verified `.btn`/`.choice` classes; no new breakpoints.

## 11. Security

- No API key, secret, token, or credential committed anywhere in this milestone — confirmed by
  pattern search across every commit's diff, independently re-confirmed during Principal Review.
- `evidenceSentence` XSS protection independently verified: this field is not escaped upstream in
  `makeQuestions()` (a genuine finding surfaced during Development, not assumed safe), so both new
  render sites explicitly wrap it in `escapeHtml()` — verified against payloads with zero live DOM
  injection in both QA and this milestone's own testing.
- No new AI provider — `registerProvider(` remains exactly 2 invocations
  (`legacy-translation`+`gemini`) across the entire branch.

## 12. Release state

- **Local branch only:** `feature/practice-loop-closure`, 12 commits ahead of `origin/main`.
- **Not pushed.**
- **Not merged.**
- **CEO push approval required** before any push/merge/release action.

---

## Verification performed this packaging session

- `index.html` unchanged: confirmed via `git status`/`git diff` showing zero changes to
  `index.html` from this session (only new/edited documentation files staged).
- `LATEST_GOLD_MASTER_NEXT.html` (Gold Master) unchanged: checksum reconfirmed
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`.
- Only release/documentation files changed this session: `docs/milestones/milestone-05/
  06-RELEASE-NOTES.md` (new), `DEVELOPMENT_PLAN.md` (Milestone 5 numbering-note + track section
  added, append-not-hide — the existing "Milestone 5 — AI Tutor" section is unchanged and clearly
  distinguished as separate, unrelated work).
- No secrets: pattern search (`api[_-]?key|secret|token|password`, case-insensitive) across this
  session's diff returned no matches outside this document's own description of the security
  verification performed.

---

## Handoff — Milestone 5 Release Manager Packaging

- **Milestone:** Milestone 5 — Practice Loop Closure: Retry & Reinforcement.
- **Source documents read:** `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
  `04-QA-REPORT.md`, `05-REVIEW-REPORT.md`, `DEVELOPMENT_PLAN.md`.
- **Scope completed:** Packaged this milestone's Development/QA/Principal-Review results into this
  release-notes document; added a new "Milestone 5 (Practice Loop Closure track)" section to
  `DEVELOPMENT_PLAN.md`, clearly distinguished from the pre-existing, unrelated "Milestone 5 — AI
  Tutor" entry via a numbering note matching the established pattern from Milestones 3/4. No
  application code touched, no re-testing performed (this phase summarizes prior verdicts, it does
  not re-judge them).
- **Files changed:** `docs/milestones/milestone-05/06-RELEASE-NOTES.md` (new, this file),
  `DEVELOPMENT_PLAN.md` (new Milestone 5 numbering-note + track section, append-not-hide).
- **Commits created:** One documentation-only commit (see commit log for hash).
- **Tests performed:** N/A (Release Manager phase; verification limited to confirming no code/secret
  changes, per §12 above).
- **Unresolved risks:** None. No LOW/deferred items exist for this milestone (unlike Milestone 4,
  which carried a deferred LOW item and a `BLOCKED_HUMAN_INPUT` — neither applies here).
- **Next agent:** CEO — to grant or withhold push approval.
- **Explicit stop point:** Per the task's implicit Release Manager scope (no push/merge/release
  performed). No push, no merge, no release performed.
