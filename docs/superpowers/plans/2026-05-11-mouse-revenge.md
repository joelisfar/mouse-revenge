# Mouse Revenge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file browser mouse-reflex game where a teleporting square must be tracked via cursor hover. Each catch requires "dwelling" inside for a shrinking duration; cumulative time spent outside the square is capped at 2 seconds (the only fail condition).

**Architecture:** One static HTML file (`index.html`) with inline `<style>` and `<script>`. No build, no dependencies, runs via `file://`. The script is organized into clearly-labeled sections (state, pure helpers, render, loop, input, init). The single non-trivial pure function (`chooseTeleportPosition`) is verified by a built-in self-test that runs at page load and writes results to `console`; if any assertion fails the game refuses to start.

**Tech Stack:** HTML, CSS, vanilla JavaScript (ES2020). No frameworks, no bundlers, no package.json.

---

## File Structure

- `index.html` — the entire game (HTML + inline CSS + inline JS). All work in this plan modifies this single file, except the initial creation in Task 1.

The script inside `index.html` is divided into named sections via comment headers, so each later task can say "add the following inside the `=== RENDER ===` section" without ambiguity.

The CSS is similarly divided into named blocks: `=== BASE ===`, `=== HUD ===`, `=== SQUARE ===`, `=== OVERLAY ===`.

---

## Task 1: Scaffold the page shell

**Files:**
- Create: `index.html`

**What this task delivers:** Open `index.html` in a browser and see a dark page with an empty HUD bar across the top. No interactivity yet.

- [ ] **Step 1: Create `index.html` with the full skeleton**

Create `index.html` with exactly this content:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Mouse Revenge</title>
  <style>
    /* === BASE === */
    *, *::before, *::after { box-sizing: border-box; }
    html, body {
      margin: 0;
      padding: 0;
      height: 100%;
      background: #111;
      color: #fff;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      overflow: hidden;
      user-select: none;
    }

    /* === HUD === */
    #hud {
      position: fixed;
      top: 0; left: 0; right: 0;
      height: 56px;
      display: flex;
      align-items: center;
      padding: 0 20px;
      gap: 20px;
      background: rgba(0, 0, 0, 0.4);
      border-bottom: 1px solid #222;
      z-index: 10;
    }
    #score    { flex: 0 0 auto; font-size: 22px; font-weight: 600; }
    #airwrap  { flex: 1 1 auto; display: flex; align-items: center; gap: 10px; }
    #airbar   { flex: 1 1 auto; height: 14px; background: #222; border-radius: 7px; overflow: hidden; }
    #airfill  { height: 100%; width: 100%; background: #4ade80; transition: background 120ms linear; }
    #airlabel { flex: 0 0 auto; font-variant-numeric: tabular-nums; font-size: 14px; opacity: 0.85; min-width: 70px; text-align: right; }
    #dwellreq { flex: 0 0 auto; font-size: 14px; opacity: 0.85; font-variant-numeric: tabular-nums; }

    /* === SQUARE === */
    /* (added in Task 3) */

    /* === OVERLAY === */
    /* (added in Task 6) */
  </style>
</head>
<body>
  <!-- === HUD === -->
  <div id="hud">
    <div id="score">Score: 0</div>
    <div id="airwrap">
      <div id="airbar"><div id="airfill"></div></div>
      <div id="airlabel">2.00s left</div>
    </div>
    <div id="dwellreq">Next: 2000ms</div>
  </div>

  <!-- === PLAYFIELD === -->
  <div id="playfield"></div>

  <!-- === GAME OVER OVERLAY === -->
  <!-- (added in Task 6) -->

  <script>
    "use strict";

    // === STATE ===
    // (added in Task 4)

    // === PURE HELPERS ===
    // (added in Task 2)

    // === RENDER ===
    // (added in Task 3)

    // === GAME LOOP ===
    // (added in Task 4)

    // === INPUT ===
    // (added in Task 3)

    // === INIT ===
    // (added in Task 3)
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify in a browser**

Open `index.html` (e.g., `open index.html` on macOS or just double-click).
Expected: a black page with a thin top bar that reads `Score: 0`, a full green bar, `2.00s left`, and `Next: 2000ms`. No square yet, no errors in DevTools console.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: scaffold game page shell with HUD bar"
```

---

## Task 2: Pure helpers + built-in self-tests

**Files:**
- Modify: `index.html` (inside the `=== PURE HELPERS ===` block)

**What this task delivers:** Four pure helper functions, plus assertions for the only non-trivial one (`chooseTeleportPosition`). On page load, the assertions run; if any fail, the page shows a visible error banner and `console.error` prints details.

The helpers:
- `decayDwell(currentMs)` → next-catch dwell requirement (drop 100ms, floor 200ms).
- `isInside(x, y, rect)` → `true` if `(x, y)` lies within `rect = {x, y, w, h}`.
- `formatSecondsLeft(usedMs, budgetMs)` → e.g., `"1.43s left"`.
- `chooseTeleportPosition(viewport, squareSize, cursor, minDistance, hudHeight, rng)` → `{x, y}` for the square's top-left such that the square stays fully on-screen below the HUD, and its center is at least `minDistance` from the cursor. Samples up to 10 candidates; if none satisfy, returns the candidate whose distance from the cursor is largest.

- [ ] **Step 1: Add the pure helpers**

Replace the line `// (added in Task 2)` under `// === PURE HELPERS ===` with:

```javascript
function decayDwell(currentMs) {
  return Math.max(200, currentMs - 100);
}

function isInside(x, y, rect) {
  return x >= rect.x && x <= rect.x + rect.w &&
         y >= rect.y && y <= rect.y + rect.h;
}

function formatSecondsLeft(usedMs, budgetMs) {
  const remaining = Math.max(0, budgetMs - usedMs);
  return (remaining / 1000).toFixed(2) + "s left";
}

function chooseTeleportPosition(viewport, squareSize, cursor, minDistance, hudHeight, rng) {
  const maxX = viewport.w - squareSize;
  const maxY = viewport.h - squareSize;
  const minY = hudHeight;
  let best = null;
  let bestDist = -1;
  for (let i = 0; i < 10; i++) {
    const x = Math.floor(rng() * (maxX + 1));
    const y = minY + Math.floor(rng() * (maxY - minY + 1));
    const cx = x + squareSize / 2;
    const cy = y + squareSize / 2;
    const dx = cx - cursor.x;
    const dy = cy - cursor.y;
    const dist = Math.hypot(dx, dy);
    if (dist >= minDistance) {
      return { x, y };
    }
    if (dist > bestDist) {
      best = { x, y };
      bestDist = dist;
    }
  }
  return best;
}
```

- [ ] **Step 2: Add a self-test function (write the failing tests first)**

Append the following directly below the helpers, still inside `// === PURE HELPERS ===`:

```javascript
function runSelfTests() {
  const failures = [];
  function assert(cond, msg) { if (!cond) failures.push(msg); }

  // decayDwell
  assert(decayDwell(2000) === 1900, "decayDwell(2000) should be 1900");
  assert(decayDwell(300) === 200,   "decayDwell(300) should clamp to 200");
  assert(decayDwell(200) === 200,   "decayDwell(200) should stay at 200");

  // isInside
  const rect = { x: 100, y: 100, w: 80, h: 80 };
  assert(isInside(140, 140, rect) === true,  "point inside should be true");
  assert(isInside(99, 140, rect) === false,  "point left of rect should be false");
  assert(isInside(181, 140, rect) === false, "point right of rect should be false");
  assert(isInside(100, 100, rect) === true,  "top-left corner should be inside (inclusive)");
  assert(isInside(180, 180, rect) === true,  "bottom-right corner should be inside (inclusive)");

  // formatSecondsLeft
  assert(formatSecondsLeft(0, 2000)    === "2.00s left", "full budget should be 2.00s left");
  assert(formatSecondsLeft(570, 2000)  === "1.43s left", "570ms used should be 1.43s left");
  assert(formatSecondsLeft(2000, 2000) === "0.00s left", "exhausted budget should be 0.00s left");
  assert(formatSecondsLeft(2500, 2000) === "0.00s left", "over-budget should clamp to 0.00s left");

  // chooseTeleportPosition: deterministic RNG
  // Far cursor — first sample should satisfy distance check.
  let pos = chooseTeleportPosition({ w: 1000, h: 800 }, 80, { x: 900, y: 700 }, 150, 56, () => 0);
  assert(pos.x === 0 && pos.y === 56, "with rng()=0 and far cursor, top-left should be (0, 56)");

  // Picks a position fully on-screen.
  pos = chooseTeleportPosition({ w: 1000, h: 800 }, 80, { x: 500, y: 400 }, 150, 56, Math.random);
  assert(pos.x >= 0 && pos.x <= 920, "x must be in [0, viewport.w - squareSize]");
  assert(pos.y >= 56 && pos.y <= 720, "y must be in [hudHeight, viewport.h - squareSize]");

  // Tiny viewport where no candidate can possibly satisfy minDistance — should still return a position.
  // Viewport just barely fits the square; cursor at center => every candidate is closer than 150.
  pos = chooseTeleportPosition({ w: 100, h: 100 }, 80, { x: 50, y: 50 }, 150, 0, Math.random);
  assert(pos !== null && pos.x >= 0 && pos.x <= 20 && pos.y >= 0 && pos.y <= 20,
    "tiny viewport should still yield a valid in-bounds position");

  if (failures.length > 0) {
    console.error("Self-tests FAILED:", failures);
    document.body.innerHTML = '<div style="padding:40px;color:#ff5b5b;font-family:monospace;">' +
      '<h2>Self-tests failed — game will not start.</h2><ul>' +
      failures.map(f => '<li>' + f + '</li>').join('') + '</ul></div>';
    return false;
  }
  console.log("Self-tests passed (" + 16 + " assertions).");
  return true;
}
```

- [ ] **Step 3: Wire the self-test into init**

Replace the line `// (added in Task 3)` under `// === INIT ===` with:

```javascript
if (!runSelfTests()) {
  // Self-tests failed; abort init.
} else {
  // Real init wired up in Task 3.
}
```

- [ ] **Step 4: Verify failing-then-passing flow**

Temporarily change one assertion to a wrong value to confirm the failure UX. For example, change `decayDwell(2000) === 1900` to `decayDwell(2000) === 1234`. Reload the page.
Expected: page is replaced with a red "Self-tests failed" banner listing the failing assertion. DevTools console shows the same.

Then revert the assertion. Reload.
Expected: page is still mostly empty (no square yet — that comes in Task 3), but DevTools console prints `Self-tests passed (16 assertions).` and the HUD is still visible.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add pure helpers and built-in self-tests"
```

---

## Task 3: Render the square + track mouse (idle state)

**Files:**
- Modify: `index.html` (CSS `=== SQUARE ===` block, the HUD subtitle, and JS sections `=== STATE ===`, `=== RENDER ===`, `=== INPUT ===`, `=== INIT ===`)

**What this task delivers:** A coral 80×80 square appears centered in the playfield, with a subtitle below it. The cursor position is tracked. No game loop yet — the square just sits there.

- [ ] **Step 1: Add square CSS**

Replace the line `/* (added in Task 3) */` under `/* === SQUARE === */` with:

```css
#square {
  position: absolute;
  width: 80px;
  height: 80px;
  background: #ff5b5b;
  border-radius: 6px;
  box-shadow: 0 4px 20px rgba(255, 91, 91, 0.3);
  transition: filter 80ms linear;
  pointer-events: none; /* hover detection is purely cursor-coords based */
  overflow: hidden;
}
#square.hovering { filter: brightness(1.2); }
#square > #fill {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  height: 0;
  background: rgba(255, 255, 255, 0.28);
}
#subtitle {
  position: fixed;
  bottom: 28px;
  left: 0; right: 0;
  text-align: center;
  font-size: 14px;
  color: #aaa;
}
```

- [ ] **Step 2: Add the square and subtitle to the DOM**

Replace this HTML block:

```html
  <!-- === PLAYFIELD === -->
  <div id="playfield"></div>
```

with:

```html
  <!-- === PLAYFIELD === -->
  <div id="playfield">
    <div id="square"><div id="fill"></div></div>
    <div id="subtitle">Hover the square to start. Don't let it escape for more than 2 seconds total.</div>
  </div>
```

- [ ] **Step 3: Add module-level state**

Replace the line `// (added in Task 4)` under `// === STATE ===` with:

```javascript
const SQUARE_SIZE      = 80;
const HUD_HEIGHT       = 56;
const AIRTIME_BUDGET   = 2000;
const INITIAL_DWELL    = 2000;
const ANTI_CHEESE_DIST = 150;

const state = {
  phase: "idle",          // "idle" | "playing" | "gameover"
  square: { x: 0, y: 0, w: SQUARE_SIZE, h: SQUARE_SIZE },
  mouse:  { x: -1, y: -1 },
  dwellMs: INITIAL_DWELL,
  dwellProgressMs: 0,
  airTimeUsedMs: 0,
  score: 0,
};

const el = {
  square:   null,
  fill:     null,
  score:    null,
  airfill:  null,
  airlabel: null,
  dwellreq: null,
};
```

- [ ] **Step 4: Add render functions**

Replace the line `// (added in Task 3)` under `// === RENDER ===` with:

```javascript
function renderSquare() {
  el.square.style.left = state.square.x + "px";
  el.square.style.top  = state.square.y + "px";
}

function renderHUD() {
  el.score.textContent    = "Score: " + state.score;
  el.airlabel.textContent = formatSecondsLeft(state.airTimeUsedMs, AIRTIME_BUDGET);
  el.dwellreq.textContent = "Next: " + state.dwellMs + "ms";
  const remaining = Math.max(0, AIRTIME_BUDGET - state.airTimeUsedMs);
  const pct = remaining / AIRTIME_BUDGET;
  el.airfill.style.width = (pct * 100) + "%";
  // Color: green > 50%, yellow 20–50%, red < 20%.
  el.airfill.style.background = pct > 0.5 ? "#4ade80" : pct > 0.2 ? "#facc15" : "#ef4444";
}

function renderDwellFill() {
  const pct = state.dwellMs === 0 ? 0 : state.dwellProgressMs / state.dwellMs;
  el.fill.style.height = Math.min(100, pct * 100) + "%";
}

function renderHoverClass(hovering) {
  el.square.classList.toggle("hovering", hovering);
}
```

- [ ] **Step 5: Add input tracking**

Replace the line `// (added in Task 3)` under `// === INPUT ===` with:

```javascript
function attachInput() {
  document.addEventListener("mousemove", (ev) => {
    state.mouse.x = ev.clientX;
    state.mouse.y = ev.clientY;
  });
}
```

- [ ] **Step 6: Wire init**

Replace the entire `// === INIT ===` block (the if/else from Task 2) with:

```javascript
if (runSelfTests()) {
  el.square   = document.getElementById("square");
  el.fill     = document.getElementById("fill");
  el.score    = document.getElementById("score");
  el.airfill  = document.getElementById("airfill");
  el.airlabel = document.getElementById("airlabel");
  el.dwellreq = document.getElementById("dwellreq");

  // Place square in center of playfield (below HUD).
  state.square.x = Math.floor((window.innerWidth - SQUARE_SIZE) / 2);
  state.square.y = HUD_HEIGHT + Math.floor((window.innerHeight - HUD_HEIGHT - SQUARE_SIZE) / 2);

  renderSquare();
  renderHUD();
  renderDwellFill();
  attachInput();
}
```

- [ ] **Step 7: Verify in a browser**

Reload `index.html`.
Expected:
- Coral square is centered in the playfield.
- Subtitle is visible near the bottom.
- HUD shows `Score: 0`, full green bar, `2.00s left`, `Next: 2000ms`.
- No console errors.
- In DevTools, you can run `state.mouse` (note: `state` is not exposed — temporarily attach `window.state = state` if you want to inspect; remove after verifying).

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: render square and track cursor in idle state"
```

---

## Task 4: Start trigger, game loop, dwell + air-time accumulators

**Files:**
- Modify: `index.html` (JS sections `=== GAME LOOP ===`, `=== INPUT ===`, `=== INIT ===`)

**What this task delivers:** Moving the cursor onto the square transitions the game into the playing state. While the cursor is inside the square, a white fill rises from the bottom. While the cursor is outside, the air-time bar drains. The square does **not** yet teleport; the player can just watch the dwell fill complete and over-fill (we'll add the teleport in Task 5). The game does not yet end when air-time hits zero (Task 6).

- [ ] **Step 1: Add the game loop**

Replace the line `// (added in Task 4)` under `// === GAME LOOP ===` with:

```javascript
let lastFrame = 0;
let rafId = 0;

function tick(now) {
  const dt = lastFrame === 0 ? 0 : now - lastFrame;
  lastFrame = now;

  if (state.phase === "playing") {
    const inside = isInside(state.mouse.x, state.mouse.y, state.square);
    renderHoverClass(inside);

    if (inside) {
      state.dwellProgressMs += dt;
      // Catch handling lands in Task 5.
    } else {
      state.dwellProgressMs = 0;
      state.airTimeUsedMs = Math.min(AIRTIME_BUDGET, state.airTimeUsedMs + dt);
      // Game-over handling lands in Task 6.
    }

    renderDwellFill();
    renderHUD();
  }

  rafId = requestAnimationFrame(tick);
}

function startPlaying() {
  if (state.phase !== "idle") return;
  state.phase = "playing";
  document.getElementById("subtitle").style.display = "none";
  lastFrame = 0;
  rafId = requestAnimationFrame(tick);
}
```

- [ ] **Step 2: Trigger play on first hover**

Replace the `attachInput` function inside `// === INPUT ===` with this expanded version:

```javascript
function attachInput() {
  document.addEventListener("mousemove", (ev) => {
    state.mouse.x = ev.clientX;
    state.mouse.y = ev.clientY;
    if (state.phase === "idle" && isInside(state.mouse.x, state.mouse.y, state.square)) {
      startPlaying();
    }
  });
}
```

- [ ] **Step 3: Verify in a browser**

Reload `index.html`. Move the cursor onto the square.
Expected:
- The subtitle disappears as soon as you hover the square.
- A translucent white fill begins rising from the bottom of the square. After ~2s of continuous hovering the fill reaches the top and stays there (no teleport yet — that's Task 5).
- Move the cursor off the square. The fill drops to 0 immediately. The air-time bar starts shrinking from the right; numeric label decreases from `2.00s left`.
- Move back onto the square: bar stops draining, fill starts rising from 0 again.
- Leave the cursor outside for long enough that the bar hits 0. The game does **not** end yet (that's Task 6); the air-time label sits at `0.00s left`, bar is empty and red.
- DevTools console: no errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add game loop with dwell and air-time accumulators"
```

---

## Task 5: Catch, dwell decay, teleport

**Files:**
- Modify: `index.html` (the `tick` function inside `// === GAME LOOP ===`)

**What this task delivers:** When the dwell fill completes, the square teleports to a new safe position (≥150px from cursor), the score increments, and the dwell requirement drops by 100ms (floor 200ms).

- [ ] **Step 1: Add a catch handler**

Add this function inside the `// === GAME LOOP ===` section, directly above `function tick(now)`:

```javascript
function onCatch() {
  state.score += 1;
  state.dwellMs = decayDwell(state.dwellMs);
  state.dwellProgressMs = 0;
  const pos = chooseTeleportPosition(
    { w: window.innerWidth, h: window.innerHeight },
    SQUARE_SIZE,
    state.mouse,
    ANTI_CHEESE_DIST,
    HUD_HEIGHT,
    Math.random
  );
  state.square.x = pos.x;
  state.square.y = pos.y;
  renderSquare();
}
```

- [ ] **Step 2: Trigger the catch inside `tick`**

In the `tick` function, replace this block:

```javascript
    if (inside) {
      state.dwellProgressMs += dt;
      // Catch handling lands in Task 5.
    } else {
```

with:

```javascript
    if (inside) {
      state.dwellProgressMs += dt;
      if (state.dwellProgressMs >= state.dwellMs) {
        onCatch();
      }
    } else {
```

- [ ] **Step 3: Verify in a browser**

Reload `index.html`. Hover the square. Within ~2 seconds the fill completes.
Expected:
- The square jumps to a new position somewhere on the screen below the HUD.
- The new position is visibly away from the cursor (i.e., not under your mouse pointer).
- `Score: 1` in the HUD; `Next: 1900ms`.
- Chase the new square and catch it. `Score: 2`, `Next: 1800ms`, and so on.
- After ~18 consecutive catches, `Next:` stops at `200ms` and stays there.
- Try to drag your cursor very slowly across the screen so a teleport happens right where you are — the square should still appear ≥150px away.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: catch, dwell decay, and anti-cheese teleport"
```

---

## Task 6: Game-over overlay and restart

**Files:**
- Modify: `index.html` (CSS `=== OVERLAY ===`, HTML overlay block, JS `=== GAME LOOP ===`, `=== INPUT ===`)

**What this task delivers:** When `airTimeUsedMs` hits 2000, the loop halts and a centered overlay appears with the final score and a "Play again" button. Clicking it resets all state and returns to the idle screen.

- [ ] **Step 1: Add overlay CSS**

Replace the line `/* (added in Task 6) */` under `/* === OVERLAY === */` with:

```css
#overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.65);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 20px;
  z-index: 20;
}
#overlay[hidden] { display: none; }
#overlay h1 {
  margin: 0;
  font-size: 36px;
  font-weight: 700;
}
#overlay .score {
  font-size: 22px;
  opacity: 0.9;
}
#overlay button {
  font: inherit;
  font-size: 16px;
  font-weight: 600;
  padding: 12px 24px;
  background: #ff5b5b;
  color: #fff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}
#overlay button:hover { filter: brightness(1.1); }
#square.dimmed { opacity: 0.35; }
```

- [ ] **Step 2: Add the overlay markup**

Replace this HTML comment:

```html
  <!-- === GAME OVER OVERLAY === -->
  <!-- (added in Task 6) -->
```

with:

```html
  <!-- === GAME OVER OVERLAY === -->
  <div id="overlay" hidden>
    <h1>Game over.</h1>
    <div class="score">Score: <span id="finalscore">0</span></div>
    <button id="playagain">Play again</button>
  </div>
```

- [ ] **Step 3: Add a game-over handler**

Add this function inside `// === GAME LOOP ===`, directly above `function tick(now)`:

```javascript
function onGameOver() {
  state.phase = "gameover";
  cancelAnimationFrame(rafId);
  rafId = 0;
  document.getElementById("finalscore").textContent = state.score;
  document.getElementById("overlay").hidden = false;
  el.square.classList.add("dimmed");
  renderHoverClass(false);
}

function resetGame() {
  state.phase = "idle";
  state.score = 0;
  state.dwellMs = INITIAL_DWELL;
  state.dwellProgressMs = 0;
  state.airTimeUsedMs = 0;
  state.square.x = Math.floor((window.innerWidth - SQUARE_SIZE) / 2);
  state.square.y = HUD_HEIGHT + Math.floor((window.innerHeight - HUD_HEIGHT - SQUARE_SIZE) / 2);
  document.getElementById("overlay").hidden = true;
  document.getElementById("subtitle").style.display = "";
  el.square.classList.remove("dimmed");
  renderSquare();
  renderHUD();
  renderDwellFill();
}
```

- [ ] **Step 4: Trigger game over from the loop**

In the `tick` function, replace this block:

```javascript
    } else {
      state.dwellProgressMs = 0;
      state.airTimeUsedMs = Math.min(AIRTIME_BUDGET, state.airTimeUsedMs + dt);
      // Game-over handling lands in Task 6.
    }
```

with:

```javascript
    } else {
      state.dwellProgressMs = 0;
      state.airTimeUsedMs = Math.min(AIRTIME_BUDGET, state.airTimeUsedMs + dt);
      if (state.airTimeUsedMs >= AIRTIME_BUDGET) {
        renderHUD();
        onGameOver();
        return;
      }
    }
```

- [ ] **Step 5: Wire the Play-again button**

Inside `attachInput`, append after the `mousemove` listener:

```javascript
  document.getElementById("playagain").addEventListener("click", () => {
    resetGame();
  });
```

- [ ] **Step 6: Verify in a browser**

Reload `index.html`. Play a few catches, then deliberately leave the cursor off the square until the air-time bar empties.
Expected:
- A dark scrim covers the page. Centered card reads `Game over.`, `Score: <n>`, with a coral "Play again" button.
- The square underneath is visibly dimmed.
- Air-time bar is empty and red; numeric label reads `0.00s left`.
- Click "Play again". The overlay disappears, the square recenters, the subtitle reappears, HUD resets to `Score: 0` / `2.00s left` / `Next: 2000ms`. Hovering the square starts a new run.
- Win/lose cycle works repeatedly (do it twice to confirm).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: game-over overlay and restart"
```

---

## Task 7: Window resize safety + final polish pass

**Files:**
- Modify: `index.html` (JS `=== INPUT ===` for resize handler, plus a quick visual audit)

**What this task delivers:** Resizing the browser window while playing won't strand the square off-screen. Final visual review and any small touch-ups.

- [ ] **Step 1: Handle window resize**

Inside `attachInput`, after the `playagain` click listener, append:

```javascript
  window.addEventListener("resize", () => {
    // Keep square fully on-screen and below the HUD after a resize.
    const maxX = Math.max(0, window.innerWidth - SQUARE_SIZE);
    const maxY = Math.max(HUD_HEIGHT, window.innerHeight - SQUARE_SIZE);
    state.square.x = Math.min(state.square.x, maxX);
    state.square.y = Math.max(HUD_HEIGHT, Math.min(state.square.y, maxY));
    renderSquare();
  });
```

- [ ] **Step 2: Verify resize behavior**

Reload `index.html`. Start a game and let the square teleport to the far right or bottom edge. Then drag the window smaller so that the previous square position would be off-screen.
Expected: the square snaps back to remain fully visible inside the smaller window. No console errors.

- [ ] **Step 3: Full-loop manual playtest**

Run through this entire script once and watch for anything off:

1. Open `index.html`.
2. Square is centered. Subtitle visible. HUD: `Score: 0`, full green bar, `2.00s left`, `Next: 2000ms`.
3. Move cursor onto square. Subtitle disappears. Square brightens slightly. White fill rises from bottom.
4. Fill reaches top. Square teleports away from cursor (visibly not under it). `Score: 1`. `Next: 1900ms`.
5. Chase the square. Air-time bar drains while you're outside. Note: bar color shifts green → yellow once it crosses 50%, yellow → red once it crosses 80%.
6. Hover the new square; bar stops draining. Fill rises. Catch it.
7. Repeat until you accumulate ≥18 catches; `Next:` should be stuck at `200ms`.
8. Deliberately let the air-time bar deplete. Game-over overlay appears with your score.
9. Click "Play again". Everything resets cleanly.
10. Resize the window mid-game. Square stays on-screen.

Expected: no console errors, no visual jank (e.g., the square never spawns under the HUD, never spawns off-screen, never under your cursor by accident).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: resize safety and final playtest polish"
```

---

## Done

After Task 7, `index.html` is a complete, self-contained game. Open it in a browser to play. No build, no dependencies, no server required.
