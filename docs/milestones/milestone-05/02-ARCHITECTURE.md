# Milestone 5 — Architecture Design: Practice Loop Closure (Retry & Reinforcement)

Grounded directly in the current implementation — every function/field named below was read from
`index.html` at HEAD `4725fdf` on `feature/dictionary-visual-polish` before this design was written,
not assumed from `01-PM-SPEC.md`'s description alone.

## 1. Approach summary

Three additive pieces, all living inside the existing quiz/report code paths with **no new
IndexedDB store, no new index, no new provider**:

1. **Retry** — a same-session, in-memory mechanism that resets only the missed questions' DOM state
   and grades them through a new, parallel code path that writes *new*, separately-tagged records
   instead of touching the original ones.
2. **Evidence sentence** — a pure render-time addition; the data already exists in every question
   object and is already persisted, it is simply never displayed today.
3. **Weak-area weighting** — a capped, best-effort reordering of the *existing* 11 question
   generators' output, computed from the *existing* `satAttempts` history, applied just before
   `makeQuestions()` slices its candidate list down to the student-chosen count.

All three preserve the question generators, randomization, explanations, scoring, and the report
tab's existing charts exactly as they behave today for the common case (no wrong answers, no retry,
no history) — verified explicitly in §8's regression section, not just asserted.

## 2. Affected functions (all in `index.html`)

| Function | Change |
|---|---|
| `sat(a)` | Add 2 hidden buttons (retry / retry-grade), id the existing grade button, add an escaped evidence-sentence line to the explanation block. |
| `gradeQuiz()` | Additive: stamp per-question first-attempt bookkeeping; call a new `maybeShowRetryButton()`. |
| `captureSatAttempt()` | Additive: stamp `q._firstAttemptId` from the record it just wrote, inside the existing loop. |
| `makeQuestions(text,sents,vocab,n)` → `makeQuestions(text,sents,vocab,n,weakTypes)` | New optional 5th param, default `[]`; capped-bias reorder inserted immediately before the existing `.slice(0,Math.min(10,Math.max(0,n)))`. |
| `analyze()` | Additive: one guarded, fail-open `await` to fetch `satAttempts` and compute `weakTypes` before calling `makeQuestions()`. |
| `renderReportAsync()` | Additive: filter `attempts` to exclude `attemptKind==="retry"` before computing `total`/`correct`/`wrong`/`trend`/`typeStats`/`skillStats`. `wrongAnswers` list is **not** filtered (see §3). |
| **New:** `maybeShowRetryButton()`, `retryMissedQuestions()`, `gradeRetry()`, `captureRetryAttempt()`, `computeWeakTypes(attempts)` | New functions, all small, all reusing existing helpers (`skillsForType`, `classifyErrorType`, `escapeHtml`, `idbGet*`, `sha256Hex`, `uid`). |

New CSS: one class, `.sat-retry-active` (a subtle border accent marking a question currently in
retry mode) — reuses the app's existing amber "in-progress" hue already used for
`.review-state-reviewing` (`#FEF3C7`/`#92400E`), for visual consistency rather than inventing a new
color.

## 3. Retry state model and persistence

### 3.1 In-memory state (not persisted — same lifetime as `state.analysis` itself)

No new top-level `state` property. Bookkeeping is attached directly to each question object already
living in `state.analysis.questions[i]`, using an underscore prefix to mark it as runtime-only,
outside the object shape `makeQuestions()` itself defines:

```js
q._firstSelectedIdx   // number|null — the option index chosen on the FIRST grade
q._firstCorrect       // boolean — was the first attempt correct
q._firstAttemptId      // string — the satAttempts.attemptId written for this question's first attempt
```

This mirrors how `state.analysis` already lives entirely in memory until explicitly saved
(`saveSessionForUser()`) — retry state does **not** need to survive a reload, matching the ungraded
quiz's own existing behavior (a reload already loses in-progress answers today; this does not change
that).

### 3.2 Why not a schema migration

`satAttempts` and `wrongAnswers` are plain-object IndexedDB stores (`keyPath` only, no `createIndex`
on the new fields) — arbitrary additional properties on a record require no store/index change to
write or read. Two additive fields are introduced, both optional and both absent-safe for every
historical record written before this milestone:

- **`attemptKind: "original" | "retry"`** — old records have no such field; every read site treats
  `undefined` as `"original"` (`attemptKind!=="retry"` is `true` for `undefined`), so zero behavior
  change for existing data.
- **`retryOfAttemptId: string | null`** — only ever set on `attemptKind:"retry"` records, pointing
  back at the original question's `attemptId` (from `q._firstAttemptId`). Not read by any existing
  code path; purely for future traceability, at effectively zero cost.

No `createObjectStore`/`createIndex` call changes. Confirmed by re-reading `openDB()`'s upgrade
handler — this design touches none of it.

### 3.3 Grading flow

**Original grade (`gradeQuiz()`, existing function, additively modified):** unchanged for the common
case. Per question, in the existing `forEach`, additionally records `q._firstSelectedIdx`/
`_firstCorrect` (computed from the same `picked`/`Number(picked.value)` values already being read —
no new DOM query). After the existing `captureSatAttempt(score,total)` call, `captureSatAttempt()` is
itself additively modified to stamp `state.analysis.questions[i]._firstAttemptId = rec.attemptId`
inside its existing per-question loop (the `rec` object is already being built there; this reads one
field off it, no new IndexedDB read). Finally, `maybeShowRetryButton()` runs: if `score < total`,
reveal `#satRetryBtn`; otherwise ensure it stays hidden (defensive, in case of a hypothetical second
full grade).

**Retry (`retryMissedQuestions()`, new):** for every question where `q._firstCorrect` is falsy — i.e.
originally wrong or unanswered — clear that question's radio selection and its `.correct`/`.wrong`
classes, and re-hide its explanation (`display:none`, matching the pre-grade state exactly). Add
`.sat-retry-active` to that question's container for a visible "in progress" marker. For every
question where `q._firstCorrect` is `true`, **disable its radio inputs** (`input.disabled=true`) so
they cannot be edited during retry — a UX clarity measure, not a data-integrity one: `gradeRetry()`
independently skips any question with `q._firstCorrect===true` regardless of DOM state, so a
correct question's original result can never be altered by this flow even if disabling were somehow
bypassed. Swap the visible action button from "채점하기"/`#satGradeBtn` to "다시 채점하기"/
`#satGradeRetryBtn`, hide `#satRetryBtn`.

**Retry grade (`gradeRetry()`, new, async):** iterates only the questions with `q._firstCorrect`
falsy, re-applies the existing correct/wrong classing logic from `gradeQuiz()` (same pattern, not a
copy-paste of unrelated logic — the choice-classing block is genuinely identical because it must
grade against the same `q.ans`/`q.choices`, which are never mutated by retry), computes
`retryScore`/`retryTotal` over just that subset, shows a toast naming **both** numbers explicitly —
`원래 점수: {originalCorrect} / {total} · 다시 풀기 결과: {retryScore} / {retryTotal}` — satisfying
the CEO's "clearly distinguish original vs retry result" requirement in the UI itself, not only in
the data model. Calls `captureRetryAttempt(retryScore, retryTotal)`.

**`captureRetryAttempt()` (new, async, mirrors `captureSatAttempt()`'s shape exactly):** for each
question with `q._firstCorrect` falsy, builds a record identical in shape to `captureSatAttempt()`'s
(`attemptId`, new `quizId` prefixed `retryquiz`, `userId`, `date`, `passageId` hash, question
type/text, selected/correct answer, explanation, evidence sentence, skill tags, `score` = this
retry's percentage), plus `attemptKind:"retry"` and `retryOfAttemptId:q._firstAttemptId||null`.
Writes to `satAttempts` via `idbPut`, and — matching `captureSatAttempt()`'s existing
`if(!rec.isCorrect)` branch exactly — writes to `wrongAnswers` (also tagged `attemptKind:"retry"`)
if still wrong. **Guest (`!auth.userId`): returns immediately without writing anything**, identical
to `captureSatAttempt()`'s own existing guest guard — same line, same behavior, no new special case.

### 3.4 Why `wrongAnswers` is intentionally left unfiltered

A retry that is still wrong represents a genuine, currently-unresolved gap — hiding it from "최근
오답" would make the report *less* honest, not more. A retry that becomes correct never produces a
`wrongAnswers` entry in the first place (same `if(!rec.isCorrect)` gate as today), so there is no
risk of a stale "wrong" card lingering after a successful retry. No filtering needed or applied here.

### 3.5 Why `attemptKind==="retry"` records ARE excluded from the report's aggregate stats

`renderReportAsync()`'s accuracy/monthly-trend/per-type/per-skill numbers currently answer "how did
the student do, unaided, over time." Counting a coached retry (attempted immediately after seeing the
correct answer and explanation) as an independent data point would inflate accuracy/trend in a way
that doesn't reflect genuine unaided performance, and would double-count the same question within one
sitting. Filtering `attempts.filter(a=>a.attemptKind!=="retry")` before every aggregate computation
keeps these numbers meaning exactly what they mean today. This is a judgment call, flagged explicitly
per `AUTONOMOUS_CONTINUATION_POLICY.md`'s Level-2 decision convention — reversible (the filter is one
line), low risk (only affects which records feed aggregate stats, never which records exist), and the
CEO can direct the opposite choice if retry-informed trend data is actually preferred.

## 4. Evidence sentence design

`evidenceSentence` already exists on every question object returned by `makeQuestions()` (sourced
from whichever generator produced it, e.g. `genCentralIdea()` sets `evidenceSentence:sents[idx]`) and
is already persisted verbatim into both `satAttempts` and `wrongAnswers` by `captureSatAttempt()`. It
has never been rendered anywhere.

**Critical finding, not previously flagged:** unlike `q.q`/`q.choices`/`q.why`, which all pass through
`escapeHtml(englishOnly(...))` inside `makeQuestions()`'s return mapping, `evidenceSentence` is
assigned raw (`evidenceSentence:q.evidenceSentence||""`) — it is unescaped passage text. Rendering it
via `innerHTML` for the first time in this milestone makes it a new XSS surface if not escaped at the
render site — the exact same category of risk this codebase has already fixed twice (the original
Milestone-1 XSS fixes, and `contextSentence`'s explicit escaping in the Vocabulary Card). **Mitigation
(required, not optional): `escapeHtml()` at both new render sites**, matching this codebase's
established default-to-escaping convention for any passage-derived content that wasn't already safe
by construction. `q.why`/`q.whyKo` remain unescaped-at-render-site only because `makeQuestions()`
already escaped them upstream — evidence sentence needs its own escaping specifically because it
skipped that step.

**Render sites, both wrapped in `q.evidenceSentence?...  :""` so absent evidence renders nothing (not
an empty label):**

- **Live quiz** (`sat()`'s per-question explanation block): a small line below the existing
  `why`/`whyKo` content — `<b>근거 문장:</b> ${escapeHtml(q.evidenceSentence)}` — reusing the
  existing `.explanation`/`.small` styling, no new CSS.
- **Wrong-answer report card** (`reportView()`'s `wrongCards` template): same label, same escaping,
  added to the existing `.example` block alongside "왜 틀렸는가"/"정답이 맞는 이유"/"다음에 주의할
  점" — `w.evidenceSentence` is read from the already-stored `wrongAnswers` record (both original and
  retry records carry it, per §3.3).

No internal field name is ever printed — the label is always the Korean "근거 문장:" string.

## 5. Weak-area weighting design

### 5.1 Detection — `computeWeakTypes(attempts)` (new, pure function)

```js
function computeWeakTypes(attempts){
 const typeStats={};
 attempts.filter(a=>a.attemptKind!=="retry").forEach(a=>{
  typeStats[a.questionType]=typeStats[a.questionType]||{correct:0,total:0};
  typeStats[a.questionType].total++;
  if(a.isCorrect)typeStats[a.questionType].correct++;
 });
 return Object.entries(typeStats)
  .filter(([,v])=>v.total>=3 && (v.correct/v.total)<0.6)
  .sort((a,b)=>(a[1].correct/a[1].total)-(b[1].correct/b[1].total))
  .slice(0,2)
  .map(([k])=>k);
}
```

Reuses the report tab's own established thresholds rather than inventing new ones: `total>=3` matches
`buildDeterministicSummary()`'s existing minimum-sample-size gate, and `<0.6` matches
`typeRows`/`skillRows`'s existing `acc<60?"weak":...` CSS-class threshold — "weak" already means
"under 60%" everywhere else in this codebase. Excludes retry-kind attempts from the input, consistent
with §3.5. Caps at the 2 weakest types, matching the existing `buildDeterministicSummary()` cap (not
a new number).

### 5.2 Selection — inserted into `makeQuestions()`, immediately before its existing final `.slice(...)`

```js
function biasForWeakTypes(built,n,weakTypes){
 if(!weakTypes||!weakTypes.length) return built.slice(0,Math.min(10,Math.max(0,n)));
 const cap=Math.max(1,Math.ceil(n*0.35));
 const weak=built.filter(q=>weakTypes.includes(q.type)).slice(0,cap);
 const rest=built.filter(q=>!weak.includes(q));
 const combined=[...weak,...rest].slice(0,Math.min(10,Math.max(0,n)));
 return combined.sort((a,b)=>QG_PRIORITY.indexOf(a.type)-QG_PRIORITY.indexOf(b.type));
}
```

- **Cap: `Math.ceil(n*0.35)`** — 35%, the midpoint of the CEO's specified 30–40% range, chosen for a
  single simple round-number implementation rather than a configurable range with no clear default.
  At the UI's default `n=6`: `cap=3` (50% would be the naive `Math.ceil(6/2)`; 35% keeps it to at most
  half that share). At `n=10` (the maximum): `cap=4`, i.e. 40% — right at the top of the approved
  range, still leaving 6 of 10 questions unbiased.
- **Guarantee, not replacement:** weak-type questions are only *included preferentially* up to the
  cap — they still have to exist in `built` (i.e. the passage actually supports that question type;
  each of the 11 generators independently decides this per-passage and may return `null`). If the
  passage doesn't support any weak type, `weak` is empty and the function is a no-op, falling back to
  today's exact behavior. This **fails open by construction**, not via a special-cased guard.
- **Display order preserved:** the final `.sort(...)` re-applies the existing `QG_PRIORITY` order, so
  weak-area biasing changes *which* questions are included within the cap, never the order they're
  displayed in — a student sees the same familiar section ordering every time.

### 5.3 Wiring into `analyze()`

Immediately before the existing `const questions=makeQuestions(text,sents,vocab,qcount);` line:

```js
let weakTypes=[];
if(auth.userId){
 try{ weakTypes=computeWeakTypes(await idbGetAllByUser("satAttempts",auth.userId)); }
 catch(e){ weakTypes=[]; }
}
const questions=makeQuestions(text,sents,vocab,qcount,weakTypes);
```

Guarded two ways: `auth.userId` check (guests never attempt the lookup at all — see §6) and a
`try/catch` so a rare IndexedDB failure degrades to unbiased selection rather than blocking analysis.
`analyze()` is already `async` with an existing `analyzeInFlight` guard (Risk C) wrapping this whole
body in `try/finally` — this one additional `await` sits inside that same guard, adding negligible
latency (one indexed read, already a pattern used elsewhere in this codebase) and no new failure mode
that could leave `analyzeInFlight` stuck (the `try/catch` here is local and never throws outward).

## 6. Guest / no-history fallback

| Feature | Guest (`!auth.userId`) | Logged-in, <3 attempts in any type |
|---|---|---|
| Retry | Fully functional, in-session only — nothing persisted (mirrors `captureSatAttempt()`'s existing guest guard exactly). | Fully functional and persisted, identical to any other user. |
| Evidence sentence | Fully functional — render-time only, no auth dependency at all. | Fully functional. |
| Weak-area weighting | Never attempted — `auth.userId` check short-circuits before any IndexedDB read. `weakTypes=[]` implicitly. | `computeWeakTypes()` runs but its `total>=3` gate returns `[]` — same effective no-op, arrived at naturally rather than via a second special case. |

No guest-specific or low-history-specific code branch was added beyond the single `if(auth.userId)`
check already shown in §5.3 — both fallback cases resolve to the same `weakTypes=[]` outcome that
`biasForWeakTypes()` already treats as its no-op path.

## 7. Diversity safeguards

Restated from §5.2 for visibility: the cap is the *entire* safeguard, deliberately kept to one number
rather than a more elaborate rule set, per the CEO's "keep implementation small" instruction. At the
UI's minimum `n=1`, `cap=Math.ceil(1*0.35)=1` — a 1-question quiz *could* end up 100% weak-biased if
the passage happens to support that type. This is a known, accepted edge case at the extreme low end
of the question-count selector (flagged here rather than silently engineered around with a
minimum-`n` special case that would add complexity for a selector value most students won't pick);
Developer Task 3's test plan (§9) explicitly includes `n=1`/`n=2` boundary checks so this is measured,
not just asserted.

## 8. Mobile UX

No new interactive element requires new CSS. The two new buttons (`#satRetryBtn`,
`#satGradeRetryBtn`) reuse the existing `.btn` class, which already carries a mobile
`min-height:44px` touch-target rule inside the existing responsive breakpoint. The evidence-sentence
line reuses `.small`/`.explanation`. `.sat-retry-active` is a border-only visual accent with no layout
impact. No new breakpoint, no new media query.

## 9. Risks and mitigations

1. **XSS via unescaped `evidenceSentence`** (§4) — real, not hypothetical; concretely required
   mitigation specified (`escapeHtml()` at both render sites). Developer Task 2 must include an
   injection-payload regression test, matching this codebase's established practice for every prior
   new innerHTML surface.
2. **Report-stat distortion if the `attemptKind` filter is missed** (§3.5) — mitigated by an explicit
   before/after comparison test in Developer Task 4 (retry a quiz, assert the report's
   accuracy/trend/type/skill numbers are unchanged from their pre-retry values).
3. **Locked-question tampering during retry** (§3.3) — mitigated at the data layer regardless of DOM
   state (`gradeRetry()` unconditionally skips `q._firstCorrect===true` questions); the `disabled`
   attribute is UX clarity only, not the actual safeguard.
4. **Small-`n` diversity edge case** (§7) — accepted and flagged, not silently special-cased.
5. **One additional `await` in `analyze()`** — negligible latency, `try/catch`-guarded, cannot leave
   `analyzeInFlight` stuck (§5.3).
6. **Backward compatibility of the `makeQuestions()` signature change** — confirmed via direct grep
   that `analyze()` is the *only* call site in the entire file; the new 5th parameter defaults to
   `[]`, so even a hypothetical future second call site omitting it degrades to today's exact
   behavior rather than erroring.

## 10. Developer Task breakdown

**Dev Task 1 — Retry Missed Questions.** Modify `sat()` (buttons + ids), `gradeQuiz()` (bookkeeping +
`maybeShowRetryButton()` call), `captureSatAttempt()` (stamp `_firstAttemptId`). New:
`maybeShowRetryButton()`, `retryMissedQuestions()`, `gradeRetry()`, `captureRetryAttempt()`. New CSS:
`.sat-retry-active`. Test: all-correct quiz (no retry button ever appears), partial-wrong quiz (retry
button appears, only wrong questions reset, correct questions locked/disabled, retry grading produces
a two-number toast and separate `attemptKind:"retry"` records with correct `retryOfAttemptId`
linkage), guest mode (retry fully functional in-session, nothing written to IndexedDB).

**Dev Task 2 — Evidence Sentence.** Modify `sat()`'s explanation block and `reportView()`'s
`wrongCards` template, both with `escapeHtml()`. Test: injection payload in a passage sentence that
becomes `evidenceSentence` is confirmed escaped in both locations; questions with no evidence sentence
render with no stray label.

**Dev Task 3 — Weak-Area Reinforcement.** New `computeWeakTypes()`; modify `makeQuestions()`
(5th param + `biasForWeakTypes()`) and `analyze()` (guarded lookup + wiring). Test: a student history
with a genuine weak type (≥3 attempts, <60%) measurably increases that type's presence in a
subsequently generated quiz, capped at the ~35% ceiling; guest and <3-attempt histories are unaffected
(byte-identical question selection to today); a passage that doesn't support the weak type at all
degrades silently to the unbiased path; `n=1`/`n=2` boundary behavior recorded (§7).

**Dev Task 4 — Report Persistence Filter.** Modify `renderReportAsync()`'s aggregate computation.
Test: before/after report-stat comparison across a retry (§9.2); confirm retry misses still surface
in "최근 오답" (§3.4).

**Dev Task 5 — Regression + documentation.** Full 10-tab render sweep; mobile (375px) pass on the SAT
tab specifically; re-confirm XSS protections; confirm IndexedDB still exactly 6 stores with no new
indexes; confirm Gold Master checksum unchanged; guest-mode full pass across all four pieces; confirm
a zero-wrong-answer quiz and a logged-out session are byte-for-byte behaviorally identical to the
pre-milestone baseline. Implementation log + this document's handoff updated.

## 11. Status

**APPROVED by the CEO — 2026-08-13.** Development may begin, scoped strictly to §2's function list
and the five Developer Tasks in §10, within the constraints recorded in `01-PM-SPEC.md` §8.

---

## Handoff — Milestone 5 Architecture

- **Milestone:** Milestone 5 — Practice Loop Closure: Retry & Reinforcement.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/ARCHITECT.md`, `docs/milestones/milestone-05/01-PM-SPEC.md` (confirmed APPROVED), plus
  direct reads of `index.html` (`sat()`, `gradeQuiz()`, `captureSatAttempt()`, `classifyErrorType()`,
  `reportView()`/`renderReportAsync()`, `buildDeterministicSummary()`, `makeQuestions()`,
  `QG_GENERATORS`/`QG_PRIORITY`, `genCentralIdea()` as a representative generator, `analyze()`,
  `openDB()`'s schema definition, the `.btn`/`.choice`/`.explanation` CSS and their mobile breakpoint
  rules) and a full-file grep confirming `makeQuestions()` has exactly one call site.
- **Scope completed:** Full architecture for all three approved pieces plus the report-filter
  necessity the CEO's constraints implied; concrete function-level diffs (not just descriptions);
  explicit persistence model with schema-migration justification; evidence-sentence XSS finding and
  required mitigation; weak-area cap derivation and diversity-safeguard reasoning; guest/no-history
  fallback table; mobile-CSS-reuse confirmation; 6 named risks with mitigations; 5-task Developer
  breakdown with test plans per task. No application code touched — confirmed via `git status`
  showing only documentation files changed throughout this session.
- **Files changed:** `docs/milestones/milestone-05/01-PM-SPEC.md` (§8 CEO decisions added, §6
  acceptance criteria finalized, status APPROVED), `02-ARCHITECTURE.md` (new, this file). Neither
  committed yet — pending CEO review of this Architecture, matching this repo's established PM/
  Architecture-phase pattern.
- **Commits created:** None this session.
- **Tests performed:** N/A (Architecture phase, no code written).
- **Unresolved risks:** None new beyond §9's 6 named items, all of which have assigned mitigations
  and, where relevant, an assigned Developer Task test.
- **Next agent:** Senior Developer, to implement the 5 Developer Tasks in §10 — or CEO, if further
  scope revision is wanted first.
- **Explicit stop point:** Per the CEO's "STOP after Architecture" instruction. No application code
  has been modified.
