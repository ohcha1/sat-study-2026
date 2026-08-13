# Milestone 4 — Principal Review Report: Dictionary / Vocabulary Experience Upgrade

Independent Principal Review. Branch `feature/vocab-experience-upgrade`, reviewed at HEAD
`4e7ecb07a4a5eae9dda7db6b7d37a9c263182fdb` (confirmed unchanged before and after review). No
application code modified during this review — confirmed via `git status` before and after.

**Method:** Read `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
`04-QA-REPORT.md` in full, then independently spot-checked the actual code rather than relying on
those documents' claims, per `.ai-company/REVIEWER.md`. This surfaced one genuine defect the QA
pass did not specifically test for (§4/§9) — confirmed by a single, narrow, targeted reproduction,
not a full QA re-run.

## 0. Hot-fix addendum (post-review, same day)

**Status of §4/§10.1 finding: MEDIUM — RESOLVED BY HOT-FIX.** CEO decision: fix, not risk-accept.
See §13 below for full detail (root cause, fix, and independent re-verification). The verdict in
§1 and the rest of this report are left exactly as originally written, per this repo's append-not-
hide documentation convention — §13 is the authoritative status for the concurrency finding; treat
any statement elsewhere in this file about it being open/outstanding as superseded by §13.

## 1. Verdict

**APPROVE WITH CONDITIONS** — see §13 for hot-fix resolution superseding the §10.1 condition.

Architecture compliance, student-visible value, and every previously-tested area hold up under
independent spot-checking. This review found one new MEDIUM-severity defect not previously
identified — a race condition in `updateVocabProgress()`'s callers — that should be fixed before
this milestone is considered fully closed, but does not, on its own, warrant rejecting the
substantial, correctly-implemented work here. See §4/§8 for the condition.

## 2. Architecture compliance

Independently confirmed against `02-ARCHITECTURE.md`:
- **Branding** (§9.4 of the PM-Spec): `<title>`, `<h1>`, and subtitle read exactly
  `SAT English Learning Studio 2026 — AI` / `SAT Reading · Vocabulary · Grammar · AI Study Tools` —
  confirmed by direct `grep`, not just trusting the implementation log.
- **Important SAT Words non-mutation** (§3.2): `vocab(a)`'s `list` is built via `a.vocab.filter(...)`,
  which returns a new array — `a.vocab` itself is never reassigned or mutated. Confirmed by reading
  the function directly.
- **`updateVocabProgress()` read-merge-write** (§3.3): confirmed the implementation genuinely reads
  the existing record, spreads it first, then overlays the patch — technically sound *in isolation*.
  However, independent testing found it is **not sound under concurrent calls** — see §4/§9.
- **Word Book standalone reconstruction** (§3.5 reuse): `vocabCardTemplate()`/`buildVocabCardData()`
  confirmed callable with no passage in memory; `contextSentence` correctly `null` in that case,
  and the AI button correctly gated on its presence.
- **Gemini `vocab-context`** (§3.6): confirmed exactly one new `taskType` added to the existing
  adapter (`canHandle` now `"summary"||"vocab-context"`), `registerProvider(` called exactly twice
  in the whole file (no new provider), and `loadVocabContext(word, sentence, localDefinition)`'s
  call site confirmed to pass only those three values — no student identity, no passage title.
- **No IndexedDB schema change**: `git diff origin/main..feature/vocab-experience-upgrade` for
  `createObjectStore`/`createIndex` returns nothing.

No deviation from the approved architecture found, aside from the newly-discovered concurrency gap.

## 3. Student-visible value assessment

Confirmed directly in the card template's field order: tags → word → pronunciation → part of
speech → context meaning → Korean meaning → English meaning → save/review actions, exactly matching
the CEO's required above-the-fold priority order. Synonyms/antonyms/example/dictionary links
correctly deferred into a native `<details>` disclosure. Curated vs. general tags use plain Korean
labels ("사전 수록 단어" / "일반 설명") with no technical/provider terminology, confirmed by direct
inspection of the template strings. QA's 375px screenshot evidence and end-to-end walkthrough
(including a genuine page-reload boundary) are independently plausible given the code read here and
are accepted as sufficient runtime evidence — not re-run in this review per the task's "do not rerun
full QA" instruction.

## 4. Word Book / persistence review

The additive-fields, no-schema-change design is sound and correctly implemented for the
**single-writer** case (confirmed: one save, one review-state change, one reload — all correct,
consistent with QA's independent findings).

**New finding, not in `04-QA-REPORT.md`:** `toggleWordBookSave()` and `cycleReviewState()` both call
`updateVocabProgress()` with no in-flight guard of any kind (unlike `loadVocabContext()`, which has
`vocabContextInFlight` specifically for this reason). `updateVocabProgress()`'s `idbGet` → merge →
`idbPut` sequence is not atomic. Reproduced directly: firing `toggleWordBookSave('racecheck')` and
`cycleReviewState('racecheck')` concurrently on the same word left the final record with
`reviewState:"reviewing"` set but **`savedToWordBook` entirely absent** — the save was silently
lost. Both actions' success `toast()` calls still fire, so the student sees two confirmations while
only one change actually persisted.

**Severity: MEDIUM**, not HIGH/BLOCKER — per `.ai-company/TESTING_STANDARDS.md`'s definitions, this
requires two *different* actions on the *same* card within a sub-request-latency window (realistic
only via a very fast double-tap across two distinct buttons, not a single rapid click on one
button — the Risk-C/SAVE-1-style single-action guards elsewhere in this codebase don't cover this
different case). It does not affect the primary, thoroughly-tested single-action flows. It is,
however, a real silent-data-loss-with-false-success-feedback pattern, not merely cosmetic, so it is
not LOW.

## 5. Review-state assessment

Default/cycle/persistence/filter-exclusivity all confirmed sound for the single-writer case (same
`updateVocabProgress()` code path already assessed in §4 — the same concurrency caveat applies
here, since a save-and-review race is symmetric regardless of which side "wins").

## 6. Gemini vocab-context assessment

Confirmed genuinely optional and enhancement-only: `vocabContextAvailable` gates the button on both
a configured key *and* a present context sentence, computed at render time (fails closed before any
click, not just after a failed request). Confirmed core Dictionary Card renders and functions fully
with the check short-circuited to `false` (no key). No new provider; `canHandle` matrix confirmed
correctly scoped (handles `summary`/`vocab-context`, not `translation`; `legacy-translation` does
not handle `vocab-context`) via direct code read, consistent with QA's own cross-check.

## 7. Regression / preservation assessment

Independently confirmed via diff-scope inspection: `git diff origin/main..feature/vocab-experience-upgrade
-- index.html` produces 18 hunks, clustered exactly in the branding strings, the vocab-building loop,
the new card/Word Book/review functions, the `gemini` adapter, login/logout, and
`saveSessionForUser()` — none near grammar, SAT-question generation, translation core logic,
PDF/OCR/HEIC import, or the IndexedDB schema block. This is consistent with, and independently
corroborates, `04-QA-REPORT.md`'s §8 regression findings (Risk A/B/C, Gemini Summary UI, local
summary, PDF/photo import, 6-store schema, 7 media queries, no stuck translations, no new XSS on the
two new escape-requiring surfaces). Not re-run live in this review, per instruction; accepted as
correctly evidenced by QA and consistent with the code-level diff scope found here.

## 8. LOW finding disposition

`04-QA-REPORT.md` §9.1 (unsaving a word doesn't reset `reviewState`, so re-saving later shows it
already marked) — **acceptable to defer.** Confirmed this is a genuine consequence of the shared-
record design, not an implementation bug (the record intentionally isn't deleted on unsave, only
`savedToWordBook`/`savedAt` are cleared). No crash, no data loss, no security exposure, and it's a
product-judgment question (should re-saving reset progress?) rather than a correctness defect. Not
fixed in this review, per instruction.

## 9. BLOCKED_HUMAN_INPUT

Real Gemini API round-trip for `vocab-context` — requires a CEO-provided key in a controllable
browser session. Continuing to not block review; every other check for this feature is complete and
independently corroborated in §6 above.

## 10. Release concerns

1. **New MEDIUM finding (§4)** should be fixed — a small, well-scoped change (e.g., a per-word
   in-flight flag shared across `toggleWordBookSave`/`cycleReviewState`, mirroring the existing
   `vocabContextInFlight` pattern) — before this milestone is fully closed. Recommend a small,
   explicitly-scoped hot-fix Developer Task, matching the precedent already established for
   Milestone 3's Risk A/B/C hot-fixes, rather than blocking this review outright.
2. Real Gemini live-key verification remains outstanding (§9) — should be resolved before final
   release sign-off, consistent with the same open item carried from Milestone 3.
3. No push/merge/schema/provider/scope-expansion concerns — all confirmed clean.

## 11. Files modified by this review

**None.** `git status` confirmed identical before and after
(`4e7ecb07a4a5eae9dda7db6b7d37a9c263182fdb`). This report file is the only addition.

## 12. Is the milestone suitable for release packaging?

Conditionally: yes, once the §4/§10.1 MEDIUM finding is fixed (or explicitly risk-accepted by the
CEO in writing, per this repo's established precedent for carrying a known issue into release with
explicit sign-off). Everything else independently verified sound.

**Superseded by §13:** the finding has since been fixed and independently re-verified. Milestone 4
is suitable for release packaging with no outstanding concurrency condition.

## 13. Hot-fix: concurrency race (§4/§9/§10.1) — root cause, fix, verification

CEO decision: fix the MEDIUM finding, do not risk-accept it. Executed as a narrowly scoped hot-fix
Developer Task on the same branch, then independently re-verified in the Principal Reviewer role,
per `.ai-company/WORKFLOW.md`'s hot-fix precedent (matching Milestone 3's Risk A/B/C pattern).

### 13.1 Root cause

`updateVocabProgress(word, patch)` performed an unguarded `idbGet` → merge → `idbPut` sequence with
no per-word in-flight guard. `toggleWordBookSave()` and `cycleReviewState()` each also read their
"current state" (`nowSaved`, `current`) from the outer `vocabProgressCache` *before* calling
`updateVocabProgress()`. Two mutations for the same word issued close together — including two
calls to the *same* function back-to-back — could both read the same `existing` record before
either wrote it back, so the second write silently discarded the first's field (or, for repeated
calls to the same function, both computed the same "next" value instead of advancing twice).

### 13.2 Fix implemented

A per-word promise-chain mutex, `vocabProgressQueue`/`withVocabProgressLock(word, fn)`
(`index.html`, immediately above `updateVocabProgress()`). Every call to `updateVocabProgress()` for
a given word is chained onto that word's existing queue entry, so its read-modify-write body never
overlaps with another mutation of the *same* word; different words get independent chain entries and
never block each other (verified in Test 5, §13.4). A rejected link is caught before being stored
back into the queue (`vocabProgressQueue[word]=run.catch(()=>{})`) so one failed mutation cannot
permanently wedge later mutations for that word — while the original caller still observes the real
rejection via the returned, uncaught `run` promise.

`updateVocabProgress(word, patch)` now also accepts `patch` as a function `(existing) => patchObject`
in addition to a plain object (existing callers passing plain objects, e.g. `saveSessionForUser()`'s
passive tracker, are unaffected). `toggleWordBookSave()` and `cycleReviewState()` were changed to
pass a function so their "current state" decision is computed from the record as read *inside* the
lock, not from the outer cache — this is what makes rapid repeated same-function calls (Test 4)
correct, not just cross-function races (Tests 1-3).

No global lock, no sleep/delay, no schema change, no new dependency, no new provider. The entire
diff is 41 insertions / 11 deletions confined to the three functions named in the finding plus the
new lock helper (`git diff --stat` on `index.html`); no lines outside `updateVocabProgress()`,
`toggleWordBookSave()`, `cycleReviewState()`, and the new helper were touched.

### 13.3 Files changed

`index.html` only (the lock helper plus the three functions listed above). This file
(`05-REVIEW-REPORT.md`, §0/§1/§12/§13 addenda) is the only other change, per the append-not-hide
convention — no historical section text was deleted or rewritten.

### 13.4 Targeted tests (live in-browser, via `mcp__Claude_Browser__javascript_tool` against the
real `index.html` served locally — same methodology as prior milestones' testing, Node/jsdom still
unavailable in this environment)

1. Save+Review near-simultaneous, same word → **PASS**. Both `savedToWordBook:true` and
   `reviewState:"reviewing"` present together on one record.
2. Review+Save reverse order → **PASS**. Same result, order-independent.
3. Unsave+Review near-simultaneous → **PASS**. `savedToWordBook:false` (with `savedAt:null`) AND the
   new `reviewState` both survive together.
4. Rapid multiple review transitions (3 concurrent `cycleReviewState()` calls on one word, starting
   at `"new"`) → **PASS**. Final state `"new"` — the mathematically correct result of three full
   `new→reviewing→learned→new` steps, proving no transition was lost or duplicated.
5. Two different words mutated concurrently → **PASS**. Both fully resolved (`savedToWordBook:true`,
   `reviewState:"reviewing"`) with near-identical timestamps and independent `vocabProgressQueue`
   entries, confirming no unnecessary cross-word blocking.
6. Forced failure (monkey-patched `idbPut` to reject once for one word) → **PASS**. The failing call
   genuinely rejected (caller's `toast()`/success line never ran — no false-success reporting, no
   unhandled promise rejection observed in the console), and a subsequent mutation for the *same*
   word immediately after succeeded normally — no permanent lock.
7. True full-page reload, then read directly from IndexedDB with a fresh JS context → **PASS**. All
   7 prior test records persisted exactly as written.
8. Old-format record (manually seeded with only a legacy `lastSeen` field, simulating data written
   before Word Book/Review existed) → **PASS**. `lastSeen` preserved, new fields added correctly via
   the same read-merge-write semantics as before the hot-fix.
9. Passive session-save tracker (`saveSessionForUser()`'s plain-object `updateVocabProgress()` call
   path) against a word with existing `savedToWordBook`/`reviewState` → **PASS**. Both fields
   preserved after the plain-object patch call.
10. IndexedDB schema → **PASS**. `openDB()` reports the same 6 stores at version 1; `git diff` shows
    zero `createObjectStore`/`createIndex` lines touched.

### 13.5 Focused regression (not a full QA re-run, per instruction)

- Real end-to-end UI flow (not just direct function calls): analyzed a live passage, clicked through
  the actual vocab tab render — the Dictionary Card for a saved/reviewed word correctly rendered
  "저장됨 ✓ (제거)" / "복습 중" while sibling cards remained at their own independent default state.
- Word Book tab standalone reconstruction: correctly listed all 7 saved words (mix of in-passage and
  standalone-only words from the targeted tests) with correct filter counts (전체 7 / 새 단어 0 /
  복습 중 7 / 학습 완료 0), matching the exact state written during targeted testing.
- Guest (no `auth.userId`) behavior: unchanged — both functions return immediately after the login
  toast, no record written, no exception thrown.
- Mobile viewport (375×812): re-verified visually, layout and toast unaffected.
- XSS escaping: a word value containing `<img src=x onerror=alert(1)>` rendered fully escaped in the
  Word Book view — this code path is untouched by the hot-fix diff, confirmed still safe.
- Confirmed unchanged by direct inspection (none of these files/regions appear in the hot-fix diff):
  Gemini adapter (`registerProvider` called exactly twice as invocations, `canHandle` matrix still
  `"translation"` xor `"summary"||"vocab-context"`), `legacy-translation` adapter, Risk A
  (`REQUEST_TIMEOUT_MS`/`fetchWithTimeout`), Risk B (savedPassages ID uniqueness loop — code region
  untouched), Risk C (`analyzeInFlight` guard still present, unmodified), `escapeHtml()`, and the
  6-store IndexedDB schema.

### 13.6 Principal re-verification (this role, independently, after the hot-fix commit)

Inspected the actual committed diff (not just the Developer's test claims): confirmed the lock is
keyed per-word (`vocabProgressQueue[word]`), not a single global flag or promise; confirmed no
stale-read path remains — both `toggleWordBookSave()` and `cycleReviewState()` now compute their
decision from the record read *inside* `withVocabProgressLock`'s callback, not from
`vocabProgressCache` read before the call; confirmed the queue recovers after a rejected link
(`.catch(()=>{})` on the stored chain reference only, not on the returned `run`, so callers still see
real failures); confirmed zero schema-related lines changed; confirmed zero lines outside the four
named functions/helper changed (no unrelated application changes); confirmed no secret, key, or
credential was added anywhere in the diff. Independently re-ran the original two-line reproduction
from §4 (`toggleWordBookSave('racecheck')` + `cycleReviewState('racecheck')` fired concurrently) —
before the fix this left `savedToWordBook` entirely absent; after the fix, both fields are present
together (Test 1/2 above are exactly this reproduction, run twice in both orders).

**Verdict: APPROVE.** The hot-fix is minimal, correctly scoped, per-word (not global), fails safe,
and independently confirmed to eliminate the originally-reproduced race without introducing any new
risk, schema change, or scope expansion.

---

## Handoff — Milestone 4 Principal Review

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/REVIEWER.md`, `.ai-company/CODING_STANDARDS.md`, `.ai-company/DEFINITION_OF_DONE.md`,
  `docs/milestones/milestone-04/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
  `04-QA-REPORT.md`, plus direct reads of `index.html` (branding strings, `vocab()`,
  `vocabCardTemplate()`, `updateVocabProgress()`, `toggleWordBookSave()`, `cycleReviewState()`, the
  `gemini` adapter, `loadVocabContext()`) and the diff against `origin/main`.
- **Scope completed:** Reviewed all 15 items requested. Independently spot-checked rather than
  trusting the QA report, which surfaced one new MEDIUM finding (concurrent-write race in
  `updateVocabProgress()` callers) not previously identified, reproduced with a single targeted
  test.
- **Files changed:** `docs/milestones/milestone-04/05-REVIEW-REPORT.md` (new, this file). No
  application code touched — confirmed via `git status` before and after.
- **Commits created:** None this session.
- **Tests performed:** Code-level inspection throughout; one narrow, targeted concurrency
  reproduction (§4) — not a full QA re-run, per instruction.
- **Unresolved risks:** The new MEDIUM finding (§4/§10.1) and the pre-existing
  `BLOCKED_HUMAN_INPUT` item (§9).
- **Next agent:** CEO — to decide between (a) authorizing a small hot-fix Developer Task for the
  MEDIUM finding before Release, or (b) explicitly risk-accepting it in writing and proceeding to
  Release Manager packaging with it logged as a known, deferred issue.
- **Explicit stop point:** Per the task's "STOP at Principal Review gate" instruction. No push, no
  merge, no code changes.

---

## Handoff addendum — Scoped Concurrency Hot-Fix (same day, post-review)

- **What changed since the handoff above:** The MEDIUM finding is no longer unresolved. CEO decision
  was (a) — authorize a hot-fix, not risk-accept. A scoped Developer hot-fix was implemented and
  independently re-verified in this same Principal Reviewer role. See §13 for full detail.
- **Files changed:** `index.html` (the fix itself) and this file (§0/§1/§12/§13 addenda). No other
  files.
- **Commits created:** One, on `feature/vocab-experience-upgrade` — see the commit referenced in the
  hot-fix's final report for the exact hash. Not pushed, not merged.
- **Unresolved risks now:** Only the pre-existing `BLOCKED_HUMAN_INPUT` item (§9) — real Gemini
  live-key verification. The concurrency finding is resolved (§13.6: APPROVE).
- **Next agent:** CEO — to decide on release packaging; the concurrency condition that previously
  gated §12 no longer applies. The LOW finding (`04-QA-REPORT.md` §9.1, unsave doesn't reset
  `reviewState`) remains intentionally deferred, unchanged, per explicit instruction.
- **Explicit stop point:** Per the hot-fix task's "STOP only after Principal re-review. Do not push
  or merge." No push, no merge performed.
