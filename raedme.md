# 6 Nimmt! Arcade Edition — Documentation

**File:** `game_1_.ipynb`
**Engine:** [Python Arcade](https://api.arcade.academy/)
**Window size:** 1400 × 830 px

---

## 1. Overview

This notebook implements a single-player-vs-3-AI digital version of the card game **6 Nimmt!** ("Take 6!"). One human player ("You") competes against three AI opponents. Cards are played simultaneously, resolved in ascending order, and placed onto four table rows according to the game's placement rules. Players accumulate penalty points ("bull points") by being forced to take a full row of cards; the round ends when hands are empty, and whoever has the **fewest** penalty points wins.

---

## 2. Requirements

| Requirement | Notes |
|---|---|
| Python 3.8+ | Uses type hints (`List`, `Tuple`, `Optional`) |
| `arcade` library | Installed via `!pip install arcade` in Cell 0 |
| Display environment | `arcade.run()` opens a native window — this will **not** render inside a headless notebook/Colab session without a display |

---

## 3. Game Rules Implemented

### 3.1 The Deck
- 104 cards, numbered 1–104.
- Each card carries a **bull point** (penalty) value:

| Condition | Bull Points |
|---|---|
| Card is exactly 55 | 7 |
| Repeating-digit numbers (11, 22, 33, …, 99) | 5 |
| Ends in 0 | 3 |
| Ends in 5 (excluding the special cases above) | 2 |
| All other cards | 1 |

### 3.2 Setup
- Deck is shuffled.
- 4 players total: 1 human (`"You"`) + 3 AI (`"AI 1"`, `"AI 2"`, `"AI 3"`).
- Each player is dealt **10 cards**.
- 4 cards are drawn from the deck to start **4 table rows** (one card each).

### 3.3 Turn Structure
1. The human selects a card from their hand by clicking it.
2. Each AI player simultaneously selects a card using its own heuristic (see §5).
3. All four played cards are collected and **sorted in ascending order**.
4. Cards are placed one at a time (1-second delay between placements) following standard 6 Nimmt! placement logic:
   - The card is added to the row whose **top (last) card** has the highest number that is still lower than the played card.
   - If that target row already has **5 cards**, the player instead **takes the entire row** into their penalty pile, and their card becomes the new sole card in that row.
   - If the played card is **lower than every row's top card** (no valid row), the player must take the row with the **fewest total bull points**, and their card becomes the new sole card in that row.
5. This repeats until all 4 cards from the turn are placed.
6. If the human's hand is empty after the turn, the round ends and scores are finalized (`GAME_OVER` state). Otherwise, play continues (`PLAYER_TURN` state).

### 3.4 Scoring & Win Condition
- A player's score is the sum of bull points across all cards in their `penalty_pile`.
- At game end, the player(s) with the **minimum score** win.
- If multiple players tie for the minimum, the result is declared a tie (no single winner).

---

## 4. Code Structure

### 4.1 Constants (Cell 2)
Defines screen dimensions, title, card scaling, and layout constants for positioning the player's hand and the four table rows.

### 4.2 `Card` (Cell 4)
- Constructor validates `1 <= number <= 104`.
- `_calculate_bull_points()` implements the bull-point rules from §3.1.
- `__repr__` gives a debug-friendly string, e.g. `Card(55, BP:7)`.

### 4.3 `CardSprite` (Cell 6)
- Subclass of `arcade.Sprite`; wraps a `Card` for rendering.
- Uses a soft square white texture as the card face.
- `draw_card_info()` draws the card's number and bull-point value as text overlays on top of the sprite.

### 4.4 `Player` (Cell 8)
- Tracks `name`, `hand` (list of `Card`), `score`, `is_ai` flag, and `penalty_pile`.
- `add_card_to_hand()` appends a card to the hand.
- `calculate_score()` sums bull points in the penalty pile and stores/returns it.

### 4.5 `SixNimmtGame` (Cell 10)
The main `arcade.Window` subclass and game controller.

**Key state:**
- `deck`, `players`, `rows` (4 lists of `Card`), `turn_placements` (this turn's sorted plays)
- Sprite lists: `player_hand_sprites`, `row_sprites` (one `SpriteList` per row), `played_card_sprites`
- `game_state`: one of `"SETUP"`, `"PLAYER_TURN"`, `"PROCESSING_TURN"`, `"GAME_OVER"`
- `selected_card_sprite`, `placement_index`, `winner`

**Key methods:**

| Method | Purpose |
|---|---|
| `setup()` | Shuffles deck, creates players, deals 10 cards each, starts 4 rows, builds hand sprites, sets state to `PLAYER_TURN` |
| `position_player_hand()` | Lays out the human's hand sprites in a horizontal strip |
| `position_row_sprites()` | Rebuilds and positions sprites for all 4 table rows |
| `on_draw()` | Renders rows, hand, labels, scores, and (if applicable) the game-over overlay |
| `draw_scores()` | Renders a live scoreboard (top-right) for all 4 players |
| `draw_game_over()` | Draws a semi-transparent overlay with the winner/tie message and a "click to restart" prompt |
| `on_mouse_press()` | Handles card selection during `PLAYER_TURN`; restarts the game via `setup()` if clicked during `GAME_OVER` |
| `execute_turn()` | Collects the human's chosen card + each AI's chosen card, removes them from hands, sorts by number, and schedules sequential placement |
| `_process_next_placement()` | Called on a 1-second `arcade.schedule` timer; places the next card in `turn_placements` per the row rules, updates penalty piles, and checks for round end |
| `_choose_card_for_ai(player)` | AI decision logic (see §5) |

### 4.6 `main()` (Cell 12)
Instantiates `SixNimmtGame`, calls `setup()`, then `arcade.run()` to start the event loop.

---

## 5. AI Logic (`_choose_card_for_ai`)

For each card in its hand, the AI evaluates every row and classifies the card as:
- **Safe** — there exists a row it can legally join without being the 5th card (i.e., no penalty, row not yet full).
- **Unsafe** — either no valid row exists, or joining would make it the 5th card and force a row pickup; a `penalty` value (total bull points it would incur) is computed for this case.

**Selection heuristic:**
1. If any *safe* option exists, play the **highest-numbered safe card** (this keeps low cards in hand for later, since they're more likely to be forced into penalties).
2. Otherwise, among unsafe options, play whichever card produces the **lowest penalty**.

This is a simple greedy, single-turn-lookahead heuristic — it doesn't account for opponents' likely plays or multi-turn strategy.

---

## 6. Controls / User Interaction

| Action | Effect |
|---|---|
| Click a card in "Your Hand" | Selects it (raises it slightly) and immediately triggers `execute_turn()` |
| Click anywhere during Game Over | Calls `setup()` to start a new round |

Note: the human currently gets **no confirmation step** — clicking a hand card immediately locks in that play for the turn.

---

## 7. Known Limitations / Potential Improvements

- **No confirmation/undo** after selecting a hand card — misclicks immediately commit the turn.
- **AI heuristic is greedy/single-turn** — no opponent modeling or hand-tracking strategy.
- **Score display** in `draw_scores()` recomputes from `penalty_pile` each frame, while `Player.calculate_score()` (used for win determination) also sets `self.score` — both paths exist but aren't unified into one source of truth.
- **No sound, animations beyond position updates, or multiplayer (human vs human) support.**
- **Requires a display** — won't run in purely headless notebook environments (e.g., some cloud notebook backends) without a virtual framebuffer.
- Only a **single round** is modeled — there's no cumulative multi-round tournament scoring or elimination at 66 points, which is standard in the physical board game.

---

## 8. How to Run

1. Run all cells in order (Cell 0 installs `arcade` if not already present).
2. The last cell's `if __name__ == "__main__": main()` launches the game window.
3. Click a card from "Your Hand" to play your turn each round; the three AI players respond automatically.
4. When your hand is empty, the round ends and a Game Over screen shows the winner — click anywhere to play again.
