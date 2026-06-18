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
- `hole[pid]` — two-card array of dealt hole cards; used only at showdown/reveal
- `deck` — shuffled array; cards popped from the end
- `dealer` — pid of the button/SB (0 or 1); swaps each hand
- `streetChecked[pid]` — tracks whether each player checked this street; both `true` triggers `advanceStreet()`
- `streetActed[pid]` — tracks whether each player has acted this street (used for preflop limp/BB-option logic)
- `actPid`, `street`, `curBetTo`, `lastRaiseAmt`, `bets`, `pot`, `stacks`

**Hand chart representation**
- The 13×13 grid: `r < c` = suited, `r > c` = offsuit, `r === c` = pairs (ranks indexed 0=A … 12=2)
- Each cell is subdivided into suit subrectangles: suited (4), offsuit (12), pairs (6)
- `htSuitKey(r, c, suit)` maps a grid cell + suit-pair string (e.g. `"sh"`) to a canonical 4-char combo key
- `cKey(c1, c2)` produces a canonical combo key (cards sorted by rank-index×4+suit-index)
- `combosFor(ht)` enumerates all raw combos for a hand-type string like `"AKs"` or `"72o"`
- `comboToRC(ck)` maps a 4-char combo key back to `{r, c}` grid coordinates

**Turn flow**
1. `startTurn(pid)` → `buildComboStrengthIndex()` → `initRange(pid)` (builds/rebuilds ranges respecting `narrowed`) → `showActionPanel(pid)`
2. Player drag-selects cells on their chart; action buttons assign the chosen action to `drag.cells`
3. `submit()` → `execAction()` → updates `G.narrowed[pid]` from submitted ranges → `checkAdvance(pid)` → next turn or next street

**`checkAdvance(pid)`** — determines what happens after a check: preflop the SB checks first then BB gets the option (detected via `!G.streetActed[opp]`); postflop BB acts first, and if `G.streetChecked[opp]` is already true both players have checked so `advanceStreet()` is called. Bet/raise/call/fold paths skip `checkAdvance` and call `advanceStreet()` or end the hand directly.

**Street advance & showdown** — `advanceStreet()` records `prevLen = G.board.length`, pushes new cards onto `G.board`, then calls `renderBoard(prevLen)` so only the newly added cards animate in. When all streets are done or someone folds, `showdown()` / `foldWin()` populates the `#showdown-overlay` and calls `renderHole(pid, true)` to flip both players' hole cards face-up. All-in with cards to come calls `runOutBoard()` which deals remaining board cards then triggers `showdown()` after an 800 ms delay.

**Card reveal animation** — `renderBoard(newFrom)` applies a `ccReveal` CSS animation only to cards at index ≥ `newFrom`, with a 90 ms stagger per card (0 / 90 / 180 ms for the flop). Passing `null` (initial deal) skips animation entirely.

**Range narrowing** — after a player submits, `G.narrowed[pid]` is set to the union of all combo keys that got assigned to the action they actually took. `initRange` checks this set on subsequent turns so grayed-out (eliminated) combos are excluded from the map entirely.

**Drag selection** — `drag.cells` is a `Set<"r,c">` of selected chart cells. Selecting adds/removes from this set; action button click calls `assignSelection(pid, action)` which applies the suit filter and clears `drag.cells`. Category buttons (`selectCategory`) add to `drag.cells` rather than directly assigning. Default selection mode is freehand (`selMode = 'free'`).

**Hand evaluator** — `evalHand(cards)` picks best 5 from 7 (or 6) cards via `eval5`. Rank thresholds: SF≥8e8, Quads≥7e8, FH≥6e8, Flush≥5e8, Straight≥4e8, Trips≥3e8, TwoPair≥2e8, Pair≥1e8. Tiebreakers are packed into the lower digits (base-15 positional for flush/high-card; rank×multiplier for made hands).

**Hand type categories** — `boardAnalysis(board)` computes rank/suit counts. `getHandCategories(pid, board)` returns visible category buttons with combo counts; straight/flush draws are suppressed on the river and when not structurally possible on the board. OESD vs gutshot: missing rank at the low end of a window is only OESD if `lo+4 < 14` (Ace not at top); missing at high end only OESD if `lo > 1` (Ace-low not at bottom). A **"Nothing"** category is always included last for combos matching no other category; on the river the 'nothing' check in `matchesCategory` omits flush/straight draw tests so missed draws are correctly classified as nothing.

**`isPairAtRank` on paired boards** — On an unpaired board, `isPairAtRank` requires the other hole card to not hit the board (prevents double-counting with `isTwoPair`). On paired boards, `isTwoPair` is disabled, so `isPairAtRank` relaxes this: when both hole cards each pair a different *unpaired* board rank (counterfeit two pair), only the higher-ranked hole card's pair is counted. If the other hole card hits a *paired* board rank (would make a full house), it still returns false.

**Check display** — `execAction` sets `bdisp[pid].textContent = 'Check'` for postflop checks; `updateUI()` (called by `collectBets()` on street advance) clears it automatically when `G.bets[p] === 0`.

**Timer bar** — the gradient (`red → orange → green`) is painted on the `.timer-bar` background and stays fixed-width (`calc(13 * 37px + 16px)` = chart width). `.timer-fill` is a dark overlay anchored to the right edge that grows from 0% → 100% as time expires, revealing progressively less of the gradient. `.timer-text` is absolutely positioned to the left of the bar so it doesn't affect the bar's centering.

**Preflop range slots** — `updateSlotsDisabled()` applies three CSS states to each `.range-slots` panel: `pf-disabled` (not preflop), fully active (active player's panel), or `opp-view` (opponent's panel — load enabled, save/name-edit disabled). `loadRange(pid, i)` always loads into `G.actPid`'s chart regardless of which slot panel `pid` the data came from, remapping actions via `remapAction(action, isFacing(dest))`.

**Strength bar** — `buildComboStrengthIndex()` (called at the start of each turn) enumerates all valid combos (excluding board cards), evaluates each with `evalHand([c1,c2,...board])`, sorts by score, and groups equal scores into `comboStrengthGroups`. `renderStrengthBars()` displays two stacked bars (one per player) showing how each player's range is distributed across hand strengths on the current board. The active player's bar colors combos by assigned action; the opponent's bar shows which combos are in their narrowed range. Hidden preflop (no board = no scores). The action panel has a toggle (`barView`) between this view and the action-frequency bar (`#range-dist-bar` + `#range-dist`); both share the same text label above them.

## Layout

`#main-area` is a CSS grid (`auto auto 200px auto auto`, `justify-content: space-evenly`): slot-panel | player-panel | center-col | player-panel | slot-panel.

Each `.player-panel` (flex column, `align-items: center`) contains from top:
1. `.hole-area` — hole cards + `.bet-disp` (absolutely positioned to outer side)
2. `.player-info-wrap` (relative) — `.player-info-box` (name + stack) with `.dealer-btn` absolutely positioned outside (`#dbtn0` right, `#dbtn1` left)
3. `.timer-row` — timer text (absolute, left of bar) + timer bar
4. `.hand-chart` — the 13×13 grid (37px × 47px cells)
5. `.range-sum` — combo count

The center column (`#center-col`) contains from top: `#board-section` (community cards + pot), `#action-panel` (hidden when not a player's turn). The action panel contains in order: prompt text, sel-mode buttons (Freehand | Rectangle | Select All), action frequency text (`#range-dist`), bar toggle row, action frequency bar (`#range-dist-bar`) / strength bars (`#strength-bar-wrap`), action buttons, bet size row, presets, suit filter, hand types, submit row.

## Key invariants

- Opponent hole cards are **never** excluded from `initRange` (would leak info and reduce 1326→1225)
- `G.narrowed` is reset to `[null, null]` only in `dealHand()`, not between streets
- Suit filter (`suitFilter`) affects drag assignment and category selection but not category combo counts (counts scan all suits)
- `drag.cells` is cleared by `deselectAll()` which is called at the top of `submit()` and on the "Deselect All" button
- Chart-width-dependent CSS values use `calc(13 * 37px + 16px)` — update when cell size changes
- `comboStrengthGroups` is `null` preflop; the strength bar toggle button is disabled and `barView` resets to `'freq'` automatically when no groups are available
