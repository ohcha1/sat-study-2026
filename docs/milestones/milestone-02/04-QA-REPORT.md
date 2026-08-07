# 04-QA-REPORT.md — Milestone 2: Translation Reliability

**Status: TEMPLATE — not started**

This document is a structured template only. Do not fill it in until an initial
`03-IMPLEMENTATION-LOG.md` entry exists. The QA Engineer role should read `.ai-company/QA.md` and
`.ai-company/TESTING_STANDARDS.md` before writing here.

Append one entry per QA pass. Do not overwrite prior entries — the Developer↔QA loop history
should remain visible.

## QA pass template

```
### QA pass: <date> — against commit(s) <hash(es)>

#### Acceptance criteria results (from 01-PM-SPEC.md)

| # | Criterion | Result | Notes |
|---|---|---|---|
| 1 |  | pass/fail |  |

#### Regression check

<what adjacent features were checked and result>

#### Defects found

| ID | Description | Severity | Repro steps | Status |
|---|---|---|---|---|

#### Handoff

<per .ai-company/HANDOFF_PROTOCOL.md — to Developer if unresolved Critical/High, else to Reviewer>
```
