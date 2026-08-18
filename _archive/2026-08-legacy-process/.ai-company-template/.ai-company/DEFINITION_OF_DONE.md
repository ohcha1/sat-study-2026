# DEFINITION_OF_DONE.md

Gates the transition to the final CEO Push Approval gate. A milestone is not complete — and may
not proceed to the CEO push-approval gate — until **all** of the following are true, in either
workflow mode.

- [ ] **Acceptance criteria pass.** Every acceptance criterion (from `01-PM-SPEC.md` in FULL mode,
  or `LIGHT_CONTEXT.md` in LIGHT mode) has been verified and marked pass in `04-QA-REPORT.md`.
- [ ] **Regression tests pass.** The regression sweep described in `.ai-company/TESTING_STANDARDS.md`
  has been run and shows no new regressions in adjacent features.
- [ ] **No unresolved Critical or High defects.** QA has no open Critical/High findings in
  `04-QA-REPORT.md`. Medium/Low findings may be logged and deferred with the CEO's knowledge.
- [ ] **Independent approval recorded.** FULL mode: `05-REVIEW-REPORT.md` shows an explicit
  **Approved** decision from the Principal Reviewer. LIGHT mode: the CEO's release approval is
  explicitly recorded in `LIGHT_CONTEXT.md`.
- [ ] **Documentation is updated.** `03-IMPLEMENTATION-LOG.md` is complete and accurate,
  `DEVELOPMENT_PLAN.md` is updated to reflect the milestone's completion, and any user-facing docs
  (e.g. `README.md`) affected by the change are updated.
- [ ] **Release notes exist.** `06-RELEASE-NOTES.md` is written by the Release Manager.
- [ ] **CEO approves push.** An explicit, recorded CEO approval exists before `git push` is
  executed.

If any box is unchecked, the milestone is not done, regardless of how much of the work "feels"
finished. The Release Manager is responsible for verifying this checklist before requesting the
final CEO gate.
