# WORKFLOW-LIGHT.md — LIGHT-Mode Development Lifecycle

This is the required sequence for a milestone the CEO has declared **LIGHT mode** in
`DEVELOPMENT_PLAN.md`, per `.ai-company/MODE_SELECTION.md`. LIGHT mode preserves every safety
invariant of FULL mode (role separation, independent QA, no self-approval, hard gates) while
compressing document count and required reading for ordinary low-risk work. If you have not yet
confirmed this milestone qualifies for LIGHT mode, read `.ai-company/MODE_SELECTION.md` first.

## Lifecycle

```
CEO Scope                     → .ai-company/LIGHT_CONTEXT.md (goal, acceptance criteria,
    │                            allowed/prohibited files, architecture constraints, test
    │                            command, known risks)
    ▼
[GATE] CEO Approval  ─────────  STOP. Owner approves LIGHT_CONTEXT.md's scope before Development
    │                            begins. (This is the same person approving their own scoping
    │                            output only in the narrow sense that the CEO is the sole author
    │                            of LIGHT_CONTEXT.md in the first place — the gate exists so the
    │                            scope is explicitly frozen, in writing, before Development starts
    │                            treating it as fixed.)
    ▼
Development                   → 03-IMPLEMENTATION-LOG.md (compact form) + code changes,
    │                            local commits. Developer reads only CLAUDE.md +
    │                            LIGHT_CONTEXT.md + relevant source files — see DEVELOPER.md.
    ▼
[GATE] Independent QA  ───────  STOP if Critical/High defects found, or if any escalation trigger
    │                            in MODE_SELECTION.md fires. QA reads only LIGHT_CONTEXT.md + the
    │                            diff + relevant TESTING_STANDARDS.md subsections — see QA.md.
    │                            QA is always a different session/persona than the Developer.
    ▼
Developer Corrections         → updates to 03-IMPLEMENTATION-LOG.md + code changes, local commits
    │        (loop with QA until no unresolved Critical/High defects, or until escalation fires)
    ▼
[GATE] CEO Release Approval  ──  STOP unless Approved. The CEO performs the independent
    │                             quality/scope check here directly — this substitutes for
    │                             Principal Review. Same rule applies: the CEO is checking the
    │                             Developer's and QA's work, not their own.
    ▼
Release Review                → 06-RELEASE-NOTES.md (compact form, drafted by Release Manager)
    │
    ▼
[GATE] CEO Push Approval  ────  STOP. Owner must explicitly approve the push. (Distinct from the
    │                            release-approval gate above — evaluate the release as a whole.)
    ▼
GitHub Push                   (Release Manager executes, only after the gate above)
```

## What's different from FULL mode

- No standalone `01-PM-SPEC.md` / `02-ARCHITECTURE.md` unless an architecture decision is actually
  required — if one is, stop and escalate to FULL rather than making the call inside
  `LIGHT_CONTEXT.md`.
- No standalone `05-REVIEW-REPORT.md` / Principal Reviewer session — the CEO performs that check
  directly at the release-approval gate.
- `03-IMPLEMENTATION-LOG.md`, `04-QA-REPORT.md`, and `06-RELEASE-NOTES.md` still exist, in a
  compact form — see the `docs/milestones/milestone-00-EXAMPLE/*.template` files.
- Roles read a minimal document set per task instead of the full governance/document history —
  see each role file's "LIGHT mode" section for exactly what to read.

## What's identical to FULL mode

- Every `[GATE]` above is still a hard stop: produce output, write a handoff, stop.
- Independent QA is still mandatory and still a different session/persona than the Developer.
- No role ever reviews, tests, or approves its own work product.
- `.ai-company/GIT_RULES.md` and `.ai-company/DEFINITION_OF_DONE.md` apply without modification.
- `.ai-company/HANDOFF_PROTOCOL.md`'s nine required fields are still required at every transition,
  just in the compact form shown in that document.

## `LIGHT_CONTEXT.md` handling

`LIGHT_CONTEXT.md` is a single live file at `.ai-company/LIGHT_CONTEXT.md`, updated in place as the
milestone progresses — a deliberate, documented exception to the "append/update, not replace"
convention in `.ai-company/WORKFLOW.md` rule 6, justified by the efficiency goal in
`MODE_SELECTION.md`. To preserve auditability despite being mutable:

- Its top-level fields (goal, acceptance criteria, allowed/prohibited files, constraints, commit
  hashes, open risks, test command, handoff status) always reflect current state.
- Its "Change log" footer is append-only: one line per stage transition (who, what, when, gate
  result).
- When the milestone closes (pushed, or explicitly abandoned/escalated), its final contents are
  copied to `docs/milestones/milestone-XX/LIGHT_CONTEXT.md` as a permanent snapshot before being
  reset for the next milestone.

## Escalating to FULL mid-milestone

If any trigger in `MODE_SELECTION.md` fires: stop LIGHT-mode work immediately, record the reason
and current state in `LIGHT_CONTEXT.md`'s change log, hand off to the CEO, and resume under
`.ai-company/WORKFLOW.md` from the appropriate FULL-mode phase (typically Architecture, since
Development has usually already started — the Architect designs around what's already been built,
flagging any rework needed). Do not continue producing LIGHT-format documents once escalation has
been recorded.
