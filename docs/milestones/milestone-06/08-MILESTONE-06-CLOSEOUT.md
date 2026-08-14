# Milestone 06 — Closeout: Example Sentence Quality Upgrade

Status: **COMPLETE**. No BLOCKER or HIGH findings remain open. Declared under the CEO's fast-track
governance decision (product-progress mode: only BLOCKER/HIGH findings block progress; MEDIUM/LOW go
to backlog unless they threaten data, security, core functionality, or student learning correctness).

Branch: `feature/example-sentence-quality`, final commit `c9aee4c`. Not pushed, not merged — awaiting
explicit CEO authorization per `.ai-company/GIT_RULES.md`.

## What this milestone fixed

**Original problem** (CEO's initial observation): vocabulary example sentences frequently used practice
words as though they were nouns regardless of real part of speech, and the Example tab specifically
showed generic, POS-blind template sentences for 97.7% of curated vocabulary due to a stale routing bug.

**Root causes found and fixed, in sequence:**

1. **Example tab routing bug** — `examples()` read from a stale 26-word `exampleBank` instead of the
   correct, already-curated `v.data[3]`/`v.data[4]` sitting on every vocab item. Retired `exampleBank`;
   `examples()` now reads curated data directly.
2. **POS-blind fallback templates** — `fallbackExamplePool` always framed genuinely-uncurated words as
   a discussed/quoted term regardless of real POS. Replaced with POS-aware frame families, then further
   refined (safer verb frames, manner-vs-connective adverb split) after independent QA found the initial
   frames still produced awkward results for many words.
3. **Silent noun-default on unknown POS** — `guessPOS()` defaulted any unrecognized word to `"noun"`,
   reproducing the original bug through a different path. Replaced with a tiered POS resolver
   (lexical-evidence reuse → suffix inference → explicit `"unknown"`, never silently `"noun"`).
4. **Fallback quality ceiling (~65% GOOD/EXCELLENT)** — independent QA found generic frames, even with
   correct POS, still frequently produced awkward or wrong collocations for arbitrary words. Added
   synonym-sourced curated-example borrowing (safe, antonym borrowing explicitly rejected after testing
   showed it can invert meaning) and a modest, evidence-driven curated-lexicon expansion (30 words,
   chosen from real fallback-frequency data, not arbitrarily).
5. **Confidence gate (Milestone 6C)** — rather than keep tuning generic frames, added a strict rule:
   only display a fallback example when it is HIGH confidence (curated, safe morphology reuse, or
   synonym-borrowed); everything else — including cases where the POS itself was well-evidenced but no
   safe example existed — is withheld with an honest, non-technical, dictionary-pointing message. A
   product-principle fast-track decision was applied mid-implementation: a first design attempted to
   also display "MEDIUM confidence" (lexical-evidence-backed but generic-frame) examples per the
   original spec's literal wording, but live testing immediately found a counter-example producing an
   awkward result; MEDIUM was folded into withheld, prioritizing "no unnatural example" over "more
   coverage."
6. **Ungated morphology-regeneration path (this session's fix)** — independent QA, testing beyond the
   minimum required word list, found that a *separate* code path (unsafe `-ed`/`-ing` morphology
   regeneration, e.g. "witnessed"/"functioned") was never brought under the 6C gate and could still
   display broken English while mislabeling the result `curated:true`. Fixed by routing that path
   through the same confidence-gate primitives (synonym-borrow first, then withhold) rather than
   building a second gate.

## Final verified state

- **Real-world coverage:** typical-length SAT-register passages remain ~96-100% trusted (curated, safe
  morphology, or synonym-borrowed) across every register independently tested (science, history,
  literature, social science, humanities, psychology, economics, art history, biology, law, sociology,
  linguistics). On a realistic mixed-difficulty 7-passage set, 90.0% of all selected vocabulary receives
  a trusted example after the final fix — an accurate figure, since the confidence gate now correctly
  excludes previously-mislabeled unsafe-morphology cases from that count.
- **Fallback quality among displayed examples:** 100% GOOD-or-EXCELLENT across every independent test
  this milestone (6 CEO-named problem words, two independent 20-30-word adversarial samples, the
  morphology-specific adversarial test) — zero AWKWARD/BAD examples shown to a student, by construction
  of the confidence gate rather than by chance.
- **Words withheld:** shown an honest, non-technical "check the dictionary links" message instead of a
  guess. Concentrated almost entirely in short passages (too few candidate words) and unusually
  dense/advanced passages — not spread evenly across normal content.
- **No regressions** in curated routing, morphology safety, synonym borrowing, antonym rejection,
  Vocabulary Card, Word Book (including a true reload+login+persistence check), Review state,
  pronunciation, SAT vocab-in-context, SAT Retry, Evidence Sentence, Weak-area reinforcement, Gemini
  Summary/vocab-context, translation, PDF/OCR/HEIC, IndexedDB (still exactly 6 stores), Gold Master
  checksum, or XSS protections — independently re-verified at every phase, most recently after this
  session's fix.

## Backlog (MEDIUM/LOW — not blocking, per fast-track governance)

**BL-1 (LOW) — `suffixPOS()`'s `-er`/`-eer` noun-suffix pattern mislabels some genuine verbs.**
`squander`, `pilfer`, `commandeer` (and likely other `-er`/`-eer`-ending verbs) display POS badge
`"noun"` instead of `"verb"`. Does not affect example safety — words in this state always have their
example correctly withheld under the confidence gate, so no unnatural or misleading example ever
reaches a student. Purely a POS-badge cosmetic inaccuracy. Not fixed this milestone because, unlike the
bounded `ATE_ADJECTIVES` exception list (~18 words, all unambiguous), the `-er`-ending-verb category is
much larger and less bounded (consider, deliver, discover, foster, wander, hover, suffer, differ,
offer, whisper, gather, scatter, and many more), so a safe fix requires meaningfully more research to
build a reliable exception list (or a different disambiguation approach) without introducing new
misclassifications. Candidate for a future, separately-scoped Dev Task if prioritized.

**BL-2 (deferred by explicit CEO decision, not a defect) — Optional Gemini-assisted fallback for the
residual withheld long tail.** Evaluated in `05-QUALITY-ARCHITECTURE.md` §7 and revisited in
`06-IMPLEMENTATION-LOG.md` §17; not required to meet the educational-correctness bar (already met
locally by the confidence gate), but would close some of the remaining withheld long tail if the CEO
wants coverage pushed further. Explicitly out of scope for Milestone 06.

## Documents produced this milestone

`01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`, `04-QA-REPORT.md` (original
Milestone 6), `05-QUALITY-ARCHITECTURE.md` (Milestone 6B architecture), `06-IMPLEMENTATION-LOG.md`
(6B + 6C implementation), `07-QA-REPORT.md` (6C QA + this session's hot-fix addendum), this closeout
document.

## Commits on this branch (chronological)

`71edcf8` PM-Spec + Architecture → `ce9ba59`..`60de44f` Milestone 6 Dev Tasks 1-5 + QA →
`735fa16` Milestone 6 QA report → `9efdcf0` Milestone 6 hot-fix (F-1/F-2) →
`ec38ca9` Milestone 6B Architecture → `db7bb83`/`ccf60cd`/`1f643b4` Milestone 6B Dev Tasks →
`99eece5` Milestone 6B docs → `19f447a` Milestone 6C confidence gate →
`c2f2949` Milestone 6C QA report → `c9aee4c` morphology-gate hot-fix (this session).

## Next step

STOP before push/merge, per instruction. Awaiting explicit CEO authorization to push and open a PR
against `main`.
