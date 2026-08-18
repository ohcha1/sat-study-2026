# English Reading & Vocabulary Studio

A single self-contained web app (`index.html`, no build step, no server — just open it in a
browser) for building English reading comprehension and vocabulary, made for people who came
from Korea and are working on real English fluency. It's framed with SAT-style reading
passages and question types because that's a convenient, well-structured source of practice
material — not because the target user is literally studying for the SAT.

## Core loop

The main thing this app is for: capture words and expressions you don't know yet, and actually
review them until you do.

1. Paste or import a passage (or add a word/phrase directly).
2. Save unfamiliar words and expressions to your **단어장·복습 (review deck)** with one click.
3. Review saved items on a spaced schedule — a 5-box Leitner system (1 / 3 / 7 / 14 / 30 days).
   Recall correctly and the interval grows; miss it and it resets, so real gaps in memory get
   more repetition automatically.

Everything else below supports that loop — none of it replaces it.

## Other features

- **Grammar detector** — identifies real grammatical structures in a passage (appositives,
  relative clauses, passive voice, parallel structure, and more).
- **SAT-style question generator** — generates reading-skill questions (central idea, inference,
  cause/effect, vocabulary-in-context, etc.) from whatever passage you're working with.
- **Pronunciation coaching** — practice pronunciation with feedback.
- **PDF / OCR / image import** — bring in passages from PDFs, scanned images, or photos.
- **Accounts + local history** — everything (review deck, study sessions, saved passages, past
  attempts) is stored per account in the browser's IndexedDB, no external server involved.

## Status / honesty notes

- The "AI Explanation" feature is currently template- and rule-based, not a live LLM call — see
  `DEVELOPMENT_PLAN.md` for the plan to either rename or actually wire it up.
- The built-in vocabulary dictionary is small (roughly 40 words) and needs to grow.

## Working on this project

See `CLAUDE.md` for how this repo is meant to be worked on, and `DEVELOPMENT_PLAN.md` for the
current priority list. There is no multi-role process and no milestone-document pipeline — an
earlier attempt at that (and a separate multi-AI provider-router design detour) produced a lot of
documentation and very little shipped feature work, so both were abandoned and removed from the
repo. Not a pattern to repeat.
