# DEVELOPMENT_PLAN.md

A plain running list of what's done and what's next. Not a milestone pipeline — no PM spec,
no architecture doc, no QA report needed to check something off. See `CLAUDE.md` for how this
project is meant to be worked on.

## Done

- **단어장·복습 (Review Deck) — Leitner-style spaced repetition.** This was the whole point of
  restarting: track which words/phrases are actually memorized vs. not, and resurface the
  not-yet-memorized ones on a spaced schedule.
  - New nav tab `📌 단어장·복습`.
  - `📌 단어장에 저장` save button added to both the vocab tab and the phrases tab, so any word
    or expression encountered during passage analysis can be added directly.
  - 5-box Leitner schedule (1 / 3 / 7 / 14 / 30 days). Correct recall promotes a box; incorrect
    recall resets to box 1. Reuses the existing `vocabularyProgress` IndexedDB store (it existed
    before but was barely used) — no new storage architecture introduced.
  - `saveSessionForUser()` now merges into existing `vocabularyProgress` records instead of
    overwriting them, so passively encountering a word during passage analysis never wipes out
    review progress you already built up on it.
  - Verified end-to-end with a headless-browser smoke test (`smoke_test.js`): account creation,
    empty-deck state, adding a word from the vocab tab and from the phrases tab, and a full
    know/know/don't-know grading cycle that correctly moves a card through boxes 1→2→3→1 with the
    right `nextReviewAt` dates.

- **예문(examples tab) part-of-speech fix.** Example sentences for words not in the small curated
  `exampleBank` used to fall back to a generic template that always quoted the word as a
  discussed "term" (e.g. `the class used the term "verb"`), so every word — verb, adjective,
  whatever — showed up in a noun-like slot instead of being used the way it's actually used.
  - The examples tab now prefers the word's own `data[3]`/`data[4]` (the same example sentence
    already shown on the vocab tab, which for real dictionary words is already written to match
    the word's real part of speech) before falling back to anything generic.
  - For words with no dictionary entry at all, added a spelling-based part-of-speech guess
    (`guessPOSForFallback`) and a set of fallback sentence templates per part of speech (verb
    base form, past tense, `-ing` gerund, adjective, adverb, singular/plural noun), so even a
    totally unknown word gets used in a grammatically appropriate slot instead of being quoted as
    a noun.
  - Verified with a Playwright test: real dictionary verbs (`demonstrate`, `illustrate`) still use
    their existing correct examples; made-up words with recognizable suffixes (`-ed`, `-ing`,
    `-ous`, `-ly`) each routed to the matching POS template and appeared as a real verb/adjective/
    adverb in the generated sentence, not a quoted noun.

- **숙어·구문 (idioms/phrases tab) level check.** Reviewed the phrase bank against actual SAT
  difficulty. Finding: about 20 of the ~90 curated phrases (`in contrast`, `as a result`,
  `according to`, `for instance`, `on the other hand`, etc.) are basic connectors a genuine
  SAT-track student already knows, but they were untagged and therefore ranked as `"academic"`
  — the same priority tier as genuinely harder phrases — so they could crowd out the harder ones
  when a passage had more than the 8-phrase display cap worth of matches.
  - Re-tagged those ~20 basic connectors as `"core"` (lowest ranking priority) and promoted
    ~10 existing phrases that are genuinely higher-register (`be attributed to`,
    `call into question`, `give rise to`, `stem from`, `in light of`, `regardless of`, etc.) to
    `"sat-high-value"`.
    - Added 19 new phrase-bank entries plus one new regex pattern, all `"sat-high-value"`,
    covering constructions that actually appear in SAT Reading passages (especially historical
    excerpts) and SAT Writing & Language idiom items but weren't covered before: inverted
    conditionals (`were it not for`, `had it not been for`), inverted time clauses
    (`no sooner ... had ... than`), and formal qualifiers/connectors (`let alone`, `far from`,
    `by virtue of`, `on the grounds that`, `in the wake of`, `at odds with`, `to the extent that`,
    and others).
  - Verified with a Playwright test against a deliberately hard, SAT-register test passage: the
    phrases tab now surfaces entirely `sat-high-value` constructions (`were it not for`,
    `in the wake of`, `no sooner had ... than`, `to the extent that`, etc.) for a hard passage,
    while the built-in simple sample passage still correctly shows its one basic connector
    (`in contrast`) as low-priority `core`.

- **Content-quality review across vocab/examples/phrases/grammar/SAT questions, plus two real
  SAT-question bugs fixed.** Ran real SAT-register test passages through every tab to check
  quality, not just the built-in demo. Findings: grammar detection is already solid (correctly
  found 6 distinct structures — passive, reduced clause, correlative conjunction, relative
  clause, etc. — in one test passage, all correctly labeled). Vocab/examples are held back by a
  narrow dictionary (~1,115 words total across `dictionary`+`supplementalVocab`+
  `broadVocabLexicon` — the older "~40 words" note below was wrong, but even ~1,115 isn't enough:
  8 of 12 words in a real geology-passage test fell through to the generic "estimated" fallback).
  SAT question generation had the most serious issues, now fixed:
  - **Vocabulary-in-Context question coverage.** This question type only fires for words with a
    hand-curated 3-distractor set in `QG_VOCAB_DISTRACTORS`, and only 11 words had one — so this
    classic SAT question type essentially never appeared on a real, non-demo passage. Added 87
    more curated entries (98 total) for genuinely high-value SAT vocabulary already present in
    `dictionary` (empirical, plausible, infer/imply, correlation/causation, scrutinize,
    ubiquitous, etc.), each with 3 deliberately-plausible-but-wrong distractors grounded in the
    word's actual meaning. Verified: went from 0/3 to 2/3 real test passages producing a
    vocabulary question.
  - **Inference/Comparison/Author's-Purpose questions could produce grammatically broken answer
    choices.** Root cause: the shared contrast-detection regex matched the "but" inside "not
    only X but also Y" (an additive construction, not a contrast), so a sentence containing that
    correlative got mis-split into two clause fragments and rewoven into run-on, ungrammatical
    answer choices. Added `qgRealContrastMatch()` to skip that false match and find a genuine
    contrast marker (or correctly find none). Also found and fixed two related bugs while
    tracking this down: `qgLooksLikeBareParticipial()` failed to catch a reduced clause when a
    short adverb ("still being rejected...") preceded the participle, and separately wrongly
    treated "being"/"been" as proof of a finite verb; and `genCauseEffect()` didn't strip its own
    matched connective ("led to") from the extracted clause, producing doubled phrasing like
    "leads to led to the development of...". All three verified fixed against the exact test
    sentences that originally exposed them, with zero regressions in the review-deck and phrase
    smoke tests.
  - Also fixed, while testing vocab quality directly: the POS-guessing heuristic added earlier
    this session mis-tagged common participial adjectives (`interesting`, `unsupported`) as
    verbs, and 3rd-person-singular verbs ending in "s" (`illustrates`) as plural nouns. Added an
    explicit adjective-override list and a dedicated `verbs3s` category, checked before the
    generic "ends in s → plural noun" guess.

- **Vocabulary dictionary breadth — first expansion pass.** Ran 5 diverse SAT-register test
  passages (plate tectonics, minimum wage economics, Roman Empire, coral bleaching, printing
  press) through the app to find real, non-demo coverage gaps rather than guessing. Found ~90
  ordinary passage words with no `dictionary`/`supplementalVocab`/`broadVocabLexicon` entry.
  Hand-wrote and added 30 new `broadVocabLexicon` entries for the highest general-value words
  from that gap list (5 of the original 35 candidates turned out to already exist, so were
  skipped to avoid duplicates): `multiplicity`, `overreach`, `redistribute`, `theoretical`,
  `institution`, `intellectual`, `innovation`, `laborious`, `mortality`, `skepticism`, `prompt`,
  `undergo`, `unify`, `tolerant`, `complicate`, `dismiss`, `reshape`, `occupy`, `invention`,
  `identify`, `spread`, `selective`, `elite`, `address`, `formation`, `discovery`, `development`,
  `narrow`, `symptom`, `manuscript` — each with full `[pos, definition, ko, exampleEN, exampleKO,
  synonyms, antonyms]` content. Deliberately excluded proper nouns/domain-jargon (`Wegener`,
  `Gutenberg's`, `tectonics`, `seafloor`) and overly-basic words (`together`, `finally`,
  `roughly`) that wouldn't be a good use of a curated slot. Verified with a Playwright test that
  all 30 words now resolve via `broadVocabLexicon` lookup with the correct 7-field shape, and
  re-ran `smoke_test.js` with zero regressions.
  - **Honest scope note:** this covers 30 of the ~90 words found in the gap analysis — a
    meaningful dent, not a fix. The vocabulary-coverage bottleneck (item 4 below) is still the
    main thing holding back vocab/examples/SAT-vocab-question quality on real passages and will
    need further passes, ideally with a more systematic approach than one-at-a-time hand curation
    (see item 4).

## Next priorities (roughly in order)

1. **Manual word/phrase entry.** Right now the only way to add something to the review deck is
   via the save button during passage analysis. Add a simple form (word + Korean meaning) so
   words picked up in ESL class, not from a pasted passage, can be added directly.
2. **Wire the review deck into the ESL-expressions workflow.** The original ask was specifically
   about ESL-class expressions and vocabulary, not just SAT-passage vocab. Make sure whatever
   "phrases" already captures lines up with real ESL-class material, and consider a quick-add
   flow that doesn't require going through passage analysis at all.
3. **"AI Explanation" honesty gap.** The explanation feature is currently template/heuristic
   based, not a real LLM call, despite the name. Either rename it to be honest about what it is,
   or wire it to a real model call if that's worth the cost/complexity for a personal tool.
4. **Vocabulary dictionary coverage is still the main bottleneck.** ~1,145 words across
   `dictionary`+`supplementalVocab`+`broadVocabLexicon` (after the first 30-word expansion pass
   above) sounds like a lot, but real passages still turn up ordinary words with no entry (roughly
   60 still missing out of the ~90-word gap list found during testing), which fall through to the
   generic "estimated" fallback for definition, Korean gloss, and example sentence alike. This is
   the throughline behind most of the remaining thinness in vocab, examples, and SAT vocabulary
   questions. One-word-at-a-time hand curation works but is slow; worth considering either several
   more focused expansion passes, or switching to a lookup approach (e.g. an API or a bundled
   larger wordlist) if the manual pace doesn't keep up.
5. **Deployment beyond `file://`.** This is a single HTML file with no build step, which is great
   for iterating, but it currently only runs opened locally. Worth deciding later whether/how to
   host it (even something as simple as static hosting) once the core loop is trustworthy.

## Explicitly not doing right now

- Multi-AI router / provider-switching architecture — an earlier design-only detour with no
  working code behind it. Not resurrecting it unless a real, concrete need shows up.
- Splitting the single `index.html` into modules/a build step. The file is large, and that's a
  real cost, but restructuring it is a mechanical follow-up once the feature set stabilizes — not
  something to do while still actively changing what the app does.

## Repo cleanup (2026-08-18)

Consolidated everything onto `main` as the single branch to develop on. `clean-restart` and
`multi-ai-v2-dev` are no longer needed — `main` now contains everything `clean-restart` had, and
`multi-ai-v2-dev`'s only unique commits were the abandoned multi-AI router detour noted above.
Also removed the legacy heavy-process artifacts (`_archive/2026-08-legacy-process/`: old
PM/QA/architect docs, milestone templates, and a superseded `index.html` version from that
process) from the repo entirely — they were cautionary history, not anything referenced by
current code.
