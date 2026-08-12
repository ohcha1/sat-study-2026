# Milestone 3 — Gold Master Adoption (Multi-AI V2)

**Status: APPROVED by the CEO — 2026-08-12.** See §6a for the CEO's resolutions to the six open
questions raised in the original draft, and §6b for what remains open for Architecture to decide.
Architecture Design (`02-ARCHITECTURE.md`) may now begin.

> **Numbering note:** `DEVELOPMENT_PLAN.md` currently labels its own "Milestone 3" as "Dictionary
> Upgrade." This document claims the next sequential `docs/milestones/milestone-XX/` folder
> (`milestone-03`) because `milestone-02` is the highest existing folder, per
> `.ai-company/WORKFLOW.md`'s "each milestone gets a folder in the order work is actually done"
> convention. It is **not** the Dictionary Upgrade milestone. This collision is unresolved and is
> the first open question below — the CEO should decide whether to renumber, insert this as a
> decimal/lettered milestone, or fold it into the existing `docs/MULTI_AI_ARCHITECTURE_V2.md` track
> instead of the numbered milestone sequence.

## 1. Milestone goal

A new, much larger version of the application — `LATEST_GOLD_MASTER_NEXT.html`, supplied directly
by the CEO into the repository root — exists outside the numbered milestone sequence and outside
`docs/MULTI_AI_ARCHITECTURE_V2.md`'s Phase 1/Phase 2 plan. The CEO's request is to adopt it as the
new application baseline on a new branch, and re-apply the Multi-AI V2 router/adapter work already
built on `multi-ai-v2-dev` (providerRegistry, aiRouter, legacy-translation adapter, aiRouter-routed
translation and retry, dormant Gemini summary adapter) on top of it, without losing Milestone 1/2's
existing bug fixes and reliability work, and without expanding scope beyond what's already built.

## 2. Repository-state findings (read directly, not assumed)

These are concrete comparisons between `LATEST_GOLD_MASTER_NEXT.html` (SHA-256
`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`, 1,919,924 bytes, 4,434 lines)
and current `index.html` (101,059 bytes, 1,018 lines) on `multi-ai-v2-dev`, gathered to make the
scope below concrete rather than guessed:

- **Not a small update.** The gold master is ~19x the line count of current `index.html`. It is
  not base64-embedded assets inflating size (no `data:...;base64,` blobs found) — it is genuinely a
  much larger, denser application.
- **New subsystems absent from current `index.html` entirely**, found by diffing function names
  (163 top-level functions in the gold master vs. today's much smaller set):
  - A full **local account system**: `createAccount`, `loginAccount`, `logoutAccount`,
    `sha256Hex` (password hashing), `renderAuthUI`/`renderAuthPopoverContent`/`toggleAuthPopover`.
  - **IndexedDB-backed persistence** replacing/supplementing `localStorage`: `openDB`, `idbGet`,
    `idbPut`, `idbDelete`, `idbGetAllByUser`, `idbGetByIndex`, `idbClearUser`,
    `saveSessionForUser`, `openSavedSession`, `deleteSavedSession`, `restoreLastSession`.
  - **Document import**: `importDocument`, `extractPdfPassage`, `getPdfLibrary` (PDF.js),
    `ocrImageSource`, `getDocumentOcrWorker` (Tesseract.js), `convertHeicForOcr` (heic2any),
    `normalizeImportedPassage`.
  - **Session backup/restore**: `exportBackup`, `handleRestoreFile`, `validateBackupSchema`.
  - **A much larger SAT-question-generation engine** (~40 `qg*`-prefixed functions —
    `qgCentralIdx`, `qgQuote`, `qgClausePairFrom`, `qgSignals`, etc.) replacing today's 10-template
    `makeQuestions()`.
  - **A much larger grammar-detection engine** (~15 `detect*`-prefixed functions —
    `detectPassive`, `detectRelativeClause`, `detectParallelStructure`, etc.) replacing today's
    2-hardcoded-regex `grammarInsights()`.
  - **Pronunciation scoring** (`scorePronunciation`, `getChromeTranslator`,
    `setRecordingIndicator`) and **progress charts** (`buildMonthlyTrend`, `svgLineChart`,
    `monthKey`).
  - **A deterministic summary builder** (`buildDeterministicSummary`, `summarizeSentences`,
    `summarizeAlignment`, `alignWords`) replacing today's simpler `overview()` aggregation.
  - Its own **`translateSentence()`** (single async function, gold master line 1889) — a
    **different implementation** from current dev's Milestone-2-hardened
    `translateSentenceReliable()` chain.
- **New external CDN dependencies.** `getPdfLibrary()`, `getDocumentOcrWorker()`, and
  `convertHeicForOcr()` lazy-load `pdf.js`, `tesseract.js`, and `heic2any` from
  `cdn.jsdelivr.net` at runtime via injected `<script src="https://cdn...">` tags. Current dev has
  **zero** such dependencies today.
- **Multi-AI V2 functions present in current dev, absent from the gold master** (confirming these
  are genuinely additive, not something the gold master already has another version of):
  `registerProvider`, `aiRouter_request`, `retrySentence`, `runWithConcurrency`,
  `translateSentenceReliable`, `translateLingva`, `tryMyMemory`, `translationRowTemplate`,
  `updateTranslationRow`, `getDevApiKey`, `fetchWithTimeout`.
- No API-key-shaped literals (`apiKey`/`secret`/`token` assigned to a quoted 10+ char value) were
  found in the gold master by pattern search.

## 3. In scope

1. Establish `LATEST_GOLD_MASTER_NEXT.html` as the content baseline for a new `index.html` on a
   new branch (name TBD by Architecture; CEO's original suggestion was
   `multi-ai-v2-latest-dev`, branched from current `multi-ai-v2-dev` HEAD
   `bc204cf48d6c7ab360329bbf2bcd6bcc581a8781`), leaving `LATEST_GOLD_MASTER_NEXT.html` itself
   byte-for-byte unchanged in the repository as the protected reference copy (matching the existing
   precedent set by `42c1389` for the prior gold master). **CEO-confirmed: this file must not be
   modified at any point in this migration.**
2. Re-apply, onto that new baseline, the **architectural intent** of the six Multi-AI V2 pieces
   already implemented on `multi-ai-v2-dev`: `providerRegistry`/`registerProvider`, `aiRouter`, the
   `legacy-translation` adapter, normal translation routed through `aiRouter`, `retrySentence`
   routed through `aiRouter`, and the dormant Gemini summary adapter. **CEO-confirmed: Architecture
   may adapt the implementation to fit the gold master's structure rather than copy the old code
   verbatim — the router/adapter pattern is what must be preserved, not the exact old diff.**
3. Preserve existing regression coverage: passage analysis, translation, translation retry,
   vocabulary, idioms/phrases, grammar, SAT quiz, saved materials, responsive/mobile CSS — all
   continue to function after adoption, verified with the project's existing throwaway-jsdom
   convention per `.ai-company/TESTING_STANDARDS.md`.
4. Preserve, as authoritative, all of the gold master's newer implementations identified in §2:
   IndexedDB persistence, PDF/OCR/HEIC document import (including its CDN dependencies), the SAT
   question-generation engine, the grammar-detection engine, pronunciation scoring, and
   progress/chart functionality. **CEO-confirmed: newer Gold Master functionality has priority —
   do not replace it with older dev-branch code merely for consistency, and do not remove the CDN
   dependencies merely because the old version lacked them.**
5. Confirm no API key is committed or hardcoded anywhere in the new baseline, and that the Gemini
   adapter remains dormant (no-op) without a configured key.
6. Keep `multi-ai-v2-dev` unchanged as a historical backup branch.
7. Preserve the gold master's current UI as-is; change it only where a migration step technically
   requires it (e.g., wiring a router call), never for taste/consistency reasons.

## 4. Out of scope

- Adding any AI provider beyond the Gemini adapter that already exists (dormant, no key).
- Any UI redesign beyond what migration technically requires (CEO-confirmed).
- Extending, redesigning, or backend-ifying the account/IndexedDB system already shipped in the
  gold master. It is adopted as-is (local-only, client-side), not built out further. This is
  explicitly **not** an implementation of `DEVELOPMENT_PLAN.md`'s Milestone 6 ("User Accounts"),
  which specifies a backend-authenticated, multi-device account system — a different, larger thing.
- Refactoring any of the new gold-master subsystems (question generation, grammar detection,
  pronunciation scoring, etc.) beyond what's needed to wire the Multi-AI router around them.
- Any other numbered `DEVELOPMENT_PLAN.md` milestone (Dictionary Upgrade, UI Improvements, AI
  Tutor, User Accounts, Study History, Deployment) — untouched by this milestone. The
  `DEVELOPMENT_PLAN.md` "Milestone 3" name collision is recorded (§6a item 1) and is **not** to be
  resolved by rewriting that document.
- Downgrading IndexedDB to `localStorage` (CEO-confirmed: IndexedDB is authoritative).
- Removing the gold master's CDN dependencies (CEO-confirmed).
- Pushing to GitHub, merging to `main`, or deleting any existing branch.
- Resolving Milestone 2's unpushed/unreleased state — that record is preserved as-is and does not
  block this milestone's Architecture work (CEO-confirmed).

## 5. Acceptance criteria

Item 3 (translation) and item 6 (persistence) are directional per the CEO's resolutions in §6a but
still need Architecture to specify the exact mechanism — see §6b.

1. Given a fresh checkout of `multi-ai-v2-latest-dev`, opening the new `index.html` in a browser
   loads without console errors and without any network request firing before user interaction
   (other than what the gold master already did before this milestone).
2. Given an arbitrary English passage pasted into the app, passage analysis (vocabulary, idioms,
   grammar, SAT quiz, summary) produces output using the gold master's existing engines, unchanged
   in behavior from the un-merged gold master.
3. Given a passage requiring translation, translation is dispatched through `aiRouter` to the
   `legacy-translation` adapter (wrapping the translation approach Architecture designs per §6a
   item 5 — gold master's structure, Milestone 2's reliability improvements retained), and produces
   either a real translation or a clearly labeled error/retry state — never a silently stuck
   loading row.
4. Given a translation in an error state, clicking retry re-dispatches through `aiRouter` and
   either succeeds or re-displays the error state; it does not throw or hang.
5. Given the app has no Gemini API key configured, the Gemini summary adapter never activates and
   its absence produces no user-visible error.
6. Given the saved-materials feature (IndexedDB, per §6a item 6 — authoritative, not downgraded to
   `localStorage`), save/reopen/search/delete of a study set works without data loss or a
   stuck-loading state (i.e., SAVE-1 from Milestone 2 does not regress).
7. Given a PDF file or a photo (including HEIC) is provided as passage input, `importDocument`'s
   existing extraction path runs unmodified and yields extracted text, verified via a throwaway
   jsdom/mock test of the code path (not requiring a live network fetch of the CDN libraries in
   automated testing).
8. Given the repository's `.gitignore`/tracked files, no file contains a real, non-placeholder API
   key for any provider; `grep` for common key-shaped patterns across the new `index.html` and any
   new config returns nothing.
9. Given a viewport at existing mobile breakpoints, the gold master's responsive CSS renders
   without layout breakage (spot-checked, not a full visual regression suite).
10. `LATEST_GOLD_MASTER_NEXT.html`'s SHA-256 is identical before and after this milestone's work
    (`a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`).

## 6a. CEO resolutions (recorded 2026-08-12, verbatim decisions summarized against each item)

1. **Milestone identity/numbering** — Resolved: use `milestone-03` for this migration.
   `DEVELOPMENT_PLAN.md`'s existing "Milestone 3" (Dictionary Upgrade) label does not control this
   work. The naming collision is recorded here, not resolved by rewriting `DEVELOPMENT_PLAN.md` or
   any other historical document.
2. **Milestone 2's unresolved state** — Resolved: preserve Milestone 2's historical state as-is.
   Its unresolved release/push record does not block Architecture work for Milestone 03. No push or
   merge is being done to resolve it as part of this milestone.
3. **Scope size vs. time-box** — Resolved: treat this as **one** migration milestone with
   role-separated phases (PM → Architecture → Senior Developer → QA → Release Manager). If
   Architecture determines the implementation should be split into smaller developer tasks, it may
   define those tasks *inside* Milestone 03 (i.e., multiple Developer Tasks within one
   `03-IMPLEMENTATION-LOG.md`, not multiple milestone folders). Architecture's own time-box is
   ~20–30 minutes, scoped to resolving migration conflicts, not re-doing the discovery in §2.
4. **New external CDN dependencies** — Resolved: do not remove them merely because the old dev
   branch didn't have them. Architecture must document (not necessarily eliminate) the security,
   availability, offline, privacy, and fallback implications of keeping `pdf.js`/`tesseract.js`/
   `heic2any` as lazy-loaded CDN dependencies.
5. **Translation implementation conflict** — Resolved directionally, mechanism left to Architecture:
   compare gold master's `translateSentence()` against Milestone 2's `translateSentenceReliable()`,
   and design an approach that preserves the gold master's newer application structure while
   retaining useful Milestone 2 reliability improvements (retry/backoff, Lingva fallback, cache,
   concurrency limiting, structured status for the loading/error UI). Neither implementation is to
   be simply overwritten by the other.
6. **Persistence-layer conflict** — Resolved: IndexedDB (the gold master's) is authoritative; it is
   not downgraded to `localStorage`. Architecture should determine whether any migration of old
   `multi-ai-v2-dev` `localStorage` saved-data is actually useful/needed, or whether it can be
   treated as dev-only and left behind.

## 6b. Remaining questions for Architecture to resolve in `02-ARCHITECTURE.md`

These are not CEO-level decisions — they're implementation-design questions the CEO's resolutions
above hand to Architecture:

1. Exact mechanism for reconciling `translateSentence()` and `translateSentenceReliable()` (e.g.,
   does the `legacy-translation` adapter wrap the gold master's function with Milestone 2's
   retry/backoff/cache/concurrency layered around it, or some other composition?), and confirmation
   that `translationRowTemplate()`/`retrySentence()`/`saveCurrent()`'s SAVE-1 guard still receive
   the `{ko,status,source}` shape they depend on.
2. Whether/how the Multi-AI router's `providerRegistry`/`aiRouter`/adapters are re-expressed to fit
   the gold master's file structure (given CEO permission to adapt rather than copy verbatim), and
   where in the gold master's ~4,400 lines that section is inserted.
3. Concrete branch name and starting point for the new development line (CEO's original suggestion,
   `multi-ai-v2-latest-dev` off current `multi-ai-v2-dev` HEAD `bc204cf`, is available but not
   mandated).
4. Whether any `multi-ai-v2-dev` `localStorage` saved-study-set data needs a one-time migration into
   the gold master's IndexedDB schema, or is left behind as dev-only per §6a item 6.
5. Documentation of CDN dependency implications per §6a item 4 (security/availability/offline/
   privacy/fallback), and whether any mitigation (e.g., a documented offline-degradation behavior)
   is needed versus purely descriptive documentation.
6. Whether Development should be split into multiple Developer Tasks within Milestone 03 (per §6a
   item 3), and if so, a proposed task breakdown for `03-IMPLEMENTATION-LOG.md`.

## 7. Status

**APPROVED by the CEO — 2026-08-12.** Architecture Design (`02-ARCHITECTURE.md`) may begin, scoped
to resolving §6b within the CEO's resolutions in §6a.

---

## Handoff — Milestone 3 (Gold Master Adoption)

- **Milestone:** Milestone 3 — Gold Master Adoption (Multi-AI V2) (numbering provisional, see §1
  note and Open Question 1).
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/PM.md`, `DEVELOPMENT_PLAN.md`, `docs/milestones/milestone-02/README.md`,
  `docs/milestones/milestone-02/06-RELEASE-NOTES.md` (tail), `docs/MULTI_AI_ARCHITECTURE_V2.md`,
  current `index.html` and `LATEST_GOLD_MASTER_NEXT.html` (function-level diff, CDN/key scan).
- **Scope completed:** Ran Phase A (baseline: checksum, size, function-level diff between the two
  files) read-only, with no modification to either file. Drafted this PM-Spec: goal, in/out of
  scope, provisional acceptance criteria, and six open questions the CEO needs to resolve before
  Architecture can proceed. No code changed, no branch created, no commit made — per `.ai-company/PM.md`,
  a PM session does not implement.
- **Files changed:** `docs/milestones/milestone-03/01-PM-SPEC.md` (new, this file). Not yet
  committed — pending CEO review per this repo's practice of committing docs after CEO input is
  incorporated (see `.ai-company/GIT_RULES.md` if a docs-only commit is wanted before that; not
  assumed here).
- **Commits created:** None this session.
- **Tests performed:** N/A (PM phase). Repository-state findings in §2 were verified directly via
  `shasum`, `wc`, and `grep`-based function/CDN/key scans against both files, not assumed.
- **Unresolved risks:** All six items in §6. Most consequential: Open Questions 3, 5, and 6 — if
  the CEO wants this done as a single 30–45 minute Developer session as originally ordered, that is
  very unlikely to be achievable safely given the scope found in §2, and should be said plainly
  rather than attempted and rushed.
- **Next agent:** CEO, to resolve §6 and approve or revise this spec.
- **Explicit stop point:** Awaiting CEO approval (or revision request) on this `01-PM-SPEC.md`
  before Architecture Design (`02-ARCHITECTURE.md`) begins. No Phase B–E work (branch creation,
  code migration, testing, commits) has been performed, per the CEO gate in `.ai-company/WORKFLOW.md`.

## Handoff — Milestone 3, CEO approval incorporated

- **Milestone:** Milestone 3 — Gold Master Adoption (Multi-AI V2).
- **Source documents read this session:** this document's own prior draft (already-read
  `CLAUDE.md`/`WORKFLOW.md`/`AGENTS.md`/`PM.md`/`GIT_RULES.md`/`HANDOFF_PROTOCOL.md` from the prior
  PM session in this same conversation, re-confirmed via `.ai-company/GIT_RULES.md` and
  `.ai-company/HANDOFF_PROTOCOL.md` read directly this session), and the CEO's decision message
  (chat, 2026-08-12) resolving all six §6 open questions.
- **Scope completed:** Updated `01-PM-SPEC.md` in place: incorporated the CEO's six resolutions
  into §3 (In scope), §4 (Out of scope), §5 (Acceptance criteria), added §6a (CEO resolutions) and
  §6b (remaining Architecture-level questions derived from those resolutions, not new CEO-level
  questions), and changed Status (§7) from DRAFT to APPROVED. No application code touched, no
  branch created, no code-level decisions made — per `.ai-company/PM.md`, a PM session describes
  *what*, not *how*.
- **Files changed:** `docs/milestones/milestone-03/01-PM-SPEC.md` (edited in place).
- **Commits created:** None this session. Per `.ai-company/GIT_RULES.md`, nothing in that file
  mandates a commit at this specific gate, and the CEO's instruction this session was explicit not
  to commit unless governance rules specifically require it — they don't, so none was made. This
  spec and the prior QA/Architecture-precedent (`docs/milestones/milestone-02/01-PM-SPEC.md`'s
  history) suggest PM-Spec commits in this repo have historically happened as part of a later,
  bundled documentation commit rather than immediately at PM approval — leaving that pattern intact
  rather than deviating from it unilaterally.
- **Tests performed:** N/A (PM phase).
- **Unresolved risks:** None new. §6b lists six Architecture-level questions that are now
  Architecture's to resolve, not open CEO decisions — flagging here only so Architecture doesn't
  mistake them for still-open CEO items.
- **Next agent:** Software Architect.
- **Explicit stop point:** Architecture Design (`02-ARCHITECTURE.md`) begins next, time-boxed to
  ~20–30 minutes per the CEO's instruction, scoped to resolving §6b within the constraints the CEO
  fixed in §6a. This PM session stops here — no Architecture, Developer, QA, or Release Manager work
  performed.
