# Milestone 6B — Implementation Log: Collocation-Aware Example Quality

Role: Senior Developer. Branch: `feature/example-sentence-quality`. Base: `9efdcf0` (stabilized
intermediate baseline). Executes the 5 Developer Tasks approved in `05-QUALITY-ARCHITECTURE.md` §11,
under CEO decisions: (1) approve synonym-only borrowing, (2) reject antonym borrowing, (3) targeted
curated expansion as primary lever, (4) defer Gemini-assisted fallback, (5) preserve ≥90% target,
(6) educational correctness over 100% coverage.

## 1. Task 1 — Synonym example borrowing

**Rule implemented:** a genuinely-uncurated word may borrow a curated entry's real example sentence
(word substituted) only when: the curated entry's own POS is an exact, clean match (no "noun/verb"
compounds) to the target word's resolved POS; the target word is listed in that entry's **synonym**
field (`data[5]`) specifically; and the entry's own headword is verified to literally appear in its
own example before any substitution is attempted (defensive check against the authoring convention,
not assumed). Article agreement (a/an) is corrected on the substituted text since the original author
wrote the article for the *source* word, which may not match the target word's vowel status.
**Antonym fields (`data[6]`) are never searched** — Architecture-phase testing found antonym
substitution can invert a sentence's meaning (`prevent` substituted into an `induce` example reversed
the causal direction), which the CEO's decision explicitly confirmed rejecting.

**Verified:**
- `contend` → `argue`: *"Some economists argue that the policy will slow growth."* (was AWKWARD via
  the generic frame)
- `ambiguous` → `vague`: *"The vague wording caused two groups to interpret the rule differently."*
- `justify` → `vindicate` (discovered during Task 5 testing): *"The committee could not vindicate the
  sudden change in policy."*
- Negative cases correctly rejected and fell through to the existing generic frame: `prevent`,
  `invalidate`, `insular` (antonym-only relations), `permeate`, `galvanize` (zero relations).
- Synthetic unit-style tests confirmed article correction in both directions (consonant stays "a",
  vowel becomes "an") and capitalization preservation.

## 2. Task 2 — Real fallback frequency analysis

Reused the Architecture's coverage-study method (real `analyze()` selection logic, not an
approximation) across a broader sample: the original 7 Architecture-phase passages plus 11 new ones
(psychology, economics, art history, philosophy, biology, astronomy, law, sociology, linguistics, and
3 short passages), 18 total.

**Finding, consistent with the Architecture doc:** typical ~90-110-word passages across every register
tested reached 100% curated coverage — 12 of 18 passages had zero fallback words. Fallback only
appeared meaningfully in short passages (too few length-≥7 candidates) and a deliberately dense/
advanced-register passage.

**Ranked candidate list** (frequency + academic usefulness + SAT relevance + absence from curated
lexicons, per the CEO's explicit criteria — mundane words that only appeared via the short-passage
length-padding mechanism, e.g. "committee's," "chapters," "professor's," were explicitly excluded per
"do NOT add random vocabulary just to increase dictionary size"):

- **Highest priority** (resolved to `"unknown"` — zero local data of any kind): circumvent, harsh,
  redundant, cursory.
- **High priority** (AWKWARD via the generic frame, zero synonym/antonym relation to borrow from):
  permeate.
- **Observed directly in the realistic-passage coverage study**: circumspection, argumentation,
  evolutionary, subjective, skepticism, reckless, tenderness, disagreement.
- **Legitimate SAT/academic-register words tested elsewhere this milestone** that would benefit from
  guaranteed curated quality rather than a generic-frame guess: meticulously, cautiously,
  systematically, volatility, proliferation, superficial, tacitly, conspicuously, erroneously,
  judiciously, perpetuate, invalidate, insular, resilience, discrepancy, delineate, dissipate.

30 words selected as the "smallest useful batch" (§3).

## 3. Task 3 — Targeted curated expansion

Added all 30 ranked words to `broadVocabLexicon` at the existing 7-field format and authoring quality
bar (POS, EN/KO meaning, natural EN example demonstrating real usage, matching KO example, synonyms,
antonyms) — no placeholder or weak content. Verified zero pre-existing key collisions before insertion.

**Caught during authoring, not shipped:** a first draft accidentally included an incomplete placeholder
entry (`caution`, with `"—"` stand-ins for the example/Korean-example/synonym/antonym fields) — removed
before commit. It was never part of the intended 30-word list, and `caution` already exists as a
complete, real entry elsewhere in `dictionary`.

All 30 verified curated (`curated:true`, complete 7-field data) with zero console errors; 6 spot-checked
end-to-end through `examples()` render natural, POS-correct, properly-escaped sentences.

## 4. Task 4 — Safe residual fallback confirmed intact

Independently re-verified, using fresh words not previously tested: unknown POS never silently becomes
noun (`frobnicate2`, `xyzzyword` → `"unknown"`; a genuine adjective `plutonic` still correctly resolves
via suffix); the manner/connective adverb split still works (`stealthily` → `adverbManner`;
`notwithstanding` → `"unknown"`, correctly, since it isn't in the connective list and doesn't end in
`-ly`); verb complement safety intact (`codify` still uses the safe pronoun-object frame, no "on"); XSS
escaping unaffected.

## 5. Task 5 — Quality re-test

### 5.1 Incidental fix found during testing

Testing 20 fresh unseen words surfaced a genuine, new POS-role defect: `suffixPOS()`'s `-ate` verb rule
misclassified adjectives like `obstinate` as verbs (English distinguishes `-ate` verbs from `-ate`
adjectives by stress/pronunciation — invisible to spelling). Fixed with `ATE_ADJECTIVES`, a small closed
exception list (obstinate, desolate, considerate, fortunate, passionate, adequate, corporate, delicate,
accurate, ornate, sedate, ultimate, intricate, affectionate, compassionate, proportionate,
disproportionate, inanimate) — deliberately excluding genuinely ambiguous `-ate` words (moderate,
separate, elaborate, graduate) rather than guessing wrong for those. Same bounded-list pattern already
established (and CEO-approved) for `CONNECTIVE_ADVERBS`. Regression-checked that real `-ate` verbs
(perpetuate, exacerbate, codify) are unaffected.

### 5.2 A — Previously-failing QA words (12 original words)

| Word | Before (`9efdcf0`) | After |
|---|---|---|
| circumvent | ACCEPTABLE (unknown) | **EXCELLENT (curated)** |
| prevent | GOOD (generic frame) | GOOD (unchanged — antonym-only relation, correctly not borrowed) |
| argue | AWKWARD | **EXCELLENT (synonym-borrowed from `contend`)** |
| vague | GOOD | **EXCELLENT (synonym-borrowed from `ambiguous`)** |
| harsh | ACCEPTABLE (unknown) | **EXCELLENT (curated)** |
| perpetuate | GOOD | **EXCELLENT (curated)** |
| exacerbate | EXCELLENT (generic frame) | EXCELLENT (unchanged) |
| meticulously | EXCELLENT | **EXCELLENT (curated)** |
| gradually | GOOD | GOOD (unchanged — not in the 30-word batch) |
| cautiously | GOOD | **EXCELLENT (curated)** |
| deliberately | EXCELLENT (already curated) | EXCELLENT (unchanged) |
| systematically | EXCELLENT | **EXCELLENT (curated)** |

**12/12 = 100% GOOD-or-EXCELLENT.** 8/12 (67%) now avoid the fallback engine entirely via curation
(up from 1/12 before). The 4 that still reach fallback (prevent, argue, vague, gradually) all rate
GOOD or EXCELLENT.

### 5.3 B — Synonym-borrowing cases

Covered in §1 — `argue`/`vague` (planned test cases) and `vindicate` (discovered organically during
this section's fresh-word testing) all confirmed EXCELLENT.

### 5.4 C — 20 fresh, independently-chosen unseen words

Deliberately selected words not used anywhere else in this milestone, spanning all 4 POS, at a
comparably advanced register to stress-test the system further:

| Word | POS resolved | Rating |
|---|---|---|
| ameliorate | verb (generic) | GOOD |
| obfuscate | verb (generic) | GOOD |
| truculent | unknown | ACCEPTABLE |
| assiduous | adjective (generic) | AWKWARD — person-trait adjective forced onto "the findings" |
| garrulous | adjective (generic) | BAD — person-trait adjective ("talkative") forced onto "a ... solution" |
| recalcitrant | unknown | ACCEPTABLE |
| perfunctory | adjective (generic) | EXCELLENT |
| sanguine | unknown | ACCEPTABLE |
| vindicate | verb (**synonym-borrowed**) | EXCELLENT |
| denigrate | verb (generic) | GOOD |
| equivocate | verb (generic) | BAD — intransitive verb given a direct object ("equivocate it") |
| pernicious | adjective (generic) | ACCEPTABLE |
| laconic | adjective (generic) | AWKWARD — same person-trait-adjective pattern as garrulous |
| obstinate | adjective (generic, **post-fix**) | AWKWARD — same pattern, no longer a POS defect after §5.1 |
| precipitous | adjective (generic) | AWKWARD |
| ubiquitously | adverb (generic) | BAD — scope/existence adverb forced onto a completion-verb frame |
| incontrovertibly | adverb (generic) | AWKWARD |
| surreptitiously | adverb (**synonym-borrowed**) | EXCELLENT |
| begrudgingly | adverb (generic) | GOOD |
| staunchly | adverb (generic) | AWKWARD |

**Distribution: 3 EXCELLENT, 4 GOOD, 4 ACCEPTABLE (3 honest "unknown" + 1 generic-but-safe), 6 AWKWARD,
3 BAD.** GOOD-or-better on words the system attempted a guess for (excluding the 3 honest "unknown"
results): 7/17 = 41%. Including "unknown" as its own honest, safe (non-GOOD) category: 7/20 = 35%.

**This independent sample is meaningfully harder than every previous test batch this milestone**, and
surfaces two **new, previously-undocumented failure categories** beyond what F-1/F-2/the Architecture
doc had already named:

1. **Person/behavior-describing adjectives forced into thing-describing frames.** `garrulous`
   (talkative), `laconic` (terse), `obstinate` (stubborn), `assiduous` (diligent) all describe people
   or their manner, not things — the adjective family's frames ("a `{word}` solution," "the findings
   were `{word}`") were designed around thing-describing adjectives (meticulous, derivative, tenuous —
   all of which scored well in the original QA report) and don't fit this class at all.
2. **Scope/degree/certainty adverbs forced into action-manner frames.** `ubiquitously` (existing
   everywhere), `incontrovertibly` (undeniably), `staunchly` (firmly, usually paired with support/
   defend/oppose) don't naturally modify simple completion verbs like "completed" the way genuine
   manner adverbs (meticulously, cautiously) do.

A third result (`equivocate`) is not new — it is the **already-disclosed intransitive-verb limitation**
from `05-QUALITY-ARCHITECTURE.md` §5 materializing in a concrete test case, exactly as predicted there.

### 5.5 D — Multiple realistic passages: two metrics measured separately

Re-ran the same 18 passages from §2 (both the original 7 and the 11 added this milestone) with all of
Tasks 1, 3, and the §5.1 fix applied, and compared against the pre-Task-3 numbers already captured in
this log's own §2 testing.

**Metric 1 — Real-world coverage** (% of selected vocab avoiding fallback entirely):

| | Before Task 3 | After Tasks 1+3+5.1 |
|---|---|---|
| All 18 passages combined | 177/206 = 85.9% | **185/206 = 89.8%** |
| 12 typical-length passages only | 144/144 = 100% | 144/144 = 100% (unchanged — already saturated) |
| short | 3/8 = 37.5% | 4/8 = 50% |
| veryAdvanced | 7/12 = 58.3% | 10/12 = 83.3% |
| philosophy | 8/12 = 66.7% | 10/12 = 83.3% |
| astronomy | 9/12 = 75.0% | 10/12 = 83.3% |
| shortLit | 2/6 = 33.3% | 3/6 = 50.0% |

Typical-length passages were already fully curated before this milestone's 06B work and remain so; the
real, measurable gain is concentrated exactly where the Architecture doc predicted fallback would
matter — short and unusually dense/advanced passages.

**Metric 2 — Fallback quality** (of the 21 words that still reached fallback across these 18 passages,
what fraction are GOOD-or-EXCELLENT): 4/21 = 19% strictly GOOD-or-better, but **16 of those 21 (76%)
are honest "unknown" results for words this milestone deliberately chose not to curate** because they
are mundane, low-value words that only appeared via the short-passage length-padding mechanism
(committee's, decision, praised, economy, professor's, preferences, mathematics, astronomers,
magnetized, satisfied, infections, occurred, reduced, revealed, chapters) — exactly the CEO's own
"do NOT add random vocabulary just to increase dictionary size" principle in action. Only 1 of the 21
(completely → "Costs completely increased...") rated AWKWARD; **zero rated BAD**, and 3 of the
generic-frame results were in fact synonym-borrowed and rated EXCELLENT (substantially, occasionally,
carefully).

## 6. Curated words added

30 (§3). Full list in §2's ranked candidate breakdown.

## 7. Before/after quality score summary

| Sample | Before | After |
|---|---|---|
| 23-word adversarial sample (`04-QA-REPORT.md` §17.6, unchanged, preserved) | 65% GOOD+ | *not re-run — see §5.4/§5.5 for fresh independent samples instead* |
| 12 previously-failing QA words | mixed (9/12 GOOD+ per prior reports) | **100% GOOD+** (12/12) |
| 20 fresh independent words (this session) | *n/a — first time tested* | 35% GOOD+ strict / 41% of attempted guesses |
| Real-world coverage, 18 realistic passages | 85.9% | **89.8%** |
| Real-world coverage, 12 typical-length passages | 100% | 100% |

## 8. Remaining long-tail gaps

Two new failure categories identified (§5.4): person/behavior-describing adjectives in the wrong frame
family, and scope/degree/certainty adverbs in the manner-adverb frame family. Both are, like the
already-disclosed intransitive-verb gap, structural consequences of using a small number of fixed
frames for open-ended vocabulary without per-word semantic classification — not implementation bugs.
Per Architecture §5-6, closing these further would require either more curated coverage (Option B,
continuing) or generative capability with real-world semantic knowledge (Option C).

## 9. Regression

Independently re-verified after every task: curated CEO words and Korean pairing unchanged; all 6
morphology-safety cases unchanged (including the untouched Task-4-from-Milestone-6 adjective
regeneration path); XSS escaping intact; `exampleBank` still retired (0 references); IndexedDB still
6 stores; Gold Master checksum unchanged (`a51c603...2ba`); `genVocabInContext()`, Vocabulary Card,
Word Book, translation, PDF/OCR/HEIC, Gemini functions all untouched by every diff this phase (changes
confined to the fallback-engine region and `broadVocabLexicon`'s new entries).

## 10. Status (Milestone 6B)

Tasks 1-5 complete. Commits: `db7bb83` (Task 1), `ccf60cd` (Task 3), `1f643b4` (Task 5's incidental
`-ate` fix). Stopping before Independent QA per instruction. No push, no merge.

---

# Milestone 6C — Safe Example Confidence Gate

CEO decision after reviewing 6B: real-world coverage improved (85.9%→89.8%), but fresh-unseen fallback
*quality* remained below the required standard (35% strict GOOD+ on a hard 20-word sample). Rather than
add more generic frame families or Gemini, gate the display of low-confidence generated examples
entirely. This section does **not** rewrite the 65%/35% results above — they stand as the historical
record of what unrestricted generic-frame display looked like before this gate existed.

## 11. Confidence-gate design

Three conceptual tiers, defined by the strength of local evidence behind an example (not an opaque
score):

- **HIGH** — exact curated example; safe morphology reuse (Milestone 6 Task 4); safe curated-synonym
  borrowing (6B Task 1). Displayed, unchanged by this gate.
- **MEDIUM** — POS backed by real curated lexical evidence (a synonym/antonym relationship to an actual
  curated entry) but no synonym example available to borrow — i.e. what used to be the generic-frame-
  family path. **Classified but not displayed** (see §12 for why).
- **LOW** — POS is fully unknown, or resolved only via a spelling-only suffix guess with no
  corroborating lexical evidence. Withheld.

**Final rule, implemented:** a fallback example is displayed only when it is HIGH confidence (curated,
safe morphology reuse, or synonym-borrowed). Every other case — MEDIUM and LOW alike — shows the
withheld message. The POS badge (`data[0]`) still shows the best-available guess in all cases (still
useful, still probably right) — only the *example sentence*, the part that can actively mislead a
student if wrong, is withheld.

**Withheld message** (both fields replaced, nothing else changed):
- EN: *"This word's exact usage needs to be confirmed. Check the dictionary links for a real example."*
- KO: *"이 단어는 정확한 용례 확인이 필요합니다. 사전 링크에서 실제 예문을 확인해 보세요."* (the CEO's
  exact specified wording)

No technical terms (fallback/confidence score/POS inference/generation failure) appear in either
message. The existing external dictionary links (`dictionaryLinks()`, Merriam-Webster/Oxford Learner's/
Cambridge/Thesaurus, rendered on the Vocabulary Card) are untouched and still render normally — the
withheld message's "check the dictionary links" line refers students to them.

## 12. Why MEDIUM is withheld, not displayed (a mid-course correction from the initial design)

The first implementation followed the CEO's literal tier description and displayed MEDIUM (lexical-
vote-backed generic frames). Live testing immediately surfaced a counter-example: `completely` resolves
via a genuine lexical vote (real curated antonym relationship) yet still produced *"Costs completely
increased throughout the following year"* — an awkward degree-adverb collocation the frame has no way
to detect. A second MEDIUM-tier case (`prevent`) scored fine, giving a 1-of-2 failure rate on the only
two genuine (non-curated, non-borrowed) MEDIUM examples found in testing — far short of "effectively
zero AWKWARD/BAD displayed." Given the explicit product principle stated first and most emphatically in
this phase's brief ("a missing example is acceptable; an unnatural or misleading example is NOT") takes
priority over the literal tier-display mechanism when the two are in tension, MEDIUM was folded into
the withheld path. `fallbackFrameFamilies` (noun/verb/adjective/adverbConnective/adverbManner) and
`isConnectiveAdverb()`/`CONNECTIVE_ADVERBS` are **not deleted** — `fallbackFrameFamilies.adjective` is
still a live, required dependency of Milestone 6 Task 4's unsafe-morphology-regeneration path (e.g.
`witnessed`/`witnessing`), which this phase does not touch. The rest of `fallbackFrameFamilies` and
`isConnectiveAdverb()` are now dormant (no remaining caller) but left in place rather than removed,
documented here rather than silently deleted, in case a future, more reliable per-word check makes
displaying MEDIUM safe again.

## 13. Known-problem-word results

All 6 CEO-named words, independently re-tested:

| Word | Result |
|---|---|
| garrulous | WITHHELD (was BAD) |
| laconic | WITHHELD (was AWKWARD) |
| ubiquitously | WITHHELD (was BAD) |
| staunchly | WITHHELD (was AWKWARD) |
| equivocate | WITHHELD (was BAD) |
| obstinate | WITHHELD (was AWKWARD, post-`-ate`-fix) |

**6/6 correctly withheld — zero displayed, zero risk of showing any of the specific problem sentences
that motivated this phase.**

## 14. 30-word (+20 revisited) unseen test

30 genuinely fresh words (never used anywhere else this milestone): nefarious, gregarious, fastidious,
querulous, vociferous, obsequious, pugnacious, loquacious, reticent, taciturn, abscond, castigate,
extricate, placate, proscribe, subjugate, vilify, acquiesce, capitulate, conflagration, exigency,
panacea, paragon, quandary, subterfuge, wistfully, inexorably, unabashedly, emphatically, dubiously —
plus the 20 from 6B Task 5's independent test re-run under the new gate, for a combined 50-word sample:

| | Count |
|---|---|
| DISPLAYED | 3 (reticent, perfunctory, vindicate — all synonym-borrowed or already curated at test time) |
| WITHHELD | 47 |
| Displayed: EXCELLENT | 3 |
| Displayed: GOOD | 0 |
| Displayed: ACCEPTABLE | 0 |
| Displayed: AWKWARD | 0 |
| Displayed: BAD | 0 |

**Displayed GOOD-or-EXCELLENT: 3/3 = 100%.** Zero AWKWARD/BAD displayed. This combined sample skews
deliberately hard (GRE-register vocabulary chosen specifically to stress-test the gate, harder than
typical SAT fallback words) — the very low 6% display rate here is a worst-case, not a typical-case,
number; §15 measures the student-actually-sees number.

## 15. Realistic passage trusted-example coverage

Same 18-passage suite used throughout Milestones 6B/6C, re-measured under the tightened gate (trusted =
curated + synonym-borrowed HIGH; everything else withheld):

| | Before 6B (baseline) | After 6B (Tasks 1+3) | After 6C (gate) |
|---|---|---|---|
| Trusted / total | 177/206 = 85.9% | 185/206 = 89.8% (curated only, medium not yet gated) | **188/206 = 91.3%** |
| Withheld | n/a (old "unknown" concept only) | 21/206 = 10.2% | 18/206 = 8.7% |
| 12 typical-length passages | 100% | 100% | **100% (unchanged)** |

The confidence gate **did not reduce** real-world trusted coverage relative to 6B — it is marginally
*higher* (91.3% vs 89.8%), because the words it additionally withholds (the MEDIUM-tier generic-frame
cases like `completely`) were a small fraction of the total (~2%) and some of what previously looked
like "coverage" in that bucket was actually the unreliable case this gate exists to catch. Every
displayed example in this 18-passage sample was independently spot-checked (§15's shortHistory/
shortScience/shortLit breakdown) and confirmed GOOD-or-EXCELLENT with zero AWKWARD/BAD.

**A normal student encounter remains fully useful**: typical ~90-110-word SAT-register passages across
every register tested are 100% trusted before and after this gate. The 8.7% residual withheld rate is
concentrated entirely in short and unusually dense/advanced passages, and even there the majority of
withheld words are low-value words (committee's, professor's, chapters) this milestone deliberately
chose not to curate — exactly the CEO's own "do not add random vocabulary" principle, now paired with
"and don't guess at an example for them either."

## 16. Remaining long-tail

The two adjective/adverb semantic-domain-mismatch categories and the intransitive-verb category
identified in 6B Task 5 (§5.4 above) no longer matter for display purposes — every word in those
categories that was tested this phase (garrulous, laconic, obstinate, ubiquitously, staunchly,
equivocate) is now correctly withheld rather than shown. The remaining long tail is now purely a
**coverage** question, not a **quality** question: genuinely rare/GRE-register words with zero curated
lexical relation will continue to show the honest withheld message rather than a guess, by design.

## 17. Is Gemini fallback now justified?

The evidence is more nuanced than a simple yes/no. Real-world trusted coverage is already 91.3%, and
100% on typical-length passages — the day-to-day student experience is already strong without AI. The
remaining 8.7% (concentrated in short/hard passages) is exactly the population Gemini would help most,
and given how rarely it would actually trigger (§15), the latency/cost/privacy tradeoffs documented in
`05-QUALITY-ARCHITECTURE.md` §7 would apply to a small minority of encounters, not a bottleneck on
every passage. This makes optional Gemini fallback a reasonable, bounded-cost enhancement *if* the CEO
wants to push past 91.3% toward closing the long tail — but it is not *required* to meet the core
educational-correctness bar, which the confidence gate already satisfies locally. Remains deferred per
explicit instruction; not implemented.

## 18. Regression (Milestone 6C)

Independently re-verified: curated CEO words and Korean pairing byte-identical; synonym borrowing
(argue/vague/vindicate/reticent) still works exactly as before; antonym rejection still holds (no
antonym-sourced borrowing anywhere — confirmed via `prevent`, still correctly excluded); all 3
morphology-safety cases (witnesses/witnessed/corroborated) byte-identical, including the untouched
Task-4 adjective-regeneration path that still depends on `fallbackFrameFamilies.adjective`; XSS escaping
intact; IndexedDB still 6 stores; Gold Master checksum unchanged. `git diff` on `index.html`: exactly 1
hunk, confined entirely to `buildFallbackVocabData()` and its immediately-preceding helper — nothing
elsewhere in the file touched.

## 19. Status (Milestone 6C)

Commit: `fix: gate low-confidence fallback examples`. Stopping before Independent QA. No push, no
merge.
