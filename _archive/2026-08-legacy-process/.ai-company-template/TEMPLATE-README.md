# .ai-company-template

A project-agnostic, file-based multi-agent development workflow. Extracted from a working
production use and generalized for reuse on other projects: web applications, education software,
business tools, internal automation, and data-processing tools.

This template gives a new project two things:

1. A durable governance layer (`.ai-company/`) defining seven roles, a mandatory approval-gated
   lifecycle, coding/testing/git standards, and a handoff protocol — so multi-session AI-assisted
   development stays disciplined even though no single session has memory of the others.
2. Two operating modes for that lifecycle: **FULL** (the complete six-document gate sequence) and
   **LIGHT** (a compressed, token-efficient sequence for ordinary low-risk work, with hard
   escalation rules back to FULL). See `.ai-company/MODE_SELECTION.md`.

## What's in here

```
.ai-company-template/
├── TEMPLATE-README.md              this file
├── CLAUDE.md.template              entry point every session reads first
├── README.md.template              project README skeleton
├── DEVELOPMENT_PLAN.md.template    milestone-sequencing skeleton
├── .ai-company/
│   ├── AGENTS.md                   role definitions — reuse verbatim
│   ├── WORKFLOW.md                 FULL-mode lifecycle — reuse verbatim
│   ├── WORKFLOW-LIGHT.md           LIGHT-mode lifecycle
│   ├── MODE_SELECTION.md           LIGHT vs FULL decision rules
│   ├── LIGHT_CONTEXT.md.template   the live compact context file (LIGHT mode)
│   ├── CEO.md                      reuse verbatim
│   ├── PM.md.template              1 placeholder
│   ├── ARCHITECT.md                reuse verbatim
│   ├── DEVELOPER.md.template       + LIGHT-mode minimal-reading note
│   ├── QA.md.template              + LIGHT-mode minimal-reading note
│   ├── REVIEWER.md                 reuse verbatim (FULL-mode only role)
│   ├── RELEASE_MANAGER.md          reuse verbatim
│   ├── CODING_STANDARDS.md.template   generalized, project fills in specifics
│   ├── TESTING_STANDARDS.md.template  generalized, branches on test framework status
│   ├── GIT_RULES.md.template       2 placeholders
│   ├── DEFINITION_OF_DONE.md       reuse verbatim (mode-aware wording)
│   └── HANDOFF_PROTOCOL.md         reuse verbatim, generic example
└── docs/milestones/milestone-00-EXAMPLE/
    ├── README.md.template
    ├── LIGHT_CONTEXT.md            worked LIGHT-mode example
    ├── 01-PM-SPEC.md.template          FULL mode only
    ├── 02-ARCHITECTURE.md.template     FULL mode only
    ├── 03-IMPLEMENTATION-LOG.md.template   both modes (LIGHT uses compact form)
    ├── 04-QA-REPORT.md.template            both modes (LIGHT uses compact form)
    ├── 05-REVIEW-REPORT.md.template        FULL mode only
    └── 06-RELEASE-NOTES.md.template        both modes
```

## Bootstrap procedure — starting a new project from this template

1. Copy this directory's contents into the new project root:
   - `.ai-company-template/.ai-company/*` → new project's `.ai-company/`
   - `CLAUDE.md.template` → new project's `CLAUDE.md`
   - `README.md.template` → new project's `README.md`
   - `DEVELOPMENT_PLAN.md.template` → new project's `DEVELOPMENT_PLAN.md`
   - Drop the `.template` suffix as each file is filled in.
   - Do **not** copy `docs/milestones/milestone-00-EXAMPLE/` yet — it stays in
     `.ai-company-template/` as reference material and gets instantiated per milestone in step 3.
2. Fill in the **global bootstrap variables only** — the 9 rows in the "Template variables" table
   below (`{{PROJECT_NAME}}`, `{{TEST_COMMAND}}`, etc.). A one-line `sed`/script pass across the
   copied files handles these; there is no build step. Every other `{{...}}` token you'll see in
   `DEVELOPMENT_PLAN.md.template`, `README.md.template`, and the milestone document skeletons
   (`{{MILESTONE_1_GOAL}}`, `{{FEATURE_1}}`, `{{N}}`, `{{ID}}`, etc.) is a **per-content fill-in
   field**, not a global constant — write those by hand as each document is actually authored (PM
   writing a spec, QA logging a defect, and so on). A single substitution pass will not, and is not
   meant to, resolve those.
3. As CEO, fill in `DEVELOPMENT_PLAN.md`'s Milestone 1 and confirm it will run in **FULL** mode —
   the first milestone of a new project is always FULL (see `MODE_SELECTION.md`), since it's what
   establishes the project's actual coding/testing conventions that later LIGHT-mode work depends
   on. Then instantiate its document folder: copy `docs/milestones/milestone-00-EXAMPLE/` to
   `docs/milestones/milestone-01/`, drop the `.template` suffixes, and keep only the documents that
   milestone's mode actually uses (FULL: all six `0N-*.md` files + `README.md`; LIGHT: just
   `README.md`, `03-IMPLEMENTATION-LOG.md`, `04-QA-REPORT.md`, `06-RELEASE-NOTES.md`, plus
   `.ai-company/LIGHT_CONTEXT.md` at the governance level, not inside the milestone folder, until
   it's archived there on completion — see `.ai-company/WORKFLOW-LIGHT.md`). Repeat this
   instantiation step for every later milestone (`milestone-02`, `milestone-03`, ...).
4. `git init`, commit the scaffolded docs as a single documentation commit.
5. Start the first session as CEO and originate the Milestone 1 request. Normal lifecycle from
   there, per `.ai-company/WORKFLOW.md`.
6. From Milestone 2 onward, default to LIGHT mode for ordinary low-risk work; the CEO declares the
   mode per milestone. See `.ai-company/MODE_SELECTION.md` for the exact triggers that force FULL.

## Template variables

| Variable | Used in | Example |
|---|---|---|
| `{{PROJECT_NAME}}` | CLAUDE.md, README.md, DEVELOPMENT_PLAN.md | Acme Inventory Tool |
| `{{PROJECT_DESCRIPTION}}` | README.md | Internal inventory tracking for warehouse ops |
| `{{PRIMARY_SOURCE_FILES}}` | PM.md, CODING_STANDARDS.md | `src/**/*.ts`, or `index.html` |
| `{{ARCHITECTURE_STYLE}}` | CODING_STANDARDS.md | React SPA + REST API, or single-file static HTML |
| `{{TEST_COMMAND}}` | TESTING_STANDARDS.md, LIGHT_CONTEXT.md | `npm test`, `pytest`, or "none" |
| `{{TEST_FRAMEWORK_STATUS}}` | TESTING_STANDARDS.md | none / partial / full |
| `{{DEFAULT_BRANCH}}` | GIT_RULES.md | main |
| `{{REPOSITORY_URL}}` | GIT_RULES.md, RELEASE_MANAGER-related docs | github.com/org/repo |
| `{{FULL_OR_LIGHT}}` | DEVELOPMENT_PLAN.md, per milestone entry | FULL (milestone 1), LIGHT (later ones, unless a `MODE_SELECTION.md` trigger applies) |

## Design notes carried over from the source project

- Independent QA and the no-self-approval rule are non-negotiable in both modes.
- Every `[GATE]` is still a hard stop in LIGHT mode — LIGHT reduces document verbosity and reading
  burden, not approval discipline.
- `LIGHT_CONTEXT.md` is a deliberate, documented exception to the "append/update, not replace"
  convention used everywhere else: it's a live, mutable snapshot of the current milestone's state,
  kept small on purpose so Developer/QA sessions don't have to reread the full governance and
  document set for routine work. It carries a short append-only change-log footer for auditability,
  and is snapshotted to `docs/milestones/milestone-XX/LIGHT_CONTEXT.md` when the milestone closes.
