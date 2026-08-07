# 03-IMPLEMENTATION-LOG.md — Milestone 2: Translation Reliability

**Status: IN PROGRESS — Development stage**

`02-ARCHITECTURE.md` Status: APPROVED (CEO approval recorded 2026-08-06; see that document's "CEO
Approval" section). Development proceeding task-by-task per its §8 "Developer task order," one
commit per logical task, stopping for CEO approval after each task per explicit instruction.

## Session entry template

```
### Session: <date> — <brief description>

- Scope addressed: <which architecture/spec items>
- Files changed: <list>
- Commits:
  - <hash> — <message>
- Deviations from architecture (if any): <description, and who was notified>
- Handoff: see .ai-company/HANDOFF_PROTOCOL.md fields below
```

---

### Session: 2026-08-06 — Dev Task 1: concurrency/timeout/cache scaffolding

- **Scope addressed:** `02-ARCHITECTURE.md` §8 row 1 ("Near `const state={...}` (line 307)") and
  §2/§3/§4's constants/helpers. Pure scaffolding — no existing function's behavior changed. This is
  the first of the 8 tasks in §8's Developer task order.
- **What was implemented:** Added `CONCURRENCY_LIMIT=3`, `REQUEST_TIMEOUT_MS=8000`,
  `RETRY_BACKOFF_MS=800`, `translationCache=new Map()`, `fetchWithTimeout(url,ms)` (fetch wrapped
  in `AbortController`, rejects on abort), and `runWithConcurrency(items,limit,worker)` (bounded
  worker-pool executor, matches the architecture's §2 pseudocode exactly). Placed immediately after
  `const state={...}` at line 307, before the existing tab-click listener, per the architecture's
  specified location. A one-line comment points back to
  `docs/milestones/milestone-02/02-ARCHITECTURE.md §2-§4` for context.
- **Files changed:** `index.html` (20 lines added, 0 removed; no existing line modified).
- **Commits:**
  - `4f8975d6f315f89757280ec67e2da61c86e49e5f` — "Add: Milestone 2 concurrency/timeout/cache
    scaffolding (translationCache, fetchWithTimeout, runWithConcurrency)"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed to the repo, per standard) loading the actual `index.html`, executing its inline
  script, and asserting on runtime state — not just "it ran without throwing":
  - Regression baseline: `splitSentences`, `simpleTranslate`, `translateSentence`, `analyze` still
    defined; `state.active==="overview"`; `splitSentences("Hello world. This is a test!")` still
    returns 2 sentences — confirms no existing behavior was disturbed by this task.
  - `CONCURRENCY_LIMIT===3`, `REQUEST_TIMEOUT_MS===8000`, `RETRY_BACKOFF_MS===800`.
  - `translationCache` is a `Map`, starts empty.
  - `fetchWithTimeout` rejects with `AbortError` once the configured timeout elapses (verified
    against a `fetch` mock that never resolves on its own; observed abort at ~106ms for a 100ms
    timeout).
  - `runWithConcurrency`: with a pool of 10 async items and `limit=3`, the maximum number of
    simultaneously in-flight workers observed was exactly 3 (never exceeded, and actually reached
    the cap under load); results array preserves input order (`results[i]` corresponds to
    `items[i]`, not completion order); also verified it doesn't hang when `items.length < limit`.
  - **Result: all 18 assertions passed.** Full output retained in this session's tool transcript;
    script itself was a scratch file in a temp sandbox directory, not checked into the repo, per
    `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None. Implementation matches §2's pseudocode and §8's file-plan
  row exactly (constant names, values, and placement).
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md` and `02-ARCHITECTURE.md` had
  pre-existing uncommitted changes from the prior PM/Architect/CEO-approval sessions when this
  Development session began. Per `CLAUDE.md`'s role-separation rule ("do not perform another role's
  work"), those documentation changes were left untouched and were **not** staged or committed as
  part of this task — only `index.html` was staged (`git add index.html`, not `git add -A`), so
  this commit contains exactly one logical change, per `.ai-company/GIT_RULES.md` rule 4/9.
- **Operational note:** a stale `.git/index.lock` (unrelated to this session's own work, already
  present in the working tree at session start) blocked `git add`/`git commit` until file-deletion
  permission was granted for the workspace folder; recorded here since it required an out-of-band
  tool call, not because it reflects a code or scope issue.
- **Handoff:** Not yet — per explicit instruction, this session stops after Task 1 for CEO/operator
  approval before proceeding to Task 2. See `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 1

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`
  (read in a prior session this milestone), `.ai-company/DEVELOPER.md`,
  `.ai-company/CODING_STANDARDS.md`, `.ai-company/TESTING_STANDARDS.md`, `.ai-company/GIT_RULES.md`,
  `docs/milestones/milestone-02/01-PM-SPEC.md` (confirmed Status: APPROVED),
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (confirmed Status: APPROVED after recording the
  CEO's verbal approval into that document — see its "CEO Approval" section), and the current
  `index.html` (re-read the exact region being modified, lines 305–312, before editing).
- **Scope completed:** Dev Task 1 only (of 8 total, per `02-ARCHITECTURE.md` §8's Developer task
  order) — concurrency/timeout/cache scaffolding, as detailed above. Tasks 2–8 not started.
- **Files changed:** `index.html` only.
- **Commits created:** `4f8975d6f315f89757280ec67e2da61c86e49e5f` — "Add: Milestone 2
  concurrency/timeout/cache scaffolding (translationCache, fetchWithTimeout, runWithConcurrency)".
- **Tests performed:** See "Tests performed" above — 18/18 assertions passed via a throwaway jsdom
  script; both new-code and regression checks included.
- **Unresolved risks:** None new. Carrying forward from `02-ARCHITECTURE.md` §10: the MyMemory
  email-quota option (deferred to CEO), the pre-existing translation-privacy gap, the read-coverage
  caveat, and Lingva instance-list staleness — none of these are implicated by Task 1 specifically.
- **Next agent:** Same session, same role (Senior Developer) — but per explicit operator
  instruction, waiting for approval before starting Dev Task 2 (refactor `simpleTranslate()` to a
  structured `{ko, matched}` return).
- **Explicit stop point:** Awaiting operator/CEO approval to proceed to Dev Task 2. No further code
  changes until that approval is given.

### Session: 2026-08-06 — Dev Task 2: simpleTranslate() structured return

- **Scope addressed:** `02-ARCHITECTURE.md` §8 row 2 ("`simpleTranslate(s)` (lines 316–320)") and
  §6's contract requirement. This is the second of the 8 tasks in §8's Developer task order.
  QA passed Dev Task 1 first (see `04-QA-REPORT.md`, QA pass 2026-08-06).
- **What was implemented:** `simpleTranslate(s)` now returns `{ko, matched}` instead of a bare
  string — `ko` is the substitution result (`out`), `matched` is `out!==s`. The phrase-map array
  and substitution loop are byte-for-byte unchanged, per the architecture's "logic otherwise
  unchanged" instruction. The old fragile `out===s ? placeholder : out` string-sniff is gone from
  `simpleTranslate` entirely.
- **Necessary adaptation (not a scope expansion into Task 3):** `translateSentence(s)` — the one
  remaining caller of `simpleTranslate()` — still needs to return a plain string, since it is not
  replaced by `translateSentenceReliable()` until Dev Task 3 (§8 row 3). Changing
  `simpleTranslate`'s return shape without touching this call site would have broken
  `translateSentence` (and, downstream, `analyze()`'s rendered `ko` values) immediately, which
  `.ai-company/GIT_RULES.md` rule 1 ("every commit ... should leave the application in a working
  state") does not permit. The one-line fix: `translateSentence`'s fallback now reads
  `const r=simpleTranslate(s);return r.matched?r.ko:"<same placeholder text as before>"` —
  reproducing, byte-for-byte, the exact string `translateSentence` already returned in this case
  before this task. This is the minimal mechanical consequence of Task 2's own interface change,
  not an implementation of Task 3's `translateSentenceReliable`/cache/retry/fallback-chain logic,
  which remains entirely unstarted. A `TODO(Milestone 2, Dev Task 3)` comment marks this bridge in
  the code for removal when Task 3 lands. Flagging this explicitly per `.ai-company/DEVELOPER.md`
  ("if you hit a case the architecture doesn't cover, stop and flag it") — the architecture already
  describes both sides of this exact interface (§6 and §8 row 3), it's only the sequencing/bridging
  detail between Task 2 and Task 3 that isn't spelled out verbatim.
- **Files changed:** `index.html` (14 lines added, 3 removed; no line outside `simpleTranslate`'s
  body and `translateSentence`'s single return-fallback statement touched).
- **Commits:**
  - `313757b` — "Refactor: simpleTranslate() returns structured {ko, matched} instead of a bare
    string"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Independently authored throwaway
  jsdom script (not committed, per standard) loading the actual `index.html` and asserting on
  runtime behavior:
  - `simpleTranslate` returns an object; `matched===true` and `ko` contains the substituted Korean
    text when a phrase-map entry applies; `matched===false` and `ko` equals the original,
    unmodified input text when nothing applies (confirms the placeholder string no longer lives
    inside `simpleTranslate`).
  - Substitution logic unchanged: multiple map entries still apply correctly in a single call.
  - `translateSentence` still returns a plain string in all three cases: MyMemory success (string
    passed through untouched), MyMemory failure + phrase-map match (returns the Korean
    substitution), MyMemory failure + no phrase-map match (returns the exact legacy placeholder
    string, verified by string equality against the original literal).
  - Regression: Dev Task 1's scaffolding (`CONCURRENCY_LIMIT`, `translationCache`,
    `runWithConcurrency`) untouched; `splitSentences` and `analyze` still defined/unchanged;
    `state.active` still initializes to `"overview"`.
  - **Result: all 17 assertions passed.**
- **Deviations from architecture:** None to the specified change itself (`simpleTranslate`'s
  return shape and unchanged substitution logic match §8 row 2 exactly). The one necessary
  addition — the one-line adapter in `translateSentence` — is documented above and is not a
  deviation from what's approved, since both `simpleTranslate`'s new contract (§6) and
  `translateSentence`'s planned replacement (§8 row 3) are already part of the approved
  `02-ARCHITECTURE.md`; only the transitional bridge between the two tasks required a judgment
  call, which is recorded here rather than made silently.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md` and `02-ARCHITECTURE.md`
  still carry the same pre-existing uncommitted changes noted in the Dev Task 1 entry above; left
  untouched again this session for the same reason (role separation — not this role's documents to
  commit). `04-QA-REPORT.md` also has uncommitted content from the QA pass on Dev Task 1; likewise
  untouched.
- **Operational note:** the same stale `.git/index.lock` behavior seen in the Dev Task 1 session
  recurred at the start of this session (present before any action this session took); resolved the
  same way (file-deletion permission requested and granted for the workspace folder) before
  `git add`/`git commit`.
- **Handoff:** Per explicit instruction, this session stops after Task 2 for QA. See
  `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 2

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`
  (all re-read this session per operator instruction), plus (carried from context established
  earlier this milestone and re-confirmed) `.ai-company/CODING_STANDARDS.md`,
  `.ai-company/GIT_RULES.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-02/01-PM-SPEC.md` (Status: APPROVED),
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (Status: APPROVED — §6 and §8 row 2/row 3
  specifically re-read before editing), `docs/milestones/milestone-02/04-QA-REPORT.md` (confirmed
  Dev Task 1 QA pass: PASS, no unresolved Critical/High defects), and the current `index.html`
  (re-read lines 330–341, the exact region modified, before editing).
- **Scope completed:** Dev Task 2 only (of 8 total, per `02-ARCHITECTURE.md` §8's Developer task
  order) — `simpleTranslate()` structured-return refactor, plus the minimal necessary adapter in
  `translateSentence()` documented above. Tasks 3–8 not started.
- **Files changed:** `index.html` only.
- **Commits created:** `313757b` — "Refactor: simpleTranslate() returns structured {ko, matched}
  instead of a bare string".
- **Tests performed:** See "Tests performed" above — 17/17 assertions passed via an independently
  authored throwaway jsdom script; new-code, adapter-behavior, and regression checks all included.
- **Unresolved risks:** None new. Carrying forward from `02-ARCHITECTURE.md` §10 (unchanged by this
  task): the MyMemory email-quota option, the pre-existing translation-privacy gap, the
  read-coverage caveat, and Lingva instance-list staleness. Newly noted: the `translateSentence`
  bridge adapter (with its duplicated placeholder-string literal) is intentionally temporary and
  should disappear as part of Dev Task 3's replacement of that function — QA/Reviewer should expect
  it to be removed, not preserved, once Task 3 lands.
- **Next agent:** QA Engineer.
- **Explicit stop point:** Awaiting QA review of Dev Task 2 (commit `313757b`) before any further
  Development work (Dev Task 3) on this milestone.

### Session: 2026-08-06 — Dev Task 3: translateSentenceReliable() + translateLingva()

- **Scope addressed:** `02-ARCHITECTURE.md` §8 row 3 ("`translateSentence(s)` (line 321) | Replace
  with `translateSentenceReliable(s, {bypassCache=false}={})`") and the "New helper ...
  `translateLingva(s)`" row, plus §6's Lingva-adoption decision. This is the third of the 8 tasks
  in §8's Developer task order. QA passed Dev Task 2 first (see `04-QA-REPORT.md`, QA pass
  2026-08-06 against `313757b`/`95f9d43`).
- **What was implemented:**
  - `LINGVA_INSTANCES` — an ordered array of two Lingva Translate mirror hostnames
    (`translate.plausibility.cloud`, `lingva.garudalinux.org`), colocated with the Dev Task 1
    scaffolding, per §6's explicit requirement that the fallback list "must not be hardcoded to a
    single instance."
  - `translateLingva(s)` — tries each `LINGVA_INSTANCES` host in order against Lingva's
    `/api/v1/en/ko/{text}` endpoint, each attempt wrapped in `fetchWithTimeout`; returns the
    translation string on the first mirror that responds `ok` with a non-empty `translation`
    field, or `null` if every mirror fails.
  - `translateSentenceReliable(s, {bypassCache=false}={})` — the full provider chain from §2–§4/§6:
    checks `translationCache` first unless `bypassCache` is set; on a cache miss, attempts
    MyMemory once, waits `RETRY_BACKOFF_MS` (800ms) and attempts MyMemory a second time on
    failure, falls through to `translateLingva(s)`, and finally to `simpleTranslate(s)`. Returns
    `{ko, status:'success'|'error', source}`; writes only successful results into
    `translationCache`, keyed by the exact sentence text, per §4's "only successful results are
    cached" rule. When every tier fails (including a `simpleTranslate` `matched:false`), returns
    `{ko:null, status:'error', source:null}` — never a placeholder string — per §6's exact
    specification.
- **Interpretation of "Replace" (flagged explicitly, not decided silently):** §8 row 3's literal
  wording says translateSentence(s) is "replaced with" translateSentenceReliable(s,...). Taken
  literally and in isolation, that would mean deleting `translateSentence` and repointing
  `analyze()`'s existing call site at the new function in this same task. That is **not** what was
  done. Reason: `analyze()`'s current call site
  (`Promise.all(sents.map(s=>translateSentence(s)))`) expects a plain string return;
  `translateSentenceReliable` returns a structured `{ko,status,source}` object. Actually rewiring
  `analyze()` to consume that shape — including the progressive per-row rendering, the switch from
  `Promise.all` to `runWithConcurrency`, and the `#status` counter changes — is explicitly §8's
  separate `analyze()` row (Dev Task 8), not this row. Doing a partial, unwired swap here (e.g.
  changing only the function name at the call site without the rest of Task 8's wiring) would
  either (a) break rendering immediately, since `ko` would render as `[object Object]`, violating
  `.ai-company/GIT_RULES.md` rule 1's "every commit should leave the application in a working
  state," or (b) require implementing part of Task 8's scope now, which
  `.ai-company/DEVELOPER.md` prohibits without flagging upward first. Resolution: this task adds
  `translateSentenceReliable`/`translateLingva` as new, fully-implemented, self-contained
  functions — exactly matching the pattern already established in Dev Task 1, where
  `fetchWithTimeout`/`runWithConcurrency` were added and left dormant until later tasks wired them
  in. `translateSentence()` and `analyze()` are completely untouched this session. This is recorded
  here per `.ai-company/DEVELOPER.md`'s "if you hit a case the architecture doesn't cover, stop and
  flag it" — the architecture fully specifies both functions' behavior, it only doesn't spell out
  the intermediate sequencing between adding them and retiring the old function, which this note
  makes explicit rather than resolving silently.
- **Files changed:** `index.html` (69 lines added, 0 removed; no existing line modified — confirmed
  via `git show 1563cef --stat`).
- **Commits:**
  - `1563cef` — "Add: translateSentenceReliable() and translateLingva() — Milestone 2 Dev Task 3"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed, per standard) loading the actual `index.html` and asserting on runtime behavior with
  a mocked/instrumented `fetch`:
  - MyMemory success on first attempt: exactly one `fetch` call, `status:'success'`,
    `source:'mymemory'`, result cached under the exact sentence key.
  - Cache hit: a second call for the same sentence issues zero new `fetch` calls.
  - `bypassCache:true` forces a fresh network attempt even when a cached result exists.
  - MyMemory fails twice with the expected ~800ms backoff observed between attempts, then falls
    through to Lingva; Lingva success yields `status:'success'`, `source:'lingva'`.
  - `translateLingva` skips a non-`ok` first mirror and succeeds on the second configured mirror;
    returns `null` (not a thrown error) when every mirror fails.
  - Total failure across all tiers with no phrase-map match: `status:'error'`, `ko:null`,
    `source:null`, and the sentence is confirmed **not** written to `translationCache`.
  - Total network failure but a phrase-map match exists: `status:'success'`,
    `source:'phrasemap'`, and the result **is** cached.
  - Regression: `CONCURRENCY_LIMIT`/`REQUEST_TIMEOUT_MS`/`RETRY_BACKOFF_MS`/`runWithConcurrency`/
    `fetchWithTimeout` (Task 1) and `simpleTranslate`'s `{ko,matched}` contract (Task 2) all
    unchanged; the old `translateSentence()` still returns a plain string; `analyze()`'s source is
    confirmed (via direct string search on its body) to not reference
    `translateSentenceReliable` yet.
  - **Result: all 36 assertions passed.** Script kept as a local scratch file only, not committed
    to the repository, per `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None to the specified functions' behavior — both match §2–§4/§6
  exactly. The sequencing interpretation of "Replace" is documented above as an explicit,
  disclosed judgment call, not a silent deviation.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in the Dev
  Task 1 and Dev Task 2 entries above; left untouched again this session for the same
  role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered this time.
- **Handoff:** Per explicit instruction, this session stops after Task 3 for QA. See
  `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 3

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`,
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (all re-read this session per operator
  instruction, §2–§4/§6/§8 rows 3 and "New helper ... translateLingva" specifically), plus
  (re-confirmed from context established earlier this milestone) `.ai-company/CODING_STANDARDS.md`,
  `.ai-company/GIT_RULES.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-02/01-PM-SPEC.md` (Status: APPROVED),
  `docs/milestones/milestone-02/04-QA-REPORT.md` (confirmed Dev Task 2 QA pass: PASS, one
  Low-severity documentation nit, no unresolved Critical/High defects), and the current
  `index.html` (re-read lines 305–360, the exact region modified, before editing).
- **Scope completed:** Dev Task 3 only (of 8 total, per `02-ARCHITECTURE.md` §8's Developer task
  order) — `translateSentenceReliable()` and `translateLingva()` added as described above. Tasks
  4–8 not started. (Numbering here follows the table's row order as Dev Tasks 1–8 have been
  executed so far: Task 1 = row 1 + the two "New helper" rows for `fetchWithTimeout`/
  `runWithConcurrency`; Task 2 = row 2 (`simpleTranslate`); Task 3 = row 3
  (`translateSentenceReliable`) + the "New helper" row for `translateLingva`, per the same
  bundling pattern as Task 1.)
- **Files changed:** `index.html` only.
- **Commits created:** `1563cef` — "Add: translateSentenceReliable() and translateLingva() —
  Milestone 2 Dev Task 3".
- **Tests performed:** See "Tests performed" above — 36/36 assertions passed via a throwaway jsdom
  script; new-code and regression checks both included.
- **Unresolved risks:** None new. Carrying forward from `02-ARCHITECTURE.md` §10: the MyMemory
  email-quota option, the pre-existing translation-privacy gap, the read-coverage caveat, and
  Lingva instance-list staleness. Newly noted: `translateSentenceReliable`/`translateLingva` are
  fully implemented but entirely unused until Dev Task 8 wires `analyze()` to call them — QA should
  expect them to be dormant/unreachable from the UI in this pass, by design, not as a defect. The
  Dev Task 2 bridge in `translateSentence()` (with the relocated placeholder string, per QA's
  DOC-1 finding) is still in place and still unremoved — it is retired by Dev Task 8's `analyze()`
  rewrite, not this task.
- **Next agent:** QA Engineer.
- **Explicit stop point:** Awaiting QA review of Dev Task 3 (commit `1563cef`) before any further
  Development work (Dev Task 4/8) on this milestone.

## Handoff log

_(Handoffs per `.ai-company/HANDOFF_PROTOCOL.md` appended here in chronological order.)_

1. Dev Task 1 handoff — see above (2026-08-06).
2. Dev Task 2 handoff — see above (2026-08-06).
3. Dev Task 3 handoff — see immediately above (2026-08-06).
