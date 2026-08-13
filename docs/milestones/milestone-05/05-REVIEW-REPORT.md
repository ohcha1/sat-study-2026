# Milestone 5 — Principal Review Report: Practice Loop Closure + Vocabulary UI Integration

Independent Principal Review. Branch `feature/practice-loop-closure`, reviewed at HEAD `bb8f717`
(confirmed unchanged before and after review). No application code modified during this review —
confirmed via `git status` before and after.

**Method:** Read `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`, `04-QA-REPORT.md`
in full (all four authored/reviewed earlier this session), then independently re-read the actual code
— the full `git diff origin/main..HEAD -- index.html`, the integration commit `bb8f717` in isolation,
and the specific safety-critical lines (`gradeRetry()`'s skip logic, `captureRetryAttempt()`'s
tagging, `renderReportAsync()`'s filter, `biasForWeakTypes()`'s cap math, both evidence-sentence
render sites) — rather than re-trusting the QA report's claims. Per instruction, this is a targeted
code-level review, not a full QA re-run.

## 1. Verdict

**APPROVE**

Every safety-critical property independently re-verified by direct code inspection, not by trusting
prior reports. The Vocabulary UI integration is narrowly and correctly scoped. No unrelated changes
found anywhere in the branch's full diff against `origin/main`.

## 2. Retry architecture

Independently confirmed by reading `gradeRetry()` directly (`index.html:4468-4485`):

```js
state.analysis.questions.forEach((q,i)=>{
 if(q._firstCorrect){ originalCorrect++; return; }
 retryTotal++;
 ...
```

The skip is unconditional and happens before any DOM read — a correct question can never be
re-graded or re-recorded, regardless of DOM state (matching the QA report's own adversarial
DOM-manipulation finding, independently re-derived here from the source rather than re-trusted).
`captureRetryAttempt()` (`4803-4832`) tags every retry record `attemptKind:"retry"` with
`retryOfAttemptId:q._firstAttemptId||null`; `captureSatAttempt()` (`4778`) explicitly tags originals
`attemptKind:"original"`. The toast (`4484`) names both scores explicitly — the "clearly distinguish"
requirement is satisfied in the UI, not only the data model.

## 3. Evidence Sentence

Confirmed by direct inspection of both render sites:

- Live quiz (`sat()`, `4435`): `${q.evidenceSentence?`<div class="small" ...><b>근거 문장:</b>
  ${escapeHtml(q.evidenceSentence)}</div>`:""}` — escaped, guarded, no internal field name printed.
- Wrong-answer report (`4924`): same pattern, `${w.evidenceSentence?`<br><b>근거 문장:</b>
  ${escapeHtml(w.evidenceSentence)}`:""}`.

No XSS regression: `evidenceSentence` is the one question field `makeQuestions()` doesn't escape
upstream (confirmed in Architecture §4's own finding, re-confirmed here still true in the current
diff), so both render-site `escapeHtml()` calls are load-bearing, not redundant — and both are
present.

## 4. Weak-area reinforcement

`biasForWeakTypes()` (`index.html:3362-3369`) read directly:

```js
if(!weakTypes||!weakTypes.length) return built.slice(0,Math.min(10,Math.max(0,n)));
const cap=Math.max(1,Math.ceil(n*0.35));
```

Confirms: uses existing history via `computeWeakTypes()` (reads `satAttempts`, excludes
`attemptKind:"retry"` at `3345`, matching §3's design); cap is exactly `Math.ceil(n*0.35)`, the
approved 35%; the empty-`weakTypes` branch returns byte-identical output to the pre-Milestone-5
unbiased slice — fail-open by construction, not a special case; the final re-sort into `QG_PRIORITY`
order (confirmed present at `3368`) preserves diversity/display order regardless of bias. `analyze()`
gates the lookup behind `if(auth.userId)`, so guest sessions never attempt it — confirmed this remains
the only guard (no second guest-specific branch was added anywhere).

## 5. Report statistics

`renderReportAsync()` (`4897`): `const attempts=attemptsAll.filter(a=>a.attemptKind!=="retry");` —
confirmed this filtered variable, not `attemptsAll`, feeds every downstream aggregate (`total`,
`correct`, `trend`, `typeStats`, `skillStats`). `wrongAnswers` is queried separately
(`idbGetAllByUser("wrongAnswers",...)`) and not filtered — confirmed a still-wrong retry remains
reviewable, matching the QA report's stronger maximal-improvement test. Backward compatibility:
`attemptKind` is absent on every pre-Milestone-5 record, and `undefined!=="retry"` evaluates `true`,
so historical data is unaffected — confirmed this is genuinely how the filter behaves, not asserted.

## 6. Vocabulary UI integration

Independently re-verified, not re-trusted from the integration commit's own message:

- `diff <(sed -n '/^function vocabCardTemplate/,/^function vocab(a)/p' index.html) <(git show
  4725fdf:index.html | sed -n '/^function vocabCardTemplate/,/^function vocab(a)/p')` → **empty
  diff**, i.e. byte-identical to the CEO-accepted reference.
- The integration commit (`bb8f717`) touches exactly 2 hunks (the CSS block, and
  `vocabCardTemplate()`) — confirmed via `git show bb8f717 -- index.html | grep '^@@'` — and contains
  zero function additions/removals — confirmed via grep for `^+function`/`^-function`, none found.
- No SAT Quiz function appears anywhere in the integration commit's diff.
- Live re-verification (this review, not re-running the Developer's own script): Word Book save,
  Review-state cycling, and the per-word concurrency hot-fix (`withVocabProgressLock`) all confirmed
  present and functioning; `.word-card-left`/`.word-card-right`/`.level-chip` structure confirmed
  present in rendered output.
- Milestone 05 SAT logic confirmed non-regressed: retry button/lock/toast, evidence-sentence
  rendering, and `computeWeakTypes`/`biasForWeakTypes` all re-exercised successfully after the
  integration commit, independent of the Developer's own post-integration checks.

## 7. Preservation

Confirmed via the full branch diff against `origin/main` (`git diff origin/main..HEAD --
index.html`), function-by-function: the *only* functions touched across the entire branch are
`makeQuestions`, `sat`, `gradeQuiz`, `captureSatAttempt`, `renderReportAsync`, `vocabCardTemplate`,
plus the wholly new functions (`computeWeakTypes`, `biasForWeakTypes`, `maybeShowRetryButton`,
`retryMissedQuestions`, `gradeRetry`, `captureRetryAttempt`). Nothing else appears in the diff —
independently confirmed, not assumed:
- All 11 `QG_GENERATORS` present and unchanged (grep-confirmed: 11 entries, `QG_PRIORITY` 11 entries).
- Scoring (`gradeQuiz`'s correct/wrong classing) and answer-position assignment (`pos=i%4`, untouched
  by any hunk) unchanged.
- Gemini Summary / Gemini vocab-context: `registerProvider(` confirmed exactly 2 invocations
  (`legacy-translation`+`gemini`) — no new provider, matching every prior baseline in this project.
- Translation architecture, PDF/OCR/HEIC import: no functions in these areas appear anywhere in the
  branch diff.
- Risk A/B/C: none of `analyzeInFlight`, `fetchWithTimeout`/`REQUEST_TIMEOUT_MS`, or the save-ID
  uniqueness loop appear in the diff — untouched.
- IndexedDB 6-store schema: `grep -c "createObjectStore\|createIndex"` against the branch diff
  returns **0** — the schema definition itself is not present anywhere in this branch's changes.
- Responsive/mobile: the Vocabulary UI's reused `.btn`/`.choice` mobile touch-target rules and the
  SAT tab's own mobile CSS are both untouched by any hunk in this diff.

## 8. Git / scope

- No unrelated code changes: confirmed via the function-level diff enumeration in §7 — every touched
  function maps directly to either an approved Milestone 5 Developer Task or the approved Vocabulary
  UI integration.
- Gold Master unchanged: checksum reconfirmed `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`,
  matching every prior recording this session.
- No secrets: pattern search across the integration commit (`bb8f717`) and spot checks across the
  full branch diff found no API keys, tokens, or credentials.
- Integration commit scoped and safe: confirmed independently (§6) — 2 hunks, zero function
  churn, byte-identical resulting template.

## 9. Findings by severity

**BLOCKER:** none. **HIGH:** none. **MEDIUM:** none. **LOW:** none.

No findings. Every property specified in this review's scope was independently confirmed true by
direct inspection of the current source, not by re-trusting the Implementation Log or QA Report.

## 10. Release concerns

None. This milestone is suitable for release packaging as-is. No condition attached to this verdict.

## 11. Files modified by this review

**None.** `git status` confirmed identical before and after (`bb8f717`). This report file is the
only addition.

## 12. Is the milestone suitable for release packaging?

**Yes**, unconditionally. All CEO-required behaviors independently re-confirmed; Vocabulary UI
integration independently confirmed byte-identical to the accepted reference and correctly scoped;
no schema/provider/secret/scope-expansion concerns.

---

## Handoff — Milestone 5 Principal Review

- **Milestone:** Milestone 5 — Practice Loop Closure: Retry & Reinforcement (+ Vocabulary UI
  integration).
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/REVIEWER.md`, `docs/milestones/milestone-05/{01-PM-SPEC,02-ARCHITECTURE,
  03-IMPLEMENTATION-LOG,04-QA-REPORT}.md`, plus direct reads of `index.html`
  (`gradeRetry`, `captureRetryAttempt`, `captureSatAttempt`, `renderReportAsync`,
  `biasForWeakTypes`, `computeWeakTypes`, `sat()`'s evidence-sentence render, `vocabCardTemplate`)
  and the full diff against `origin/main` and against the integration commit `bb8f717` specifically.
- **Scope completed:** All 7 requested review areas independently verified by direct code
  inspection — retry architecture, evidence sentence, weak-area reinforcement, report statistics,
  Vocabulary UI integration, broad preservation, and git/scope safety. Not a full QA re-run, per
  instruction.
- **Files changed:** `docs/milestones/milestone-05/05-REVIEW-REPORT.md` (new, this file). No
  application code touched — confirmed via `git status` before and after.
- **Commits created:** None this session.
- **Tests performed:** Code-level inspection throughout, no live re-testing beyond direct grep/diff
  verification, per the "do not rerun full QA" instruction.
- **Unresolved risks:** None found.
- **Next agent:** CEO — to decide on release packaging / push approval.
- **Explicit stop point:** No push, no merge, no code changes, per instruction.
