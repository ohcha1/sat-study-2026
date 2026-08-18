# CLAUDE.md — Entry Point

This is a solo personal project. There is no multi-role process, no approval gates, and no
milestone-document pipeline — that was tried before (a heavy multi-agent process, plus a separate
multi-AI provider-router design), and it produced a lot of documentation and very little shipped
feature work, so both were abandoned and removed from the repo. Don't recreate them.

## What this app actually is

A single self-contained `index.html` (no build step, no server). All logic is in one inline
`<script>` block. Real audience: people who came from Korea, working on English reading
comprehension — not literally "SAT prep." The core loop that matters most: paste/import a
passage or add a word/phrase directly → save it to the **단어장·복습 (review deck)** →
review it on a spaced schedule until it's actually memorized. Everything else (grammar
detector, SAT-style question generator, pronunciation coach, translation) is a supporting
tool around that core loop, not the main point.

## Working on this repo

1. Read `DEVELOPMENT_PLAN.md` for current priorities — it's a plain running list, not a
   milestone pipeline.
2. Make the change directly in `index.html`.
3. Sanity-check with a real browser before calling it done. `smoke_test.js` (Playwright) is a
   working example of how to load the file headlessly and exercise a feature end-to-end —
   extend it or write a throwaway script in the same style; there's no test framework wired in.
4. Commit with a plain, honest commit message describing what changed and why. Don't invent
   process ceremony (no PM specs, no QA reports, no reviewer sign-off) for a one-person project.
5. This repo pushes to GitHub only when the human operator explicitly asks for it.

## If anything is ambiguous

Ask the human operator directly. Don't design a new process framework to resolve the
ambiguity — that's the trap this project already fell into once.
