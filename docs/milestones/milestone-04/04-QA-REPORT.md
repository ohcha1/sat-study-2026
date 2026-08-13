# Milestone 4 — QA Report: Dictionary / Vocabulary Experience Upgrade

Independent QA pass. Branch `feature/vocab-experience-upgrade`, tested at HEAD
`4e7ecb07a4a5eae9dda7db6b7d37a9c263182fdb` (confirmed matches expected before testing). No
application code modified during this pass — confirmed via `git status`/diff-scope check before and
after.

**Method:** Live in-browser testing (no Node/jsdom in this environment, consistent with every prior
milestone). Verified the actual implementation, not the logs: read `index.html` directly for the
`canHandle()` matrix, the diff hunk locations, and the read-merge-write code path before testing
behavior; independently re-derived test cases rather than re-running the Developer's own scripts
verbatim (several — the genuine multi-navigation reload sequence, the mixed old/new-format record
seeding, the unsave→reviewState edge case, the cross-adapter `canHandle` matrix — go beyond what
`03-IMPLEMENTATION-LOG.md` describes testing).

## 1. QA Verdict

**PASS**

All 10 PM-Spec acceptance criteria verified. No BLOCKER, HIGH, or MEDIUM findings. One LOW design
observation (§9). One `BLOCKED_HUMAN_INPUT` item, pre-existing and already correctly scoped as
non-blocking by the Developer (§10).

## 2. PM acceptance criteria results (`01-PM-SPEC.md` §6)

| # | Criterion | Result |
|---|---|---|
| 1 | Important SAT Words default + Show all toggle | **PASS** — exact counts (6/12/6 curated-vs-fallback split in one test, 10/12 in another), `a.vocab` confirmed unmutated by toggling |
| 2 | Curated vs. fallback distinguishable, non-technical wording | **PASS** — "사전 수록 단어"/"일반 설명" tags confirmed; no "fallback"/"AI"/"provider" string found anywhere in student-facing card output |
| 3 | Mobile above-the-fold content, collapsible secondary | **PASS** — 375px screenshot confirmed word/IPA/POS/context/definitions/actions visible without scrolling past secondary content; `<details>` open/close confirmed functionally (not just visually) |
| 4 | Word Book save persists via `vocabularyProgress`, no new store | **PASS** — confirmed via two independent save/reload/re-login cycles with zero passage in memory each time; `openDB()` store list unchanged (6 stores) at every checkpoint |
| 5 | Review/learned state + filtered Word Book view | **PASS** — new→reviewing→learned cycle confirmed, persisted across a genuine reload, filter counts exact and mutually exclusive across two independent word sets |
| 6 | Guest-mode Word Book messaging | **PASS** — confirmed guest gate renders correctly, matches Report tab's established pattern |
| 7 | AI enhancement fails closed; everything else works with zero AI | **PASS** — no-key state confirmed to hide the button entirely (render-time check, not just post-click failure); full Dictionary Card/Word Book/Review flow re-verified with `window.SAT_STUDIO_DEV_KEYS` deleted throughout most of this session |
| 8 | AI enhancement succeeds with a key configured | **PASS (mocked)** — real key not available this session (see §10); mocked success/failure/payload/XSS/double-click all verified |
| 9 | `DEVELOPMENT_PLAN.md` corrected word-count figure | **PASS** — correction note present, original text preserved inline, new milestone entry added following the established numbering-collision pattern |
| 10 | End-to-end product-success flow | **PASS** — see §3 |

**10 of 10 PASS.**

## 3. End-to-end student walkthrough result

**PASS — full flow completed successfully, including a genuine browser navigation (not a simulated
reload) partway through:**

passage → `analyze()` → Important SAT Words (default, 12 cards) → selected a curated word with a
context sentence → confirmed IPA/POS/context sentence/Korean meaning all present → opened the
`<details>` disclosure (confirmed actually toggled, not just present in markup) → saved to Word
Book → reopened Word Book tab, confirmed the word listed → marked **Reviewing** → **navigated the
browser to a fresh load of the page** (zero JS state, zero passage in memory) → re-logged in →
reopened Word Book, confirmed the same word still listed with `reviewState:"reviewing"` intact →
marked **Learned** → confirmed final state `learned`.

**The CEO's required success criterion — "a student using one SAT passage can discover, understand,
save, and later reopen an unfamiliar word without leaving SAT Studio" — is MET**, independently
verified with a real reload boundary in the middle, not merely a same-session function-call chain.

## 4. 5-minute visible-improvement result

**YES.** Within the first screen of the Vocabulary tab, a student immediately sees a materially
different experience from the prior version: a curated/general tag on every card (previously all
cards looked identical regardless of data quality), a context-sentence box showing the word used in
their own passage (previously absent entirely), a default filtered list instead of every extracted
word at once, and visible Save/Review controls. Opening the "단어장" (Word Book) tab — new, not
present in the prior version — and seeing a saved word persist after a real reload is a concrete,
noticeable capability change reachable within the first few interactions, well under 5 minutes.

## 5. Word Book persistence result

**PASS**, with the read-merge-write fix specifically re-verified independently (not just re-trusted
from the implementation log):
- Two separate save→reload→re-login cycles (different accounts, different sessions) both correctly
  reconstructed the saved word with zero passage/analysis in memory.
- A whole-passage `saveCurrent()` call after a Word Book save was independently re-tested twice
  (once mid-session, once after a fresh reload) and both times preserved `savedToWordBook` while
  correctly updating `lastSeen` — confirming architecture §4 risk 2's mitigation holds under
  realistic, repeated use, not just a single test.
- A hand-seeded **old-format** record (`{id, userId, word, lastSeen}`, no Milestone-4 fields) read
  back cleanly with no error and no corruption, and correctly did **not** appear in the Word Book's
  saved list (since it was never explicitly saved) — confirms backward compatibility without silent
  data reinterpretation.
- `openDB()`'s object store list confirmed identical (6 stores) at three separate points across this
  session.

## 6. Review-state result

**PASS.** Default state is `new` (confirmed both by absence of the field on a fresh record and by
the rendered badge label). Cycling confirmed new→reviewing→learned in order. State confirmed
persisted across a genuine page reload. Word Book's four-way filter (전체/새 단어/복습 중/학습 완료)
confirmed to produce exact, mutually exclusive counts across two independently-constructed word
sets — a word in the "learned" filter never appeared in "new," and vice versa.

## 7. Gemini vocab-context result

**PASS** for everything verifiable without a real key:
- Core Dictionary Card fully functional with `window.SAT_STUDIO_DEV_KEYS` deleted — re-confirmed
  across this entire session, not just once.
- No-key state hides the AI button entirely at render time (fails closed before any click).
- `gemini.canHandle()` confirmed to accept `"summary"` and `"vocab-context"`, reject `"translation"`;
  cross-checked that `legacy-translation.canHandle("vocab-context")` is `false` — no accidental
  cross-routing in either direction.
- Mocked success renders correctly and is HTML-escaped (a `<script>`/`onerror` payload in the mocked
  response did not execute).
- Mocked HTTP 503 failure shows a graceful message; the local word card remained fully intact
  (word, definitions, other buttons all still present) after the failure.
- Double-click guard confirmed: two concurrent `loadVocabContext()` calls produced exactly one
  network request.
- Captured request payload confirmed to contain only the `contents` key (i.e., only the
  constructed prompt — no separate student/session/passage-title fields).
- No API key or secret found anywhere in `index.html` or in this branch's commit history (`git log
  -p origin/main..feature/vocab-experience-upgrade`, pattern search); none of this session's own
  test values (fake keys, test usernames) leaked into any committed diff.

**BLOCKED_HUMAN_INPUT:** a real Gemini API round-trip was not performed — requires a CEO-provided
key in a controllable browser session. Does not block this verdict; every other check for this
feature was completed.

## 8. Regression result

**PASS.** Full 9-tab render sweep (including the new `wordbook` tab); Risk A (`translateSentenceReliable`
present)/B (rapid-save ID uniqueness re-confirmed)/C (`analyzeInFlight` guard present) all intact;
`providerRegistry` unchanged (`legacy-translation` + `gemini`, no new provider); Gemini Summary UI
(`loadGeminiSummary`) still present and still dormant without a key; local extractive summary
(`summaryEn`) still populated independent of any AI call; SAT quiz grading (`gradeQuiz()`) ran
without error; PDF/OCR/HEIC import functions (`importDocument`, `getPdfLibrary`,
`getDocumentOcrWorker`, `convertHeicForOcr`) all present; IndexedDB schema unchanged (6 stores,
confirmed at three points); 7 responsive `@media` blocks unchanged; no translation left in
`'loading'` state; no new XSS surface found (both new escape-requiring points — `contextSentence`
and the AI explanation text — independently tested with injection payloads, both escaped).

Independently confirmed at the code level (not just runtime): diffing this branch against
`origin/main` shows exactly 18 hunks in `index.html`, all clustered in the expected regions
(branding strings, vocab-building loop, the new card/Word Book/review functions, the `gemini`
adapter, login/logout, `saveSessionForUser()`) — none anywhere near grammar, SAT-question-generation,
translation core logic, PDF/OCR import, or the `openDB()` schema definition.

## 9. Findings by severity

**BLOCKER:** none.
**HIGH:** none.
**MEDIUM:** none.
**LOW:**
1. **Review state is not reset when a word is unsaved from the Word Book, and reappears if the same
   word is re-saved later.** Reproduced directly: save → cycle to "learned" → unsave → the stored
   record still shows `reviewState:"learned"` (only `savedToWordBook`/`savedAt` are cleared) →
   re-saving the same word immediately shows it as "learned" again, with no chance to start over.
   This is a consequence of the shared-record, additive-field design (architecture's own explicit
   choice, working as designed — not a bug in the implementation) rather than a defect; whether a
   student would find this surprising or reasonable is a product/UX judgment call, not a
   correctness issue. No crash, no data loss, no security exposure. Does not block this milestone;
   recommend logging as a candidate for a future small polish item (e.g., clearing `reviewState` on
   unsave) if the CEO judges it worth changing.

No other findings. Every other test case in this report resolved exactly as designed.

## 10. BLOCKED_HUMAN_INPUT items

1. Real Gemini API round-trip for the `vocab-context` taskType — requires a CEO-provided key in a
   controllable browser session. All other verification for this feature (§7) was completed and is
   not gated by this item. Same constraint already on record for the Milestone 3 Gemini Summary UI
   feature.

## 11. Files modified

**None** by this QA session. `git status` confirmed clean of application-code changes before and
after testing (only pre-existing untracked `.DS_Store`/`.claude/`). This report file is the only
addition.

## 12. Recommended next action

Proceed to Principal Review. No Critical/High/Medium defects exist to send back to Development. The
one LOW finding (§9.1) is optional polish, not a release blocker, and may be logged for a future
milestone at the CEO's discretion rather than reopening this one. The `BLOCKED_HUMAN_INPUT` item
(§10) should be resolved with a real key test before final release sign-off, but does not block
Principal Review from proceeding on the code/logic verification already complete.

---

## Handoff — Milestone 4 QA

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Source documents read:** `CLAUDE.md` (incl. §5 governance reference), `AI-Company/GOVERNANCE/
  AUTONOMOUS_CONTINUATION_POLICY.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/QA.md`, `.ai-company/TESTING_STANDARDS.md`, `.ai-company/DEFINITION_OF_DONE.md`,
  `docs/milestones/milestone-04/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
  and `index.html` directly (vocab-building loop, `vocabCardTemplate`/`vocab`/`wordBookView`/
  `renderWordBookAsync`, `updateVocabProgress`/`ensureVocabProgressLoaded`, the `gemini` adapter,
  `render()`'s tab dispatch).
- **Scope completed:** Independent verification of all 10 PM-Spec acceptance criteria, the
  end-to-end student walkthrough (with a genuine mid-flow page reload), the 5-minute-improvement
  assessment, Word Book/review-state persistence (including two independent read-merge-write
  regression checks and an old-format-record compatibility check), Gemini vocab-context (all
  non-key-requiring checks), and a full regression sweep including an independent code-level diff
  scope check. One LOW finding surfaced (§9.1), not blocking.
- **Files changed:** `docs/milestones/milestone-04/04-QA-REPORT.md` (new, this file). No application
  code touched — confirmed via `git status` before and after.
- **Commits created:** None this session (not yet committed — pending CEO/Principal Review, matching
  this repo's established pattern).
- **Tests performed:** see §2-§8 in full above.
- **Unresolved risks:** none newly found. The LOW finding (§9.1) and the `BLOCKED_HUMAN_INPUT` item
  (§10) are both already fully characterized above.
- **Next agent:** Principal Reviewer.
- **Explicit stop point:** per the task's "STOP at QA milestone gate" instruction, this session stops
  here. No push, no merge, no code changes.
