# 06-RELEASE-NOTES.md — Milestone 3: Gold Master Adoption + Multi-AI Migration

**Status (updated, metadata-sync session): `multi-ai-v2-latest-dev` has been pushed to the real
GitHub `origin` (`https://github.com/ohcha1/sat-study-2026.git`) and a Draft PR (#1) into `main` is
open. `main` has NOT been merged and remains at `d490e1a`. See "Push result" and "Pull request"
below for the current, factual state — this supersedes the original "awaiting CEO push-approval
gate" line below, which is preserved for history rather than deleted, per
`.ai-company/WORKFLOW.md`'s append-not-hide rule.**

*(Original packaging-session status, preserved: "Packaged, awaiting CEO push-approval gate. Not
pushed, not merged.")*

Per `.ai-company/RELEASE_MANAGER.md`: confirmed `05-REVIEW-REPORT.md` shows **APPROVE WITH
CONDITIONS** (not a plain "Approved," but not a rejection either — the one condition was a
procedural `DEVELOPMENT_PLAN.md` update, satisfied as part of this same packaging session; see
"Definition of Done verification" below) before starting this session.

## File roles (clarified, metadata-sync session)

- **`index.html`** — the active Release Candidate application. This is the file to open/launch/
  deploy. It carries the Gold Master's content plus this milestone's translation-reliability,
  Multi-AI router, and persistence changes.
- **`LATEST_GOLD_MASTER_NEXT.html`** — an immutable reference/baseline copy only, preserved
  byte-for-byte throughout this milestone (checksum
  `a51c603348b9b6b507787db4807dd3d0e54ac22ee42407d4afc2070d963162ba`, unchanged since it was first
  committed in `b14c179`). **Not** the file end users or QA should launch — it exists purely so the
  exact baseline this milestone started from remains independently verifiable.

## User-facing summary

The app's development line now starts from the CEO-supplied, much larger Gold Master build —
carrying forward its on-device Chrome-based translation, local account/IndexedDB study-history
storage, photo/PDF passage import, and richer SAT-question and grammar engines — while keeping the
translation reliability work already built for this app: automatic fallback across four translation
sources (on-device Chrome → MyMemory → Lingva → offline phrase map), per-sentence retry on failure,
and session caching. A routing layer (Multi-AI V2) now sits in front of translation so future AI
providers can be added without touching the translation logic itself; a summary-generation provider
(Gemini, model `gemini-3.1-flash-lite`) is wired in and has now passed a live API smoke test
(`success=true`, `error=null`, ~1261ms observed latency; the API key used was browser-session-only
and was never committed to the repository), but remains inactive by default — no UI call site
invokes it yet and it still requires a developer-provided key via `window.SAT_STUDIO_DEV_KEYS`, so
there is no behavior change for end users from that provider today. Two reliability bugs surfaced
during testing were
fixed before release: a rare case where translation could get permanently stuck if the on-device
translator failed to start, and a rare case where saving two study sets in very quick succession
could cause one to silently disappear.

## Technical changelog

Full Milestone 3 range on `multi-ai-v2-latest-dev` (`ae3d3eb`..`9c4cc73`), per
`03-IMPLEMENTATION-LOG.md`:

- `ae3d3eb` — Docs: PM-Spec + Architecture (CEO-approved)
- `b14c179` — Dev Task 1: seed `index.html` from `LATEST_GOLD_MASTER_NEXT.html`
- `29523bb` — Dev Task 2: `translateSentenceReliable()` reconciliation (Chrome primary + MyMemory 429-stop, preserved; Lingva fallback, retry/backoff, structured status, added)
- `43b2ccf` — Dev Task 3: per-row translation loading/error/retry UI, progressive sequential repaint
- `b90362b` — Dev Task 4: Multi-AI V2 router (`providerRegistry`/`aiRouter`/`legacy-translation`/dormant `gemini` adapter)
- `89184a7` — Dev Task 5: persistence carry-overs (`getSavedList()`, SAVE-1 guard, `renderSaved()` XSS-escaping fix)
- `dae5dce` — Docs: implementation log
- `9c4cc73` — Hot-fix: Chrome Translator init timeout/fail-open (Risk A), collision-safe save IDs (Risk B)

QA and review documents (this same range of work): `04-QA-REPORT.md` (PASS after hot-fix, with a
focused re-verification pass also recorded PASS), `05-REVIEW-REPORT.md` (APPROVE WITH CONDITIONS).

## Definition of Done verification (`.ai-company/DEFINITION_OF_DONE.md`)

| Item | Status |
|---|---|
| Acceptance criteria pass | ✅ All 10 PM-Spec acceptance criteria pass — AC #3 initially failed under a reproducible Chrome-hang condition, fixed by the Risk A hot-fix, then independently re-verified pass. |
| Regression tests pass | ✅ Full regression sweep clean at Development, QA, the focused re-verification, and Principal Review (direct diff confirms zero unintended changes anywhere outside the scoped regions). |
| No unresolved Critical/High defects | ✅ Risk A (High) and Risk B (data-loss/Critical-by-taxonomy) both fixed and independently re-verified resolved. Risk C (Medium) is logged and deferred, per this row's own "may be logged and deferred" allowance. |
| Reviewer approves | ✅ `05-REVIEW-REPORT.md` — Approve with Conditions; the one condition (this document) is satisfied by this packaging session. |
| Documentation is updated | ✅ `03-IMPLEMENTATION-LOG.md` complete; `DEVELOPMENT_PLAN.md` now records this milestone's completion, branch/commit, the numbering-collision resolution, deferred Risk C, and the Gemini live-test caveat (this session). No other user-facing docs (e.g. `README.md`) reference this app's feature set in a way this milestone changed. |
| Release notes exist | ✅ This document. |
| CEO approves push | ✅ **Branch push approved and executed** — explicit, scoped CEO authorization ("push ONLY `multi-ai-v2-latest-dev` to the GitHub origin") was granted and `multi-ai-v2-latest-dev` is now on `origin`. ⚠️ **Merge into `main` is a separate, still-unrecorded approval** — a Draft PR (#1) is open for that future decision; `main` has not been touched. |

## GIT_RULES.md compliance verification

- **Current branch:** `multi-ai-v2-latest-dev`, at `43d68d96178b5295dadc35b2637f071e0cfd069f` (as of
  the metadata-sync session; was `9c4cc731ed6e283964fb8d3095106884ed67d02f` when this section was
  first written, before the documentation-packaging commit `43d68d9` was added).
- **Remote `origin` (updated, metadata-sync session):** the local bundle file originally at `origin`
  was renamed to `bundle-backup` (preserved, unchanged, not deleted) and `origin` now points to the
  real GitHub repository, `https://github.com/ohcha1/sat-study-2026.git`. `multi-ai-v2-latest-dev`
  has been pushed there and now exists as `origin/multi-ai-v2-latest-dev`.
  *(Original text, preserved for history: "a local bundle file..., not a GitHub URL — this is not
  yet a real GitHub-hosted repository... Any 'push' from this environment today would write to that
  local bundle path, not to GitHub.")*
- **No force-pushes.** Only `git push -u origin multi-ai-v2-latest-dev` has been executed. `main`
  has never been pushed to, merged into, or force-pushed from this branch's work.
- **No secrets/credentials found:** independently re-confirmed this session via pattern search
  across the full `bc204cf..HEAD` diff — none found (consistent with every prior QA/Review pass).
- **Working tree:** clean of application-code changes at the time of this check — `index.html` and
  `LATEST_GOLD_MASTER_NEXT.html` are unmodified by this session (confirmed via `git status`/`git
  diff` before writing this file). Untracked-but-expected: `docs/milestones/milestone-03/{04-QA-REPORT,05-REVIEW-REPORT}.md`
  (written in prior QA/Review sessions this milestone, not yet committed — bundled into this
  session's single documentation commit below, together with this file and the
  `DEVELOPMENT_PLAN.md` update, since they collectively represent one logical change: finalizing
  this milestone's documentation record ahead of the push-approval request) and `.claude/launch.json`
  (Developer's local test-server config, not part of the milestone's scope — left untouched,
  uncommitted, not blocking).

## CEO push approval

- **Requested:** packaging session (branch push scope only).
- **Decision:** **Approved and executed**, in a later session. CEO statement authorized, verbatim
  in scope: "Explicit approval is granted to push ONLY `multi-ai-v2-latest-dev` to the GitHub
  origin," with explicit exclusions recorded at the time: no push of `main`, no merge, no force
  push, no branch deletion, no code/documentation changes, no additional commits, no changes to
  Gold Master files.
- **Merge-to-`main` approval:** **still not recorded.** The push approval above covers the branch
  push only, not a merge. A Draft PR (#1) is open for that future, separate decision.

*(Original text, preserved for history: "Not yet recorded... no push executed as part of this
packaging session.")*

## Push result

**Branch push: SUCCESS.** Pre-push safety checks (branch, HEAD, `origin` URL, live `origin/main`
HEAD, clean working tree, no staged secrets) all passed before `git push -u origin
multi-ai-v2-latest-dev` was executed.

- Remote branch: `origin/multi-ai-v2-latest-dev`, HEAD `43d68d96178b5295dadc35b2637f071e0cfd069f`
  (exact match to local HEAD at push time).
- `origin/main` before and after this push: unchanged, `d490e1aadfe55b0314a4179f024b5734b0d8abee`.
- No secrets/credentials committed or pushed (independently re-confirmed at push time and again
  this session).

## Pull request

**Draft PR #1** opened: `multi-ai-v2-latest-dev` → `main`, at
`https://github.com/ohcha1/sat-study-2026/pull/1`. State: `OPEN`, `isDraft: true`. Not merged, no
auto-merge enabled. This is the mechanism by which the CEO's eventual merge-approval decision (see
above) would be acted on — opening it did not itself constitute that approval.

## Handoff

- **Milestone:** Milestone 3 (Gold Master Adoption track) — Gold Master Adoption + Multi-AI Migration.
- **Source documents read:** `CLAUDE.md`, `.ai-company/WORKFLOW.md`, `.ai-company/AGENTS.md`,
  `.ai-company/RELEASE_MANAGER.md`, `.ai-company/GIT_RULES.md`, `.ai-company/DEFINITION_OF_DONE.md`,
  `docs/milestones/milestone-03/01-PM-SPEC.md`, `02-ARCHITECTURE.md`, `03-IMPLEMENTATION-LOG.md`,
  `04-QA-REPORT.md`, `05-REVIEW-REPORT.md`, `DEVELOPMENT_PLAN.md`.
- **Scope completed:** Verified `git status`/branch/HEAD/remote, confirmed no secrets in the full
  milestone diff, wrote this release-notes document, and updated `DEVELOPMENT_PLAN.md` to record
  this milestone's completion, branch/commit, the "Milestone 3" numbering collision (resolved by a
  clearly labeled, separate entry rather than rewriting the pre-existing "Dictionary Upgrade"
  Milestone 3 section, per the CEO's original instruction not to rewrite unrelated historical
  documents), the deferred Risk C backlog item, and the Gemini adapter's not-yet-live-tested status.
  No application code touched.
- **Files changed:** `docs/milestones/milestone-03/06-RELEASE-NOTES.md` (new, this file),
  `DEVELOPMENT_PLAN.md` (updated — new Gold Master Adoption track entry inserted; existing
  "Milestone 3 — Dictionary Upgrade" section left unchanged in content/scope).
- **Commits created:** None yet this session — see the single documentation commit made immediately
  after this handoff, bundling this file, `DEVELOPMENT_PLAN.md`, and the previously-uncommitted
  `04-QA-REPORT.md`/`05-REVIEW-REPORT.md` as one logical change.
- **Tests performed:** N/A (Release Manager phase — git-state/documentation verification only, no
  code testing, per this task's explicit instruction not to re-test code).
- **Unresolved risks:** Risk C (Medium, deferred, logged). The CEO push-approval decision itself,
  plus the open question of what "push" should target given `origin` is a local bundle file rather
  than a live GitHub remote.
- **Next agent:** CEO — to record an explicit, written push-approval decision (and clarify the
  `origin`/GitHub question above) before any push or merge occurs.
- **Explicit stop point:** Blocked pending CEO push-approval. No push, no merge, performed.

## Handoff — Final Release Metadata Sync

- **Milestone:** Milestone 3 (Gold Master Adoption track), same as above.
- **Source documents read (this session):** this document and `DEVELOPMENT_PLAN.md` (both to locate
  exact text to update), `README.md` (checked for existing file-launch guidance — found none, so no
  README change made, to stay within "update only release/documentation files as needed" and avoid
  creating unrelated new documentation).
- **Scope completed:** Synced this document and `DEVELOPMENT_PLAN.md` with facts that became true in
  intervening sessions not reflected here: (1) the Gemini live API smoke test passed
  (`success=true`, `error=null`, ~1261ms, key never committed); (2) `multi-ai-v2-latest-dev` was
  pushed to the real GitHub `origin`; (3) Draft PR #1 was opened; (4) merge has not occurred; (5)
  `main` remains at `d490e1a`; (6) clarified `index.html` (active Release Candidate) vs.
  `LATEST_GOLD_MASTER_NEXT.html` (immutable reference, not for launching) file roles. Updates were
  made by appending/correcting status fields and preserving prior text inline as historical record
  (per `.ai-company/WORKFLOW.md`'s append-not-hide rule), not by silently rewriting what previous
  sessions recorded as true at the time. No application code touched, no fix for Risk C, no PR
  merge.
- **Files changed:** `docs/milestones/milestone-03/06-RELEASE-NOTES.md` (this file),
  `DEVELOPMENT_PLAN.md` (Gemini live-test status line only — see that file's own diff).
- **Commits created:** None yet this session — see the single documentation commit made immediately
  after this handoff.
- **Tests performed:** N/A (documentation sync only; the Gemini live test itself was already
  performed manually in a prior session/browser context, not re-run here, per instruction).
- **Unresolved risks:** Risk C (Medium, deferred, logged — unchanged). Merge-to-`main` approval is
  still not recorded — Draft PR #1 remains open pending that decision.
- **Next agent:** CEO — to review Draft PR #1 and record an explicit merge-approval decision (or
  further changes) when ready; no automatic action is expected from this sync.
- **Explicit stop point:** Documentation synced and committed/pushed to
  `origin/multi-ai-v2-latest-dev` only. No merge. No push to `main`. No code changes.
