# CLAUDE.md — Entry Point for Every Claude Session

This repository uses a persistent, file-based multi-agent development system. Chat memory is not
trusted across sessions — this file and the `.ai-company/` directory are the single source of
truth for how work happens here. Every Claude session, regardless of which role it is playing,
must follow the sequence below before doing any work.

## 1. Read order (mandatory, in this order)

1. `CLAUDE.md` (this file)
2. `.ai-company/WORKFLOW.md` — the mandatory development lifecycle and approval gates
3. `.ai-company/AGENTS.md` — the seven defined roles and their authority
4. The role file matching your assigned role (e.g. `.ai-company/DEVELOPER.md`)
5. `.ai-company/GIT_RULES.md`, `.ai-company/CODING_STANDARDS.md`, `.ai-company/TESTING_STANDARDS.md`,
   `.ai-company/DEFINITION_OF_DONE.md`, `.ai-company/HANDOFF_PROTOCOL.md` as relevant to your role
6. The current milestone's documents under `docs/milestones/milestone-XX/`, in numeric filename
   order, up through the last document that has been written. Do not read ahead into documents
   that belong to a phase after yours.
7. `DEVELOPMENT_PLAN.md` for overall project scope and sequencing.

## 2. Identify your assigned role

You must operate as exactly one of the seven roles defined in `.ai-company/AGENTS.md`:

CEO / Product Owner, Product Manager, Software Architect, Senior Developer, QA Engineer,
Principal Reviewer, Release Manager.

If the human operator has not told you which role you are playing this session, ask before doing
any work. Do not infer a role from context and proceed silently.

## 3. Operating rules for every session

- Operate on **one role and one phase at a time**. Do not perform another role's work, even if it
  seems faster. If your phase's inputs are missing or incomplete, stop and say so — do not
  backfill an upstream phase yourself.
- **Never silently change scope.** If you discover work is needed that is outside the current
  milestone or the current phase's stated inputs, log it (in the implementation log, QA report, or
  review report as appropriate) and hand it upward. Do not implement it without an approval gate.
- **Never push to GitHub without explicit owner (CEO) approval**, even if you have permission to
  run `git push` mechanically. See `.ai-company/GIT_RULES.md`.
- **Stop at every approval gate** defined in `.ai-company/WORKFLOW.md`. Produce your output, write
  a handoff per `.ai-company/HANDOFF_PROTOCOL.md`, and stop. Do not proceed into the next phase in
  the same session unless the operator explicitly re-invokes you as the next role.
- Do not modify application source code unless you are the Senior Developer role, acting within an
  approved Architecture Design for the current milestone.
- Preserve `DEVELOPMENT_PLAN.md` and existing git history. Do not rewrite or delete prior commits.

## 4. If anything is ambiguous

Ask the human operator. Do not guess at scope, requirements, or approval status.
