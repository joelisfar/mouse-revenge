# Mouse Revenge — Design

A single-page browser game. A square teleports to random positions on the screen; the player must keep their mouse cursor hovering over it. Each successful "catch" requires dwelling inside the square for a shrinking duration. The only fail condition is a global 2-second budget for time spent outside the square.

## Goals

- Fun, immediately playable mouse-reflex game.
- Zero install — open `index.html` in a browser and play.
- Self-contained single file. ~150 lines of HTML/CSS/JS total.

## Non-goals

- No backend, no persistence, no leaderboards.
- No sound (could be added later).
- No mobile/touch support — this is a mouse game by design.
- No build system, no framework, no dependencies.

## Game states

### Idle (start screen)
- A square sits centered in the playfield.
- Subtitle below: "Hover the square to start. Don't let it escape for more than 2 seconds total."
- Nothing is ticking. `score = 0`, `airTimeUsedMs = 0`, `dwellMs = 2000`.

### Playing
- Triggered by the cursor entering the square for the first time.
- The `requestAnimationFrame` loop drives all updates from this point on.
- See **Core mechanics** for behavior.

### Game over
- Triggered when `airTimeUsedMs >= 2000`.
- Square freezes in place, dimmed.
- Centered overlay shows `Game over. Score: <n>` and a "Play again" button.
- Clicking "Play again" resets all state and returns to **Idle**.

## Core mechanics

### Catch (dwell)
- Each square has a `dwellMs` requirement.
- Initial value: `2000ms` (the player's first catch is intentionally generous).
- Each catch:
  1. `score += 1`
  2. `dwellMs = max(200, dwellMs - 100)`
  3. Square teleports to a new random position.
- Dwell progress is accumulated only while the cursor is inside the square.
- **Leaving the square resets the in-progress dwell to 0.** The `dwellMs` *requirement* itself does NOT reset — it only ratchets down per successful catch.
- Dwell floor: `200ms`. Reached at score = 18 and stays there for the rest of the run.

### Air-time budget (only fail condition)
- Total budget: `2000ms` for the entire run (cumulative, not per-teleport).
- Decrements only while the cursor is outside the square.
- Updated each frame using real elapsed ms (frame `dt`), so it remains accurate under frame drops.
- When `airTimeUsedMs >= 2000`, the game ends.
- There is no per-teleport deadline. The player can take as long as they want to find the next square, as long as their *total* air time stays under 2 seconds.

### Teleport position
- Random `(x, y)` chosen so the square stays fully within the playfield (accounting for square size and HUD).
- **Anti-cheese constraint:** the new position must be at least `150px` from the current cursor position (measured center-to-cursor). Otherwise, the square could spawn under the cursor and auto-catch.
- Implementation: sample up to 10 random candidates; pick the first that satisfies the distance check. If all 10 fail (tiny viewport), pick the farthest of the candidates.

### Frame loop
- Single `requestAnimationFrame` loop, running only while state = Playing.
- Each frame, compute `dt = now - lastFrame`:
  - If cursor is inside square: `dwellProgressMs += dt`; if `dwellProgressMs >= dwellMs`, trigger catch.
  - Else: `airTimeUsedMs += dt`; if `airTimeUsedMs >= 2000`, trigger game over.
- Mouse position is tracked via a `mousemove` listener on `document` and stored in a `mouse = {x, y}` variable read each frame.
- "Cursor inside square" is recomputed each frame from `mouse` and the square's current rect — not from `mouseenter`/`mouseleave` events (which can miss fast movements across teleports).

## UI layout

### Playfield
- Full viewport. Dark background (`#111`).
- Square positioned absolutely via `left` / `top`.

### HUD (fixed top bar)
- **Left:** `Score: <n>` — large.
- **Center:** Air-time bar — horizontal, represents the *remaining* budget. Bar starts full (2000ms) and visually shrinks from the right as `airTimeUsedMs` grows. Bar color shifts green → yellow → red as the remaining budget shrinks. Numeric label beside it: e.g., `1.43s left`.
- **Right:** `Next: <dwellMs>ms` — shows the current dwell requirement so the player can see it ratcheting down.

### Square
- 80×80 px.
- Color: `#ff5b5b` (coral).
- Slight brightness increase on hover for "you're in" feedback.
- While the cursor is inside, an inner fill rises from the bottom (like filling a glass), proportional to `dwellProgressMs / dwellMs`. Reaches the top → catch fires.
- Fill drops to 0 immediately when the cursor leaves.

### Game over overlay
- Semi-transparent dark scrim over the playfield.
- Centered card: `Game over.` / `Score: <n>` / `[Play again]` button.

### Start screen
- Square centered. Subtitle text below it.
- Standard cursor (no custom cursor — the OS pointer is the player's tool).

### Style
- Dark background (`#111`), one accent color (`#ff5b5b`), white text.
- System font stack.
- Minimal: no animations beyond the dwell fill, hover brightness, and color transitions on the HUD bar.

## File structure

```
mouse-revenge/
├── index.html          # the whole game (HTML + inline CSS + inline JS)
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-05-11-mouse-revenge-design.md   # this file
```

Single file. Open `index.html` in any modern browser to play.

## Open questions / future work

- Tunability — the magic numbers (square size 80px, dwell floor 200ms, decrement 100ms, air-time budget 2000ms, anti-cheese distance 150px) may want adjustment after playtesting.
- Sound effects on catch / game over.
- Persistent high score in `localStorage`.
- Optional difficulty mode where the square also shrinks over time.
