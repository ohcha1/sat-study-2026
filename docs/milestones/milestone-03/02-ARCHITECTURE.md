# Milestone 3 — Architecture Design: Gold Master Adoption + Multi-AI Migration

**Status: DRAFT — awaiting CEO approval.**

Scoped to `docs/milestones/milestone-03/01-PM-SPEC.md` (APPROVED 2026-08-12) and its CEO
resolutions in §6a. This document resolves §6b's six remaining questions and nothing beyond that —
no new scope, no UI redesign, no new AI providers.

## 1. Approach summary

Adopt `LATEST_GOLD_MASTER_NEXT.html` (SHA-256 `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`,
untouched) as the literal starting point for a new `index.html`, then make the smallest possible
set of additive changes to re-establish the Multi-AI V2 router and Milestone 2's translation
reliability behavior *on top of* the gold master's own structure — not by porting old code
verbatim, per the CEO's explicit permission to adapt.

The central technical finding driving this design: **the gold master's own `translateSentence()`
already improves on Milestone 2 in one respect (an on-device Chrome Translator primary tier, tried
before any network call) and deliberately regresses in another (no Lingva fallback, no retry/
backoff, no per-sentence structured status, no progressive/error/retry UI) — and its `analyze()`
dispatches translations in a strict sequential loop specifically *because* concurrent dispatch
would break its own MyMemory 429-stop logic** (this is stated directly in the gold master's source
comment at the loop). Milestone 2's `runWithConcurrency`/`CONCURRENCY_LIMIT` bounded-pool dispatch
is therefore **not** carried over — reusing it here would reintroduce exactly the race the gold
master's own authors already fixed. Everything else about Milestone 2's reliability work (Lingva
fallback, retry/backoff, timeout-wrapped fetches, session cache, structured `{ko,status,source}`
result, per-row loading/error/retry UI, the SAVE-1 guard) is preserved by layering it around the
gold master's existing sequential loop and its two translation tiers, not by replacing them.

Rejected alternative: keep `runWithConcurrency` and just swap in the gold master's translation tiers
underneath it. Rejected because it silently reintroduces the exact bug class the gold master's own
sequential-dispatch comment warns about (a burst of concurrent requests can each observe
`mymemoryRateLimited===false` before the first 429 response sets it, so several sentences would
still fire against an already-rate-limited endpoint) — this is a regression, not a neutral choice,
and the CEO's instruction is that gold-master behavior wins where the two conflict.

## 2. Affected files/modules

- **New:** `index.html` on the new branch (see §6b.3), built from `LATEST_GOLD_MASTER_NEXT.html`.
  This is the only application file touched.
- **Unchanged:** `LATEST_GOLD_MASTER_NEXT.html` (protected reference copy — never edited),
  `multi-ai-v2-dev` branch and its `index.html` (historical backup), all other milestone folders.
- **New:** this document, and `03-IMPLEMENTATION-LOG.md` (created by the Senior Developer).
- Not touched: `docs/MULTI_AI_ARCHITECTURE_V2.md`. It remains historically accurate for the
  `multi-ai-v2-dev` line; this document supersedes it for the new branch's translation/router
  design but rewriting it is out of scope (matches the PM-Spec's instruction not to rewrite
  unrelated historical documents).

## 3. Data flow / interfaces

### 3a. Translation reconciliation (§6b.1)

A single function, kept under the existing name **`translateSentenceReliable(s, {bypassCache}={})`**
(the name every consumer already depends on by contract, not by implementation), replaces the gold
master's `translateSentence()`. New tier order:

```
translateSentenceReliable(s):
  1. cache check (translationCache, unchanged Map, unless bypassCache)   — gold master + M2, same
  2. Chrome on-device Translator (getChromeTranslator/-State, unchanged) — gold master's addition, tried first
       success → {ko, status:'success', source:'chrome'}
  3. MyMemory, fetchWithTimeout-wrapped                                  — merged behavior:
       - if mymemoryRateLimited is already true: skip straight to step 4 (gold master's 429-stop, preserved)
       - attempt 1 → on non-429 failure: RETRY_BACKOFF_MS pause → attempt 2 (M2's addition)
       - a 429 on *either* attempt: set mymemoryRateLimited=true, stop retrying, fall through
       success → {ko, status:'success', source:'mymemory'}
  4. Lingva (translateLingva, LINGVA_INSTANCES, fetchWithTimeout-wrapped) — M2's addition, new to gold master
       success → {ko, status:'success', source:'lingva'}
  5. simpleTranslate() phrase-map                                        — both, unchanged
       matched → {ko, status:'success', source:'phrasemap'}
  6. → {ko:null, status:'error', source:null}                            — M2's addition
  Only status:'success' results are written to translationCache (M2's existing rule, unchanged).
```

`providerUsageThisRun` (gold master's `{chrome,mymemory,local}` counters, logged once per
`analyze()` run) gains a `lingva` key; `local` continues to mean "phrasemap or error", matching its
existing meaning.

`analyze()`'s dispatch stays a **sequential** `for` loop (gold master's structure, not
`runWithConcurrency`) over `sents`, but each iteration now:
- calls `aiRouter.request('translation', {text:s})` instead of `translateSentence(s)` directly,
- writes the structured result into `translations[i]` (`{en, ko, status, source}` — new fields
  `status`/`source` add to the gold master's existing `{en,ko}` shape; `ko` stays `string|null`),
- calls `updateTranslationRow(i)` (ported from `multi-ai-v2-dev`, adapted to patch a row inside the
  gold master's `#result`/tab DOM structure) so each sentence's row updates as it resolves — this
  reproduces Milestone 2's "visible per-sentence progress" outcome *without* concurrent dispatch,
  since sequential-with-progressive-repaint is sufficient for that requirement and doesn't touch
  the 429-stop invariant.

`translation(a)` (gold master's flat `en`/`ko` renderer) is replaced by
`translationRowTemplate(x,i)` (ported from `multi-ai-v2-dev`, restyled to the gold master's
existing card/CSS conventions rather than importing `multi-ai-v2-dev`'s CSS) — rendering
loading/success/error states and the retry control per row, keyed by the same `data-i` hook
`updateTranslationRow` expects.

`retrySentence(i)` is ported as-is in logic (routes through `aiRouter.request('translation', ...,
{bypassCache:true})`, patches one row) — new to the gold master's UI (it has no retry affordance
today), added as the row's error-state action per the CEO's "retain useful M2 reliability
improvements" instruction.

**`saveCurrent()`'s SAVE-1 guard** (`if (state.analysis.translations.some(t=>t.status==='loading'))
{ toast(...); return }`) is added to the gold master's `saveCurrent()`, ahead of both its
`localStorage` write and its `if(auth.userId) saveSessionForUser()` call — a single guard clause
covers both persistence paths since `saveSessionForUser()` is only reached after it. Without this,
progressive per-row translation (newly reintroduced here) reopens the exact SAVE-1 race Milestone 2
fixed, since `state.analysis` is now reachable while `translations[i].status==='loading'`.

### 3b. Multi-AI integration point (§6b.2)

The `providerRegistry`/`registerProvider`/`aiRouter`/`aiRouter_request` scaffold and the two
existing adapters (`legacy-translation`, dormant `gemini`) move into the gold master **verbatim in
logic**, inserted as one clearly delimited block immediately before the new
`translateSentenceReliable()` (i.e., where the gold master's own `translateSentence()` currently
sits, ~line 1889), block-commented citing this document by section (matching the existing
convention already established in both the gold master and `multi-ai-v2-dev`). Only the
`legacy-translation` adapter's `request()` body changes — it wraps the new
`translateSentenceReliable()` (unchanged wrapping pattern, same function name as today). The
`gemini` adapter, `getDevApiKey()`, `GEMINI_MODEL`, and the dev-key mechanism move unmodified;
`canHandle("summary")` stays false-reachable from any real call site — still dormant, still
requires `window.SAT_STUDIO_DEV_KEYS.gemini` to do anything, still not wired into `overview()`,
matching the CEO's "no additional provider, Gemini stays dormant" instruction.

### 3c. Persistence (§6b's persistence items)

No structural change. The gold master's dual-layer persistence is preserved exactly as it exists
today:
- Anonymous/base layer: `localStorage["satStudioSaved"]` via `saveCurrent()`/`getSavedList()`*/
  `renderSaved()`/`deleteSaved()`/`loadSaved()` (*`getSavedList()`'s defensive-parse helper is
  ported from `multi-ai-v2-dev` to replace the gold master's bare `JSON.parse(localStorage
  .getItem(...)||"[]")` calls in `deleteSaved()`/`renderSaved()`, per `CODING_STANDARDS.md`'s
  "validate/handle localStorage reads defensively" rule — a direct carry-forward of an already-
  established Milestone-1 fix, not new scope).
- Account-scoped layer: IndexedDB via `openDB()`'s six object stores (`profiles`, `satAttempts`,
  `wrongAnswers`, `studySessions`, `savedPassages`, `vocabularyProgress`), `saveSessionForUser()`,
  `openSavedSession()`, `deleteSavedSession()`, gated behind `auth.userId` — used only for logged-in
  users, additive to the base layer.

"IndexedDB is authoritative" (CEO, §6a.6) is interpreted as: **this dual-layer structure is
authoritative and is not collapsed into a single layer in either direction** — IndexedDB is not
removed/downgraded to `localStorage`-only, and `localStorage`'s base layer (which the gold master
itself still relies on for anonymous use) is not force-migrated into IndexedDB, since the gold
master's own design already treats them as separate, intentional tiers. No migration of
`multi-ai-v2-dev`'s existing dev-only `localStorage` save data into the new branch is performed
(§6a.6 — treated as dev-only, not needed).

## 4. Risks and tradeoffs

**HIGH**
1. Reintroducing concurrent translation dispatch (e.g., a future session "restoring"
   `runWithConcurrency` for perceived speed) would silently break the MyMemory 429-stop invariant.
   *Mitigation:* the sequential loop and the reason for it are documented in-line at the dispatch
   site, citing this section, per existing convention.
2. `translationRowTemplate()`/`retrySentence()`/the SAVE-1 guard all depend on the exact
   `{ko,status,source}` contract. Any tier implementation drifting from that shape breaks all three
   silently. *Mitigation:* single return point per tier in `translateSentenceReliable()`, verified
   by throwaway jsdom tests per §6 below before/after.

**MEDIUM**
3. `renderSaved()`'s legacy `localStorage` path interpolates `x.title`/other fields into `innerHTML`
   without `escapeHtml()` in the gold master today — a live regression against the documented
   Milestone-1 XSS-fix pattern `CODING_STANDARDS.md` explicitly calls out. *Mitigation:* apply the
   gold master's own already-defined `escapeHtml()` (it exists, at line 3127, just isn't called
   here) to this render call as part of this migration — a narrow, established-pattern fix, not new
   scope, consistent with `CODING_STANDARDS.md`'s default-to-defect-risk rule for unescaped
   `innerHTML`.
4. Adding a retry affordance/error row to the gold master's translation UI is a small, additive UI
   change (a button + two new row states) inside an otherwise-unmodified card. The CEO's "preserve
   current UI unless technically required for migration" instruction is read as permitting exactly
   this, since it's required to give `retrySentence()`/error status somewhere to render — but
   nothing else in the translation card's visual design changes.
5. `providerUsageThisRun`'s new `lingva` counter and the `source` field's new `'lingva'`/`'chrome'`
   values are additive; nothing reads the old shape positionally, so no downstream breakage
   expected, but QA should confirm no code elsewhere assumes only `{chrome,mymemory,local}` keys.

**LOW (documented per CEO's CDN-dependency instruction, §6a.4 — not mitigated, not removed)**
6. **Security:** `pdf.js`/`tesseract.js`/`heic2any` load from `cdn.jsdelivr.net` at runtime with no
   Subresource Integrity hash today; a compromised CDN or MITM on a non-HTTPS context could serve
   malicious JS. Kept as-is per CEO instruction; noted as a candidate for a future SRI-hardening
   follow-up, not this milestone's scope.
7. **Availability:** if `cdn.jsdelivr.net` is unreachable, PDF/photo import fails; there is no local
   fallback bundled. Existing gold master behavior — `getPdfLibrary()`/`getDocumentOcrWorker()`
   reject with a Korean-language error surfaced via `setDocumentImportStatus()`, already a
   reasonably graceful failure, unchanged here.
8. **Offline:** these three features do not work under `file://` with no network or on a fully
   offline machine (first use always requires a fetch to warm the CDN scripts); already true of the
   gold master, unchanged.
9. **Privacy:** student-authored passage photos/PDFs are processed client-side by these libraries
   (no upload of the *file itself* to a third party — only the initial library `.js`/`.wasm` fetch
   touches the CDN), which is consistent with `docs/MULTI_AI_ARCHITECTURE_V2.md` §9's existing
   privacy stance for the app's other features. Unchanged by this migration.

## 5. Implementation plan

Proposed as separate Developer Tasks inside this one milestone (CEO-authorized in §6a.3), each
independently jsdom-testable and independently committable:

1. **Branch setup.** Create `multi-ai-v2-latest-dev` from `multi-ai-v2-dev` HEAD
   `bc204cf48d6c7ab360329bbf2bcd6bcc581a8781` (§6b.3 — CEO's suggested name, no objection found;
   Architect does not see a reason to deviate). Copy `LATEST_GOLD_MASTER_NEXT.html`'s content into
   `index.html` on that branch (the gold master file itself stays byte-for-byte unchanged in the
   repo root throughout). Verify `multi-ai-v2-dev` is untouched (checksum/diff its `index.html`
   before and after).
2. **Reconcile translation.** Rewrite the gold master's `translateSentence()` into
   `translateSentenceReliable()` per §3a's tier order; add `LINGVA_INSTANCES`,
   `RETRY_BACKOFF_MS`/`REQUEST_TIMEOUT_MS` (gold master doesn't define these — port from
   `multi-ai-v2-dev`), `fetchWithTimeout()`, `translateLingva()`. Keep gold master's
   `getChromeTranslator()`/`chromeTranslatorState` and `mymemoryRateLimited` untouched.
3. **Port per-row UI.** Add `translationRowTemplate()` (restyled to gold master CSS),
   `updateTranslationRow()`, `retrySentence()`; replace `translation(a)`; update `analyze()`'s
   dispatch loop to stay sequential but call the router and repaint per row (§3a).
4. **Insert the Multi-AI router.** Add the `providerRegistry`/`aiRouter`/`legacy-translation`/
   dormant `gemini` block (§3b) immediately before `translateSentenceReliable()`. Route `analyze()`
   and `retrySentence()` through `aiRouter.request('translation', ...)`.
5. **Persistence carry-overs.** Add the SAVE-1 guard to `saveCurrent()`; port `getSavedList()` and
   apply `escapeHtml()` in `renderSaved()`'s legacy-path interpolation (§3c, §4 risk 3). No
   IndexedDB code changes.
6. **Regression pass + docs.** Run the checks in §6 below; write `03-IMPLEMENTATION-LOG.md`
   recording every commit; one commit per task above, matching `GIT_RULES.md`'s "one logical change
   per commit."

## 6. Test strategy pointer

Per `.ai-company/TESTING_STANDARDS.md`'s existing throwaway-jsdom convention (no new test
framework, matching the PM-Spec's instruction):
- Reuse Milestone 2's existing jsdom-script pattern to assert `translateSentenceReliable()`'s
  `{ko,status,source}` shape for: cache hit, Chrome-translator success, MyMemory success, MyMemory
  429 (verify `mymemoryRateLimited` short-circuits subsequent calls), MyMemory non-429 failure →
  backoff → retry success, Lingva success, total failure → `status:'error'`.
  wire-through of the SAVE-1 guard blocking `saveCurrent()` while any row is `'loading'`.
- One jsdom pass confirming `analyze()`'s dispatch is still strictly sequential (mock `fetch` with
  a counter/order assertion) — this is the single highest-risk regression per §4 risk 1.
- Spot-check `renderSaved()` escaping with a title containing `<img onerror=...>`, matching
  Milestone 1's original XSS regression test pattern.
- Manual/code-level check (not live-network, per PM-Spec acceptance criterion 7) that
  `importDocument`/`getPdfLibrary`/`getDocumentOcrWorker`/`convertHeicForOcr` are present,
  unmodified, and still reachable from the UI.
- Full QA pass against `01-PM-SPEC.md`'s ten acceptance criteria is QA's responsibility next, not
  re-run here.

## 7. Status

**DRAFT.** Awaiting CEO approval before Development begins.

---

## Handoff — Milestone 3 Architecture

- **Milestone:** Milestone 3 — Gold Master Adoption + Multi-AI Migration.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/ARCHITECT.md`, `.ai-company/CODING_STANDARDS.md`,
  `docs/milestones/milestone-03/01-PM-SPEC.md` (confirmed APPROVED before starting),
  `docs/MULTI_AI_ARCHITECTURE_V2.md` (translation/router sections). Targeted code reads (not full
  files): gold master's `translateSentence()`/`getChromeTranslator()`/`analyze()`/`translation(a)`/
  `saveCurrent()`/`renderSaved()`/`deleteSaved()`/`openDB()`/`escapeHtml()`; current dev's
  `providerRegistry`/`aiRouter`/adapters/`translateSentenceReliable()`/`translateLingva()`/
  `runWithConcurrency()`/`translationRowTemplate()`/`retrySentence()`/`saveCurrent()`.
- **Scope completed:** Resolved all six §6b questions (translation reconciliation, Multi-AI
  integration point, persistence, CDN-dependency documentation, branch name/start point, Developer
  Task breakdown). Wrote `02-ARCHITECTURE.md`. Found and scoped one additional narrow risk (missing
  `escapeHtml()` in the gold master's `renderSaved()` legacy path) directly traceable to an existing
  `CODING_STANDARDS.md` rule — included in the plan as risk 3 / task 5, not treated as new scope.
- **Files changed:** `docs/milestones/milestone-03/02-ARCHITECTURE.md` (new). No application code
  touched, no branch created, no commit made — per `.ai-company/ARCHITECT.md`, this phase designs,
  it does not implement.
- **Commits created:** None this session.
- **Tests performed:** N/A (Architecture phase). Test *strategy* specified in §6 for the Developer/
  QA phases to execute.
- **Unresolved risks:** §4's nine items, all documented with mitigation or explicit
  CEO-instructed non-mitigation (CDN items 6–9). None block Development; all are things Development/
  QA must specifically verify against, not open design questions.
- **Next agent:** CEO, for the architecture approval gate; then Senior Developer.
- **Explicit stop point:** Awaiting CEO approval (or revision request) on this
  `02-ARCHITECTURE.md`. No Development work has been performed.
