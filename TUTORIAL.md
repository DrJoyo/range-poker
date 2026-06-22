# Range Poker Trainer — Tutorial

## What Is This?

Range Poker Trainer is a heads-up poker training tool with a twist: **you never know your own cards**. Instead of playing one specific hand, you play an entire *range* of starting hands simultaneously. Each turn, you select groups of hands on your chart and assign them a poker action (fold, call, check, bet, raise). The game tracks which hands remain consistent with both players' actions, narrows each player's range over the course of the hand, and finally reveals the actual dealt cards at showdown.

This forces you to think in ranges — the same mental model used by strong poker players — rather than reacting to a single hand in isolation.

---

## The Setup Screen

When you open the app, you'll see the setup screen.

**Game Mode** — Select which variant to play from the dropdown. See the mode-specific sections below for details on each.

**Local tab** configures a two-player game on one device:
- **Starting Stack** — how many big blinds each player starts with (default 100bb)
- **Time per Turn** — seconds each player has to act (default 120s)
- **Player 1 / Player 2 Name** — display names shown in-game
- **Range EV Split at Showdown** — instead of the player who was dealt the better hand winning the pot outright, the pot is split proportionally to each player's range equity (useful for studying without luck variance)

**Online tab** lets you play against someone remotely:
- **Host Game** — creates a room and shows a 6-character code and shareable URL to send your opponent
- **Join Game** — enter the room code your opponent shared, then click Join

Click **Start Game** (or **Create Room** for online) to begin.

---

## The Game Interface

### Layout

The screen is divided into five columns:

```
[Range Slots P1] [Player 1 Panel] [Center Column] [Player 2 Panel] [Range Slots P2]
```

**Player Panels** (left and right) each contain, top to bottom:
- **Hole cards** — face-down until showdown; face-up cards are visible in Hollywood modes
- **Position badge** — SB (small blind / button) or BB (big blind)
- **Player name + stack** in big blinds
- **Dealer button** (D) marker
- **Timer bar** — counts down from your time limit; turns red as it expires
- **Hand chart** — the grid you click and drag to select hands
- **Combo count** — how many hand combos are still in this player's range

**Center Column** contains:
- **Community cards** (board) — revealed as the hand progresses
- **Pot size**
- **Action panel** — only visible when it is your turn to act

**Range Slots** (far left and right panels) hold up to 5 saved preflop ranges per player.

---

### The Hand Chart

The hand chart is the central mechanic. In most modes it is a **13×13 grid** (or 9×9 in Short Deck):

- **Top-left to bottom-right diagonal** — pocket pairs (AA, KK, QQ, …)
- **Above the diagonal (top-right triangle)** — suited hands (AKs, AQs, …)
- **Below the diagonal (bottom-left triangle)** — offsuit hands (AKo, AQo, …)

Each cell is subdivided into small colored rectangles representing individual suit combinations within that hand type. The color of each sub-rectangle shows the action currently assigned to those combos:

| Color | Action |
|---|---|
| Dark gray | Unassigned / default (fold if facing a bet; check if not) |
| Red | Fold |
| Blue | Call / Check |
| Orange | Raise / Bet |
| Faded | Not in range (eliminated by prior action) |

---

### Playing a Turn

When it is your turn, the action panel appears in the center column. The flow is:

1. **Select hands** on your chart
2. **Click an action button** to assign that action to your selected hands
3. Repeat for different groups of hands
4. Click **Submit** when you are done assigning

#### Selecting Hands

Three selection modes are available:

- **✏ Freehand** — click and drag to paint cells; click a cell again to deselect it
- **▭ Rectangle** — drag a rectangular region to select everything inside it
- **+ Select All** — selects all cells in your current range at once

You can also use the **Hand Types** panel (below the action buttons) to select by category — e.g. click "Top Pair (48)" to instantly select all top-pair combos.

#### Action Buttons

The buttons shown depend on the betting situation:

- **Fold** — fold these hands
- **Call** — call the current bet with these hands
- **Check** — check (shown when no bet is facing you)
- **Raise / Bet** — raise or make a bet; set the size in the bet-size field above the presets
- **Bet presets** — quick-select common bet sizes (e.g. 1/3, 1/2, 2/3, Pot, All-in)

#### Submitting

Click **Submit** to confirm your action. The game checks what action your actual dealt hand falls into and executes that action. Your range is then **narrowed** to the combos consistent with the action you took — combos you assigned to other actions are eliminated from your range for the rest of the hand.

---

### Tools and Panels

**Undo** — reverts the last assignment you made. Disabled at the start of each new turn.

**Save Draft** — saves your current chart state as a fallback. If your timer expires before you submit, your draft is submitted automatically.

**Fold Early** — permanently removes the selected combos from your range (marks them folded) without submitting. Useful for cleaning up combos you know you will never play before you start assigning.

**Randomize** — opens a panel where you set percentages (e.g. 33% Call, 33% Raise, 34% Fold) and randomly assign those proportions across your entire unassigned range.

---

### Suit Filter

Below the action buttons, the **Suit Filter** panel lets you restrict assignments to specific suit combinations. Useful for assigning, say, only the spade-flush-draw combos within a hand type.

- Toggle individual suits on/off for Suited, Offsuit, and Pair categories
- The ♠♥♦♣ buttons in the Offsuit/Pairs sections select *exactly* the combos containing that suit (replacing the current filter, not adding)

Hidden in Half Hollywood mode (the chart already shows one cell per suit).

---

### Strength Bar

Postflop, the **Strength Bar** (toggle with "By Strength" button) shows a visual breakdown of all combos sorted by hand strength, weakest on the left to strongest on the right. Each column represents combos of equal strength, colored by assigned action. Hovering a column shows:
- The hand category label (e.g. "Top Pair", "Flush Draw + OESD")
- Action frequency stats split for the hovered point

Click the bar to expand it into a larger popup.

---

### Range Slots

The panels on the far left and right hold **5 saved preflop ranges** per player. These are only active preflop.

- **Save** — saves your current chart to that slot
- **Load** — loads the saved range into the active player's chart (actions are remapped if the facing situation has changed)
- Slot names are editable by clicking the name text
- **Export / Import** — serialize all 5 slots to a compact string for backup or sharing

Range slots are disabled in Half Hollywood mode.

---

### Showdown

When the hand ends (fold or all streets complete), the **showdown overlay** appears:
- Both players' hole cards flip face-up
- The winner is determined by the actual dealt hands
- If Range EV Split is on, the pot is divided by range equity instead
- Click **Next Hand** to deal again, or **Play Again** if a player has busted out

---

## Game Modes

---

### No Limit Hold'em (NLHE)

The standard mode. Two face-down hole cards per player, standard 52-card deck.

- **Chart:** 13×13 grid, 1326 total combos
- **Evaluator:** standard hold'em rankings (Royal Flush → High Card)
- **Hand categories:** depend on board texture — set, two pair, top pair, flush draw, OESD, gutshot, etc.
- **Blinds:** dealer posts SB (0.5bb), opponent posts BB (1bb); dealer acts first preflop, second postflop

This is the baseline mode. All other modes build on NLHE rules unless noted.

---

### Short Deck Hold'em (SD)

Played with a 36-card deck — all cards below 6 are removed (2, 3, 4, 5 gone). Otherwise NLHE rules apply with two key rule changes:

- **Flush beats Full House**
- **A-6-7-8-9 is a valid straight** (Ace plays low; 6 is the "5" of Short Deck)

**Chart:** 9×9 grid (ranks A K Q J T 9 8 7 6), 630 total combos. The grid looks the same as NLHE but smaller.

**Hand categories** reflect the altered rankings — in the Hand Types panel and strength bar, Flush appears *above* Full House.

**Draw detection:** 6-7-8-9 is double-ended (open-ended) with outs to T and A. The gutshot/OESD thresholds are adjusted for the Short Deck wheel.

---

### Aces Cracked (CTA)

A novelty variant where one player is always dealt pocket aces.

- **BB** (the non-dealer) always holds **A♠A♥**, dealt face-up and visible to both players
- **SB** cannot hold any aces — those combos are automatically excluded from their range
- **BB's chart:** a single 1×1 grid showing only the AA cell (the AsAh combo is the only possible hand)
- **BB's preflop options:** Call or Fold only (no raise). If SB limps in, BB may also Check

**Gameplay:** SB plays normally on a full 13×13 chart. BB's AA cell is automatically selected at the start of each preflop turn — they just choose Call or Fold. Postflop, both players act normally.

The interesting training question is: how does SB play against a known AA? And how aggressively should BB play postflop with a hand their opponent can never fold out preflop?

---

### Hollywood

Each player holds **1 face-down card + 1 face-up card**. The face-up cards are visible to both players from the start.

- **Chart:** standard 13×13 grid, but filtered to only the combos that include your face-up card
- Because each player's visible card is different, both players' charts show a different subset of combos
- The opponent's face-up card is shown on their player panel throughout the hand
- Hands are evaluated from up to 7 cards (2 hole cards + up to 5 community), but since one hole card is already revealed, the opponent has partial information about your hand
- Hand evaluation supports up to 8 cards (face-up + face-down + board)

**Strategy note:** the face-up card creates asymmetric information. If your face-up card is a K and the board is K-7-2, your opponent knows you likely have a pair of kings. Your range is automatically limited to hands containing your face-up card, so you cannot bluff with completely unrelated holdings.

---

### Half Hollywood

The deepest variant. Each player holds **1 face-down card + 1 face-up card**, and the range chart represents individual cards rather than hand pairs.

**The Chart:** a **4×13 grid** (suits on rows, ranks on columns):

```
    2  3  4  5  6  7  8  9  T  J  Q  K  A
♠ [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
♥ [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
♦ [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
♣ [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
```

Each cell represents **one specific card** — your possible face-down card. Select cells to assign actions to specific cards you might be holding.

**Blocked cells:** cells are blacked out if that card:
- Is already dealt face-up (either player's face-up card)
- Is already on the community board

You cannot hold a card that is already visible, so blocked cells are automatically excluded from your range.

**Hand categories and strength:** treat the face-down card + your face-up card together as a 2-card hold'em hand. For example, if your face-up card is K♥ and you are considering the 7♥ cell, the hand is K♥-7♥ (suited king-seven).

**Key differences from Hollywood:**
- No suit filter (already separated by suit in the grid)
- No range slots (not applicable to single-card ranges)
- The combo count shows how many individual cards remain in your range

**Selecting hands:** drag across cells the same way as other modes — each cell you touch selects one specific card. Hand Types still work: clicking "Top Pair" selects all face-down cards that make top pair with your face-up card given the current board.

---

## Online Multiplayer

To play online:

1. **Host:** go to Online → Host Game. Configure your settings, click **Create Room**. Share the 6-character code or the link shown on screen.
2. **Guest:** go to Online → Join Game. Enter the room code, type your name, click **Join Game**.

Once both players are connected, the game starts automatically. The host's game mode and settings are authoritative — the guest's dropdown is ignored.

During online play:
- You can only interact with your own chart when it is your turn
- The opponent's chart is visible but locked
- The opponent's face-up cards (Hollywood/Half Hollywood) are always visible to you
- Range slots are read-only for the opponent's panel

---

## Tips

- **Start with Hand Types.** Rather than painting individual cells, use the Hand Types panel to select full categories (Top Pair, Flush Draw, etc.) and assign them in bulk.
- **Use Fold Early.** If you know you will always fold certain combos (e.g. the weakest offsuit trash), fold them early at the start of your turn to keep your chart clean.
- **Watch the combo count.** The number below each chart shows how many combos are in that player's range. Watch it shrink as streets progress — it tells you how much information has been revealed.
- **Strength bar postflop.** Switch to "By Strength" view to see how your range is distributed across hand strengths. A well-constructed range has actions spread across the strength spectrum, not concentrated at one end.
- **Save Draft often.** If your timer is running low, hit Save Draft. That snapshot will be submitted if time expires.
- **Range EV Split** is useful for studying — it removes luck from individual hands and rewards accurate range construction over a session.
