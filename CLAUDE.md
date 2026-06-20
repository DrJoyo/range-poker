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
- `presetBoard` — snapshot of the module-level `presetBoard` array taken at `dealHand()`; used by `advanceStreet()` and `runOutBoard()` to override deck draws with specific cards (debug mode only)

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

**Range EV Split** — a setup-screen toggle (`cfg.evSplit`) that changes showdown pot award from "winner takes all based on dealt cards" to "split proportional to range equity". `computeRangeEV(board)` enumerates all valid combos from each player's `G.narrowed` set (or full range if `null`), pre-computes hand ranks for every combo via `evalHand`, then double-loops over all `(combo0, combo1)` pairs that share no card to tally wins/ties. Returns `{ ev0, ev1 }` fractions; `showdown()` awards `Math.round(ev0 * pot * 100) / 100` to player 0 and the remainder to player 1 (rounded to 0.01bb). The pre-computation of ranks outside the inner loop is critical — evaluating inside the inner loop causes O(N²) `evalHand` calls and freezes the page with large ranges. `cfg.evSplit` is written to `rooms/<code>/config` in Firebase so the guest receives it automatically.

**Card reveal animation** — `renderBoard(newFrom)` applies a `ccReveal` CSS animation only to cards at index ≥ `newFrom`, with a 90 ms stagger per card (0 / 90 / 180 ms for the flop). Passing `null` (initial deal) skips animation entirely.

**Range narrowing** — after a player submits, `G.narrowed[pid]` is set to the union of all combo keys that got assigned to the action they actually took. `initRange` checks this set on subsequent turns so grayed-out (eliminated) combos are excluded from the map entirely.

**Drag selection** — `drag.cells` is a `Set<"r,c">` of selected chart cells. Selecting adds/removes from this set; action button click calls `assignSelection(pid, action)` which applies the suit filter and clears `drag.cells`. Category buttons (`selectCategory`) add to `drag.cells` rather than directly assigning. Default selection mode is freehand (`selMode = 'free'`).

**Undo stack** — `undoStack` is a module-level array of `{ pid, delta }` entries where `delta` is a flat map of `comboKey → previousAction` for every key modified by that `assignSelection` call. Pushed inside `assignSelection` before writing (only if at least one key changed); popped by `undoLast()` which iterates existing map keys and restores any found in the delta. The stack is cleared and the Undo button disabled at the start of each turn in `startTurn`. `foldEarly` **deletes** combo keys from `G.ranges` entirely, so those keys are invisible to `undoLast`'s restore loop — folded-early combos intentionally stay removed even if the assignment that set them to "fold" is undone.

**Hand evaluator** — `evalHand(cards)` picks best 5 from 7 (or 6) cards via `eval5`. Rank thresholds: SF≥8e8, Quads≥7e8, FH≥6e8, Flush≥5e8, Straight≥4e8, Trips≥3e8, TwoPair≥2e8, Pair≥1e8. Tiebreakers are packed into the lower digits (base-15 positional for flush/high-card; rank×multiplier for made hands).

**Hand type categories** — `boardAnalysis(board)` computes rank/suit counts and returns:
- `rankCounts`, `suitCounts`, `uniqueRanks` (sorted desc by RVAL), `isPaired`, `topRankVal`
- `hasBoardTrips` — some rank appears exactly 3×
- `hasBoardFH` — some rank appears 3× AND another appears 2× (trips + pair on board)
- `hasBoardQuads` — some rank appears 4×
- `isDoublePaired` — exactly two ranks each appear 2×, no trips/quads
- `lowestPairedRankVal` — RVAL of the lowest rank that appears 2+ times
- `boardScore` — `evalHand(board).rank` (precomputed for FH boards; 0 otherwise)

`getHandCategories(pid, board)` dispatches on board type before falling through to the standard single-paired/unpaired path:

| Board type | Categories shown |
|---|---|
| Quads on board | Nothing only |
| FH on board | Quads → Full House (strictly beats `boardScore`) → Nothing |
| Trips on board (no FH) | SF → Quads → FH → Flush → Straight → Draws → Nothing |
| Double-paired | SF → Quads → FH → Flush → Straight → Overpair → [Top/2nd Pair if unpaired card isn't the lowest board card] → Underpair → Draws → Nothing |
| Single-paired / unpaired | Existing logic (Set, Two Pair, Trips, etc.) |

For the double-paired path: the one unpaired card's pair category is suppressed when that card is the lowest-ranked card on the board (counterfeit). `isUnderpair` additionally requires the pocket pair to exceed `lowestPairedRankVal` on double-paired boards — pocket pairs below the lowest paired rank go to Nothing. `matchesCategory` 'nothing' mirrors this: on double-paired boards it excludes `'trips'` and the counterfeit pair index from its allIds check so `selectCategory('nothing')` correctly selects those combos.

OESD vs gutshot: missing rank at the low end of a window is only OESD if `lo+4 < 14` (Ace not at top); missing at high end only OESD if `lo > 1` (Ace-low not at bottom). Straight/flush draws are suppressed on the river and when not structurally possible.

**`isPairAtRank` on paired boards** — On an unpaired board, `isPairAtRank` requires the other hole card to not hit the board (prevents double-counting with `isTwoPair`). On paired boards, `isTwoPair` is disabled, so `isPairAtRank` relaxes this: when both hole cards each pair a different *unpaired* board rank (counterfeit two pair), only the higher-ranked hole card's pair is counted. If the other hole card hits a *paired* board rank (would make a full house), it still returns false.

**Debug board preset** — Module-level `presetBoard[5]` (array of card strings or `null`) is shown as a rank+suit picker row (`#board-preset`) below the debug button, visible only when `debugMode` is on. `buildPresetUI()` renders 5 slots (F1/F2/F3/T/R); selecting a rank enables the suit select and writes to `presetBoard[i]`. At `dealHand()` the array is snapshotted into `G.presetBoard` and preset cards are removed from the shuffled deck before hole cards are dealt (preventing collision). `advanceStreet()` and `runOutBoard()` call `nextCard(i)` which returns `G.presetBoard[i]` if set, otherwise `G.deck.pop()`. Preset persists across hands until cleared or debug mode is toggled off (which resets all slots to `null`). Hidden in online mode alongside the debug button.

**Check display** — `execAction` sets `bdisp[pid].textContent = 'Check'` for postflop checks; `updateUI()` (called by `collectBets()` on street advance) clears it automatically when `G.bets[p] === 0`.

**Timer bar** — the gradient (`red → orange → green`) is painted on the `.timer-bar` background and stays fixed-width (`calc(13 * 37px + 16px)` = chart width). `.timer-fill` is a dark overlay anchored to the right edge that grows from 0% → 100% as time expires, revealing progressively less of the gradient. `.timer-text` is absolutely positioned to the left of the bar so it doesn't affect the bar's centering.

**Preflop range slots** — `updateSlotsDisabled()` applies CSS states to each `.range-slots` panel: `pf-disabled` (not preflop, or in online mode: opponent's panel or not your turn), fully active (active player's panel), or `opp-view` (offline only — opponent's panel: load enabled, save/name-edit disabled). `loadRange(pid, i)` always loads into `G.actPid`'s chart regardless of which slot panel `pid` the data came from, remapping actions via `remapAction(action, isFacing(dest))`. In online mode, `saveRange` and `loadRange` both guard `pid !== myPid || G.actPid !== myPid` and return early. Export serialises all 5 slots to a compact JSON string (`{"v":1,"slots":[...]}`) where each slot's ranges are encoded as a 1326-character string (one char per combo in canonical grid order: r=0–12, c=0–12, `typeSuits` order; chars `f/c/k/r/b/.`); `importSlots` reverses this via `encodeSlotRanges` / `decodeSlotRanges`.

**Strength bar** — `buildComboStrengthIndex()` (called at the start of each turn) enumerates all valid combos (excluding board cards), evaluates each with `evalHand([c1,c2,...board])`, sorts by score ascending, and groups equal scores into `comboStrengthGroups`. Each group gets a `label` (e.g. `"Top Pair"`, `"Flush Draw + OESD"`) precomputed via `comboLabel(c1, c2, ba, board)`, which mirrors the board-type dispatch of `getHandCategories` (quads board / FH board / trips board / double-paired / standard) and joins all matching category names with `" + "`. `renderStrengthBarsInto(bar0, bar1)` is the shared render helper: each group becomes a column div with `data-label` and `data-gidx` attributes, width proportional to combo count; height segments stacked top-to-bottom as not-in-range → fold → call/check → raise/bet (raise sits at the visual bottom). `renderStrengthBars()` calls this helper for the inline bars (`#sbar0`/`#sbar1`) and, if the popup is open, for `#sbar0-lg`/`#sbar1-lg` inside `#strength-popup`. Clicking the inline bars opens a fixed-position popup (70vw wide) with 120px-tall versions; clicking anywhere dismisses it via a one-shot `document.addEventListener('click', dismiss)`. The popup also has: a `#strength-tooltip` (positioned absolutely, shows the category label on hover), and `#strength-stats` (split action-frequency breakdown — combos to the left/right of the hovered group, showing Fold/Call/Check/Raise/Bet counts for the active player's bar or In-Range/Not-In-Range for the opponent's bar). Both are updated in `onBarMove` (guarded by `lastKey = pid:gidx` to avoid redundant DOM work) and cleared on mouse leave. Hidden preflop. The action panel has a toggle (`barView`) between this view and the action-frequency bar (`#range-dist-bar` + `#range-dist`).

## Layout

`#main-area` is a CSS grid (`auto auto 200px auto auto`, `justify-content: space-evenly`): slot-panel | player-panel | center-col | player-panel | slot-panel.

Each `.player-panel` (flex column, `align-items: center`) contains from top:
1. `.hole-area` — hole cards + `.bet-disp` (absolutely positioned to outer side)
2. `.player-info-wrap` (relative) — `.player-info-box` (name + stack) with `.dealer-btn` absolutely positioned outside (`#dbtn0` right, `#dbtn1` left)
3. `.timer-row` — timer text (absolute, left of bar) + timer bar
4. `.hand-chart` — the 13×13 grid (37px × 47px cells)
5. `.range-sum` — combo count

The center column (`#center-col`) contains from top: `#board-section` (community cards + pot), `#action-panel` (hidden when not a player's turn). The action panel contains in order: prompt text, sel-mode buttons (Freehand | Rectangle | Select All), action frequency text (`#range-dist`), bar toggle row, action frequency bar (`#range-dist-bar`) / strength bars (`#strength-bar-wrap`), action buttons, bet size row, presets, suit filter, hand types, submit row.

## Online multiplayer (Firebase)

Firebase Realtime Database is used for peer-to-peer synchronization. The host (pid 0) owns all authoritative game state; the guest (pid 1) is a receiver who only writes their own actions.

**Setup flow**
- Host: `hostGame()` → writes room config to `rooms/<code>`, listens on `rooms/<code>/players/1` for guest join → calls `newMatch()` and `writeHandToFirebase()`
- Guest: `joinGame()` → reads room config, writes `rooms/<code>/players/1`, calls `setupHandListener()` + `setupActionListener()`, then `newMatch()` (which returns early in `dealHand()` since `myPid === 1`)
- `?room=CODE` in the URL auto-fills the join screen and switches to the online tab

**Firebase data paths**
- `rooms/<code>/hand` — host writes full hand state (deck, hole cards, stacks, dealer, sequence number `n`); guest listens via `setupHandListener()` → `onRemoteHand()`
- `rooms/<code>/pendingAction` — whoever just acted writes their action; the other player listens via `setupActionListener()` → `onRemoteAction()`. A single node (not an array): overwritten each action, deduplicated by `handN + n + pid`

**Deduplication** — `actionSeenKey` (`"pid:n"`) and `currentHandN` prevent replaying own actions or stale actions from a previous hand.

**Draft & timer fallback** — `G.draft[pid]` / `G.draftBetSz[pid]` store a snapshot saved by `saveDraft()`. When the timer expires (`fromTimer=true`), `submit()` uses the draft ranges instead of the live ones so a player's last manual save is submitted rather than an empty range.

**Key online-mode guards** — `if (onlineMode && myPid === 1)` in `dealHand()` (guest skips local deal); `if (onlineMode && pid !== myPid)` in `startTurn()` (opponent's turn shows waiting message); `if (onlineMode && myPid !== 0)` in `nextHand()` / `closeShowdown()` (only host advances the hand).

## Key invariants

- `fmt(bb)` formats chip amounts as `+bb.toFixed(2)+'bb'` — the unary `+` strips trailing zeros so integers display as `100bb` not `100.00bb`. Bet presets and input clamping use `.toFixed(2)`; the bet-input `step` is `0.5` (arrow buttons) but accepts any decimal when typed. EV split awards round to `0.01bb`.
- Opponent hole cards are **never** excluded from `initRange` (would leak info and reduce 1326→1225)
- `G.narrowed` is reset to `[null, null]` only in `dealHand()`, not between streets
- Suit filter (`suitFilter`) affects drag assignment and category selection but not category combo counts (counts scan all suits)
- `drag.cells` is cleared by `deselectAll()` which is called at the top of `submit()` and on the "Deselect All" button
- Chart-width-dependent CSS values use `calc(13 * 37px + 16px)` — update when cell size changes
- `comboStrengthGroups` is `null` preflop; the strength bar toggle button is disabled and `barView` resets to `'freq'` automatically when no groups are available
- Suit filter "contains suit" buttons (♠♥♦♣, in Offsuit and Pairs sections) call `sfAddSuit(type, suitChar)` which **replaces** the current filter with exactly the combos containing that suit — they do not toggle or accumulate
- `foldEarly` (non-fold case) **deletes** combo keys from `G.ranges[pid][r][c]` (does not just reassign them) and tracks them in `earlyFolded[pid]` for ghost rendering; all other range logic assumes every combo key present in a map has a valid action string
- There is **no "unassigned" action state** — `initRange` sets every in-range combo to `'fold'` (if facing) or `'check'` (if not) as the default; a key absent from the map means the combo was narrowed out ("not in range"). Always use `ck in map` before reading `map[ck]` — a missing key is not-in-range, not a missing default
