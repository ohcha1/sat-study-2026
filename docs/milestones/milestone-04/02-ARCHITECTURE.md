# Milestone 4 — Architecture Design: Dictionary / Vocabulary Experience Upgrade

**Status: DRAFT — awaiting CEO approval.**

Scoped to `docs/milestones/milestone-04/01-PM-SPEC.md` (APPROVED 2026-08-12) §4/§5/§6, within the
CEO's constraints recorded in §9. No new provider, no IndexedDB schema change unless proven
necessary (see §3.3 — it isn't), no dictionary-data-expansion scope, no change to translation/
grammar/quiz core logic.

## 1. Approach summary

Redesign the Vocabulary tab's per-word card and add three small, additive capabilities on top of
it — Word Book (save), Review state (learned/reviewing), and an optional AI context-explanation —
by extending existing data structures with new fields rather than building new systems. The
guiding constraint from the CEO (§9.2) is reuse: the ~1,141-word curated lexicon, the IPA/TTS
system, and the `vocabularyProgress` IndexedDB store already do most of the work; this milestone is
primarily a **UI and data-surfacing** project, not a new-systems project.

Three structural decisions anchor everything below:

1. **"No schema change" is achieved by extending record *contents*, not object-store
   *definitions*.** IndexedDB stores in this app are schemaless beyond `keyPath`/index
   declarations (`openDB()`, index.html:4284-4321) — `idbPut()` just writes whatever object shape
   it's given. `vocabularyProgress` already has the exactly-right key (`userId+"_"+word`, one row
   per user per word) for a word-book/review model. Adding fields like `savedToWordBook`,
   `savedAt`, `reviewState` to that record is **not** a schema change (no new `createObjectStore`/
   `createIndex`), it's a data-shape extension — which is what lets this milestone honor the CEO's
   constraint literally, not just in spirit.
2. **Card content must be reconstructable from a word string alone**, not just from the current
   in-memory analysis. Word Book/Review Queue need to render full cards (definition, IPA, examples,
   etc.) for words saved from *past* sessions, when `state.analysis` no longer contains them. This
   requires extracting today's inline card-data lookup (currently embedded in `analyze()`,
   index.html:2321-2347) into a standalone, reusable function.
3. **AI is additive, not load-bearing.** Every acceptance criterion in the PM-Spec must hold with
   zero Gemini involvement. The AI enhancement layer is a new `taskType` on the *existing* `gemini`
   adapter (index.html:1962-1998), not a new provider, not a new router concept.

## 2. Affected files/modules

Single file, as with every prior milestone on this line: `index.html`. No new files. Sections
touched (by current line number, all subject to shifting as earlier tasks land):

- `analyze()` (2299-2394) — vocab-building loop refactored to call the new extracted card-builder
  and to tag each entry with a curated/fallback flag.
- New: `buildVocabCardData(word, passageText)` — extracted/generalized from `analyze()`'s inline
  lookup.
- `vocab(a)` (3383-3389) — replaced with the redesigned card renderer + Important/Show-all toggle.
- `openDB()` (4284-4321) — **unchanged** (no new store/index); referenced only to confirm the
  reuse claim.
- `saveSessionForUser()` (4664) — the existing `vocabularyProgress` write extended with new
  optional fields on explicit user action (Word Book save / review-state change), in addition to
  its existing passive "seen" write.
- New: `renderWordBook()`/`renderReviewQueue()` (or a combined view — see §3.3) plus their `render()`
  dispatch wiring, following the same tab-registration pattern already used for `translation`,
  `vocab`, etc.
- `providerRegistry`'s `gemini` adapter (1962-1998) — `canHandle()` extended to accept a new
  `taskType` (`"vocab-context"`); no change to its `"summary"` handling or to `legacy-translation`.
- CSS: new rules for the collapsible card sections and Important/Show-all/Word-Book/Review UI,
  added alongside existing `.word-card`/`.meaning-block`/`.syn-ant` rules (index.html:56-90 region),
  not replacing them — the existing grid-based card layout (`grid-template-columns:56px
  minmax(140px,260px) 1fr`, collapsing to `1fr` under a mobile media query) is preserved as the
  outer structure; new content is added inside it.
- `DEVELOPMENT_PLAN.md` — correct the stale "~40-word" figure (PM-Spec §6.9), a documentation-only
  change, not part of the Developer phase's code tasks.

## 3. Data flow / interfaces

### 3.1 Dictionary Card component structure

One card per word, mobile-first. Two zones:

```
┌───────────────────────────────────────────┐
│ ABOVE THE FOLD (always visible)            │
│  word · level badge                        │
│  IPA + 🔊 speak button                     │
│  part of speech                            │
│  context meaning (passage sentence w/ word)│
│  English definition                        │
│  Korean meaning                            │
│  [Save to Word Book]  [학습 상태: 새 단어 ▾]│
├───────────────────────────────────────────┤
│ BELOW THE FOLD (native <details> toggle)   │
│  synonyms · antonyms                       │
│  word family (if present)                  │
│  root/etymology (if present)                │
│  SAT-level example (EN/KO)                 │
│  dictionary links                          │
│  [AI: 왜 이 뜻일까요? (문맥 설명)] (if key)  │
└───────────────────────────────────────────┘
```

- Uses native `<details>`/`<summary>` for the collapsible zone — zero new JS state, works on every
  browser without a framework, degrades gracefully, matches "smallest safe diff." Desktop can style
  it open-by-default via CSS if desired (a `details[open]` default), mobile defaults closed.
  `<summary>`text: "더 보기 (동의어·어원·예문)" ("more: synonyms/etymology/examples").
- **Context meaning** is *not* AI-generated by default: it's the literal sentence from
  `state.analysis.sents` (or the passed `passageText`) containing the word, shown as-is — cheap,
  local, always available. The AI enhancement (§3.6) is an *additional*, optional explanation of
  *why* that sense fits, layered on top, not a replacement.
- **Word family / root-etymology**: new fields in the card template. Since no current lexicon entry
  carries this data, they render only when present and are omitted (not shown as empty/broken)
  otherwise — which, given today's data, means they will be absent for effectively all words in
  this milestone. This is a conscious, CEO-confirmed tradeoff (PM-Spec §9.3): the field *exists* in
  the component so a future milestone can populate it (via curated-data expansion or AI
  enhancement) without another UI change, but this milestone does not backfill the data itself.
- **Save/state controls**: a "Save to Word Book" button (becomes "저장됨 ✓ / 단어장에서 제거" once
  saved) and a small state selector (새 단어 / 복습 중 / 학습 완료 — new/reviewing/learned). Both are
  only interactive when `auth.userId` is set (§3.3); when logged out, they render disabled with the
  existing guest-mode message pattern rather than being hidden entirely, so a guest understands the
  feature exists and how to unlock it (matches `reportView()`'s precedent exactly, index.html
  ~4600).

### 3.2 Important-word selection logic

`analyze()`'s existing vocab-building loop (2321-2347) already distinguishes, structurally, between
words found via a real lexicon lookup (`dictionary[w]`/`supplementalVocab[w]`/`broadVocabLexicon[w]`/
morphology-matched base) and the synthesized generic-fallback branch (2346). Today this distinction
is thrown away — every `vocab` entry is pushed with only `{word, data, level}`. This milestone adds
one field: `curated: boolean` (or a `source: 'dictionary'|'supplemental'|'broad'|'morphology'|
'fallback'` string, more informative for §3.4's transparency requirement and future analytics — no
behavior difference, string is preferred).

- **"Important SAT Words" (default view)**: `a.vocab.filter(v => v.curated)`.
- **"Show all words" (expansion)**: the full `a.vocab` array, toggled via a plain checkbox/button
  driving a CSS class or re-render — no new data fetch, everything is already in memory.
- This does **not** change the Overview tab's `핵심 어휘` count (`a.vocab.length`), which continues
  to reflect total extracted vocabulary — the Important/Show-all toggle is a Vocabulary-tab display
  concern only, not a change to what `analyze()` collects.

### 3.3 Word Book data/read flow, `vocabularyProgress` reuse, Review Queue

**Current shape** (`saveSessionForUser()`, index.html:4664): `{id: userId+"_"+word, userId, word,
lastSeen}`, written automatically for every vocab word whenever a whole study set is saved
(`if(auth.userId) saveSessionForUser()` inside `saveCurrent()`).

**Extended shape (additive fields only, same `idbPut`, same store, same key)**:
```js
{
  id: userId+"_"+word,       // unchanged — natural one-row-per-user-per-word key
  userId,                     // unchanged
  word,                       // unchanged
  lastSeen,                   // unchanged — passive "last appeared in an analyzed passage" signal
  savedToWordBook: boolean,   // NEW — explicit user action, default false/absent
  savedAt: isoString|null,    // NEW — when Save to Word Book was clicked
  reviewState: 'new'|'reviewing'|'learned', // NEW — default 'new' when absent
  reviewedAt: isoString|null  // NEW — last state-change timestamp
}
```
Old records (written before this milestone) simply lack the new fields — every read site treats
`undefined` as the documented default (`savedToWordBook` → falsy/not-saved, `reviewState` →
`'new'`), matching this codebase's established defensive-read convention (e.g. `loadSaved()`'s
missing-`status`-defaults-to-`'success'` pattern from Milestone 2). **No migration script, no data
rewrite.**

**Write paths:**
1. Existing passive write (unchanged): whole-passage save still updates `lastSeen` for every vocab
   word in that analysis.
2. **New**: clicking "Save to Word Book" on a card calls `idbPut("vocabularyProgress", {...merge of
   any existing record for this id, word, userId, savedToWordBook:true, savedAt:now})` — an
   explicit, single-word action, independent of whether the whole passage is ever saved. Must read
   the existing record first (`idbGet`) and merge, not blind-overwrite, so a state change doesn't
   clobber `lastSeen` or vice versa.
3. **New**: the review-state selector similarly reads-merges-writes just the `reviewState`/
   `reviewedAt` fields.

**Read path (Word Book / Review Queue view):**
- `idbGetAllByUser("vocabularyProgress", auth.userId)` (existing helper, unmodified) →filter to
  `savedToWordBook===true` for Word Book, or group by `reviewState` for Review Queue. These can be
  **one view with two filter modes** (simpler, one new render function) or two tabs — recommend one
  view with a segmented control ("저장한 단어" / "복습 중" / "학습 완료"), since the underlying data
  and card-rendering are identical; only the filter differs. Smaller diff, matches "keep changes
  minimal."
- Each returned `{word}` record is rendered via `buildVocabCardData(word)` (§3.5) — **not** by
  re-running `analyze()` or requiring the original passage in memory. This is why the extraction in
  §3.5 is required, not optional.
- **Guest (no `auth.userId`) state**: render the same message pattern as `reportView()`'s existing
  guest branch ("게스트 모드에서는... 로그인 버튼에서 로컬 프로필을 만들어 주세요"), reworded for
  vocabulary — do not invent a new guest-mode data path (e.g. localStorage) for this milestone; this
  keeps the feature's data model single-sourced and matches the CEO's reuse-first direction.

**Why this satisfies "do not change the schema unless proven necessary":** it isn't necessary. The
existing `keyPath:"id"` + `userId` index already support every query this milestone needs
(get-all-by-user, then filter in memory — the store's total per-user row count is bounded by
vocabulary size, not large enough to need a new index for `savedToWordBook` or `reviewState`).

### 3.4 Fallback transparency (student-friendly, non-technical)

Curated cards get a small, plain-language badge/accent — e.g. a colored left-border or a small
"사전 수록 단어" ("dictionary word") tag matching the app's existing badge visual language (reusing
the `.level` badge pattern's styling approach, not its exact text). Fallback cards get a visually
distinct but *not alarming* treatment — e.g. a neutral "일반 설명" ("general explanation") tag and a
slightly muted card background — never the words "fallback," "AI," "provider," "generic," or
similar internal/technical terms in the student-facing copy. (Internal code comments may of course
use precise engineering language — this constraint is about the rendered UI text only.)

### 3.5 `buildVocabCardData(word, passageText)` — the reuse-enabling extraction

New standalone function, factored out of `analyze()`'s inline lookup (2321-2347) with **identical**
output for the cases `analyze()` already handles, plus one new capability:

```js
function buildVocabCardData(word, passageText){
 // 1. Try dictionary[word] / supplementalVocab[word] / broadVocabLexicon[word] / morphology-based
 //    base lookup, in the same priority order analyze() already uses.
 // 2. If none match, synthesize the existing generic-fallback shape (same fields, same
 //    fallbackKoreanGloss()/fallbackExamplePool cycling logic).
 // 3. NEW: if passageText is provided, extract the containing sentence via the existing
 //    splitSentences()/word-boundary matching for "context meaning" (§3.1). If passageText is
 //    absent (e.g. rendering a saved word from a past session with no passage in memory), the
 //    context-meaning slot is simply omitted from the card — not an error state.
 // Returns: {word, level, data:[...existing 7-element shape...], curated:boolean, contextSentence:string|null}
}
```
`analyze()`'s vocab-building loop becomes a thin wrapper that calls this once per selected word;
`vocab(a)` and the new Word Book/Review Queue renderer both call it too — one code path, one place
to keep the "curated flag" and "context sentence" logic correct.

### 3.6 AI enhancement interface

New `taskType`, `"vocab-context"`, added to the **existing** `gemini` adapter's `canHandle()`
(index.html:1968: `canHandle(taskType){return taskType==="summary"}` becomes `return
taskType==="summary"||taskType==="vocab-context"`). Its `request()` branches on `taskType` to build
a different prompt/parse path for `vocab-context` than for `summary`, reusing the same
`getDevApiKey("gemini")`/timeout/error-shape conventions already established (and already proven
live) for the summary adapter. No new adapter object, no new provider, no router change beyond this
one `canHandle` line and the branched prompt logic inside the existing `request()`.

- **Payload**: `{word, sentence, localDefinition}` only — no student identity, no passage title,
  matching the same privacy discipline already applied to the summary adapter (PM-Spec §9's "never
  the source of truth... optional enhancement" framing, and the earlier Gemini Summary UI
  precedent's "send only what's needed" rule).
- **Call site**: a button inside the card's collapsible zone ("AI: 왜 이 뜻일까요?"), user-triggered
  only (never automatic), following the exact `loadGeminiSummary()` pattern already shipped
  (render-time `getDevApiKey("gemini")` check to decide whether the button or a disabled/
  informational note renders; synchronous in-flight guard; `try/finally`; graceful failure message
  that never disturbs the local content around it).
- **Result placement**: appended into its own small area within the card, never replacing the local
  `data[1]`/`data[2]` (English/Korean definition) or the local context sentence — satisfies "AI is
  never the source of truth for basic lexical data."

## 4. Risks and tradeoffs

**HIGH**
1. `buildVocabCardData()` must produce byte-identical output to today's inline logic for every
   existing test case, or the Vocabulary tab regresses. *Mitigation:* Development Task 1 is
   extraction-only (no behavior change), verified by a direct before/after comparison across a
   fixed set of passages before any new feature logic is added on top.
2. Word Book/Review write paths must read-merge-write, not blind-overwrite `vocabularyProgress`
   records, or they'll silently destroy the existing `lastSeen` passive-tracking data (or vice
   versa — a whole-passage save overwriting a student's saved/reviewed state). *Mitigation:*
   explicit `idbGet` merge step specified in §3.3, tested with an interleaving-order regression case
   (save-to-word-book, then whole-passage save, then verify `savedToWordBook` survived).

**MEDIUM**
3. The Important/Show-all toggle must not change `a.vocab.length` (the Overview tab's stat) or
   `makeQuestions()`'s vocab-in-context question generator input — both must continue operating on
   the full extracted list, not the filtered display. *Mitigation:* the filter is display-only,
   applied inside `vocab(a)`'s renderer, never mutating `a.vocab` itself.
4. Guest-mode Word Book/Review Queue must not throw if `auth.userId` is unset. *Mitigation:* same
   guard pattern as `reportView()`, checked first.
5. Old `vocabularyProgress` records without the new fields must render sensible defaults, not
   `undefined`/`NaN` in the UI. *Mitigation:* explicit default-coalescing at every read site (§3.3).

**LOW**
6. Word family/root-etymology fields will be empty for nearly all words this milestone (§3.1) —
   acceptable per CEO direction, but Developer/QA should confirm the empty state renders cleanly
   (no visible placeholder text, no layout gap) rather than looking broken.
7. The new `gemini` `"vocab-context"` taskType shares the adapter's existing timeout/error handling,
   already proven live (Gemini Summary UI's smoke test) — low incremental risk, but QA should still
   verify the branch selects the correct prompt path and doesn't cross-contaminate the `"summary"`
   path.

## 5. Implementation plan (proposed Developer Tasks)

1. **Extract `buildVocabCardData()`.** Pure refactor — no visible behavior change. Regression-test
   against current `vocab(a)` output for a fixed passage set before proceeding.
2. **Tag curated vs. fallback** in the vocab-building loop (`curated`/`source` field) and add
   context-sentence extraction to `buildVocabCardData()`. Still no visible UI change yet (data-only).
3. **Redesign the Dictionary Card** (`vocab(a)` template): above/below-fold split via `<details>`,
   new field slots (context meaning, word family/etymology placeholders, save/state controls —
   controls non-functional/disabled in this task), fallback-transparency styling (§3.4). Mobile
   layout verified against the existing responsive breakpoints.
4. **Important/Show-all toggle** using the `curated` flag from Task 2. No data change, display only.
5. **Word Book write + read**: extend `vocabularyProgress` records (additive fields, read-merge-
   write), wire the Save button, build the Word Book read view (or combined view per §3.3) using
   `buildVocabCardData()`, guest-mode gating matching `reportView()`.
6. **Review state**: extend the same write path with `reviewState`/`reviewedAt`, wire the state
   selector, extend the Task 5 view with state filtering/grouping.
7. **AI enhancement layer**: extend the `gemini` adapter's `canHandle()`/`request()` for
   `"vocab-context"`, wire the card's AI-explanation button following the established
   `loadGeminiSummary()` pattern exactly (render-time key check, in-flight guard, graceful failure).
8. **`DEVELOPMENT_PLAN.md` correction** (documentation-only) + **regression pass** covering: normal
   vocab tab rendering, Important/Show-all toggle, Word Book save/reload, review-state persistence,
   guest-mode messaging, AI button fail-closed-without-key and success-with-mocked-key paths, and a
   full existing-tab/save/translate/quiz sweep to confirm no unrelated regression.

Tasks 1-4 ship the Dictionary Card + selection-logic priorities (CEO's A/D/E/F); Tasks 5-6 ship Word
Book + Review (B/C); Task 7 ships the AI layer; Task 8 closes out documentation and verification.
This ordering lets each task be independently regression-tested, matching this repo's established
per-task-commit convention.

## 6. Test strategy pointer

Per `.ai-company/TESTING_STANDARDS.md` (no automated framework; this environment additionally has no
Node/jsdom, so live in-browser testing via the Browser tool, as used throughout Milestone 3):

- Task 1: before/after output diff of `buildVocabCardData()` vs. the current inline logic, across a
  passage set that exercises all four lookup tiers (`dictionary`/`supplemental`/`broad`/morphology)
  plus the pure-fallback case.
- Tasks 2-4: visual/DOM inspection of the redesigned card at a mobile viewport width, confirming
  above-the-fold content and the `<details>` collapse; toggle behavior confirmed via DOM diff.
- Tasks 5-6: read-merge-write regression case from Risk 2 above; guest-mode render check; reload
  persistence check (save a word, reload the page, confirm it's still marked saved).
- Task 7: mocked-Gemini-response success path, mocked-failure path, no-key path (all mirroring the
  exact test methodology already used and documented for the Gemini Summary UI feature), plus
  confirmation the `"summary"` taskType still behaves identically (no cross-contamination).
- Task 8: full 8-tab regression sweep + save/translate/quiz spot-check, matching every prior
  milestone's closing regression pattern.
- Overall: verify the CEO's §8 product-success flow end-to-end in one session (analyze → find an
  important word → context meaning visible → hear it → expand details → save → reload → mark
  reviewed) as a final acceptance walkthrough before QA sign-off.

## 7. Status

**DRAFT.** Awaiting CEO approval before Development begins.

---

## Handoff — Milestone 4 Architecture

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/ARCHITECT.md`, `.ai-company/CODING_STANDARDS.md`,
  `docs/milestones/milestone-04/01-PM-SPEC.md` (confirmed APPROVED before starting). Targeted code
  reads: `analyze()`'s vocab-building loop, `vocab(a)`, `.word-card`/`.meaning-block`/`.syn-ant` CSS,
  `openDB()`/`idbPut`/`idbGet`/`idbGetAllByUser`/`idbClearUser`, `saveSessionForUser()`,
  `reportView()`'s guest-mode pattern, the `gemini` adapter's `canHandle()`/`request()`, and the
  `loadGeminiSummary()` precedent from the prior milestone.
- **Scope completed:** Resolved all eight items the CEO asked Architecture to define (Dictionary
  Card structure, important-word selection logic, Word Book data/read flow, `vocabularyProgress`
  reuse, Review Queue behavior, AI enhancement interface, mobile presentation, migration/regression
  risks) and produced an 8-task Developer breakdown. Confirmed the "no schema change" constraint is
  achievable (additive record fields only) rather than assumed.
- **Files changed:** `docs/milestones/milestone-04/02-ARCHITECTURE.md` (new). No application code
  touched, no branch created, no commit made — per `.ai-company/ARCHITECT.md`, this phase designs,
  it does not implement.
- **Commits created:** None this session.
- **Tests performed:** N/A (Architecture phase). Test strategy specified in §6 for Development/QA.
- **Unresolved risks:** §4's seven items, all with documented mitigations; none block Development.
- **Next agent:** CEO, for the architecture approval gate; then Senior Developer.
- **Explicit stop point:** Awaiting CEO approval (or revision request) on this `02-ARCHITECTURE.md`,
  per the task's explicit "STOP after Architecture for CEO approval" instruction. No Development
  work has been performed.
