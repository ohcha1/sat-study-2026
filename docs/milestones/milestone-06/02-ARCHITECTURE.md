# Milestone 6 — Architecture Design: Example Sentence Quality Upgrade

Grounded directly in the current implementation — every function/field named below was read from
`index.html` at `main` @ `53bf0cc03a1b8831f091c1b8f7b1c6ead67df669` before this design was written,
not assumed from `01-PM-SPEC.md`'s description alone. One fact was independently re-confirmed for
this document specifically: `exampleBank` is referenced at exactly one call site in the entire file
(`index.html:4132`, inside `examples()`) — confirmed via `grep -n "exampleBank" index.html`, safe to
retire.

## 1. Approach summary

Two phases, both entirely local/offline, no new IndexedDB store, no new provider:

1. **Phase 1 — source-precedence fix.** `examples()` stops consulting the stale `exampleBank` and
   reads the current vocab item's own already-curated example (`v.data[3]`/`v.data[4]`) first. Zero
   new content, near-zero risk, fixes 1,089 of 1,115 curated words immediately.
2. **Phase 2 — POS-aware fallback engine.** Replaces the single POS-blind `fallbackExamplePool` with
   a small local POS-detection layer plus multiple sentence-frame families per part of speech, used
   only for genuinely-uncurated words and for the narrow "unsafe morphology reuse" case defined in
   §4.

Both phases together touch exactly four functions/objects: `examples()`, `buildFallbackVocabData()`,
`buildVocabCardData()` (its morphology branch only), and `fallbackExamplePool` (replaced by the new
frame-family data). `exampleBank` is deleted. Nothing else in `index.html` is touched.

## 2. Example source-precedence order (binding design)

```
1. Curated exact-form example
   — word matches dictionary[w] / supplementalVocab[w] / broadVocabLexicon[w] exactly.
   — Use data[3] (EN) / data[4] (KO) verbatim. Never rewritten, never regenerated.

2. Safely-reusable morphology/base example
   — word has no exact match, but safeMorphologyBase(w) finds a base word that does.
   — Reuse the base record's data[3]/data[4] verbatim ONLY if the §4 safety rule allows it.
   — If the safety rule disallows example reuse, meaning/POS/synonyms/antonyms are still
     reused from the base (word sense doesn't change under inflection — see §4), but the
     EXAMPLE fields specifically fall through to step 3 instead, using the base's POS
     (or the §4-adjusted POS) to select the right frame family.

3. POS-aware local fallback
   — word matches no lexicon and no safely-reusable base (or reached here from step 2's
     example-only fallthrough).
   — POS source, in order: (a) the base word's curated POS if step 2 supplied one but
     disallowed example reuse; (b) the local suffix-based POS guesser (§5) otherwise.
   — Select a sentence frame from that POS's frame family (§6), substitute {word}.

4. Final safe generic fallback
   — POS could not be determined by (a) or (b) at all (expected to be rare — English
     derivational suffixes are systematic; see §5).
   — Defaults to the NOUN frame family, per §21's "prefer conservative generic academic
     patterns" instruction — noun is the statistical default/elsewhere category in English
     derivational morphology, and this avoids ever reintroducing the "the class used the
     term '{word}'" meta-discussion framing anywhere in the new design.
```

This precedence order directly answers the investigation's own question (§5 of `01-PM-SPEC.md`):
prefer existing curated data at every step before falling through to generated content, and never
regress a word that already has a good example.

## 3. `exampleBank` disposition

**Retire — delete the object and its one call site.** Confirmed via direct grep this is the object's
only reference in the entire file; no other function, tab, or data path depends on it. Its content
is a byte-identical subset of `dictionary`'s first 26 entries (confirmed in the PM investigation), so
deleting it loses zero information — every word it covered is already served, correctly, by step 1
of §2's precedence order. Leaving it as inert dead code was considered and rejected: this repo's own
convention (applied repeatedly across Milestones 4 and 5) is to delete confirmed-unused code rather
than leave silent dead weight.

## 4. Morphology safety rules

**Field-level reuse, not whole-record reuse.** The current `buildVocabCardData()` morphology branch
treats a base-word match as an all-or-nothing curated hit — POS, meaning, synonyms, antonyms, *and*
example all come from the base record together. This is safe for four of those five fields (word
sense, part of speech, synonyms, and antonyms do not change under pure inflection — "researchers" and
"researcher" mean the same thing and are both nouns), but the CEO's own observation is specifically
about the fifth: an *example sentence* written to demonstrate the base word's usage does not
necessarily still demonstrate the inflected surface form's usage correctly.

**Safety rule, by inflection type** (`safeMorphologyBase()`'s own candidate types, `index.html:2333`):

| Inflection | POS-stability | Example reuse |
|---|---|---|
| Plural `-s`/`-es`/`-ies` | Always POS-stable (a noun stays a noun) | **Safe** — reuse base's example verbatim |
| 3rd-person singular `-s`/`-es` on a verb base | Always POS-stable | **Safe** — reuse verbatim |
| Past tense `-ed` | POS-stable **only if** the base's curated POS tag is a single, unqualified `"verb"` (no `adjective` component anywhere in the tag) | **Safe** if condition holds; otherwise falls through to Phase 2 (§2 step 3) using `"adjective"` as the working POS — `-ed` participles are exactly the class of word most likely to also function adjectivally, which is precisely why the base's verb-focused example can't be trusted here |
| Progressive `-ing` | Same rule as `-ed` | Same handling as `-ed` |

This keeps `curated:true` and the existing "Important SAT Words" / non-fallback tagging semantics
unchanged for the 4 safely-reused fields, while specifically protecting only the example sentence —
the one field the CEO identified as actually breaking. No other part of `buildVocabCardData()`'s
contract changes.

## 5. POS-detection strategy

- **Curated words (1,115):** POS is never guessed — `data[0]` is authoritative, already present for
  100% of curated entries.
- **Safely-reused morphology base:** base's `data[0]` is authoritative (§4).
- **Unsafe morphology base (`-ed`/`-ing`, adjective-capable base):** working POS forced to
  `"adjective"` for frame selection (§4's rationale).
- **Genuinely-uncurated words:** local suffix table, checked in this priority order (most reliable
  signal first, to resolve ties deterministically):

| Priority | POS | Suffixes |
|---|---|---|
| 1 | Adverb | `-ly` (a near-unambiguous signal in English) |
| 2 | Verb | `-ate`, `-ify`, `-ize`/`-ise`, `-en` |
| 3 | Adjective | `-ive`, `-ous`, `-able`/`-ible`, `-al`, `-ic`, `-ful`, `-less`, `-ary` |
| 4 | Noun | `-tion`, `-sion`, `-ment`, `-ness`, `-ity`, `-ance`, `-ence`, `-ism`, `-hood`, `-ship`, agent `-er`/`-or` |
| — | Unknown | no suffix matched → §2 step 4, defaults to the noun frame family |

Same technique class already proven in this codebase (`safeMorphologyBase()`'s own suffix matching,
`fallbackKoreanGloss()`'s suffix-based gloss heuristic) — no new architectural pattern, no external
dependency, fully deterministic and offline.

## 6. Sentence-frame families (Phase 2)

Per the CEO's explicit instruction, **not four rigid single templates** — a small family per POS, so
selection has real variety and the "not repetitive across words" quality bar holds. Wording below is
illustrative (per `01-PM-SPEC.md`'s own framing, finalized at Dev-Task time), the *structure*
(family count, grammatical role, frame category) is the binding Architecture decision:

- **Verb** (3 frames, matching the CEO's own categories) — infinitive position after "to," avoiding
  any need to inflect an unknown/possibly-irregular verb:
  - SVO: *"Students used the reading to {word} their argument more clearly."*
  - that-clause: *"The evidence seemed to {word} that the original explanation was incomplete."*
  - prepositional complement: *"The team continued to {word} on the same approach throughout the
    project."*
- **Adjective** (3 frames):
  - adjective + noun: *"The committee proposed a {word} solution to the ongoing problem."*
  - linking verb + adjective: *"The results seemed {word} once researchers reviewed the complete
    data."*
  - adjective + prepositional complement: *"The findings were {word} in ways the researchers had not
    expected."*
- **Adverb** (3 frames):
  - verb-modifying: *"Costs {word} increased throughout the following year."*
  - sentence adverb: *"{Word}, the results supported the original hypothesis."* (capitalized,
    sentence-initial, comma-separated — the standard slot for adverbs like "consequently,"
    "significantly")
  - degree/manner: *"The two approaches were {word} similar in their overall structure."*
- **Noun** (3 frames):
  - subject: *"The {word} became a central topic of discussion among researchers."*
  - object: *"Teachers explained the {word} using a familiar classroom example."*
  - academic collocation: *"The report provided a detailed analysis of the {word}."*
- **Unknown POS:** no dedicated frame family — routes to the Noun family (§2 step 4), never a new
  "meta-discussion of the word" frame, so the exact pattern being fixed can never resurface anywhere
  in the new system.

Each frame requires a hand-written, natural Korean counterpart at Dev-Task time (§9) — 12 EN/KO pairs
total (3 × 4 POS families), a small, boundable authoring task, not open-ended generation.

## 7. How repetition is reduced across multiple words

Selection within a POS family uses the same deterministic word-hash technique already established
and CEO-accepted in Milestone 4 (`buildFallbackVocabData()`'s `[...w].reduce((h,c)=>h+c.charCodeAt(0),0)%pool.length`
pattern) — applied per-POS-family instead of over one flat 12-entry pool. With 4 families × 3 frames
= 12 distinct structures split across POS categories (versus today's 12 structures competing in one
undifferentiated pool), two different words are less likely to collide on the same visible sentence
shape than today, and words of different POS can never collide with each other at all (they draw from
disjoint families). `examples()`'s existing de-dupe loop (`while(used.has(ex[0]))`, re-picks on
collision within a single render) is preserved, adapted to search within the appropriate POS family
rather than the old flat pool.

## 8. Determinism for the same word

Unchanged principle from the existing, CEO-accepted convention: frame selection is a pure function of
the word string itself (hash-based), not of render order, batch position, or session — so the same
word always produces the same example on repeat visits (Word Book, Review, re-analysis of a passage
containing the word again), matching the exact rationale already recorded in Milestone 4's
implementation log for the original `buildFallbackVocabData()` design.

## 9. Korean fallback approach

Each of the 12 new frame templates ships with a hand-authored Korean translation, following the exact
existing pattern in `fallbackExamplePool` (English frame paired 1:1 with a natural Korean sentence,
target word quoted inline in the Korean text the same way today's fallback pool already does). This
is translation-authoring work sized to 12 sentence pairs, not a translation *system* — no MT, no new
dependency, consistent with §16-17's offline/local requirement. Curated and safely-reused-morphology
paths need no new Korean content at all — `data[4]` already supplies it.

## 10. XSS / escaping requirements

Independently verified: `examples()` currently performs **zero** `escapeHtml()` calls on `v.word`,
`ex[0]`, or `ex[1]` before interpolating them into `innerHTML`. This is safe today only because none
of the current inputs are passage-derived — `v.word` is constrained to `[a-z']+` by the `words()`
extraction regex (cannot contain HTML-special characters), and `exampleBank`/`fallbackExamplePool`
are both static, hardcoded, author-controlled strings.

**Neither Phase 1 nor Phase 2 changes this safety property.** `v.data[3]`/`v.data[4]` (Phase 1's new
source) are static curated content from `dictionary`/`supplementalVocab`/`broadVocabLexicon` — the
exact same kind of data already rendered unescaped elsewhere in this codebase with an explicit
"safe by construction" rationale (documented at the Vocabulary Card's own `vocabCardTemplate()`).
Phase 2's frame templates are equally static, with only the safe-by-construction `{word}` substituted
in. **No new escaping is required for this milestone to be safe.**

**Recommended hardening, not required for safety:** wrap the final rendered `v.word`/`ex[0]`/`ex[1]`
in `escapeHtml()` anyway inside `examples()`, as a zero-cost defensive measure consistent with this
codebase's belt-and-suspenders pattern elsewhere (e.g. the wrong-answer report escapes fields that
are also already-safe-by-construction). This is cheap insurance against a *future* change accidentally
routing passage-derived content (e.g. `contextSentence`) through this same render path without
remembering to escape it at that time — flagged as a small, low-risk addition for Dev Task 5, not a
blocking requirement for this milestone's actual safety.

**Binding rule for any future change:** if this render path is ever extended to include genuinely
passage-derived text, that specific addition must be wrapped in `escapeHtml()` at the point of
interpolation, per this codebase's established default-to-escaping convention (already applied twice
before: the original Milestone-1 XSS fixes, and `contextSentence`'s explicit escaping in the
Vocabulary Card).

## 11. Regression risks

1. **Vocabulary Card / Word Book / Review must be unaffected.** These already read `v.data[3]`/
   `v.data[4]` correctly via `vocabCardTemplate()`/`buildVocabCardData()` — Phase 1 only changes
   `examples()`, a separate function; Phase 2's morphology-safety change is inside
   `buildVocabCardData()` but is additive/narrowing (only changes behavior for the specific unsafe
   `-ed`/`-ing` case identified in §4), not a rewrite of the existing correct paths.
2. **`exampleBank` removal** — single call site confirmed (§0), safe.
3. **`genVocabInContext()` (SAT vocabulary-in-context questions) must remain byte-for-byte
   unchanged.** Confirmed in the PM investigation and reconfirmed here: this function never calls
   `examples()`, `buildFallbackVocabData()`, `exampleBank`, or `fallbackExamplePool` — it builds its
   own evidence directly from the student's actual passage sentences and a separate, hand-curated
   `QG_VOCAB_DISTRACTORS` table. Nothing in this milestone's scope touches it, and Dev Task 5 must
   include an explicit confirmation test, not just an absence-of-touched-code assumption.
4. **`curated`/fallback tagging semantics must not shift.** The Important-SAT-Words filter and the
   "사전 수록 단어"/"일반 설명" tag both depend on `v.curated`. §4's field-level reuse keeps
   `curated:true` for the safely-reused fields even when the example itself is regenerated — this
   must not accidentally flip a word to `curated:false` (which would also incorrectly change its tag
   and its eligibility for the default Important-Words view).
5. **Word Book standalone reconstruction** (`buildVocabCardData(word)` with no `passageText`) must
   continue to work identically — Phase 2's fallback/frame-selection logic must not depend on a
   passage being in memory (it doesn't; it only depends on the word string itself, matching the
   existing `buildFallbackVocabData()` contract).
6. **Mobile Example tab.** Neither phase touches `examples()`'s outer HTML/CSS structure (the
   `.card`/`.en`/`.ko` classes, the mint/yellow alternating background) — only the string content fed
   into that unchanged structure changes. Structurally low-risk; still worth an explicit mobile
   smoke-test in Dev Task 5 per the CEO's acceptance criteria.
7. **No IndexedDB schema, no Gold Master, no Vocabulary UI layout, no Word Book/Review/pronunciation/
   SAT-Retry/Evidence-Sentence/weak-area-reinforcement/Gemini/translation/PDF-import code anywhere
   near this milestone's four touched functions** — confirmed by this and the PM investigation's
   direct reads; none of those systems reference `examples`, `buildFallbackVocabData`,
   `fallbackExamplePool`, or `exampleBank`.

## 12. Acceptance criteria (final, per CEO decision)

1. Curated words use their own curated example on the Example tab (§2 step 1) — verified for a
   representative sample across all four POS categories.
2. The Example tab no longer falls back to noun-like generic sentences for curated verbs,
   adjectives, or adverbs.
3. A representative curated verb functions as a verb in its rendered example.
4. A representative curated adjective functions as an adjective in its rendered example.
5. A representative curated adverb functions as an adverb in its rendered example.
6. A representative curated noun functions naturally as a noun in its rendered example.
7. Genuinely-uncurated words receive a POS-aware fallback example when a POS can be determined
   (§5), and a conservative noun-family fallback otherwise (§2 step 4) — never the old quoted-term
   framing.
8. Morphological fallback does not reuse a grammatically inappropriate base-word example (§4) —
   verified with at least one `-ed`/`-ing` word whose base is adjective-capable.
9. Korean translation remains available for every example, curated or fallback.
10. No AI/API dependency is introduced; no new provider; no paid service.
11. No IndexedDB schema change.
12. `genVocabInContext()` (SAT vocabulary-in-context questions) is byte-for-byte unchanged in
    behavior — explicit regression test, not an assumption.
13. Vocabulary Card and Word Book examples remain correct (unaffected by this milestone).
14. The mobile Example tab remains usable — no overflow, no clipping, existing layout intact.
15. No new XSS issue is introduced (§10) — recommended defensive `escapeHtml()` hardening applied.

## 13. Developer Task breakdown

1. **Fix `examples()`'s source precedence (Phase 1).** Read `v.data[3]`/`v.data[4]` as primary when
   `v.curated` is `true`; delete `exampleBank` and its one call site; keep the existing
   `fallbackExamplePool` call as an interim placeholder for the `curated:false` path until Task 3
   replaces it. This task alone is independently shippable and delivers the majority of the CEO's
   requested improvement.
2. **Build the local POS guesser** (§5) as a small, pure, testable function; verify against a sample
   word list spanning all four target POS categories plus a genuine "no signal" case.
3. **Design and implement the POS-aware frame families** (§6/§9) — 12 EN/KO template pairs across
   the four POS families, wired into `buildFallbackVocabData()` (replacing `fallbackExamplePool` for
   the example fields specifically) and into `examples()`'s fallback path for `curated:false` words.
4. **Implement the morphology field-level safety split** (§4) in `buildVocabCardData()`'s morphology
   branch — meaning/POS/synonyms/antonyms always reused when a base matches; example fields reused
   only when the `-s`/`-es`/`-ies` rule or the qualified `-ed`/`-ing` rule is satisfied, otherwise
   regenerated via Task 3's fallback engine using the appropriate working POS.
5. **Regression + documentation.** Full verification against §12's 15 acceptance criteria, explicit
   `genVocabInContext()` non-regression test, mobile Example-tab smoke test, before/after sentence
   samples for a representative word in each POS category, the recommended defensive-escaping
   addition (§10), and an implementation log.

## 14. Status

**APPROVED by the CEO — 2026-08-14.** Development may begin, scoped strictly to §13's five
Developer Tasks, within the constraints recorded in `01-PM-SPEC.md` §23.

---

## Handoff — Milestone 6 Architecture

- **Milestone:** Milestone 6 — Example Sentence Quality Upgrade.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/ARCHITECT.md`, `docs/milestones/milestone-06/01-PM-SPEC.md` (confirmed APPROVED),
  plus direct reads of `index.html` (`examples()`, `buildFallbackVocabData()`,
  `buildVocabCardData()`, `safeMorphologyBase()`, `fallbackExamplePool`, `exampleBank`,
  `fallbackKoreanGloss()`, `escapeHtml()`, `genVocabInContext()`) and a targeted grep confirming
  `exampleBank`'s single call site.
- **Scope completed:** Full architecture for both approved phases — exact source-precedence order,
  `exampleBank` disposition with rationale, field-level morphology safety rules (the CEO's specific
  "define when reuse is safe" requirement), POS-detection strategy with priority ordering,
  POS-aware frame-family design (4 families × 3 frames, explicitly not 4 rigid templates),
  repetition-reduction and determinism mechanisms, Korean-fallback authoring approach, XSS/escaping
  analysis (concluding no new escaping is strictly required, with a recommended hardening addition),
  11 named regression risks, 15 finalized acceptance criteria, and a 5-task Developer breakdown. No
  application code touched — confirmed via `git status` showing only documentation files changed.
- **Files changed:** `docs/milestones/milestone-06/01-PM-SPEC.md` (§23 CEO decisions added, status
  APPROVED), `02-ARCHITECTURE.md` (new, this file). Neither committed yet — pending CEO review,
  matching this repo's established PM/Architecture-phase pattern.
- **Commits created:** None this session.
- **Tests performed:** N/A (Architecture phase, no code written).
- **Unresolved risks:** None beyond §11's 11 named items, all with assigned mitigations and, where
  relevant, an assigned acceptance criterion or Developer Task test.
- **Next agent:** Senior Developer, to implement the 5 Developer Tasks in §13 — or CEO, if further
  scope revision is wanted first.
- **Explicit stop point:** Per the CEO's "STOP after Architecture" instruction. No application code
  has been modified.
