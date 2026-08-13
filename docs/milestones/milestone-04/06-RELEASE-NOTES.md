# Milestone 4 — Release Notes: Dictionary / Vocabulary Experience Upgrade

Release Manager packaging pass. Branch `feature/vocab-experience-upgrade`, packaged at HEAD
`3143123` (the concurrency hot-fix commit — the last code change on this branch). No application
code modified during this packaging session — confirmed via `git status`/diff-scope check before and
after (see §10).

**Purpose:** package this milestone's already-complete Development/QA/Principal-Review/hot-fix work
for CEO push approval. This document summarizes what ships; it does not re-run or re-judge any prior
phase's verdict.

## 1. Branding

- Title: `SAT English Learning Studio 2026 — AI`
- Subtitle: `SAT Reading · Vocabulary · Grammar · AI Study Tools`

CEO-approved (`01-PM-SPEC.md` §9.4), deliberately narrower than the originally proposed
"AI-Powered SAT Reading · Vocabulary · Grammar · Practice" wording, which was rejected as overstating
functionality that is not actually AI-generated (Reading/Vocabulary/Grammar/Practice are all
deterministic/local; only the on-device Chrome Translator tier and the opt-in Gemini adapter are
AI/ML-adjacent).

## 2. Student-visible improvements

- Redesigned Dictionary Card: word, pronunciation/IPA, part of speech, **context meaning** (the
  word's sense as used in the actual passage sentence), Korean meaning, English definition —
  above-the-fold on mobile; synonyms/antonyms/example/dictionary links in a collapsible
  `<details>` section below the fold.
- **Important SAT Words** default view (curated-only), with a "Show all words" toggle that reveals
  every extracted word without mutating the underlying list.
- Curated-vs-general **transparency tags** ("사전 수록 단어" / "일반 설명") — a student can tell a
  real dictionary entry from a generic-fallback one without any technical/provider terminology
  appearing in the UI.
- Passage-specific **context sentence** shown on every card (local, free, not AI-generated).
- **Word Book** — an explicit per-word save/unsave action, independent of whole-passage saving, with
  its own "단어장" tab that reconstructs saved words with zero passage in memory.
- Persistent **Review states** (new / reviewing / learned) per word, with a four-way filter
  (전체/새 단어/복습 중/학습 완료) on the Word Book view.
- Optional **Gemini vocab-context** explanation ("why this meaning fits this sentence") — dormant
  without a configured key, never the source of truth for local lexical data.

## 3. Persistence

- Reuses the existing `vocabularyProgress` IndexedDB store (same keyPath, same `userId` index) —
  **no schema migration**.
- Word Book/Review fields (`savedToWordBook`, `savedAt`, `reviewState`, `reviewedAt`) are additive
  only; the pre-existing passive `lastSeen` tracker was routed through the same read-merge-write
  helper so it can no longer blind-overwrite these fields.
- Saved words and review state reconstruct correctly across a genuine page reload and re-login
  (verified with zero passage/analysis in memory at reconstruction time).
- **Read-merge-write protection**: every writer to this store goes through one
  `updateVocabProgress()` entry point.
- **Per-word concurrency hot-fix** (commit `3143123`): `updateVocabProgress()` is now serialized
  per word via a promise-chain queue, so two mutations targeting the same word (e.g. a save and a
  review-state change fired close together) can no longer silently overwrite each other. Different
  words remain fully independent — no global lock. A failed mutation cannot permanently block later
  mutations for that word. Old-format records (written before this milestone) remain compatible.

## 4. QA

- **10/10 PM-Spec acceptance criteria: PASS** (`04-QA-REPORT.md` §2).
- **End-to-end student flow: PASS**, including a genuine mid-flow browser reload, not a simulated one
  (`04-QA-REPORT.md` §3).
- **5-minute visible-improvement criterion: YES** (`04-QA-REPORT.md` §4).

## 5. Principal Review

- **Final verdict: APPROVE** (originally "APPROVE WITH CONDITIONS" pending the MEDIUM finding below;
  the condition is now resolved — see `05-REVIEW-REPORT.md` §0/§13).
- **MEDIUM concurrency finding: RESOLVED.** Root cause, fix, targeted tests, and independent
  Principal re-verification are fully documented in `05-REVIEW-REPORT.md` §13.
- **Hot-fix commit:** `3143123`.

## 6. Deferred LOW item

Unsaving a word from the Word Book does not reset its `reviewState` (re-saving the same word later
shows it already marked, e.g. "learned"). Confirmed as a consequence of the shared-record additive
design, working as intended, not a defect — no crash, no data loss, no security exposure. **Accepted
and deferred**, non-blocking, per explicit CEO instruction not to fix it as part of this milestone.

## 7. Gemini

- Adapter-level real API smoke test (from the Milestone 3 Gemini Summary UI feature, same adapter
  this milestone extends): previously **PASS**.
- `vocab-context` mocked path (success, failure, payload shape, double-click guard, XSS escaping):
  **PASS**.
- Real `vocab-context` live-key round-trip: **BLOCKED_HUMAN_INPUT** — requires a CEO-provided key in
  a controllable browser session, not obtainable in this environment. Not a release blocker unless
  the CEO decides otherwise; every other verification for this feature is complete.

## 8. Preservation

Confirmed unmodified throughout Development, QA, Principal Review, the hot-fix, and this packaging
pass:
- Translation architecture (Multi-AI router, `legacy-translation` adapter, sequential-dispatch
  requirement).
- Risk A (Chrome Translator timeout), Risk B (save-ID uniqueness), Risk C (`analyzeInFlight` guard).
- Gemini Summary UI (dormant without a key, independent of this milestone's new `vocab-context`
  taskType on the same adapter).
- PDF/photo/OCR/HEIC import.
- IndexedDB schema — still exactly 6 object stores, no new store or index.
- Responsive/mobile behavior — all 7 `@media` blocks unchanged.

## 9. Security

No API key, secret, token, or credential committed anywhere on this branch — confirmed by pattern
search across every commit's diff (`git log -p origin/main..feature/vocab-experience-upgrade`,
re-confirmed for the hot-fix commit specifically in `05-REVIEW-REPORT.md` §13.6) and independently
re-checked during this packaging pass (§10).

## 10. Release state

- **Local branch only:** `feature/vocab-experience-upgrade`, 10 commits ahead of `origin/main`.
- **Not pushed.**
- **Not merged.**
- **CEO push approval still required** before any push/merge/release action.

---

## Auto-decision log (carried forward)

Per `AI-Company/GOVERNANCE/AUTONOMOUS_CONTINUATION_POLICY.md`, only the Level-2 decisions with
lasting relevance are carried forward here (full detail in `03-IMPLEMENTATION-LOG.md`):

1. **Fallback-example selection made word-deterministic** (was batch-position-based) so
   `buildVocabCardData()` works standalone for Word Book reconstruction. Low risk, cosmetic only,
   reversible. CEO-confirmed accepted.
2. **Word Book/Review Queue built as one view with a segmented filter**, not separate tabs per state,
   per Architecture's smallest-diff recommendation. Low risk, reversible.
3. **`saveSessionForUser()`'s passive tracker routed through `updateVocabProgress()`** instead of a
   blind `idbPut()`, as part of Dev Task 5 — required to prevent the new Word Book/Review fields from
   being silently destroyed by the pre-existing passive write path.

## Verification performed this packaging session

- `index.html` unchanged: confirmed via `git status`/`git diff` showing zero changes to
  `index.html` from this session (only new/edited documentation files staged).
- `LATEST_GOLD_MASTER_NEXT.html` (Gold Master) unchanged: confirmed untouched, not referenced by any
  file this session edited.
- Only release/documentation files changed this session: `docs/milestones/milestone-04/
  06-RELEASE-NOTES.md` (new), `DEVELOPMENT_PLAN.md` (release-status line updated).
- No secrets: pattern search (`api[_-]?key|secret|token|password`, case-insensitive) across this
  session's diff returned no matches.

---

## Handoff — Milestone 4 Release Manager Packaging

- **Milestone:** Milestone 4 — Dictionary / Vocabulary Experience Upgrade.
- **Source documents read:** `01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
  `04-QA-REPORT.md`, `05-REVIEW-REPORT.md`, `DEVELOPMENT_PLAN.md`.
- **Scope completed:** Packaged this milestone's Development/QA/Principal-Review/hot-fix results into
  this release-notes document; updated `DEVELOPMENT_PLAN.md`'s release-status line only. No
  application code touched, no re-testing performed (this phase summarizes prior verdicts, it does
  not re-judge them).
- **Files changed:** `docs/milestones/milestone-04/06-RELEASE-NOTES.md` (new, this file),
  `DEVELOPMENT_PLAN.md` (release-status line updated, append-not-hide).
- **Commits created:** One documentation-only commit (see commit log for hash).
- **Tests performed:** N/A (Release Manager phase; verification limited to confirming no code/secret
  changes, per §10 above).
- **Unresolved risks:** None newly introduced. Carried forward: the deferred LOW item (§6) and the
  `BLOCKED_HUMAN_INPUT` Gemini live-key item (§7), both already fully characterized and both
  explicitly non-blocking for this packaging decision.
- **Next agent:** CEO — to grant or withhold push approval.
- **Explicit stop point:** Per the task's "STOP at CEO push-approval gate" instruction. No push, no
  merge, no release performed.
