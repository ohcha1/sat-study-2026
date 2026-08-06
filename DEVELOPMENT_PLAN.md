# Development Plan — SAT English Learning Studio 2026

This plan sequences the work identified in the initial project review into eight milestones. Each milestone is scoped to be independently shippable and testable before moving to the next. No feature behavior should change outside of a milestone's stated scope.

## Milestone 1 — Bug Fixes Only

Goal: fix defects in existing behavior without adding or redesigning features.

- Escape all user- and API-controlled content before inserting it via `innerHTML` (passage preview in `overview()`, sentence translations in `translation()`, saved item titles/text in `renderSaved()`) to close the DOM-based XSS holes.
- Randomize the correct-answer position in `makeQuestions()` instead of always placing it at index 0 (index 1 for Transitions), so the SAT practice questions can't be gamed by pattern.
- Wire up the "분석 난이도" (difficulty) selector or remove it — it is currently read from the DOM but never used.
- Add a confirmation step before "새 지문" (New Passage) discards unsaved input.
- Add a confirmation step before "삭제" (delete) permanently removes a saved study set.
- Disable the "지문 분석하기" button while `analyze()` is in flight to prevent duplicate/overlapping submissions.

Exit criteria: existing features behave identically, except the bugs above no longer reproduce. No new features introduced.

## Milestone 2 — Translation Reliability

Goal: make the translation feature behave predictably for passages beyond the built-in sample.

- Replace the silent "번역 서비스를 불러오지 못했습니다" placeholder (which currently renders as if it were a real translation) with a clearly labeled error/retry state.
- Add request batching/throttling for the per-sentence MyMemory API calls to avoid bursts on long passages.
- Add a local caching layer (session-level) so re-analyzing the same passage doesn't re-request identical sentences.
- Evaluate and document a fallback/secondary translation source for when MyMemory is unavailable or rate-limited.
- Add a visible loading state per sentence (not just the global status line) so partial translation progress is legible.

Exit criteria: any arbitrary English passage (not just the sample) produces either a real translation or an honest, clearly-marked failure state — never a misleading canned string.

## Milestone 3 — Dictionary Upgrade

Goal: replace the ~40-word hardcoded dictionary and grade-level heuristics with a scalable vocabulary source.

- Integrate a real dictionary data source (API or a substantially larger local word list) to replace `dictionary`, `ipaMap`, and `exampleBank`.
- Replace the length-based grade-level heuristic (`levelFor`) with a defensible frequency/CEFR-based leveling approach.
- Replace `fallbackKoreanGloss`'s suffix-guessing with real lookups where possible; keep a labeled heuristic fallback only as a last resort.
- Ensure vocabulary and IPA lookups scale to arbitrary passages without visibly degrading to "no data" for common academic words.

Exit criteria: vocabulary tab produces real, sourced definitions for a representative set of test passages outside the current dictionary's coverage.

## Milestone 4 — UI Improvements

Goal: address usability gaps surfaced in the review, without changing underlying analysis logic.

- Add a proper loading indicator during analysis (beyond the status text line).
- Improve error/empty states across tabs (translation failure, empty vocab, no saved items) for clarity.
- Review responsive layout at additional breakpoints (mobile widths below current 980px handling).
- Add basic accessibility pass: focus states, ARIA labeling for tabs/drawer, keyboard navigation for the tab bar.
- Visual polish pass on spacing/typography consistency across tabs.

Exit criteria: usability issues from the review are resolved; no analysis/feature logic changes.

## Milestone 5 — AI Tutor

Goal: replace the templated, hardcoded "analysis" (grammar insights, SAT question generation) with genuine AI-generated content, addressing the gap between the README's "AI Explanation" claim and current template-based logic.

- Design an API integration for real grammar explanation generation (replacing the two hardcoded sample-passage regexes and generic pattern fallback in `grammarInsights()`).
- Design an API integration for genuine, passage-grounded SAT question generation (replacing the fixed 10-template bank in `makeQuestions()`).
- Define cost/rate-limit handling and a graceful fallback if the AI service is unavailable.
- Add clear labeling so users understand what is AI-generated vs. static reference content (dictionary links, phrase bank).

Exit criteria: grammar notes and SAT questions are generated from the actual input passage's content rather than fixed templates, for arbitrary passages.

## Milestone 6 — User Accounts

Goal: move beyond single-browser `localStorage` persistence.

- Design authentication (sign-up/login) and decide on provider (self-hosted vs. third-party auth).
- Design a backend data model for users and their saved study sets, replacing/augmenting `localStorage`.
- Migrate save/load/search/delete flows to the account-backed store, with a migration path for existing local data.
- Add basic account settings (profile, sign-out, data export/delete for privacy compliance).

Exit criteria: a user can log in from a different device/browser and see their saved study sets.

## Milestone 7 — Study History

Goal: turn quiz results and study activity into tracked progress, dependent on accounts from Milestone 6.

- Persist SAT quiz scores per attempt, linked to the study set and user.
- Add a history/progress view (score trends over time, per-passage performance).
- Add vocabulary review tracking (e.g., words marked difficult, spaced-repetition-style resurfacing).
- Add export of history/progress data (CSV at minimum).

Exit criteria: a returning user can see their past quiz scores and vocabulary progress, not just their saved passages.

## Milestone 8 — Deployment

Goal: move from a manually-opened static file to a properly hosted, reliably reachable application.

- Stand up a minimal build/serve pipeline (static hosting for frontend; hosting for any backend introduced in Milestones 6–7).
- Serve over HTTPS to ensure `getUserMedia` (mic recording) and `SpeechRecognition` work reliably (currently at risk under `file://`).
- Add environment configuration for API keys (translation, dictionary, AI tutor) rather than embedding logic client-side where relevant.
- Set up basic CI (lint/build check) and a deployment process (e.g., GitHub Actions to a static host).
- Update README with live URL, setup instructions, and environment variable requirements.

Exit criteria: the app is reachable via a stable URL over HTTPS, with all features (including mic-dependent ones) functioning as expected for end users.

---

## Sequencing Notes

- Milestones 1–4 depend only on the current codebase and can proceed in order without external services.
- Milestone 5 (AI Tutor) can be developed in parallel with 6/7 design work but should land before 6/7 if possible, since account-backed history is more valuable once question/grammar quality is real.
- Milestone 6 (User Accounts) is a prerequisite for Milestone 7 (Study History).
- Milestone 8 (Deployment) should happen incrementally where possible (e.g., deploy after Milestone 1 so bug fixes reach users quickly) rather than being held until the very end, but is listed last as the milestone that formalizes and hardens the full deployment story.
