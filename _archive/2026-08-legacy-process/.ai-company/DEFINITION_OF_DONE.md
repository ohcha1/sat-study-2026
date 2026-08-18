# DEFINITION_OF_DONE.md

Gates the transition from Release Review to the final CEO Push Approval gate in
`.ai-company/WORKFLOW.md`. A milestone is not complete — and may not proceed to the CEO
push-approval gate — until **all** of the following are true.

- [ ] **Acceptance criteria pass.** Every acceptance criterion listed in `01-PM-SPEC.md` has been
  verified and marked pass in `04-QA-REPORT.md`.
- [ ] **Regression tests pass.** The combined regression sweep described in
  `.ai-company/TESTING_STANDARDS.md` has been run and shows no new regressions in adjacent
  features.
- [ ] **No unresolved Critical or High defects.** QA has no open Critical/High findings in
  `04-QA-REPORT.md`. Medium/Low findings may be logged and deferred with the CEO's knowledge, per
  the precedent in `DEVELOPMENT_PLAN.md`'s Milestone 1 follow-up.
- [ ] **Reviewer approves.** `05-REVIEW-REPORT.md` shows an explicit **Approved** decision from
  the Principal Reviewer.
- [ ] **Documentation is updated.** `03-IMPLEMENTATION-LOG.md` is complete and accurate,
  `DEVELOPMENT_PLAN.md` is updated to reflect the milestone's completion (matching the pattern of
  prior milestone entries), and any user-facing docs (e.g. `README.md`) affected by the change are
  updated.
- [ ] **Release notes exist.** `06-RELEASE-NOTES.md` is written by the Release Manager.
- [ ] **CEO approves push.** An explicit, recorded CEO approval exists before `git push` is
  executed.

If any box is unchecked, the milestone is not done, regardless of how much of the work "feels"
finished. The Release Manager is responsible for verifying this checklist before requesting the
final CEO gate.
