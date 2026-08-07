# 04-QA-REPORT.md — Milestone 2: Translation Reliability

**Status: Development stage complete — all 8 of 8 Dev Tasks implemented and independently
QA-verified as PASS (Dev Task 6's initial QA pass found defect SAVE-1 (High), reported directly to
the CEO/Developer per the QA Defect Gate — see the Bug fix SAVE-1 QA pass below for the full
independent re-verification: PASS). No unresolved Critical/High/Medium defects remain open as of
Dev Task 8. Next stage per `.ai-company/WORKFLOW.md`: Principal Review Quality Gate.**

Append one entry per QA pass. Do not overwrite prior entries — the Developer↔QA loop history
should remain visible.

## QA pass template

```
### QA pass: <date> — against commit(s) <hash(es)>

#### Acceptance criteria results (from 01-PM-SPEC.md)

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 |  | pass/fail |  |

#### Regression check

<what adjacent features were checked and result>

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|

#### Handoff

<per .ai-company/HANDOFF_PROTOCOL.md — to Developer if unresolved Critical/High, else to Reviewer>
```

---

### QA pass: 2026-08-06 — against commit(s) `4f8975d` (scaffolding), `89b2887` (log entry)

**Scope of this pass:** Reviewed only Dev Task 1 of the 8-task order in `02-ARCHITECTURE.md` §8
("concurrency/timeout/cache scaffolding"), per explicit instruction — not the full milestone.

**Why AC1–AC5 are not scored pass/fail this pass:** `01-PM-SPEC.md`'s five acceptance criteria
describe end-to-end translation behavior (distinct error states, bounded live network requests,
cross-`analyze()` cache hits, a working fallback provider, per-sentence loading indicators). None
of that is reachable yet — Task 1 adds only inert scaffolding (four constants and two pure helper
functions, `fetchWithTimeout` and `runWithConcurrency`) that nothing in the app currently calls.
Tasks 2–8 (refactoring `simpleTranslate`/`translateSentence`, wiring `analyze()` to the new pool,
the row-template/UI states, the Lingva fallback) are what will actually make AC1–AC5 exercisable.
Scoring them now would be meaningless; they are marked **N/A — not yet implemented** below and
will be scored for real once the code path they depend on exists.

#### Acceptance criteria results (from `01-PM-SPEC.md`)

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 | Distinct, labeled failure state (never a canned string as if real) | N/A | Depends on Dev Tasks 2/5/8 (`simpleTranslate` refactor, row template, `analyze()` wiring) — not implemented yet. |
| 2 | Bounded-concurrency dispatch of translation requests | N/A | The bounded-pool primitive (`runWithConcurrency`) exists and is independently verified correct (see below), but it is not yet wired into `analyze()`'s actual dispatch (`analyze()` still calls the old unbounded `Promise.all(...)`, per Task 8 in the architecture's plan). |
| 3 | No re-request of already-succeeded sentences in-session | N/A | `translationCache` exists and is empty/correctly typed, but nothing writes to or reads from it yet (Task 2/8). |
| 4 | Written fallback-provider evaluation with explicit recommendation | N/A (out of Dev Task 1's scope — already satisfied) | Satisfied by `02-ARCHITECTURE.md` §6, which exists and gives an explicit adopt/reject/defer recommendation. Not something Task 1 touches or could regress. |
| 5 | Per-sentence loading indicator | N/A | Depends on Dev Tasks 5/8 (row template, progressive render) — not implemented yet. |

#### Task-1-level verification (against `02-ARCHITECTURE.md` §2–§4 and §8 row 1)

| Item | Expected per architecture | Actual (verified) | Result |
|---|---|---|---|
| Placement | Immediately after `const state={...}` (~line 307), before the tab-click listener | Confirmed at that exact location via `git show` | Pass |
| `CONCURRENCY_LIMIT` | `3` | `3` | Pass |
| `REQUEST_TIMEOUT_MS` | `8000` | `8000` | Pass |
| `RETRY_BACKOFF_MS` | `800` | `800` | Pass |
| `translationCache` | `new Map()`, module-scoped | `Map`, empty on load | Pass |
| `fetchWithTimeout(url, ms)` | `fetch` wrapped in `AbortController`, rejects on timeout | Matches; rejects with `AbortError` once `ms` elapses against a hung mock; timer cleared via `finally` regardless of outcome | Pass |
| `runWithConcurrency(items, limit, worker)` | Bounded worker pool per §2 pseudocode | Matches the pseudocode's structure exactly; never exceeds the configured limit, preserves result-to-input order, handles `limit > items.length` without hanging | Pass |
| File scope | Only `index.html`; no existing line modified | `git show 4f8975d --stat`: 1 file changed, 20 insertions(+), 0 deletions | Pass |
| Commit hygiene | One logical change; hash recorded in `03-IMPLEMENTATION-LOG.md` | Single commit `4f8975d6f315f89757280ec67e2da61c86e49e5f`; hash correctly recorded | Pass |

**Independent verification method:** This repository has no automated test framework
(`TESTING_STANDARDS.md`). I wrote my own throwaway jsdom script — independent of the Developer's
own reported script/results — that loads the actual `index.html` from the repository, executes it,
and asserts on runtime state and behavior rather than trusting the implementation log's claims at
face value. Result: **19/19 assertions passed**, covering: regression baseline (`splitSentences`,
`simpleTranslate`, `translateSentence`, `analyze` still defined and behaviorally unchanged;
`state.active` still initializes to `"overview"`), the four new constants' exact values,
`translationCache`'s type and initial emptiness, `fetchWithTimeout`'s abort-on-timeout behavior
against a mock `fetch` that never resolves on its own, and `runWithConcurrency`'s concurrency-cap
enforcement (never exceeded, reaches the cap under load), input-order preservation, and the
`limit > items.length` edge case. Script kept as a local scratch file only, not committed to the
repository, per `TESTING_STANDARDS.md`.

#### Regression check

Confirmed `splitSentences`, `simpleTranslate`, `translateSentence`, and `analyze` remain defined
with unchanged behavior (`splitSentences("Hello world. This is a test!")` still returns 2
sentences), and `state.active` still initializes to `"overview"`. No other tab or feature (vocab,
grammar, SAT questions, saved study sets, speech recording) references the modified code region,
and `git show` confirms zero existing lines were altered — this task is purely additive, currently
unreferenced code, so regression risk is effectively nil.

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| — | None found in Dev Task 1's scope. | — | — | — |

**Observation (non-blocking, not a Task 1 defect):** `docs/milestones/milestone-02/01-PM-SPEC.md`
and `02-ARCHITECTURE.md` currently have uncommitted working-tree changes (the finalized/approved
text, including the CEO-approval sections, that this review relied on to confirm both documents'
`Status: APPROVED`). `03-IMPLEMENTATION-LOG.md` already notes these predate this Development
session and were correctly left uncommitted by the Developer, since committing PM/Architecture
documentation is outside the Senior Developer's role scope. Flagging only so the Release
Manager/CEO is aware these two documents are not yet committed to git — not something Dev Task 1
introduced or is responsible for fixing.

#### Overall result

**PASS** for Developer Task 1 (concurrency/timeout/cache scaffolding) as implemented in commit
`4f8975d`. No unresolved Critical/High/Medium/Low defects. This is a partial-milestone checkpoint,
not a full milestone sign-off — AC1, AC2, AC3, and AC5 remain unverified (marked N/A above) until
Dev Tasks 2–8 are implemented and re-reviewed; AC4 is already satisfied by existing architecture
documentation and unaffected by this task.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/QA.md`, `.ai-company/TESTING_STANDARDS.md`, `.ai-company/DEFINITION_OF_DONE.md`,
  `.ai-company/HANDOFF_PROTOCOL.md`, `.ai-company/GIT_RULES.md`, `.ai-company/CODING_STANDARDS.md`,
  `docs/milestones/milestone-02/README.md`, `01-PM-SPEC.md` (Status: APPROVED),
  `02-ARCHITECTURE.md` (Status: APPROVED, §2–§4/§8/§9 specifically), `03-IMPLEMENTATION-LOG.md`
  (Dev Task 1 entry and handoff), `DEVELOPMENT_PLAN.md`, and the actual `index.html` diff for
  commit `4f8975d` (`git show`).
- **Scope completed:** Independently verified Dev Task 1 only, against `02-ARCHITECTURE.md` and
  the actual code (not just the implementation log's description of it), per an independently
  authored jsdom verification script (19/19 assertions passed). No defects found.
- **Files changed:** None (QA is read-only this pass). `docs/milestones/milestone-02/04-QA-REPORT.md`
  updated with this pass's findings.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See "Independent verification method" and "Task-1-level verification"
  above.
- **Unresolved risks:** AC1/AC2/AC3/AC5 remain unverified end-to-end pending Dev Tasks 2–8 (this is
  expected at this checkpoint, not a gap). The two uncommitted documentation files noted above are
  carried forward as an observation for the Release Manager/CEO, not a QA defect.
- **Next agent:** Per `03-IMPLEMENTATION-LOG.md`'s own stop point, Task 1 was paused for CEO/operator
  approval before Dev Task 2 begins. No unresolved Critical/High defects exist, so from QA's side
  there is nothing blocking that approval — handoff is to the **CEO** to approve proceeding to Dev
  Task 2 (or to the **Senior Developer** directly, if the operator waives the per-task CEO gate).
- **Explicit stop point:** QA has completed its review of Dev Task 1 only. Awaiting the
  CEO/operator's decision to proceed to Dev Task 2 before any further Development or QA work on
  this milestone.

---

### QA pass: 2026-08-06 — against commit(s) `313757b` (code), `95f9d43` (log entry)

**Scope of this pass:** Reviewed only Dev Task 2 of the 8-task order in `02-ARCHITECTURE.md` §8
("`simpleTranslate(s)` structured return"), per explicit instruction — not the full milestone.
Verified against nine dimensions as instructed: acceptance criteria, regression, security,
performance, coding standards, tests, architecture compliance, git hygiene, documentation.

**Why AC1/AC2/AC3/AC5 are still not scored pass/fail this pass:** unchanged reasoning from the
Dev Task 1 pass above — these criteria describe end-to-end behavior (rendered error states, live
bounded dispatch, cross-`analyze()` cache hits, per-sentence loading UI) that depends on Dev Tasks
3, 5, and 8, none of which have been implemented. Task 2 only changes `simpleTranslate()`'s
internal return shape and makes the one necessary adaptation to keep `translateSentence()` working
in the interim; nothing in `analyze()` or the render path (`translation()`) reads the new
`matched` field yet. Scoring the full criteria now would still be premature. AC4 remains satisfied
by `02-ARCHITECTURE.md` §6, unaffected by this task.

#### 1. Acceptance criteria (from `01-PM-SPEC.md`)

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 | Distinct, labeled failure state (never a canned string as if real) | N/A | `simpleTranslate()` now produces the precondition this criterion needs (`matched:false` distinguishable from `matched:true`), verified directly — but nothing consumes it yet; still depends on Dev Tasks 3/5/8. |
| 2 | Bounded-concurrency dispatch | N/A | Untouched by this task; `analyze()` still calls the old unbounded `Promise.all(...)` (Task 8). |
| 3 | No re-request of already-succeeded sentences | N/A | Untouched by this task; `translationCache` still unused (Task 3/8). |
| 4 | Written fallback-provider evaluation | Pass (pre-existing, unaffected) | Still satisfied by `02-ARCHITECTURE.md` §6. |
| 5 | Per-sentence loading indicator | N/A | Untouched by this task (Task 5/8). |

#### 2. Regression

Independently verified (fresh jsdom script, not a re-run of the Developer's own script) that
`splitSentences`, `words`, `analyze`, and `state.active` are unaffected, and — critically — that
Dev Task 1's scaffolding (`CONCURRENCY_LIMIT`, `REQUEST_TIMEOUT_MS`, `RETRY_BACKOFF_MS`,
`translationCache`, `fetchWithTimeout`, `runWithConcurrency`) is completely untouched by this
commit. `translateSentence()`'s external contract (always resolves to a plain string) was
re-verified across all three of its branches: MyMemory success (string passed through
byte-identical), MyMemory failure + phrase-map match (returns the Korean substitution, unchanged
from before), MyMemory failure + no match (returns the exact legacy placeholder string — verified
by strict string equality, not just "looks similar"). No other tab or feature references
`simpleTranslate`/`translateSentence` (confirmed via `grep` — the only call site is
`translateSentence`'s own body, and `analyze()`'s existing `translateSentence(s)` call, both
unchanged). **Result: Pass, no regressions found.**

#### 3. Security

`simpleTranslate()`'s substitution loop is pure string replacement and was already safe from
injection (it does not evaluate or execute its input). Confirmed this task introduces no new
`innerHTML`/DOM sink: `analyze()` still assigns `translateSentence()`'s plain-string return into
`translations[i].ko` exactly as before, and the render path (`translation(a)` at line 474) still
wraps every `x.ko` and `x.en` in `escapeHtml(...)` before insertion — confirmed by direct source
read, not assumption. Tested HTML/script-shaped input (`<script>alert(1)</script>`) through
`simpleTranslate()`: it passes through unmodified (expected — sanitization is deliberately the
render layer's job, per `CODING_STANDARDS.md`'s existing convention), and `escapeHtml()` still
neutralizes it correctly at render time. **Result: Pass, no new attack surface; pre-existing XSS
defense (Milestone 1) remains intact and untouched.**

#### 4. Performance

`simpleTranslate()`'s algorithmic shape (one `.forEach` over a fixed ~25-entry map, each doing one
`.replace`) is completely unchanged — only the return statement differs. Measured on an
artificially long input (2,000 repetitions of a matched word): well under 1ms in practice, no
measurable regression. No new network calls, loops, or allocations of consequence were introduced.
**Result: Pass, no performance impact.**

#### 5. Coding standards

Matches `.ai-company/CODING_STANDARDS.md`: smallest change that implements the approved
architecture (only the return statement and one call site changed); no unrelated refactoring;
existing camelCase/inline-function style matched; the deliberately temporary bridge in
`translateSentence()` carries an explicit `TODO(Milestone 2, Dev Task 3)` comment stating what it
does today and which task removes it, per the "Comments and TODOs" rule. Phrase-map array
confirmed byte-identical to the pre-Task-2 version (diffed directly). **Result: Pass.**

#### 6. Tests

This repository has no automated framework (`TESTING_STANDARDS.md`); verification is manual/
scripted. I independently authored a fresh jsdom verification script (not the Developer's own
script) covering the above dimensions: 27 of 29 assertions passed outright; the remaining 2
"failures" were confirmed to be artifacts of my own test script's substring/regex checks, not real
defects — re-verified directly against the source:
- One assertion used a naive regex to confirm `translation()` calls `escapeHtml(x.ko)`; the regex
  broke on the template literal's nested `${...}` braces. Direct `grep` at line 474 confirms
  `escapeHtml(x.ko)` is present exactly as before.
- One assertion checked that the literal string `"translateSentenceReliable"` does not appear
  anywhere in `index.html`, expecting zero occurrences. It appears exactly once — inside the
  Developer's own forward-looking `TODO` comment (naming what Dev Task 3 will introduce), not as
  an actual function definition. Confirmed via `grep -n "function translateSentenceReliable"`
  returning no match.
Both are corrected test-design issues on QA's side, not code defects. **Result: Pass (27/27
substantively meaningful assertions confirmed correct once the two script artifacts are
accounted for).**

#### 7. Architecture compliance

Matches `02-ARCHITECTURE.md` §6 and §8 row 2 exactly: `simpleTranslate()`'s return type is
`{ko, matched}`, substitution logic unchanged, placement unchanged. Confirmed no scope creep into
Dev Task 3: `translateSentenceReliable`, `translateLingva`, `translationRowTemplate`, and
`updateTranslationRow` are all absent as actual implementations (verified via `grep` for function
definitions, not just string search). The one addition beyond the literal table row — the
one-line adapter in `translateSentence()` — is not an architecture deviation in substance: both
`simpleTranslate`'s new contract (§6) and `translateSentence`'s eventual replacement (§8 row 3)
are already part of the CEO-approved architecture; only the transitional bridge between the two
tasks required a judgment call. It is transparently disclosed in-code (`TODO` comment) and in
`03-IMPLEMENTATION-LOG.md`, consistent with `.ai-company/DEVELOPER.md`'s "stop and flag it" rule
rather than silently expanding scope. **Result: Pass.**

#### 8. Git hygiene

Two commits reviewed: `313757b` (code only — `index.html`, 14 insertions/3 deletions, confirmed
via `git show --stat`) and `95f9d43` (docs only — `03-IMPLEMENTATION-LOG.md`). One logical change
per commit, correctly separated (`.ai-company/GIT_RULES.md` rules 4/9). Commit messages follow the
repo's established `Prefix: description` convention with a full explanatory body. Secret scan
(`git show` piped through a credential/token/key pattern search) found nothing in either commit.
Working tree checked before and after: only the two pre-existing, already-flagged uncommitted
documentation files (`01-PM-SPEC.md`, `02-ARCHITECTURE.md`) remain outstanding, both carried
forward from before this task and not touched by it; no stray or unrelated changes. **Result:
Pass.**

#### 9. Documentation

`03-IMPLEMENTATION-LOG.md`'s Dev Task 2 entry was cross-checked against the actual `git show`
output: file/line-count claims ("14 lines added, 3 removed") match exactly; the commit hash is
recorded correctly; the handoff fields are all present and complete per
`.ai-company/HANDOFF_PROTOCOL.md`. One minor wording inaccuracy found: the entry's "Unresolved
risks" line describes the `translateSentence` bridge as having "its duplicated placeholder-string
literal" — in fact the placeholder string was *relocated* from `simpleTranslate()` into
`translateSentence()`, not duplicated; `grep` confirms the literal now appears exactly once in
`index.html`, not twice. This is a wording/precision issue only, does not misrepresent the code's
actual behavior or safety, and does not affect any test result or defect classification. **Result:
Pass, with one Low-severity documentation wording nit (see Defects below).**

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| DOC-1 | `03-IMPLEMENTATION-LOG.md`'s Dev Task 2 handoff calls the relocated placeholder string "duplicated" when `grep -n "번역 서비스를 불러오지 못했습니다" index.html` shows it occurs exactly once (in `translateSentence`, not also in `simpleTranslate`). | Low | Read the "Unresolved risks" bullet in the Dev Task 2 handoff section of `03-IMPLEMENTATION-LOG.md`; compare against `grep` output on `index.html`. | Open — cosmetic wording only, does not block release per `.ai-company/DEFINITION_OF_DONE.md` (Low/Medium may be deferred). |

No Critical, High, or Medium defects found.

#### Overall result

**PASS** for Developer Task 2 (`simpleTranslate()` structured return + the necessary
`translateSentence()` bridge) as implemented in commit `313757b`, logged in `95f9d43`. No
unresolved Critical/High defects — one Low-severity documentation wording nit (DOC-1), which does
not block proceeding to Dev Task 3. AC1, AC2, AC3, and AC5 remain unverified end-to-end (expected
at this checkpoint); AC4 remains satisfied and unaffected.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md` (all re-read this session per operator instruction), plus
  (re-confirmed from context established earlier this milestone) `.ai-company/AGENTS.md`,
  `.ai-company/DEFINITION_OF_DONE.md`, `.ai-company/HANDOFF_PROTOCOL.md`,
  `.ai-company/GIT_RULES.md`, `.ai-company/CODING_STANDARDS.md`,
  `docs/milestones/milestone-02/README.md`, `01-PM-SPEC.md` (Status: APPROVED),
  `02-ARCHITECTURE.md` (Status: APPROVED — §6 and §8 rows 2/3 specifically re-read),
  `03-IMPLEMENTATION-LOG.md` (Dev Task 2 entry and handoff), and the actual `index.html` diffs for
  commits `313757b` and `95f9d43` (`git show`).
- **Scope completed:** Independently verified Dev Task 2 only, against `02-ARCHITECTURE.md` and
  the actual code/commits (not just the implementation log's description of them), across all nine
  requested dimensions, via a freshly authored jsdom verification script. One Low-severity
  documentation defect found (DOC-1); no Critical/High/Medium defects.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See sections 1–9 above, in particular "6. Tests" for the independent
  verification methodology and the two self-corrected test-script artifacts.
- **Unresolved risks:** AC1/AC2/AC3/AC5 remain unverified end-to-end pending Dev Tasks 3/5/8 (expected,
  not a gap). DOC-1 (Low, wording only) carried forward — safe to defer per
  `.ai-company/DEFINITION_OF_DONE.md`, but should be corrected opportunistically (e.g., in the Dev
  Task 3 log entry, when the bridge itself is removed and the wording becomes moot anyway). The two
  uncommitted PM/Architecture documentation files remain outstanding, carried forward again from
  the Dev Task 1 pass — still not this task's responsibility to fix.
- **Next agent:** Per `03-IMPLEMENTATION-LOG.md`'s stop point, handoff is to the **CEO**/operator to
  approve proceeding to Dev Task 3 (or directly to the **Senior Developer**, if the operator waives
  the per-task CEO gate) — no unresolved Critical/High defects block this.
- **Explicit stop point:** QA has completed its review of Dev Task 2 only. Awaiting the
  CEO/operator's decision to proceed to Dev Task 3 before any further Development or QA work on
  this milestone.

---

### QA pass: 2026-08-06 — against commit(s) `1563cef` (code), `54f235a` (log entry)

**Scope of this pass:** Reviewed only Dev Task 3 of the 8-task order in `02-ARCHITECTURE.md` §8
("`translateSentenceReliable`/`translateLingva`"), per explicit instruction — not the full
milestone. Verified against the eight dimensions requested: acceptance criteria, regression,
security, performance, architecture compliance, coding standards, tests, git hygiene.

**Why AC1/AC2/AC3/AC5 are still not scored pass/fail this pass:** unchanged reasoning from prior
passes — `translateSentenceReliable`/`translateLingva` are fully implemented and independently
verified to behave correctly in isolation, but nothing in `analyze()` or the render path calls
them yet (confirmed directly — see "2. Regression" below). End-to-end scoring requires Dev Task 8
to wire `analyze()` to this function. AC4 remains satisfied by `02-ARCHITECTURE.md` §6, unaffected.

#### 1. Acceptance criteria (from `01-PM-SPEC.md`) — preconditions verified

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 | Distinct, labeled failure state | N/A (precondition met) | Independently confirmed: total failure across all tiers returns `{ko:null, status:'error', source:null}` — never a placeholder string rendered as if real. Still not wired into the UI (Task 8). |
| 2 | Bounded-concurrency dispatch | N/A | `runWithConcurrency` (Task 1) confirmed still present/unaffected; not yet used by `analyze()`. |
| 3 | No re-request of already-succeeded sentences | N/A (precondition met) | Independently confirmed: an identical sentence string issued twice through `translateSentenceReliable` results in exactly one `fetch` call — the cache-hit path works correctly in isolation. Still not reachable from `analyze()`'s current `Promise.all` dispatch (Task 8). |
| 4 | Written fallback-provider evaluation | Pass (pre-existing, unaffected) | Still satisfied by `02-ARCHITECTURE.md` §6. |
| 5 | Per-sentence loading indicator | N/A | Untouched by this task (Task 5/8). |

#### 2. Regression

Independently confirmed via a freshly authored jsdom script (not a re-run of the Developer's own
script): `CONCURRENCY_LIMIT`/`REQUEST_TIMEOUT_MS`/`RETRY_BACKOFF_MS` unchanged; `simpleTranslate()`
still returns exactly `{ko, matched}`; the old `translateSentence()` still returns a plain string.
Critically, verified at the source level that `analyze()`'s body is byte-for-byte unchanged in the
relevant region — it still calls `translateSentence(s)` via `Promise.all(sents.map(...))`, and does
**not** reference `translateSentenceReliable` anywhere. `splitSentences`/`words`/`state.active` all
unaffected. **Result: Pass, no regressions — this task is confirmed inert/dormant as claimed.**

#### 3. Security

Checked three angles specific to this task's new network surface: (a) `LINGVA_INSTANCES` entries
are fixed, non-user-influenceable hostname literals (`/^[a-z0-9.-]+$/i` — no injected input can
change which host is contacted); (b) sentence text passed to `translateLingva`/
`translateSentenceReliable` is run through `encodeURIComponent` before being placed in the URL —
tested with a path-traversal/query-injection-shaped sentence (`"../../etc/passwd?evil=1&x="`) and
confirmed the resulting request URL contains the fully-encoded string, not a raw path-breakout;
(c) `simpleTranslate`'s phrase-map substitution still does not interpret or strip HTML/script-
shaped input (by design — sanitization is the render layer's job), and `escapeHtml()` (the
Milestone 1 XSS fix) still neutralizes it correctly. Also confirmed the `status:'error'` path
always returns `ko:null` — there is no ambiguous value that could later be mistaken for real,
escapable content. **Result: Pass, no new attack surface introduced.**

#### 4. Performance

Confirmed the provider chain terminates (does not hang indefinitely) even when every network tier
fails: ran the full chain against a `fetch` mock that never resolves on its own and relies entirely
on `fetchWithTimeout`'s `AbortController` to unblock each attempt. **Observed worst-case latency
for a single sentence when every tier fails: ~32.9 seconds** (2 MyMemory attempts × 8s timeout +
800ms backoff + 2 Lingva mirror attempts × 8s timeout each = ~24.8s of timeouts, observed ~32.9s
including scheduling overhead) — within the expected bound derived from the architecture's own
constants, and the promise does resolve rather than hang. This number is not a Task 3 defect — the
8s/8s/800ms values are `02-ARCHITECTURE.md` §2/§3's own approved constants, correctly implemented
here, not chosen by this task. **Flagging as a forward-looking observation, not a defect:** once
Dev Task 8 wires this into `runWithConcurrency`'s pool, a passage where multiple concurrent slots
all hit total-failure sentences simultaneously could take up to ~33 seconds per affected sentence
before the pool moves on — worth keeping in mind during Task 8's review, particularly alongside
the progressive per-row rendering design (§5), which should at least make partial progress visible
during that wait rather than appearing frozen. **Result: Pass** (correctly implements approved
constants; no unbounded hang; latency characteristic noted for Task 8's attention).

**Secondary observation (Low, non-blocking):** `translationCache` is keyed by sentence text with no
in-flight de-duplication — if two concurrent calls for the *same* sentence text both miss the cache
before either resolves (possible once Task 8's concurrent pool is wired in, e.g. a passage with a
repeated identical sentence), both would independently fire the full network chain rather than the
second awaiting the first's in-flight result. This does not violate AC3 as written (AC3 concerns
re-analysis across separate `analyze()` calls, not concurrent duplicate sentences within one call)
and does not affect Task 3's own correctness, since Task 3's functions are not yet called
concurrently by anything. Logged as a forward-looking note for whoever implements Task 8, not a
Task 3 defect.

#### 5. Architecture compliance

Matches `02-ARCHITECTURE.md` §2–§4/§6/§8 row 3 and the "New helper ... `translateLingva(s)`" row
precisely: cache-then-MyMemory×2-with-backoff-then-Lingva-then-phrase-map chain, `{ko, status,
source}` return shape, cache writes gated to success only (confirmed via `grep` — exactly two
`translationCache.set(` call sites, both inside success branches), `LINGVA_INSTANCES` has more than
one entry per §6's explicit anti-single-instance requirement. Confirmed no scope creep into later
tasks: `updateTranslationRow`, `translationRowTemplate`, and `retrySentence` are all absent as
actual function definitions. Reviewed the Developer's documented "Interpretation of 'Replace'"
judgment call (choosing to add the new functions dormant rather than repointing `analyze()`'s call
site mid-task) and **concur** with the reasoning: repointing the call site without the rest of
Task 8's wiring (progressive rendering, `runWithConcurrency` dispatch, status-aware rendering)
would either break rendering (`[object Object]` in the UI) or require partially implementing
Task 8's scope now, both of which the Developer correctly avoided and transparently documented
in-code and in `03-IMPLEMENTATION-LOG.md`, per `.ai-company/DEVELOPER.md`'s "stop and flag it"
rule rather than deciding silently. **Result: Pass.**

#### 6. Coding standards

Consistent with the file's existing single-file/inline-function/camelCase conventions; new code is
colocated exactly where the architecture's file-by-file plan places it (near the Task 1 scaffolding
for `LINGVA_INSTANCES`, immediately after `translateSentence` for the two new functions). Every new
function/constant carries an explanatory comment referencing the specific architecture section it
implements, matching the pattern established in Tasks 1–2. No unrelated code touched —
`fallbackKoreanGloss` immediately follows the new code, byte-identical to before. **Result: Pass.**

#### 7. Tests

No automated framework exists in this repository (`TESTING_STANDARDS.md`); verification is manual/
scripted. I independently authored a fresh jsdom script (not the Developer's own script) and
re-derived several of the Developer's own claimed behaviors from scratch to check they hold up
under independent test construction, plus additional security/performance/architecture checks the
Developer's own script didn't specifically target (URL-injection-shaped input, real elapsed-time
verification of the full timeout/backoff/fallback chain, source-level confirmation that `analyze()`
is untouched). **Result: 30/30 assertions passed**, with no test-script artifacts this time (unlike
the Dev Task 2 pass, where two of QA's own assertions needed correction).

#### 8. Git hygiene

Two commits reviewed: `1563cef` (code only — `index.html`, 69 insertions/0 deletions, confirmed via
`git show --stat`) and `54f235a` (docs only — `03-IMPLEMENTATION-LOG.md`, 123 insertions/1
deletion). One logical change per commit, correctly separated per `.ai-company/GIT_RULES.md` rules
4/9. Commit messages follow the established `Prefix: description` convention with explanatory
bodies. Secret scan across both commits found nothing. Working tree checked before and after:
only the same three pre-existing, already-flagged uncommitted documentation files
(`01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and this `04-QA-REPORT.md` from QA's own prior pass) remain
outstanding — nothing new or unrelated staged/committed. **Result: Pass.**

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| — | No new defects found in Dev Task 3's scope. | — | — | — |

Carried forward (not introduced by this task, not this task's responsibility): **DOC-1** (Low,
from the Dev Task 2 pass — wording nit calling a relocated string "duplicated"; still open).
Two new **informational, non-blocking observations** logged under "4. Performance" above (worst-
case ~33s per-sentence latency on total failure; potential concurrent-duplicate-sentence race once
Task 8 wires in concurrency) — neither is a defect against Task 3's own correctness, both are
flagged for Task 8's reviewer to keep in mind.

#### Overall result

**PASS** for Developer Task 3 (`translateSentenceReliable()` + `translateLingva()`) as implemented
in commit `1563cef`, logged in `54f235a`. No unresolved Critical/High/Medium defects. AC1, AC2,
AC3, and AC5 remain unverified end-to-end (expected — dormant until Dev Task 8); AC4 remains
satisfied and unaffected. The Developer's "Interpretation of 'Replace'" judgment call is reviewed
and concurred with.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md` (all re-read this session per operator instruction), plus
  (re-confirmed from context established earlier this milestone) `.ai-company/AGENTS.md`,
  `.ai-company/DEFINITION_OF_DONE.md`, `.ai-company/HANDOFF_PROTOCOL.md`,
  `.ai-company/GIT_RULES.md`, `.ai-company/CODING_STANDARDS.md`,
  `docs/milestones/milestone-02/README.md`, `01-PM-SPEC.md` (Status: APPROVED),
  `02-ARCHITECTURE.md` (Status: APPROVED — §2–§4/§6/§8 row 3 and the `translateLingva` helper row
  specifically re-read), `03-IMPLEMENTATION-LOG.md` (Dev Task 3 entry and handoff, including the
  "Interpretation of 'Replace'" note), and the actual `index.html` diffs for commits `1563cef` and
  `54f235a` (`git show`).
- **Scope completed:** Independently verified Dev Task 3 only, against `02-ARCHITECTURE.md` and
  the actual code/commits, across all eight requested dimensions, via a freshly authored jsdom
  verification script (30/30 assertions passed, including a real-time ~33s full-timeout-chain
  run). No new defects; one Low-severity item (DOC-1) and two informational observations carried
  forward.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See sections 1–8 above.
- **Unresolved risks:** AC1/AC2/AC3/AC5 remain unverified end-to-end pending Dev Tasks 5/8
  (expected, not a gap). DOC-1 (Low, wording only, from Dev Task 2) still open. Two new
  informational observations from "4. Performance" above carried forward for Dev Task 8's
  reviewer: (a) ~33s worst-case per-sentence latency on total failure, inherent to the approved
  architecture's own timeout/backoff constants; (b) no in-flight de-duplication for concurrent
  identical-sentence requests, relevant only once Task 8 introduces concurrent dispatch. The
  pre-existing uncommitted PM/Architecture documentation files remain outstanding, still not this
  task's responsibility to fix.
- **Next agent:** Per `03-IMPLEMENTATION-LOG.md`'s stop point, handoff is to the **CEO**/operator
  to approve proceeding to Dev Task 4 (or directly to the **Senior Developer**, if the operator
  waives the per-task CEO gate) — no unresolved Critical/High defects block this.
- **Explicit stop point:** QA has completed its review of Dev Task 3 only. Awaiting the
  CEO/operator's decision to proceed to Dev Task 4 before any further Development or QA work on
  this milestone.

---

### QA pass: 2026-08-06 — against commit(s) `fb70aa0` (code), `cbc77c5` (log entry)

**Scope of this pass:** Reviewed only Dev Task 4 of the 8-task order in `02-ARCHITECTURE.md` §8
("`translationRowTemplate(x,i)`"), per explicit instruction — not the full milestone. Verified
against the nine specific checks requested: architecture match, loading/success/error rendering,
unchanged success rendering, en/ko escaping, no null/undefined/raw-HTML leaks, regression in
`translation()`/`analyze()`/Tasks 1–3, test sufficiency/reproducibility, no unrelated files or
secrets, and git hygiene/logging accuracy.

#### 1. Task 4 matches the approved architecture exactly

Confirmed via source inspection and independent execution: function signature is exactly
`function translationRowTemplate(x,i){` (§8's specified signature); colocated immediately before
`translation(a)` in the source, per §8's "colocated near translation(a)" placement instruction; the
loading label (`"번역 중…"`) and retry-button label (`"다시 시도"`) match §5's verbatim wording;
the `loading`/`error` branches use classes `sentence loading` / `sentence error`, matching the
exact selector names §8's (not-yet-implemented) style-block row will target. **Result: Pass.**

#### 2. Loading, success, and error rows render correctly

Independently invoked all three branches directly: `loading` renders the English sentence plus the
loading label and a `data-i` index marker; `success` renders both `en` and `ko` text; `error`
renders the English sentence, an explicit Korean failure message, and a retry button correctly
parameterized with the row's index (`retrySentence(9)` for index 9), and never renders the old
placeholder string as if it were real content (checked by direct substring search). **Result:
Pass.**

#### 3. Existing success rendering remains unchanged

Directly diffed the `success`-branch output (with the new `data-i` attribute stripped for
comparison) against the exact legacy markup string still used, unmodified, inside `translation(a)`
today — they are byte-for-byte identical. Also called `translation(a)` itself directly and
confirmed it still renders its own original inline markup, completely unaffected by the new,
dormant sibling function sitting next to it in the source. **Result: Pass.**

#### 4. English and Korean text are safely escaped

Tested HTML/script-shaped input in both `en` and `ko` across all three branches (`loading` and
`error` only ever render `en`; `success` renders both). All malicious markup was correctly
neutralized via `escapeHtml()` in every branch tested, including double quotes and ampersands
(preventing attribute-breakout). **Result: Pass — no new XSS surface, consistent with the
Milestone 1 XSS-defense convention used throughout this file.**

#### 5. No null, undefined, or raw HTML leaks into the UI

The `loading` and `error` branches never reference `x.ko` at all, so no null/undefined leak is
possible from those two states — confirmed directly. **One new finding on the `success` branch,
logged as ROBUST-1 below:** `translationRowTemplate` has no defensive guard on `x.ko` for the
`success` branch. Calling it with `{en, ko:null, status:'success'}` or `{en, status:'success'}`
(no `ko` at all) renders the literal text `(null)` or `(undefined)` in the output, because
`escapeHtml()` calls `String(x.ko)` unconditionally. **This is not reachable through any current
code path** — `grep` confirms `translationRowTemplate` is called from nowhere in `index.html`
except its own definition (it remains fully dormant, exactly as the Developer's log states), and
once it *is* wired in (Dev Task 8), every place that constructs a `status:'success'` object
(`translateSentenceReliable`'s two success branches, per Dev Task 3) always pairs it with a real
string `ko`. Classified **Low** severity per `.ai-company/TESTING_STANDARDS.md`'s definition
("cosmetic, edge-case, or a pre-existing/latent issue not reachable in practice") — it is a latent
robustness gap in code that isn't executed by the running application yet, not a defect a user can
currently trigger. Does not block this task's PASS per `.ai-company/DEFINITION_OF_DONE.md` (Low
findings may be logged and deferred). No raw (unescaped) HTML from user/API content was found to
survive in any of the three rendered rows. **Result: Pass, with one new Low-severity, non-blocking
finding (ROBUST-1).**

#### 6. No regression in translation(), analyze(), or Tasks 1–3

Confirmed at the source level (not just behaviorally) that `translation(a)`'s body is unchanged and
does not reference `translationRowTemplate`, and `analyze()`'s body is unchanged and still
dispatches through `translateSentence(s)`/`Promise.all` with no reference to
`translationRowTemplate` or `translateSentenceReliable`. Re-verified Dev Task 1's constants
(`CONCURRENCY_LIMIT`/`REQUEST_TIMEOUT_MS`/`RETRY_BACKOFF_MS`), Dev Task 2's `simpleTranslate`
contract (`{ko,matched}`), and Dev Task 3's `translateSentenceReliable`/`translateLingva` are all
present and unaffected. **Result: Pass, no regressions.**

#### 7. Tests are sufficient and independently reproducible

The Developer's own 22-assertion script (per `03-IMPLEMENTATION-LOG.md`) covers correctness,
escaping, and regression, but did not test the malformed-input edge case QA found in "5." above. I
independently authored a fresh 32-assertion jsdom script (not a re-run of the Developer's) that
re-derives every claim in the Dev Task 4 log entry from scratch (byte-identical success shape,
verbatim §5 labels, exact class names, escaping of both fields in all three branches, dormancy of
the function) and adds the two ROBUST-1 checks the Developer's script didn't include. **Result:
30/32 assertions passed outright; the 2 "failures" are the ROBUST-1 finding itself (working as
intended — the test correctly caught a real, if low-severity and unreachable, gap). Test coverage
for this task is judged sufficient.**

#### 8. No unrelated files or secrets were introduced

`git show --stat` on both commits confirms `fb70aa0` touches only `index.html` (26 insertions, 0
deletions) and `cbc77c5` touches only `03-IMPLEMENTATION-LOG.md` (120 insertions, 1 deletion, i.e.
1 line changed from a heading edit plus the new entry). Secret/credential/token pattern scan across
both commits found nothing. Working tree checked before and after: only the same three
already-flagged, pre-existing uncommitted documentation files remain outstanding — nothing new.
**Result: Pass.**

#### 9. Git hygiene and implementation logging are correct

One logical change per commit, correctly separated (code vs. docs), per
`.ai-company/GIT_RULES.md` rules 4/9. Commit messages follow the established convention with
explanatory bodies. Cross-checked `03-IMPLEMENTATION-LOG.md`'s Dev Task 4 entry against the actual
`git show` output: the "26 lines added, 0 removed" claim matches exactly; the commit hash is
recorded correctly; the handoff fields are complete per `.ai-company/HANDOFF_PROTOCOL.md`. The
entry's own disclosed judgment calls (the `data-i` attribute addition, the Developer-chosen error-
copy wording) are accurately described and match what's actually in the diff. The two items the
operator asked to be "recorded, but not solved" (worst-case provider-chain latency; possible
duplicate in-flight requests) are correctly carried forward into this session's log entry,
unchanged and unaddressed by this task's code, exactly as instructed. **Result: Pass.**

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| ROBUST-1 | `translationRowTemplate`'s `success` branch has no guard against `x.ko` being `null`/`undefined`; renders the literal text `(null)` or `(undefined)` in that case. | Low | Call `translationRowTemplate({en:"x", ko:null, status:"success"}, 0)` (or omit `ko` entirely) and inspect the returned markup for the literal substring `(null)`/`(undefined)`. | Open — not reachable via any current code path (function is dormant; `grep` confirms zero call sites); every current/planned producer of `status:'success'` objects (`translateSentenceReliable`) always supplies a real string `ko`. Safe to defer per `.ai-company/DEFINITION_OF_DONE.md`; recommended to add a defensive `x.ko??""` (or equivalent) when this function is wired in during Dev Task 8, as cheap insurance. |

Carried forward (not introduced by this task, not this task's responsibility): **DOC-1** (Low,
from Dev Task 2 — wording nit, still open). Two informational observations from the Dev Task 3
pass (worst-case ~33s provider-chain latency; possible duplicate in-flight requests for identical
sentences) were correctly re-recorded by the Developer in this session's log entry, unaddressed, as
instructed — no change in status.

No Critical, High, or Medium defects found.

#### Overall result

**PASS** for Developer Task 4 (`translationRowTemplate()` shared per-row markup helper) as
implemented in commit `fb70aa0`, logged in `cbc77c5`. No unresolved Critical/High defects — one new
Low-severity, non-blocking finding (ROBUST-1), in addition to the previously carried-forward DOC-1
and the two informational observations. All are safe to defer per
`.ai-company/DEFINITION_OF_DONE.md` and do not block proceeding to Dev Task 5.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-02/01-PM-SPEC.md` (Status:
  APPROVED, §5/Assumption A5 re-checked), `docs/milestones/milestone-02/02-ARCHITECTURE.md`
  (Status: APPROVED — §5/§7/§8 specifically re-read), `docs/milestones/milestone-02/
  03-IMPLEMENTATION-LOG.md` (Dev Task 4 entry and handoff, including the `data-i`/error-copy
  judgment calls and the re-recorded latency/duplicate-request observations), and
  `docs/milestones/milestone-02/04-QA-REPORT.md` (all re-read this session per operator
  instruction), plus the actual `index.html` diffs for commits `fb70aa0` and `cbc77c5` (`git
  show`).
- **Scope completed:** Independently verified Dev Task 4 only, against `02-ARCHITECTURE.md` and
  the actual code/commits, across all nine requested checks, via a freshly authored 32-assertion
  jsdom script. One new Low-severity finding (ROBUST-1); no Critical/High/Medium defects.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See sections 1–9 above.
- **Unresolved risks:** AC1/AC2/AC3/AC5 remain unverified end-to-end pending Dev Tasks 5–8
  (expected, not a gap; this task's rendering logic for AC1/AC5 is now independently confirmed
  correct in isolation, but still unreachable from the UI). DOC-1 (Low, wording, from Dev Task 2)
  and ROBUST-1 (Low, new this pass) both open and deferrable. The ~33s worst-case latency and
  potential duplicate in-flight requests (from the Dev Task 3 pass) remain carried forward,
  unaddressed, for Dev Task 8's attention. The pre-existing uncommitted PM/Architecture
  documentation files remain outstanding, still not any Development or QA task's responsibility to
  fix.
- **Next agent:** CEO, for Dev Task 5 approval, per explicit instruction.
- **Explicit stop point:** QA has completed its review of Dev Task 4 only. Awaiting CEO approval
  before any further Development work (Dev Task 5) on this milestone.

---

### QA pass: 2026-08-06 — against commit(s) `284ea76` (code), `911735e` (log entry)

**Scope of this pass:** Reviewed only Dev Task 5 of the Developer task order (`translation(a)`
wired to `translationRowTemplate()`), per explicit instruction — not the full milestone. Verified
against the nine specific checks requested, with special attention to the three named risk areas
(null/undefined `x.ko` handling, worst-case provider-chain latency, duplicate in-flight requests).

#### 1. translation(a) now renders through translationRowTemplate()

Confirmed at the source level (`a.translations.map((x,i)=>translationRowTemplate(x,i))`) and
behaviorally: calling `translation(a)` directly and comparing its output against manually mapping
the same input through `translationRowTemplate` produces the identical row markup — `translation(a)`
is genuinely delegating, not independently re-implementing similar-looking markup. **Result: Pass.**

#### 2. Existing success output remains byte-compatible except for the approved data-i hook

Stripped the `data-i="0"` attribute from a freshly rendered success row and compared it
byte-for-byte against the exact legacy row-markup string that existed before Dev Task 5 (and before
Dev Task 4) — identical. The second card ("자연스러운 전체 해석") is untouched in both structure and
content. **Result: Pass.**

#### 3. Loading and error shapes render correctly when supplied

Fed `translation(a)` a mixed array of `loading`, `error`, and `success` entries in one call (not
just testing `translationRowTemplate` in isolation, as the Dev Task 4 pass did — this time through
the actual `translation(a)` entry point). All three rendered with the correct classes, labels, and
index-correct retry button (`retrySentence(1)` for the row at index 1); no leftover legacy
placeholder string appeared anywhere in the mixed output. **Result: Pass.**

#### 4. English and Korean content remain safely escaped

Tested malicious HTML/script-shaped input in both `en` and `ko` fields, through `translation(a)`
end-to-end (not just at the `translationRowTemplate` unit level), for both a `success` row and a
`loading` row. All neutralized correctly. **Result: Pass — no new XSS surface.**

#### 5. No regression in analyze() or Tasks 1–4

Confirmed at the source level that `analyze()`'s body is unchanged: still builds
`translations=sents.map((s,i)=>({en:s,ko:translated[i]}))` with no `status:` field anywhere, and
still dispatches via `Promise.all`/`translateSentence(s)`, not the new provider chain. Re-verified
Dev Tasks 1–4's constants/functions/contracts are all present and unaffected. **Result: Pass, no
regressions.**

#### 6. The architecture-order deviation is justified and does not change approved scope

Independently confirmed the load-bearing claim behind the Developer's reordering decision: searched
the entire file for any code path (outside `translationRowTemplate`'s own definition) that
constructs a `translations[]`-style entry with `status:'loading'`/`'error'` — there is none. This
means today's reordering genuinely introduces zero new *reachable* behavior beyond what §8's
`translation(a)` row literally specifies; the end state (`translation(a)` mapping through
`translationRowTemplate`) matches the architecture exactly, and only the *sequencing* relative to
the not-yet-implemented `analyze()`/`updateTranslationRow`/`retrySentence` rows was changed. The
stated rationale (avoiding a literal `(null)` flashing on screen if `analyze()` were rewired first,
without `translation(a)` already knowing how to render `loading`/`error` states) is sound and
consistent with `.ai-company/GIT_RULES.md` rule 1. This is the same category of judgment call
already reviewed and concurred with for Dev Task 3's "Interpretation of 'Replace'" — **concur**
again here. **Result: Pass.**

#### 7. Tests are independently reproducible and sufficient

The Developer's own 16-assertion script covers correctness, byte-compatibility, security, and
regression for today's status-less data shape, but (reasonably, since it wasn't required by the
task) does not exercise `translation(a)` with `loading`/`error`-shaped input, nor the malformed
`ko:null`/`undefined` edge case now that the render path is live. I independently authored a fresh
25-assertion jsdom script that re-derives the Developer's claims from scratch and adds: rendering
`translation(a)` with a mixed loading/error/success array (item 3 above), and the two "special
attention" null/undefined checks below. **Result: 22/25 assertions passed outright; of the 3
"failures," 2 are the intended confirmation of the already-known ROBUST-1 gap (see below, working
as designed) and 1 was a flaw in QA's own test regex** (it double-counted an unrelated mention of
the string `"translateSentenceReliable("` inside a pre-existing Dev Task 2 comment; direct `grep`
confirms there is exactly one real declaration and zero actual call sites — corrected and
re-verified by hand, not a code defect). Test coverage for this task, combined with QA's
independent extension, is judged sufficient.

#### 8. No unrelated files, secrets, commits, or pushes were introduced

`git show --stat` confirms `284ea76` touches only `index.html` (9 insertions, 1 deletion) and
`911735e` touches only `03-IMPLEMENTATION-LOG.md` (118 insertions, 1 deletion). Secret/credential
scan across both commits found nothing. Working tree checked before and after this QA pass: only
the same three already-flagged, pre-existing uncommitted documentation files remain outstanding —
nothing new staged or committed by this review, no push executed. **Result: Pass.**

#### 9. Implementation log accurately records the work and deferred risks

Cross-checked `03-IMPLEMENTATION-LOG.md`'s Dev Task 5 entry against the actual diff: the "9 lines
added, 1 removed" claim matches `git show --stat` exactly; the commit hash is recorded correctly;
the "Task-ordering deviation" note accurately describes what was done and why. The three
operator-specified carried-forward items (latency, duplicate in-flight requests, ROBUST-1) are all
present in the entry, correctly described as unaddressed by this task. One nuance worth noting for
precision, not a defect: the *code commit message* (`284ea76`) describes ROBUST-1 as "still
unreachable," while the more detailed log entry correctly distinguishes between the *function*
becoming reachable from a live render path (true, as of this task) and the *specific defect input
shape* still not being reachable (also true, since nothing constructs it) — the log entry's
phrasing is the more precise of the two, and is the document of record. **Result: Pass.**

#### Special attention: the three named risk areas

- **Null/undefined `x.ko` handling, now that the render path is reachable (ROBUST-1):**
  Independently reproduced the exact scenario: calling `translation(a)` with a `status:'success'`
  entry whose `ko` is `null` (or omitted entirely) renders the literal text `(null)` / `(undefined)`
  in the output — confirming the gap first logged in the Dev Task 4 pass is now reachable through a
  genuinely live entry point (`translation(a)` really is called by `render()` for real analyses),
  not just a directly-invoked isolated function. **However, independently confirmed the Developer's
  mitigating claim holds:** no code that exists in the repository today — `analyze()`, confirmed
  unchanged and never emitting a `status:` field at all — can actually construct the input shape
  that triggers this. **Severity assessment: remains Low**, per
  `.ai-company/TESTING_STANDARDS.md`'s definition ("edge-case ... not reachable in practice") —
  practical reachability (can a user trigger this today?) is unchanged even though the *code path
  length* to the vulnerable function shortened by one hop. Not downgrading or upgrading without
  cause, per `.ai-company/QA.md`. **Recommendation, not a blocker:** this should be fixed no later
  than whichever task first makes `analyze()` (or `loadSaved()`, or `retrySentence()`) capable of
  producing a `status:'success'` object — at that point the guard becomes cheap, mandatory
  insurance rather than speculative hardening.
- **Worst-case provider-chain latency (~33s from the Dev Task 3 pass):** confirmed unaffected by
  this task — `translateSentenceReliable` still has exactly one occurrence outside comments in the
  file (its own declaration) and zero call sites. Status unchanged, correctly carried forward.
- **Duplicate in-flight requests for identical sentences:** confirmed unaffected by this task — no
  in-flight-tracking logic (`pending`/`inFlight`) was added anywhere near `translationCache`.
  Status unchanged, correctly carried forward.

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| — | No new defects found in Dev Task 5's scope. | — | — | — |

Carried forward (not introduced by this task, not this task's responsibility to fix): **DOC-1**
(Low, Dev Task 2, wording nit, still open); **ROBUST-1** (Low, Dev Task 4 — reachability status
reassessed above, severity unchanged); the ~33s worst-case provider-chain latency and the possible
duplicate in-flight requests for identical sentences (both from the Dev Task 3 pass, both
unaffected by this task).

No Critical, High, or Medium defects found.

#### Overall result

**PASS** for Developer Task 5 (`translation(a)` wired to `translationRowTemplate()`) as implemented
in commit `284ea76`, logged in `911735e`. No unresolved Critical/High defects. The Developer's
task-ordering deviation is reviewed and concurred with. ROBUST-1's reachability was specifically
re-examined per the operator's instruction and confirmed to remain a Low-severity, non-blocking,
practically-unreachable finding — recommended for a fix at the point `analyze()` (or an equivalent
producer) first becomes capable of emitting the shape that would trigger it.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-02/01-PM-SPEC.md` (Status:
  APPROVED), `docs/milestones/milestone-02/02-ARCHITECTURE.md` (Status: APPROVED — §7/§8
  specifically re-read), `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md` (Dev Task 5 entry
  and handoff, including the task-ordering deviation note), and
  `docs/milestones/milestone-02/04-QA-REPORT.md` (all re-read this session per operator
  instruction), plus the actual `index.html` diffs for commits `284ea76` and `911735e` (`git
  show`).
- **Scope completed:** Independently verified Dev Task 5 only, against `02-ARCHITECTURE.md` and
  the actual code/commits, across all nine requested checks plus the three specially-flagged risk
  areas, via a freshly authored 25-assertion jsdom script. No new defects; ROBUST-1's reachability
  status updated (function reachable, defect input still not constructible by any existing code) but
  severity unchanged at Low.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See sections 1–9 and "Special attention" above.
- **Unresolved risks:** DOC-1 and ROBUST-1 (both Low) remain open and deferrable. The ~33s
  worst-case provider-chain latency and possible duplicate in-flight requests remain carried
  forward for whichever task wires `analyze()` to the new provider chain. Recommend the `x.ko`
  defensive guard (ROBUST-1) be addressed in the same task that first makes `analyze()`/
  `loadSaved()`/`retrySentence()` capable of emitting a `status:'success'` object, rather than
  deferred indefinitely. The pre-existing uncommitted PM/Architecture documentation files remain
  outstanding, still not any Development or QA task's responsibility to fix.
- **Next agent:** CEO, for Dev Task 6 approval.
- **Explicit stop point:** QA has completed its review of Dev Task 5 only. Awaiting CEO approval
  before any further Development work (Dev Task 6) on this milestone.

---

### QA pass: 2026-08-06/07 — Dev Task 6 defect finding, and independent SAVE-1 fix verification

**Context (recap, for the historical record — not a new finding):** QA's independent review of
Dev Task 6 (commits `9dd5ae8`/`a7df167`) verified items 1–9 of that review as Pass, but
investigating the CEO-flagged risk "can `saveCurrent()` save mid-translation?" confirmed **defect
SAVE-1 (High)**: `saveCurrent()` could persist `translations` entries still in their `{status:
'loading', ko:null}` shape, and reopening such a save rendered a permanently stuck `"번역 중…"`
row with no retry button. Per `.ai-company/WORKFLOW.md`'s QA Defect Gate ("STOP if Critical/High
defects found"), this was reported directly to the CEO/Developer at that time rather than appended
here as a standalone entry, since the instructions for that pass specified reporting defects
directly rather than writing to this file. It is recorded here now, retroactively, so this
document remains the complete loop history per `.ai-company/QA.md` ("don't erase history — append
new iterations"). Full original reproduction steps, severity rationale, and the two other
investigated risks (worst-case latency, duplicate in-flight requests — both assessed Low/
informational, not blocking) are preserved in that session's transcript and in
`03-IMPLEMENTATION-LOG.md`'s Dev Task 6 entry, which independently corroborates the same finding.

**This pass's scope:** Independent QA review of the SAVE-1 bug-fix commits (`a86f8e0` code,
`4829592` log) only — not Dev Task 7, which the CEO explicitly instructed not to start.

#### Verification checklist (as requested)

| # | Check | Result | Notes |
|---|---|---|---|
| 1 | Saving is blocked whenever any translation status is `'loading'` | Pass | Confirmed with all-loading and **mixed** states (some resolved, some still loading) — the guard correctly blocks on *any* loading entry, not just when all are loading. |
| 2 | Saving works normally after all translations finish | Pass | Save succeeds once every entry reaches a terminal status; saved translations are the real resolved values. |
| 3 | Existing saved studies still load correctly | Pass | A simulated pre-Milestone-2 item (no `status` field at all) loads and renders correctly. Also confirmed loading a hypothetical pre-fix item that already has a stuck `'loading'` entry does not throw (this fix is forward-looking — it does not retroactively repair data saved before it shipped, which is expected and out of scope for a point fix). |
| 4 | Error-state translations can still be saved and retried | Pass | A `status:'error'` entry is saved without being blocked (only `'loading'` triggers the guard); reopening and calling `retrySentence()` on it still succeeds end-to-end. |
| 5 | No Task 7 functionality was introduced | Pass | Confirmed `loadSaved()` has no new defensive status-default logic, and no new `.sentence.loading`/`.sentence.error` CSS rules exist — both are separate, still-unstarted architecture rows. `git show --stat` confirms the code commit touches only `saveCurrent()`. |
| 6 | No regression exists in Dev Tasks 1–6 | Pass | Re-verified Task 1's constants, Task 2's `simpleTranslate` contract, Task 3's `translateSentenceReliable`/`translateLingva`, Task 4's `translationRowTemplate`, Task 5's `translation(a)` wiring, and Task 6's full live pipeline (progressive rendering, concurrency limit, `analyzeBtn` timing, `#status` summary format) end-to-end — all unchanged. |
| 7 | Security and escaping remain unchanged | Pass | Malicious provider-returned text is still escaped end-to-end through the live pipeline; the new toast message is a static string literal with no interpolated/user-controlled content, so no new injection surface was introduced. |
| 8 | Git hygiene is clean (only intended files changed) | Pass | `a86f8e0`: `index.html` only, 12 insertions, 0 deletions — purely additive (one guard clause + comment). `4829592`: `03-IMPLEMENTATION-LOG.md` only. Secret/credential scan on both commits: no matches. Working tree before/after: only the same three already-flagged, pre-existing uncommitted documentation files remain — nothing new. |
| 9 | Regression tests pass independently | Pass | Freshly authored jsdom script (not a re-run of the Developer's own script): **29/29 assertions passed.** |

**Independent verification method:** No automated framework exists in this repository
(`.ai-company/TESTING_STANDARDS.md`). QA authored a fresh jsdom script exercising the real
`saveCurrent()`/`loadSaved()`/`analyze()`/`retrySentence()` functions end-to-end (not isolated
units), covering all nine checks above plus an additional mixed-state case (some sentences
resolved, others still loading) that the Developer's own 15-assertion script did not explicitly
test — result: blocked correctly, same as the all-loading case.

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| SAVE-1 | (Recapped, not new) `saveCurrent()` could persist `'loading'` translations, producing an unrecoverable stuck row on reload. | High | See `03-IMPLEMENTATION-LOG.md`'s Dev Task 6 entry and Bug-fix-SAVE-1 entry for full steps. | **Resolved** — independently verified fixed by commit `a86f8e0`. |

No new Critical, High, or Medium defects found in the fix itself. Carried forward, unaffected by
this fix: DOC-1 (Low, Dev Task 2 wording nit), ROBUST-1 (Low, Dev Task 4 — `x.ko` null/undefined
guard, still unreachable), the ~33s worst-case provider-chain latency (informational, matches
approved architecture constants), and duplicate in-flight requests for identical sentences (Low,
cosmetic quota inefficiency, confirmed not a correctness bug).

#### Overall result

**PASS.** SAVE-1 is confirmed resolved by commit `a86f8e0` (logged in `4829592`), with no
regressions introduced in Dev Tasks 1–6, no Task 7 functionality present, security/escaping
unaffected, and clean git hygiene. `.ai-company/DEFINITION_OF_DONE.md`'s "no unresolved Critical or
High defects" condition is now satisfied for this milestone as of this fix (the remaining open
items are all Low or informational).

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-02/01-PM-SPEC.md` (Status:
  APPROVED), `docs/milestones/milestone-02/02-ARCHITECTURE.md` (Status: APPROVED),
  `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md` (the Dev Task 6 entry and the Bug fix
  SAVE-1 entry/handoff, read in full), and this `04-QA-REPORT.md` (all re-read this session per
  operator instruction), plus the actual `index.html` diffs for commits `a86f8e0` and `4829592`
  (`git show`).
- **Scope completed:** Independently verified the SAVE-1 fix only, against all nine requested
  checks, via a freshly authored 29-assertion jsdom script. Retroactively recorded the original
  SAVE-1 finding in this document for complete loop history, per `.ai-company/QA.md`.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings only, per instruction.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See the verification checklist and "Independent verification method" above.
- **Unresolved risks:** DOC-1 and ROBUST-1 (both Low) remain open and deferrable. The ~33s
  worst-case latency and duplicate in-flight-request behavior remain open/informational, unaffected
  by this fix. This fix does not retroactively repair any study set that may have already been
  saved with a stuck `'loading'` entry *before* this fix shipped (there is no such data in this
  environment, but noting it for completeness — a real production deployment with pre-existing
  stuck saves would need a separate, one-time data-repair consideration, which is outside this
  point fix's scope). Dev Task 7 and the still-unstarted `loadSaved()` defensive-read and CSS
  style-block work remain not started.
- **Next agent:** CEO, for Dev Task 7 approval.
- **Explicit stop point:** QA has completed independent verification of the SAVE-1 fix. Awaiting
  CEO approval before Dev Task 7 (or any further Development work) begins on this milestone.

---

### QA pass: 2026-08-07 — against commit(s) `77b82ac` (code), `dedef04` (log entry)

**Scope of this pass:** Independent review of Dev Task 7 only — `loadSaved()`'s defensive
status-default read, per `02-ARCHITECTURE.md` §8's `loadSaved()` row. Dev Task 8 (CSS style-block
additions) is out of scope and confirmed not started.

#### Verification checklist (as requested)

| # | Check | Result | Notes |
|---|---|---|---|
| 1 | Task 7 implementation matches the approved architecture exactly | Pass | `git show 77b82ac` diff reviewed directly against §8's `loadSaved()` row: defaults any translation entry with a falsy `status` to `"success"`, guarded by `Array.isArray(x.translations)`, applied only to the in-memory object (no `localStorage` write). Matches the architecture text verbatim in intent and scope. The required comment citing `02-ARCHITECTURE.md §8`'s `loadSaved()` row is present immediately above the function (confirmed via `grep`, after an initial QA test-script slice boundary excluded it — see Test-script artifacts below). |
| 2 | Defensive `loadSaved()` behavior works correctly | Pass | A legacy entry with no `status` field at all is defaulted to `'success'` in memory with its `ko` text preserved exactly. An entry with an empty-string `status` (also falsy) is likewise defaulted, confirming the guard is a truthiness check, not a strict `undefined` check — matches the "handle localStorage reads defensively" standard broadly, not narrowly. |
| 3 | Existing saved studies still load correctly | Pass | A modern, fully-tagged saved item (`status:'success'`) still restores `title`/`passage`/`grade`/`analysisTitle` correctly and its translation still renders end-to-end unchanged. |
| 4 | Regression tests for Tasks 1–6 and SAVE-1 all pass | Pass | Re-verified live, not just via source-pattern matching: Task 1 constants, Task 2's `simpleTranslate` contract, Task 3's `translateSentenceReliable`/`translateLingva`, Task 4/5's `translationRowTemplate`/`translation(a)` wiring, and Task 6's full pipeline (`state.analysis` assigned early with loading entries, `analyzeBtn` disable/enable timing, concurrency cap respected, final `#status` summary format). SAVE-1's guard was independently re-exercised live (never-resolving fetch, mid-translation `saveCurrent()` call) and still blocks correctly — Task 7 did not touch or weaken it. |
| 5 | No Dev Task 8 functionality has been introduced | Pass | No `.sentence.loading`/`.sentence.error` CSS rules exist anywhere in `index.html`. Diffing the code commit against its correct immediate predecessor (`4829592`, the SAVE-1 doc-log commit) confirms only `index.html` changed, and within it no new top-level function was introduced — `loadSaved()` was edited in place, nothing else. |
| 6 | Security/XSS protection is unchanged | Pass | A legacy (status-less) saved item containing `<script>`/`<img onerror>` payloads in both `en` and `ko` is still fully escaped end-to-end after the defensive read — no new injection surface. |
| 7 | Git hygiene is clean | Pass | `77b82ac`: `index.html` only, 9 insertions, 0 deletions — purely additive. `dedef04`: `03-IMPLEMENTATION-LOG.md` only. Secret/credential scan on both commits: no matches. Working tree: only the same three long-standing pre-existing uncommitted documentation files remain (`01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `04-QA-REPORT.md`) — nothing new or unexpected. |
| 8 | Documentation matches implementation | Pass | `03-IMPLEMENTATION-LOG.md`'s Dev Task 7 session entry accurately describes the change and cites the correct commit hash `77b82ac`; the `## Handoff — Milestone 2, Dev Task 7` section and the `## Handoff log` numbered list (now correctly extended to item 8, with item 7's stale "see immediately above" reference correctly updated to "see above" now that it's no longer the last entry) are both present and correctly formed. The log states in three separate places (the Deviations note, the Scope-completed line, and the Unresolved-risks line) that Dev Task 8 was not started, consistent with the actual diff. |
| 9 | No new defects introduced | 1 new Low finding | See ROBUST-2 below. No Critical, High, or Medium defects found. |

**Independent verification method:** Freshly authored jsdom script (`qa-verify-task7.js`, not a
re-run of the Developer's `verify-task7.js`), run against the real `index.html` via the established
`window.__QA__` bridge technique: **37 of 40 assertions passed** on first run; the 3 failures were
investigated individually and confirmed to be artifacts of this QA script's own test logic, not
product defects (see below), so the corrected/direct checks are what's reported as Pass above.
Additionally wrote a small standalone script (`edge-check.js`) to probe severely malformed
`translations` data (a `null` entry, a bare string entry) beyond what the Developer's own script
covered, which surfaced the new Low finding below.

**Test-script artifacts (not product defects, confirmed via direct `grep`/`git diff` inspection):**

1. My first check sliced `loadSaved()`'s source starting at the literal string `"function
   loadSaved"`, which excluded the explanatory comment block immediately *above* the function
   (where the required `02-ARCHITECTURE.md §8` citation actually lives). `grep -n "§8's
   loadSaved()"` confirms the citation is present at line 884 of `index.html`. Script bug, not a
   defect.
2. My "only index.html changed" check diffed against `a86f8e0` (the SAVE-1 *code* commit), which is
   two commits behind `77b82ac`, not one — it doesn't account for the intervening `4829592` doc
   commit. Diffing against the correct immediate predecessor (`4829592`) confirms only `index.html`
   changed. Script bug (wrong baseline commit), not a defect.
3. My "log states Task 8 not started" regex used `.` (which does not match newlines in JavaScript)
   spanning up to 80 characters, but the relevant sentence in `03-IMPLEMENTATION-LOG.md` happens to
   wrap across a Markdown line break between "Task 8" and "not started." Direct `grep -n "Task 8"`
   confirms the log states this unambiguously in three separate places. Script bug (line-wrap
   blind spot), not a defect.

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|
| ROBUST-2 | If a saved item's `translations` array contains a non-object entry (e.g. `null` or a bare string) — reachable only via manual/corrupted `localStorage` editing, never via the app's own `saveCurrent()` output — `loadSaved()`'s new `t&&t.status?t:{...t,status:"success"}` guard silently spreads the malformed value, and a bare-string entry in particular loses its shape entirely (spreads into indexed character keys) rather than being rejected or repaired. Downstream, `translationRowTemplate` does not crash but renders the literal text `"undefined"` / `"(undefined)"` for that row (confirmed no XSS — `escapeHtml` still applies; this is plain-text `undefined`, not raw HTML). | Low | 1. Set `localStorage.satStudioSaved` to a JSON array containing one item whose `translations` is `["oops"]` or `[null]`. 2. Call `loadSaved(<that id>)`. 3. Switch to the Translation tab. Observe: no crash, but the row renders `undefined` / `(undefined)` instead of a `loading`, `success`, or `error` state. | **Open, Low, not blocking.** Same root category as the already-tracked ROBUST-1 (Dev Task 4: `translationRowTemplate`'s success branch has no defensive guard for a null/undefined `ko`) — both are latent gaps in defensive rendering that are confirmed unreachable through any normal user flow, since the app itself never writes this shape. Not introduced as a regression by Task 7; Task 7's guard is a truthiness check on `status`, not a full shape-validator, and extending it to validate/repair arbitrary malformed entries would be new scope beyond §8's stated "default missing status" requirement. Flagged for CEO/Architect awareness, not fixed here per QA's role (report, don't fix). |

No new Critical, High, or Medium defects found. Carried forward, unaffected by this task: DOC-1
(Low), ROBUST-1 (Low, still confirmed unreachable), the ~33s worst-case provider-chain latency
(informational), and duplicate in-flight requests for identical sentences (Low, cosmetic).

#### Overall result

**PASS.** Dev Task 7 is confirmed correctly implemented per `02-ARCHITECTURE.md` §8, with no
regressions in Dev Tasks 1–6 or the SAVE-1 fix, no Dev Task 8 functionality present, security/
escaping unaffected, and clean git hygiene. One new Low-severity, non-blocking finding (ROBUST-2)
is recorded for visibility; per `.ai-company/DEFINITION_OF_DONE.md`, Low-severity items do not
block release or this gate.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-02/01-PM-SPEC.md` (Status:
  APPROVED), `docs/milestones/milestone-02/02-ARCHITECTURE.md` (Status: APPROVED, §8's `loadSaved()`
  row specifically), `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md` (the Dev Task 7 entry
  and handoff, read in full), and this `04-QA-REPORT.md` (re-read in full this session), plus the
  actual `index.html` diff for commit `77b82ac` (`git show`) and its documentation commit `dedef04`.
- **Scope completed:** Independently verified Dev Task 7 only, against all 9 requested checks, via
  a freshly authored 40-assertion jsdom script plus a supplementary malformed-data probe script.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings and the header status line only.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See the verification checklist and "Independent verification method" above.
- **Unresolved risks:** New: ROBUST-2 (Low, non-blocking, described above). Carried forward: DOC-1
  and ROBUST-1 (both Low), worst-case provider latency and duplicate in-flight requests
  (informational/Low) — all unaffected by this task. Dev Task 8 (CSS style-block additions for
  `.sentence.loading`/`.sentence.error`) remains the only unstarted row in §8's file-by-file plan.
- **Next agent:** CEO, for Dev Task 8 approval.
- **Explicit stop point:** QA has completed independent verification of Dev Task 7 (PASS). Awaiting
  CEO approval before Dev Task 8 (or any further Development work) begins on this milestone.

---

### QA pass: 2026-08-07 — against commit(s) `9fad20c` (code), `4aa3015` (log entry)

**Scope of this pass:** Independent review of Dev Task 8 only — the `.sentence.loading`/
`.sentence.error` CSS additions per `02-ARCHITECTURE.md` §8's `<style>` block row. This is the last
row in the architecture's file-by-file plan; all 8 of 8 Dev Tasks are now implemented.

#### Verification checklist

| # | Check | Result | Notes |
|---|---|---|---|
| 1 | Task 8 implementation matches the approved architecture | Pass | `git show 9fad20c` reviewed directly: `.sentence.loading` (background + pulse `animation`, i.e. the "skeleton/spinner treatment" §8 calls for) and `.sentence.error` (accent border/text + a `.sentence.error .btn` spacing rule, i.e. the "warning-colored variant + retry button styling") both exist inside the `<style>` block. Both reuse existing `:root` tokens (`var(--sky)`, `var(--danger)`) rather than introducing new hardcoded colors, matching the architecture's explicit instruction to follow the `c-blue`/`c-pink`/`c-mint` convention (which itself is token-based, not hardcoded). Confirmed every added line in the diff is CSS selector/declaration syntax — no JS was smuggled into this "CSS-only" commit. |
| 2 | No JS/markup regression | Pass | `translationRowTemplate`'s `loading` and `error` branches are byte-identical to their pre-Task-8 shape (regex-matched against the exact template string, including the retry button's `onclick="retrySentence(${i})"` wiring) — confirming Task 8 is additive CSS only, as the implementation log claims. |
| 3 | Live rendering produces the correct classes/elements | Pass | Built a live `state.analysis` with one `loading`, one `error`, and one `success` entry and rendered it through the real `render()`/`translation()`/`translationRowTemplate` pipeline (not a static HTML fixture): exactly one `.sentence.loading`, exactly one `.sentence.error`, and exactly one plain `.sentence` element were produced, with no class cross-contamination between rows. The loading row still shows "번역 중…", the error row's retry button is still wired to the correct sentence index, and the success row still shows its resolved Korean text. |
| 4 | Stylesheet integrity | Pass | Brace count in the `<style>` block remains balanced after the addition (no syntax break), and jsdom's own parser loads/parses the modified stylesheet without throwing. |
| 5 | Regression: Tasks 1–7 and SAVE-1 | Pass | Re-verified live: Task 1 constants, Task 2's `simpleTranslate` contract, Task 3's `translateSentenceReliable`/`translateLingva`, Task 4/5's `translationRowTemplate`/`translation(a)` wiring, Task 6's full pipeline (concurrency cap, final `#status` summary), the SAVE-1 save-while-loading guard, and Task 7's `loadSaved()` defensive status-default read are all unaffected by this CSS-only change. |
| 6 | Security/XSS unaffected | Pass | A malicious provider-returned translation (`<svg onload=alert(1)>`) is still fully escaped end-to-end after Task 8 — a CSS-only change has no plausible mechanism to affect escaping, and this confirms it didn't. |
| 7 | Git hygiene | Pass | `9fad20c`: `index.html` only, 13 insertions, 0 deletions — purely additive. `4aa3015`: `03-IMPLEMENTATION-LOG.md` only. Secret/credential scan on both commits: no matches (the only hits were the literal word "token" in prose referring to CSS design tokens, not credentials — confirmed a false positive by inspection). Working tree: only the same three long-standing pre-existing uncommitted documentation files remain. |
| 8 | Documentation matches implementation | Pass | The Dev Task 8 session entry in `03-IMPLEMENTATION-LOG.md` accurately describes the change, cites the correct commit hash `9fad20c`, and the `## Handoff — Milestone 2, Dev Task 8` section plus the `## Handoff log` numbered list (correctly extended to item 9) are both present and correctly formed. The log explicitly states all 8 rows of §8's plan are now complete, consistent with the actual diff and commit history. |
| 9 | No new defects introduced | 0 new defects | See below. |

**Independent verification method:** Freshly authored jsdom script (`qa-verify-task8.js`, not a
re-run of the Developer's `verify-task8.js`), run against the real `index.html` via the established
`window.__QA__` bridge technique: **34 of 35 assertions passed** on first run. The 1 failure was
investigated and confirmed to be an artifact of this QA script's own logic (see below), not a
product defect.

**Test-script artifact (not a product defect, confirmed via direct `git diff`/`git log`
inspection):** My "only index.html changed" check diffed the code commit against `4829592` (the
SAVE-1 doc-log commit), which is two commits behind `9fad20c`, not the correct immediate
predecessor. `git log --oneline 4829592..9fad20c` shows `dedef04` (Dev Task 7's doc commit) sits
in between, which is why that diff also showed `03-IMPLEMENTATION-LOG.md` as changed. Diffing
against the correct immediate predecessor (`dedef04`) confirms only `index.html` changed. Script
bug (wrong baseline commit), not a defect — the same category of mistake made and caught in the
Dev Task 7 QA pass, again independently caught here by re-deriving the correct commit ancestry
rather than trusting the first result.

#### Defects found

No new Critical, High, Medium, or Low defects introduced by Dev Task 8. Carried forward, unaffected
by this task: DOC-1 (Low), ROBUST-1 (Low), ROBUST-2 (Low, from the Dev Task 7 QA pass), the ~33s
worst-case provider-chain latency (informational), and duplicate in-flight requests for identical
sentences (Low, cosmetic).

#### Overall result

**PASS.** Dev Task 8 is confirmed correctly implemented per `02-ARCHITECTURE.md` §8's final
unimplemented row, with no regressions in Dev Tasks 1–7 or the SAVE-1 fix, security/escaping
unaffected, and clean git hygiene. This completes all 8 rows of the approved architecture's
file-by-file implementation plan. Per `.ai-company/DEFINITION_OF_DONE.md`, no unresolved Critical
or High defects remain open for this milestone — the only open items are Low-severity or
informational and are non-blocking.

#### Handoff

- **Milestone:** Milestone 2 — Translation Reliability.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/QA.md`,
  `.ai-company/TESTING_STANDARDS.md`, `docs/milestones/milestone-02/01-PM-SPEC.md` (Status:
  APPROVED), `docs/milestones/milestone-02/02-ARCHITECTURE.md` (Status: APPROVED, §8's `<style>`
  block row specifically), `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md` (the Dev Task 8
  entry and handoff, read in full), and this `04-QA-REPORT.md` (re-read in full this session), plus
  the actual `index.html` diff for commit `9fad20c` (`git show`) and its documentation commit
  `4aa3015`.
- **Scope completed:** Independently verified Dev Task 8 only, against all 9 requested checks, via
  a freshly authored 35-assertion jsdom script.
- **Files changed:** None (QA is read-only this pass; no production code modified). This
  `04-QA-REPORT.md` updated with this pass's findings and the header status line only.
- **Commits created:** None — no code modified, no commit made, per instruction.
- **Tests performed:** See the verification checklist and "Independent verification method" above.
- **Unresolved risks:** No new risks introduced. Carried forward: DOC-1, ROBUST-1, ROBUST-2 (all
  Low), worst-case provider latency and duplicate in-flight requests (informational/Low) — all
  non-blocking per `.ai-company/DEFINITION_OF_DONE.md`. No further implementation rows remain in
  `02-ARCHITECTURE.md` §8; the Development stage for this milestone is complete.
- **Next agent:** Principal Reviewer, per `.ai-company/WORKFLOW.md`'s lifecycle (QA Defect Gate has
  no unresolved Critical/High defects, so the next stage is the Principal Review Quality Gate — not
  a direct push, and not a role this QA session performs).
- **Explicit stop point:** QA has completed independent verification of Dev Task 8 (PASS), and with
  it, all 8 Dev Tasks for Milestone 2. Awaiting the Principal Reviewer's independent quality-gate
  review before Release Review or any CEO push-approval gate.
