# Milestone 3 — Implementation Log: Gold Master Adoption + Multi-AI Migration

Senior Developer session. Implements `02-ARCHITECTURE.md` (APPROVED 2026-08-12) exactly, using the
6-task breakdown the CEO directed to reuse.

**Branch:** `multi-ai-v2-latest-dev`, created from `multi-ai-v2-dev` HEAD `bc204cf48d6c7ab360329bbf2bcd6bcc581a8781`.

**Testing method note:** `.ai-company/TESTING_STANDARDS.md` specifies jsdom-based throwaway
scripts. Node/npm are not available in this execution environment (confirmed: `node`/`nodejs` not
on `PATH`, no install found). Substituted live in-browser verification instead: a local static
server (`python3 -m http.server`, via `.claude/launch.json`, not committed — see "Uncommitted
tooling" below) serving the real `index.html`, driven through the Browser tool, exercising actual
functions/DOM/localStorage rather than a jsdom simulation. This is arguably stronger verification
(real browser engine, real `fetch` to MyMemory/Lingva) but is manual-per-session rather than a
reusable script, unlike the jsdom convention. Flagging this as a deviation per `DEVELOPER.md`'s
"if you hit a case the architecture doesn't cover, flag it" — QA should decide whether to adopt the
same approach or install Node for its own pass.

## Dev Task 1 — Branch setup + seed index.html from Gold Master

Created `multi-ai-v2-latest-dev`. Committed `LATEST_GOLD_MASTER_NEXT.html` (untracked until now) as
the protected reference copy, and seeded `index.html` as a byte-for-byte copy of it.

- Files changed: `LATEST_GOLD_MASTER_NEXT.html` (new), `index.html` (replaced).
- Verified: `shasum -a 256` of both files identical (`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`)
  immediately after seeding; `multi-ai-v2-dev`'s `index.html` (via `git show`) unchanged
  (`b13794ac8a186fc62b7fc05ac32c40fd45df832e6cad5888b25e2f5104a4884c`).
- Commit: `b14c179` — "Add: seed index.html from LATEST_GOLD_MASTER_NEXT.html — Milestone 3 Dev Task 1"
- (Preceding docs commit `ae3d3eb` — "Docs: add Milestone 3 PM-Spec and Architecture Design
  (CEO-approved)" — records both approved planning documents before any code change.)

## Dev Task 2 — Reconcile translateSentence() into translateSentenceReliable()

Per architecture §3a. Rewrote the Gold Master's `translateSentence()` as `translateSentenceReliable(s,
{bypassCache}={})`: cache → Chrome on-device (Gold Master's, unchanged) → MyMemory (429-stop
preserved unchanged; retry/backoff on non-429 failure added) → Lingva (new, ported from
`multi-ai-v2-dev`) → phrasemap → structured error. Added `LINGVA_INSTANCES`, `RETRY_BACKOFF_MS`,
`REQUEST_TIMEOUT_MS`, `fetchWithTimeout()`, `translateLingva()` (all ported unchanged).

Also fixed `simpleTranslate()` to return `{ko,matched}` instead of the Gold Master's bare string
(which returned the old misleading placeholder as if it were a real translation on no-match) —
required by the architecture's explicit step-5 contract and by `CODING_STANDARDS.md`'s "prefer
explicit error/loading states" rule. Updated its two call sites (`analyze()`'s `translations`
construction and `summaryKo` computation) to the new shapes — a direct, unavoidable consequence of
the contract change, not new scope.

`analyze()`'s dispatch loop calls `translateSentenceReliable()` directly at this point (sequential,
matching the Gold Master's own loop) — full router wiring is Dev Task 4.

- Files changed: `index.html`.
- Commit: `29523bb`
- Verified live: cache miss → Chrome tier skipped (forced `chromeTranslatorState='unsupported'`
  since the on-device Translator API hung in this headless environment — see Risk 1 below) →
  MyMemory success → correct `{ko,status:'success',source:'mymemory'}`. Full `analyze()` run
  produced correct `translations` array and `summaryKo` (verified matches translated cache values).
  `providerUsageThisRun` counts correct including new `lingva` key.

## Dev Task 3 — Port per-row translation UI

Per architecture §3a. Ported `translationRowTemplate()`, `updateTranslationRow()`,
`retrySentence()` from `multi-ai-v2-dev`, adapted to this file's existing `.sentence`/`.en`/`.ko`
markup (not new markup). Added `.sentence.loading`/`.sentence.error` CSS + `sentencePulse` keyframe
(confirmed all referenced CSS variables — `--sky`, `--navy2`, `--danger` — already existed in the
Gold Master). Replaced `translation(a)` with the per-row-template version (also fixes a missing
`escapeHtml()` the Gold Master's original had dropped, both per-sentence and in the "자연스러운 전체
해석" block).

Restructured `analyze()`: `state.analysis` is now assigned *before* translations resolve
(`translations` start as `{ko:null,status:'loading',source:null}` placeholders), so
`updateTranslationRow()` can progressively repaint each row as the still-sequential loop resolves
it — reproducing Milestone 2's "visible per-sentence progress" outcome without concurrent dispatch.
`summaryEn`/`summaryKo` are computed after the loop and patched into `state.analysis` in place
(they depend on the same sentences having been translated). This ordering change was not spelled
out at this level of detail in the architecture doc; resolved it directly as the only way to
satisfy the architecture's stated intent ("progressive... without concurrent dispatch") without a
new design decision — flagging here per `DEVELOPER.md` rather than treating it as silent.

- Files changed: `index.html`.
- Commit: `43b2ccf`
- Verified live: 3-sentence `analyze()` run — DOM inspection after completion showed all 3 rows
  correctly rendered via `data-i` hooks; `translationRowTemplate()` checked directly for all three
  states (loading/error/success) including an XSS payload (`<b>`) in the error row's sentence text,
  confirmed escaped. `retrySentence`/`updateTranslationRow` confirmed defined and reachable.

## Dev Task 4 — Insert Multi-AI router

Per architecture §3b. Ported `providerRegistry`/`registerProvider`/`aiRouter`/`aiRouter_request`,
the `legacy-translation` adapter (wraps `translateSentenceReliable()` unchanged), and the dormant
`gemini` adapter (`getDevApiKey`/`GEMINI_MODEL`, summary-only, requires
`window.SAT_STUDIO_DEV_KEYS.gemini`) from `multi-ai-v2-dev`, inserted immediately before
`translateSentenceReliable()`. Rewired `analyze()`'s dispatch loop and `retrySentence()` to call
`aiRouter.request('translation',...)` instead of the direct call Dev Tasks 2/3 used as an interim
step (kept the app in a working state at every commit, matching the precedent already established
in `multi-ai-v2-dev`'s own Milestone 2 history of adding a function before wiring it in).

- Files changed: `index.html`.
- Commit: `b90362b`
- Verified live: `aiRouter.request('translation',{text:...})` resolves via `legacy-translation`
  with the expected `{success,provider,model,data,latencyMs,fallbackUsed,error}` shape.
  `aiRouter.request('summary',{text:...})` with no dev key set fails closed
  (`success:false`, `latencyMs:0` confirming no network call fired) — Gemini stays dormant.
  `Object.keys(providerRegistry)` === `["legacy-translation","gemini"]` (no unintended providers).
  `grep` for API-key-shaped literals returned nothing.

## Dev Task 5 — Persistence carry-overs

Per architecture §3c / risk 3. Added `getSavedList()` (defensive `JSON.parse`, matches
`CODING_STANDARDS.md`'s localStorage-defensiveness rule and `multi-ai-v2-dev`'s Milestone 1 fix),
replacing the Gold Master's bare `JSON.parse(localStorage.getItem(...)||"[]")` in `saveCurrent()`,
`renderSaved()`, `loadSaved()`, `deleteSaved()` (architecture named `deleteSaved()`/`renderSaved()`
explicitly; included `loadSaved()` too as the same mechanical consequence of introducing the
helper — flagging this one small addition beyond the literal task text).

Added the SAVE-1 guard to `saveCurrent()` (blocks save while any translation is `'loading'`) — Dev
Task 3's progressive translation reopens this exact race, so this guard is required for correctness,
not optional. One guard clause covers both the `localStorage` path and the `auth.userId`-gated
`saveSessionForUser()` IndexedDB path, since the latter is only reached after the guard.

Applied `escapeHtml()` (already defined in the file) to `renderSaved()`'s title/grade/date
interpolation — fixes a live XSS regression in the Gold Master's original `renderSaved()`.

No IndexedDB code touched — `openDB`/`idb*`/`saveSessionForUser`/`openSavedSession`/
`deleteSavedSession` all unmodified, per the architecture's "dual-layer persistence preserved
as-is" decision.

- Files changed: `index.html`.
- Commit: `89184a7`
- Verified live: `saveCurrent()` called mid-`analyze()` (translations still `'loading'`) was
  correctly blocked (0 items persisted); called again after `analyze()` completed, succeeded (1
  item persisted). `renderSaved()` with a saved title of
  `SAVE1 Guard Test <img src=x onerror=alert(1)>` rendered it HTML-escaped, not executed.

## Dev Task 6 — Regression pass

Consolidated live-browser regression sweep (single `analyze()` run against a 3-sentence passage,
`chromeTranslatorState` forced `'unsupported'` to bypass the hang noted in Risk 1):

| Check | Result |
|---|---|
| Application loads, no console errors | Pass |
| Passage analysis (`analyze()`) | Pass |
| Translation (all 3 sentences → real MyMemory translations) | Pass |
| Translation retry (`retrySentence()` on a forced-error row) | Pass — recovered to `status:'success'` |
| Vocabulary tab | Pass (`vocab(a)` rendered, non-empty) |
| Idioms/phrases tab | Pass (`phrases(a)` rendered, non-empty) |
| Grammar tab | Pass (`grammar(a)` rendered, non-empty) |
| SAT quiz tab + `gradeQuiz()` | Pass — rendered and graded without error |
| Saved materials / localStorage (save, blocked-while-loading, escaping) | Pass — see Dev Task 5 |
| Photo/image import present at code level | Pass — `importDocument`/`getDocumentOcrWorker`/`convertHeicForOcr` all present, type `function`, unmodified |
| PDF import present at code level | Pass — `extractPdfPassage`/`getPdfLibrary` present, type `function`, unmodified |
| Responsive/mobile CSS present | Pass — 7 `@media` blocks present (unchanged count from Gold Master) |
| No translation stuck in `'loading'` after analysis | Pass — confirmed `translations.some(t=>t.status==='loading')` is `false` post-`analyze()` |
| Gemini adapter dormant without a key | Pass — see Dev Task 4 |
| No API key committed/hardcoded | Pass — pattern search clean; `getDevApiKey()` reads only `window.SAT_STUDIO_DEV_KEYS`, never a literal |

Examples/ESL/Overview tabs also spot-checked as part of the 8-tab sweep (all rendered, non-empty,
no exceptions) — not individually named in the CEO's checklist but adjacent/coupled per
`TESTING_STANDARDS.md`'s "adjacent features" requirement.

**Not tested (out of reach of this environment):** live PDF/photo extraction end-to-end (would
require fetching `pdf.js`/`tesseract.js`/`heic2any` from `cdn.jsdelivr.net` and a real file), Chrome
on-device Translator's actual translate path (the API exists in this browser but its
`availability()`/`create()` call hung — see Risk 1), and IndexedDB account-flow (`createAccount`/
`loginAccount`/`saveSessionForUser`) — untouched by this milestone's changes, so not exercised
beyond confirming the functions still exist.

## Risks / observations flagged, not fixed (outside the 6 approved tasks' scope)

1. **Chrome Translator can hang with no timeout.** `getChromeTranslator()`'s
   `Translator.availability()`/`create()` call hung indefinitely in this test environment (never
   resolved or rejected). `translateSentenceReliable()` `await`s it with no timeout, so if this
   happens in a real user's browser, the *entire sequential translation loop* would stall on the
   first sentence with no visible progress and no way to recover short of reloading. This is
   pre-existing Gold Master behavior, unchanged by any of the 6 tasks (the architecture explicitly
   said to keep `getChromeTranslator()` untouched) — not fixed here. Recommend QA verify severity
   and the CEO decide whether a timeout wrapper is worth a follow-up milestone.
2. **`saveCurrent()`'s `Date.now()` id can still collide** on rapid double-saves (the Gold Master
   reverted `multi-ai-v2-dev`'s Milestone 1 fix for this). Not named in the approved architecture's
   Task 5 scope, so not fixed — flagging for the same reason as Risk 1.
3. **No "disable analyze button while in flight" guard.** The Gold Master has no equivalent of
   Milestone 1's overlap-guard fix; `analyze()` can be invoked again while already running. Not
   named in the approved architecture, not fixed.
4. These three are all pre-existing Gold Master behavior, not regressions introduced by this
   milestone's changes — listed so QA/Principal Review/CEO can decide whether any warrant a
   follow-up, per `WORKFLOW.md`'s "log and hand upward" rule rather than silently expanding scope.

## Uncommitted tooling

`.claude/launch.json` (a `python3 -m http.server` config for the Browser tool preview) was added to
the working tree to enable live regression testing per the note above. Left uncommitted — it's dev
tooling, not part of the approved architecture's file list. QA may find it useful for its own pass;
otherwise it can be deleted without affecting the milestone.

## Summary

All 6 Developer Tasks complete. Six commits on `multi-ai-v2-latest-dev`:

1. `ae3d3eb` — Docs: PM-Spec + Architecture (CEO-approved)
2. `b14c179` — Dev Task 1: seed from Gold Master
3. `29523bb` — Dev Task 2: translateSentenceReliable() reconciliation
4. `43b2ccf` — Dev Task 3: per-row translation UI
5. `b90362b` — Dev Task 4: Multi-AI router
6. `89184a7` — Dev Task 5: persistence carry-overs

`LATEST_GOLD_MASTER_NEXT.html` confirmed byte-for-byte unchanged
(`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`) and `multi-ai-v2-dev`
confirmed unchanged (`bc204cf48d6c7ab360329bbf2bcd6bcc581a8781`, `index.html` checksum
`b13794ac8a186fc62b7fc05ac32c40fd45df832e6cad5888b25e2f5104a4884c`) throughout.

---

## Handoff — Milestone 3 Development

- **Milestone:** Milestone 3 — Gold Master Adoption + Multi-AI Migration.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/DEVELOPER.md`, `.ai-company/CODING_STANDARDS.md`, `.ai-company/GIT_RULES.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-03/01-PM-SPEC.md` (confirmed
  APPROVED), `02-ARCHITECTURE.md` (confirmed APPROVED).
- **Scope completed:** All 6 Developer Tasks per `02-ARCHITECTURE.md` §5, exactly as scoped. No
  work outside the approved architecture was implemented; three pre-existing Gold Master issues
  noticed during testing were flagged (above), not fixed, since they weren't part of the approved
  task list.
- **Files changed:** `index.html` (all 5 code commits), `LATEST_GOLD_MASTER_NEXT.html` (added,
  never modified after), `docs/milestones/milestone-03/01-PM-SPEC.md` +
  `02-ARCHITECTURE.md` (committed), this file (new).
- **Commits created:** `ae3d3eb`, `b14c179`, `29523bb`, `43b2ccf`, `b90362b`, `89184a7` (all on
  `multi-ai-v2-latest-dev`). None on `multi-ai-v2-dev` or `main`. No push performed.
- **Tests performed:** See per-task sections and the Dev Task 6 regression table above. Live
  in-browser verification substituted for the jsdom convention (Node unavailable in this
  environment) — flagged for QA's awareness/decision.
- **Unresolved risks:** The three items in "Risks / observations flagged, not fixed" above, plus
  the untested-in-this-environment items listed under Dev Task 6 (PDF/OCR live extraction, Chrome
  Translator's actual translate path, IndexedDB account flow — all pre-existing/unmodified code,
  not exercised beyond existence checks).
- **Next agent:** QA Engineer, to independently verify against `01-PM-SPEC.md`'s 10 acceptance
  criteria and check for regressions per `TESTING_STANDARDS.md`.
- **Explicit stop point:** Per the CEO's instruction, this session stops here — before merge or
  release. Awaiting CEO review of these results before any further phase (QA, Principal Review,
  Release) proceeds.
