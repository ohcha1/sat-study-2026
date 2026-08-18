# HANDOFF_PROTOCOL.md

Every transition between roles — including gate submissions to the CEO — must include a handoff
containing all of the fields below. Write the handoff at the end of your session, in the relevant
milestone document (or as a dedicated section if the document doesn't already have one), so the
next session (which will not have your chat memory) has everything it needs.

In LIGHT mode, use the compact form: all nine fields still required, but as a short block rather
than prose — see the "Handoff status" section of `LIGHT_CONTEXT.md` and
`.ai-company/WORKFLOW-LIGHT.md`.

## Required handoff fields

1. **Milestone** — which milestone (e.g., "Milestone 3 — Search Performance").
2. **Source documents read** — every document you read before starting, per `CLAUDE.md`'s read
   order and your role file's required inputs.
3. **Scope completed** — precisely what you did this session, referencing the specific items from
   `01-PM-SPEC.md` / `02-ARCHITECTURE.md` / `LIGHT_CONTEXT.md` addressed.
4. **Files changed** — full list of files created, modified, or deleted this session.
5. **Commits created** — every commit hash and message created this session (empty if none, e.g.
   for a PM/Architect/CEO/Reviewer session that produces docs only, which are also committed per
   `.ai-company/GIT_RULES.md` but as documentation commits).
6. **Tests performed** — what was verified and how, per `.ai-company/TESTING_STANDARDS.md` (QA and
   Developer roles); for other roles, what was reviewed/checked.
7. **Unresolved risks** — anything you noticed but didn't address: open questions, deferred
   defects, scope ambiguities, technical debt.
8. **Next agent** — which role should pick this up next.
9. **Explicit stop point** — a one-line statement of exactly what the next session should do first
   (e.g., "Awaiting CEO approval of `01-PM-SPEC.md` before Architecture begins").

## Example (FULL mode)

```
## Handoff — Milestone 3

- Milestone: Milestone 3 — Search Performance
- Source documents read: CLAUDE.md, WORKFLOW.md, AGENTS.md, PM.md, DEVELOPMENT_PLAN.md,
  docs/milestones/milestone-03/README.md
- Scope completed: Drafted 01-PM-SPEC.md covering all 4 bullet items from
  DEVELOPMENT_PLAN.md's Milestone 3 section as DRAFT acceptance criteria.
- Files changed: docs/milestones/milestone-03/01-PM-SPEC.md
- Commits created: none (docs-only draft, not yet committed pending CEO review)
- Tests performed: N/A (PM phase)
- Unresolved risks: no existing PM package found in repo history for this milestone; spec is
  built from DEVELOPMENT_PLAN.md only and needs CEO confirmation that scope is accurate.
- Next agent: CEO
- Explicit stop point: Awaiting CEO approval or revision request on 01-PM-SPEC.md before
  Architecture Design begins.
```

## Example (LIGHT mode, compact form)

```
## Handoff — Milestone 4 (LIGHT)

- Milestone: Milestone 4 — Fix pagination off-by-one
- Source documents read: CLAUDE.md, LIGHT_CONTEXT.md, DEVELOPER.md (LIGHT section)
- Scope completed: Fixed the off-by-one in pageOffset() per LIGHT_CONTEXT.md acceptance criterion 1.
- Files changed: src/pagination.js
- Commits created: a1b2c3d "Fix: pagination off-by-one in pageOffset()"
- Tests performed: ran {{TEST_COMMAND}}; added regression case for page 1 and last page.
- Unresolved risks: none
- Next agent: QA
- Explicit stop point: Awaiting independent QA against LIGHT_CONTEXT.md acceptance criteria.
```

## Rules

- Do not omit a field. If a field is genuinely not applicable, write "N/A" and say why.
- Do not overwrite a previous handoff — append new ones, so the document is a full history of the
  milestone's progress. (Within `LIGHT_CONTEXT.md`, the current handoff status is mutable, but each
  transition still gets a one-line entry in its change-log footer — see `WORKFLOW-LIGHT.md`.)
- A session is not finished until its handoff is written. This is what allows the next session to
  operate with zero chat memory.
