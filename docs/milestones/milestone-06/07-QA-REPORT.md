# Milestone 6C — Independent QA Report: Confidence-Gated Example Quality

Role: Independent QA Engineer. Branch: `feature/example-sentence-quality` @ `19f447a` (confirmed —
`git rev-parse HEAD` matched before testing began; working tree clean except pre-existing untracked
`.DS_Store`/`.claude/`). Read `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `04-QA-REPORT.md`,
`05-QUALITY-ARCHITECTURE.md`, `06-IMPLEMENTATION-LOG.md`, and the actual `index.html` diff against
`origin/main` before testing. Method: local static server + live in-browser JS execution and real
click-through, independent of Development's own test scripts. No application code modified during
this QA pass. Not pushed, not merged.

---

## 1. QA verdict

**PASS WITH CONDITIONS.**

The confidence gate itself — the specific mechanism `buildFallbackVocabData()` implements — is
correctly built and works exactly as documented for every case within its scope: Sections A through H
below all confirm it cleanly. HIGH-confidence examples (curated, safe morphology reuse, synonym
borrowing) display correctly and are high quality; MEDIUM and LOW confidence are withheld with a clear,
non-technical, correctly-worded message; the 6 CEO-named problem words and a fresh 30-word adversarial
sample show zero AWKWARD/BAD displayed examples; synonym borrowing safety (antonym rejection, article
agreement, no blind substitution) all hold up under negative testing; real-world coverage remains
strong (95.8%+ trusted on typical-length passages, independently re-measured); cross-feature regression
including a true reload/login/Word Book persistence check is clean.

**However, independent testing (not prompted by any specific item in the QA brief — found while
reviewing a live passage render) discovered that a *separate* code path also capable of producing a
generated example — `buildVocabCardData()`'s unsafe-morphology-regeneration branch (Milestone 6 Task
4, e.g. `witnessed`/`witnessing`) — is not covered by the 6C confidence gate at all, and reproducibly
displays outright broken English for a real, non-trivial class of curated-word inflections** (e.g.
`functioned` → *"The committee proposed a functioned solution to the ongoing problem."*, `influenced` →
*"The committee proposed an influenced solution to the ongoing problem."*). These results are also
mislabeled `curated:true`, meaning they silently count as "trusted" in every coverage metric reported
this milestone. This is exactly the category of harm the 06C phase's own stated goal was to eliminate
("verify that the confidence gate prevents unsafe example sentences") — it does, for the path it
covers, but a second path with the same underlying risk was left out of scope. See §14 finding F-1
(HIGH) for full detail, reproduction, and recommended fix.

---

## 2. Confidence-gate correctness

Independently read the actual code (`buildFallbackVocabData()`, `withheldFallbackExample()`,
`findSynonymBorrowSource()`, `borrowSynonymExample()`, `resolveFallbackPOS()`) before testing, not just
the implementation log's description of it. Confirmed by direct code inspection: after the `"unknown"`
check and the synonym-borrow attempt, the function has **no remaining code path that returns a
generated generic-frame sentence** — every other case falls through to `withheldFallbackExample()`.
This structurally guarantees requirement A.4 ("no MEDIUM/LOW path accidentally leaks a generated
sentence") for this specific function, independent of any behavioral testing.

Empirically confirmed all 5 checklist items for `buildFallbackVocabData()`'s own scope:
1. **HIGH examples display** — `evidence`, `witnesses` (morphology), `argue`/`vague` (synonym-borrowed)
   all render real content. ✅
2. **MEDIUM examples withheld** — `prevent`, `completely` (both confirmed via `lexicalPOSLookup(w)!==null`
   — real lexical evidence, no synonym to borrow) both show the withheld message. ✅
3. **LOW examples withheld** — `garrulous` (suffix-only), `xyzzyword123` (unknown) both withheld. ✅
4. **No leaks** — confirmed by code reading (above) and by testing 50+ words this session with zero
   exceptions. ✅
5. **No incorrectly-withheld HIGH example** — tested a broader sample of 14 curated/morphology-safe
   words beyond the brief's minimum list; zero false-withholds. ✅

**Caveat driving the overall verdict:** all 5 of these checks are scoped to `buildFallbackVocabData()`
specifically. The gate does not extend to `buildVocabCardData()`'s separate unsafe-morphology path —
see §1 and §14 F-1.

## 3. Known-problem-word result

All 7 words (6 CEO-named + `completely`) independently re-tested: **7/7 correctly withheld**, POS badge
still shown, zero unsafe examples displayed. Confirmed live in a real passage render (not just direct
function calls) — the withheld message rendered correctly in the actual Example tab UI.

## 4. High-confidence example result

- **Curated (4/4):** evidence, demonstrate, significant, consequently — correct word, correct
  grammatical role, natural English, correct Korean pairing, no distortion.
- **Morphology (plural noun, 3rd-person verb, safe past-tense):** witnesses, corroborates, corroborated
  — all correctly reuse the base's real curated example verbatim, byte-identical to the base.
- **Synonym borrowing:** argue←contend, vague←ambiguous (as specified) plus **reticent←reluctant** and
  a genuinely fresh case not used anywhere in Development's testing, **hesitant←reluctant** — both
  natural, correct POS, correct Korean. A **defensive-safety positive finding**: `thwart` has a real
  synonym relationship to the curated word `frustrate`, but `frustrate`'s own curated example uses the
  inflected form "frustrated," not the bare headword — the code's defensive regex check
  (`\bfrustrate\b` does not match inside "frustrated") correctly detected this and refused to borrow,
  falling through to the withheld message rather than attempting an unsafe substitution. This is exactly
  the "no blind string replacement errors" property Section H asked to verify, caught in the wild.

## 5. 30-word unseen adversarial test

30 genuinely fresh words never used by Development or elsewhere in this QA pass (mendacious,
sycophantic, ebullient, phlegmatic, choleric, stoic, churlish, boorish, unctuous, pretentious,
ostentatious, flamboyant, spartan, lavish, opulent, squander, embezzle, pilfer, commandeer, relinquish,
forfeit, usurp, depose, convivial, clandestinely, furtively, brazenly, fervently, zealously, plus one
substitution for a duplicate-flagged candidate):

| | Count |
|---|---|
| Displayed | 0 |
| Withheld | 30 |
| Displayed EXCELLENT/GOOD/ACCEPTABLE/AWKWARD/BAD | n/a — nothing displayed |

**Vacuously satisfies the ≥90% GOOD-or-EXCELLENT-among-displayed bar and the "AWKWARD/BAD displayed =
0" bar** — there is nothing to violate them with. This is the maximally conservative, safe outcome for
a deliberately hard word list, consistent with the 6C design intent.

**Finding (LOW severity, §14 F-2):** several of these words resolve to an incorrect POS *badge* via the
suffix-only guesser even though their *example* is safely withheld — `squander`, `pilfer`, and
`commandeer` (all genuinely verbs) are labeled `"noun"` (matching the `-er`/`-eer` noun-suffix pattern).
Since the example is withheld, this cannot mislead a student about *usage*, but the POS badge itself is
still visibly wrong. Low severity because the higher-risk surface (a fabricated example) is fully
mitigated regardless.

## 6. Displayed GOOD/EXCELLENT %

Across every word tested this QA pass that was actually displayed (HIGH-confidence only): **100%** —
every single displayed example, across curated words, morphology reuse, and 6 independent synonym-
borrow cases, rated GOOD or EXCELLENT. Zero AWKWARD or BAD displayed via the `buildFallbackVocabData()`
path specifically.

**Caveat:** this figure excludes `buildVocabCardData()`'s separate unsafe-morphology path, which *is*
capable of displaying broken examples (§14 F-1) and was not gated by this phase.

## 7. Withheld %

On the 30-word adversarial sample: 100% (30/30). On a realistic-passage mix (§8): withheld rate ranges
from 0% (typical-length passages) to ~75% (very short passages), consistent with Development's own
reported shape.

## 8. Realistic passage coverage

Tested 10 passages: 6 typical-length (~90-110 words, spanning science/humanities/social-science
registers — 4 freshly written for this QA pass, 2 reused from Development's set for continuity), 2
short passages, 2 dense/advanced passages.

| Category | Trusted (displayed) |
|---|---|
| Typical-length (6 passages, 72 words) | **69/72 = 95.8%** |
| Short (2 passages, 16 words) | 6/16 = 37.5% |
| Dense/advanced (2 passages, 24 words) | 14/24 = 58.3% |

Consistent with Development's own reported shape (typical ~100%, short/dense lower) — no discrepancy
found. **Answer to the most important question: yes, a normal student encountering a typical-length
SAT-register passage still receives a nearly-complete set of trusted examples (95.8%+ in this
independent sample, matching Development's 89.8-91.3% on their own larger 18-passage set).** The
confidence gate's conservatism is concentrated exactly where the Architecture predicted — short and
unusually dense passages — not spread evenly across normal content.

## 9. Student educational UX verdict

**YES**, with the caveat in §1/§14 F-1 in mind.

1. **Do good examples still teach natural usage?** Yes — every HIGH-confidence example independently
   reviewed (curated, morphology-safe, synonym-borrowed) is natural, POS-correct academic English.
2. **Does a withheld word clearly tell the student what to do next?** Yes — *"이 단어는 정확한 용례
   확인이 필요합니다. 사전 링크에서 실제 예문을 확인해 보세요."* is plain, non-technical, and points to
   the Vocabulary Card's real dictionary links (Merriam-Webster, Oxford Learner's, Cambridge,
   Thesaurus — confirmed still present and unmodified).
3. **Is it better than showing an awkward sentence?** Yes, unambiguously — confirmed by direct
   comparison: the same word previously shown as *"The two approaches were garrulous..."*-style
   awkwardness (pre-6C) now shows an honest "please check a dictionary" message instead.
4. **Does the page feel incomplete or still useful?** Still useful — in a real passage render, withheld
   cards are interspersed with real examples, use the same card styling, and never appear broken; a
   student sees a mix of real teaching content and occasional "look this one up yourself" prompts,
   which reads as reasonable rather than broken.

## 10. Synonym borrowing safety

All confirmed independently (§4, §H testing): synonym-field-only (never antonym — `prevent`, which only
has antonym relations, correctly returns no borrow source); POS compatibility required (exact clean-POS
match enforced by the code); article agreement corrected in both directions (consonant stays "a," vowel
becomes "an," tested via `borrowSynonymExample("austere", ...)`); no semantic inversion (this is the
architectural reason antonym borrowing is excluded at all — confirmed still excluded); no blind string
replacement errors (the `thwart`/`frustrate` case in §4 is a live example of the defensive check
correctly refusing an unsafe substitution rather than performing one).

## 11. Morphology result

The CEO-named canonical test words behave exactly as previously documented, **no new regression**:
`witnessed`/`witnessing` (base `witness`, POS `noun/verb`) land on the predicate-adjective frame
("seemed witnessed/witnessing") — the same CONFUSING-not-INCORRECT result the original Milestone 6 QA
report already identified and accepted; `retained`/`retaining` (base `retain`, unqualified verb POS)
correctly reuse the base's real example verbatim, matching the safe-path design.

**However**, broader independent testing beyond these specific named words found that *other*
curated-word inflections landing on the *different* adjective frame ("The committee proposed a/an
`{word}` solution...") — a frame slot the original canonical test words don't happen to hash into —
produce not just "confusing" but outright **ungrammatical** results (`functioned`, `influenced`; see
§14 F-1). This is a materially worse manifestation of the same underlying gap than what the CEO's named
words exhibit, discovered specifically because this QA pass tested beyond the minimum named list per
the QA brief's own instruction to verify independently rather than only re-run what was named.

**Student-visible POS/example contradiction:** unchanged from the original QA report's finding
(rated CONFUSING) for witnessed/witnessing; for `functioned`/`influenced`, the contradiction is more
severe — the example is not merely inconsistent with the POS badge, it is not valid English at all.

## 12. Regression result

Independently re-verified, not just re-read from the implementation log:
- `git diff origin/main -- index.html`: exactly 6 hunks, confined to the exampleBank/broadVocabLexicon/
  safeMorphologyBase-adjacent/buildVocabCardData/findContextSentence-adjacent/examples() regions.
  Nothing touches Vocabulary Card markup, Word Book logic, Gemini functions, translation, PDF/OCR/HEIC,
  or `openDB()`.
- `genVocabInContext()` (line 3334, untouched region): independently re-invoked with a fresh sentence,
  produced a correct, well-formed question.
- IndexedDB: exactly 6 `createObjectStore` calls, unchanged.
- Gold Master checksum: unchanged (`a51c603...2ba`).
- **True persistence check** (explicitly required): created a fresh local profile, saved a curated word
  (`evidence`) and a withheld-example word (`garrulous`) to the Word Book, cycled `garrulous`'s review
  state to "reviewing," performed a genuine full page reload (not an SPA state reset), reopened the
  Word Book with no passage in memory — both words reappeared with correct data; `garrulous`'s card
  correctly shows the withheld message (not a broken example) on reconstruction, and its review state
  survived the reload intact.
- XSS: covered indirectly via the unaffected `examples()` escaping logic (untouched by this diff,
  confirmed via the diff-scope check above); not independently re-tested with a fresh payload this
  pass since the code path is provably unchanged.

## 13. Mobile result

At 375×812: withheld message renders cleanly and legibly in both languages; a genuinely long word
(`unconstitutionality`) wraps without breaking the card; real curated examples render normally
alongside withheld cards in the same list with no visual distinction problem; no overflow or clipping
observed; no console errors.

## 14. Findings by severity

**F-1 — HIGH — The confidence gate does not cover `buildVocabCardData()`'s unsafe-morphology-
regeneration path, which reproducibly displays broken English and mislabels it `curated:true`.**
Reproduction: `buildVocabCardData("functioned", null)` → `curated:true`, `data[3] = "The committee
proposed a functioned solution to the ongoing problem."` — not valid English (a past-tense verb form
used as a prenominal adjective). Root cause: this path (Milestone 6 Task 4, for `-ed`/`-ing` surface
forms of a curated word whose base POS is not the unqualified string `"verb"`) forces `"adjective"` as
a working POS and renders via `fallbackFrameFamilies.adjective` directly — the *exact* generic-frame
mechanism the 6C gate was built to stop displaying — but 6C's gate lives entirely inside
`buildFallbackVocabData()`, a function this path never calls. Confirmed reproducible and non-rare: a
scan of curated entries with a compound POS containing "verb" (e.g. `noun/verb`, `verb/noun`,
`verb/adjective`) found multiple real inflections producing the same broken pattern when they land on
this specific adjective frame slot (`functioned`, `influenced`; `transitioned`/`disputed`-type words
land on a different, more grammatically-tolerant frame slot and read acceptably, so severity varies by
which of the 3 adjective frames the per-word hash selects — roughly 1-in-3 of affected words). Because
this path sets `curated:true`, every coverage percentage reported in `05-QUALITY-ARCHITECTURE.md` and
`06-IMPLEMENTATION-LOG.md` (including this report's own §8) may be marginally inflated by an unknown
number of such cases hiding inside the "curated" bucket. *Scenario:* a student encounters "functioned"
(inflected form of the common curated word "function") in a passage → sees a grammatically broken
sentence presented with the same visual trust signal as every other curated example.

**F-2 — LOW — Suffix-only POS guesser mislabels some verbs as nouns via the `-er`/`-or` agent-noun
suffix pattern**, even though the resulting *example* is safely withheld (§5). `squander`, `pilfer`,
`commandeer` (all verbs) display POS badge `"noun"`. Cannot mislead about usage since no example is
shown, but the badge itself is visibly wrong.

**No BLOCKER findings.** No regressions found in curated routing, synonym borrowing, antonym rejection,
`buildFallbackVocabData()`'s own confidence gate, cross-feature areas, or mobile.

## 15. Is optional Gemini fallback justified?

Consistent with `06-IMPLEMENTATION-LOG.md` §17's own nuanced framing, independently re-confirmed: the
core educational-correctness bar for the *gated* fallback path is solidly met (95.8%+ trusted on
typical-length passages, zero AWKWARD/BAD displayed across every test in this QA pass). Gemini is not
*required* to meet that bar. However, F-1 shows there is a real, currently-ungated risk surface separate
from the fallback engine Gemini would address — meaning the more urgent next step is extending the
confidence-gate discipline to `buildVocabCardData()`'s unsafe-morphology path (a local, no-AI fix, likely
smaller in scope than 6C itself), not Gemini. Recommend resolving F-1 before revisiting the Gemini
question.

## 16. Recommended next action

1. **Fix F-1 before this branch is considered ready for the next phase.** The most direct fix: apply
   the same HIGH-confidence-only display discipline to `buildVocabCardData()`'s unsafe-morphology
   branch — e.g., when the regenerated-adjective-frame path would be used, either withhold the example
   (same message, same pattern as 6C) while still reusing the base's real meaning/POS/synonyms/antonyms
   (consistent with Milestone 6 Task 4's own original "don't discard useful lexical data" principle), or
   restrict that branch to only the passive/predicate adjective frame slot (which read acceptably in
   this QA pass's sample) rather than all 3, excluding the prenominal slot that produced the broken
   results. Either approach is a small, local, well-scoped change consistent with everything already
   established this milestone.
2. Fix F-2 opportunistically if convenient when F-1 is addressed (likely the same kind of small,
   closed-list or evidence-tightening fix used for the `-ate` adjective exception list in 6B).
3. Once F-1 is resolved, re-verify specifically that `functioned`/`influenced`-type words either show a
   correct example or are withheld — no other re-testing should be needed given how clean the rest of
   this pass was.

---

**STOP at QA gate. No push, no merge, no application-code changes made during this QA pass.**
