# 01-PM-SPEC.md — Milestone 2: Translation Reliability

**Status: APPROVED**

## CEO Approval

- **Decision:** Approved.
- **Recorded:** 2026-08-06.
- **CEO statement (verbatim):** "PM Specification Approved. Architecture phase may begin. Proceed
  with Architecture Design. Do not implement code. Do not commit. Return only Architecture
  documentation. Wait for CEO approval."
- **Effect:** Per `.ai-company/WORKFLOW.md`, Architecture Design may begin. Development remains
  blocked until `02-ARCHITECTURE.md` clears its own CEO approval gate.

## Finalization note (read first)

This is the finalized version of the DRAFT spec produced earlier for this milestone. All four open
questions from that draft have been resolved below using **only** `DEVELOPMENT_PLAN.md`, the
existing repository (including `.ai-company/` role definitions), and the current implementation in
`index.html` — no external sources were introduced, and no new features were added beyond what
`DEVELOPMENT_PLAN.md`'s Milestone 2 section already states. Where a question could not be resolved
from those three sources, it is labeled **ASSUMPTION** below rather than presented as a settled
requirement, per `.ai-company/PM.md`'s rule against inventing requirements.

## Milestone goal

Make the translation feature behave predictably for passages beyond the built-in sample passage.
(Source: `DEVELOPMENT_PLAN.md`, Milestone 2 goal statement, verbatim.)

## In scope

Transcribed from `DEVELOPMENT_PLAN.md`'s Milestone 2 bullet list, with resolution notes added
where the original draft had left an open question:

1. Replace the silent "번역 서비스를 불러오지 못했습니다" placeholder — which currently renders in
   `translation()` as if it were a real translation (confirmed in `index.html`: `simpleTranslate()`
   returns this string as `x.ko` on no-match, with no distinct handling downstream) — with a
   clearly labeled error/retry state.
2. Add request batching/throttling for the per-sentence MyMemory API calls (currently issued via
   `Promise.all(sents.map(s=>translateSentence(s)))` in `analyze()`, with no concurrency limit) to
   avoid bursts on long passages. **Resolved:** the specific concurrency/spacing values are an
   Architecture-owned parameter, not specified in this spec — `DEVELOPMENT_PLAN.md` does not state
   a numeric limit, and selecting one is a technical decision outside PM authority per
   `.ai-company/PM.md` ("do not make architecture or implementation decisions").
3. Add a local caching layer (session-level) so re-analyzing the same passage doesn't re-request
   identical sentences. **Resolved:** "session-level" means in-memory, scoped to the current page
   load, cleared on reload/navigation — not persisted via `localStorage`. See Assumption A1.
4. Evaluate and document a fallback/secondary translation source for when MyMemory is unavailable
   or rate-limited. **Resolved:** this item requires a written evaluation and recommendation only,
   not implementation of a second provider, and candidate provider selection belongs to
   `02-ARCHITECTURE.md`, not this spec. See Assumption A2.
5. Add a visible loading state per sentence (not just the global status line) so partial
   translation progress is legible.

## Out of scope

Confirmed against `DEVELOPMENT_PLAN.md`'s explicit per-milestone scoping and its "Sequencing
Notes" section (which states Milestones 1–4 proceed in order as independent, logically distinct
milestones):

- Any change to the dictionary/vocabulary source (`dictionary`, `ipaMap`, `exampleBank`,
  `levelFor`, `fallbackKoreanGloss`) — that is Milestone 3's stated scope.
- Any UI/visual redesign beyond what's needed to show the error/degraded/loading states this
  milestone requires — general UI/accessibility work is Milestone 4's stated scope.
- Any change to grammar insight generation (`grammarInsights()`) or SAT question generation
  (`makeQuestions()`) — that is Milestone 5's stated scope.
- Switching the *primary* translation provider away from MyMemory outright. Item 4 above requires
  only that a fallback/secondary source be evaluated and documented, not adopted as primary.
- See Assumption A4 for the basis of this list.

## Acceptance criteria

Testable, pass/fail statements, finalized from the draft's directional bullets:

1. Given a translation request that fails (network error, non-OK API response, or a request that
   times out/never resolves to a usable translation), the UI shows a distinct, clearly labeled
   failure state for that sentence — never the untranslated original or a canned string rendered as
   if it were a real translation. Verifiable by mocking `fetch` to reject or return a non-OK
   response for `translateSentence()` and asserting the rendered `translation()` output for that
   sentence is visually/textually distinct from a successful translation row.
2. Given a passage with enough sentences that all translation requests would previously have fired
   simultaneously, requests are issued through a bounded-concurrency mechanism (a fixed, configurable
   maximum number of in-flight requests) rather than all at once. Verifiable by instrumenting or
   mocking `fetch` and asserting the number of concurrently in-flight requests never exceeds the
   configured maximum at any sampled instant. The maximum itself is an Architecture-defined
   parameter (see In Scope, item 2).
3. Given the same passage analyzed twice within one page session (no reload in between), the second
   analysis does not re-issue translation requests for sentences whose exact text was already
   successfully translated earlier in that session. Verifiable via network request count or a
   fetch mock/spy showing zero new calls for unchanged, previously-succeeded sentences.
4. A written evaluation exists (in `02-ARCHITECTURE.md` or a linked document) comparing at least
   one candidate fallback/secondary translation source against MyMemory, with an explicit
   recommendation (adopt / defer / reject). This criterion is satisfied by the evaluation existing,
   not by a second provider being integrated, per the "evaluate and document" phrasing in
   `DEVELOPMENT_PLAN.md`.
5. During translation, each sentence that has not yet returned a result shows its own loading
   indicator, distinguishable from sentences that have already completed (successfully or as a
   failure). Verifiable by delaying mock responses and asserting per-row loading state is present
   before resolution and absent after.

## Resolution of prior open questions

The DRAFT spec listed four open questions for the CEO. Per instruction, all four are resolved here
using only `DEVELOPMENT_PLAN.md`, the repository, and the current implementation:

1. **"What is the actual MyMemory API rate limit to design against?"** — Resolved by removing the
   need for a specific number from this spec: `DEVELOPMENT_PLAN.md` does not state one, and
   choosing a numeric threshold is an Architecture-level implementation decision, not a PM-spec
   requirement (per `.ai-company/PM.md`). Acceptance criterion 2 is now written to be testable
   without a hardcoded number — it tests that a bound exists and is respected, not what the bound's
   value is.
2. **"Does 'session-level' caching mean in-memory or `localStorage`?"** — Resolved: in-memory,
   scoped to the current page load. Basis: the existing implementation already draws this exact
   distinction — `state.analysis` (ephemeral, in-memory, lost on reload) versus the saved-study-sets
   feature (`saveCurrent()`/`getSavedList()`/`localStorage`, deliberately persistent). Reusing
   `localStorage` for translation caching would blur a distinction the codebase already makes on
   purpose. See Assumption A1 for the residual interpretive risk.
3. **"Does the PM spec name fallback-provider candidates, or is that Architecture's job?"** —
   Resolved: Architecture's job. `.ai-company/PM.md` explicitly prohibits the PM role from making
   architecture or implementation decisions and instructs describing *what*, not *how*. Naming a
   specific third-party API is a *how* decision.
4. **"Confirm the out-of-scope list matches intent."** — Resolved by direct reference to
   `DEVELOPMENT_PLAN.md`'s own milestone boundaries and Sequencing Notes, which state Milestones
   1–4 are independent and sequential; each excluded item above is explicitly the stated scope of a
   different, later milestone in the same document. See Assumption A4 for the one residual
   inference this still requires.

## Assumptions

Every item below is an inference, not a verbatim requirement from `DEVELOPMENT_PLAN.md`. Each is
labeled and justified individually per `.ai-company/PM.md`'s rule that assumptions must never be
presented as settled requirements.

- **A1 — "Session" scope.** Assumed to mean "current page load/tab, cleared on reload or
  navigation away," not "browser session across tabs" and not persisted to disk. `DEVELOPMENT_PLAN.md`
  uses the word "session-level" without further definition, and no other repository document
  defines it. The assumption is grounded in the existing code's own in-memory-vs-`localStorage`
  precedent (see Resolution 2 above), but remains an inference rather than a quoted requirement.
- **A2 — "Evaluate and document" excludes implementation.** Assumed to require a written comparison
  and recommendation only. Basis: `DEVELOPMENT_PLAN.md` uses "Evaluate and document" for this item
  specifically, in contrast to action verbs like "Replace," "Add," and "Integrate" used elsewhere in
  the same document for items that do require implementation (e.g., Milestone 3's "Integrate a real
  dictionary data source"). This is a textual pattern inference, not an explicit statement scoping
  item 4 to documentation-only.
- **A3 — Retry granularity is per-sentence.** `DEVELOPMENT_PLAN.md` item 1 says "clearly labeled
  error/retry state" but does not specify whether retry re-runs the full passage or just the failed
  sentence. Assumed to be per-sentence, since a full-passage retry would conflict with item 3's goal
  (avoid re-requesting already-successful sentences) and with item 5's per-sentence loading model.
  Not stated explicitly anywhere in `DEVELOPMENT_PLAN.md` or the current implementation (which has
  no existing retry pattern to model this on).
- **A4 — Out-of-scope boundaries.** Treated as confirmed (see Resolution 4), but the underlying
  inference — that adjacent milestones' stated scope implies Milestone 2's exclusion of that same
  work — is still an inference from the plan's structure, not a line in `DEVELOPMENT_PLAN.md` that
  explicitly enumerates Milestone 2's exclusions. Confidence is high because it is drawn directly
  from the plan's own document, not from external judgment.
- **A5 — Visual treatment is unspecified by design.** The exact visual presentation of the new
  error/failure and loading states (colors, icons, copy wording) is intentionally left to
  Architecture/Development, to be made consistent with the existing design system already visible
  in `index.html` (card color variants such as `c-blue`/`c-mint`, the `.status` line, `toast()`).
  This spec requires only that the states be *distinguishable*, not how they look, per PM's *what
  not how* boundary.

## Exit criteria (from `DEVELOPMENT_PLAN.md`, verbatim)

"Any arbitrary English passage (not just the sample) produces either a real translation or an
honest, clearly-marked failure state — never a misleading canned string."

---

## Handoff — Milestone 2

- **Milestone:** Milestone 2 — Translation Reliability
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/PM.md`, `.ai-company/HANDOFF_PROTOCOL.md`, `DEVELOPMENT_PLAN.md`,
  `docs/milestones/milestone-02/README.md`, the prior `docs/milestones/milestone-02/01-PM-SPEC.md`
  (DRAFT), and the current `index.html` (translation-related functions: `translateSentence()`,
  `simpleTranslate()`, `translation()`, `analyze()`, and the saved-materials `localStorage`
  functions used for the A1 comparison).
- **Scope completed:** Finalized `01-PM-SPEC.md` — removed DRAFT status, resolved all four open
  questions from the prior draft using only `DEVELOPMENT_PLAN.md`/repository/implementation,
  converted directional bullets into fully testable acceptance criteria, and added a separate,
  individually-justified Assumptions section (A1–A5). No new features or requirements were added
  beyond `DEVELOPMENT_PLAN.md`'s Milestone 2 section.
- **Files changed:** `docs/milestones/milestone-02/01-PM-SPEC.md` (rewritten in place). No other
  file modified, per instruction.
- **Commits created:** None — not committed, per instruction.
- **Tests performed:** N/A (PM phase; no code changes). Verified acceptance-criteria wording
  against the actual current implementation in `index.html` (function names, current
  `Promise.all` behavior, `simpleTranslate()`'s placeholder-string behavior, and the existing
  `localStorage` vs. in-memory `state` distinction) rather than assuming behavior.
- **Unresolved risks:** None blocking this spec. Carried forward for Architecture's attention:
  the exact concurrency/throttle threshold (AC 2) and fallback-provider candidate selection (AC 4)
  are unresolved by design — they are Architecture-phase decisions, not omissions.
- **Next agent:** CEO.
- **Explicit stop point:** Awaiting CEO approval of this finalized `01-PM-SPEC.md` at the gate
  defined in `.ai-company/WORKFLOW.md` ("PM Specification → [GATE] CEO Approval"). Architecture
  Design must not begin until that approval is recorded in this document.
