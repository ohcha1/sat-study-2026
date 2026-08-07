# 01-PM-SPEC.md — Milestone 2: Translation Reliability

**Status: DRAFT — not yet reviewed or approved by CEO**

## Provenance note (read first)

No prior PM package for Milestone 2 was found anywhere in this repository (git history, working
tree, or existing docs) as of this drafting. This spec is therefore built **only** from verifiable
repository information: the Milestone 2 section of `DEVELOPMENT_PLAN.md` (committed at `79315cf`,
unmodified since) and a reading of the translation-related code currently in `index.html`. It does
not incorporate any requirement that isn't traceable to one of those two sources. Anything not
directly stated in `DEVELOPMENT_PLAN.md` is marked as an **assumption** or **open question** below
rather than presented as settled scope. The CEO should treat this as a starting point for review,
not a final spec.

## Milestone goal

Make the translation feature behave predictably for passages beyond the built-in sample passage.
(Source: `DEVELOPMENT_PLAN.md`, Milestone 2 goal statement, verbatim.)

## In scope

Transcribed directly from `DEVELOPMENT_PLAN.md`'s Milestone 2 bullet list, with no additions:

1. Replace the silent "번역 서비스를 불러오지 못했습니다" placeholder — which currently renders as
   if it were a real translation — with a clearly labeled error/retry state.
2. Add request batching/throttling for the per-sentence MyMemory API calls to avoid bursts on long
   passages.
3. Add a local caching layer (session-level) so re-analyzing the same passage doesn't re-request
   identical sentences.
4. Evaluate and document a fallback/secondary translation source for when MyMemory is unavailable
   or rate-limited.
5. Add a visible loading state per sentence (not just the global status line) so partial
   translation progress is legible.

## Out of scope

Not stated explicitly in `DEVELOPMENT_PLAN.md` for this milestone; the following are called out
here because they are adjacent and a reader could otherwise assume they're included. **These
exclusions are an inference from sequencing, not a direct quote — flagged for CEO confirmation:**

- Any change to the dictionary/vocabulary source (that's Milestone 3).
- Any UI/visual redesign beyond what's needed to show error/loading states for translation
  specifically (broader UI work is Milestone 4).
- Any change to grammar insight generation or SAT question generation (Milestone 5).
- Switching the *primary* translation provider away from MyMemory outright — item 4 above asks
  only to "evaluate and document" a fallback/secondary source, not to implement or switch to one.
  **Open question below.**

## Acceptance criteria (DRAFT — needs CEO/PM refinement into fully testable form)

The bullets above are directional, not yet written as testable pass/fail statements. Draft
testable versions, pending CEO confirmation of intent:

1. Given a translation request that fails (network error, API error response, or rate limit), the
   UI shows a distinct, clearly labeled failure state per sentence — never the untranslated
   original or a canned string presented as if it were a translation.
2. Given a passage with N sentences where N is large enough to previously risk bursting the
   MyMemory API, requests are batched/throttled such that [specific rate limit — **open question,
   see below**] is not exceeded.
3. Given the same passage analyzed twice in one session, the second analysis does not re-request
   translations for sentences already translated in the first — verifiable via network request
   count or a mock/spy on the fetch call.
4. A written evaluation exists (in `02-ARCHITECTURE.md` or a linked doc) comparing at least one
   fallback/secondary translation source against MyMemory, with a documented recommendation. This
   criterion is about producing the evaluation, not shipping the fallback itself, per the "evaluate
   and document" phrasing in `DEVELOPMENT_PLAN.md`.
5. During translation, each sentence that hasn't yet returned a result shows its own loading
   indicator, distinguishable from sentences that have already completed or failed.

## Open questions for CEO

- What is the actual MyMemory API rate limit this milestone should design against? Not stated in
  `DEVELOPMENT_PLAN.md` or found elsewhere in the repo — needed to make acceptance criterion 2
  testable.
- Should "session-level" caching (item 3) mean in-memory only (cleared on page reload) or should it
  persist via `localStorage` similar to saved study sets? `DEVELOPMENT_PLAN.md` says
  "session-level," which this draft reads as in-memory-only, but that's an interpretation, not a
  quote.
- For item 4 (fallback source evaluation): should this spec name candidate providers to evaluate,
  or is selecting candidates part of the Architect's job in `02-ARCHITECTURE.md`? This draft
  assumes the latter, since `DEVELOPMENT_PLAN.md` does not name any candidates.
- Confirm the "out of scope" list above matches CEO intent — it is inferred from milestone
  sequencing, not stated directly.

## Exit criteria (from `DEVELOPMENT_PLAN.md`, verbatim)

"Any arbitrary English passage (not just the sample) produces either a real translation or an
honest, clearly-marked failure state — never a misleading canned string."

---

**Next step:** CEO review. Approve, request revisions, or answer the open questions above so this
draft can be finalized before Architecture Design begins.
