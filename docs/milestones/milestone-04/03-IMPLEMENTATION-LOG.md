# Milestone 4 — Implementation Log: Dictionary / Vocabulary Experience Upgrade

Senior Developer session, executed under `AUTONOMOUS CONTINUATION MODE: ON`
(`AI-Company/GOVERNANCE/AUTONOMOUS_CONTINUATION_POLICY.md`, referenced from this repo's `CLAUDE.md`
§5) per the CEO's explicit Checkpoint A pass-through instruction. Implements
`02-ARCHITECTURE.md` (APPROVED 2026-08-12) exactly, using the 8-task breakdown across Checkpoints
A/B/C/D the CEO directed to reuse.

**Branch:** `feature/vocab-experience-upgrade`, forked from `origin/main` @
`174dc37e237e0731f1ee16c7e54223b965431c51`.

**Testing method:** live in-browser testing (Node/jsdom unavailable in this environment, same
constraint as every prior milestone on this line).

## Checkpoint A — Dev Tasks 1-4

- **Dev Task 1** (`8e37f6f`) — extracted `buildVocabCardData(word)` from `analyze()`'s inline
  per-word lookup. Verified via a true before/after parity test (pre- and post-refactor
  `index.html` served side by side, same 12-word test passage, identical output). One documented,
  deliberate deviation: the generic-fallback example sentence is now chosen deterministically from
  the word itself rather than its position within the original analysis batch, since the function
  now also needs to work standalone with no batch context (Word Book). See AUTO-DECISION LOG below.
- **Dev Task 2** (`6d46f87`) — added `curated` (boolean) and `contextSentence` (local, free, not
  AI-generated) to `buildVocabCardData()`'s return shape. Verified: correct per-word tagging,
  correct standalone (`contextSentence:null`) behavior with no passage.
- **Dev Task 3** (`6e8894d`) — redesigned the Dictionary Card: above/below-fold split via native
  `<details>`, plain-language curated/fallback tag, inert Save/Review controls (finalizing the
  above-the-fold layout before Tasks 5-6 wire them). `contextSentence` explicitly escaped (new
  passage-derived content, unlike `v.word`/`v.data` which are safe by construction). Verified:
  mobile (375px) and desktop viewports, `<details>` expand/collapse, XSS payload in a passage
  sentence confirmed escaped.
- **Dev Task 4** (`2cd922b`) — default "Important SAT Words" (curated-only) view with a "Show all
  words" toggle. Display-only; `a.vocab` never mutated. Verified: exact counts (6 curated / 11
  total / 5 fallback in the test passage), Overview tab's word count unaffected by the toggle.

**Checkpoint A regression:** full 8-tab render sweep, save/load, Risk A/B/C fixes,
`providerRegistry`/Gemini Summary UI, PDF/photo import functions, IndexedDB's 6 stores, 7 responsive
`@media` blocks — all confirmed intact. Reported to CEO; CEO passed Checkpoint A and authorized
autonomous continuation through B/C/D.

## Checkpoint B — Dev Tasks 5-6

- **Dev Task 5** (`4a90434`) — Word Book: explicit per-word save/unsave, reusing
  `vocabularyProgress` (same store, same keyPath, same `userId` index) with additive fields only
  (`savedToWordBook`, `savedAt`) — no schema change. New `updateVocabProgress()` is the single
  read-merge-write entry point for every writer on this store, including the pre-existing passive
  "seen" tracker in `saveSessionForUser()`, which was a blind `idbPut()` that would have silently
  erased Word Book/review state on the next whole-passage save (architecture §4 risk 2) — fixed as
  part of this task since it's the same store this task extends. New "단어장" (Word Book) tab,
  handled in `render()` before the `if(!a)` guard exactly like the existing "report" tab
  (account-level, not tied to a passage), with the same guest-mode gating pattern as `reportView()`.
  `vocabCardTemplate()` extracted from `vocab(a)` so both call sites share one template, fed by
  `buildVocabCardData(word)` for cold reconstruction of a saved word's full card.
  Verified: guest gate, save/unsave, the read-merge-write fix (confirmed directly — a whole-passage
  save after a Word Book save preserved `savedToWordBook` while correctly updating `lastSeen`), and
  **true reload persistence** via an actual page navigation + re-login with zero passage in memory.
- **Dev Task 6** (`193e1f8`) — Review/learned state: a simple new → reviewing → learned → new cycle
  per word (not a spaced-repetition schedule, per the CEO's minimal-scope direction), same
  read-merge-write path (`reviewState`, `reviewedAt` — additive). Word Book gained a segmented
  filter/count control (전체/새 단어/복습 중/학습 완료) — one view with a filter, not a second tab.
  Verified: state cycling, filter counts exact, filtering correctly excludes non-matching words,
  full 9-tab regression clean, mobile screenshot confirmed clean layout.

**Checkpoint B regression:** same sweep as Checkpoint A plus the new `wordbook` tab — all clean, no
Human Gate encountered. Continued automatically to Checkpoint C per the CEO's standing instruction.

## Checkpoint C — Dev Task 7

- **Dev Task 7** (`3759632`) — optional Gemini "vocab-context" enhancement: extends the *existing*
  `gemini` adapter with one new `taskType` (`canHandle` now accepts `"summary"` or
  `"vocab-context"`, still correctly excludes `"translation"`) — no new provider, no router change.
  Shares the proven fetch/timeout/error shell from the Milestone 3 Gemini Summary UI feature; only
  the prompt and response field differ. Button appears inside each card's collapsible zone only when
  a dev key is configured *and* a context sentence exists to explain (a Word Book card reconstructed
  cold has no context sentence and correctly shows no button). Per-word in-flight guard (an object,
  not a single flag, since multiple cards can each have an open request). Payload is exactly
  `{word, sentence, localDefinition}`.
  Verified: button visibility toggles correctly on key presence; full success flow via a mocked
  response (payload confirmed to contain only the `contents` key — no extra metadata); mocked
  HTTP 500 failure degrades gracefully without touching the local definition; double-click guard
  confirmed (2 overlapping calls → 1 request); an XSS payload in a mocked AI response confirmed
  escaped; full 9-tab regression clean; no new provider registered.

**BLOCKED_HUMAN_INPUT:** a real Gemini live-API round-trip for the new `vocab-context` taskType
requires a CEO-entered browser-session credential (same constraint as the Milestone 3 Gemini Summary
UI feature). Not performed this session. All other verification for this task (mocked success/
failure, static review, regression) was completed and is not blocked by this.

## Checkpoint D — Dev Task 8

- **Branding applied** (CEO-approved, `01-PM-SPEC.md` §9.4): `<title>`, `<h1>`, and the subtitle
  `<p>` updated to `SAT English Learning Studio 2026 — AI` / `SAT Reading · Vocabulary · Grammar ·
  AI Study Tools`, replacing the rejected `AI-Powered SAT Reading · Vocabulary · Grammar · Practice`
  wording (never implemented) and the prior `... 2026 V6` / `Passage Analysis · Vocabulary · Grammar
  · ESL · SAT Practice` text.
- **`DEVELOPMENT_PLAN.md` corrected**: the stale "~40-word" figure in the existing "Milestone 3 —
  Dictionary Upgrade" section now has a correction note (original text preserved inline, per
  `.ai-company/WORKFLOW.md`'s append-not-hide rule) pointing to the real ~1,141-word baseline found
  in `01-PM-SPEC.md` §2. Also added this milestone's own entry, following the same
  numbering-collision pattern already established for the Gold Master Adoption track's "Milestone
  3" vs. this document's original "Milestone 3."
- **Full focused regression** (this session, post-branding): all 9 tabs render; branding confirmed
  applied (`document.title`/`<h1>`/subtitle all match exactly); Risk A/B/C fixes intact; router/
  Gemini dormant-without-key intact; IndexedDB schema still exactly 6 stores; PDF/photo import
  functions present; 7 responsive `@media` blocks; no translation left in `'loading'` state.

## AUTO-DECISION LOG

Per `AI-Company/GOVERNANCE/AUTONOMOUS_CONTINUATION_POLICY.md` — Level 2 decisions from this session:

```
Decision: Fallback-example selection in buildVocabCardData() changed from batch-position-based
  to word-deterministic.
Reason: The function must work standalone (Word Book/Review Queue reconstruct a saved word's card
  with no original analysis batch in memory), so a position-dependent choice cannot be replicated
  cold. A per-word-deterministic choice is the smallest change that makes the function correctly
  standalone while preserving the same visible behavior shape (a generic example sentence for
  non-curated words).
Level: 2
Risk: Low — cosmetic only (which of 12 template sentences is shown for an unmatched word), never
  affects curated (real dictionary) words, never affects correctness of any other field.
Reversible: YES — the position-based variant could be restored, but would then be unusable
  standalone, reopening the exact problem this change solves.
Alternative considered: Keep position-based selection only inside analyze()'s loop, and give
  buildVocabCardData() a separate, simpler standalone-only fallback path (two code paths for the
  same concern). Rejected as unnecessary duplication for a cosmetic difference.
Result: Accepted at Checkpoint A; CEO explicitly confirmed acceptance in the Checkpoint B/C/D
  authorization message ("The fallback-example deterministic choice from Checkpoint A is accepted").
```

```
Decision: Word Book/Review Queue implemented as one view with a segmented filter, not a second tab
  beyond the single new "단어장" tab, and not split further by state.
Reason: Architecture §3.3 recommended this explicitly ("smaller diff... since the underlying data
  and card rendering are identical; only the filter differs"). Matches the smallest-safe-change
  principle.
Level: 2
Risk: Low — purely a UI organization choice, easily changed later without any data model impact.
Reversible: YES.
Alternative considered: Separate tabs per review state. Rejected as more UI surface for no added
  capability over a filter control.
Result: Implemented as designed; verified working (exact filter counts, correct exclusion).
```

```
Decision: saveSessionForUser()'s passive vocabularyProgress write routed through the new
  updateVocabProgress() read-merge-write helper, changing existing (pre-Milestone-4) code as part
  of Dev Task 5 rather than only adding new code.
Reason: Necessary to prevent the new Word Book/review fields from being silently destroyed by the
  existing passive write, per architecture §4 risk 2 — not optional, and not an unrelated change
  since it modifies the same store this task is extending.
Level: 2 (borderline — could be read as Level 1, since the architecture doc already named this
  exact fix as required; logged for visibility given it touches pre-existing code).
Risk: Low — behavior for the existing lastSeen field is unchanged (verified directly); only adds
  merge-safety for fields that didn't exist before this milestone.
Reversible: YES.
Alternative considered: Leave the passive write as a blind idbPut() and accept the data-loss risk.
  Rejected — the architecture document explicitly identified this as a required mitigation, not an
  optional one.
Result: Implemented and verified (whole-passage save after a Word Book save preserved
  savedToWordBook while updating lastSeen).
```

No other Level 2 decisions or unusually important Level 1 decisions this session; all other choices
were direct applications of the approved architecture with no reasonable alternative to weigh.

## Files changed

`index.html` only, across 7 code commits (Dev Tasks 1-7) plus the branding/regression pass folded
into this documentation commit. `LATEST_GOLD_MASTER_NEXT.html` untouched throughout (checksum
`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`, reconfirmed unchanged).
`DEVELOPMENT_PLAN.md` corrected as described above. No IndexedDB schema change, no new provider, no
other tab's core logic touched.

## Commits

1. `4c708b3` — Docs: PM-Spec + Architecture (CEO-approved)
2. `8e37f6f` — Dev Task 1: extract buildVocabCardData()
3. `6d46f87` — Dev Task 2: curated flag + context sentence
4. `6e8894d` — Dev Task 3: redesign Dictionary Card
5. `2cd922b` — Dev Task 4: Important SAT Words / Show All toggle
6. `4a90434` — Dev Task 5: Word Book
7. `193e1f8` — Dev Task 6: Review/learned state
8. `3759632` — Dev Task 7: Gemini vocab-context enhancement
9. (this commit) — Dev Task 8: branding, DEVELOPMENT_PLAN.md correction, implementation log

## Remaining limitations

- Word family / root-etymology fields are not present in the Dictionary Card — no current lexicon
  entry carries this data, and the CEO's instruction was explicit not to spend this milestone
  building a new dataset for them. The card template has room for them to be added later.
- A real Gemini live-API round-trip for `vocab-context` was not performed (`BLOCKED_HUMAN_INPUT`
  above) — the adapter logic, prompt construction, and response parsing were verified via mocked
  responses only, following the exact same pattern already accepted for the Milestone 3 Gemini
  Summary UI feature's own live-test constraint.
- No automated test framework / Node·jsdom in this environment — all verification is live in-browser
  and manual per session, consistent with every prior milestone on this line.

## Recommended QA scope

Independently re-verify, per `.ai-company/QA.md`, against `01-PM-SPEC.md`'s 10 acceptance criteria:
1. Important SAT Words default view + Show all toggle.
2. Curated vs. fallback visual distinction (non-technical wording).
3. Mobile above-the-fold content for the redesigned card.
4. Word Book save persists via `vocabularyProgress`, no new object store.
5. Review/learned state + Word Book filter view.
6. Guest-mode messaging for Word Book (no throw, matches Report tab's pattern).
7. AI enhancement fails closed without a key; every other criterion holds with zero AI involvement.
8. AI enhancement succeeds with a key configured (QA may need to arrange its own key, or accept the
   mocked-response verification already on record, matching this session's own constraint).
9. `DEVELOPMENT_PLAN.md`'s corrected word-count figure.
10. The CEO's end-to-end product-success flow (passage → important word → context meaning → hear it
    → inspect details → save → reopen → mark reviewed) in one continuous walkthrough.

Also recommend QA independently re-verify the `updateVocabProgress()` read-merge-write fix (Risk-2
mitigation) with its own test, given it modifies pre-existing (not brand-new) code.

---

## Handoff — Milestone 4 Development (Checkpoints A-D complete)

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Source documents read:** `CLAUDE.md` (including §5 AI Company Shared Governance),
  `AI-Company/GOVERNANCE/{AI_COMPANY_OPERATING_RULES,AUTONOMOUS_CONTINUATION_POLICY}.md`,
  `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`, `.ai-company/DEVELOPER.md`,
  `.ai-company/CODING_STANDARDS.md`, `.ai-company/GIT_RULES.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-04/01-PM-SPEC.md` (confirmed APPROVED), `02-ARCHITECTURE.md`
  (confirmed APPROVED).
- **Scope completed:** All 8 Developer Tasks across Checkpoints A/B/C/D, executed autonomously per
  the CEO's explicit instruction after Checkpoint A passed, with one `BLOCKED_HUMAN_INPUT` item
  (real Gemini live test) correctly not blocking any other work. No work outside the approved
  architecture was implemented.
- **Files changed:** `index.html` (7 code commits + this documentation-adjacent branding change),
  `DEVELOPMENT_PLAN.md` (corrected + new milestone entry), this file (new).
- **Commits created:** listed above — 9 total on `feature/vocab-experience-upgrade`, none on `main`.
  No push, no merge.
- **Tests performed:** see per-checkpoint sections above and the Task 8 comprehensive regression
  sweep; full detail in each commit message.
- **Unresolved risks:** none newly introduced. The `BLOCKED_HUMAN_INPUT` item (real Gemini live
  test) remains open pending a CEO-provided key in a controllable browser context.
- **Next agent:** QA Engineer, per the "Recommended QA scope" above.
- **Explicit stop point:** per the CEO's Checkpoint D instruction, this session stops here — before
  merge or release. Awaiting CEO review of these results.
