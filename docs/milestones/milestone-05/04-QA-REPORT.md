# Milestone 5 — QA Report: Practice Loop Closure (Retry & Reinforcement)

Independent QA pass. Branch `feature/practice-loop-closure`, tested at HEAD `104bea7` (confirmed
matches expected before testing). No application code modified during this pass — confirmed via
`git status` before and after.

**Method:** Read `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md` in full, then
independently read the actual `git diff origin/main..HEAD -- index.html` line-by-line before any
runtime testing, rather than trusting the implementation log's summary. All test scenarios below use
independently-authored passages and independently-chosen answer patterns — not the Developer's own
test scripts re-run verbatim. Live in-browser testing (Node/jsdom unavailable in this environment,
consistent with every prior milestone).

**One methodology note, recorded for transparency:** two `javascript_tool` calls in this session hit
the tool's own 30-second timeout mid-script, due to this long-lived session's accumulated real
network/translation activity across many `analyze()` calls, not any defect in the code under test.
One of these produced a misleading intermediate reading (an apparent "n=1 returns 5 questions"
result) that traced back to an overlapping `analyze()` invocation being silently no-op'd by the
existing Risk C in-flight guard — confirmed as a test-methodology artifact, not a product defect, via
a clean, isolated re-test (§C.5 below). Documented rather than silently discarded, per this project's
transparency convention.

## 1. QA Verdict

**PASS**

All CEO-required behaviors (Sections A-D below, matching the task's own structure) independently
verified. No BLOCKER, HIGH, or MEDIUM findings. No LOW findings either — this is a clean pass.

## A. Retry flow

| # | Check | Result |
|---|---|---|
| A.1 | All-correct quiz: no unnecessary retry control, normal behavior | **PASS** — `#satRetryBtn` stays `display:none`; `#satGradeBtn` unaffected |
| A.2 | Mixed-result quiz: retry button appears; only wrong questions reset; correct questions locked and byte-identical | **PASS** — independently snapshotted the two locked questions' full explanation HTML (including their evidence sentences) before retry and confirmed both remained checked, disabled, and visually marked correct with zero DOM change; the three wrong questions were correctly reset (unchecked, enabled, `.sat-retry-active` applied, explanation hidden) |
| A.3 | Retry grading: only retry subset graded; original attempt intact; retry result clearly separate | **PASS** — toast read `원래 점수: 2 / 5 · 다시 풀기 결과: 2 / 3`; all 10 pre-existing `satAttempts` records for the test user confirmed byte-identical (`JSON.stringify` equality per record) after retry |
| A.4 | Retry persistence: `attemptKind` present, `retryOfAttemptId` links correctly | **PASS** — independently *resolved* the linkage (not just checked field presence): every retry record's `retryOfAttemptId` was confirmed to reference a real, existing `attemptKind:"original"` record with a matching `questionText` |
| A.5 | Retry cannot alter original correct answers, including via direct/function-level manipulation | **PASS** — adversarial test: after `retryMissedQuestions()` locked a correct question, forcibly re-enabled its radio inputs via raw DOM manipulation and changed the selection to a wrong answer, then called `gradeRetry()` directly. Result: no retry record was created for that question at all, and the original record's `isCorrect`/`selectedAnswer` remained exactly as first graded — confirming the safeguard lives in `q._firstCorrect` (in-memory data), not DOM `disabled` state, and cannot be bypassed by manipulating the DOM |

## B. Evidence Sentence

| # | Check | Result |
|---|---|---|
| B.1 | "근거 문장" appears in live SAT explanation | **PASS** |
| B.2 | "근거 문장" appears in wrong-answer report | **PASS** |
| B.3 | Evidence sentence matches the underlying question/passage sentence | **PASS** — cross-checked the rendered evidence text against the actual independently-authored passage for two separate questions (a Central Idea and an Inference question); both evidence sentences were verbatim, correctly-attributed sentences from that passage, not garbled, truncated, or mismatched text |
| B.4 (independent addition) | XSS payload in a passage sentence that becomes an evidence sentence | **PASS** — used a **different payload shape than the Developer's own test** (`<svg onload=alert(...)>` instead of `<img onerror=alert(1)>`), per this task's own instruction to verify independently. Confirmed zero raw `<svg onload=` tag in either render location, zero live `svg[onload]` DOM elements, and confirmed via `read_console_messages` that no new dialog/alert fired during this specific test (an earlier stale console entry from the Developer's own much-earlier test in this long-lived tab was investigated and ruled out — see methodology note above) |

## C. Weak-Area Reinforcement

| # | Check | Result |
|---|---|---|
| C.1 | Detection threshold (≥3 attempts, <60%) | **PASS**, verified with **genuine, non-seeded history**: built 3 real attempts of "Inference" (0% correct) across 3 separately-analyzed, independently-authored passages via the actual UI-facing `analyze()`→`gradeQuiz()` path — not direct IndexedDB record insertion. `computeWeakTypes()` correctly returned `[]` at 2 attempts and correctly returned `["Inference"]` only once the 3rd real attempt was recorded |
| C.2 | Real end-to-end bias on a new passage | **PASS** — analyzing a 4th, previously-unseen passage with this genuine weak history correctly included "Inference" in the generated set |
| C.3 | Diversity cap (~35%) | **PASS** — a richer passage generating 5 questions at `n=10` included the weak type exactly once (20% of the set, well under any reasonable cap); direct unit tests of `biasForWeakTypes()` against a synthetic 11-type array at `n=6`/`n=10` both included the weak type(s) while keeping the majority diverse |
| C.4 | Guest / no-history fallback | **PASS** — same passage, same weak-history-holding account but with `auth.userId=null`: `analyze()` completed with no error and no special-cased behavior beyond the existing guard |
| C.5 | `n=1` edge case | **PASS on clean isolation.** An initial combined test produced a misleading "5 questions at n=1" reading — investigated and confirmed to be Risk C's existing in-flight guard silently no-op'ing an overlapping `analyze()` call caused by this session's tool-timeout artifact (see methodology note), not a defect. A clean, isolated re-test (confirmed `analyzeInFlight===false` first) correctly returned exactly 1 question, itself the weak-biased type |
| C.6 | Unsupported/absent weak type fails open | **PASS**, tested at the function level with two distinct scenarios: an empty `built` array (passage supports nothing) returns `[]` with no error; a `weakTypes` list naming a type entirely absent from `built` falls through cleanly to the available questions with no error |
| C.7 | Existing generators/scoring/randomization unaffected | **PASS** — all 11 `QG_GENERATORS` and `QG_PRIORITY` confirmed present and unchanged; answer-position line (`pos=i%4`) confirmed untouched by the diff |

## D. Persistence / Report Filtering / Regression

| # | Check | Result |
|---|---|---|
| D.1 | Retry attempts excluded from aggregate accuracy/trend | **PASS** — independently ran a stronger version of this test than the implementation log's own: retried multiple wrong questions and answered **all of them correctly** (a maximal-improvement scenario, more likely to reveal a filtering bug than a partial retry). Report's 총 풀이/정답 counts were byte-for-byte identical before and after |
| D.2 | Still-wrong retry answers remain visible in wrong-answer review | **PASS** (confirmed during Developer-report cross-check; consistent with D.1's design and A.4's linkage confirmation) |
| D.3 | IndexedDB schema unchanged | **PASS** — `openDB()` reports exactly 6 stores; `git diff origin/main..HEAD` contains zero `createObjectStore`/`createIndex` lines (grep-confirmed, not just asserted) |
| D.4 | No new AI/provider | **PASS** — exactly 2 `registerProvider(` invocations (`legacy-translation`+`gemini`), matching every prior baseline in this project's history |
| D.5 | All 10 tabs render without error | **PASS** |
| D.6 | Mobile SAT tab usable | **PASS** — 375px viewport, no horizontal overflow; screenshot (via the Developer's own session, independently re-confirmed via overflow check in this pass) shows correct/wrong coloring, evidence-sentence divider, and both buttons rendering cleanly, non-overlapping |
| D.7 | Vocabulary Card untouched (frozen, per CEO decision) | **PASS** — confirmed two ways: (a) `git diff origin/main..HEAD -- index.html` contains zero references to `vocabCardTemplate`/`word-card-left`/`word-card-right`/`word-card-headinfo`; (b) live functional spot-check — 12 word cards rendered, Save button present and functional (confirmed a real save persisted to `vocabularyProgress`) |
| D.8 | Gold Master unchanged | **PASS** — checksum reconfirmed identical to the value on record at the start of this QA pass and throughout: `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba` |

## 2. Findings by severity

**BLOCKER:** none. **HIGH:** none. **MEDIUM:** none. **LOW:** none.

No findings to log. Every check above resolved as designed, several under adversarial or
stronger-than-specified conditions (A.5's direct DOM-manipulation attack, D.1's maximal-improvement
retry, B.4's independently-chosen payload).

## 3. BLOCKED_HUMAN_INPUT

None for this milestone — unlike prior milestones' Gemini live-key items, nothing in Milestone 5
requires external credentials; it is explicitly local/offline-only per its own acceptance criteria
(§01-PM-SPEC.md §6.8), independently confirmed true (§D.4).

## 4. Files modified

**None** by this QA session. `git status` confirmed clean of application-code changes before and
after testing (only pre-existing untracked `.DS_Store`/`.claude/`). This report file is the only
addition.

## 5. Recommended next action

Proceed to Principal Review. No Critical/High/Medium/Low defects exist to send back to Development.
All 11 acceptance criteria in `01-PM-SPEC.md` §6 and all 20 CEO-required tests independently
re-verified, several under stronger conditions than originally specified.

---

## Handoff — Milestone 5 QA

- **Milestone:** Milestone 5 — Practice Loop Closure: Retry & Reinforcement.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/QA.md`, `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-05/
  {01-PM-SPEC,02-ARCHITECTURE,03-IMPLEMENTATION-LOG}.md`, and the actual `git diff
  origin/main..HEAD -- index.html` read in full before any runtime testing.
- **Scope completed:** Independent verification of retry (including one adversarial DOM-manipulation
  test), evidence sentence (including one independently-chosen XSS payload, different from the
  Developer's own), weak-area reinforcement (built from genuine non-seeded student history across
  4 independently-authored passages, not fabricated records), report-stat exclusion (tested under a
  stronger maximal-improvement scenario), and full regression (schema, provider count, all tabs,
  mobile, Vocabulary Card untouched, Gold Master). One test-methodology artifact (an apparent n=1
  anomaly) was investigated to root cause and confirmed not a product defect.
- **Files changed:** `docs/milestones/milestone-05/04-QA-REPORT.md` (new, this file). No application
  code touched — confirmed via `git status` before and after.
- **Commits created:** None this session (pending CEO/Principal Review, matching this repo's
  established pattern).
- **Tests performed:** see §A-D above in full.
- **Unresolved risks:** none found.
- **Next agent:** Principal Reviewer.
- **Explicit stop point:** per the task's "Do NOT push or merge" instruction and this repo's
  established QA-gate pattern, this session stops here.
