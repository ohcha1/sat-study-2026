# Milestone 5 — PM Assessment: Closing the Student Study Loop

**Status: APPROVED by the CEO — 2026-08-13, as "Practice Loop Closure: Retry & Reinforcement."** See
§8 for the CEO's scope decisions. Architecture Design (`02-ARCHITECTURE.md`) may proceed. Produced
under "Autonomous Continuation Mode: ON" for analysis and documentation, per the CEO's explicit "STOP
before application-code implementation" instruction. No application code was touched while producing
either this document or the Architecture Design; `index.html` and `LATEST_GOLD_MASTER_NEXT.html`
remain unchanged from this document's baseline verification.

> **Numbering note**, matching the precedent already established for "Milestone 3" and "Milestone 4"
> in `DEVELOPMENT_PLAN.md`: this claims the next sequential `docs/milestones/milestone-05/` folder.
> `DEVELOPMENT_PLAN.md` already has a planned "Milestone 5 — AI Tutor" entry (replacing templated
> grammar/question generation with real AI-generated content). **This is not that milestone.** This
> document proposes a smaller, AI-independent milestone that closes a specific, currently-missing
> link in the student study loop, and recommends it be sequenced *before* the larger AI Tutor work —
> see §5. If approved, the CEO may wish to renumber one of the two to avoid the same "Milestone 5"
> label collision `DEVELOPMENT_PLAN.md` already carries for "Milestone 3" and "Milestone 4"; not
> resolved here, per the same precedent of recording rather than silently renumbering.

## 0. Baseline verification (read-only, this session)

- Branch: `feature/dictionary-visual-polish`, HEAD `4725fdf` — confirmed via `git rev-parse`.
- Working tree: clean except pre-existing untracked `.DS_Store`/`.claude/`.
- `LATEST_GOLD_MASTER_NEXT.html` SHA-256: `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`
  — unchanged, matches every prior recording this session.
- No application code modified while producing this document.

## 1. Vocabulary UI status (per explicit CEO instruction, recorded here for cross-reference)

**Accepted for now and FROZEN.** Not considered visually perfect by the CEO, but no further
cosmetic iteration is authorized this milestone — only genuine functional defects should prompt
further changes. Full detail and commit history recorded in `DEVELOPMENT_PLAN.md`'s Milestone 4
(Vocabulary Experience Upgrade track) section, 2026-08-13 update.

## 2. The student study loop, as it exists today

The CEO's stated target loop:

```
Passage → Analysis → Vocabulary → SAT Questions → Answer → Explanation
  → Wrong-answer review → Repeat practice
```

Verified directly against `index.html` (function bodies read, not inferred):

| Step | Status | Evidence |
|---|---|---|
| Passage → Analysis | **Works** | `analyze()` drives the whole pipeline; Risk C guard prevents re-entrant races. |
| Analysis → Vocabulary | **Works, recently polished** | Milestone 4's Dictionary Card, Word Book, Review states (now frozen per §1). |
| Vocabulary → SAT Questions | **Partially works** | `genVocabInContext()` is one of 11 question-type generators (`QG_GENERATORS`) and specifically draws on curated (non-fallback) vocab words with a hand-curated distractor set — a real, if narrow, connection. No other generator references vocabulary/Word Book data. |
| SAT Questions → Answer | **Works** | `sat(a)` renders up to 10 questions (student-selectable 1–10, default 6) from `makeQuestions()`; `gradeQuiz()` grades on submit. |
| Answer → Explanation | **Works, substantive** | Each question carries a real `why`/`whyKo` explanation string built from the actual chosen passage sentence/content (see §3), not a single generic sentence reused everywhere. Shown inline after grading. |
| Wrong-answer review | **Works, well-built** | `captureSatAttempt()` persists every attempt to `satAttempts` and, for misses, to `wrongAnswers` (IndexedDB). `reportView()`'s "학습 리포트" tab shows accuracy, a monthly trend chart, per-question-type and per-skill accuracy tables, and up to 8 recent wrong-answer cards with question text, selected vs. correct answer, an internally-classified error type (`classifyErrorType()`), the original explanation, and generic type-level advice (`errorAdvice()`). |
| **Repeat practice** | **Does not exist** | No code path lets a student re-attempt a missed question, retry only the questions they got wrong, or generate a fresh practice set targeting a weak question type/skill. Confirmed by exhaustive search (`다시 풀`, `재도전`, `연습 문제`, "retry", "practice again" — zero matches in `index.html` outside this analysis). |

**This is the one broken link.** Every other stage of the CEO's stated loop already exists and was
verified directly in code, several stages in real depth (the wrong-answer report in particular is
already a genuinely useful, non-trivial feature — accuracy trend chart, per-skill breakdown, and
per-mistake explanation). The loop has a genuine "review" half and no "repeat" half.

## 3. Detailed SAT Quiz assessment (per explicit CEO request)

**What already works:**
- **Question generation is real content extraction, not generic filler.** `makeQuestions()` runs 11
  independent generator functions (`genCentralIdea`, `genInference`, `genPurposeFunction`,
  `genTextStructure`, `genCauseEffect`, `genComparisonContrast`, `genTransition`,
  `genDetailEvidence`, `genVocabInContext`, `genCommandOfEvidence`, `genAuthorsPurpose`), each
  independently deciding whether the passage supports its question type (returns `null` if not) via
  sentence-signal detection (`qgSignals`) and clause-extraction regexes. Distractors are built from
  *other real sentences in the same passage*, not swapped-in boilerplate — read `genCentralIdea()` in
  full: its three wrong choices are constructed from other actual passage clauses with a scope/
  emphasis/relationship error deliberately introduced, not disconnected text. This is a materially
  more sophisticated system than "10 fixed templates" might suggest at a glance.
- **Answer-choice quality/safety:** the correct answer's position is randomized per question
  (`pos=i%4`-derived, not fixed); every choice is passed through `englishOnly()` (rejects
  Korean-contaminated fragments, falls back to a safe generic choice) and `escapeHtml()` before
  render — both defenses were specifically hardened in Milestones 1/2 after real defects were found
  there.
- **Explanations are per-question, not per-type.** `why`/`whyKo` are generated inside each generator
  function, referencing that specific question's actual correct content — e.g.
  `genCentralIdea()`'s explanation names the mechanism ("uses real supporting details but incorrectly
  elevate one of those details"), not just "the correct answer is correct."
- **Scoring:** `gradeQuiz()` grades client-side against `q.ans`, shows correct/wrong styling per
  choice, reveals all explanations, and reports a score toast — straightforward and correct.
- **Persistence:** every attempt (not just the score) is recorded per-question in `satAttempts`
  (`attemptId`, passage hash, question type/text, selected vs. correct answer, explanation, evidence
  sentence, skill tags, score) — a genuinely rich record, not just a running total. Missed questions
  additionally land in `wrongAnswers` with an internally-classified error type. Guest users
  (`!auth.userId`) explicitly do not persist attempts (existing, correct, documented behavior).
- **Mobile usability:** `.choice` has a dedicated `min-height:44px` touch-target rule inside the
  existing responsive breakpoint — already addressed, not a gap.
- **Connection to the analyzed passage:** strong. Every question is generated from — and every
  distractor drawn from — the specific passage just analyzed; nothing is passage-agnostic.
- **Connection to Vocabulary/Word Book:** narrow. Only `genVocabInContext()` touches vocabulary data,
  and only curated (non-fallback) words; it does not read Word Book/Review state at all, so a
  student's saved/struggling words never influence which vocabulary question (if any) they're asked.
- **AI vs. local:** 100% local/deterministic. Zero AI involvement in question generation, distractor
  construction, or explanations — consistent with `DEVELOPMENT_PLAN.md`'s existing Milestone 5 (AI
  Tutor) problem statement, which this document does not dispute.

**Gaps found, beyond the missing repeat-practice loop:**
1. `evidenceSentence` is captured on every question and stored in both `satAttempts` and
   `wrongAnswers`, but is **never rendered anywhere** — not in the live quiz, not in the wrong-answer
   report cards. A student sees *that* they were wrong and *why the correct answer is correct* in the
   abstract, but not *which exact sentence in their own passage* proves it. This is a small, already-
   available-data fix, not a new feature.
2. The wrong-answer report shows only the 8 most recent misses across all passages/time — there is no
   way to see all of them, or filter by passage/type beyond the separate aggregate tables.
3. `errorAdvice()` (the "다음에 주의할 점" line) is a fixed 7-entry lookup by error type — generic,
   not tied to the specific question. Reasonable as-is; flagged only because §5's proposal reuses it
   and should not be read as silently expanding this into a new AI-writing task.

## 4. Why "repeat practice" is the highest-value next gap to close

- It is the **only completely missing stage** in the CEO's explicitly stated loop — every other stage
  exists and was independently verified in this session, not assumed from documentation.
- The infrastructure it needs already exists end-to-end: `wrongAnswers`/`satAttempts` records, the 11
  question generators, `classifyErrorType()`/`skillsForType()`, and the report tab's aggregation
  logic. This is a **connecting** feature, not a new subsystem.
- It requires **no AI integration, no new IndexedDB store, no new provider, no account/backend
  work** — it is achievable entirely within the current local/deterministic architecture, unlike
  `DEVELOPMENT_PLAN.md`'s existing Milestone 5 (AI Tutor) or Milestones 6/7 (accounts/history, which
  the app already substantially has locally via `profiles`/`satAttempts` — a genuinely separate,
  larger, and lower-urgency gap than this one).
- It is the step most directly tied to actual learning outcomes: a report a student cannot act on
  (today's state) is materially less valuable than a report that lets them immediately retry what
  they got wrong.

## 5. Recommended milestone scope (smallest high-value increment)

**"Practice Loop Closure: Retry & Reinforcement."** Three additive pieces, in priority order; the
CEO may choose to approve all three or a subset:

1. **In-quiz retry ("오답만 다시 풀기").** After grading, missed questions can be reset (radio
   selection cleared, correct/wrong styling removed) and re-attempted within the same session, while
   already-correct questions stay locked/graded. Reuses `sat()`/`gradeQuiz()` almost entirely — the
   smallest-diff piece of this proposal, and the most immediately useful (a student rarely leaves the
   passage they're currently working on).
2. **Weak-area-biased question selection.** When a student later analyzes a *new* passage,
   `makeQuestions()`'s generator selection/ordering is biased to guarantee coverage of the question
   type(s) the student's `wrongAnswers` history shows as weakest (reusing the report tab's existing
   `typeStats` aggregation), rather than every passage always producing whatever the fixed
   `QG_PRIORITY` order happens to surface. No new content generation — purely a selection-order
   change within the existing 11 generators.
3. **Surface `evidenceSentence`** in both the live quiz's explanation panel and the report tab's
   wrong-answer cards (§3, gap 1) — a small, low-risk fix that makes existing explanations
   concretely verifiable against the student's own passage.

**Explicitly out of scope for this milestone** (flagged, not silently expanded into it):
- Generating *new* questions for a *past* passage no longer in memory (would require re-deriving
  sentence/signal data from stored `passageId`/hash alone, or storing the full passage text per
  attempt — an architecture decision the CEO should make explicitly, not one this document assumes).
- Any AI-generated question/explanation content — remains `DEVELOPMENT_PLAN.md`'s separate Milestone
  5 (AI Tutor) scope.
- Expanding `wordBookFilterBar`-style filtering into the wrong-answer report (nice-to-have, not
  required to close the loop).

## 6. Acceptance criteria (final, per CEO decision — see §8)

1. A student can retry only missed questions without re-analyzing the passage.
2. Retry does not overwrite or corrupt the original attempt — original `satAttempts`/`wrongAnswers`
   records are untouched; retry results are stored as new, separately-tagged records.
3. The retry result is clearly distinguished from the original result in the UI (not merely
   non-corrupting — actually visible as two separate outcomes).
4. Evidence sentence is visible for each relevant question, in student-friendly wording ("근거
   문장"), with no internal field name ever exposed, in both the live quiz explanation and the
   wrong-answer report card.
5. Weak-area reinforcement uses existing student history only (`satAttempts`, excluding prior retry
   attempts — see `02-ARCHITECTURE.md` §3) — no new tracking mechanism.
6. Quiz diversity remains intact: weak-area-biased questions are capped at approximately 30–40% of a
   given quiz (35% implemented, see §8.3); the remaining majority always follows the existing
   unbiased `QG_PRIORITY` selection.
7. Core quiz remains fully functional with no login/history available (guest mode) — retry works
   in-session without persistence; weak-area biasing and the report tab correctly no-op/show existing
   guest messaging.
8. No AI/API key required anywhere in this milestone's functionality.
9. No IndexedDB schema migration — no new object store, no new index; only additive fields on
   existing `satAttempts`/`wrongAnswers` records.
10. Mobile flow remains usable — no new CSS beyond reusing existing `.btn`/`.choice`/`.small` classes,
    already touch-target-verified.
11. Existing SAT Quiz behavior is backward compatible: a quiz with zero wrong answers, or a
    logged-out/guest session, behaves identically to today (no retry button appears, no report-stat
    change, no weak-area bias applied).

## 7. Explicit stop point

Per the CEO's instruction: **no application code has been modified.** This document is a PM-phase
assessment and recommendation only. Architecture Design should not begin until the CEO reviews and
either approves this scope, requests changes, or selects a different next milestone.

## 8. CEO decisions (recorded 2026-08-13)

**Approved in full:** all three pieces from §5 (retry, evidence sentence, weak-area weighting), plus
the report-tab persistence handling needed to keep retry attempts from distorting existing statistics
(§5 did not separately enumerate this as a fourth piece; the CEO's approval message's Architecture
Definitions §9 explicitly requires it — see `02-ARCHITECTURE.md` §3 for the resulting design).

Binding constraints added beyond §5/§6's draft framing:

1. **Retry:** must clearly distinguish original vs. retry results (not just avoid corrupting the
   original) — the toast/UI must show both, not silently replace one with the other.
2. **Evidence sentence:** must use student-friendly wording ("근거 문장"), never expose internal
   field names. Render in both the live quiz explanation and the wrong-answer report card — confirmed
   scope from §5, no change.
3. **Weak-area weighting:** must be a **conservative, capped bias — approximately 30–40% of a given
   quiz's questions — not full personalization.** The CEO explicitly warned against a quiz consisting
   only of weak categories; Architecture must define and justify the exact cap within that range (see
   `02-ARCHITECTURE.md` §4 — 35%, `Math.ceil(n*0.35)`, chosen as the range's midpoint for a simple,
   round implementation).
4. **No new AI** — no provider, no Gemini call for quiz generation or explanations, no paid API. This
   milestone works fully offline/local, confirming §5's framing (this was already the proposal's
   premise, now made a hard constraint rather than a design preference).
5. **No schema migration** — reuse `satAttempts`/`wrongAnswers` as-is; additive fields only if
   Architecture finds them unavoidable (it does — see `02-ARCHITECTURE.md` §3 for the two additive
   fields, `attemptKind`/`retryOfAttemptId`, and the explicit justification for why they don't require
   a schema/index change).
6. **Preserve existing systems** — explicit list in the CEO's approval (11 question generators, answer
   randomization, explanations, scoring, wrong-answer records, report charts/statistics, Vocabulary/
   Word Book, Gemini Summary, Gemini vocab-context, translation architecture, PDF/photo import, mobile
   responsiveness, XSS protections) — matches this document's own §5 "explicitly out of scope" list
   and adds no new exclusions, just makes them binding acceptance criteria (§6, revised).
7. **Vocabulary UI:** reaffirmed FROZEN (§1, unchanged).
8. Scope stays this size — "This milestone is intended to CLOSE THE STUDY LOOP, not redesign the
   entire SAT Quiz system." Any additional idea surfaced during Architecture/Development that grows
   beyond §5's three pieces must be logged and deferred, not silently implemented.

**Explicitly deferred (unchanged from §5, reconfirmed, not re-litigated):** AI-generated
questions/explanations (remains `DEVELOPMENT_PLAN.md`'s separate Milestone 5 — AI Tutor scope);
regenerating fresh questions for a passage no longer in memory; Word-Book-style filtering inside the
wrong-answer report.

---

## Handoff — Milestone 5 PM Assessment

- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/PM.md`, `DEVELOPMENT_PLAN.md` (full), plus direct reads of `index.html`
  (`makeQuestions`, `QG_GENERATORS`/`QG_PRIORITY`, `genCentralIdea`, `sat()`, `gradeQuiz()`,
  `captureSatAttempt()`, `classifyErrorType()`, `reportView()`/`renderReportAsync()`,
  `wordBookView()`/`renderWordBookAsync()`, IndexedDB schema/store definitions).
- **Scope completed:** Verified current branch/HEAD/Gold Master checksum; recorded the Vocabulary UI
  freeze decision in `DEVELOPMENT_PLAN.md`; walked the full CEO-specified student study loop against
  actual code (not documentation) and identified "repeat practice" as the sole missing stage;
  performed the requested detailed SAT Quiz assessment (question quality, answer choices,
  explanations, scoring, wrong-answer handling, persistence, mobile usability, passage/vocab
  connection, AI-vs-local); proposed a scoped, AI-independent next milestone with draft acceptance
  criteria. No application code touched.
- **Files changed:** `DEVELOPMENT_PLAN.md` (Vocabulary UI freeze note appended), this file (new).
- **Commits created:** None this session — pending CEO review, per this repo's established pattern
  for PM-phase documents.
- **Unresolved risks:** None new. The "Explicitly out of scope" items in §5 are open questions for
  the CEO/Architecture phase, not risks.
- **Next agent:** CEO, to approve/revise this scope (or select a different next milestone) before
  Architecture Design begins.
- **Explicit stop point:** Per the CEO's "STOP before application-code implementation" instruction.
