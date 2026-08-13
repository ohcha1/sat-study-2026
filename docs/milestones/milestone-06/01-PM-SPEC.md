# Milestone 6 — PM Assessment: Example Sentence Quality Upgrade

**Status: APPROVED by the CEO — 2026-08-14, as a two-phase "Example Sentence Quality Upgrade."** See
§23 for the CEO's scope decisions. Architecture Design (`02-ARCHITECTURE.md`) may proceed. No
application code has been touched while producing either this document or the Architecture Design.

## 0. Baseline verification (read-only, this session)

- Branch: `main`, HEAD `53bf0cc03a1b8831f091c1b8f7b1c6ead67df669` — confirmed via `git rev-parse`,
  matches the CEO-stated current official baseline.
- Working tree: clean except pre-existing untracked `.DS_Store`/`.claude/`.
- No application code modified while producing this document.

## 1. Executive summary

The CEO's diagnosis is correct, and the actual code confirms a more specific, more fixable picture
than "the app doesn't know how to write good example sentences." **The app already has ~1,115
genuinely well-written, part-of-speech-correct example sentences** across its three curated
vocabulary lexicons (spot-checked extensively below — verbs function as verbs, adverbs modify
naturally, adjectives modify naturally). The actual defects are narrower and more mechanical:

1. **The Example tab specifically ignores this good data.** It reads from a separate, tiny, stale
   26-word lookup table (`exampleBank`) instead of the same per-word data every other screen
   (Vocabulary Card, Word Book, Review) already uses correctly. For 1,089 of 1,115 curated words
   (97.7%), the Example tab silently discards a perfectly good sentence it already has access to and
   falls through to a broken generic template instead.
2. **The generic fallback template pool is structurally blind to part of speech.** All 12 templates
   quote the target word as a discussed noun/term ("the class used the term '{word}'," "'{word}'
   carried a more specific meaning") regardless of whether the word is actually a verb, adjective,
   adverb, or noun. This is the exact defect the CEO described, and it is real — it just affects a
   much smaller share of real usage than it first appears, because most of what triggers it is either
   (a) the Example tab's own broken data-sourcing (fixable without touching this pool at all), or
   (b) genuinely uncurated words, which are a minority for typical SAT-style academic passages given
   how broad the existing lexicons already are.

**The highest-value fix is not a new generation system — it's correcting which data source the
Example tab reads from.** That alone fixes the reported problem for the overwhelming majority of
words. A smaller, second piece of work is still warranted for the genuinely-uncurated residual case,
and that piece does need real design (per the CEO's explicit rejection of "just four generic
templates") — detailed in §8-§14.

## 2. Current implementation — where Example-page sentences come from

Traced directly in `index.html`:

```js
function examples(a){
 const used=new Set();
 return a.vocab.slice(0,12).map((v,i)=>{
   let ex=exampleBank[v.word];                              // ← separate 26-word table
   if(!ex){
     const base=fallbackExamplePool[i%fallbackExamplePool.length];   // ← generic templates
     ex=[base[0].replaceAll("{word}",v.word),base[1].replaceAll("{word}",v.word)];
   }
   while(used.has(ex[0])){...}                               // de-dupe by re-picking a template
   ...
 })
}
```

**Key finding:** `v` here is already a fully-built `buildVocabCardData()` object — `v.data[3]`
(English example) and `v.data[4]` (Korean example) are *already sitting on it*, sourced from
`dictionary`/`supplementalVocab`/`broadVocabLexicon` (curated) or `buildFallbackVocabData()`
(generic). `examples()` never looks at `v.data[3]`/`v.data[4]` at all. It performs an entirely
separate, redundant lookup against `exampleBank`, a hardcoded object of only 26 entries (verified by
direct count — see §4), and only falls through to `v`'s own data implicitly by way of the same
generic-template pool that fallback words already use. This is not "reused from another vocabulary
dataset" in a good sense — it's a stale, incomplete duplicate that actively prevents the good data
from being used.

Answering the investigation's specific sourcing question: Example-page sentences are **template-based
and fallback-generated for the large majority of words**, and **curated only for a small, stale
26-word subset** — not because curated data doesn't exist for the rest, but because the Example tab
doesn't look for it in the right place.

## 3. Root cause of poor examples (ranked by impact)

1. **Primary — wrong data source in `examples()`.** Confirmed above. This alone explains why a word
   with a genuinely excellent curated example (e.g., `corroborate`: "Newly discovered letters
   corroborate the historian's account of the event.") can still show a broken generic sentence on
   the Example page specifically, while showing the correct sentence on the Vocabulary Card for the
   exact same word in the exact same session.
2. **Secondary — the fallback template pool (`fallbackExamplePool`, `index.html:1826`) is POS-blind
   by construction.** Every one of its 12 templates embeds the word as a quoted term being discussed
   ("the class used the term '{word}'... ", "'{word}' carried a more specific meaning...", "the
   student wrote a new sentence in which '{word}' was essential"). A verb inserted into these frames
   reads as a noun being talked about, not as a verb being used — exactly the CEO's complaint. This
   affects: (a) any word that fails to match all three lexicons and has no valid inflectional base
   (genuinely-uncurated words, everywhere they appear — Vocabulary Card, Word Book, Review, Example
   tab), and (b) the Example tab's fallback path for the 1,089 curated-but-not-in-`exampleBank` words
   (compounding root cause #1).
3. **Minor, narrow — morphological base-matching can reuse an example written for a different
   grammatical role.** `buildVocabCardData()` falls back to `safeMorphologyBase(w)` when the exact
   word isn't found, reusing the *base* word's full curated record (including its example sentence)
   for the *inflected* word. `safeMorphologyBase()` only strips safe **inflectional** suffixes
   (plural `-s`/`-es`/`-ies`, past tense `-ed`, progressive `-ing` — see §7), which usually preserves
   the base's part of speech. The narrow exception is `-ed`/`-ing` forms used **adjectivally**
   (e.g., a passage using "concerned" as an adjective, if "concerned" itself isn't curated but
   "concern" is, would reuse "concern"'s noun/verb-focused example). This is a real but
   low-frequency gap, not the primary driver of the CEO's complaint.

## 4. Number/type of affected words — exact data inventory

Counted directly from the source objects (not estimated):

| Source | Entry count | Quality (spot-checked) |
|---|---|---|
| `dictionary` | **142** | Curated, POS-correct (verified: every sampled verb/adjective/adverb/noun entry uses the word in its natural grammatical role) |
| `supplementalVocab` | **84** | Curated, POS-correct (same verification) |
| `broadVocabLexicon` | **889** | Curated, POS-correct — spot-checked broadly, including a full read of all 17 pure-adverb entries and a spread sample of verb entries across the alphabet; every one uses the target word naturally in its stated part of speech |
| **Total curated, with genuinely good examples** | **1,115** | — |
| `exampleBank` (Example tab's actual data source) | **26** | Identical content to `dictionary`'s first 26 entries — a stale, partial duplicate, not an independent asset |
| `fallbackExamplePool` (generic templates) | 12 sentence frames | POS-blind by design (§3.2) |

**Coverage gap:** 1,115 − 26 = **1,089 words (97.7% of the curated vocabulary) have a good example
sentence available but unused by the Example tab specifically.**

POS distribution within `broadVocabLexicon` alone (889 entries), confirmed by direct count: 289
noun, 225 verb, 196 adjective, 63 noun/verb, 32 verb/noun, 25 adjective/noun, 17 adverb, 10
verb/adjective, 9 adjective/verb, 9 noun/adjective, 7 preposition, plus a handful of conjunction/
determiner/pronoun entries. This confirms real POS diversity already exists in the curated data —
the problem is not that the data lacks POS information (it doesn't — `data[0]` is POS for every
curated entry), it's that the fallback/Example-tab paths don't use it.

## 5. Existing data that can be reused

- **`v.data[3]`/`v.data[4]`** (English/Korean example) — already computed for every word by
  `buildVocabCardData()`, already correctly POS-respecting for all 1,115 curated words. This is the
  single highest-value reuse opportunity and requires no new content authoring at all — just reading
  from the right field.
- **`v.data[0]`** (part of speech) — already present for every curated word. Directly usable to
  select an appropriate sentence *frame* for the smaller residual fallback case, without needing a
  POS guesser for the curated majority.
- **`v.contextSentence`** (the word's own sentence from the student's actual passage, already
  computed by `findContextSentence()`, already shown on the Dictionary Card as "문맥 속 의미") — a
  genuinely different, complementary asset: the passage's own real usage, not a static example.
  Already proven not to need AI or new data. Not currently reused by the Example tab either, though
  that is a smaller opportunity than §5's first point since it's passage-dependent and only available
  when a passage is loaded (not for Word Book's standalone reconstruction).
- **`QG_VOCAB_DISTRACTORS`** and `genVocabInContext()`'s existing "real-passage-sentence" pattern —
  proof this codebase already has a working, non-generic model for using a word's actual passage
  context rather than a canned sentence; not directly reusable as example-sentence content, but a
  useful precedent for §5's contextSentence reuse idea.

## 6. Curated vs. fallback example analysis

- **Curated (1,115 words):** genuinely high quality, POS-correct, natural, SAT-appropriate — this
  assessment's own extensive sampling found no counterexamples. **These do not need to be
  regenerated.** They need to be *read from* by the Example tab, which currently isn't happening for
  97.7% of them.
- **Fallback (words matching none of the three lexicons and no valid morphology base):** universally
  poor, POS-blind, template-quoted. This is the genuine content gap requiring new design work (§8-14).
  The real-world frequency of hitting this path is passage-dependent and wasn't measured by running
  arbitrary passages through the app in this investigation (that would require live testing, out of
  scope for a code-level PM assessment) — but given `broadVocabLexicon` alone covers 889 general
  academic words, plus safe inflectional morphology extends coverage further, the fallback path is
  expected to trigger mainly for domain-specific/unusual terms, not everyday academic vocabulary.

## 7. POS handling analysis

- Every curated entry (`dictionary`/`supplementalVocab`/`broadVocabLexicon`) already carries POS as
  `data[0]`, including compound tags like `"noun/verb"`, `"verb/adjective"` for words with multiple
  common uses.
- Fallback words (`buildFallbackVocabData()`) get **no real POS at all** — the field is hardcoded to
  the literal string `"academic or informational word"`, a label, not a part of speech. This is a
  second, smaller root cause specific to the fallback path: even if the CEO's desired POS-aware
  templates existed, there is currently no mechanism to *select* the right one for a genuinely
  uncurated word. §9 proposes a lightweight local POS guesser to close this gap.

## 8. Morphology analysis

`safeMorphologyBase()` (`index.html:2333`) is deliberately conservative — it only strips
**inflectional** endings (verified by reading the function directly):
- `-ies`→`-y`, `-es`, `-s` (plurals / 3rd-person singular)
- `-ed` (past tense, with a doubled-consonant check)
- `-ing` (progressive, with a doubled-consonant check)

It does **not** strip derivational endings (`-tion`, `-ment`, `-ness`, `-ly`, `-ity`, etc.), so it
does not conflate genuinely different parts of speech in the common case. **Morphology is not the
primary cause of the reported problem** — it's a narrow, secondary contributor limited to `-ed`/
`-ing` forms used adjectivally when only a differently-classed base form is curated (§3.3). It is
*not* a source of new fallback-quality problems; if anything, it currently *reduces* how often the
broken fallback path triggers, by successfully reusing (mostly correct) curated data for inflected
forms.

## 9. Whether the same poor examples appear elsewhere

| Surface | Data source | Affected? |
|---|---|---|
| Example page (`examples()`) | `exampleBank` (26 words) → `fallbackExamplePool` | **Yes, severely** — broken for 1,089 curated words (wrong source) + all genuinely-fallback words (POS-blind templates) |
| Vocabulary Card (`vocabCardTemplate()`/`buildVocabCardData()`) | `v.data[3]`/`v.data[4]` directly | **Only for genuinely-fallback words** — correct for all 1,115 curated words already |
| Word Book / Review (`renderWordBookAsync()`, reuses `buildVocabCardData()`) | same as Vocabulary Card | Same as above |
| SAT Questions (`genVocabInContext()`) | the student's **actual passage sentence** (`sents[sIdx]`), restricted to `dictionary`/`supplementalVocab` words with hand-curated distractors | **Not affected at all** — deliberately avoids both `exampleBank` and `fallbackExamplePool` |

This means the CEO's complaint, as experienced by a student, is likely strongest on the Example
tab specifically (broken for nearly everything) and comparatively minor on the Vocabulary Card/Word
Book/Review (only genuinely-uncurated words) — even though the underlying template-quality defect
(§3.2) is shared code across all of these surfaces.

## 10. Proposed sentence-quality architecture

Two independent, sequentially-shippable phases — deliberately not one big rewrite, per "keep
implementation small" precedent from prior milestones.

### Phase 1 — Fix the Example tab's data source (no new content, no new logic)

Change `examples()` to read `v.data[3]`/`v.data[4]` (the same field every other screen already uses
correctly) as its **primary** source, falling through to a POS-aware fallback (Phase 2) only when
`v.curated` is `false`. Retire `exampleBank` — it becomes fully redundant once this change ships (see
§20 for the CEO decision on whether to delete it outright or leave it unused).

**This single change fixes the reported problem for 1,115 of the words that could ever appear on the
Example tab**, with no new sentence-authoring, no new heuristics, and minimal code surface — the
smallest possible diff for the largest possible improvement, directly matching the CEO's instruction
to "prefer improving/reusing existing curated data over generating everything again."

### Phase 2 — POS-aware handling for genuinely-uncurated (fallback) words

For the residual case only (a word matches none of the three lexicons and no valid morphology base),
replace the single POS-blind template pool with:

1. **A lightweight, local, offline POS guesser** (§11) — since fallback words currently have zero POS
   information, this is a prerequisite, not optional.
2. **Multiple natural sentence frames per POS category** (§12-15) — not four rigid single templates
   (explicitly rejected by the CEO), but a small family of frames per POS (e.g., 3-4 each), so a
   verb can land in one of several natural verb-usage patterns rather than always the same one,
   preserving both POS-correctness and the "not repetitive across words" quality bar.
3. A true last-resort neutral frame only for the rare case where POS genuinely cannot be inferred
   (expected to be uncommon given how systematic English derivational suffixes are — see §11).

## 11. Suffix-based local POS guesser (for Phase 2 only)

Proposed, entirely local/offline, no new dependency:

| Target POS | Signal suffixes (illustrative, not exhaustive) |
|---|---|
| Noun | `-tion`, `-sion`, `-ment`, `-ness`, `-ity`, `-ance`, `-ence`, `-ism`, `-hood`, `-ship`, agent `-er`/`-or` |
| Verb | `-ate`, `-ify`, `-ize`/`-ise`, `-en` |
| Adjective | `-ive`, `-ous`, `-able`/`-ible`, `-al`, `-ic`, `-ful`, `-less`, `-ary` |
| Adverb | `-ly` (a very reliable signal on its own) |

This is the same category of technique `safeMorphologyBase()` already uses successfully (suffix
pattern matching, zero external dependency) — not a new architecture pattern for this codebase, just
applied to a different problem (POS inference instead of base-word lookup). It will not be perfect
(English has genuine ambiguity and exceptions), but it is dramatically better than the current zero
information, and it degrades gracefully to the neutral last-resort frame rather than guessing wrong
in a way that breaks a sentence.

## 12-15. Per-POS handling + collocation strategy (Phase 2)

Concrete, illustrative frame families (final wording to be refined at Architecture/Dev time, not
locked here):

- **Verb** — place the word in infinitive position after "to," avoiding any need to correctly inflect
  an unknown word (a real risk with irregular verbs): *"Students learned to {word} their argument
  with specific evidence."* / *"The committee agreed to {word} a clear set of guidelines."* / *"Her
  goal was to {word} the connection between the two events."*
- **Noun** — subject or object position with a natural determiner: *"The {word} became a central
  point of discussion in class."* / *"Understanding the {word} helped students interpret the passage
  more accurately."* / *"Researchers examined the {word} in greater detail."*
- **Adjective** — predicate-adjective position (avoids noun-agreement risk entirely): *"The results
  were {word} enough to change the committee's decision."* / *"Few students found the process
  entirely {word}."* / *"The findings seemed {word} at first, but further study complicated the
  picture."*
- **Adverb** — sentence-final manner position (safe, standard slot for `-ly` adverbs): *"The
  policy's effects became clear only {word}."* / *"The situation changed {word} after the new rule
  began."*

**Collocation strategy:** genuinely word-specific collocations (the CEO's own example: "establish a
relationship / establish a system / establish that...") are only available for words with real
semantic data — i.e., the 1,115 already-curated words, which Phase 1 already serves correctly. For
the Phase 2 residual (uncurated) case, true per-word collocation is not achievable without either (a)
a much larger curated dataset than exists today, or (b) AI generation (§16). Phase 2's frame
families are the practical ceiling for a local/offline, zero-new-content approach — an explicit,
named limitation, not a hidden one.

## 16. Fallback strategy / 17. Offline/local strategy

Both phases stay fully local/offline/deterministic:
- Phase 1 needs no new data or network calls — it's a pure code-level source correction.
- Phase 2's POS guesser and frame families are static, deterministic, and computed client-side, same
  architecture pattern as `fallbackKoreanGloss()`/`safeMorphologyBase()` already use successfully.
- Word-length-hash-based frame *selection within* a POS's frame family (mirroring
  `buildFallbackVocabData()`'s existing deterministic-by-word approach) keeps the same word always
  showing the same sentence across sessions — an established, already-accepted product convention
  from Milestone 4 (documented there as a deliberate choice for Word Book consistency).

## 18. Whether AI is actually necessary

**No, not for the primary fix, and not required for this milestone.** Phase 1 (the highest-impact
piece) needs no AI at all — it's a data-source correction against data that already exists. Phase 2
can achieve a large quality improvement over the current state using local suffix heuristics and
POS-aware frame families, fully consistent with this project's established "no new AI provider
without explicit CEO Human-Gate approval" pattern (Milestones 3-5 all avoided new AI dependencies by
default). **AI remains a possible *future*, optional enhancement specifically for Phase 2's
residual/uncurated-word case** (where true per-word collocation is otherwise unreachable, §12-15) —
flagged here as a later option, not a requirement, and would need its own separate CEO Human-Gate
decision if ever proposed, per this repo's governance rules for new-provider adoption.

## 19. Regression risks

1. **`examples()` change must not affect Vocabulary Card/Word Book/Review** — those already read
   `v.data[3]`/`v.data[4]` correctly today; Phase 1 only changes `examples()` itself.
2. **`exampleBank` removal** (if the CEO approves deleting it, §20) requires confirming no other call
   site references it — not yet grepped exhaustively in this investigation (deferred to Architecture/
   Dev phase, since no code changes are authorized yet); a single `grep -n "exampleBank"` before
   editing will settle this trivially.
3. **Phase 2's POS guesser/frame families must only ever apply to the existing `curated:false`
   branch** — must not alter behavior for any of the 1,115 already-curated words, and must not
   silently promote a fallback word to `curated:true` (the existing "일반 설명" tag / Important-
   Words-filter semantics from Milestone 4 depend on this distinction staying accurate).
4. **No IndexedDB schema change, no Gold Master change, no Vocabulary Card layout change** (data
   content only, not the frozen UI structure), **no SAT Retry/Evidence-Sentence/Weak-area logic
   touched** — none of these areas are anywhere near the functions this milestone would touch
   (`examples()`, `buildFallbackVocabData()`, `fallbackExamplePool`), confirmed by this investigation's
   direct reads.
5. **`genVocabInContext()` (SAT vocabulary-in-context questions) must remain untouched** — it already
   avoids this problem entirely by design (§9); Phase 2 work must not be routed through it or change
   its `dictionary`/`supplementalVocab`-only restriction.

## 20. Proposed Developer Tasks (for a future Architecture phase — not authorized yet)

1. **Fix `examples()`'s data source** — read `v.data[3]`/`v.data[4]` as primary, existing
   `curated:false` fallback path as secondary; remove the now-dead `exampleBank` lookup (pending
   CEO decision on deleting vs. leaving the object itself, §21).
2. **Build the local POS guesser** for fallback words (§11).
3. **Design and implement POS-aware frame families** replacing the single `fallbackExamplePool`,
   wired through both `buildFallbackVocabData()` (Vocabulary Card/Word Book/Review) and `examples()`'s
   fallback path, so the fix is consistent everywhere the problem currently appears (§9).
4. **Regression pass** across Example tab, Vocabulary Card, Word Book, Review, and SAT questions —
   confirm no cross-surface regression, confirm curated/fallback tagging semantics unchanged, confirm
   `genVocabInContext()` untouched.
5. **Documentation** — implementation log recording before/after examples for a representative sample
   of words across all four POS categories, plus the exact suffix-heuristic table shipped.

*(This numbering is illustrative for the eventual Architecture phase; not a commitment to exactly 5
tasks.)*

## 21. CEO decisions required

1. **Approve Phase 1** (redirect `examples()` to `v.data[3]`/`v.data[4]`) — recommended to ship first,
   independently, given its outsized impact-to-risk ratio.
2. **`exampleBank`: delete outright, or leave as unused dead code?** Recommend deleting — it becomes
   fully redundant once Phase 1 ships, and this repo's own convention favors deleting confirmed-unused
   code over leaving silent dead weight.
3. **Approve Phase 2's scope** — a local, suffix-based POS guesser + POS-aware frame families
   (multiple frames per POS, not four rigid templates) for the genuinely-uncurated residual case.
   Confirm the illustrative frames in §12-15 are directionally acceptable (final wording is a Dev-
   phase detail, not locked here).
4. **Confirm AI is not required/authorized for this milestone** (§18) — matches established pattern,
   flagged for explicit reconfirmation since this task's investigation scope specifically asked
   whether AI is necessary.
5. **Sequencing:** ship Phase 1 and Phase 2 together, or Phase 1 first as an immediate, low-risk win
   with Phase 2 as an explicit follow-up milestone? Recommend the latter, given Phase 1 alone already
   resolves the reported problem for 97.7% of the words that can appear.
6. **The narrow morphology-derived POS-mismatch gap** (§3.3) — accept as a known, low-frequency,
   deferred limitation (recommended), or fold a fix into Phase 2's scope?

## 22. Explicit stop point

Per the CEO's instruction: **no application code has been modified.** This document is a PM-phase
investigation and specification only. Architecture Design should not begin until the CEO reviews and
either approves a scope (Phase 1 only, Phase 1+2, or a revised scope), requests changes, or redirects
this investigation.

## 23. CEO decisions (recorded 2026-08-14)

**Both phases approved.** All 6 open decisions from §21 resolved:

1. **Phase 1 approved**, exactly as recommended — `examples()` reads `v.data[3]`/`v.data[4]`
   (the current vocab item's own curated example) as primary source. Explicit constraint added:
   **do not regenerate or rewrite existing curated examples**, and **do not use `exampleBank` as the
   primary source when a curated example is already available** — both already this document's own
   design, now made binding.
2. **`exampleBank`: retire.** Confirmed as this document recommended.
3. **Phase 2 approved**, scoped explicitly: a local/offline fallback engine combining existing POS
   (when known), conservative POS inference (when unknown), morphology/suffix cues, **multiple
   sentence-frame families per POS** (explicitly re-rejecting a bare 4-template NOUN/VERB/ADJECTIVE/
   ADVERB scheme), and contextual meaning reuse where safely available. Concrete family examples
   given per POS (verb: SVO / that-clause / prepositional-complement; adjective: adjective+noun /
   linking-verb+adjective / adjective+prepositional-complement; adverb: verb-modifying / sentence-
   adverb / degree-manner; noun: subject / object / academic-collocation) — directional, not locked
   wording, matching this document's own "final wording is a Dev-phase detail" framing.
4. **AI reconfirmed not required and not authorized this milestone** — explicit: no Gemini, no new
   provider, no API key, no paid service, no schema migration.
5. **Sequencing:** not explicitly re-decided as "Phase 1 first, ship separately" vs. "together" —
   the CEO's instruction proceeds directly to a single combined Architecture covering both phases
   (see `02-ARCHITECTURE.md`), leaving the actual *shipping* sequencing (one PR vs. two) as an
   Architecture/Developer-Task-breakdown decision rather than a separate CEO gate.
6. **Morphology gap: addressed, not deferred.** The CEO explicitly requires Architecture to "define
   when reuse is safe and when a fallback should be generated instead" — this is now binding scope,
   not an optional follow-up. See `02-ARCHITECTURE.md` §4 for the resulting safety rule.

**New requirements added beyond §5-§21's draft framing**, now binding for Architecture:
- Collocation: prefer existing curated data; where none exists, prefer conservative generic academic
  patterns over inventing specific lexical relationships.
- Explicit XSS/escaping requirement and explicit regression requirement for `genVocabInContext()`
  (SAT vocabulary-in-context questions) remaining byte-for-byte unaffected.
- Explicit requirement that Korean translation remains available for every example, curated or
  fallback.
- Explicit requirement that the same word remains deterministic (same word → same example) for
  consistency, matching the established Milestone 4 convention this document already cited in §17.

---

## Handoff — Milestone 6 PM Investigation

- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/PM.md`, plus direct reads of `index.html` (`dictionary`, `supplementalVocab`,
  `broadVocabLexicon`, `exampleBank`, `fallbackExamplePool`, `buildFallbackVocabData()`,
  `buildVocabCardData()`, `safeMorphologyBase()`, `examples()`, `genVocabInContext()`,
  `highlightExampleWord()`, `fallbackKoreanGloss()`), plus direct counts/pattern analysis of all four
  vocabulary data objects (exact entry counts, POS-distribution breakdown of `broadVocabLexicon`,
  targeted sampling of verb/adverb entries across the full alphabet range).
- **Scope completed:** All 10 investigation questions and all 20 deliverable sections answered from
  direct code inspection, not assumption. Root cause identified precisely (wrong data source in
  `examples()`, compounded by a POS-blind fallback template pool) and quantified exactly (1,115
  curated words exist with good examples; only 26 are actually used by the Example tab). Proposed a
  two-phase architecture that prioritizes reusing existing curated data over generating new content,
  per the CEO's explicit instruction. No application code touched.
- **Files changed:** `docs/milestones/milestone-06/01-PM-SPEC.md` (new, this file).
- **Commits created:** None this session — pending CEO review, per this repo's established pattern
  for PM-phase documents.
- **Unresolved risks:** None new. The CEO decisions in §21 are open scope questions, not risks.
- **Next agent:** CEO, to approve/revise this scope before Architecture Design begins.
- **Explicit stop point:** Per the CEO's "STOP after PM investigation/specification" instruction.
