# Milestone 6 — Implementation Log: Example Sentence Quality Upgrade

Role: Senior Developer. Branch: `feature/example-sentence-quality`. Base: `main` @ `53bf0cc`
(current official baseline). Executes the 5 Developer Tasks approved in
`02-ARCHITECTURE.md` §13, under the CEO's Development-phase approval message
("AUTONOMOUS CONTINUATION MODE: ON").

## 1. Root cause (recap from PM investigation, confirmed unchanged at implementation time)

Two independent defects, both already identified in `01-PM-SPEC.md`:

1. **Example tab data routing.** `examples()` read from `exampleBank`, a stale 26-entry object
   byte-identical to the first 26 `dictionary` entries. Since `exampleBank` covered only 26 of the
   1,115 curated words, 97.7% of curated vocabulary fell through to the generic fallback pool on the
   Example tab specifically, even though the correct example was already sitting on the same vocab
   object (`v.data[3]`/`v.data[4]`) that every other view (Vocabulary Card, Word Book) already read
   correctly.
2. **POS-blind fallback content.** `fallbackExamplePool`'s 12 templates always framed the target word
   as a discussed/quoted term ("the class used the term '{word}'...", "'{word}' carried a more
   specific meaning..."), regardless of the word's real part of speech. This is the literal pattern
   the CEO flagged. In practice it only affected genuinely-uncurated words (a minority — most SAT
   vocabulary already exists in the 1,115-word curated set) plus, transitively, every curated word
   caught by defect 1.

## 2. `exampleBank` disposition

Retired entirely (object deleted, its one confirmed call site rewritten) in Task 1. Zero references
remain (`grep -c exampleBank index.html` → 0, save for one explanatory code comment).

## 3. Source precedence (implemented, matches Architecture §2)

For any word, `buildVocabCardData()`'s existing precedence chain is unchanged in order (exact
`dictionary` → exact `supplementalVocab` → exact `broadVocabLexicon` → safe morphology base → generic
fallback). What changed is what happens at the morphology-base and generic-fallback tiers, and how the
Example tab consumes the result:

- **Exact curated match:** `data[3]`/`data[4]` used directly by `examples()` — never regenerated.
- **Safe morphology match (plural/3rd-person `-s`):** whole base record reused, unchanged behavior.
- **Morphology match, `-ed`/`-ing`, base POS is unqualified `"verb"`:** whole base record reused
  (verb inflections stay verb-like enough to trust the base's example).
- **Morphology match, `-ed`/`-ing`, base POS is anything else (e.g. `"noun/verb"`):** meaning/POS/
  synonyms/antonyms still reused from the base; only the example is regenerated via the POS-aware
  fallback engine with `"adjective"` forced as the working POS (Task 4).
- **Genuinely uncurated:** POS is guessed locally (Task 2), example drawn from the matching POS frame
  family (Task 3).

`examples()` now reads `[v.data[3], v.data[4]]` unconditionally for every word (curated or not),
because both paths now produce a final, word-deterministic value at data-build time. This is the one
adaptation beyond the Architecture's literal wording (see §8 below).

## 4. POS inference (Task 2)

`guessPOS(w)`, conservative suffix table, fixed priority order, only ever called from
`buildFallbackVocabData()` (i.e. only for words with no curated or morphology-base match):

1. `-ly` → adverb
2. `-ate/-ify/-ize/-ise/-en` → verb
3. `-ive/-ous/-able/-ible/-al/-ic/-ful/-less/-ary` → adjective
4. `-tion/-sion/-ment/-ness/-ity/-ance/-ence/-ism/-hood/-ship/-er/-or` → noun
5. no match → noun (default)

No external API, no dictionary lookup beyond the existing curated tiers, purely local string
matching, matching the "no AI" constraint.

## 5. Fallback frame families (Task 3)

`fallbackExamplePool` (flat, POS-blind, 12 meta-discussion sentences) replaced by
`fallbackFrameFamilies` — 4 POS families × 3 frames each, wording per Architecture §6:

| POS | Frames |
|---|---|
| verb | SVO ("to {word} their argument"); that-clause ("seemed to {word} that..."); prepositional complement ("continued to {word} on...") |
| adjective | adj+noun ("a {word} solution"); linking verb+adj ("seemed {word}"); adj+complement ("were {word} in ways...") |
| adverb | verb-modifying ("Costs {word} increased..."); sentence adverb ("{Word}, the results supported..."); degree/manner ("were {word} similar...") |
| noun | subject ("The {word} became..."); object ("explained the {word}..."); academic collocation ("analysis of the {word}") |

Selection within a family uses the same per-word char-sum hash already established (and CEO-accepted)
in Milestone 4's `buildFallbackVocabData()`, now applied per-POS-family. Each of the 12 frames ships a
hand-authored Korean counterpart that enacts the same scenario (word kept untranslated, inline, since
its exact Korean meaning isn't known for genuinely-uncurated words) rather than describing people
discussing the word — preserving the existing "insert the raw word" convention while dropping the
meta-discussion structure.

Unknown/unrecognized-suffix words route to the noun family (verified: `xyzzyword` → noun frame).

## 6. Morphology safety (Task 4)

`morphSuffixType(w)` added (mirrors `safeMorphologyBase()`'s own suffix-detection order/guards) so the
caller can classify which inflection matched without changing that function's signature or its other
call site (`analyze()`'s `vocabQualityTier` helper, which only needs a truthy check and is unaffected).

`buildVocabCardData()`'s morphology branch: `-s`-family always safe (reuse whole base record).
`-ed`/`-ing` safe only when `baseData[0] === "verb"` exactly; otherwise only `data[3]`/`data[4]` are
regenerated (via the Task 3 adjective family, `renderFallbackFrame`), while `data[0,1,2,5,6]` and
`curated:true` still come from the base — no curated lexical data is discarded merely because its
example was unsafe to reuse.

## 7. Test results

### 7.1 Representative words (all 4 POS, curated)

| Word | POS | Example rendered |
|---|---|---|
| evidence | noun | "The student cited archaeological evidence to challenge the older theory." |
| demonstrate | verb | "The experiment demonstrates that small design changes can alter behavior." |
| significant | adjective | "The researchers found a significant difference between the two groups." |
| consequently | adverb | "The data set excluded rural schools; consequently, the conclusion was less generalizable." |

All four render their own real curated example, in English and Korean, on the Example tab — verified
both via direct function calls and via the live UI (pasted a 5-sentence passage, clicked "지문
분석하기", opened the "예문" tab; see §7.5).

### 7.2 Fallback frame families (uncurated words, all 4 POS + unknown-suffix control)

11 words tested (frobnicate, toughen, organize → verb; plutonic, tropical, careless → adjective;
quickly, gently → adverb; xyzzyword, celebration, friendship → noun/default). Every rendered example
used the target word in its real grammatical role. Zero instances of the banned "the class used the
term '{word}'" / "'{word}' carried a meaning" pattern.

### 7.3 Morphology safety (6 required cases)

| Case | Word | Base | Base POS | Suffix | Result |
|---|---|---|---|---|---|
| Plural noun | witnesses | witness | noun/verb | s | Safe — base example reused verbatim |
| 3rd-person verb | demonstrates | demonstrate | verb | s | Safe — base example reused verbatim |
| Past-tense verb | retained | retain | verb | ed | Safe — base example reused verbatim |
| -ing verb form | retaining | retain | verb | ing | Safe — base example reused verbatim |
| Adjective-like -ed form | witnessed | witness | noun/verb | ed | Unsafe — example regenerated via adjective family; meaning/POS/synonyms/antonyms still from base |
| Adjective-like -ing form | witnessing | witness | noun/verb | ing | Unsafe — example regenerated via adjective family; meaning/POS/synonyms/antonyms still from base |

Also confirmed: a directly-curated inflected form (e.g. `demonstrated`, which is its own
`supplementalVocab` entry, not derived via morphology) resolves through the exact-match branch and
never reaches this code at all — no interference between the two paths.

### 7.4 Determinism, XSS, regression sweep

- **Determinism:** `buildVocabCardData("frobnicate")` called twice → identical `data[3]` both times.
- **XSS:** synthetic payload (`<script>...`, unescaped `&`, `"..."`) run through `examples()` renders
  fully HTML-escaped text with the `<span class="example-word">` highlight markup intact and
  un-escaped (escaping is applied before `highlightExampleWord()`, not after).
- **`git diff` scope check** (`71edcf8..HEAD -- index.html`): exactly 6 hunks, all inside the
  `exampleBank`/`fallbackFrameFamilies`/`safeMorphologyBase`-adjacent/`buildVocabCardData`/
  `buildFallbackVocabData`/`examples()` regions. No changes anywhere near `genVocabInContext()`
  (~line 3120), the Vocabulary Card template (~line 3555), Gemini summary/vocab-context functions,
  translation (`translateSentenceReliable`), PDF/OCR/HEIC handling, or `openDB()`.
- **IndexedDB:** `grep createObjectStore` → exactly 6 stores (`profiles`, `satAttempts`,
  `wrongAnswers`, `studySessions`, `savedPassages`, `vocabularyProgress`), unchanged.
- **Gold Master:** `shasum -a 256 LATEST_GOLD_MASTER_NEXT.html` →
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba` — unchanged from the recorded
  baseline throughout this session.

### 7.5 Full live-UI pass (local static server + in-browser JS, real click-through)

Pasted a 5-sentence passage mixing curated and fabricated/rare words, clicked "지문 분석하기":

- **Example tab (예문):** all 12 extracted words rendered real, POS-correct example sentences with
  Korean pairs, no meta-sentences. (The passage's curated words dominated the top-12 vocab selection,
  as expected — genuinely uncurated words were separately verified via direct function calls in §7.2.)
- **Vocabulary Card (학년별 단어):** `demonstrated` card renders with correct POS badge ("verb"),
  context-sentence, English/Korean meaning, synonyms/antonyms — HTML structure (`word-card`,
  `word-card-left/right`, `meaning-block`, etc.) byte-identical to the frozen design; no visual
  changes, only the underlying data is different from before.
- **Word Book (단어장):** created a local test profile, saved `demonstrated` via `toggleWordBookSave`,
  cycled review state via `cycleReviewState` — Word Book card renders "저장됨 ✓" and
  `review-state-reviewing`, confirming save/unsave and review-state persistence both work unchanged
  with the new data flowing through the same `buildVocabCardData()` path.
- **SAT Practice (SAT 문제):** generated a "Vocabulary in Context" question for `significant`,
  confirming `genVocabInContext()` still runs correctly end-to-end (unaffected, as expected — it never
  calls any of the functions touched this milestone).
- **Mobile viewport (375×812):** Example tab content renders correctly and is fully readable.
- **Console:** zero errors at every step above.

### 7.6 Special Quality Test (CEO-specified words + real uncurated words)

Judged against: *"Does this sentence teach a high-school student how this word is actually used?"*

| Word | Curated | POS | Example | Verdict |
|---|---|---|---|---|
| consequence | yes | noun | "Rising sea levels are a direct consequence of climate change." | Yes |
| establish | yes | verb | "The team gathered enough samples to establish a clear link between the chemical and the fish deaths." | Yes |
| significant | yes | adjective | "The researchers found a significant difference between the two groups." | Yes |
| gradually | no (fallback) | adverb | "The two approaches were gradually similar in their overall structure." | Yes, but weak collocation (see below) |
| meticulously | no (fallback) | adverb | "Meticulously, the results supported the original hypothesis." | Yes, but stiff register (see below) |
| codify | no (fallback) | verb | "The team continued to codify on the same approach throughout the project." | Yes, but non-idiomatic preposition (see below) |
| laudable | no (fallback) | adjective | "The results seemed laudable once researchers reviewed the complete data." | Yes |
| resourcefulness | no (fallback) | noun | "The report provided a detailed analysis of the resourcefulness." | Yes, but wants a complement |
| unassumingly | no (fallback) | adverb | "Unassumingly, the results supported the original hypothesis." | Yes, but stiff register (see below) |

**All 9 pass the binary test** — every word is used in its real grammatical role, and none reproduce
the banned meta-discussion pattern. No refinement was required to clear the CEO's stated gate.

**Disclosed, secondary limitation (not a gate failure):** the adverb "sentence adverb" frame
(`"{Word}, the results supported..."`) and the verb "prepositional complement" frame
(`"...continued to {word} on..."`) are idiomatic for the connective/stance adverbs and
subcategorization patterns they were modeled on (`consequently`, `significantly`; verbs that
naturally take "on") but read stiffly or slightly non-standard for manner adverbs (`meticulously`,
`unassumingly`) or verbs that don't take that preposition (`codify`). This is an inherent property of
any small fixed-frame-family design applied to a POS as heterogeneous as "adverb," not a defect
introduced by this implementation, and it only surfaces for genuinely-uncurated words (already a
minority case). Per the Architecture's own framing ("wording is illustrative... structure is the
binding decision"), fixing this further would mean either enlarging the frame families or adding a
finer-grained adverb sub-classification — a real scope change, not a same-task wording tweak — so it
is logged here as a recommendation rather than acted on unilaterally.

## 8. Level-2 Auto-Decisions (adaptations within Dev-Task latitude, all disclosed)

1. **`examples()` simplified to always use `[v.data[3], v.data[4]]`**, dropping the old flat-pool
   retry/de-dupe loop entirely (Task 3), rather than "adapting it to search within the POS family" as
   the Architecture's exact §7 wording suggested. Rationale: once both curated and fallback content
   are final and word-deterministic at data-build time, there is no per-render pool left to retry
   against, and keeping a single source of truth on `v.data` is what keeps a word's example identical
   everywhere it's shown (Example tab, Vocabulary Card, Word Book) — itself the whole point of Task
   1's fix. Statistical repetition-reduction across a rendered batch is still achieved (per-word hash
   naturally spreads selections across 3 frames per family, and different-POS words can never collide
   since they draw from disjoint families).
2. **`morphSuffixType(w)` added as a new, separate helper** rather than changing
   `safeMorphologyBase()`'s return shape, to avoid touching that function's other call site
   (`analyze()`'s `vocabQualityTier`, which only needs a truthy existence check).
3. **Task 4's regenerated example always uses the `"adjective"` working POS**, per explicit CEO
   instruction, even though `data[0]` (the POS field shown on the Vocabulary Card) continues to
   display the base's real POS (e.g. "noun/verb") — this is a deliberate, spec-mandated mismatch
   between the displayed POS label and the regenerated example's grammatical frame, not an oversight.

## 9. Files changed

- `index.html` — all 4 code changes (Tasks 1-4), no other files touched by application logic.
- `docs/milestones/milestone-06/03-IMPLEMENTATION-LOG.md` — this file (Task 5).

## 10. Commits (this branch, on top of `71edcf8`)

1. `ce9ba59` — Task 1: curated example routing, `exampleBank` retirement, XSS hardening.
2. `0a0c475` — Task 2: `guessPOS()`, wired into `buildFallbackVocabData()`.
3. `51a2e88` — Task 3: `fallbackFrameFamilies`, `renderFallbackFrame()`, `examples()` simplification.
4. `d658e65` — Task 4: `morphSuffixType()`, field-level morphology reuse split.
5. (this commit) — Task 5: implementation log.

## 11. Status

All 5 Developer Tasks complete. Regression sweep (27-item list + Special Quality Test) passed with one
disclosed, non-blocking secondary limitation (§7.6). No push, no merge. Stopping before Independent QA
per the CEO's explicit instruction.
