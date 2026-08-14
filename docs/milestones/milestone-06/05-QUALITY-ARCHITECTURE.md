# Milestone 6B — Collocation-Aware Example Quality Architecture

Role: Software Architect (PM investigation + Architecture combined, per CEO's investigation brief).
Branch: `feature/example-sentence-quality` @ `9efdcf0` (stabilized intermediate baseline — verified
`git rev-parse HEAD` matches before this investigation began; working tree clean except pre-existing
untracked `.DS_Store`/`.claude/`). **No application code was modified during this investigation.**
Method: read the current source directly; live in-browser JS execution (local static server, no build
step) to empirically test hypotheses against the real curated data and the real `analyze()` selection
logic, not synthetic assumptions.

---

## 1. Current stabilized state at `9efdcf0`

The hot-fix (F-1, F-2, adverb split) is locked in and independently re-verified during this
investigation as still intact:

- Curated exact-form routing, `exampleBank` retirement, field-level morphology safety, Korean pairing,
  determinism, and XSS protection: unchanged and untouched by this investigation.
- Fallback POS resolution: 5-tier chain — exact curated → safe morphology base → **lexical reuse**
  (`lexicalPOSLookup()`, mines curated entries' own synonym/antonym fields) → suffix inference
  (`suffixPOS()`) → explicit `"unknown"` (never silently "noun").
- Fallback frame families: `noun` / `verb` (bare-pronoun-object, "it") / `adjective` /
  `adverbConnective` / `adverbManner`, selected via `resolveFallbackPOS()` + `isConnectiveAdverb()`.
- Quality on the CEO-directed re-test (23 genuinely-fallback words, §17.6 of `04-QA-REPORT.md`): 0%
  BAD, 0% systematic POS-role defects, 0% false/broken verb-complement patterns, **65% GOOD-or-
  EXCELLENT** — short of the 90% target, which is why this investigation was commissioned.

## 2. Why quality plateaued at 65%

Re-classified every non-GOOD/EXCELLENT result from the hot-fix's 23-word test sample (§17.6 of
`04-QA-REPORT.md`) against the CEO's failure-cause taxonomy:

| Word | Rating | Root cause |
|---|---|---|
| circumvent | ACCEPTABLE | missing POS (zero lexical votes, no suffix match) → honest "unknown" |
| harsh | ACCEPTABLE | missing POS → "unknown" |
| redundant | ACCEPTABLE | missing POS → "unknown" |
| cursory | ACCEPTABLE | missing POS → "unknown" |
| volatility | ACCEPTABLE | noun countability/relational-complement gap (wants "volatility *of X*", frame gives it bare) |
| tacitly | ACCEPTABLE | weak collocation (adverb-manner frame's verb "increased" doesn't suit this specific adverb well) |
| argue | AWKWARD | subject-selectional mismatch (verb needs an animate/rhetorical subject; the hash-selected frame's subject, "the new policy," doesn't fit) |
| permeate | AWKWARD | subject-selectional mismatch (verb needs a non-human/abstract subject like a smell or influence; "the team planned to permeate it" doesn't fit) |

**Quantified:** of the 8 non-GOOD+ cases, **4/8 (50%) are "missing POS"** (the safety mechanism
correctly refusing to guess — this is the system working as designed, not a quality defect, but it
caps at ACCEPTABLE by construction), **2/8 (25%) are verb subject-selectional mismatches** (a frame
picked a grammatical-but-ill-fitting subject for that specific verb), and **2/8 (25%) are weak-
collocation/relational-complement gaps** in the noun and adverb families (not wrong, just generic).

**Zero** instances trace to article/determiner issues (fixed in the hot-fix), connective-vs-manner
confusion (fixed), or a false/broken complement pattern (fixed). The remaining gap is entirely in two
categories: (a) words with literally no local data to resolve confidently, and (b) frame-subject/object
fit for the words that *do* resolve.

## 3. Existing local data available for reuse

Inspected every field of the curated records (`dictionary`, `supplementalVocab`, `broadVocabLexicon` —
1,115 entries total, each `[POS, EN definition, KO meaning, EN example, KO example, synonyms,
antonyms]`) for additional signal beyond what the hot-fix already uses:

- **POS, synonyms, antonyms** — already fully exploited by the hot-fix's tier-2 lexical POS index.
- **EN/KO definitions and examples of *other* curated words** — confirmed, by direct search, that the
  QA-cited "unknown"-bucket words (circumvent, harsh, redundant, cursory) appear **zero times** anywhere
  in any curated example or definition text. There is no hidden local signal about these specific words
  to recover — "unknown" is not a missed opportunity for these four, it is the only honest answer the
  app's current data can give.
- **New finding, empirically validated this investigation — curated examples as *borrowable templates*
  for synonym-resolved words:** when a word's POS came from tier 2 specifically via a **synonym** match
  (not antonym), that synonym's own real curated example sentence can be reused as a template with the
  target word substituted in, instead of a generic frame. Tested directly:
  - `argue` ← synonym `contend` → *"Some economists contend that the policy will slow growth."* →
    substituted: *"Some economists **argue** that the policy will slow growth."* — **EXCELLENT**,
    markedly better than the generic frame's AWKWARD result for this word.
  - `vague` ← synonym `ambiguous` → *"The ambiguous wording caused two groups to interpret the rule
    differently."* → substituted: *"The **vague** wording caused two groups to interpret the rule
    differently."* — **EXCELLENT**.
  - **Antonym substitution tested and found unsafe, confirming the CEO's caution not to assume
    interchangeability:** `prevent` ← antonym `induce` → *"Sleep deprivation can induce difficulty
    concentrating."* → substituted: *"Sleep deprivation can **prevent** difficulty concentrating."* —
    this **inverts the causal direction** of the original sentence and would teach a confusing,
    backwards impression of the word. A second antonym case (`insular` ← antonym `cosmopolitan`)
    happened to substitute acceptably, but the inconsistency itself is the finding: antonym pairs
    describing a scalar *property* (insular/cosmopolitan) tolerate substitution far better than antonym
    pairs describing *causal direction* (prevent/induce), and distinguishing those reliably would
    require semantic classification this investigation was not asked to build. **Safety rule: template-
    borrowing must be restricted to synonym-sourced matches only, never antonym-sourced matches** — POS
    voting can keep using both (POS doesn't invert under negation), but sentence-template borrowing
    must not.
  - **Coverage limit:** words with *zero* curated synonym or antonym relation at all (confirmed for
    `permeate`, `galvanize` in this sample) get no benefit from this technique — it only helps the
    subset of fallback words that already resolve via tier 2 with at least one synonym-sourced match.

No other reusable local data source was found (no separate phrase/collocation dataset exists in the
app beyond the idiom/phrase bank used elsewhere, which is keyed to fixed idiomatic phrases, not POS-
tagged single words, and was not designed for this purpose).

## 4. Real-world fallback frequency (coverage study)

Ran 7 realistic passages — 4 typical SAT-register passages (science, history, literature, social
science, ~90-110 words each), one short passage (~27 words), one deliberately dense/advanced-register
literary-criticism passage, and one moderately-challenging mixed-register passage — through the
**actual** `analyze()` selection logic (dictionary-exact-match first, then the same `vocabQualityTier`-
sorted, `length>=7`, non-proper-noun padding to 12 the real app uses), not a hypothetical approximation.

| Passage | Total vocab shown | Curated | Fallback (known POS) | Fallback (unknown) |
|---|---|---|---|---|
| Science (109 words) | 12 | 12 (100%) | 0 | 0 |
| History (97 words) | 12 | 12 (100%) | 0 | 0 |
| Literature (88 words) | 12 | 12 (100%) | 0 | 0 |
| Social science (94 words) | 12 | 12 (100%) | 0 | 0 |
| Mixed register, moderately challenging (73 words) | 12 | 12 (100%) | 0 | 0 |
| Short (27 words) | 8 | 3 (38%) | 2 (25%) | 3 (38%) |
| Deliberately dense/advanced register (85 words) | 12 | 7 (58%) | 4 (33%) | 1 (8%) |

**Finding: in 5 of 7 realistic passages — including every passage at typical SAT reading-passage
length (~90-110 words) across four different subject registers — the fallback engine was never
exercised at all.** The 1,115-word curated lexicon already covers a strong majority of real academic
vocabulary that appears in normally-constructed passages. Fallback only becomes materially relevant in
two specific scenarios: **(a) unusually short passages**, where there simply aren't 12 candidate words
of any tier to fill the list, and **(b) unusually dense/rare-vocabulary passages** that deliberately use
less common academic words (more characteristic of the hardest official SAT passages than typical
practice material).

This is the single most important input to the option comparison below: **the practical, everyday
impact of fallback-quality problems on real students is smaller than the QA testing (which necessarily
used deliberately-obscure words to stress-test the system) might suggest.** That does not make the
remaining 35% gap unimportant — short passages and hard passages are real, valid use cases — but it
strongly shapes which architecture gives the best return for its cost.

## 5. Option A — Local collocation-aware engine (existing data only)

**Description:** keep the current hot-fix architecture and add the synonym-template-borrowing
mechanism validated in §3, restricted to synonym-sourced (never antonym-sourced) matches.

- **Achievable quality:** estimated to lift the GOOD-or-EXCELLENT rate meaningfully for the subset of
  words that resolve via a synonym match (validated: 2 of the hot-fix's own AWKWARD cases would become
  EXCELLENT), but **cannot help the "missing POS" 50% of the remaining gap** — those words have zero
  curated relation of any kind, synonym or antonym, so there is nothing to borrow. Realistic estimate:
  roughly **70-78% GOOD-or-EXCELLENT** on a QA-style adversarial word sample, still short of 90%.
- **Complexity:** low-moderate. One new lookup path (synonym-sourced-entry → example → substitute),
  reusing the same `lexicalPOSLookup()` infrastructure and safety-rule discipline already established.
- **Coverage:** bounded by how many of the 1,115 curated entries' synonym fields happen to include the
  target word — a property of the existing data, not extensible without curating more entries (which is
  Option B).
- **Offline capability:** full — no network dependency, same as today.
- **Maintenance burden:** low. No new data to keep in sync; purely derived from the existing curated
  lexicon at runtime (same lazy-memoized pattern as the tier-2 index).

## 6. Option B — Curated fallback expansion

**Description:** instead of generating open-ended examples, expand `broadVocabLexicon` (or a new
similarly-structured table) with real, hand-written entries for the specific words that most often
actually reach the fallback path in realistic passages.

- **How many fallback words actually occur in realistic passages:** per §4, the honest answer is "not
  many, in typical passages — but a meaningful, recurring, and almost certainly *predictable* set in
  short and advanced-register passages." The specific words seen this investigation (circumspection,
  documentation, argumentation, evolutionary, decision, reckless, economy, praised, and the 20+
  independently-chosen QA/hot-fix words) are unremarkable, common SAT/AP-register vocabulary — exactly
  the kind of word a systematic gap-filling pass (run this investigation's own coverage-study method
  across a larger corpus of real SAT-style passages, rank uncurated words by recurrence, curate the top
  N) would predictably and efficiently close.
- **Would adding several hundred curated examples solve most practical cases:** very likely yes, based
  on the coverage study's shape — the existing 1,115-word lexicon already achieves 100% coverage on
  4 of 4 typical-length passages tested; the gap is concentrated in shorter/denser text, which likely
  draws from a smaller, more learnable pool of higher-register academic words rather than an
  unbounded long tail.
- **Maintenance cost:** one-time authoring effort (bounded, estimable — e.g. 200-300 entries at the
  same quality bar as the existing 1,115), following the exact existing data shape and existing
  authoring convention (already proven at scale for 1,115 entries). No runtime cost, no new code
  paths, zero risk to anything already working.
- **Quality advantage:** the single highest per-word quality option available — a real, hand-written,
  POS-correct, natural example is definitionally better than any generated frame, at the same
  authoring quality bar the app already trusts for 1,115 words.
- **Ceiling:** cannot reach 100% coverage of all possible English words a student might encounter — genuinely rare/unusual vocabulary in an unusually hard passage will still fall through to fallback
  no matter how large the curated set grows. This is a coverage-vs-effort curve with diminishing
  returns, not a complete solution on its own.

## 7. Option C — Optional AI-assisted fallback (existing Gemini provider only)

**Description:** for words that reach the fallback path with local confidence still low (i.e., resolve
to `"unknown"`, or possibly the weaker ACCEPTABLE-tier cases), optionally call the existing Gemini
summary/vocab-context provider (already wired into `providerRegistry`/`aiRouter`, already gated on a
present API key, already fails closed) to generate a POS-correct example, with the local frame/unknown
message as the guaranteed fallback if the call fails or no key is present.

- **Quality:** highest potential per-word quality for genuinely novel words no local data can inform —
  an LLM can correctly infer both POS and a natural collocation for essentially any real English word,
  which neither Option A nor Option B can guarantee for the true long tail.
- **Latency:** a real cost — one or more network round-trips per uncurated word, on the critical path of
  passage analysis (which is otherwise fully local/instant). Would need to be async/deferred (e.g.,
  show the local fallback immediately, upgrade in place if/when the AI result returns) to avoid
  blocking the Example tab, adding real UI-state complexity not present today.
- **Cost:** per-call API cost. Per §4, fallback is rare in typical passages, so aggregate cost would
  likely be low in practice — but this is exactly the kind of usage-pattern claim that should be
  measured (e.g., via `providerUsageThisRun` tracking, which already exists) before being relied on,
  not assumed.
- **Privacy:** the target word (and possibly its context sentence) would leave the device to a third-
  party API — a real, if narrow, difference from every other fallback path in this milestone, which is
  fully local. Needs explicit user-facing disclosure, consistent with how the existing Gemini features
  are already gated behind an explicit API-key opt-in.
- **Offline degradation:** must degrade gracefully to the local "unknown"/generic-frame result with no
  key or no network — already the established pattern for every other Gemini integration in this app
  (render-time key check, fails closed, matches the existing "Milestone 4 Dev Task 7" pattern cited in
  the codebase's own comments).
- **API-key requirement:** yes — this feature would simply not activate for guest/no-key sessions,
  which per the app's own auth model is a normal, already-accepted degradation path.
- **Caching strategy:** should cache by word (not by sentence/context) in the same spirit as
  `translationCache`, since a fallback word's example is already deterministic-per-word by design in
  this milestone — an AI-generated result should be cached (in-memory for the session at minimum;
  `vocabularyProgress`/IndexedDB for persistence across sessions would need its own small schema
  addition, which is explicitly out of this investigation's scope to decide unilaterally).
- **Deterministic behavior:** AI generation is not naturally deterministic; if adopted, would need a
  caching layer between the app and the API to preserve this milestone's explicit "same word → same
  example every time" acceptance criterion, or determinism would need to be re-scoped as "same within a
  session" only — a real design decision requiring CEO sign-off, not something to decide here.
- **Failure handling:** must be indistinguishable in effect from "no AI available" — always fall back to
  the local result, never show an error state or block the Example tab.
- **No new provider:** confirmed compliant with this constraint — reuses the existing Gemini adapter
  already registered in `providerRegistry`, no new registration.

## 8. Recommended architecture

**A staged combination, not a single option — evaluated in the order the CEO's own fallback-precedence
principle already implies (curated → local → optional AI → honest limitation):**

1. **Ship Option A's synonym-template-borrowing enhancement first.** It is local, fully offline,
   deterministic, low-complexity, zero new risk to the curated path, and directly validated on real
   failure cases from this milestone's own QA data. This should be the immediate next Developer Task
   regardless of what else is approved.
2. **Run Option B's coverage-study-driven curated expansion as the primary lever for closing the
   remaining gap.** §4's finding — that fallback is concentrated in short and advanced-register
   passages, not spread evenly across all vocabulary — means a bounded, well-targeted authoring effort
   (guided by re-running the coverage-study method across a larger real-passage corpus to rank
   recurring uncurated words) is likely to close most of the remaining 35% at zero runtime cost, zero
   latency, zero privacy exposure, and zero new failure modes — the same reason the original 1,115-word
   lexicon already achieves 100% coverage on typical passages today.
3. **Recommend Option C as an explicitly optional, gated enhancement for the true residual long tail**
   (words with zero curated relation and zero curated expansion coverage) — not required to reach the
   90% target if Options A+B prove sufficient in practice, but available as a final, opt-in layer for
   the CEO to authorize separately if measurement after A+B shows a persistent gap. Given §4's frequency
   data, this layer would trigger rarely, which meaningfully reduces its cost/latency/privacy downsides
   in aggregate even though those downsides are real per-call.

**This order matters:** A and B carry no downside risk and directly address the two largest quantified
failure causes (§2) — subject-selectional mismatch (A) and missing-POS on common, recurring words (B).
C should not be built first; it is the most expensive and highest-risk option and, per §4, would be
solving a smaller problem than it appears to from adversarial QA testing alone.

## 9. Expected achievable quality

| Stage | Estimated GOOD-or-EXCELLENT (adversarial QA-style sample) | Estimated real-passage impact (per §4) |
|---|---|---|
| Current (`9efdcf0`) | 65% | Already ~100% in typical passages; gap only visible in short/dense passages |
| + Option A enhancement | ~70-78% | Marginal — most typical passages already unaffected |
| + Option B (targeted expansion) | ~85-92%+ | Directly shrinks how often fallback triggers at all in short/dense passages |
| + Option C (residual long tail) | ~93-97%+ | Closes the last gap for genuinely novel words, rarely triggered |

These are reasoned estimates from the failure-taxonomy quantification (§2) and the validated
synonym-borrowing spot-tests (§3), not a re-run of the full 23-word suite against unbuilt code — the
90% target is realistically reachable via A+B alone without needing C, given the taxonomy shows only
25% of the remaining gap (subject-selectional mismatch) is a pure frame-quality issue or that A can
address, while the other 50% (missing POS) is best addressed by B's targeted curation rather than by
further local cleverness once the zero-local-data cases are confirmed to have literally nothing to
reuse.

## 10. Cost/privacy/offline tradeoffs summary

| | Option A | Option B | Option C |
|---|---|---|---|
| Runtime cost | negligible (one more lookup) | zero (pure data) | per-call API cost |
| Latency | none | none | real, needs async handling |
| Privacy | none (fully local) | none (fully local) | word/context sent to Gemini |
| Offline | full | full | degrades to local result |
| Determinism | preserved | preserved | needs a caching layer to preserve |
| Authoring/maintenance | none (derived) | one-time bounded authoring effort | ongoing (prompt/cache upkeep) |
| New risk to curated path | none | none | none (if scoped as fallback-only, fails closed) |

## 11. Developer Tasks (for the next approved Development phase — not authorized to start yet)

1. **Option A:** implement synonym-sourced template borrowing in `buildFallbackVocabData()`'s tier-2
   path — when `lexicalPOSLookup()` resolves via a synonym-field match specifically, reuse that curated
   entry's own `data[3]`/`data[4]` as a template (word-substituted) instead of the generic frame family;
   fall through to the existing generic frame when the match was antonym-only or no match exists.
2. **Coverage-study tooling:** formalize this investigation's coverage-study script into a reusable
   internal tool (not user-facing) for running a larger corpus of realistic passages and ranking
   recurring uncurated words by frequency — the direct input Option B's authoring pass needs.
3. **Option B:** author and add the top-N recurring words identified by Task 2 to `broadVocabLexicon`
   (or a new table, Architect's call at that time), at the existing 7-field curated-entry quality bar.
4. **(Conditional on separate CEO authorization)** Option C: async, cached, fails-closed optional
   Gemini-assisted fallback for words still unresolved after Tasks 1 and 3, with explicit determinism
   and privacy handling per §7.
5. **Regression + quality re-test:** re-run the full 27-item regression list, the Special/QA word sets,
   and a fresh independently-chosen unseen-word set against the ≥90% bar, exactly as done for the
   `9efdcf0` hot-fix.

## 12. Acceptance criteria

- ≥90% GOOD-or-EXCELLENT on an independently-selected, genuinely-uncurated word sample (same rubric and
  independence standard as the QA report and hot-fix re-test).
- 0 systematic POS-role defects, 0 clearly false verb-complement patterns, 0 systematic adverb misuse,
  0 fabricated collocation presented as authoritative (all already true at `9efdcf0`; must remain true).
- Every item in §1's "locked" list remains unchanged and independently re-verifiable (curated routing,
  `exampleBank` retirement, morphology safety, unknown POS ≠ noun, lexical POS reuse, connective/manner
  split, Korean pairing, determinism, XSS protection, and all unrelated app functionality).
- If Option C is authorized: offline/no-key sessions must produce identical behavior to Option A+B alone
  (no error states, no blocked UI, no silent quality regression when AI is unavailable).

## 13. CEO decisions required before Development can proceed

1. **Approve Option A's synonym-template-borrowing enhancement** (low-risk, recommended to proceed
   regardless of the other decisions below).
2. **Approve Option B's scope**: authorize running the coverage-study tool across a larger passage
   corpus and curating the top-N recurring words — and set N (this investigation suggests an initial
   200-300 word batch as a reasonable, boundable first pass, re-measured before deciding whether a
   second batch is warranted).
3. **Decide whether Option C is in scope for this milestone at all**, or deferred to a separate,
   explicitly-scoped future milestone given its cost/latency/privacy/determinism tradeoffs (§7, §10) —
   if approved, also decide the determinism question (cache-for-true-determinism vs. accept
   session-level-only determinism for AI-assisted words) and the privacy-disclosure requirement.
4. **Confirm the 90% target's evaluation basis**: an adversarial, independently-selected word sample
   (as QA has used throughout) remains the fairest test given §4's finding that typical passages rarely
   exercise fallback at all — the CEO should confirm this is still the intended measurement, since a
   real-passage-weighted measurement would already show a much higher effective quality rate today.

---

**STOP after Architecture. No application code was modified. No push, no merge.**

## 14. Addendum — Development phase complete

CEO approved this Architecture (decisions: synonym-only borrowing approved, antonym borrowing rejected,
targeted curated expansion as primary lever, Gemini deferred). Tasks 1-5 executed on top of this
Architecture; full results, before/after numbers, and the two newly-discovered residual failure
categories (person/behavior-describing adjectives; scope/degree/certainty adverbs) are recorded in
`docs/milestones/milestone-06/06-IMPLEMENTATION-LOG.md`. This document's own findings (§1-13) are left
unmodified above — nothing here is superseded, only extended.
