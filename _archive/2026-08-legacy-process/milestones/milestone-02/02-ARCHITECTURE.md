# 02-ARCHITECTURE.md — Milestone 2: Translation Reliability

**Status: APPROVED**

## CEO Approval

- **Decision:** Approved.
- **Recorded:** 2026-08-06.
- **CEO statement (verbatim):** "CEO APPROVAL. The Milestone 2 architecture is approved. Now
  switch roles and act only as the Senior Developer for Milestone 2. ... Implement Milestone 2
  exactly as approved."
- **Effect:** Per `.ai-company/WORKFLOW.md`, Development may begin against this document exactly as
  written in §0–§11 below. No section of this architecture was altered to record this approval.

Prepared by: Software Architect role, against `01-PM-SPEC.md` (Status: APPROVED, CEO approval
recorded 2026-08-06). No production code had been written or modified as part of preparing this
document; `index.html` was read only.

---

## 0. Approved scope traceability

Every item below traces to a specific line in `01-PM-SPEC.md`. Nothing here introduces a
requirement the spec doesn't already state.

| PM spec item | Architecture section that addresses it |
|---|---|
| In-scope 1 (labeled error/retry state, replacing the silent placeholder) | §3 Retry/timeout, §5 Loading/progress/error states |
| In-scope 2 (batching/throttling) | §2 Concurrency and throttling policy |
| In-scope 3 (session-level cache) | §4 Session cache design |
| In-scope 4 (evaluate + document a fallback provider) | §6 Fallback-provider selection rules |
| In-scope 5 (per-sentence visible loading) | §5 Loading/progress/error states |
| AC1 (distinct failure state, never a canned string presented as real) | §3, §5, §9 |
| AC2 (bounded concurrency, value is Architecture's to set) | §2, §9 |
| AC3 (no re-request of already-succeeded sentences in-session) | §4, §9 |
| AC4 (written evaluation + explicit recommendation) | §6, §9 |
| AC5 (per-sentence loading indicator) | §5, §9 |
| Out-of-scope: dictionary/vocab, general UI redesign, grammar/SAT generation, switching *primary* provider | Not touched anywhere below — confirmed against the file-by-file plan in §8, which lists only translation-path functions |

Source code actually read before writing this document (not assumed from memory): `index.html`
lines 306–378 (`state`, tab wiring, `splitSentences`, `words`, `simpleTranslate`,
`translateSentence`, `fallbackKoreanGloss`, `levelFor`, `analyze`), lines 423–444 (`render`,
`overview`, `translation`), lines 473 (`escapeHtml`), and lines 674–725 (`getSavedList`,
`saveCurrent`, `loadSaved`, `deleteSaved`, `toast`). The file is 728 lines total; these are the
only regions with translation-related logic, located via `grep` on `translate`, `simpleTranslate`,
`state.analysis`, and `localStorage`.

---

## 1. Approach summary

Keep everything in `index.html` — no new files, no build step, no dependency additions. This
repository's `CODING_STANDARDS.md` requires matching the existing single-file convention unless an
approved architecture calls for otherwise, and nothing in this milestone's scope justifies
splitting the app apart (that's explicitly Milestone 8's job).

The core defect (spec item 1) is that `simpleTranslate()` returns a Korean *string* that is
sometimes a real (if crude) phrase-substitution and sometimes a placeholder sentence saying
translation failed — and the caller (`translation()`) renders both identically as `x.ko`, with no
way to tell them apart. The fix is structural, not cosmetic: replace string-sniffing with an
explicit status field that flows from the network call all the way to the rendered row. Everything
else in this milestone (throttling, caching, fallback provider, per-sentence loading) hangs off
that same status-carrying data shape.

**Rejected alternative:** wrapping the existing `Promise.all(sents.map(translateSentence))` call
with a generic retry-the-whole-batch-on-any-failure loop. Rejected because it conflicts with the
caching goal (item 3) — a whole-passage retry would re-request sentences that already succeeded —
and gives no per-sentence signal for item 5's loading requirement. Per-sentence handling (Assumption
A3 in the PM spec) is the only approach that satisfies items 1, 3, and 5 simultaneously without
contradicting one another.

**Rejected alternative:** a queue/library-based concurrency manager (e.g., pulling in a small npm
package via CDN for a promise pool). Rejected because the concurrency need here is simple (N
bounded workers pulling from a shared cursor) and `CODING_STANDARDS.md` prefers the smallest change
using existing conventions; a ~15-line helper function is more auditable than a new external
script tag for a single-file app that currently has zero runtime dependencies.

---

## 2. Concurrency and throttling policy

**Finding (verified against MyMemory's own published docs, not assumed):** MyMemory's usage-limits
page states quota in *daily characters* (5,000 chars/day anonymous; 50,000/day if a contact email
is supplied via the `de=` parameter) and separately says it "keeps track of call rate and enforces
limits when necessary" for its `keygen` endpoint — but publishes **no numeric per-second or
per-burst request limit** for the `get` (translation) endpoint used here. This confirms the PM
spec's resolution of open question 1: there is no published number to design against, so the
concurrency cap is a defensive Architecture parameter, not a derived constant.

**Decision:** bounded-concurrency worker pool, cap = **3 simultaneous in-flight requests**, plus a
~120ms stagger between the pool's initial dispatches (avoids opening N connections in the same
event-loop tick even when the pool "starts" simultaneously).

Rationale for 3, specifically: low enough that a 40-sentence passage never opens more than 3
sockets to MyMemory/Lingva at once (eliminates the burst pattern item 2 exists to prevent), high
enough that total wall-clock time for a long passage stays reasonable given the new progressive
per-sentence rendering (§5) means the user sees results streaming in rather than waiting on one
blocking spinner. Expose it as a single named constant (`CONCURRENCY_LIMIT`) so it's a one-line
tuning change if real-world use shows it's too conservative or too aggressive — not a hardcoded
magic number scattered through the code.

**Implementation shape** (illustrative, not full implementation — per `.ai-company/ARCHITECT.md`):

```
async function runWithConcurrency(items, limit, worker) {
  let cursor = 0;
  const results = new Array(items.length);
  async function run() {
    while (cursor < items.length) {
      const i = cursor++;
      results[i] = await worker(items[i], i);
    }
  }
  await Promise.all(Array.from({length: Math.min(limit, items.length)}, run));
  return results;
}
```

This replaces `Promise.all(sents.map(s=>translateSentence(s)))` in `analyze()` (line 365) as the
dispatch mechanism, but each `worker` call now also drives the progressive per-row UI update
described in §5, not just a same-shape return value.

---

## 3. Retry and timeout behavior

Two independent concerns, both currently absent from `translateSentence()`:

**Timeout.** The current `fetch(url)` call has no timeout — a hung connection to MyMemory blocks
that sentence's pool slot indefinitely. Wrap every provider call in `AbortController` with an
**8000ms** timeout. 8s is chosen as generous enough for slow/mobile connections but short enough
that one stuck request doesn't meaningfully stall a pool slot that other sentences are waiting on.

**Retry.** Two layers, kept deliberately separate so they compose without duplicating logic:

1. **Automatic, same-provider, single retry.** If the primary provider (MyMemory) throws or times
   out on the first attempt, wait a fixed **800ms** backoff and try once more before falling
   through to the fallback provider. Covers ordinary transient blips without escalating to a
   different third party for every minor hiccup.
2. **Manual, per-sentence, user-initiated retry.** Each row that ends in `status: 'error'` renders
   a "다시 시도" (retry) button (per PM spec Assumption A3: retry granularity is per-sentence, not
   whole-passage). Clicking it re-runs the full provider chain (§6) for **only that sentence**,
   bypassing the cache (since a cached result wouldn't exist for a failed sentence — see §4) and
   patches just that row via `updateTranslationRow(i)` (§5), not a full re-render.

**Explicitly not doing:** automatic retry of the entire passage on any single sentence's failure.
This was the "rejected alternative" in §1 — it would undermine both the caching goal and the
per-sentence loading model.

---

## 4. Session cache design

`const translationCache = new Map()`, declared once at module scope next to the existing
`const state={...}` (line 307), so it shares the same page-load lifetime as `state` — reads,
writes, and the entire cache are lost on reload/navigation, matching the PM spec's Assumption A1
("session" = current page load, not `localStorage`, not cross-tab).

- **Key:** the exact sentence text as produced by `splitSentences()` (trimmed, case-sensitive) —
  matches AC3's "exact text" wording precisely, so a sentence that reappears verbatim in a second
  `analyze()` call within the same session is a cache hit; anything even slightly different (e.g.,
  user edits the passage) is correctly treated as a new sentence.
- **Value:** `{ ko, status: 'success', source: 'mymemory' | 'lingva' | 'phrasemap' }`.
- **Only successful results are cached.** Failed sentences are never written to the cache. This is
  a deliberate tradeoff (see §7 Risks): it means a sentence that fails twice in the same session
  will re-attempt the full provider chain (and re-spend MyMemory quota) on every subsequent
  `analyze()` or manual retry, rather than short-circuiting to a remembered failure. The
  alternative — caching failures too — was rejected because it risks showing a stale failure after
  the underlying network/provider issue has already recovered, which would contradict AC1's
  "honest" bar in a different direction (falsely persistent failure instead of falsely-successful
  placeholder).
- **Interaction with `analyze()` re-runs:** because the cache is keyed by sentence text and not by
  passage identity, re-analyzing the *same* passage is a full-hit; analyzing a *different* passage
  that happens to share a sentence (e.g., re-using the sample passage) also benefits, which is a
  reasonable and harmless side effect of the "session" framing, not a scope expansion.
- **Interaction with saved study sets:** `saveCurrent()` (line 687) already spreads
  `state.analysis` (which includes `translations`) into the saved item and writes it to
  `localStorage`. No change needed to that mechanism — the cache itself is never persisted, only
  the already-resolved `translations` array (now carrying `status`/`source` fields per sentence, per
  §8) is, exactly as `en`/`ko` are today. `loadSaved()` (line 704) must treat `status`/`source` as
  optional on older saved sets (see §8, defensive-read note) so previously-saved data — which
  predates this milestone and won't have those fields — doesn't break.

---

## 5. Visible loading, progress, and error states

Per-sentence row, three states (replacing today's single always-rendered `x.ko` string):

- **`loading`** — shown the instant `analyze()` builds the initial `translations` array (before any
  network call resolves). Rendered as a distinct skeleton/placeholder row with a Korean "번역
  중…" label, visually distinguishable from both a completed and a failed row.
- **`success`** — today's existing row shape (`en` / `ko` pair), unchanged visually.
- **`error`** — a new, visually distinct row (e.g., a warning-colored variant of the existing `card
  c-blue`/`c-pink`/etc. convention already used elsewhere in `index.html`) with an explicit Korean
  failure message and a "다시 시도" retry button. Never falls back to rendering the old placeholder
  string as if it were content.

**Progressive rendering, not one blocking wait:** `analyze()` currently does not call `render()`
for the translation tab until every sentence has resolved (`await Promise.all(...)` blocks the
whole function before `state.analysis` is even assigned). This is the direct cause of item 5's
gap — there is no partial-progress UI today, only a global `#status` text line. The fix:
`state.analysis` is assigned as soon as `translations` exists in its `loading` shape (before any
network call starts), `render()` is called once immediately so the loading skeleton is visible, and
then `runWithConcurrency` (§2) drives the pool — each worker, on settling, mutates that same
sentence's entry in place and calls a new **`updateTranslationRow(i)`** function that patches only
that row's DOM node (if the translation tab is currently active) rather than re-invoking the full
`translation(a)` template string and replacing `#result.innerHTML` wholesale. This avoids
re-render flicker/cost scaling with passage length, and avoids destroying focus/scroll position if
the user is mid-interaction with the tab (e.g., about to click a retry button on another row) while
other sentences are still resolving.

**Global status line (`#status`, line 364):** updated to a running counter during translation
(e.g., "한글 번역 진행 중… (3/12 완료)") instead of one static message, then a final summary that
now also surfaces failures instead of silently omitting them: "분석 완료 · 12문장 · 4개 핵심어휘 ·
번역 1건 실패" when applicable. The rest of the analysis (vocab/phrases/questions) is unaffected by
translation success/failure, consistent with today's behavior — those tabs don't depend on
translation completing.

**Explicitly out of scope for this milestone** (flagged so it isn't accidentally implemented):
surfacing a translation-failure indicator on the Overview tab's stat cards, or any other tab.
Milestone 2's spec scopes this to the translation tab and the shared status line only; broader
error/empty-state UI polish across tabs is Milestone 4's stated scope.

---

## 6. Fallback-provider selection rules (written evaluation — satisfies AC4)

**Constraint that shapes every option below:** this is currently a fully static, client-only page
(`index.html` opened directly, no backend, no server-side secret storage — that infrastructure is
Milestone 8's stated scope, not yet built). Per `.ai-company/GIT_RULES.md` and
`CODING_STANDARDS.md`, no API key or credential may be committed into source. This rules out any
candidate that requires a private key to function from the browser today.

**Candidate 1 — LibreTranslate.** The project's officially hosted public instance
(`libretranslate.com`) requires an API key for meaningful use and does not reliably support
browser CORS on that hosted endpoint; a keyless, CORS-friendly deployment would require
self-hosting an instance, which is out of scope pre-Milestone 8 (no hosting/deployment story
exists yet). **Decision: Reject** for this milestone. Revisit once Milestone 8 introduces a
backend/hosting story and proper API-key configuration.

**Candidate 2 — Lingva Translate.** An open-source front-end that proxies Google Translate,
distributed as multiple independently-run public instances (e.g. `translate.plausibility.cloud`
and other community mirrors), exposing a plain keyless REST GET endpoint returning JSON — no
credential, no CORS proxy needed for a standard fetch call, same request shape as the existing
MyMemory integration. Translation quality inherits from Google Translate, generally strong for
en→ko. **Caveat:** individual community-run mirrors carry no uptime SLA and can disappear or rate-
limit independently. **Decision: Adopt as the secondary/fallback provider**, used only after
MyMemory's primary+retry attempt (§3) fails, and implemented against a short **ordered list of at
least two known instance hostnames**, tried in sequence, so a single mirror's outage doesn't take
down the whole fallback tier. Exact instance hostnames are a Developer-level configuration detail,
not an architectural decision, but the list must not be hardcoded to a single instance.

**Candidate 3 — existing `simpleTranslate()` static phrase map.** **Decision: Retain as the final
fallback tier**, but its output contract changes (this is the direct fix for spec item 1): today it
returns a single string that is ambiguous between "a real, if crude, phrase substitution happened"
and "nothing matched, here's a placeholder sentence." It must instead return a structured result —
`{ ko, matched: boolean }` — so the caller can make an explicit status decision instead of
string-comparing output against a hardcoded placeholder sentence (the current, fragile
`out===s ? placeholder : out` check at line 319). If `matched` is `false` after every tier
(MyMemory, Lingva, and the phrase map) has been tried, the sentence's final status is `error`, not
a rendered placeholder string.

**Considered, not adopted here — flagged for CEO/PM, not decided unilaterally:** MyMemory's own
docs (verified above) show that supplying a contact email via the `de=` parameter raises the daily
quota 10x (5,000 → 50,000 chars/day) with no verification step. This is a legitimate reliability
lever, but embedding a real, monitorable email address into public client-side source (this is a
public static file with no server to hide it behind) is a product/privacy decision — whose address,
whether the team is comfortable with that address being publicly associated with the project and
potentially receiving MyMemory-side correspondence — not a technical one. Architecture is not
making this call; it's listed as an open option in §10 for the CEO.

**Recommendation summary (AC4's required explicit recommendation):** Adopt Lingva Translate as the
sole secondary provider for this milestone. Reject LibreTranslate. Retain and fix
`simpleTranslate()` as the tertiary/last-resort tier with a corrected, non-string-sniffed contract.
Defer the MyMemory email-quota option to the CEO as a separate, non-blocking option.

---

## 7. Risks and tradeoffs

- **Uncached failures re-spend quota.** A sentence that fails repeatedly across `analyze()` calls
  in the same session re-attempts the full provider chain every time (§4's deliberate tradeoff).
  Acceptable given MyMemory's daily-character (not per-request) quota model, but worth monitoring
  if real usage shows repeated-failure sentences are common.
- **No SLA on fallback mirrors.** Lingva's community instances can go down without notice; the
  ordered-instance-list mitigation (§6) reduces but doesn't eliminate this. Total failure across
  all tiers is possible and, per AC1, must degrade to an honest `error` state — that is the correct
  outcome, not a bug, when every tier genuinely fails.
- **Third-party exposure of passage text, doubled.** Sentences already leave the browser to
  MyMemory today (pre-existing, not introduced by this milestone). Adding Lingva as a fallback
  means a second third party can receive passage text when the primary fails. This is a pre-
  existing category of exposure (no privacy notice currently exists in the app for the *existing*
  MyMemory calls either), not a new category — flagged upward in §10, not treated as a blocker for
  this milestone, since fixing the underlying gap for MyMemory itself is out of this milestone's
  stated scope.
- **Progressive per-row DOM patching adds a second render path.** `updateTranslationRow(i)` must
  stay in sync with what `translation(a)`'s full-render path produces, or the tab can show
  inconsistent markup if a user switches tabs mid-translation and back. Mitigation: both paths must
  render from the exact same per-row template function (a small shared helper), not two
  independently-maintained markup strings — called out explicitly in §8 as a implementation
  requirement, not left to Developer discretion.
- **Read coverage caveat.** This document is based on the translation-related regions of
  `index.html` located via targeted `grep`/`Read`, not a full line-by-line read of all 728 lines.
  If a translation code path exists outside the regions listed in §0 that wasn't surfaced by the
  search terms used, it is not accounted for here — flagged as a residual assumption in §10.

---

## 8. Exact file-by-file implementation plan

This is a single-file application; there is one file to change.

### `index.html` — only file touched

| Region | Change |
|---|---|
| Near `const state={...}` (line 307) | Add `const CONCURRENCY_LIMIT=3`, `const REQUEST_TIMEOUT_MS=8000`, `const RETRY_BACKOFF_MS=800`, `const translationCache=new Map()`. |
| `simpleTranslate(s)` (lines 316–320) | Change return type from a bare string to `{ko, matched}`. Logic otherwise unchanged (same phrase-map array, same substitution loop). |
| `translateSentence(s)` (line 321) | Replace with `translateSentenceReliable(s, {bypassCache=false}={})`: checks `translationCache` (unless bypassed), then runs the provider chain — MyMemory attempt 1 → 800ms backoff → MyMemory attempt 2 → Lingva (ordered instance list) → `simpleTranslate()` — each network attempt wrapped by the new `fetchWithTimeout`. Returns `{ko, status: 'success'|'error', source}` and writes successful results into `translationCache`. |
| New helper, colocated near the above | `fetchWithTimeout(url, ms)` — `fetch` wrapped in `AbortController`, rejects on abort. |
| New helper, colocated near the above | `runWithConcurrency(items, limit, worker)` — per §2's pseudocode. |
| New helper, colocated near the above | `translateLingva(s)` — tries each configured Lingva instance hostname in order, returns `null` on total failure (caller falls through to phrase map). |
| New helper, colocated near `translation(a)` | `translationRowTemplate(x, i)` — the single shared per-row markup function used by both the full `translation(a)` render and the targeted `updateTranslationRow(i)` patch (per §7's consistency requirement), branching on `x.status`. |
| `analyze()` (lines 343–378) | Build `translations = sents.map(s=>({en:s, ko:null, status:'loading', source:null}))` immediately after `sents` is computed; assign `state.analysis` and call `render()` once at that point (loading skeleton visible); replace the blocking `Promise.all(sents.map(s=>translateSentence(s)))` with `runWithConcurrency(sents, CONCURRENCY_LIMIT, async (s,i)=>{ const r = await translateSentenceReliable(s); Object.assign(translations[i], {ko:r.ko, status:r.status, source:r.source}); updateTranslationRow(i); })`; update `#status` progressively per §5; keep the existing `analyzeBtn.disabled` guard and `finally` block exactly as-is (still awaits the full pool before re-enabling, preserving Milestone 1's overlap-guard behavior). |
| New function, colocated near `analyze()` | `updateTranslationRow(i)` — if `state.active==='translation'`, patch the DOM node for row `i` using `translationRowTemplate`; otherwise no-op (avoids wasted DOM work on tabs the user isn't viewing; the full `translation(a)` render already picks up current state whenever the user switches to that tab). |
| New function, colocated near `analyze()` | `retrySentence(i)` — exposed for the retry button's `onclick` (matches the existing inline-handler convention already used throughout, e.g. `onclick="analyze()"`); calls `translateSentenceReliable(sentenceText, {bypassCache:true})` for that one sentence, updates `state.analysis.translations[i]`, calls `updateTranslationRow(i)`. |
| `translation(a)` (lines 443–444) | Rewritten to map each entry through `translationRowTemplate(x,i)` instead of the current single-shape template string. |
| `loadSaved()` (line 704) | Defensive read: when restoring a saved item's `translations`, default any entry missing `status` to `'success'` (older saved sets predate this milestone and won't have the field) — matches `CODING_STANDARDS.md`'s "handle `localStorage` reads defensively" rule. |
| `<style>` block | Add CSS for `.sentence.loading` (skeleton/spinner treatment) and `.sentence.error` (warning-colored variant + retry button styling), following the existing `c-blue`/`c-pink`/`c-mint` card-color convention already present. Exact values are a Developer-level styling decision, not specified further here. |

No other function, tab, or file is touched. In particular: `dictionary`, `ipaMap`, `exampleBank`,
`levelFor`, `fallbackKoreanGloss` (Milestone 3), `grammarInsights`, `makeQuestions` (Milestone 5),
and all saved-study-set CRUD logic beyond the one defensive-read line above, are untouched.

---

## 9. Architecture acceptance criteria

Restates each PM acceptance criterion as an architecture-level pass condition a developer can
verify their implementation against before handing off to QA:

1. **AC1 (distinct failure state):** `translationRowTemplate` has a `status==='error'` branch whose
   rendered output is visually and textually distinct from the `status==='success'` branch, and
   `translateSentenceReliable` only ever returns `status:'error'` — never a `ko` string containing
   the old placeholder sentence rendered as if `status:'success'`.
2. **AC2 (bounded concurrency):** `runWithConcurrency` never has more than `CONCURRENCY_LIMIT`
   worker iterations executing their `await` simultaneously — verifiable by instrumenting a mock
   `fetch` with an in-flight counter during a manual/jsdom test.
3. **AC3 (no re-request of already-succeeded sentences):** a second `analyze()` call in the same
   page session, given an unchanged passage, results in zero new `fetch` calls for sentences whose
   exact text is already a key in `translationCache` — verifiable via a `fetch` spy call-count
   assertion.
4. **AC4 (written evaluation + recommendation):** satisfied by §6 of this document existing with an
   explicit adopt/reject/defer decision per candidate.
5. **AC5 (per-sentence loading indicator):** `updateTranslationRow`/`translationRowTemplate` render
   a `status==='loading'` state for every sentence between `state.analysis` being assigned and that
   sentence's worker settling — verifiable by delaying a mocked `fetch` response and asserting the
   loading row is present before resolution and replaced after.

---

## 10. Unresolved risks and open questions for the CEO

- **MyMemory quota-bump via `de=` email parameter** (§6): a real, low-effort reliability
  improvement, deliberately not adopted here because it requires choosing a public-facing contact
  email — a product/privacy decision, not a technical one. Needs a CEO/PM call, independent of this
  architecture's approval.
- **Pre-existing privacy gap** (§7): passage text already leaves the browser to a third-party
  translation service today (MyMemory), with no privacy notice in the app; this milestone adds a
  second third party (Lingva) to that same, pre-existing gap rather than introducing a new category
  of exposure. Flagged as a candidate for a future milestone, not fixed here (out of Milestone 2's
  stated scope).
- **Read coverage caveat** (§7): this design is based on the translation-related code located via
  targeted search in a 728-line file, not an exhaustive line-by-line read. If Development discovers
  an additional translation-related code path not covered by §8's file-by-file plan, per
  `CODING_STANDARDS.md`'s scope-discipline rule it should be flagged back to the CEO/Architect
  rather than silently folded in.
- **Lingva instance list staleness:** the specific hostnames used for the ordered fallback list
  (§6) are a point-in-time choice; if all configured mirrors go stale simultaneously post-release,
  that's an operational follow-up (updating a constant), not an architecture defect — noted so it
  isn't mistaken for one during QA/Review.

---

## 11. Test strategy pointer

Per `.ai-company/TESTING_STANDARDS.md` and the precedent set in Milestone 1 (no automated test
framework exists in this repo): verify via throwaway jsdom-based scripts that load `index.html`,
mock/spy on `fetch`, and assert on rendered DOM/state — not checked into the repo as permanent
test files unless `TESTING_STANDARDS.md` says otherwise. At minimum, QA/Developer should expect to
exercise: (a) all-success passage, (b) forced MyMemory failure with successful Lingva fallback, (c)
forced failure of every tier down to the explicit `error` state, (d) concurrency-cap adherence on a
long passage, (e) cache-hit behavior on a second `analyze()` of the same passage, (f) manual
per-sentence retry, (g) backward-compatible load of a saved study set predating this milestone
(missing `status`/`source` fields). Full pass/fail criteria and severity classification are QA's
responsibility per `.ai-company/QA.md`, not specified further here.

---

## Handoff — Milestone 2

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/ARCHITECT.md`, `.ai-company/HANDOFF_PROTOCOL.md`, `.ai-company/CODING_STANDARDS.md`,
  `DEVELOPMENT_PLAN.md`, `docs/milestones/milestone-02/README.md`,
  `docs/milestones/milestone-02/01-PM-SPEC.md` (confirmed Status: APPROVED before proceeding), and
  the translation-related regions of `index.html` (see §0 for exact line ranges read). Also
  consulted MyMemory's own published usage-limits documentation and current public information on
  LibreTranslate/Lingva Translate to ground §6's evaluation in verifiable, current facts rather than
  assumption.
- **Scope completed:** Wrote `docs/milestones/milestone-02/02-ARCHITECTURE.md` in full: scope
  traceability, approach summary, concurrency/throttling policy, retry/timeout behavior, session
  cache design, fallback-provider evaluation with an explicit recommendation, loading/progress/error
  state design, security/privacy considerations, performance constraints, an exact file-by-file
  (function-by-function, since this is a single-file app) implementation plan, developer task order,
  and architecture-level acceptance criteria mapped to each PM acceptance criterion.
- **Files changed:** `docs/milestones/milestone-02/02-ARCHITECTURE.md` only. `index.html` and every
  other application source file were read but not modified, per instruction.
- **Commits created:** None — not committed, per explicit instruction for this session.
- **Tests performed:** N/A (Architecture phase; no code changes). Verified the design's technical
  claims against primary sources rather than assuming: read the actual `index.html` functions this
  design modifies (not assumed from memory), and fetched MyMemory's own usage-limits documentation
  page directly to confirm no published per-request/burst rate limit exists (only a daily character
  quota), which grounds §2's concurrency-cap decision.
- **Unresolved risks:** listed in full in §10 — the MyMemory email-quota option (deferred to CEO,
  not a blocker), the pre-existing translation-privacy gap (flagged upward, out of this milestone's
  scope), the read-coverage caveat (targeted search, not exhaustive), and Lingva instance-list
  staleness (operational, not architectural).
- **Next agent:** CEO.
- **Explicit stop point:** Awaiting CEO approval of this `02-ARCHITECTURE.md` at the gate defined in
  `.ai-company/WORKFLOW.md` ("Architecture Design → [GATE] CEO Approval"). Development must not
  begin until that approval is recorded in this document.
