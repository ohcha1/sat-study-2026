# Milestone 3 — QA Report: Gold Master Adoption + Multi-AI Migration

Independent QA pass. Branch `multi-ai-v2-latest-dev`, tested at HEAD `dae5dce2d0d85e612bb9a9fc06083067d5f2b335`
(confirmed matches expected before testing). No application code modified during this pass —
verified via `git status`/`git rev-parse HEAD` before and after.

**Method:** Live in-browser testing (Node/jsdom unavailable in this environment, same constraint
the Developer noted in `03-IMPLEMENTATION-LOG.md`). Static file served locally, driven through the
Browser tool, exercising real functions/DOM/`localStorage`/`IndexedDB`/`fetch`. Test cases were
independently authored, not reused from the Developer's own test script, per the intent of
independent verification — several (the Chrome-hang re-test, the Lingva fetch-trace, the 429 trace,
the cache-hit counting, the frozen-clock id-collision test, the sharper re-entrancy test) go beyond
what the implementation log describes testing.

## 1. QA Verdict

**PASS WITH CONDITIONS**

The 6 approved Developer Tasks were implemented correctly and faithfully against
`02-ARCHITECTURE.md` — every design decision it specifies was independently confirmed working as
designed. However, one of the milestone's own PM-Spec acceptance criteria (AC #3) fails under a
real, reproducible condition, and a confirmed data-loss bug exists in code this milestone
deliberately left untouched. Neither was introduced by this milestone's changes, and both were
already transparently flagged by the Developer — but per `.ai-company/QA.md`'s explicit instruction
not to downgrade severity to avoid blocking, they are reported at full severity below. See §5 for
what "conditions" means concretely.

## 2. Acceptance criteria results (`01-PM-SPEC.md` §5)

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 | Loads without console errors / no premature network | **PASS** | Clean load, no console errors, confirmed independently. |
| 2 | Passage analysis produces gold-master-engine output | **PASS** | All 8 tabs render non-empty content; `qg*`/`detect*` engines, `buildDeterministicSummary`, etc. all reachable and produce output. |
| 3 | Translation dispatches through `aiRouter`, never a silently stuck loading row | **FAIL (conditional)** | See Risk A below. Independently reproduced `getChromeTranslator()` hanging indefinitely (confirmed twice), and confirmed `translateSentenceReliable()` has **no timeout around the Chrome tier specifically** — it awaits the hang with no bound, producing exactly the "silently stuck loading row" this criterion says must never happen. Reproducible in this session's real Chromium engine, not a jsdom artifact. |
| 4 | Retry re-dispatches through `aiRouter`, succeeds or re-shows error, never hangs | **PASS** (independent of #3) | `retrySentence()` confirmed routes through `aiRouter`; a forced-error row retried and recovered correctly in Developer testing, and my own router-level tests confirm the same code path. Same Chrome-hang caveat as #3 applies if retry is attempted while Chrome tier is stuck. |
| 5 | Gemini adapter never activates without a key, no visible error | **PASS** | Independently confirmed: `aiRouter.request('summary',...)` with no `window.SAT_STUDIO_DEV_KEYS` fails closed, `latencyMs` ≈0-1ms confirming no network call fired. |
| 6 | Saved materials work without data loss or stuck-loading regression (SAVE-1) | **PASS** (for the SAVE-1 case specifically) | Independently confirmed `saveCurrent()` blocks while any translation is `'loading'` (0 items persisted), succeeds once resolved (1 item persisted). Note: a **different**, pre-existing data-loss path exists in this same persistence layer — see Risk B, which AC #6 does not name and which this milestone didn't touch. |
| 7 | PDF/photo import: extraction path present, code/runtime level | **PASS** | `importDocument`/`extractPdfPassage`/`getPdfLibrary`/`getDocumentOcrWorker`/`convertHeicForOcr` all confirmed present as functions, unmodified. Live CDN fetch (`pdf.js`/`tesseract.js`/`heic2any`) not exercised — consistent with the PM-Spec's own acceptance criterion wording ("not requiring a live network fetch... in automated testing"). |
| 8 | No API key committed, real, non-placeholder | **PASS** | Independent pattern search across the full rendered document found no key-shaped literal; `getDevApiKey()` reads only `window.SAT_STUDIO_DEV_KEYS`, confirmed no hardcoded fallback. |
| 9 | Responsive CSS renders at mobile breakpoints without breakage | **PASS** (structural) | 7 `@media` blocks confirmed present, unchanged count from the Gold Master. Visual breakpoint rendering not pixel-verified (matches Developer's own scope — "spot-checked, not a full visual regression suite" per the PM-Spec). |
| 10 | `LATEST_GOLD_MASTER_NEXT.html` checksum unchanged | **PASS** | Independently reconfirmed: `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`. |

**8 of 10 PASS, 1 conditional FAIL (#3), 1 PASS-with-caveat (#6, re: an adjacent unfixed bug it doesn't itself name).**

## 3. Regression findings (adjacent features, per `TESTING_STANDARDS.md`)

- Full 8-tab render sweep (overview/translation/vocab/phrases/grammar/examples/esl/sat): all
  render non-empty, no exceptions — independently re-run, not just re-trusting the Developer's log.
- `IndexedDB` schema creation independently verified: `openDB()` produces all 6 expected object
  stores (`profiles`, `satAttempts`, `savedPassages`, `studySessions`, `vocabularyProgress`,
  `wrongAnswers`) — matches the architecture's documented schema exactly, confirms Dev Task 5's "no
  IndexedDB code touched" claim.
- `providerRegistry` contains exactly `["legacy-translation","gemini"]`; `legacy-translation`
  correctly `canHandle("translation")`; `gemini` correctly `canHandle("summary")` and **not**
  `"translation"` (confirms it can't accidentally intercept translation routing).
- `renderSaved()` escaping independently re-tested with a different payload
  (`<script>window.__xssFired=true</script>` as a saved title) — correctly escaped, script did not
  execute.
- Translation fallback chain independently traced via `fetch` instrumentation: confirmed the exact
  designed sequence (MyMemory attempt → backoff → MyMemory retry → Lingva instance 1 → Lingva
  instance 2 → phrasemap → structured error) fires in the correct order. Real-world Lingva mirrors
  were unreachable from this sandboxed network during testing (both configured instances failed),
  which is an environment/network limitation, not a code defect — the code correctly attempted both
  and correctly fell through to the next tier each time.
- 429 handling independently traced: a simulated 429 sets `mymemoryRateLimited=true` immediately
  (single attempt, no retry-then-fail), and a second call with the flag already set skips MyMemory
  entirely (0 MyMemory fetches) — exactly the "429-stop" behavior the architecture requires be
  preserved.
- Cache behavior independently verified via fetch counting: first call = 1 fetch, identical second
  call = 0 additional fetches (cache hit), `bypassCache:true` = 1 additional fetch (correctly
  bypasses). Only successful results are cached (confirmed indirectly: the error-path test sentence
  above was never found in `translationCache` on a repeat, forcing a full re-attempt each time).
- Sequential (non-concurrent) dispatch independently confirmed by source inspection: `grep` for
  `runWithConcurrency` in the shipped `index.html` finds it only inside comments, never called.

No new regressions in adjacent features were found beyond the two risks below (which are not new —
both are Gold Master behavior that predates this milestone's first commit).

## 4. Risk classification (A/B/C, independently assessed)

### A. Chrome Translator initialization can hang indefinitely — **HIGH**

Independently reproduced twice (once calling `getChromeTranslator()` directly — stuck at
`"initializing"` after 10+ seconds with the promise never settling; once calling
`translateSentenceReliable()` itself — same hang, 8+ seconds with no timeout, confirming there is
**no timeout wrapper around the Chrome tier** in the Dev Task 2 reconciliation, unlike the MyMemory/
Lingva tiers which both use `fetchWithTimeout`). This is not a hypothetical — it happened in a real
Chromium-based engine in this session, twice.

Impact when triggered: the *entire* sequential translation loop stalls on sentence 1 — no error,
no partial progress past that point, no retry possible (retry hits the same hung path), and the
SAVE-1 guard then also permanently blocks saving that analysis. Only a page reload recovers.

This directly violates PM-Spec Acceptance Criterion #3 ("never a silently stuck loading row") when
triggered. Not classified BLOCKER because: (a) it doesn't affect any other tab/feature — vocabulary,
grammar, quiz, save-when-not-mid-translation all remain fully functional; (b) prevalence in
mainstream Chrome desktop (vs. this sandboxed/headless test engine) is unconfirmed — Chrome's
on-device Translator API is comparatively new and its hang behavior here may be specific to
environments without an already-downloaded language pack or without permission to download one;
(c) it is Gold Master behavior inherited unchanged, not introduced by this milestone, and adding a
timeout wrapper is a small, well-scoped, low-risk fix if the CEO authorizes it as a follow-up.

### B. `Date.now()` save ID may collide — **BLOCKER (for release), out of Milestone 3's scope**

Independently reproduced with a frozen clock: two consecutive `saveCurrent()` calls at the same
millisecond produced two saved items with the **identical** id. Confirmed actual impact, not just a
theoretical field collision: calling `deleteSaved(id)` on that id deleted **both** items at once
(`beforeDelete: 2, afterDelete: 0` for a single delete call) — silent, unrecoverable loss of a
study set the user did not choose to delete. This is not a new finding — it's the same defect
`DEVELOPMENT_PLAN.md`'s Milestone 1 already found and fixed on `multi-ai-v2-dev` (empirically
confirmed reliable there too), which the Gold Master's `saveCurrent()` reverted to the unfixed form.

Classified BLOCKER using the project's own severity taxonomy (`TESTING_STANDARDS.md`: "Critical —
... data loss") for the purpose of **release readiness**, per this task's instruction to assess
"unacceptable release risk" independent of milestone scope. It does **not** fail any of Milestone
3's own 10 acceptance criteria (none address save-ID collision), and it is not a regression
introduced by any of the 6 Developer Tasks — Dev Task 5 explicitly, correctly left it unfixed as
outside the approved architecture's scope, and flagged it in the implementation log rather than
hiding it. So: this does not fail Milestone 3's QA gate on its own terms, but it should not be
carried into an actual release without an explicit CEO risk-acceptance or a scoped hot-fix.

### C. Analyze button can be triggered again while analysis is running — **MEDIUM**

Independently reproduced a race: starting `analyze()` on passage A, then immediately starting it
again on passage B before A resolves. Result: no exception, no crash — the second call's
`state.analysis` assignment "wins," and the first call's in-flight translation results are silently
computed but discarded (never displayed, since `state.analysis` no longer references the first
call's array) — a reasonable-looking "last request wins" outcome in the common case. A narrower
failure mode exists: if the user has the Translation tab open while this race occurs *and* the two
passages have different sentence counts, the first call's `updateTranslationRow(i)` can index past
the second call's (now-current) shorter `translations` array, throwing inside that promise chain
(an unhandled rejection, visible only in the console — does not crash the page or corrupt saved
data). Classified Medium: a real correctness gap under an uncommon combination of conditions (rapid
re-trigger *and* tab-open *and* differing sentence counts), not a crash, not data loss, not a
security issue, and not something a user is likely to hit without deliberately trying to.

## 5. Release blockers

- **Not blocking Milestone 3's own completion** (the 6 approved Developer Tasks, verified against
  `02-ARCHITECTURE.md`): none. All 6 tasks are correctly implemented.
- **Blocking an actual Release Manager push of this branch**, pending CEO decision:
  - Risk B (data loss) — recommend either a scoped hot-fix (restore the `while(list.some(...))
    id++` uniqueness loop `multi-ai-v2-dev` already had) before release, or an explicit, written
    CEO risk-acceptance if shipping as-is.
  - Risk A (AC #3 failure under a reproducible condition) — recommend either a scoped hot-fix (wrap
    `getChromeTranslator()`'s call with the same `fetchWithTimeout`-style pattern already used for
    MyMemory/Lingva) or an explicit, written CEO risk-acceptance.
- Risk C is Medium and does not, on its own, block release; recommend logging it for a future
  milestone per `DEFINITION_OF_DONE.md`'s "Medium/Low findings may be logged and deferred."

## 6. Files modified by this QA session

**None.** `git status`/`git rev-parse HEAD` confirmed identical before and after testing
(`dae5dce2d0d85e612bb9a9fc06083067d5f2b335`, working tree clean apart from pre-existing untracked
`.DS_Store` and the Developer's uncommitted `.claude/launch.json` test-server config, neither
touched this session). This report file is the only addition.

## 7. Recommended next action

Not a clean pass-through to Principal Review as-is. Per `.ai-company/QA.md`, since there are no
**unresolved Critical/High defects that this milestone's own scope requires fixing** (Risks A/B are
both pre-existing, out-of-scope, and already transparently flagged by the Developer), this could
proceed to Principal Review on the 6 Developer Tasks' own merits — but recommend the CEO first
decide, given AC #3's conditional failure is a criterion *this milestone itself* set:
1. Authorize two small, scoped hot-fix Developer Tasks (Chrome-tier timeout; save-ID uniqueness)
   before Principal Review, treating them as part of Milestone 3, or
2. Explicitly risk-accept both in writing (per `WORKFLOW.md` Rule 1's stage-waiver mechanism) and
   proceed to Principal Review as-is, with these two items logged for an immediate follow-up
   milestone.

---

## Handoff — Milestone 3 QA

- **Milestone:** Milestone 3 — Gold Master Adoption + Multi-AI Migration.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/QA.md`, `.ai-company/TESTING_STANDARDS.md`, `.ai-company/DEFINITION_OF_DONE.md`,
  `docs/milestones/milestone-03/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`.
- **Scope completed:** Independent verification of all 10 PM-Spec acceptance criteria, a regression
  sweep of adjacent features, and independent risk assessment of the three items (A/B/C) flagged in
  the implementation log. All testing live in-browser (Node/jsdom unavailable); test cases
  independently authored, several going beyond the Developer's own verification.
- **Files changed:** `docs/milestones/milestone-03/04-QA-REPORT.md` (new, this file). No
  application code touched — confirmed via `git status`/`git rev-parse HEAD` before and after.
- **Commits created:** None this session (report not yet committed — pending CEO/Principal Review
  per this repo's established pattern of committing docs after each gate's input is incorporated).
- **Tests performed:** See §2/§3 tables above in full.
- **Unresolved risks:** Risk A (HIGH) and Risk B (BLOCKER-for-release) per §4 — both pre-existing
  Gold Master behavior, not introduced by this milestone, both already flagged by the Developer.
  Risk C (MEDIUM) — real but narrow, does not block release on its own.
- **Next agent:** CEO — to decide between the two paths in §7 (authorize hot-fixes now vs. explicit
  risk-acceptance + follow-up milestone) before Principal Review proceeds.
- **Explicit stop point:** No Critical/High defect exists that falls within Milestone 3's own
  approved scope to fix, so this does not hand back to the Senior Developer automatically per
  `.ai-company/QA.md`'s loop rule — it hands to the CEO for the scope decision in §7, since that
  decision determines whether Development reopens (hot-fix path) or Principal Review proceeds
  directly (risk-acceptance path).
