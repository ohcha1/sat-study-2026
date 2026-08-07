# WORKFLOW.md — Mandatory Development Lifecycle

This is the required sequence for every milestone in this repository. No stage may be skipped,
reordered, or collapsed into another stage, even if the same human operator is directing every
session. Each stage produces a specific document under `docs/milestones/milestone-XX/` and ends
at a defined stop point. See `.ai-company/AGENTS.md` for the full definition of each role named
below, including exactly which gates each role has authority over.

## Lifecycle

```
CEO Request
    │
    ▼
PM Specification              → 01-PM-SPEC.md
    │
    ▼
[GATE] CEO Approval  ─────────  STOP. Owner must approve the spec before Architecture begins.
    │
    ▼
Architecture Design           → 02-ARCHITECTURE.md
    │
    ▼
[GATE] CEO Approval  ─────────  STOP. Owner must approve the design before Development begins.
    │
    ▼
Development                   → 03-IMPLEMENTATION-LOG.md (+ code changes, local commits)
    │
    ▼
[GATE] QA Defect Gate  ───────  STOP if Critical/High defects found. QA Engineer (not the author
    │                            of the code) verifies against 01-PM-SPEC.md's acceptance criteria.
    ▼
Developer Corrections         → updates to 03-IMPLEMENTATION-LOG.md (+ code changes, local commits)
    │        (loop with QA until no unresolved Critical/High defects)
    ▼
[GATE] Principal Review Quality Gate  ──  STOP unless Approved. Principal Reviewer (not the
    │                                      Developer or QA who did the work) independently verifies.
    ▼
Release Review                → 06-RELEASE-NOTES.md (drafted by Release Manager)
    │
    ▼
[GATE] CEO Push Approval  ────  STOP. Owner must explicitly approve the push.
    │
    ▼
GitHub Push                   (Release Manager executes, only after the gate above)
```

## Rules

1. **No stage may be skipped.** If a stage seems unnecessary for a small change, that judgment
   belongs to the CEO, expressed by explicitly waiving the stage in writing in the relevant
   document — not by an agent deciding to skip it.
2. **Every arrow is a handoff.** Each handoff must follow `.ai-company/HANDOFF_PROTOCOL.md` and be
   recorded in the relevant milestone document.
3. **Every `[GATE]` is a hard stop.** The agent that reaches a gate produces its output, writes its
   handoff, and stops. It does not proceed past the gate itself, does not assume approval, and
   does not simulate the next role in the same session.
4. **The Developer↔QA loop repeats** until QA reports no unresolved Critical or High defects (see
   `.ai-company/DEFINITION_OF_DONE.md`). Each loop iteration is logged, not overwritten.
5. **Only the Release Manager pushes to GitHub**, and only after the final CEO gate. See
   `.ai-company/GIT_RULES.md`.
6. **Documents are append/update, not replace.** Earlier milestone documents are historical record
   and should not be rewritten to hide what actually happened, including false starts or reverted
   decisions.
7. **No role may review, verify, or approve its own work product.** Every gate in the lifecycle
   above is executed by a role other than the one that produced the artifact being gated: the CEO
   gates PM/Architect/Release Manager output; QA gates Developer output; the Principal Reviewer
   gates Developer *and* QA output independently (not just a rubber stamp of the QA report); the
   CEO gates the push independently of the Release Manager's packaging. If a session finds itself
   about to approve or sign off on something it authored earlier in the same milestone, it must
   stop and hand off to the correct role instead.
8. **Every phase transition above has an approval checkpoint**, not only the three stages marked
   CEO gate. Development is gated by QA (blocking on Critical/High defects). The QA↔Developer loop
   is gated by Principal Review (blocking on an explicit Approved decision). PM Specification,
   Architecture Design, and the final push are gated directly by the CEO. See `.ai-company/AGENTS.md`
   for each role's specific "Approval authority."

## Document map

| Stage | Owner role | Document |
|---|---|---|
| PM Specification | Product Manager | `01-PM-SPEC.md` |
| Architecture Design | Software Architect | `02-ARCHITECTURE.md` |
| Development | Senior Developer | `03-IMPLEMENTATION-LOG.md` |
| QA | QA Engineer | `04-QA-REPORT.md` |
| Principal Review | Principal Reviewer | `05-REVIEW-REPORT.md` |
| Release Review | Release Manager | `06-RELEASE-NOTES.md` |

`README.md` in each milestone folder tracks overall milestone status and links the documents above
in order.
