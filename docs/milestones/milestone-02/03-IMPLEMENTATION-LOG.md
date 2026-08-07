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

### Session: 2026-08-06 — Dev Task 6: analyze() progressive dispatch + updateTranslationRow + retrySentence

- **Scope addressed:** `02-ARCHITECTURE.md` §8's `analyze()` row, the `updateTranslationRow(i)`
  row, and the `retrySentence(i)` row (§2/§3/§5/§7 provide the underlying design). This is the
  point at which the whole translation-reliability feature actually goes live. QA passed Dev Task
  5 first (see `04-QA-REPORT.md`, QA pass 2026-08-06 against `284ea76`/`911735e`).
- **What was implemented:**
  - **`analyze()` rewrite:** `translations` is now built in its `'loading'` shape
    (`{en, ko:null, status:'loading', source:null}`) immediately after `vocab`/`phrases`/`questions`
    are computed (unchanged). `state.analysis` is assigned with all fields real except
    `translations` (still loading), then `render()` is called once immediately — since Dev Task 5
    already wired `translation(a)` to `translationRowTemplate`, this render genuinely shows a
    correct overview/vocab/phrases/SAT tab plus a real translation-tab loading skeleton, not a
    blocking wait. The old `Promise.all(sents.map(s=>translateSentence(s)))` is replaced by
    `runWithConcurrency(sents, CONCURRENCY_LIMIT, async(s,i)=>{...})`, where each worker calls
    `translateSentenceReliable(s)` (Dev Task 3), mutates `translations[i]` in place, and calls
    `updateTranslationRow(i)`. `#status` now shows a progressive counter
    (`"한글 번역 진행 중… (N/M 완료)"`) during translation, then a final summary that surfaces a
    failure count when nonzero (`"분석 완료 · N문장 · M개 핵심어휘 · 번역 K건 실패"`), per §5's exact
    wording. `analyzeBtn`'s disable/enable and the `finally` block are byte-identical in structure
    to before — the `await runWithConcurrency(...)` stays inside the `try`, so the button remains
    disabled for the whole translation phase, preserving Milestone 1's overlap guard exactly as
    instructed.
  - **`updateTranslationRow(i)`:** patches only the DOM node for row `i` (found via the `data-i`
    attribute `translationRowTemplate` has rendered since Dev Task 4) when
    `state.active==='translation'`; no-ops otherwise. Added one defensive no-op the architecture
    text doesn't explicitly call out: if the target node can't be found (e.g. `render()` hasn't
    happened yet for some reason), it returns instead of throwing — flagged here per
    `.ai-company/DEVELOPER.md`'s "stop and flag it" rule, since it's a necessary robustness
    addition, not a behavior change to anything specified.
  - **`retrySentence(i)`:** re-runs `translateSentenceReliable(a.translations[i].en,
    {bypassCache:true})` for one sentence, updates that entry, and calls `updateTranslationRow(i)`
    — exactly the three steps §8 describes, wired to the `다시 시도` button Dev Task 4 already
    rendered into error rows.
  - **Removed the old `translateSentence()` function** (and its Dev Task 2 bridge comment/TODO).
    `grep` confirms zero remaining references anywhere in the file. This is the point Dev Tasks 2
    and 3's logs both anticipated ("it is retired by [whichever task] rewires `analyze()`") — that
    task turned out to be this one, numbered Task 6 in this session's sequence rather than "Task 8"
    as those earlier logs guessed, since Dev Task 3 already absorbed two architecture-table rows
    and Dev Task 1 absorbed three. Also updated the now-stale "NOT YET WIRED IN" comment above
    `translateSentenceReliable` to reflect current reality.
- **Bundling rationale (flagged explicitly):** `analyze()`'s rewrite hard-requires
  `updateTranslationRow` to exist (it's called directly from the new worker callback), so those two
  could not be split across tasks. `retrySentence(i)` was bundled in for a different, disclosed
  reason: Dev Task 4 already rendered a live `다시 시도` button (`onclick="retrySentence(i)"`) into
  error rows; once `analyze()` can produce real `status:'error'` rows in production (which it now
  can, as of this task), leaving `retrySentence` unimplemented would ship a button that throws a
  `ReferenceError` when clicked — a real, immediately-reachable dead-button defect, not a
  theoretical one like Dev Task 4's ROBUST-1. Closing that gap in the same commit avoids
  introducing a known regression window, consistent with `.ai-company/GIT_RULES.md` rule 1.
- **ROBUST-1 status re-examined, deliberately not fixed here:** `04-QA-REPORT.md`'s Dev Task 5 pass
  recommended fixing `translationRowTemplate`'s `x.ko` null/undefined guard "no later than
  whichever task first makes `analyze()` capable of producing a `status:'success'` object" — this
  task is exactly that trigger point. However, re-verified that `translateSentenceReliable`'s own
  contract (Dev Task 3) still guarantees `status:'success'` is only ever returned together with a
  real string `ko` (both of its success branches set `ko` from an actual provider/phrase-map
  result); `analyze()`'s `Object.assign(translations[i], {ko:r.ko, status:r.status, source:r.source})`
  simply passes that guarantee through unchanged. So even after this task, no code path constructs
  the malformed shape ROBUST-1 describes — it remains exactly as unreachable as QA last assessed.
  Deliberately did **not** add the defensive guard anyway, despite it being cheap: doing so would
  require editing `translationRowTemplate`, a Dev Task 4 function, which is outside this task's
  named scope (`analyze()`/`updateTranslationRow`/`retrySentence`) — per explicit instruction to
  "implement ONLY Task 6" and "do not touch unrelated code." Carried forward again, unchanged.
- **New risk discovered during testing (flagged, not fixed — outside Task 6's scope):**
  `saveCurrent()` (unmodified, pre-Milestone-2 function) spreads `state.analysis` — including
  `translations` — directly into `localStorage` whenever the user clicks "학습자료 저장," with no
  awareness of whether translation is still in progress. Before this task, `state.analysis` was
  only ever assigned once every translation had already resolved, so this was never reachable. As
  of this task's progressive-rendering design, `state.analysis` is assigned *before* translations
  resolve — meaning a user who saves while translations are still `'loading'` will persist entries
  with `ko:null, status:'loading'` into their saved study set. **Confirmed by direct test:**
  reopening such a saved item via `loadSaved()` renders a permanently stuck `"번역 중…"` skeleton
  row with **no retry button** (only `status:'error'` rows render one), since neither `loadSaved()`
  nor `translationRowTemplate` currently has any path to recover a stuck `'loading'` entry. This is
  a newly-introduced, realistically-reachable gap — not a crash or data loss (the rest of the saved
  analysis is intact and usable), but a dead end for that specific sentence. **Not fixed here**:
  the fix would need to touch `saveCurrent()` and/or `loadSaved()` (§8's separate, not-yet-started
  `loadSaved()` defensive-read row) and/or `translationRowTemplate` (also give a retry affordance
  to stuck `'loading'` rows, e.g. treating them the same as `'error'` after some threshold, or
  simply disabling/warning on save while any entry is still `'loading'`) — none of which are Dev
  Task 6's named scope. Logged here per `CLAUDE.md`'s "never silently change scope... log it...
  hand it upward" rule, for the CEO/Architect to prioritize as a new item (most naturally folded
  into the existing `loadSaved()` task, or a new one).
- **Files changed:** `index.html` (55 lines added, 19 removed — confirmed via `git show 9dd5ae8
  --stat`). Only the `simpleTranslate`/`translateSentenceReliable` comment blocks, the old
  `translateSentence()` removal, and `analyze()` plus its two new sibling functions were touched;
  vocab/phrase/question computation logic, `translation(a)`, `translationRowTemplate`, and every
  other function are byte-identical to before.
- **Commits:**
  - `9dd5ae8` — "Add: analyze() progressive translation dispatch, updateTranslationRow(),
    retrySentence() — Milestone 2 Dev Task 6"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed, per standard) loading the actual `index.html`, filling in the real `#passage`/
  `#title`/`#grade`/`#qcount` form fields, and calling `analyze()` end-to-end against a mocked
  `fetch` — not just unit-testing isolated functions:
  - `state.analysis` is assigned early (before any translation resolves) with `vocab`/`phrases`/
    `questions` already fully populated and every translation entry in `'loading'` status;
    `analyzeBtn` is disabled and `#status` shows the progressive counter during this window.
  - Bounded concurrency verified end-to-end through `analyze()` itself (not just
    `runWithConcurrency` in isolation, as Dev Task 1 tested): never exceeded `CONCURRENCY_LIMIT`
    simultaneous mocked `fetch` calls across an 8-sentence passage.
  - On completion: every translation reaches `status:'success'` with real `ko` text from the
    mocked provider; `analyzeBtn` is only re-enabled after the full pool settles; `#status` shows
    the correct final summary, with no failure mention when there were none.
  - A total-failure sentence (no network, no phrase-map match) ends in `status:'error'`,
    `ko:null`; `#status` correctly surfaces `"번역 1건 실패"`; switching to the translation tab
    renders the explicit error row, never `"(null)"` and never the old placeholder string.
  - `updateTranslationRow`: verified DOM patching actually occurs when a user switches to the
    translation tab mid-analysis (simulated realistically — not just calling the function directly
    with a hand-built object); confirmed no-op when the tab isn't active; confirmed no-throw when
    the target node is missing.
  - `retrySentence`: verified it flips a failed sentence to success and patches its row live;
    verified it uses `bypassCache:true` (still hits the network even when a cache entry already
    exists for that sentence).
  - Security: malicious text returned by the (mocked) translation provider is still escaped
    end-to-end through the full live `analyze()` → `translation(a)` → `translationRowTemplate`
    pipeline, not just at the unit level.
  - Regression: the overview tab still renders correctly after a full `analyze()` run; Dev Tasks
    1–4's constants/functions/contracts all confirmed unchanged.
  - **Result: all 30 assertions passed** (one assertion needed correction mid-session: it assumed
    `state.active` would remain `'translation'` through an entire `analyze()` call, but `analyze()`
    itself always resets `state.active` to `"overview"` partway through — existing, unchanged
    Milestone 1 behavior. The test was corrected to simulate a realistic mid-analysis tab switch
    instead, which is what actually exercises `updateTranslationRow`'s live-patching path). A
    separate, additional script investigated and confirmed the save-mid-analysis finding above.
    Both scripts kept as local scratch files only, not committed to the repository, per
    `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None to the specified behavior of `analyze()`/
  `updateTranslationRow`/`retrySentence` — all three match §2/§3/§5/§7/§8 exactly. The bundling of
  `retrySentence` into this task, and the decision not to fix ROBUST-1 here, are both disclosed
  judgment calls, not silent deviations.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in prior
  session entries; left untouched again this session for the same role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered.
- **Handoff:** Per explicit instruction, this session stops after Task 6 for independent QA. See
  `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 6

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`,
  `.ai-company/CODING_STANDARDS.md`, `.ai-company/TESTING_STANDARDS.md`,
  `docs/milestones/milestone-02/02-ARCHITECTURE.md` (§2/§3/§5/§7/§8 specifically re-read),
  `docs/milestones/milestone-02/03-IMPLEMENTATION-LOG.md`, and
  `docs/milestones/milestone-02/04-QA-REPORT.md` (all re-read this session per operator
  instruction — confirmed Dev Task 5 QA pass: PASS, with ROBUST-1's reachability re-assessed and
  kept Low, and the latency/duplicate-request items still open), and the current `index.html`
  (re-read the full `analyze()` function and surrounding comments before editing).
- **Scope completed:** Dev Task 6 only — `analyze()` rewritten for progressive dispatch,
  `updateTranslationRow(i)` and `retrySentence(i)` added, old `translateSentence()` removed as dead
  code. `loadSaved()`'s defensive read and the CSS style-block additions remain not started, plus
  the newly-discovered save-mid-analysis gap noted above.
- **Files changed:** `index.html` only.
- **Commits created:** `9dd5ae8` — "Add: analyze() progressive translation dispatch,
  updateTranslationRow(), retrySentence() — Milestone 2 Dev Task 6".
- **Tests performed:** See "Tests performed" above — 30/30 assertions passed via a throwaway jsdom
  script exercising the real, end-to-end `analyze()` flow; a second scratch script confirmed the
  save-mid-analysis finding.
- **Unresolved risks:** DOC-1 and ROBUST-1 (both Low) remain open and deferrable, status
  re-confirmed unchanged by this task. The ~33s worst-case provider-chain latency is **no longer
  purely theoretical** — `translateSentenceReliable` is now genuinely reachable from production use
  of `analyze()`; QA should re-assess this against real usage now that it's live. Duplicate
  in-flight requests for identical sentences: also now live-reachable for the first time (a passage
  with a repeated sentence, translated concurrently, could double-fire) — previously only a
  theoretical concern about future wiring, now an actual property of shipped code; recommend QA
  specifically test this scenario. **New: the save-mid-analysis stuck-loading-row gap**, detailed
  above, not fixed in this task, needs a CEO/Architect decision on which task absorbs the fix.
- **Next agent:** QA Engineer (independent QA, per explicit instruction).
- **Explicit stop point:** Awaiting independent QA review of Dev Task 6 (commit `9dd5ae8`) before
  any further Development work on this milestone.

### Session: 2026-08-06 — Bug fix: SAVE-1 (High) — saveCurrent() could persist 'loading' translations

- **Scope addressed:** Independent QA's Dev Task 6 review (`04-QA-REPORT.md`, QA pass against
  `9dd5ae8`/`a7df167`) found defect **SAVE-1 (High)**: because Dev Task 6 assigns `state.analysis`
  and calls `render()` before translations resolve (by design, for progressive rendering),
  `saveCurrent()` — unmodified since before Milestone 2 — could persist a study set whose
  `translations` were still in their `{status:'loading', ko:null}` shape. Reopening such a save via
  `loadSaved()` rendered a permanently stuck `"번역 중…"` row with **no retry button** (only
  `status:'error'` rows render one in `translationRowTemplate`), leaving no recovery path. Per the
  CEO's explicit decision, this session fixes **only** SAVE-1 — no Dev Task 7 work is included.
- **Solution chosen (of the three CEO-approved options):** "Prevent Save while any translation is
  loading." Chosen over "wait until translations finish before saving" (would require tracking the
  in-flight `analyze()` promise as new state — more invasive) and "save only completed translations
  and mark incomplete items recoverable" (would require changing the saved data shape and/or
  `translationRowTemplate`'s rendering rules — broader surface). The chosen option is the narrowest
  possible fix: a single guard clause inside `saveCurrent()` itself, touching no other function and
  requiring no new data shape.
- **What was implemented:** `saveCurrent()` now checks
  `state.analysis.translations.some(t=>t.status==="loading")` immediately after its existing
  `!state.analysis` guard; if true, it shows a toast ("한글 번역이 진행 중입니다. 번역이 완료된 후
  저장해 주세요.") and returns without saving — mirroring the existing early-return pattern already
  used one line above it. `status:'error'` entries are deliberately **not** blocked — they were
  never SAVE-1's problem, since `translationRowTemplate` already renders a working retry button for
  `'error'` regardless of when the item is reopened; only `'loading'` entries had no recovery path.
- **Files changed:** `index.html` (12 lines added, 0 removed — confirmed via `git show a86f8e0
  --stat`). Only `saveCurrent()`'s guard clause and its preceding explanatory comment were added;
  no other function modified.
- **Commits:**
  - `a86f8e0` — "Fix: SAVE-1 (High) — saveCurrent() could persist translations still in 'loading'
    state"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Throwaway jsdom script (not
  committed, per standard), testing against the real `saveCurrent()`/`loadSaved()`/`analyze()`
  functions, not isolated units:
  - Confirmed the fix: calling `saveCurrent()` while translations are genuinely still `'loading'`
    (fetch mocked to never resolve) adds nothing to the saved list.
  - Confirmed normal saving still works once translations complete, and the saved item's
    translations are the real, resolved values.
  - Confirmed `saveCurrent()` is **not** blocked by `status:'error'` entries — only `'loading'`
    triggers the guard.
  - Confirmed retry on a reopened saved item with an `'error'` entry still works end-to-end
    (`retrySentence` behavior unchanged).
  - Confirmed backward compatibility: a simulated pre-Milestone-2 saved item (no `status` field at
    all on its translations) still loads and renders correctly — this fix does not touch
    `loadSaved()` or the load path at all.
  - Regression: re-verified Dev Task 6's own behavior is fully intact — progressive rendering
    (`state.analysis` assigned early with `'loading'` translations), the concurrency limit,
    `analyzeBtn`'s disable/enable timing, and the final `#status` summary are all unchanged.
  - **Result: all 15 assertions passed.** Script kept as a local scratch file only, not committed
    to the repository, per `TESTING_STANDARDS.md`.
- **Deviations from architecture:** None — this is a bug fix responding to a QA-found defect, not
  an architecture-scoped task; per `.ai-company/DEVELOPER.md` item 5 ("address only the reported
  defects within scope — don't use it as an opportunity to make unrelated changes"), no Task 7
  functionality was implemented and no other function was touched.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in prior
  session entries; left untouched again this session for the same role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered.
- **Handoff:** Per explicit instruction, this session stops after the SAVE-1 fix for independent
  QA. See `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Bug fix SAVE-1

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/DEVELOPER.md`,
  `docs/milestones/milestone-02/04-QA-REPORT.md` (the Dev Task 6 QA pass reporting SAVE-1, read in
  full for exact reproduction steps and severity rationale), and the current `index.html`
  (`saveCurrent()`, `loadSaved()`, and `translationRowTemplate()`, re-read before editing).
- **Scope completed:** SAVE-1 only, per explicit CEO decision. No Dev Task 7 work started.
- **Files changed:** `index.html` only.
- **Commits created:** `a86f8e0` — "Fix: SAVE-1 (High) — saveCurrent() could persist translations
  still in 'loading' state".
- **Tests performed:** See "Tests performed" above — 15/15 assertions passed via a throwaway jsdom
  script covering the fix itself, the unaffected `'error'`-entry save path, retry-after-reload,
  backward compatibility with pre-Milestone-2 saves, and full Dev Task 6 regression.
- **Unresolved risks:** The two Low-severity items from the Dev Task 6 QA pass (worst-case
  provider latency — informational, confirmed expected; duplicate in-flight requests for identical
  sentences — Low, cosmetic quota inefficiency) are unaffected by this fix and remain open,
  deferrable per `.ai-company/DEFINITION_OF_DONE.md`. DOC-1 and ROBUST-1 (both Low, from earlier
  passes) also remain open, unaffected. Dev Task 7 (and the still-unstarted `loadSaved()`
  defensive-read and CSS style-block work) remain not started.
- **Next agent:** QA Engineer (independent QA, per explicit instruction).
- **Explicit stop point:** Awaiting independent QA review of the SAVE-1 fix (commit `a86f8e0`)
  before Dev Task 7 may begin.

### Session: 2026-08-07 — Dev Task 7: loadSaved() defensive status-default read

- **Scope addressed:** `02-ARCHITECTURE.md` §8's `loadSaved()` row only, per explicit CEO
  instruction to begin Task 7 and not Task 8. Context confirmed before starting: Tasks 1–6 are
  complete, and SAVE-1 has been independently verified as Resolved (PASS) in `04-QA-REPORT.md`.
- **Problem:** Saved study sets created before Milestone 2 have `translations` entries shaped only
  as `{en, ko}` — no `status` field at all. `translationRowTemplate` (Dev Task 4/5) branches on
  `x.status`, so an entry with `status: undefined` falls through to neither the `'loading'` nor the
  `'error'` branch's intended meaning; it happens to render via the default (success) branch today,
  but relying on `undefined` matching that branch by accident is fragile and not what
  `CODING_STANDARDS.md`'s "handle localStorage reads defensively" rule calls for.
- **What was implemented:** `loadSaved(id)` now maps over `x.translations` (if it is an array) and
  defaults any entry whose `status` is falsy to `status: "success"`, leaving entries that already
  have a truthy `status` (`'success'` or `'error'`) completely untouched. This is applied only to
  the in-memory object assigned to `state.analysis` — the underlying `localStorage` record is never
  rewritten, so saved data on disk is left exactly as it was. An `Array.isArray` guard prevents a
  crash if `translations` is missing or malformed on an old or hand-edited record.
- **Files changed:** `index.html` (9 lines added, 0 removed — confirmed via `git show 77b82ac
  --stat`). Only `loadSaved()` and its preceding explanatory comment were touched; `deleteSaved()`
  and every other function are unchanged (confirmed via `git diff` before committing).
- **Commits:**
  - `77b82ac` — "Add: loadSaved() defensive status-default read — Milestone 2 Dev Task 7"
- **Tests performed (per `.ai-company/TESTING_STANDARDS.md`):** Freshly authored throwaway jsdom
  script (`verify-task7.js`, not committed), run against the real `index.html` via the established
  `window.__QA__` bridge technique:
  - A legacy item with translations missing `status` entirely is defaulted to `'success'` in
    memory, keeps its original `ko` text, and renders via the normal success row (no loading
    skeleton, no error message).
  - `localStorage` itself is confirmed byte-identical before and after `loadSaved()` — the patch is
    read-only/in-memory, as intended.
  - An item whose translations already carry an explicit `status` (tested with `'error'`) is left
    unchanged by the new guard; its retry button still renders and `retrySentence()` still works
    end-to-end afterward.
  - A malformed record (`translations: null`) does not throw thanks to the `Array.isArray` guard.
  - Full save→load round-trip of a normal (non-legacy) item produced by the current `analyze()` is
    unaffected.
  - Regression: re-verified Tasks 1–6 (constants, `simpleTranslate` contract, `translateSentenceReliable`/
    `translateLingva`, `translationRowTemplate`, `translation(a)` wiring, `analyze()`'s progressive
    dispatch/concurrency limit/final status summary) and the SAVE-1 fix (saving is still blocked
    while any translation is `'loading'`) are all unaffected.
  - Security: a legacy item containing malicious `en`/`ko` text is still fully escaped after the
    defensive read — no new injection surface introduced.
  - **Result: all 19 assertions passed.** Script kept as a local scratch file only, not committed.
- **Deviations from architecture:** None. The implementation matches §8's `loadSaved()` row exactly
  as scoped; no other function was touched, and no Task 8 (CSS style-block) work was started.
- **Working tree note:** `docs/milestones/milestone-02/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, and
  `04-QA-REPORT.md` continue to carry the same pre-existing uncommitted content noted in every prior
  session entry; left untouched again this session for the same role-separation reason.
- **Operational note:** none this session — no stale `.git/index.lock` encountered.
- **Handoff:** Per explicit instruction, this session stops after Task 7 for independent QA. Task 8
  is explicitly not started. See `.ai-company/HANDOFF_PROTOCOL.md` fields below.

## Handoff — Milestone 2, Dev Task 7

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/CODING_STANDARDS.md`,
  `.ai-company/DEFINITION_OF_DONE.md`, `docs/milestones/milestone-02/01-PM-SPEC.md`,
  `02-ARCHITECTURE.md` (§8's `loadSaved()` row), `03-IMPLEMENTATION-LOG.md`, `04-QA-REPORT.md`, and
  the current `index.html` (`loadSaved()`, `translationRowTemplate()`, `saveCurrent()`, re-read
  before editing).
- **Scope completed:** Dev Task 7 only — `loadSaved()`'s defensive status-default read. Task 8 (CSS
  style-block additions) explicitly not started, per instruction.
- **Files changed:** `index.html` only.
- **Commits created:** `77b82ac` — "Add: loadSaved() defensive status-default read — Milestone 2
  Dev Task 7".
- **Tests performed:** See "Tests performed" above — 19/19 assertions passed via a freshly authored
  throwaway jsdom script covering the defensive read itself, non-interference with already-tagged
  entries, malformed-data safety, the read-only/no-localStorage-rewrite property, security/escaping,
  and full regression across Tasks 1–6 and the SAVE-1 fix.
- **Unresolved risks:** None introduced by this task. Previously open Low-severity items (DOC-1,
  ROBUST-1, worst-case provider latency, duplicate in-flight requests) are unaffected and remain
  open, deferrable per `.ai-company/DEFINITION_OF_DONE.md`. Dev Task 8 (CSS style-block additions
  for `.sentence.loading`/`.sentence.error`) remains the only unstarted row in §8's file-by-file
  plan.
- **Next agent:** QA Engineer (independent QA).
- **Explicit stop point:** Awaiting independent QA review of Dev Task 7 (commit `77b82ac`) before
  Dev Task 8 may begin.

## Handoff log

_(Handoffs per `.ai-company/HANDOFF_PROTOCOL.md` appended here in chronological order.)_

1. Dev Task 1 handoff — see above (2026-08-06).
2. Dev Task 2 handoff — see above (2026-08-06).
3. Dev Task 3 handoff — see above (2026-08-06).
4. Dev Task 4 handoff — see above (2026-08-06).
5. Dev Task 5 handoff — see above (2026-08-06).
6. Dev Task 6 handoff — see above (2026-08-06).
7. Bug fix SAVE-1 handoff — see above (2026-08-06).
8. Dev Task 7 handoff — see immediately above (2026-08-07).
