# Milestone 4 — Dictionary / Vocabulary Experience Upgrade

**Status: APPROVED by the CEO — 2026-08-12.** See §9 for the CEO's decisions on all open questions
and the resulting scope direction. Architecture Design may proceed.

> **Numbering note**, matching the precedent set in `docs/milestones/milestone-03/01-PM-SPEC.md`:
> this claims the next sequential `docs/milestones/milestone-XX/` folder (`milestone-04`), per
> `.ai-company/WORKFLOW.md`'s "folder number follows chronological order of work, not topic
> number" convention. `DEVELOPMENT_PLAN.md` currently labels its own "Milestone 3" as "Dictionary
> Upgrade" (the same underlying topic as this document) — that section's scope is the historical
> problem statement this milestone updates and supersedes (see §2's finding that its "~40-word"
> framing is stale), not a different, unrelated track. Recommend the CEO decide whether to
> renumber `DEVELOPMENT_PLAN.md`'s Milestone 3 to this milestone-04 folder, or keep them separately
> labeled as was done for the Gold Master Adoption track.

## 1. Baseline verification (read-only, this session)

Verified directly, not assumed:

- `origin/main` HEAD: `174dc37e237e0731f1ee16c7e54223b965431c51` — matches expected.
- Application-code baseline `12da7b4b6fc6d3d5e46fa188ced5a20e9af6e5d6` confirmed as an ancestor of
  current `HEAD` (`git merge-base --is-ancestor`).
- `index.html` SHA-256: `7ab6e51243baf5151966d5b641c6cac711fb78dd800768f1d5cf7e9d929076ba` — matches
  the value recorded in `docs/BASELINE_MULTI_AI_V2.md` after PR #3.
- `LATEST_GOLD_MASTER_NEXT.html` SHA-256: `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`
  — unchanged, matches every prior recording.
- No application code modified during this PM session.

## 2. Current vocabulary experience — findings from direct code inspection

(Full method note: read against `index.html` directly — function bodies, object literals, grep-counted
entries — not inferred from `DEVELOPMENT_PLAN.md`'s description, which turned out to be stale; see
below.)

### A. Already works (real, not mock)

- **Vocabulary tab** (`vocab(a)`): renders, per word, a grade-level badge, IPA, a "🔊 미국식 발음
  듣기" speak button (Web Speech API, US-voice preference), part of speech, English definition,
  Korean meaning, synonyms/antonyms, one EN+KO example sentence, and outbound dictionary links
  (Merriam-Webster, Oxford Learner's, Cambridge, Thesaurus.com).
- **Vocabulary data is far larger than `DEVELOPMENT_PLAN.md` currently states.** The plan's
  Milestone 3 section says "the ~40-word hardcoded dictionary." Actual grep-counted entries:
  `dictionary` 142 words, `supplementalVocab` 110 words, `broadVocabLexicon` 889 words — **~1,141
  words total** with full curated definitions, Korean meanings, synonyms/antonyms, and examples.
  **This is a stale problem statement that should not be carried into this milestone's scope
  unmodified** — see Open Questions.
- **IPA/pronunciation data**: curated `ipaMap` + `IPA_VARIANTS` (~1,011 entries) plus a bundled,
  attributed CMU Pronouncing Dictionary conversion (`broadIpaLexicon`, ~48,251 entries) as a final
  fallback — pronunciation data coverage is already very broad and offline (no network dependency).
- **Speak button**: real Web Speech API TTS, works today.
- **Quiz integration**: `makeQuestions()` genuinely draws on the vocab list — one of 11 SAT
  question generators (`genVocabInContext`) produces "word in context" questions, gated to only
  fire for words with real (non-fallback) dictionary entries and a hand-curated distractor set.
- **`vocabularyProgress` IndexedDB store is actively written** on every save (one row per vocab
  word: `{id, userId, word, lastSeen}`), and round-trips correctly through the existing backup/
  export/restore flow.
- **Outbound dictionary links** are real, correctly constructed, open real reference sites.

### B. Present but mock/static (not sourced/dynamic)

- **Fallback vocabulary cards** (word not in any of the three curated lexicons and no morphology
  match): synthesized with a generic "academic or informational word" label, a suffix-heuristic
  Korean gloss (`fallbackKoreanGloss`), one of 12 generic `{word}`-templated example sentences, and
  placeholder synonym/antonym text ("문맥에 따라 확인" — "check based on context"). This fallback is
  clearly labeled in code comments but **not visually distinguished from a real entry in the UI** —
  a student cannot tell a curated definition from a generic filler one by looking at the card.
- **Example sentences for fallback words** are template-cycled, not passage-specific or dynamically
  generated — repeat patterns become visible once a passage surfaces 2+ fallback words.

### C. Missing entirely

- **No per-word save/bookmark ("Word Book") feature.** No flashcard mode. No spaced-repetition
  logic. Grep for word-book/flashcard/bookmark/단어장/spaced-repetition terminology returns nothing
  relevant anywhere in the file.
- **No vocabulary-specific `localStorage` key** — only the whole-study-set save (`satStudioSaved`)
  and the auth session key exist; there is no notion of "words I've saved" separate from an entire
  saved passage.
- **`vocabularyProgress` is write-only from the user's perspective** — populated on every save, but
  no screen anywhere reads it back. There is no "words you've studied," no review/repetition count,
  no due-for-review view. This is the closest existing thing to a Review Notebook data model, and
  today it does nothing visible for the student.
- **No standalone vocabulary quiz/drill/self-test mode** outside the 10-question SAT practice set —
  a student cannot drill just the vocabulary list.
- **No real dictionary API integration** — zero `fetch()` calls for definitions/IPA; 100% local/
  hardcoded data plus outbound links to third-party dictionary *websites* (not APIs).
- **No Multi-AI/Gemini connection** — `aiRouter.request()` is wired only for `"translation"` and
  `"summary"`; nothing vocabulary-related routes through the Multi-AI router today.

### D. Difficult for a student to use / UX rough edges

- **No cap or pagination on the Vocabulary tab's card list** — unlike Examples/ESL (capped at 12),
  every matched word renders unconditionally; a passage with many dictionary hits produces a long,
  uncollapsed scroll.
- **Fallback (generic) cards are visually indistinguishable from curated ones** — a student has no
  way to know which definitions are trustworthy/sourced vs. generic filler.
- **`vocabularyProgress` being write-only** is itself a UX gap: the data needed for a "words you've
  seen before" or review feature already exists per-user in IndexedDB but surfaces nowhere.

### E. What can be reused safely (should not be rebuilt from scratch)

- The three curated lexicons (`dictionary`, `supplementalVocab`, `broadVocabLexicon`) — ~1,141
  words of real, hand-written EN+KO definitions/synonyms/antonyms/examples. Large, valuable,
  already-working asset.
- The IPA data (curated + CMU-derived fallback, ~49,000+ entries) and `speakAmerican()` TTS.
- `dictionaryLinks()`'s outbound-link pattern.
- The `vocabularyProgress` IndexedDB store and its existing write path (`saveSessionForUser()`) —
  the write side is already correct; a Review Notebook feature would primarily need a *read* side
  (a new screen/tab) rather than a new data model.
- `genVocabInContext()`'s existing gating logic (only fires for non-fallback words) as a model for
  how a future standalone vocab quiz mode could reuse the same "real entry only" guard.

## 3. Product branding change — assessment

Proposed: title `SAT English Learning Studio 2026 — AI`, subtitle `AI-Powered SAT Reading ·
Vocabulary · Grammar · Practice`.

**Current state, verified:** `<title>` is `SAT English Learning Studio 2026 V6`; the on-page `<h1>`
is `SAT English Learning Studio 2026` (no "V6", no "AI").

**Concern — the proposed subtitle is not accurate against current functionality, for any of its
four named areas:**

- **Vocabulary**: 100% local/static (§2 above) — zero connection to the Multi-AI router. Calling
  this "AI-Powered" today would be a factually false claim about a named feature.
- **Grammar**: the grammar-insights engine (`detect*` functions — passive voice, relative clauses,
  parallel structure, etc.) is deterministic regex/heuristic pattern matching, not AI-generated —
  also not "AI-Powered."
- **Practice** (SAT quiz): `makeQuestions()`/the `qg*` generator functions are template- and
  rule-based, not AI-generated.
- **Reading**: passage analysis (summary, vocab extraction, question generation) is likewise
  deterministic/local, apart from the translation feature.
- **What IS actually AI/ML-adjacent today**: the on-device Chrome Translator tier (arguably ML, not
  generative AI) and the Gemini summary adapter — which is **dormant by default and requires a
  developer-configured key**, i.e., not reachable by an ordinary end user without local setup.

**Recommendation (flagged, not decided here per PM.md's "do not invent requirements" rule):** this
subtitle should not ship as-is against the current feature set — it would materially overstate what
the product does for an educational audience where accuracy matters. Options for the CEO to choose
among: (a) hold the branding change until Vocabulary (this milestone) and/or Grammar/Practice are
genuinely wired through the Multi-AI router in a way an ordinary user can reach; (b) adopt a
narrower, accurate subtitle today (e.g., naming only what's real — translation reliability and an
opt-in AI summary); (c) proceed with the branding change now as an intentional product-positioning
decision independent of literal feature-by-feature accuracy, which is a legitimate call but should
be made explicitly and knowingly, not by default.

**CEO decision (§9.4): option (b), effectively** — approved title `SAT English Learning Studio
2026 — AI` with subtitle `SAT Reading · Vocabulary · Grammar · AI Study Tools` (drops the
inaccurate "AI-Powered" qualifier from the four core-feature names while still naming "AI Study
Tools" as a distinct, accurately-scoped category). Explicitly rejected the original
"AI-Powered SAT Reading · Vocabulary · Grammar · Practice" wording for the exact reason raised
above.

## 4. In scope (final, per CEO decision — see §9)

1. **Rich Dictionary Card UX** — redesign the per-word card around: word, pronunciation/IPA, part
   of speech, **context meaning** (the word's sense as used in the actual passage sentence),
   English definition, Korean meaning, synonyms, antonyms, word family, root/etymology (structurally
   present, may render empty/hidden for most words in this milestone — see §9.3 and Architecture),
   an SAT-level example, a "Save to Word Book" action, and a learned/review-state control.
   Above-the-fold on mobile: word, pronunciation, part of speech, context meaning, definitions,
   save/state actions. Below-the-fold/collapsible: synonyms/antonyms, word family, etymology,
   example, dictionary links.
2. **Word Book** — an explicit, per-word "save" action (distinct from the existing whole-passage
   save), reusing the `vocabularyProgress` IndexedDB store's existing write path where safely
   possible (additive fields only — see §9.2).
3. **Review Queue / learned status** — a simple learned/reviewing/new state per saved word, and a
   view that surfaces it, built on the same reused store.
4. **Curated-vs-fallback transparency** — student-friendly (non-technical) visual distinction
   between real dictionary entries and generic-fallback ones (§9.5).
5. **Default "Important SAT Words" view** with an optional "Show all words" expansion, so a student
   isn't shown every extracted word by default (§9.4).
6. **Mobile-first presentation** of the redesigned card and the above views.
7. **AI enhancement layer** (optional, dormant without a key, reusing the existing `aiRouter`/
   `gemini` adapter): contextual "why this meaning fits this sentence" explanations and similar
   lexical nuance — never the source of truth for basic lexical data (§9.6).
8. Correct the stale "~40-word" problem statement in `DEVELOPMENT_PLAN.md` (the real baseline is
   ~1,141 curated words — see §2) as part of this milestone's documentation.

## 5. Out of scope (explicitly, to prevent scope creep — per CEO decision)

- **Dictionary data expansion** (growing the curated word lists, adding a real dictionary API) —
  explicitly secondary per the CEO; not part of this milestone unless Architecture finds a specific
  card field (e.g. word family/etymology) genuinely requires it and flags that back up.
- **IndexedDB schema changes** (new object stores, new indexes) — unless Architecture proves
  reusing the existing `vocabularyProgress` store/fields is genuinely insufficient, in which case
  that finding must be surfaced explicitly, not assumed away.
- Changes to the Multi-AI router's core dispatch logic or the `legacy-translation`
  adapter — the AI enhancement layer (§4.7) extends the existing `gemini` adapter's capability, it
  does not change routing/translation.
- Any additional AI provider.
- Grammar, SAT quiz generation, translation, or any other tab's core logic (vocabulary's existing
  read-only *use* by the SAT quiz generator, `genVocabInContext()`, is unaffected but not itself
  in scope to change).
- A full spaced-repetition scheduling algorithm — "Review Queue / learned status" is a simple state
  (e.g. new/reviewing/learned), not an SRS engine, per the CEO's "reuse existing data, keep it
  minimal" framing.
- The product branding/title change is **approved** (§9.7) but is a documentation/copy change, not
  an application-code change — implemented as part of this milestone's Developer phase only if the
  CEO confirms timing; tracked separately from the Dictionary Card work itself.

## 6. Acceptance criteria (testable, final scope)

1. Given a passage, the Vocabulary tab defaults to showing only "Important SAT Words" (curated,
   non-fallback entries), with a visible, working "Show all words" control that reveals the rest.
2. Given a curated-entry card and a fallback-entry card side by side, a student can tell them apart
   without reading any technical/provider terminology (e.g. "fallback," "AI," "provider" must not
   appear in this UI).
3. Given a word's card, the above-the-fold content on a mobile viewport (see Architecture for the
   exact breakpoint) shows word/pronunciation/part-of-speech/context-meaning/definitions/actions
   without requiring a scroll past secondary content to reach them; synonyms/antonyms/word-family/
   etymology/example are reachable via a collapse/expand control.
4. Given a logged-in student, clicking "Save to Word Book" on a card persists that word via the
   existing `vocabularyProgress` IndexedDB path (additive fields only, no new object store), and
   reopening the app later still shows it as saved.
5. Given a saved word, a student can mark it learned/reviewing, and a Review Queue view lists saved
   words filtered/grouped by that state, reading from the same store.
6. Given a guest (not logged in), Word Book/Review Queue shows a clear, non-broken explanation
   (matching the existing Report tab's guest-mode message pattern) rather than failing silently or
   throwing.
7. Given no Gemini dev key configured, the AI enhancement layer fails closed with a graceful,
   student-friendly (non-technical) message, and every other acceptance criterion above still holds
   — i.e., the Dictionary Card is fully usable with zero AI involvement.
8. Given a Gemini dev key configured, requesting a contextual "why this meaning fits" explanation
   returns and displays a real result without altering the local definition/context-meaning content
   underneath it.
9. `DEVELOPMENT_PLAN.md`'s Milestone 3/4 problem statement is corrected to reflect the actual
   ~1,141-word baseline, not the stale "~40-word" figure.
10. A student can complete, within one SAT Studio session and without leaving the app: analyze a
    passage → find an important unfamiliar word → see its context meaning → hear it → inspect
    lexical details → save it → (in a later session) reopen and find it saved → mark it
    reviewed/learned. (Matches the CEO's §8 product success criterion — Architecture should design
    the concrete flow this criterion implies.)

## 7. Open questions as originally raised (for record — see §9 for resolutions)

1. Milestone numbering.
2. Ambition level for "Word Book"/review — minimal reuse of existing data vs. a full
   bookmark/flashcard/spaced-repetition system.
3. Whether out-of-lexicon words should get a real dictionary API integration.
4. The branding change — which option, and on what timeline.
5. Whether to correct `DEVELOPMENT_PLAN.md`'s stale "~40-word" claim now or leave it as history.

## 9. CEO decisions (recorded 2026-08-12)

1. **Milestone numbering** — not explicitly re-addressed; `milestone-04` stands, consistent with the
   precedent already applied to `milestone-03`.
2. **Ambition level for Word Book/Review** — resolved toward the minimal, reuse-first
   interpretation: "the verified existing vocabulary assets are already substantial... dictionary
   data expansion is secondary." Reuse `vocabularyProgress`'s existing persistence path; do not
   change the IndexedDB schema unless Architecture proves it's necessary; prefer surfacing existing
   data over building new systems. Priority order set explicitly: (A) Rich Dictionary Card UX, (B)
   Word Book, (C) Review Queue/learned status, (D) curated-vs-fallback distinction, (E)
   passage-context meaning, (F) mobile-first usability — with dictionary data expansion
   deprioritized below all of these.
3. **Dictionary API integration** — not pursued this milestone (consistent with #2). Word
   family/root-etymology are included in the Dictionary Card's field set, but since no such data
   exists in the current local lexicons, Architecture must decide how these fields degrade
   (structurally present but empty/hidden) rather than assume new data sourcing.
4. **Branding** — resolved. Approved: title `SAT English Learning Studio 2026 — AI`, subtitle
   `SAT Reading · Vocabulary · Grammar · AI Study Tools`. Explicitly rejected: `AI-Powered SAT
   Reading · Vocabulary · Grammar · Practice`, because current Reading/Vocabulary/Grammar systems
   are not primarily AI-generated — matching this document's own §3 concern precisely.
5. **`DEVELOPMENT_PLAN.md` correction** — implicitly confirmed via acceptance criterion §6.9 (carried
   over unchanged into the final scope); correct the stale figure as part of this milestone's
   documentation, following the established append-not-hide pattern.

Additional CEO direction beyond the original open questions, now reflected in §4/§5/§6 above:
- AI's role is explicitly bounded: an optional enhancement layer only (contextual "why this meaning
  fits," SAT nuance, similar-word distinctions, generated examples where useful) — never the source
  of truth for basic lexical data when local structured data exists. Core Dictionary Card must
  remain fully usable with zero Gemini/AI involvement.
- Fallback transparency must use student-friendly visual distinction, with no technical/provider
  terminology exposed in the student-facing UI.
- Word selection must default to an "Important SAT Words" view, not every extracted word, with an
  optional "Show all words" expansion.
- A concrete product success criterion is defined (§6.10): passage → find an important unfamiliar
  word → understand in context → hear it → inspect lexical details → save it → reopen later →
  review/quiz it, entirely within SAT Studio, with a student-noticeable improvement within ~5
  minutes of use. Architecture should design toward this flow explicitly, not just the individual
  feature list.

## 8. Status

**APPROVED by the CEO — 2026-08-12.** Architecture Design (`02-ARCHITECTURE.md`) may begin, scoped
to §4/§5/§6 within the constraints in §9.

---

## Handoff — Milestone 4 PM Phase

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade (numbering provisional,
  see §1 note and Open Question 1).
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/PM.md`, `DEVELOPMENT_PLAN.md`, `docs/BASELINE_MULTI_AI_V2.md`, current `index.html`
  (vocabulary/dictionary/IndexedDB/router code, via direct grep/read — not inferred from docs).
- **Scope completed:** Verified the official baseline (branch, HEAD, both file checksums)
  independently rather than assuming the values given in the task order were still current. Read
  the actual vocabulary/dictionary implementation in `index.html` in full (extraction, rendering,
  pronunciation, synonyms/antonyms, examples, saving, IndexedDB usage, external dependencies,
  Multi-AI router connection, quiz integration) and classified findings as already-works/mock/
  missing/UX-rough-edge/safely-reusable. Assessed the proposed branding change against verified
  current functionality and found it would overstate the product's actual AI capabilities across
  all four named subtitle areas. Drafted this PM-Spec with provisional in/out-of-scope,
  acceptance criteria, and five open questions. No application code touched, no branch created.
- **Files changed:** `docs/milestones/milestone-04/01-PM-SPEC.md` (new, this file). Not committed —
  pending CEO review, per this repo's established pattern.
- **Commits created:** None this session.
- **Tests performed:** N/A (PM phase). All factual claims above verified directly against
  `index.html` (function bodies read, object-literal entries grep-counted), not inferred from
  `DEVELOPMENT_PLAN.md`'s existing description, which this document found to be stale in one
  specific, material way (§2, the "~40-word" figure).
- **Unresolved risks:** None new; the five items in §7 are scope-defining questions for the CEO,
  not risks. Most consequential: Open Question 2 (ambition level for "Word Book"/review) —
  materially changes this milestone's size and whether it needs its own Architecture-level
  sub-scoping.
- **Next agent:** CEO, to resolve §7 and approve or revise this spec.
- **Explicit stop point:** Awaiting CEO approval (or revision request) on this `01-PM-SPEC.md`
  before Architecture Design begins. No Developer-phase work has been performed, per the task's
  explicit "STOP before Developer phase" instruction and the CEO gate in `.ai-company/WORKFLOW.md`.

## Handoff — Milestone 4, CEO approval incorporated

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Scope completed:** Incorporated the CEO's decisions (verbatim message, 2026-08-12) into §4 (In
  scope), §5 (Out of scope), §6 (Acceptance criteria — expanded from 4 draft items to 10 final
  ones), added §9 (CEO decisions) mapping each resolution back to its original open question, and
  changed Status (§8) from DRAFT to APPROVED. No application code touched, no branch created.
- **Files changed:** `docs/milestones/milestone-04/01-PM-SPEC.md` (edited in place).
- **Commits created:** None this session (consistent with this repo's pattern of committing PM/
  Architecture docs later, alongside or after the Architecture Design).
- **Next agent:** Software Architect — proceeding directly in this same session per the CEO's
  explicit "Then create the Milestone 04 Architecture Design" instruction.
- **Explicit stop point:** N/A — continuing directly into Architecture Design per instruction.
