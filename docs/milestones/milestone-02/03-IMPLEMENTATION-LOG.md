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

### Session: 2026-08-06 — Dev Task 4: translationRowTemplate() shared per-row markup helper

- **Scope addressed:** `02-ARCHITECTURE.md` §8's "New helper, colocated near `translation(a)` |
  `translationRowTemplate(x, i)`" row, and the consistency requirement in §7 ("both paths must
  render from the exact same per-row template function, not two independently-maintained markup
  strings — called out explicitly in §8 as an implementation requirement, not left to Developer
  discretion"). This is the fourth of the 8 tasks in §8's Developer task order. QA passed Dev Task
  3 first (see `04-QA-REPORT.md`, QA pass 2026-08-06 against `1563cef`/`54f235a`).
- **What was implemented:** `translationRowTemplate(x, i)`, colocated immediately before
  `translation(a)` per the architecture's placement instruction. Branches on `x.status`:
  - `'loading'` → a `.sentence.loading` row (the exact class name §8's later style-block row names
    for CSS targeting) with the Korean `"번역 중…"` label quoted verbatim from §5.
  - `'error'` → a `.sentence.error` row (again, the exact class name §8's style-block row names)
    with an explicit Korean failure message and a `"다시 시도"` retry button (label quoted verbatim
    from §5) wired to `retrySentence(i)` via the same inline-`onclick` convention already used
    throughout this file — never the old placeholder string rendered as if it were content, per
    AC1 and §6.
  - `'success'` (default) → visually identical to today's existing row shape (en/ko pair in
    parens), per §5's "today's existing row shape ... unchanged visually."
  - Every branch escapes `x.en`/`x.ko` via the existing `escapeHtml()` helper, matching the
    Milestone 1 XSS-defense convention used everywhere else in this file.
  - Each branch's outer `<div>` carries a `data-i="${i}"` attribute. This is not explicitly
    specified in the architecture text, but the function's own signature (`translationRowTemplate(x,
    i)`) and §7's description of `updateTranslationRow(i)` needing to "patch the DOM node for row
    i" both presuppose some way to address a specific row's DOM node by index; `data-i` is the
    minimal, non-visual mechanism that fulfills that evident purpose without inventing new
    behavior beyond what `i` is already there for. Flagged here explicitly, per
    `.ai-company/DEVELOPER.md`'s "if you hit a case the architecture doesn't cover, stop and flag
    it," rather than deciding silently.
  - The exact error-message wording (`"이 문장의 번역에 실패했습니다. 인터넷 연결을 확인한 뒤 다시
    시도해 주세요."`) is a Developer-level text choice — §5/Assumption A5 of `01-PM-SPEC.md`
    explicitly leaves exact copy/wording to Development, requiring only that the state be
    *distinguishable*, not prescribing its exact text.
- **Files changed:** `index.html` (26 lines added, 0 removed — confirmed via `git show fb70aa0
  --stat`). No existing line modified; `translation(a)` immediately below is untouched.
- **Commits:**
  - `fb70aa0` — "Add: translationRowTemplate() shared per-row markup helper — Milestone 2 Dev
    Task 4"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed, per standard) loading the actual `index.html` and asserting on runtime behavior:
  - `success` row: byte-for-byte matches the pre-existing `translation()` row shape (same
    `en`/`ko`-in-parens structure), plus the new `data-i` attribute.
  - `loading` row: correct class, shows the `"번역 중…"` label, never renders `ko:null` as the
    literal text `"null"`.
  - `error` row: correct class, explicit Korean failure message present, retry button correctly
    wired to `retrySentence(i)` for the given index, never renders the old placeholder string,
    never renders `ko:null` as `"null"`.
  - Security: malicious HTML/script-shaped `en`/`ko` input is correctly escaped in the output
    (verified both are neutralized, not just one).
  - Regression: `translation(a)`'s own source is confirmed byte-for-byte unchanged and does not
    reference `translationRowTemplate` yet; calling `translation(a)` directly still works exactly
    as before (the new, dormant sibling function doesn't interfere). All of Dev Tasks 1–3's
    constants/functions/contracts confirmed unchanged; `analyze()` confirmed to not reference
    `translationRowTemplate`.
  - **Result: all 22 assertions passed.** Script kept as a local scratch file only, not committed
    to the repository, per `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None to the specified function's branching/behavior — matches
  §5/§7/§8 exactly. The `data-i` attribute and the exact error-copy wording are documented above as
  necessary, disclosed judgment calls within the function's own evident scope, not deviations from
  approved behavior.
- **Recorded per explicit instruction, not solved in this task (carried forward from the Dev Task
  3 QA pass, `04-QA-REPORT.md`):**
  - **Worst-case provider-chain latency:** independently measured by QA at ~33 seconds per
    sentence when every tier of `translateSentenceReliable` fails (2× MyMemory timeout + backoff +
    2× Lingva timeout). Inherent to `02-ARCHITECTURE.md` §2/§3's own approved timeout/backoff
    constants; not something Dev Task 4 touches, causes, or is positioned to fix — `analyze()` is
    not even wired to the new provider chain yet. Relevant to whoever implements the `analyze()`
    rewrite (Dev Task 8's concurrency dispatch) and the loading-state UX this task's
    `translationRowTemplate` renders, since a `'loading'` row could remain visible for up to ~33s
    per stuck sentence in the worst case.
  - **Possible duplicate in-flight requests for identical sentences:** `translationCache` has no
    in-flight de-duplication — two concurrent calls for the same sentence text that both miss the
    cache before either resolves would each independently fire the full network chain. Not
    reachable today (nothing calls `translateSentenceReliable` concurrently yet), and not
    something `translationRowTemplate` (a pure rendering function with no network/cache
    interaction) could introduce or fix. Relevant to whoever implements Dev Task 8's concurrent
    `runWithConcurrency` dispatch.
  - Neither item is addressed by this task's code; both are recorded here again, unchanged, per
    explicit operator instruction to track without solving outside Task 4's scope.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in prior
  session entries; left untouched again this session for the same role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered.
- **Handoff:** Per explicit instruction, this session stops after Task 4 for independent QA. See
  `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 4

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`,
  `.ai-company/CODING_STANDARDS.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (§5/§7/§8 specifically re-read), and
  `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md` (all re-read this session per operator
  instruction), plus `docs/milestones/milestone-02/04-QA-REPORT.md` (confirmed Dev Task 3 QA pass:
  PASS, no unresolved Critical/High defects; noted the two carried-forward observations recorded
  above), `docs/milestones/milestone-02/01-PM-SPEC.md` (Status: APPROVED, §5/Assumption A5
  re-checked for the error-copy wording question), and the current `index.html` (re-read lines
  536–545, the exact region modified, before editing).
- **Scope completed:** Dev Task 4 only (of 8 total, per `02-ARCHITECTURE.md` §8's Developer task
  order) — `translationRowTemplate(x, i)` added as described above. Tasks 5–8 not started.
- **Files changed:** `index.html` only.
- **Commits created:** `fb70aa0` — "Add: translationRowTemplate() shared per-row markup helper —
  Milestone 2 Dev Task 4".
- **Tests performed:** See "Tests performed" above — 22/22 assertions passed via a throwaway jsdom
  script; new-code, security, and regression checks all included.
- **Unresolved risks:** None new from this task's own code. Carrying forward from
  `02-ARCHITECTURE.md` §10 and the Dev Task 3 QA pass: the MyMemory email-quota option, the
  pre-existing translation-privacy gap, the read-coverage caveat, Lingva instance-list staleness,
  DOC-1 (Low, wording nit from Dev Task 2, still open), the ~33s worst-case provider-chain latency,
  and the possible duplicate in-flight requests for identical sentences (both recorded again above
  per explicit instruction, not addressed by this task). Newly noted: `translationRowTemplate` is
  fully implemented but entirely unused until Dev Task 8 (or an earlier `translation(a)`-rewrite
  task) wires it in — QA should expect it to be dormant/unreachable from the UI in this pass, by
  design, not as a defect.
- **Next agent:** QA Engineer (independent QA, per explicit instruction).
- **Explicit stop point:** Awaiting independent QA review of Dev Task 4 (commit `fb70aa0`) before
  any further Development work on this milestone.

### Session: 2026-08-06 — Dev Task 5: translation(a) wired to translationRowTemplate()

- **Scope addressed:** `02-ARCHITECTURE.md` §8's `translation(a)` row ("Rewritten to map each
  entry through `translationRowTemplate(x,i)` instead of the current single-shape template
  string") and §7's consistency requirement. QA passed Dev Task 4 first (see `04-QA-REPORT.md`,
  QA pass 2026-08-06 against `fb70aa0`/`cbc77c5`, which also logged a new Low-severity finding,
  ROBUST-1 — see "Carried forward" below).
- **What was implemented:** `translation(a)`'s per-sentence card now builds its rows via
  `a.translations.map((x,i)=>translationRowTemplate(x,i)).join("")` instead of its own inline
  `` `<div class="sentence">...` `` template string. The second card ("자연스러운 전체 해석") is
  untouched — that block is not named in this architecture row, and
  `Array.prototype.join()` already renders a `null`/`undefined` element as an empty string rather
  than the literal text `"null"`/`"undefined"`, so no defensive change was needed there.
- **Task-ordering deviation (flagged explicitly, not decided silently):** `02-ARCHITECTURE.md` §8
  lists this `translation(a)` row *after* the `analyze()` rewrite, `updateTranslationRow(i)`, and
  `retrySentence(i)` rows in the table. This session implements it first, as Dev Task 5, ahead of
  those three. Reason: `analyze()` today still produces plain `{en, ko}` objects with no `status`
  field (confirmed directly — see Tests below). Because `translationRowTemplate` treats a missing/
  unrecognized `status` as the default success case, wiring `translation(a)` to it *now* produces
  byte-identical visible output to the old inline markup (plus the harmless `data-i` attribute
  Dev Task 4 already added) — a genuinely zero-behavior-change commit. Doing the `analyze()`
  rewrite *before* this one would have been unsafe: `analyze()` producing `status:'loading'`/
  `'error'` objects while `translation(a)` still used its old inline template (which unconditionally
  reads `x.ko`) would have rendered the literal text `(null)` on screen for every in-flight or
  failed sentence — reintroducing, in a different shape, exactly the "misleading rendered output"
  problem this milestone exists to fix. Sequencing `translation(a)` first, and `analyze()`
  afterward, is the only order that keeps every commit in a working state per
  `.ai-company/GIT_RULES.md` rule 1. Recorded here per `.ai-company/DEVELOPER.md`'s "if you hit a
  case the architecture doesn't cover, stop and flag it" — the architecture fully specifies both
  rows' end content, it just doesn't state that their *relative order* matters for safety, which
  this note makes explicit.
- **Files changed:** `index.html` (9 lines added, 1 removed — confirmed via `git show 284ea76
  --stat`). Only the `translation(a)` line/comment changed; no other function touched.
- **Commits:**
  - `284ea76` — "Refactor: translation(a) now renders rows via translationRowTemplate() —
    Milestone 2 Dev Task 5"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed, per standard) loading the actual `index.html` and asserting on runtime behavior:
  - `translation(a)` called with today's plain `{en,ko}`-shaped data renders both sentences'
    English and Korean text correctly, plus the summary card with the joined translation.
  - Each row now carries a `data-i` attribute (from `translationRowTemplate`); rows use the plain
    `sentence` class (fall through to the success branch) for today's status-less data.
  - `translation(a)`'s output is confirmed, by direct string comparison, to be produced by mapping
    each entry through `translationRowTemplate(x,i)` — not a coincidentally-similar independent
    implementation.
  - Security: malicious HTML/script-shaped `en`/`ko` input passed all the way through
    `translation(a)` is still correctly escaped (regression-checked end-to-end, not just at the
    `translationRowTemplate` unit level already covered in Dev Task 4).
  - Edge case: an empty `translations` array does not throw.
  - Confirmed at the source level that `analyze()` is completely unchanged by this task — it still
    builds `translations=sents.map((s,i)=>({en:s,ko:translated[i]}))` with no `status:` field
    anywhere in its body (the only occurrence of the word "status" in `analyze()` is the
    pre-existing, unrelated `document.getElementById("status")` DOM element id from before this
    milestone).
  - Regression: Dev Tasks 1–4's constants/functions/contracts (`CONCURRENCY_LIMIT`,
    `simpleTranslate`'s `{ko,matched}`, `translateSentenceReliable`/`translateLingva`,
    `translationRowTemplate`'s signature) all confirmed unchanged; `splitSentences`/`state.active`
    unaffected; the old `translateSentence()` still returns a plain string.
  - **Result: all 16 assertions passed** (one assertion needed a one-line fix mid-session — an
    overly broad `!includes("status")` check false-matched the pre-existing `#status` DOM id; the
    fix narrowed it to a `status\s*:` regex, and the corrected assertion still passed). Script kept
    as a local scratch file only, not committed to the repository, per `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None to the specified end state (`translation(a)` does map
  through `translationRowTemplate(x,i)` exactly as §8 states). The task-*ordering* deviation is
  documented above as an explicit, disclosed judgment call made for safety, not a silent departure
  from approved scope.
- **Carried forward, not solved (per explicit instruction; none of the three items were required
  by this task's actual code change):**
  - **ROBUST-1** (Low, from the Dev Task 4 QA pass): `translationRowTemplate`'s success branch has
    no guard against `x.ko` being `null`/`undefined`. This task makes `translationRowTemplate`
    reachable from a *live* render path (`translation(a)` now calls it for real) for the first
    time, but the specific malformed-input shape still cannot occur in practice, because
    `analyze()` — confirmed unchanged above — never produces a `status:'success'` object paired
    with a missing `ko`; every entry it builds always has a real string `ko`. Still deferred,
    unchanged in status.
  - **Worst-case provider-chain latency** (~33s, from the Dev Task 3 QA pass): unaffected by this
    task; `translateSentenceReliable` is still not called by anything.
  - **Duplicate in-flight requests for identical sentences** (from the Dev Task 3 QA pass):
    unaffected by this task, same reason.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in prior
  session entries; left untouched again this session for the same role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered.
- **Handoff:** Per explicit instruction, this session stops after Task 5 for independent QA. See
  `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 5

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`,
  `.ai-company/CODING_STANDARDS.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (§7/§8 specifically re-read),
  `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md`, and
  `docs/milestones/milestone-02/04-QA-REPORT.md` (all re-read this session per operator
  instruction — confirmed Dev Task 4 QA pass: PASS, with ROBUST-1 logged as Low/deferred), and the
  current `index.html` (re-read lines 559–570, the exact region modified, before editing).
- **Scope completed:** Dev Task 5 only (of the Developer task order established across this
  milestone's sessions) — `translation(a)` rewired to `translationRowTemplate`, as described
  above. The `analyze()` rewrite, `updateTranslationRow(i)`, `retrySentence(i)`, `loadSaved()`'s
  defensive read, and the CSS style-block additions remain not started.
- **Files changed:** `index.html` only.
- **Commits created:** `284ea76` — "Refactor: translation(a) now renders rows via
  translationRowTemplate() — Milestone 2 Dev Task 5".
- **Tests performed:** See "Tests performed" above — 16/16 assertions passed via a throwaway jsdom
  script; new-code, security, edge-case, and regression checks all included.
- **Unresolved risks:** None new from this task's own code. Carrying forward unchanged: the
  MyMemory email-quota option, the pre-existing translation-privacy gap, the read-coverage caveat,
  Lingva instance-list staleness, DOC-1 (Low, wording, Dev Task 2), ROBUST-1 (Low, Dev Task 4 QA
  pass — reachability status updated above but still not actually triggerable), the ~33s
  worst-case provider-chain latency, and the possible duplicate in-flight requests for identical
  sentences. None of the three items the operator asked to carry forward were solved, as
  instructed, since none were required by this task's actual change.
- **Next agent:** QA Engineer (independent QA, per explicit instruction).
- **Explicit stop point:** Awaiting independent QA review of Dev Task 5 (commit `284ea76`) before
  any further Development work on this milestone.

## Handoff log

_(Handoffs per `.ai-company/HANDOFF_PROTOCOL.md` appended here in chronological order.)_

1. Dev Task 1 handoff — see above (2026-08-06).
2. Dev Task 2 handoff — see above (2026-08-06).
3. Dev Task 3 handoff — see above (2026-08-06).
4. Dev Task 4 handoff — see above (2026-08-06).
5. Dev Task 5 handoff — see immediately above (2026-08-06).
