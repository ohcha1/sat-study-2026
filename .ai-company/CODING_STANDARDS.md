# CODING_STANDARDS.md

Applies to all application source code changes made by the Senior Developer role during the
Development stage of `.ai-company/WORKFLOW.md`. This document describes standards for this
specific repository (a single-file static HTML/CSS/JS application at `index.html`), not a generic
style guide.

## General principles

- Match the existing code's conventions before introducing new ones. This codebase is currently a
  single `index.html` with inline `<script>`/`<style>`; do not fragment it into multiple files
  unless an approved architecture explicitly calls for it (e.g., Milestone 8's build/serve
  pipeline).
- Prefer the smallest change that correctly implements the approved architecture. Do not
  refactor unrelated code "while you're in there" — file it as a follow-up note instead.
- No behavior changes outside the current milestone's approved scope, per `DEVELOPMENT_PLAN.md`'s
  standing rule.

## Security

- Never insert user- or API-controlled content into the DOM via `innerHTML` without escaping.
  This codebase has a documented history of XSS defects from this exact pattern (see
  `DEVELOPMENT_PLAN.md` Milestone 1) — treat any new `innerHTML` usage as a defect risk by default
  and prefer `textContent`, explicit escaping, or a shared escape helper.
- Never commit API keys, tokens, or credentials into source. See `.ai-company/GIT_RULES.md`.
- Validate/handle `JSON.parse` and `localStorage` reads defensively — corrupted or missing local
  state should degrade gracefully, not throw.

## JavaScript conventions

- Match existing naming and function style already present in `index.html` (camelCase functions,
  small single-purpose helpers).
- Guard against duplicate/overlapping async operations that mutate shared state (see the
  `Date.now()` id-collision and in-flight-button defects fixed in Milestone 1 as the pattern to
  avoid repeating).
- Prefer explicit error/loading states over silent fallbacks that could be mistaken for real data
  (see Milestone 2's translation-failure requirement).

## Comments and TODOs

- A deliberate no-op or deferred feature (e.g., the difficulty selector deferred to Milestone 3)
  must have a TODO comment in the code stating what it does today and which milestone will address
  it — do not leave silent dead code.

## Scope discipline

- If implementing the approved architecture reveals a necessary change outside its stated file
  list, stop and flag it in `03-IMPLEMENTATION-LOG.md` and back to the CEO/Architect rather than
  expanding scope unilaterally.
