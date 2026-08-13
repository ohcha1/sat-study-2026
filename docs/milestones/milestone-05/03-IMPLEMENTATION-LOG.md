# Milestone 5 — Implementation Log: Practice Loop Closure (Retry & Reinforcement)

Senior Developer session, executed under "Autonomous Continuation Mode: ON" per the CEO's explicit
approval to proceed through all 5 Developer Tasks without stopping for routine acknowledgment.
Implements `02-ARCHITECTURE.md` (APPROVED 2026-08-13) exactly.

**Branch:** `feature/practice-loop-closure`, forked from `origin/main` @
`9fa6e1bbd2f99b1e829c5e5b3c7a909e8265ed4a` (the `docs: add Milestone 5 PM-Spec and Architecture
Design` commit was cherry-picked on top from `feature/dictionary-visual-polish`, so this branch
carries the Milestone 5 planning docs without any of that other branch's unmerged, frozen
Vocabulary Card visual work — the two milestones' history is kept fully separable).

**Testing method:** live in-browser testing (Node/jsdom unavailable in this environment, same
constraint as every prior milestone on this line).

## Dev Task 1 — Retry Missed Questions (`ca13bdb`)

`sat()`: two new hidden buttons (`#satRetryBtn`, `#satGradeRetryBtn`), existing grade button given
`#satGradeBtn`. `gradeQuiz()`: additively stamps `q._firstSelectedIdx`/`_firstCorrect` per question
(runtime-only, on the existing in-memory question objects — no new `state` variable, no persistence
dependency), calls new `maybeShowRetryButton()`. New `retryMissedQuestions()` resets only wrong
questions' DOM state (radio, correct/wrong classes, explanation visibility) and adds a
`.sat-retry-active` marker; disables already-correct questions' radios for UX clarity (not the actual
safeguard — see below). New `gradeRetry()` grades only the retried subset and shows both scores in
one toast. New `captureRetryAttempt()` mirrors `captureSatAttempt()`'s exact record shape, tagged
`attemptKind:"retry"`/`retryOfAttemptId`, for the retried questions only. `captureSatAttempt()`
additively stamps `q._firstAttemptId` from the record it just built, and now writes
`attemptKind:"original"` explicitly.

Verified: an all-correct quiz never shows the retry control (Test 1); a mixed quiz's retry touches
only the wrong questions, confirmed both via DOM state (correct questions locked/disabled, unchanged)
and by diffing all 10 original `satAttempts` records before/after retry — byte-identical (Test 2/3);
`gradeRetry()` structurally cannot regrade a correct question regardless of DOM state, since it
unconditionally skips any `_firstCorrect===true` question (Test 4); the toast explicitly names both
scores (`원래 점수: 2 / 5 · 다시 풀기 결과: 2 / 3`) (Test 5); retry records carry
`attemptKind`/`retryOfAttemptId` correctly, and a still-wrong retry produces a new `wrongAnswers`
entry (Test 6/8).

## Dev Task 2 — Evidence Sentence (`2238008`)

Added an escaped `근거 문장:` line to `sat()`'s per-question explanation block and to
`reportView()`'s wrong-answer card template, both guarded so a question with no evidence sentence
renders no stray label. **Finding surfaced during implementation, not anticipated in the PM/Architecture
docs in this level of detail until Architecture caught it:** `evidenceSentence` is the one question
field `makeQuestions()` does *not* pass through `escapeHtml(englishOnly(...))` — it is raw passage
text, unlike `q.q`/`q.choices`/`q.why`. Rendering it for the first time without escaping at the render
site would have been a new XSS surface; both new render sites wrap it in `escapeHtml()`.

Verified: evidence sentence renders in both locations for every question that has one; an
`<img src=x onerror=alert(1)>` payload placed in a passage sentence that becomes a question's evidence
sentence is confirmed escaped in both the live quiz and the report card — zero live DOM elements
created, string-level escaping confirmed in both `innerHTML` outputs; no internal field name
("evidenceSentence") ever appears in rendered output.

## Dev Task 3 — Weak-Area Reinforcement (`cca703a`)

New `computeWeakTypes(attempts)`: reuses the report tab's own established "weak" convention
(`total>=3` matches `buildDeterministicSummary()`'s existing sample-size gate, `<0.6` matches
`typeRows`/`skillRows`'s existing `acc<60?"weak"` CSS threshold) rather than introducing new numbers;
excludes `attemptKind:"retry"` records; caps at the 2 weakest types, matching
`buildDeterministicSummary()`'s existing cap. New `biasForWeakTypes(built,n,weakTypes)`, wired into
`makeQuestions()` immediately before its existing final slice: preferentially includes up to
`Math.ceil(n*0.35)` weak-type questions (35%, the CEO-approved range's midpoint) if the passage's
generators produced any, fills the remainder normally, then re-sorts back into the existing
`QG_PRIORITY` display order. `analyze()` gains one guarded (`auth.userId` check + `try/catch`),
fail-open lookup immediately before its existing `makeQuestions()` call.

Verified: `biasForWeakTypes()` unit-tested directly against a synthetic 11-type `built` array —
`n=6`/single weak type correctly includes it within a 6-question diverse set; `n=10`/two weak types
both included among 10 diverse questions; empty `weakTypes` produces byte-identical output to a plain
slice (confirming the no-op path); `n=1` returns exactly 1 question with no error (documented edge
case, per Architecture §7). Live end-to-end via `analyze()` with seeded weak-history (5 attempts,
20% accuracy in one type) correctly biases real question generation. Guest mode and a
fabricated/unsupported weak type both complete `analyze()` without error, falling through to normal
generation. All 11 generators, `QG_PRIORITY`, and answer-position assignment (`pos=i%4`, pre-existing
and untouched) confirmed unchanged.

## Dev Task 4 — Report Persistence Filter (`ac330f2`)

`renderReportAsync()` now filters `attemptKind!=="retry"` before computing every aggregate stat
(total/correct/wrong, monthly trend, per-type/per-skill accuracy) — absent on every pre-Milestone-5
record, so this is a no-op for existing data. `wrongAnswers` is queried separately and deliberately
left unfiltered.

Verified: retrying a wrong question to a correct result leaves the report's 총 풀이/정답 counts
byte-identical to their pre-retry values (confirmed via direct before/after DOM comparison); a
separate retry that is still wrong correctly appears in "최근 오답 리포트".

## Dev Task 5 — Regression + documentation (this commit)

Full 10-tab render sweep (overview/translation/vocab/phrases/grammar/examples/esl/sat/report/
wordbook) — zero errors. IndexedDB confirmed still exactly 6 object stores, no new store or index.
`registerProvider(` confirmed exactly 2 real invocations (`legacy-translation`+`gemini`) via grep —
no new provider. Gemini Summary/vocab-context functions present and unmodified, still dormant without
a dev key. Translation (`translateSentenceReliable`) and PDF import (`importDocument`) functions
present and unmodified. Mobile (375px): no horizontal overflow, SAT tab screenshot-confirmed clean —
correct/wrong choice coloring, evidence-sentence divider, and both grade/retry buttons all render
correctly sized and non-overlapping. Vocabulary Card confirmed untouched — `git diff main...HEAD --
index.html` contains zero references to `vocabCardTemplate`/`word-card-left`/`word-card-right`/
`word-card-headinfo`. Gold Master checksum reconfirmed unchanged
(`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`) throughout every task. Full diff
scope (`main...feature/practice-loop-closure`) touches only: `sat()`, `gradeQuiz()`,
`captureSatAttempt()`, `makeQuestions()`, `analyze()`, `renderReportAsync()`, the `wrongCards`
template inside `reportView()`, plus the new functions named above and one new CSS rule
(`.question.sat-retry-active`) — nothing outside this list.

## Files changed

`index.html` only, across 4 code commits (Dev Tasks 1-4) plus this documentation commit (Dev Task 5).
`LATEST_GOLD_MASTER_NEXT.html` untouched throughout (checksum reconfirmed at every task). No
IndexedDB schema change, no new provider, no other tab's core logic touched, Vocabulary Card
untouched (frozen per CEO decision).

## Commits

1. `d381dab` — Docs: Milestone 5 PM-Spec + Architecture (cherry-picked, CEO-approved)
2. `ca13bdb` — Dev Task 1: Retry Missed Questions
3. `2238008` — Dev Task 2: Evidence Sentence
4. `cca703a` — Dev Task 3: Weak-Area Reinforcement
5. `ac330f2` — Dev Task 4: Report Persistence Filter
6. (this commit) — Dev Task 5: Regression + implementation log

## Required test results (20/20)

| # | Test | Result |
|---|---|---|
| 1 | All-correct attempt: no retry action needed / no broken state | PASS |
| 2 | Mixed correct/incorrect: retry includes only missed questions | PASS |
| 3 | Retry cannot alter original-attempt answers/history | PASS (10/10 original records byte-identical post-retry) |
| 4 | Correct questions cannot accidentally be regraded in retry | PASS (structural: `gradeRetry()` skips `_firstCorrect===true` unconditionally) |
| 5 | Retry result is clearly distinguished from original result | PASS (two-number toast) |
| 6 | Retry persistence contains attemptKind/retryOfAttemptId | PASS |
| 7 | Retry attempts do NOT affect aggregate accuracy/trend | PASS (before/after report comparison identical) |
| 8 | Retry wrong answers still appear in wrong-answer review | PASS |
| 9 | evidenceSentence renders in quiz explanation | PASS |
| 10 | evidenceSentence renders in wrong-answer report | PASS |
| 11 | evidenceSentence XSS payload is escaped | PASS (both render sites, zero live injection) |
| 12 | Weak-area bias respects ~35% cap | PASS (unit-tested at n=6/n=10) |
| 13 | No-history/guest path remains normal and unbiased | PASS (no error, empty weakTypes no-op) |
| 14 | Weak type unsupported by current passage fails open gracefully | PASS (fabricated type, no error) |
| 15 | n=1 edge case works without error | PASS |
| 16 | Existing 11 question generators remain functional | PASS (all 11 confirmed present/callable) |
| 17 | Existing scoring/explanations/randomization preserved | PASS (`pos=i%4` line untouched) |
| 18 | Mobile SAT Quiz remains usable | PASS (screenshot-confirmed, no overflow) |
| 19 | No IndexedDB schema change | PASS (6 stores, confirmed) |
| 20 | No new AI/provider/API dependency | PASS (2 provider invocations, unchanged) |

## Remaining limitations (flagged, not fixed — out of this milestone's approved scope)

- Retry is a single round per quiz session by design (matches the CEO's singular "다시 풀기" framing);
  nothing technically prevents calling it again, but the UI only offers it once per grade.
- Weak-area reinforcement only ever activates for a *new* passage analysis — there is no way to
  regenerate fresh questions for a passage no longer in memory, per `01-PM-SPEC.md` §5's explicit
  "out of scope" list.
- `errorAdvice()`'s guidance text remains generic per-error-type (7-entry lookup), not
  question-specific — unchanged from before this milestone, not expanded.

## Recommended QA scope

Independently re-verify, per `.ai-company/QA.md`, against `01-PM-SPEC.md` §6's 11 acceptance
criteria and this log's 20-test table above. Recommend QA specifically re-attempt: a guest-mode full
pass across all four pieces; a multi-passage weak-area scenario (build real history across 2+
different passages, not seeded records, to confirm the natural end-to-end path); and an independent
XSS re-check on the evidence-sentence render sites with a different payload shape than used here.

---

## Handoff — Milestone 5 Development (Dev Tasks 1-5 complete)

- **Milestone:** Milestone 5 — Practice Loop Closure: Retry & Reinforcement.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/DEVELOPER.md`, `.ai-company/CODING_STANDARDS.md`, `.ai-company/GIT_RULES.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-05/01-PM-SPEC.md` (confirmed
  APPROVED), `02-ARCHITECTURE.md` (confirmed APPROVED).
- **Scope completed:** All 5 Developer Tasks, executed autonomously per the CEO's explicit
  instruction to continue through all approved tasks without stopping for routine acknowledgment. No
  work outside the approved Architecture was implemented.
- **Files changed:** `index.html` (4 code commits + this documentation commit), this file (new).
- **Commits created:** listed above — 6 total on `feature/practice-loop-closure`, none on `main`. No
  push, no merge.
- **Tests performed:** all 20 CEO-required tests, detailed per-task above; full table above.
- **Unresolved risks:** none newly introduced beyond `02-ARCHITECTURE.md` §9's 6 named risks, all
  mitigated and tested per this log.
- **Next agent:** QA Engineer, per the "Recommended QA scope" above.
- **Explicit stop point:** per the CEO's "STOP before QA" instruction, this session stops here.
  Awaiting CEO/QA review of these results. No push, no merge, no release.
