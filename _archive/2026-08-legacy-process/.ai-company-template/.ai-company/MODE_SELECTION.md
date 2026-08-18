# MODE_SELECTION.md — LIGHT vs FULL

Every milestone runs in exactly one of two modes. The CEO declares the mode explicitly per
milestone in `DEVELOPMENT_PLAN.md` — it is never inferred silently by an agent, and it can change
mid-milestone only via an explicit escalation (LIGHT → FULL; never the reverse without a fresh CEO
decision).

## Default

- **A brand-new project's first milestone always runs FULL.** It's what establishes the project's
  real coding/testing conventions and architecture — there is nothing yet for LIGHT mode's
  compressed scoping to rely on.
- **After that, ordinary low-risk milestones default to LIGHT.** FULL becomes the exception,
  reserved for the trigger conditions below.

## FULL is mandatory when any of these apply (classify up front)

- The change touches authentication, payments, or PII.
- The change involves a persistent data-schema change or migration.
- The change is destructive or not easily reversible.
- The change introduces or alters behavior of an external API integration or credential.
- The change spans multiple modules/files in a way that requires an actual architecture decision
  (not just "which files to touch," but "how should this be structured").
- It's the first milestone of a new project (see above).
- The CEO flags it high-risk for any other reason — this is always sufficient on its own.

## Escalate immediately from LIGHT to FULL, mid-milestone, if

- QA finds a Critical or High severity defect.
- The work needs to touch a file outside `LIGHT_CONTEXT.md`'s "files allowed to change" list.
- An actual architecture decision turns out to be necessary (not anticipated when scoped as LIGHT).
- Any of the mandatory-FULL trigger conditions above is discovered after the milestone started.
- **The agent is uncertain whether the change is still low-risk.** Uncertainty defaults to
  escalation, not to continuing in LIGHT.

To escalate: stop LIGHT-mode work, record the reason in `LIGHT_CONTEXT.md`'s change log, hand off
to the CEO, and resume as a FULL-mode milestone from whatever phase is appropriate (usually
Architecture, since Development has already started) — do not silently keep working in the
compressed LIGHT format once a trigger has fired.

## What LIGHT mode does NOT relax

- Independent QA — always a different session/persona than the Developer.
- No self-approval, ever, in either mode.
- Every `[GATE]` in `.ai-company/WORKFLOW-LIGHT.md` is still a hard stop.
- `.ai-company/GIT_RULES.md` and `.ai-company/DEFINITION_OF_DONE.md` apply in full.

LIGHT mode only relaxes: how many standalone documents a milestone produces, and how much of the
governance/document set each role has to reread for routine work. See
`.ai-company/WORKFLOW-LIGHT.md`.

## Efficiency goal

LIGHT mode should normally use substantially less context and fewer agent turns than FULL mode for
comparable work — as a rough target, 50% or more reduction in routine-workflow token/turn usage
where practical. This comes from three sources, not from cutting corners on safety:

1. `LIGHT_CONTEXT.md` replaces rereading the full FULL-mode document set on every task.
2. PM and Architecture phases collapse into CEO scoping when no architecture decision is needed.
3. Principal Review is replaced by a direct CEO release-approval check, skipping a full role
   session.

If achieving the efficiency goal would require skipping independent QA, self-approval, or a hard
gate, the efficiency goal loses — escalate to FULL instead.
