# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"The Floor — Europese Royals Trainer" is a single-page Dutch-language trivia web app for practicing European royal family trivia (modeled on the "The Floor" TV game show). The entire app — markup, CSS, and JS — lives in one file: `index.html`. There is no build step, no package.json, no framework.

## Development

There is no build/test/lint tooling. To work on the app:
- Open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `python3 -m http.server`) since `images/*.jpg` are referenced as relative paths.
- All edits happen directly in `index.html` — CSS in the `<style>` block, markup in `<body>`, logic in the trailing `<script>` block.
- There are no automated tests. Verify changes manually in a browser by playing through each of the three round types.

Images referenced by the photo round (`images/*.jpg`) are **not committed to the repo** — they are downloaded fresh on every deploy (see below). They won't exist locally unless you fetch them yourself or run the download step from `.github/workflows/deploy.yml`.

## Deployment

`.github/workflows/deploy.yml` deploys to GitHub Pages on every push to `main`:
1. Downloads each royal's photo at build time from Wikipedia/Wikimedia Commons: it resolves the current canonical filename via the Wikipedia REST summary API, then fetches the actual image via `commons.wikimedia.org/w/thumb.php` (chosen because it isn't blocked, unlike other Commons endpoints tried previously — see commit history for the trial-and-error). The Wikipedia article title for each royal is hardcoded in the `wiki` associative array in the workflow and must stay in sync with the `img:` filenames used in `PHOTO_QS` in `index.html`.
2. The job **fails the build** if any photo fails to download or isn't a valid JPEG — there's no fallback/placeholder path.
3. The whole repo directory is then uploaded as the Pages artifact and deployed as-is (`index.html` plus the freshly downloaded `images/`).

When adding/removing/renaming a royal in the photo round, update both `PHOTO_QS` in `index.html` and the `wiki` map in the workflow together — they're joined by the `images/<name>.jpg` path convention.

## App architecture (inside `index.html`)

**Data**: three parallel question banks, each an array of plain objects with a `level` (1-7, used for a Makkelijk/Gemiddeld/Moeilijk difficulty badge):
- `PHOTO_QS` — `{img, answer, alt[], hint, level}`. Free-text answer, fuzzy-matched.
- `MC_QS` — `{q, options[4], answer, uitleg, level}`. Multiple choice.
- `ASSOC_QS` — `{chips[], answer, uitleg, level}`. Chips (word associations) reveal progressively; free-text answer.

**Round flow** (global mutable state: `roundId`, `questions`, `idx`, `score`, `missed`):
`startRound(id)` shuffles and takes 20 questions → `loadQuestion()` dispatches to `loadPhoto`/`loadMC`/`loadAssoc` → each round type has its own check function (`checkPhoto`/`pickMC`/`checkAssoc`) that stops the timer, scores the answer, and calls `showReveal()` → `nextQuestion()` advances `idx` or calls `showResults()`.

**Timer**: every question has a shared 45-second countdown (`startTimer`/`stopTimer`/`resetTimer`) that changes color as it runs low and auto-fails the question via each round's `*TimeUp()` handler.

**Answer matching**: free-text rounds (photo, assoc) use `isOk()` which normalizes (lowercase, strips diacritics/punctuation via `norm()`) and accepts substring/word matches against the answer plus any `alt` aliases — not exact match only.

**Persistence**: only aggregate stats (`{rounds, correct, total}`) are saved to `localStorage` under key `tf_gs3`, shown on the home screen. Per-round question state is not persisted.

**Screens**: plain show/hide via `.screen`/`.screen.active` classes (`showScreen()`), not routing — `screen-home`, `screen-question` (which itself toggles `photo-block`/`mc-block`/`assoc-block`/`reveal-block`), and `screen-results`.
