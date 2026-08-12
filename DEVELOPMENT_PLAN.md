# Development Plan — SAT English Learning Studio 2026

This plan sequences the work identified in the initial project review into eight milestones. Each milestone is scoped to be independently shippable and testable before moving to the next. No feature behavior should change outside of a milestone's stated scope.

## Milestone 1 — Bug Fixes Only — ✅ Completed 2026-08-06

Goal: fix defects in existing behavior without adding or redesigning features.

- [x] Escape all user- and API-controlled content before inserting it via `innerHTML` (passage preview in `overview()`, sentence translations in `translation()`, saved item titles/text in `renderSaved()`) to close the DOM-based XSS holes. — commit `7e862e6`
- [x] Randomize the correct-answer position in `makeQuestions()` instead of always placing it at index 0 (index 1 for Transitions), so the SAT practice questions can't be gamed by pattern. — commit `c3b7f07`
- [x] **(Discovered during verification of the above, not originally scoped)** Fix `englishOnly()` root cause: a `>=12` Latin-letter minimum silently discarded valid short English choices — e.g. all four real Transitions choices ("For example,", "In contrast,", "Therefore,", "Similarly,") and the short-synonym Precision choices — and replaced them with generic filler text, independent of the answer-shuffle change. Fixed by rejecting on any Hangul presence (was: strip-then-count, which let mixed-language fragments through) and lowering the minimum to `>=3` Latin letters, low enough for short valid words but still rejecting empty/symbol/numeric/Korean content. — commit `c86383c`
- [x] Left the "분석 난이도" (difficulty) selector in place and functionally unchanged per explicit decision — it remains a documented no-op (see TODO comment in `index.html`) rather than being wired up or removed, deferred to Milestone 3 where it will drive real vocabulary/question difficulty. — commit `5e49b0c`
- [x] Add a confirmation step before "새 지문" (New Passage) discards unsaved input. Only prompts when there's actually something to lose (non-empty title/passage or an existing analysis). — commit `d37306d`
- [x] Add a confirmation step before "삭제" (delete) permanently removes a saved study set, naming the item's title in the prompt. — commit `2db26eb`
- [x] Disable the "지문 분석하기" button while `analyze()` is in flight to prevent duplicate/overlapping submissions, re-enabling via `finally` so it can't get stuck disabled on error. — commit `b37795c`

Exit criteria: met. Existing features behave identically for well-formed input; the bugs above no longer reproduce. No new features introduced, no UI redesign.

Verification method: this project has no automated test framework, so each task was verified with a throwaway jsdom-based script (loading `index.html`, running scripts, exercising the changed function, and asserting on DOM/state output), plus a final combined regression sweep re-running all of Milestone 1's checks together before the last commit. Scripts were scratch files, not checked into the repo.

### Milestone 1 follow-up inspection — ✅ Completed 2026-08-06

A full feature-by-feature inspection was run after the items above shipped (all 8 result tabs across five passage types, save/load/search/delete, quiz grading, recording graceful-failure paths). No feature regressions found. The inspection did surface four additional defects not caught by the original fixes; the two Critical and two High findings were fixed immediately per instruction, each in its own commit. Medium/Low findings were logged, not fixed:

- [x] **Critical — XSS in SAT tab.** `sat()` rendered `q.type`, `q.q`, each choice, and `q.why` via `innerHTML` unescaped. Several of these are built from the user's own passage sentences (`makeQuestions()`'s `first`/`second`/`third`/`last`), so a passage containing literal HTML could inject and execute it — confirmed with a live `<img>` element via jsdom, reachable through normal paste-a-passage usage, no adversarial input needed. — commit `c629c9d`
- [x] **Critical — XSS in Grammar tab.** `grammarInsights()` embedded raw regex-captured substrings from the sentence (e.g. the relative-clause match, which captures everything from "who/which/that" to the next comma/semicolon/dash) directly into the notes alongside intentional `<b>` tags, unescaped. Confirmed the same way. Fixed by escaping each capture at its interpolation point while preserving the surrounding `<b>` formatting. — commit `22d16a3`
- [x] **High — corrupted localStorage broke the entire saved-materials feature.** `saveCurrent()`, `renderSaved()`, `loadSaved()`, and `deleteSaved()` all did a bare `JSON.parse()` with no error handling; corrupted storage crashed all four with no user-facing message and no recovery path. Added `getSavedList()`, a resilient read helper that falls back to an empty list (existing empty-state UI already covers this) instead of throwing. — commit `7bca51b`
- [x] **High — silent data loss from `Date.now()` id collisions.** Two `saveCurrent()` calls fired back-to-back could get the identical millisecond-resolution id; `deleteSaved()`'s `filter(x=>x.id!==id)` would then delete both at once. Confirmed empirically (2 rapid saves reliably collided). Fixed by incrementing the id until it's unique against the current saved list. — commit `dfd7397`

**Logged, not fixed (Medium/Low, per instruction to fix only Critical/High):**
- No error handling around `localStorage.setItem` for quota-exceeded on save (plausible after heavy long-term use; distinct from the corrupted-*read* issue fixed above, which only covered `JSON.parse` on existing data).
- A latent design flaw in `examples()`'s duplicate-avoidance `while` loop (if a collision ever recurred, the retry logic wouldn't change on subsequent iterations) — stress-tested with duplicate/adversarial vocab inputs and could not find a path to actually trigger it, since the first-attempt template index is 1:1 with the map index for realistic list sizes. Not reachable in practice, so not actionable now.
- Per-sentence translation API burst / no batching — already explicitly in Milestone 2's scope below, not new.
- The sentence splitter's naive period-handling (surfaced incidentally while constructing an XSS test payload) and the two hardcoded grammar special-cases for the sample passage — both pre-existing, already tracked under Milestone 5.

## Milestone 2 — Translation Reliability — ✅ Development/QA/Review Complete 2026-08-07

Goal: make the translation feature behave predictably for passages beyond the built-in sample.

- [x] Replace the silent "번역 서비스를 불러오지 못했습니다" placeholder (which currently renders as if it were a real translation) with a clearly labeled error/retry state. Implemented via `simpleTranslate()`'s structured `{ko,matched}` return (commit `313757b`), `translateSentenceReliable()`'s explicit `status:'success'|'error'` contract (commit `1563cef`), and `translationRowTemplate()`'s distinct loading/success/error row rendering (commit `fb70aa0`), with the error state's warning-colored styling and retry button finished last (commit `9fad20c`).
- [x] Add request batching/throttling for the per-sentence MyMemory API calls to avoid bursts on long passages. Implemented via `runWithConcurrency()`, bounded to `CONCURRENCY_LIMIT=3` (commit `4f8975d`), wired into `analyze()`'s dispatch loop (commit `9dd5ae8`).
- [x] Add a local caching layer (session-level) so re-analyzing the same passage doesn't re-request identical sentences. Implemented via the in-memory `translationCache` Map (commit `4f8975d`), read/written by `translateSentenceReliable()` (commit `1563cef`).
- [x] Evaluate and document a fallback/secondary translation source for when MyMemory is unavailable or rate-limited. Documented in `docs/milestones/milestone-02/02-ARCHITECTURE.md` §6 (adopts Lingva Translate as secondary, rejects LibreTranslate, retains a corrected `simpleTranslate()` as tertiary fallback) and implemented via `translateLingva()` (commit `1563cef`).
- [x] Add a visible loading state per sentence (not just the global status line) so partial translation progress is legible. Implemented via `translationRowTemplate()`'s loading branch (commit `fb70aa0`), `translation(a)`'s per-row wiring (commit `284ea76`), `analyze()`'s progressive per-sentence dispatch and `updateTranslationRow()` (commit `9dd5ae8`), and the loading-state skeleton/pulse styling (commit `9fad20c`).
- [x] **(Discovered during QA of the above, not originally scoped)** Fix SAVE-1 (High): `saveCurrent()` could persist `translations` entries still in their `{status:'loading'}` shape once progressive rendering (above) began assigning `state.analysis` before translations resolved, producing a permanently stuck, unrecoverable "번역 중…" row if the study set was reopened. Fixed by blocking `saveCurrent()` while any translation is still loading; independently re-verified resolved. — commit `a86f8e0`
- [x] Backward compatibility: `loadSaved()` now defaults any translation entry missing a `status` field (i.e., study sets saved before this milestone) to `'success'` in memory, without rewriting the underlying saved data. — commit `77b82ac`

Exit criteria: met. Any arbitrary English passage (not just the sample) now produces either a real translation (MyMemory → Lingva → phrase-map fallback chain) or an honest, clearly-marked failure state with a retry option — never the old misleading canned string. Verified against all 5 acceptance criteria in `docs/milestones/milestone-02/01-PM-SPEC.md`, independently re-checked by Principal Review against the final integrated code — see `docs/milestones/milestone-02/05-REVIEW-REPORT.md`.

Verification method: same as Milestone 1 — no automated test framework exists in this repo, so each of the 8 Developer Tasks (plus the SAVE-1 fix) was independently verified with freshly-authored, throwaway jsdom scripts by both the Senior Developer (self-test) and an independent QA pass, per `.ai-company/TESTING_STANDARDS.md`. Full QA history for all 8 tasks plus the SAVE-1 fix is recorded in `docs/milestones/milestone-02/04-QA-REPORT.md` (all PASS, no unresolved Critical/High/Medium defects).

**Release status:** Development, QA, and a first Principal Review pass are complete. Release
packaging and the CEO push-approval gate have not yet occurred — this milestone has not been pushed.
See `docs/milestones/milestone-02/05-REVIEW-REPORT.md` for the current gate status.

## Milestone "3" numbering note (Gold Master Adoption track)

A second, unrelated body of work also uses the label "Milestone 3": **Gold Master Adoption +
Multi-AI Migration**, tracked entirely under `docs/milestones/milestone-03/` on the
`multi-ai-v2-latest-dev` branch (forked from `multi-ai-v2-dev`, not from this document's numbered
sequence). Per the CEO's explicit decision recorded in `docs/milestones/milestone-03/01-PM-SPEC.md`
§1/§6a.1, this collision is recorded here rather than resolved by renumbering either track — the
"Milestone 3 — Dictionary Upgrade" section immediately below is unrelated to it and unchanged by
it. See the dedicated entry immediately below this note for the Gold Master Adoption track's status.

## Milestone 3 (Gold Master Adoption track) — Gold Master Adoption + Multi-AI Migration — ✅ Development/QA/Review Complete, pending CEO push approval

Goal: adopt `LATEST_GOLD_MASTER_NEXT.html` (CEO-supplied) as the new application baseline and
re-integrate the Multi-AI V2 router/reliability work already built on `multi-ai-v2-dev`, without
losing the Gold Master's newer subsystems (IndexedDB persistence, PDF/OCR/HEIC import, SAT
question/grammar engines, pronunciation scoring, on-device Chrome Translator) or Milestone 2's
translation reliability improvements. Full document set:
`docs/milestones/milestone-03/{01-PM-SPEC,02-ARCHITECTURE,03-IMPLEMENTATION-LOG,04-QA-REPORT,05-REVIEW-REPORT}.md`.

- **Branch / final commit:** `multi-ai-v2-latest-dev` at `9c4cc731ed6e283964fb8d3095106884ed67d02f`
  (`multi-ai-v2-dev` and `LATEST_GOLD_MASTER_NEXT.html` both left unchanged throughout).
- **Gold Master baseline:** `LATEST_GOLD_MASTER_NEXT.html`, SHA-256
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`, preserved byte-for-byte as the
  protected reference copy; `index.html` seeded from it and confirmed unchanged outside the scoped
  translation/router/persistence regions (Principal Review §2/§3).
- [x] Reconciled the Gold Master's `translateSentence()` (on-device Chrome Translator primary,
  preserved) with Milestone 2's reliability chain (Lingva fallback, retry/backoff, timeout,
  structured `{ko,status,source}`), keeping the Gold Master's sequential dispatch and MyMemory
  429-stop intact — commits `29523bb`, `43b2ccf`.
- [x] Re-inserted the Multi-AI V2 router (`providerRegistry`/`aiRouter`/`legacy-translation`
  adapter) and wired `analyze()`/`retrySentence()` through it — commit `b90362b`.
- [x] Carried over persistence fixes (`getSavedList()` defensive read, SAVE-1 guard,
  `renderSaved()` XSS-escaping fix) without touching the Gold Master's IndexedDB schema — commit
  `89184a7`.
- [x] **Hot-fix, found during QA (not originally scoped):** Risk A — Chrome Translator
  initialization could hang indefinitely with no timeout, stalling translation with a silently
  stuck loading row. Fixed by racing initialization against the existing request-timeout budget and
  failing open to MyMemory. Risk B — `Date.now()`-only save IDs could collide on rapid consecutive
  saves, and deleting one colliding item deleted both (confirmed data loss). Fixed with a
  collision-safe increment, fully backward compatible with existing saved records. Both
  independently re-verified resolved. — commit `9c4cc73`
- [x] **RESOLVED (was: Deferred, Medium, logged as backlog):** Risk C — calling analyze() again
  while one is already running could race; the common outcome was benign ("last request wins"),
  with a narrower console-only failure mode under an uncommon combination of conditions. No crash,
  no data loss, no security exposure. Fixed via a synchronous in-flight guard on `analyze()`
  (`try`/`finally`-protected; a second call while one is active is now a silent no-op) — commit
  `196fac857d35a6d1c59526e55c2ba46191c1d6dc`, merged via
  [`ohcha1/sat-study-2026#2`](https://github.com/ohcha1/sat-study-2026/pull/2). Verified with 5
  focused runtime tests; no other function touched.
- **Multi-AI Router status:** live and wired for translation (`legacy-translation` adapter). Dormant
  `gemini` adapter (summary-only) is present and isolated (no UI call site invokes it, still
  requires `window.SAT_STUDIO_DEV_KEYS.gemini`, fails closed without one). **Update (metadata-sync
  session): a live API smoke test against the real Gemini endpoint has since passed** —
  `gemini-3.1-flash-lite`, `success=true`, `error=null`, ~1261ms observed latency; the key used was
  browser-session-only and was never committed to the repository. It remains dormant/inactive by
  default for end users — this confirms the adapter's request/response handling works end-to-end
  against the real API, not just its fail-closed path, without changing default behavior. No
  additional provider was added.
- **Preserved unmodified (confirmed via direct diff, not just re-tested):** IndexedDB schema (all 6
  object stores), photo/PDF/HEIC import (`importDocument`/`extractPdfPassage`/`getPdfLibrary`/
  `getDocumentOcrWorker`/`convertHeicForOcr`), and all 7 responsive `@media` blocks.

Exit criteria: met — no deferred items remain (Risk C resolved above). QA result: **PASS** (all 10
PM-Spec acceptance criteria met after the hot-fix; see `04-QA-REPORT.md` and its focused
re-verification). Principal Review result: **APPROVE WITH CONDITIONS** (the one condition — this
`DEVELOPMENT_PLAN.md` update — is satisfied by this entry; see `05-REVIEW-REPORT.md`).

**Multi-AI V2 stabilization: COMPLETE.** All three QA-identified risks (A, B, C) are resolved; no
known open defect remains against this baseline. See `docs/BASELINE_MULTI_AI_V2.md` for the full
post-merge baseline record, including current checksums.

- [x] **Gemini Summary UI: MERGED / COMPLETE** (follow-up feature, not part of the original 6
  Developer Tasks). Adds an optional, user-triggered "AI 요약 (선택, Gemini)" action to the Summary
  tab, reusing the existing `providerRegistry`/`aiRouter`/`gemini` adapter unmodified — no new
  provider, no router change, no IndexedDB schema change. The existing local extractive summary
  (`summaryEn`/`summaryKo`) is preserved and remains the default; the Gemini path is additive only
  and never overwrites it. Sends only the passage text to the adapter — no title, grade, date, id,
  or other metadata. Dormant unless a dev key is configured (checked at render time, fails closed
  before any click, not just after). Gemini adapter: live API smoke test previously PASS (real
  endpoint, `success=true`, `error=null`, ~1261ms — key never committed). Gemini Summary UI itself:
  end-to-end mocked-response verification PASS (a second live-key UI test could not be performed
  without exposing the browser-session key to the controlled review surface — see
  `docs/BASELINE_MULTI_AI_V2.md`'s "Gemini Summary UI" section for the full verification-composition
  rationale). Feature commit `8ec8921c944fbe334f631945901621a43b275752`, merged via
  [`ohcha1/sat-study-2026#3`](https://github.com/ohcha1/sat-study-2026/pull/3).

Verification method: no automated test framework exists in this repo (same as Milestones 1-2); this
environment additionally had no Node/jsdom available, so verification used live in-browser testing
(a local static server driven through the Browser tool) instead of the usual throwaway-jsdom
scripts — noted as a methodology deviation in `03-IMPLEMENTATION-LOG.md` for future milestones'
awareness.

**Release status: MERGED.** [`ohcha1/sat-study-2026#1`](https://github.com/ohcha1/sat-study-2026/pull/1)
(Gold Master adoption + Multi-AI router + reliability work),
[`#2`](https://github.com/ohcha1/sat-study-2026/pull/2) (Risk C fix), and
[`#3`](https://github.com/ohcha1/sat-study-2026/pull/3) (Gemini Summary UI) have all merged into
`main` with explicit CEO push/merge approval at each step. Official `main` HEAD:
`12da7b4b6fc6d3d5e46fa188ced5a20e9af6e5d6`. See `docs/BASELINE_MULTI_AI_V2.md` for the full
post-merge baseline record.

*(Prior text, preserved for history: "Official main HEAD: 0b60ba29951bb799c3fd3d8e30230e85636a19f0"
— superseded by PR #3 above.)*

*(Original text, preserved for history: "Development, QA (including a hot-fix cycle), and
Principal Review are complete. Release packaging (`06-RELEASE-NOTES.md`) and the CEO push-approval
gate are the only remaining steps — this milestone has not been pushed or merged to `main`.")*

## Milestone 3 — Dictionary Upgrade

**Correction (recorded by Milestone 4's PM-Spec, 2026-08-12):** the "~40-word" figure below was
stale even when Milestone 3's own work began — direct code inspection found the actual baseline is
~1,141 curated words across `dictionary`/`supplementalVocab`/`broadVocabLexicon` combined, plus
~49,000 IPA entries (curated + a bundled CMU Pronouncing Dictionary conversion). See
`docs/milestones/milestone-04/01-PM-SPEC.md` §2 for the full finding. This does not change this
section's original goal/scope — a scalable vocabulary source is still a legitimate goal — only the
size of the problem it was solving. Original text preserved below, per
`.ai-company/WORKFLOW.md`'s append-not-hide rule.

Goal: replace the ~40-word hardcoded dictionary and grade-level heuristics with a scalable vocabulary source.

- Integrate a real dictionary data source (API or a substantially larger local word list) to replace `dictionary`, `ipaMap`, and `exampleBank`.
- Replace the length-based grade-level heuristic (`levelFor`) with a defensible frequency/CEFR-based leveling approach.
- Replace `fallbackKoreanGloss`'s suffix-guessing with real lookups where possible; keep a labeled heuristic fallback only as a last resort.
- Ensure vocabulary and IPA lookups scale to arbitrary passages without visibly degrading to "no data" for common academic words.

Exit criteria: vocabulary tab produces real, sourced definitions for a representative set of test passages outside the current dictionary's coverage.

## Milestone "4" numbering note (Vocabulary Experience Upgrade track)

A second body of work also uses the label "Milestone 4": **Dictionary / Vocabulary Experience
Upgrade**, tracked under `docs/milestones/milestone-04/` on branch `feature/vocab-experience-upgrade`
(forked from `main` post-Milestone-3-merge, not from this document's numbered sequence — same
pattern as the Gold Master Adoption track's relationship to this document's "Milestone 3"). Per the
CEO's direction recorded in `docs/milestones/milestone-04/01-PM-SPEC.md` §9, this reuses/updates
Milestone 3's dictionary assets (§ above) rather than duplicating or superseding that milestone's
scope wholesale — it prioritizes Dictionary Card UX, Word Book, and Review state over further
dictionary data expansion. The "Milestone 4 — UI Improvements" section immediately below is
unrelated to it and unchanged by it.

## Milestone 4 (Vocabulary Experience Upgrade track) — Dictionary / Vocabulary Experience Upgrade

Goal: make the existing, already-substantial vocabulary data (§ Milestone 3 above) actually useful
to a student — a richer per-word Dictionary Card, an explicit Word Book, simple review/learned
tracking, and an honest curated-vs-general distinction — rather than growing the underlying word
lists further. Full document set:
`docs/milestones/milestone-04/{01-PM-SPEC,02-ARCHITECTURE,03-IMPLEMENTATION-LOG}.md`.

- **Branch:** `feature/vocab-experience-upgrade`, forked from `main` post-Milestone-3 merge.
- [x] Redesigned the Dictionary Card mobile-first (above/below-fold split via native `<details>`),
  added a plain-language curated-vs-general-fallback tag, and a default "Important SAT Words" view
  with a "Show all words" expansion (Dev Tasks 1-4).
- [x] Word Book: explicit per-word save/unsave, reusing the existing `vocabularyProgress`
  IndexedDB store with additive fields only (no schema change) via a single read-merge-write entry
  point (Dev Task 5).
- [x] Review/learned state: a simple new/reviewing/learned cycle per word, with a filtered Word Book
  view — not a spaced-repetition engine (Dev Task 6).
- [x] Optional Gemini "vocab-context" enhancement (why a word's meaning fits its sentence),
  extending the existing `gemini` adapter with one new taskType — no new provider — dormant/fails
  closed without a dev key; core Dictionary Card fully usable without it (Dev Task 7).
- **Branding (CEO-approved):** title `SAT English Learning Studio 2026 — AI`, subtitle
  `SAT Reading · Vocabulary · Grammar · AI Study Tools`, applied to `<title>`/`<h1>`/subtitle text.
- **Preserved unmodified:** IndexedDB schema (still exactly 6 object stores), Multi-AI router,
  Gemini Summary UI, Risk A/B/C fixes, PDF/photo import, all 7 responsive `@media` blocks.

Exit criteria: met — see `docs/milestones/milestone-04/03-IMPLEMENTATION-LOG.md` for full
verification detail per Dev Task.

**Release status:** Development and a full regression pass are complete on
`feature/vocab-experience-upgrade`, not yet pushed, merged, or released — QA/Principal Review/CEO
push-approval have not yet occurred for this milestone.

## Milestone 4 — UI Improvements

Goal: address usability gaps surfaced in the review, without changing underlying analysis logic.

- Add a proper loading indicator during analysis (beyond the status text line).
- Improve error/empty states across tabs (translation failure, empty vocab, no saved items) for clarity.
- Review responsive layout at additional breakpoints (mobile widths below current 980px handling).
- Add basic accessibility pass: focus states, ARIA labeling for tabs/drawer, keyboard navigation for the tab bar.
- Visual polish pass on spacing/typography consistency across tabs.

Exit criteria: usability issues from the review are resolved; no analysis/feature logic changes.

## Milestone 5 — AI Tutor

Goal: replace the templated, hardcoded "analysis" (grammar insights, SAT question generation) with genuine AI-generated content, addressing the gap between the README's "AI Explanation" claim and current template-based logic.

- Design an API integration for real grammar explanation generation (replacing the two hardcoded sample-passage regexes and generic pattern fallback in `grammarInsights()`).
- Design an API integration for genuine, passage-grounded SAT question generation (replacing the fixed 10-template bank in `makeQuestions()`).
- Define cost/rate-limit handling and a graceful fallback if the AI service is unavailable.
- Add clear labeling so users understand what is AI-generated vs. static reference content (dictionary links, phrase bank).

Exit criteria: grammar notes and SAT questions are generated from the actual input passage's content rather than fixed templates, for arbitrary passages.

## Milestone 6 — User Accounts

Goal: move beyond single-browser `localStorage` persistence.

- Design authentication (sign-up/login) and decide on provider (self-hosted vs. third-party auth).
- Design a backend data model for users and their saved study sets, replacing/augmenting `localStorage`.
- Migrate save/load/search/delete flows to the account-backed store, with a migration path for existing local data.
- Add basic account settings (profile, sign-out, data export/delete for privacy compliance).

Exit criteria: a user can log in from a different device/browser and see their saved study sets.

## Milestone 7 — Study History

Goal: turn quiz results and study activity into tracked progress, dependent on accounts from Milestone 6.

- Persist SAT quiz scores per attempt, linked to the study set and user.
- Add a history/progress view (score trends over time, per-passage performance).
- Add vocabulary review tracking (e.g., words marked difficult, spaced-repetition-style resurfacing).
- Add export of history/progress data (CSV at minimum).

Exit criteria: a returning user can see their past quiz scores and vocabulary progress, not just their saved passages.

## Milestone 8 — Deployment

Goal: move from a manually-opened static file to a properly hosted, reliably reachable application.

- Stand up a minimal build/serve pipeline (static hosting for frontend; hosting for any backend introduced in Milestones 6–7).
- Serve over HTTPS to ensure `getUserMedia` (mic recording) and `SpeechRecognition` work reliably (currently at risk under `file://`).
- Add environment configuration for API keys (translation, dictionary, AI tutor) rather than embedding logic client-side where relevant.
- Set up basic CI (lint/build check) and a deployment process (e.g., GitHub Actions to a static host).
- Update README with live URL, setup instructions, and environment variable requirements.

Exit criteria: the app is reachable via a stable URL over HTTPS, with all features (including mic-dependent ones) functioning as expected for end users.

---

## Sequencing Notes

- Milestones 1–4 depend only on the current codebase and can proceed in order without external services.
- Milestone 5 (AI Tutor) can be developed in parallel with 6/7 design work but should land before 6/7 if possible, since account-backed history is more valuable once question/grammar quality is real.
- Milestone 6 (User Accounts) is a prerequisite for Milestone 7 (Study History).
- Milestone 8 (Deployment) should happen incrementally where possible (e.g., deploy after Milestone 1 so bug fixes reach users quickly) rather than being held until the very end, but is listed last as the milestone that formalizes and hardens the full deployment story.
