# Title Pending — Design Spec

## Overview

A browser-based word-guessing game where players deduce the title of an academic paper from its abstract. Inspired by Redactle. One puzzle per day, stats tracked locally.

Implemented as a single self-contained HTML file with vanilla JavaScript — no frameworks, no build step, no external dependencies.

## Core Gameplay

### Game Screen

1. The full abstract is displayed in readable form. Title words appearing in the abstract are intentionally visible — the abstract IS the clue source. No redaction of the abstract.
2. The title is shown with meaningful words replaced by redacted blocks. Each block displays the number of letters in the hidden word.
3. Pre-revealed tokens (visible from the start, not guessable):
   - Common English stop-words: the, a, an, of, in, on, for, and, to, is, are, was, were, with, by, at, from, that, this, it, as, or, be, has, had, have, its, but, not, no, which, their, can, do, does, into, than, then, also, been, being, between, both, each, few, more, most, other, some, such, only, over, same, so, these, those, through, under, very, will, about, after, all, because, before, could, did, during, if, may, might, much, should, still, until, up, upon, what, when, where, while, who, would
   - Any non-alphabetic tokens: numbers, hyphens, colons, parentheses, slashes, punctuation
4. Stop-words and symbols are shown in a faded/muted style to distinguish them from guessable words.

### Guessing

- Player types a word into an input field and submits (Enter key or button click).
- Matching is **case-insensitive exact match** — "neuron" matches "Neuron" but not "neurons".
- A brief rules hint is shown near the input: "Exact word match (plurals won't match singulars)".
- If the guess matches one or more unrevealed title words, those blocks are revealed in green.
- If the guess doesn't match, it's recorded as a miss.
- Duplicate guesses are rejected with a brief message (no penalty).
- Only alphabetic input is accepted; non-alpha characters are stripped. If the result is empty, the submission is silently ignored (not recorded).
- A running guess history is displayed below the input, newest first, color-coded:
  - Green: correct guesses (matched a title word)
  - Muted beige/gray: incorrect guesses

### Stats Bar

Displayed between the input and guess history:
- **Guesses:** total number of guesses made
- **Found:** X / Y words (guessable words found vs total guessable)

## Daily Puzzle Rotation

- ~100 papers hardcoded in a JavaScript array within the HTML file.
- Each paper entry: `{ title, abstract, url }`
- Daily puzzle index uses **UTC day**: `Math.floor(Date.now() / 86400000) % puzzles.length`
  - Rollover happens at UTC midnight, consistent for all timezones.
  - Deterministic: everyone gets the same puzzle on the same UTC day.
  - Cycles through the pool sequentially when exhausted.
- Puzzle number displayed in the header.

### Unsolved Games

- There is no "give up" button — players can guess as much as they want within the day.
- When a new UTC day begins and the puzzle changes, any unsolved game is recorded as a loss:
  - `played` increments, `solved` does not.
  - `history` gets an entry: `{ "solved": false, "guesses": N, "date": "..." }`
  - `currentStreak` resets to 0.
- This check happens on page load: if `titlePending_current` has a puzzle number different from today's, finalize the old game as unsolved before starting the new one.

## Win State

When all guessable words are revealed:

1. **Title area** transitions to a fully green "Solved!" state.
2. **"Read the Paper"** button appears, linking to the paper's URL/DOI.
3. **Score card** is shown:
   - Guess count
   - Words found (X/X)
4. **Shareable scorecard** — preformatted emoji text, copyable to clipboard:
   ```
   Title Pending #42 🎓
   March 24, 2026
   🟩🟩🟩🟩🟩 5/5 words
   📝 12 guesses
   ```
5. **Lifetime stats** displayed below the scorecard.

## Stats & Persistence (localStorage)

### Stored Data

Key: `titlePending_stats`

```json
{
  "played": 18,
  "solved": 15,
  "currentStreak": 5,
  "bestStreak": 7,
  "totalGuesses": 256,
  "history": {
    "42": { "solved": true, "guesses": 12, "date": "2026-03-24" },
    "41": { "solved": true, "guesses": 8, "date": "2026-03-23" }
  }
}
```

### Displayed Stats

- Games played
- Games solved
- Win rate (%)
- Current streak
- Best streak
- Average guesses (across solved games)

### Replay Prevention

- On load, check if today's puzzle number exists in `history`.
- If already played, show the completed state (revealed title, stats, scorecard) instead of the game.

### In-Progress State

Key: `titlePending_current`

```json
{
  "puzzleIndex": 42,
  "guesses": ["neural", "cortex", "brain", "learning"]
}
```

- `puzzleIndex`: today's puzzle number — used to detect day change.
- `guesses`: ordered array of all guesses made.
- Revealed state is derived from replaying `guesses` against the title on restore — no separate tracking needed.
- Restored on page reload so progress isn't lost.
- On load, if `puzzleIndex` differs from today's index, finalize the old game (see "Unsolved Games") and clear this key.

### Streak Logic

- A **streak** counts consecutive UTC days where the player solved the puzzle.
- On page load, before starting today's game, check for gaps:
  - Look at the most recent entry in `history` by date.
  - If yesterday's date has no solved entry (either missing or `solved: false`), reset `currentStreak` to 0.
- Solving today's puzzle increments `currentStreak`.
- `bestStreak` is updated whenever `currentStreak` exceeds it.

### Data Integrity

- On load, if localStorage data fails to parse, reset to default empty state rather than crashing.

## Visual Design

### Theme: Academic / Paper-like

- **Font:** Georgia, Times New Roman, serif
- **Background:** Warm off-white (#faf8f5)
- **Accent color:** Muted brown (#8b7355) for borders, labels, buttons
- **Success color:** Muted green (#5a8f5a) for correct guesses and reveals
- **Text:** Dark gray (#2c2c2c) for body, lighter for secondary text

### Layout (top to bottom)

1. **Header** — centered, game title "TITLE PENDING", subtitle, puzzle number and date. Bottom border in accent color.
2. **Title area** — left-bordered card. Label "Title". Redacted blocks with letter counts, pre-revealed stop-words/symbols in muted text. On solve: green background, "Solved!" label.
3. **Abstract** — label "Abstract", justified serif text.
4. **Input area** — text input + "Guess" button, horizontally aligned.
5. **Stats bar** — guess count and found count.
6. **Guess history** — label "Guesses", wrapped pills/tags showing all guesses color-coded.
7. **Win section** (on solve, inline below guess history) — paper link, scorecard, copy button, lifetime stats. Not a modal overlay — appears as part of the page flow.

### Redacted Blocks

- Tan/brown background (#d4c9b0)
- Rounded corners
- Letter count displayed centered within the block (e.g., "8" for an 8-letter word)
- When revealed: green background (#5a8f5a), white text showing the word

## Data Format

```javascript
const PUZZLES = [
  {
    title: "Dynamics of Neural Circuits in Developing Cortex",
    abstract: "This study investigates the formation and maturation...",
    url: "https://doi.org/10.1234/example",
  },
  // ... ~100 entries
];
```

Papers should span diverse academic fields to keep the game interesting across days.

## Tokenization

The title string is split on whitespace to produce an array of tokens. Each token is then classified:

1. **Non-alphabetic token** — matches `/^[^a-zA-Z]+$/` (e.g., "-", ":", "(2024)"). Pre-revealed.
2. **Stop-word** — the lowercase form is in the stop-words Set. Pre-revealed in faded text.
3. **Guessable word** — everything else. Shown as a redacted block.

Hyphenated compounds like "well-known" are treated as a single token (one redacted block). The player must guess the full hyphenated form. The letter count shown on the block includes the hyphen.

Parenthesized words like "(fMRI)" are treated as one token. If the inner word (stripped of parens) is alphabetic, it's guessable — the player guesses "fMRI" (without parens) and the full token is revealed.

## Stop-Words List

A Set of common English stop-words checked against each title word. Words in the set are rendered as pre-revealed faded text. The stop-words list should be generous — the fun is in guessing the meaningful/technical words, not articles and prepositions.

## Technical Notes

- Single HTML file, all CSS and JS inline.
- No external requests, no CDN, no fonts loaded — fully offline-capable.
- Responsive: should work on mobile (stacked layout, appropriately sized touch targets).
- Clipboard API (`navigator.clipboard.writeText`) for scorecard copying. Fallback: create a temporary textarea, select, and `document.execCommand('copy')`. Show "Copied!" confirmation either way.
- All game logic runs client-side.

## Out of Scope

- User accounts or server-side anything
- Hint system
- Difficulty levels
- Multiplayer / leaderboards
- Dynamic paper loading from APIs
