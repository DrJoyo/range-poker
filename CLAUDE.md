# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Heads-up No-Limit Texas Hold'em "range poker" trainer. Players don't know their own cards — instead, each turn they assign a poker action (check/call/bet/raise/fold) to ranges of starting hands. The game narrows each player's range as streets progress and reveals both hands at showdown or on a fold.

## Running the app

```bash
npx serve -p 5500 .
```

Then open `http://localhost:5500`. The entire app is a single file: `index.html`. There is no build step, no dependencies, no package.json. Hard-refresh (`Cmd+Shift+R`) after edits — the dev server may cache.

## Architecture

Everything lives inside one IIFE exported as `window.RP`. Structure within the script tag:

**Constants** — `RANKS`, `SUITS`, `RVAL`, `SUITED_SUITS`, `OFFSUIT_SUITS`, `PAIR_SUITS`, `suitFilter`

**Game state (`G`)** — single mutable object created by `newMatch()`. Key fields:
- `ranges[pid][r][c]` — map of `comboKey → action` for every cell in the 13×13 chart
- `narrowed[pid]` — `Set<comboKey>` of combos still in a player's range after they've acted; `null` means "full range". Persists across streets within a hand; reset to `null` only at `dealHand()`.
- `board` — array of dealt community cards (excluded when building ranges to keep combo count at 1326, not 1225)
- `actPid`, `street`, `curBetTo`, `lastRaiseAmt`, `bets`, `pot`, `stacks`

**Hand chart representation**
- The 13×13 grid: `r < c` = suited, `r > c` = offsuit, `r === c` = pairs (ranks indexed 0=A … 12=2)
- Each cell is subdivided into suit subrectangles: suited (4), offsuit (12), pairs (6)
- `htSuitKey(r, c, suit)` maps a grid cell + suit-pair string (e.g. `"sh"`) to a canonical 4-char combo key
- `cKey(c1, c2)` produces a canonical combo key (cards sorted by rank-index×4+suit-index)
- `combosFor(ht)` enumerates all raw combos for a hand-type string like `"AKs"` or `"72o"`

**Turn flow**
1. `startTurn(pid)` → `initRange(pid)` (builds/rebuilds ranges respecting `narrowed`) → `showActionPanel(pid)`
2. Player drag-selects cells on their chart; action buttons assign the chosen action to `drag.cells`
3. `submit()` → `execAction()` → updates `G.narrowed[pid]` from submitted ranges → `checkAdvance()` → next turn or next street

**Range narrowing** — after a player submits, `G.narrowed[pid]` is set to the union of all combo keys that got assigned to the action they actually took. `initRange` checks this set on subsequent turns so grayed-out (eliminated) combos are excluded from the map entirely.

**Drag selection** — `drag.cells` is a `Set<"r,c">` of selected chart cells. Selecting adds/removes from this set; action button click calls `assignSelection(pid, action)` which applies the suit filter and clears `drag.cells`. Category buttons (`selectCategory`) add to `drag.cells` rather than directly assigning.

**Hand evaluator** — `evalHand(cards)` picks best 5 from 7 (or 6) cards via `eval5`. Rank thresholds: SF≥8e8, Quads≥7e8, FH≥6e8, Flush≥5e8, Straight≥4e8, Trips≥3e8, TwoPair≥2e8, Pair≥1e8.

**Hand type categories** — `boardAnalysis(board)` computes rank/suit counts. `getHandCategories(pid, board)` returns visible category buttons with combo counts; straight/flush draws are suppressed on the river and when not structurally possible on the board. OESD vs gutshot: missing rank at the low end of a window is only OESD if `lo+4 < 14` (Ace not at top); missing at high end only OESD if `lo > 1` (Ace-low not at bottom).

## Key invariants

- Opponent hole cards are **never** excluded from `initRange` (would leak info and reduce 1326→1225)
- `G.narrowed` is reset to `[null, null]` only in `dealHand()`, not between streets
- Suit filter (`suitFilter`) affects drag assignment and category selection but not category combo counts (counts scan all suits)
- `drag.cells` is cleared by `deselectAll()` which is called at the top of `submit()` and on the "Deselect All" button
