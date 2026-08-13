# Milestone 6 — Independent QA Report: Example Sentence Quality Upgrade

Role: Independent QA Engineer. Branch: `feature/example-sentence-quality` @ `60de44f` (confirmed —
`git rev-parse HEAD` matched the expected head before any testing began; working tree clean except
pre-existing untracked `.DS_Store`/`.claude/`). Method: local static server (`python3 -m http.server`,
no build step needed for this single-file app) + live in-browser JS execution and real click-through,
independent of and without reference to the Developer's own test scripts. No application code was
modified during this QA pass. Not pushed, not merged.

The governing question throughout, per the QA brief: **not** "is this grammatically valid?" but
**"would this genuinely help a high-school student understand when and how to use this word?"**

---

## 1. QA verdict

**PASS WITH CONDITIONS.**

The two defects that motivated this milestone — the Example tab's stale-`exampleBank` routing bug
(affecting 97.7% of curated vocabulary) and the "word treated as a noun regardless of real part of
speech" pattern — are **completely fixed for curated vocabulary and for the morphology-safety cases**,
which is the majority of real usage (1,115 curated words plus every safe/unsafe-classified inflection
of them). Curated-routing, `exampleBank` retirement, morphology safety, determinism, all cross-feature
regression, and mobile layout all pass cleanly and are ready as-is.

However, independent testing found that the **fallback engine for genuinely-uncurated words** (Tasks 2
and 3) does not meet the quality bar this QA pass was asked to apply. Specifically:

- A **systematic POS-misclassification defect**: common academic verbs/adjectives that don't end in one
  of `guessPOS()`'s specific recognized suffixes silently default to `"noun"` and are then rendered
  into a noun frame — reproducing the *exact* original bug (word used as a bare noun regardless of its
  real function) for a real, non-trivial slice of ordinary vocabulary, not just exotic edge cases. See
  §4/§15 finding F-1.
- The **verb frame family** (SVO/"argument", that-clause, prepositional-"on") produces AWKWARD or BAD
  sentences for the majority of genuinely-uncurated verbs tested. See §5/§15 finding F-2.
- Only **35% (7/20)** of the required Section-C word sample rated GOOD or better — well under the
  ~90% PASS bar defined for this QA pass (§8, §15 finding F-3).

None of this is a regression risk to anything else in the app (§13 — full regression clean, confirmed
independently including a true reload/re-login persistence check), and it does not undo the milestone's
real, large improvement. But per the explicit threshold in the QA brief ("a systematic frame family
still teaches unnatural usage" → FAIL), the fallback-quality area on its own does not clear the bar, and
Section G's realistic-passage walkthrough (§12) produced one case (`exacerbate`) that would actively
teach a student the wrong sense of a word. **Recommendation: treat curated/morphology behavior as
shippable, and route the fallback-engine gap back for a follow-up refinement pass before considering
Milestone 6 fully closed** (see §16).

---

## 2. Curated-routing result

**PASS — no issues found.** All 12 CEO-specified words (evidence, consequence, hypothesis /
demonstrate, establish, mitigate / significant, plausible, ambiguous / consequently) were independently
tested. For each: the example belongs to that exact word (confirmed by comparing `data[3]`/`data[4]`
directly against the rendered Example-tab HTML), the target word is used in its real grammatical role,
the sentence's content matches the dictionary meaning, the English is natural academic register, and
the Korean is a faithful pairing. Two of the twelve CEO-listed words — `empirically` and `progressively`
— turned out **not** to be curated in this codebase's lexicon (confirmed via direct lookup across
`dictionary`/`supplementalVocab`/`broadVocabLexicon`); they were re-tested as fallback-adverb data
points instead (§4/§7) rather than as curated-routing cases, since testing them as "curated" would have
been factually incorrect.

Also independently confirmed: `git diff 71edcf8..HEAD -- index.html` touches exactly 6 hunks, all
inside the targeted regions — no risk of curated routing being accidentally coupled to unrelated code.

## 3. `exampleBank` retirement result

**PASS.** `grep -c exampleBank index.html` → 0 live references (one explanatory comment only).
Independently reconstructed the deleted object's original 26-word list from
`git show 71edcf8:index.html` and confirmed all 26 words still have intact `dictionary` entries with
non-empty `data[3]`/`data[4]` — zero data loss from the deletion. Also confirmed curated Example-tab
coverage is now independent of that former table: 8 curated words that were **never** in the old
26-word list (welfare, vulnerable, withstand, witness, workforce, voluntary, consequence, establish)
all render correct, non-generic examples.

## 4. Noun fallback quality

Tested 5 required (resilience, discrepancy, juxtaposition, ambivalence, dichotomy) + 1 mobile spot-check
(unconstitutionality) — genuinely uncurated, confirmed via direct lookup.

| Word | Frame | Rating |
|---|---|---|
| resilience | subject ("The resilience became...") | ACCEPTABLE — grammatical, but an abstract relational noun like this normally needs a complement ("resilience *of* the community"); the bare form reads slightly incomplete |
| discrepancy | subject | GOOD — "a discrepancy became a topic of discussion" is a natural, plausible academic sentence |
| juxtaposition | object ("Teachers explained the juxtaposition...") | ACCEPTABLE — same relational-noun gap ("juxtaposition *of X and Y*" is the natural form) |
| ambivalence | subject | ACCEPTABLE — same relational-noun gap |
| dichotomy | object | ACCEPTABLE — same relational-noun gap |
| unconstitutionality | academic collocation ("...detailed analysis of the unconstitutionality") | GOOD |

**No BAD/AWKWARD results in the noun family.** Secondary, low-severity pattern: relational abstract
nouns that idiomatically require an "of X" complement (resilience, ambivalence, dichotomy,
juxtaposition) read slightly bare/incomplete in the bare-noun frames, though never wrong or misleading.
Noun family: 2 GOOD, 4 ACCEPTABLE, 0 AWKWARD/BAD out of 6.

## 5. Verb fallback quality

Tested 5 required (delineate, circumvent, dissipate, perpetuate, exacerbate) + 1 CEO-named check
(codify) — genuinely uncurated, confirmed via direct lookup.

| Word | Frame | Rating | Why |
|---|---|---|---|
| delineate | SVO ("...to delineate their argument more clearly") | GOOD | natural collocation |
| circumvent | **misclassified as noun** → "The report provided a detailed analysis of the circumvent." | **BAD** | POS-detection failure (F-1) — a verb rendered as a bare noun, the exact original bug pattern |
| dissipate | SVO ("...to dissipate their argument more clearly") | AWKWARD | semantically contradictory — "dissipate" means disperse/fade, incompatible with "more clearly" |
| perpetuate | that-clause ("...seemed to perpetuate that the original explanation was incomplete") | **BAD** | "perpetuate that X" is not standard English — perpetuate doesn't take a that-clause |
| exacerbate | SVO ("...to exacerbate their argument more clearly") | AWKWARD/BAD | "exacerbate" means worsen; pairing with "more clearly" and "their argument" is both semantically contradictory and would actively mislead a student about the word's negative valence |
| codify | prepositional-"on" ("...continued to codify on the same approach") | BAD | "codify" is transitive ("codify the rules"); "codify on X" is not idiomatic English |

**1 GOOD out of 6 (17%).** This is the most severe finding of the QA pass: every frame in the verb
family (SVO/"argument", that-clause, prepositional-"on") produced at least one clearly bad result, and
5 of 6 tested verbs landed on AWKWARD or BAD. See §15 finding F-2.

## 6. Adjective fallback quality

Tested 5 required (meticulous, unwarranted, insular, derivative, tenuous), genuinely uncurated.

| Word | Frame | Rating |
|---|---|---|
| meticulous | adj+noun ("...proposed a meticulous solution") | ACCEPTABLE — grammatical but an unusual collocation (meticulous more naturally pairs with plan/analysis/records, not "solution") |
| unwarranted | **misclassified as noun** → "Teachers explained the unwarranted using a familiar classroom example." | **BAD** | POS-detection failure (F-1) |
| insular | **misclassified as noun** → "Teachers explained the insular using a familiar classroom example." | **BAD** | POS-detection failure (F-1) |
| derivative | linking-verb+adj ("The results seemed derivative...") | GOOD |
| tenuous | linking-verb+adj ("The results seemed tenuous...") | EXCELLENT — textbook-natural collocation |

**2 GOOD/EXCELLENT, 1 ACCEPTABLE, 2 BAD (40%)** — both BAD cases are the same systematic
POS-misclassification defect as the verb family (F-1), not a frame-wording problem.

## 7. Adverb fallback quality

Tested 5 required (ostensibly, markedly, inadvertently, unequivocally, overtly) + the CEO's named
manner-adverb set (meticulously, cautiously, deliberately*, systematically, gradually). (*"deliberately"
turned out to be curated — see §8.)

| Word | Frame | Rating |
|---|---|---|
| ostensibly | degree/manner ("...were ostensibly similar...") | GOOD |
| markedly | degree/manner | EXCELLENT |
| inadvertently | sentence-adverb ("Inadvertently, the results supported...") | GOOD |
| unequivocally | verb-modifying ("Costs unequivocally increased...") | AWKWARD — unequivocally normally modifies assertion verbs (stated/confirmed), not quantity-change verbs |
| overtly | verb-modifying ("Costs overtly increased...") | AWKWARD — overtly means "openly," doesn't naturally pair with a quantity change |
| meticulously | sentence-adverb ("Meticulously, the results supported...") | AWKWARD — manner adverb, not idiomatic as a sentence-initial stance adverb |
| cautiously | degree/manner ("...were cautiously similar...") | AWKWARD — "cautiously similar" is not a real collocation |
| systematically | sentence-adverb | AWKWARD (borderline ACCEPTABLE) — marginally more tolerable than "meticulously" but still not a natural connective use |
| gradually | degree/manner ("...were gradually similar...") | AWKWARD — gradually belongs with change-over-time verbs, not a static "similar" |

**4 GOOD/EXCELLENT, 5 AWKWARD (56%)** out of 9. Confirms the disclosed limitation is real and
systematic: the sentence-adverb and degree/manner frames were modeled on connective/degree adverbs
(consequently, markedly) and reliably misfire on manner adverbs (meticulously, cautiously,
systematically, gradually). Note `deliberately` is actually curated in this lexicon (own real example:
"The director deliberately left the ending ambiguous..." — EXCELLENT) and `consequently`/`nevertheless`
are also curated — meaning the fallback engine's adverb weakness never surfaces for those specific
words in practice, only for uncurated manner adverbs like the ones tested here.

## 8. Naturalness score distribution

Combined tally across every fallback word independently tested this QA pass (Sections C+D+G, excluding
duplicates and excluding words later found to be curated):

| Rating | Count | % |
|---|---|---|
| EXCELLENT | 2 | 6% |
| GOOD | 9 | 26% |
| ACCEPTABLE | 6 | 17% |
| AWKWARD | 11 | 31% |
| BAD | 7 | 20% |

n = 35 (distinct genuinely-uncurated words tested: 5 nouns + 1 mobile spot-check, 6 verbs, 5
adjectives, 9 adverbs, plus 9 additional words from the broader POS-misclassification probe in §15
finding F-1 — all confirmed uncurated). **GOOD-or-better = 11/35 = 31%; on the required 20-word
Section-C-only sample specifically, GOOD-or-better = 7/20 = 35%.** Both are far short of the ~90%
PASS bar defined for this QA pass.

## 9. Morphology result

**PASS — the safety mechanism itself behaves exactly as designed**, independently re-verified for all 6
required classes using fresh word choices (not reused from the Developer's own test set):

| Case | Word | Base | Base POS | Result |
|---|---|---|---|---|
| Plural noun | witnesses | witness | noun/verb | Safe — base reused verbatim |
| 3rd-person verb | corroborates | corroborate | verb | Safe — base reused verbatim |
| Past-tense verb | corroborated | corroborate | verb | Safe — base reused verbatim |
| -ing verb | corroborating | corroborate | verb | Safe — base reused verbatim |
| Adjective-like -ed | witnessed | witness | noun/verb | Unsafe — example regenerated via adjective frame; meaning/POS/synonyms/antonyms still from base |
| Adjective-like -ing | witnessing | witness | noun/verb | Unsafe — example regenerated via adjective frame; meaning/POS/synonyms/antonyms still from base |

All match the Developer's claims exactly. **One additional finding from independent probing beyond the
required 6 cases**, rated moderate severity: the "regenerate via adjective frame" fallback for unsafe
`-ed` forms is not itself foolproof. Testing `interested` vs. `interesting` (same base "interest,"
same code path, both routed to the adj+prep-complement frame) produced:
- `interesting` → "The findings were interesting in ways the researchers had not expected." — GOOD.
- `interested` → "The findings were interested in ways the researchers had not expected." — **BAD**
  ("interested" is an experiencer-adjective requiring an animate subject; "findings" cannot be
  "interested." A student reading this would learn an incorrect usage.)

This is a real gap in the Task 4 safety net, not just a Task 3 frame-naturalness issue: the -ed/-ing
"stimulus vs. experiencer adjective" distinction (interesting/interested, exciting/excited,
surprising/surprised, boring/bored, etc. — a well-known English grammar category) is not accounted for.
See §15 finding F-4.

## 10. POS/example-consistency assessment

For the `witness → witnessed/witnessing` unsafe-morphology case, the Vocabulary Card displays
**POS = "noun/verb"** while the example sentence uses the word in an adjective slot ("The results
seemed witnessed..."). Judged from a student's perspective: the underlying facts aren't false (the
*base word* "witness" genuinely is noun/verb), but the presentation gives no indication that the
*specific inflected form shown* is being demonstrated in a different role than the label states, and
separately, "The results seemed witnessed" is itself a semantically odd sentence even on its own terms
(a "witnessed" result isn't a natural predicate-adjective usage the way "documented" or "corroborated"
results would be). **Verdict: CONFUSING** (not INCORRECT — no factual claim is wrong, but the
unexplained label/example mismatch, compounded by a shaky example sentence, could leave a student
unsure what they're actually being taught).

## 11. Determinism/variety result

**PASS.** Determinism: 5 repeated calls to `buildVocabCardData("resilience")` returned an identical
`data[3]` every time. Variety: a small 5-word noun sample showed some clustering (3/5 on one frame,
2/5 on another, 0/5 on the third) — but a broader 15-word verb sample showed a reasonable 6/3/6 spread
across all 3 frames, confirming the clustering was small-sample noise, not a systematic hash bias.
Korean pairing: independently confirmed each fallback word's Korean sentence structurally mirrors its
selected English frame (subject-frame Korean paired with subject-frame English, etc.) across all
samples checked — no mismatches found. Adjacent-card visual check (12-word mixed-POS batch): no two
adjacent cards shared an identical sentence shape, since different-POS words draw from disjoint frame
families by construction.

## 12. Student educational-quality verdict

**Overall: PARTIAL.** Walkthrough passage: "Researchers found significant evidence to demonstrate the
resilience of the plan. They tried to delineate the boundaries with meticulous care. Ostensibly the map
was correct, but a discrepancy remained that could exacerbate delays. The two studies showed a tenuous,
derivative link that was markedly weaker than expected." → top-12 extracted: discrepancy, significant,
demonstrate, boundaries, resilience, meticulous, ostensibly, exacerbate, derivative, researchers,
evidence, studies (curated: significant, demonstrate, boundaries, researchers, evidence, studies;
fallback: discrepancy, resilience, meticulous, ostensibly, exacerbate, derivative).

Per-word "would I know how to use this word in my own sentence after reading this?":

- **YES (8/12):** significant, demonstrate, boundaries, ostensibly, derivative, researchers, evidence,
  studies — all curated words plus 2 of the better-performing fallback words.
- **PARTIAL (3/12):** discrepancy, resilience (relational nouns shown without their natural "of X"
  complement — teaches the POS correctly but not the full natural usage), meticulous (unnatural
  collocation risk).
- **NO (1/12):** exacerbate — "Students used the reading to exacerbate their argument more clearly."
  A student reading this would plausibly conclude "exacerbate" means something like *clarify* or
  *strengthen*, which is close to the **opposite** of its real meaning (worsen). This is not merely
  awkward; it is actively miseducating.

This mirrors the Section 1-8 findings precisely: curated content is excellent, the fallback engine is
not yet reliable enough for unqualified confidence.

## 13. Regression result

**PASS — no regressions found**, independently re-verified (not just re-reading the Developer's log):

- `git diff 71edcf8..HEAD -- index.html`: exactly 6 hunks, confined to the exampleBank/
  fallbackFrameFamilies/safeMorphologyBase-adjacent/buildVocabCardData/buildFallbackVocabData/
  examples() regions. Nothing touches `genVocabInContext()` (line 3152), the Vocabulary Card template
  (word-card markup), Gemini functions, `translateSentenceReliable`, PDF/OCR/HEIC handling, or
  `openDB()`.
- `genVocabInContext()`: independently re-invoked directly (fresh passage, not reused from Development)
  → produced a correct, well-formed Vocabulary-in-Context question with 3 plausible distractors.
- IndexedDB: `grep createObjectStore` → exactly 6 stores (profiles, satAttempts, wrongAnswers,
  studySessions, savedPassages, vocabularyProgress) — unchanged.
- Gold Master: checksum `a51c603...2ba` — unchanged from the session baseline.
- XSS: re-tested independently — escaping is applied before `highlightExampleWord()` inserts its
  `<span>`, confirmed the span markup survives while a synthetic `<script>`/`&`/`"..."` payload renders
  fully escaped.
- Vocabulary Card, Word Book (save/unsave + review-state cycling), pronunciation button, SAT Practice
  generation — all functioned correctly during live click-through, zero console errors.
- **True persistence check (explicitly required beyond Development's single-page-load test):** created
  a fresh local profile, saved 2 fallback words to the Word Book with distinct review states
  (`discrepancy` → reviewing, `resilience` → learned), then performed a genuine full page reload
  (`navigate` with `force:true`, not an SPA state reset) and re-opened the Word Book **without
  re-analyzing any passage**. Both words reappeared with correct data (POS, source tag) and their exact
  review states intact, confirming IndexedDB-backed persistence survives a real reload, not just
  in-session state.

## 14. Mobile result

**PASS.** At 375×812: Example tab is fully readable, cards stack correctly with no horizontal overflow,
Korean translations wrap normally, and a genuinely long target word (`unconstitutionality`, 20
characters) rendered without breaking the card layout in either the heading or body text. No console
errors at this viewport.

## 15. Findings by severity

**F-1 — HIGH — Systematic POS-misclassification defaults to noun for common vocabulary not covered by
`guessPOS()`'s suffix table.** `guessPOS()` only recognizes `-ly` (adverb); `-ate/-ify/-ize/-ise/-en`
(verb); `-ive/-ous/-able/-ible/-al/-ic/-ful/-less/-ary` (adjective); and a noun-suffix list — anything
else silently defaults to noun. A targeted 21-word probe of ordinary academic verbs/adjectives that
fall outside this table (circumvent, prevent, invent, suggest, argue, claim, depict, convey, present,
address / unwarranted, insular, adverse, subtle, blunt, overt, covert, vague, stark, bleak, harsh) was
misclassified as noun **21/21 times (100%)**. This pattern also appeared organically in the
required Section-C sample (circumvent, unwarranted, insular — 3/20, 15%) without any deliberate
cherry-picking. The downstream effect — e.g. "The report provided a detailed analysis of the
circumvent." — is a verbatim reproduction of the exact defect this milestone exists to fix, just
occurring in the new fallback POS-guesser instead of the old `exampleBank`. This is not a frame-wording
issue; it's a coverage gap in POS detection itself, and it is the single largest factor behind the
low GOOD-or-better rate in §8.
*Scenario:* a student encounters "prevent," "argue," "vague," or "harsh" as a genuinely-uncurated word
in a passage → Vocabulary Card and Example tab both show it framed as a noun → student learns the wrong
grammatical function for a common word.

**F-2 — HIGH — Verb frame family produces unnatural/incorrect English for the majority of tested
verbs, independent of F-1.** Even when POS is correctly detected as "verb," the 3 verb frames
(SVO-with-"their argument", that-clause, prepositional-"on") are narrow: the that-clause frame doesn't
suit verbs outside the cognition/assertion class (perpetuate → "seemed to perpetuate that..." is not
standard English); the "on" frame doesn't suit verbs that aren't naturally on-complementing (codify →
"continued to codify on..." is not idiomatic); and the SVO frame's fixed object "their argument"
produces semantically contradictory sentences for verbs whose core meaning conflicts with
"clarify"/argue-related content (dissipate, exacerbate). 5 of 6 independently-tested genuinely-uncurated
verbs (excluding the F-1 misclassification case) landed AWKWARD or BAD.
*Scenario:* a student looks up "exacerbate" (a common SAT word) as a genuinely-uncurated item → shown
"Students used the reading to exacerbate their argument more clearly" → could reasonably conclude
exacerbate means "clarify/strengthen," the near-opposite of "worsen."

**F-3 — MEDIUM — Adverb sentence-adverb and degree/manner frames misfire on manner adverbs.**
Disclosed by Development as a known limitation; independently confirmed and quantified: 5/9 tested
adverbs (56%) rated AWKWARD, specifically manner adverbs (meticulously, cautiously, systematically,
gradually) forced into frames modeled on connective/degree adverbs. Lower severity than F-1/F-2 because
the words affected are typically less central to POS confusion (the adverb is still recognizably an
adverb; the *collocation* is what's off) and because several of the most common connective adverbs
(consequently, nevertheless, deliberately) are already curated and never hit this path.

**F-4 — MEDIUM — Task 4's "regenerate via adjective frame" safety net is not foolproof for
experiencer/stimulus -ed/-ing adjective pairs.** `interested` (base "interest", noun/verb, -ed form)
regenerates to "The findings were interested in ways the researchers had not expected" — ungrammatical,
since "interested" requires an animate/sentient subject, unlike its counterpart "interesting" (same
base, same code path) which produces a correct sentence. Affects a known but narrow English grammar
category (interesting/interested, exciting/excited, surprising/surprised, boring/bored, etc.).

**F-5 — LOW — POS-label/example mismatch for unsafe-morphology words is unexplained to the student.**
`witnessed`/`witnessing` display POS "noun/verb" while their example demonstrates adjectival use, with
no annotation bridging the two. Rated CONFUSING, not INCORRECT (§10).

**F-6 — LOW — Relational abstract nouns read slightly incomplete without their natural "of X"
complement.** resilience, ambivalence, dichotomy, juxtaposition — grammatical and not misleading, but
the bare-noun frames don't show the collocation pattern (resilience *of* the community) a student would
need to use the word naturally themselves.

**No HIGH/CRITICAL findings in curated routing, `exampleBank` retirement, morphology safety, XSS,
determinism, or any cross-feature regression area — all clean.**

## 16. Recommended next action

Do **not** block or roll back the curated-routing, `exampleBank`-retirement, or morphology-safety work
(Tasks 1 and 4) — independently verified excellent, zero regressions, ready as-is. Recommend a **scoped
follow-up Dev Task** (not a full re-architecture) targeting specifically F-1 and F-2, since those two
findings account for most of the shortfall against the 90% bar:

1. **F-1 fix:** expand `guessPOS()`'s suffix coverage and/or add a small curated exception list for
   very common short verbs/adjectives that don't match any current suffix pattern, so the "default to
   noun" fallback triggers less often for ordinary vocabulary. This is the highest-leverage fix — it
   alone would likely resolve most of the BAD-rated cases in §5/§6.
2. **F-2 fix:** either narrow the that-clause and prepositional-"on" verb frames to cases where they're
   safe, or replace them with a frame that's grammatical for a wider range of verbs (e.g. a plain
   transitive-object frame requiring no specific preposition or complement type).
3. F-3/F-4/F-5/F-6 are lower priority and could reasonably ship as-is with these limitations documented,
   or be addressed in the same follow-up pass if convenient.

This QA pass found no reason to distrust anything the Development phase reported — every claim in
`03-IMPLEMENTATION-LOG.md` was independently reproduced. The gap here is specifically that the
Development team's own disclosed "known limitation" (stiff wording for a few adverbs) undersold the
actual scope of the fallback-engine issue, which independent testing showed is more systematic and
includes outright incorrect-POS and incorrect-meaning-implying cases, not just naturalness roughness.

---

**STOP at QA gate. No push, no merge, no application-code changes made during this QA pass.**
