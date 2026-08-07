# GIT_RULES.md

Mandatory for every role that touches git in this repository.

## Rules

1. **`main` remains stable.** Every commit to `main` should leave the application in a working
   state. Do not commit known-broken intermediate states.
2. **No force push, ever.** History is never rewritten on a shared branch.
3. **No secret, credential, or token in any file, ever.** This includes API keys, auth tokens,
   `.env` contents, personal access tokens, or anything resembling them — even in comments, test
   scripts, or documentation examples. Verify before every commit.
4. **One logical change per commit.** Match the granularity already established in this
   repository's history (see `git log` — each Milestone 1 commit is a single fix with a
   descriptive message naming the defect and its class, e.g. `Fix: XSS in SAT tab — ...`).
5. **Check the working tree before and after work.** Run `git status` before starting so you know
   your baseline, and after finishing so you know exactly what you're committing. Never commit
   unrelated or accidental changes.
6. **Local commits first.** All commits happen locally. Nothing is pushed to GitHub except by the
   Release Manager, and only after the CEO's push-approval gate (see `.ai-company/WORKFLOW.md`).
7. **GitHub push only after CEO approval**, recorded in writing in `06-RELEASE-NOTES.md` (or the
   relevant approval document) before the push command is run.
8. **Every commit hash is recorded** in `03-IMPLEMENTATION-LOG.md` (Development/QA-correction
   commits) as it's made. Do not batch-record from memory after the fact — record each hash
   immediately after committing.
9. **Documentation-only commits are still one logical change per commit** — don't mix a docs
   commit with a source change, and vice versa.
10. **Commit messages** should follow the existing convention in this repo's history: a short
    prefix (`Fix:`, `Add:`, `Docs:`, etc.) followed by a specific, concrete description — not
    generic messages like "updates" or "wip".

## Pre-commit checklist (every role, every commit)

- [ ] `git status` reviewed — only intended files are staged.
- [ ] `git diff --cached` reviewed for secrets/credentials/tokens.
- [ ] No application source file modified outside approved scope.
- [ ] Commit message is specific and follows the existing convention.
- [ ] Commit hash recorded in the implementation log immediately after committing.
