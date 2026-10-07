# Implementation Plan: game-effects-and-scoring

## Overview

Add high score persistence and four particle/animation systems to the existing single-file `index.html` Pac-Kiro game. All changes are made inside the existing `<script>` block using vanilla JS and the HTML5 Canvas 2D API. Property-based tests (fast-check) validate the seven correctness properties defined in the design document.

---

## Tasks

- [ ] 1. ScoreManager — high score persistence, HUD display, corrupt-data guard
  - [ ] 1.1 Add `ScoreManager` object to the `<script>` block with `load()`, `check()`, and `hideLabel()` methods
    - Read `pac-kiro-highscore` from localStorage inside `load()`; fall back to `0` for missing or non-numeric values and overwrite corrupt entries with `"0"`
    - Wrap all localStorage access in `try/catch` (private-browsing safety)
    - In `check(currentScore)`: if `currentScore > hi`, update `hi`, write to localStorage, set `_labelTimer = 3000`, and set `_newHiTriggered = true`; guard with `_newHiTriggered` so confetti spawns only once per crossing
    - In `hideLabel()`: set `_labelTimer` to `0` and `_labelVisible` to `false`
    - Call `ScoreManager.load()` at the top of `initGame()`; reset `_newHiTriggered = false` on game reset
    - _Requirements: 1.1, 1.2, 1.4, 1.5, 1.6_

  - [ ] 1.2 Extend the HUD HTML and `hud()` function to display the high score
    - Add `<span>HI: <span id="hi">0</span></span>` to the `#hud` div
    - Update `hud()` to set `hiEl.textContent = ScoreManager.hi`
    - Wire `hiEl` constant alongside `scEl` and `lvEl`
    - _Requirements: 1.3, 1.5_

  - [ ]* 1.3 Write property tests for ScoreManager (Properties 1, 2, 3, 4)
    - Set up a `test/scoreManager.test.js` file; import fast-check (`fc`) and the extracted ScoreManager logic (or use jsdom to load index.html)
    - **Property 1: High score load round-trip** — generate random non-negative integer `n`, set localStorage, call `load()`, assert `ScoreManager.hi === n` _(Validates: Req 1.1)_
    - **Property 2: Corrupt data falls back to zero** — generate arbitrary non-integer strings, assert `hi === 0` and localStorage reset to `"0"` _(Validates: Req 1.6)_
    - **Property 3: High score monotonically non-decreasing** — generate array of non-negative integers, run `check()` for each, assert `hi === max(initial, s[0..i])` at every step _(Validates: Req 1.4)_
    - **Property 4: Confetti fires exactly once per crossing** — mock `spawnConfetti`, generate initial hi + score sequence crossing it once, assert call count === 1 _(Validates: Req 5.5)_
    - _Requirements: 1.1, 1.4, 1.6, 5.5_

- [ ] 2. ParticleSystem foundation — pool, update/draw loop, game loop integration
  - [ ] 2.1 Add `ParticleSystem` object with `pool[]`, `update(dt)`, and `draw(ctx)` to the `<script>` block
    - `update(dt)`: iterate pool, advance `age += dt`, compute `alpha = 1 - age/lifetime` (clamped to [0,1]), apply `vx`/`vy` to `x`/`y`; mark `dead = true` when `age >= lifetime`; filter out dead particles at end of update
    - `draw(ctx)`: guard `if (!ctx) return`; for each particle dispatch on `type`: `'trail'`, `'explosion'`, `'sparkle'` → filled circle scaled by current `alpha`; `'confetti'` → `save/translate/rotate/fillRect/restore`
    - _Requirements: 2.2, 2.3, 3.3, 3.4, 4.3, 4.5, 5.3_

  - [ ] 2.2 Integrate ParticleSystem into the game loop with a dt-capped delta
    - Track `let lastTime = 0` and compute `dt = Math.min(timestamp - lastTime, 100)` at the top of `loop(timestamp)`; update `lastTime = timestamp`
    - Call `ParticleSystem.update(dt)` after `checkCollision()` and before `drawPlayer()`
    - Call `ParticleSystem.draw(cx)` after `drawGhosts()` and before any overlay drawing
    - Decrement `ScoreManager._labelTimer` by `dt` each frame and set `_labelVisible = _labelTimer > 0`; call `ScoreManager.hideLabel()` when entering `S_OVER` or `S_WIN` states
    - _Requirements: 2.3, 2.5, 3.7, 5.6, 5.7_

  - [ ] 2.3 Checkpoint — verify game still runs with no visual regressions before adding particle spawning
    - Ensure all existing gameplay (movement, dots, ghosts, score, lives) works after integration; HI: displays in HUD

- [ ] 4. Trail, explosion, and sparkle particles — spawn helpers hooked to game events
  - [ ] 4.1 Implement `ParticleSystem.spawnTrail(x, y)` and hook into `updatePlayer()`
    - Emit 1–3 TrailParticles per call only when `player.dc !== 0 || player.dr !== 0`
    - Each particle: `type:'trail'`, `color:'#790ECB'`, `radius` ∈ [3,6], `lifetime:300`, `alpha` starts at `0.6`, `vx`/`vy` random drift ∈ [−1,1] px/frame (magnitude ≤ 1)
    - Call `spawnTrail(px, py)` at the end of `updatePlayer()` using the player's canvas centre coordinates
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.6_

  - [ ] 4.2 Implement `ParticleSystem.spawnExplosion(x, y, scared)` and hook into `checkCollision()`
    - Emit 12–20 ExplosionParticles; directions uniformly distributed across 360°, speed ∈ [2,5] px/frame
    - Normal collision colours: `['#790ECB','#c084fc','#ef4444','#ffffff']`; scared-ghost kill colours: `['#790ECB','#c084fc','#a5f3fc']`
    - Each particle: `radius` ∈ [4,8], `lifetime` ∈ [300,600] ms, `alpha` linear 1→0
    - Guard with `if (!cx) return` (Req 3.7); allow concurrent bursts to coexist (Req 3.6)
    - Call `spawnExplosion(gx, gy, g.sc > 0)` in `checkCollision()` at both the eaten-ghost and loseLife branches
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_

  - [ ] 4.3 Implement `ParticleSystem.spawnSparkle(x, y, isPower)` and hook into `updatePlayer()`
    - Dot eat: emit 3–5 SparkleParticles; power pellet eat: emit 8–12
    - Each particle: `type:'sparkle'`, `color:'#c084fc'`, `radius` ∈ [2,5], `lifetime:400`, `alpha` linear 1→0, speed ∈ [1,3] px/frame across 360°
    - Enforce sparkle cap of 100: count active sparkles before emitting; silently drop excess (do not remove existing particles)
    - Call at the `cell===DOT` and `cell===POWER` branches inside `updatePlayer()`
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ]* 4.4 Write property tests for particle lifecycle (Properties 5, 7)
    - **Property 5: Alpha reaches zero at end of lifetime** — generate random lifetime ∈ [300,600]; create a particle and simulate `update()` calls until `age >= lifetime`; assert `alpha ≈ 0` (within 0.001) and `dead === true` _(Validates: Req 2.3, 3.4, 4.5)_
    - **Property 7: Sparkle cap preserves existing particles** — generate existing sparkle count (0–100) and batch size; call `spawnSparkle`; assert existing particles unchanged and total sparkle count ≤ 100 _(Validates: Req 4.6)_
    - _Requirements: 2.3, 3.4, 4.5, 4.6_

- [ ] 5. Confetti burst and NEW HIGH SCORE! blinking label
  - [ ] 5.1 Implement `ParticleSystem.spawnConfetti()` and trigger it from `ScoreManager.check()`
    - Emit 60–100 ConfettiParticles: `x` uniform across [0, W], `y = 0`, `vy` ∈ [2,5] px/frame, `vx` ∈ [−1,1] px/frame, `spin` ∈ [−0.1,0.1] rad/frame
    - Each particle: `w` ∈ [6,10], `h` ∈ [4,7], `color` fixed from `['#790ECB','#c084fc','#ffffff','#a78bfa','#fbbf24']`, `lifetime: Infinity`, `dead` set to `true` only when `y > H` (handled in `update()` with special `lifetime === Infinity` branch)
    - Inside `ScoreManager.check()`: when `currentScore > hi` and `!_newHiTriggered`, call `ParticleSystem.spawnConfetti()` and set `_newHiTriggered = true`
    - Reset `_newHiTriggered = false` in `initGame()` so confetti can fire again in a new game
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ] 5.2 Implement `drawHiScoreLabel(ctx)` and wire it into the game loop
    - When `ScoreManager._labelVisible` is `true`: compute blink phase as `Math.floor(ScoreManager._labelTimer / 500) % 2`; draw `NEW HIGH SCORE!` centred on canvas in `#fbbf24` only when phase is odd (on-phase); font `bold 22px "Courier New"`, positioned at `(W/2, 40)`
    - Draw this label after `ParticleSystem.draw(cx)` and before game-over/win overlays so it appears above particles but below full-screen overlays
    - On re-trigger (score crosses a higher threshold): `ScoreManager.check()` resets `_labelTimer = 3000` without interrupting display
    - _Requirements: 5.6, 5.7, 5.8_

  - [ ]* 5.3 Write property test for confetti lifecycle (Property 6)
    - **Property 6: Confetti removed only when below canvas** — generate a sequence of increasing y-positions for a confetti particle; simulate `update()` calls; assert `dead === false` for all frames where `y ≤ H` and `dead === true` on the first frame where `y > H` _(Validates: Req 5.4)_
    - _Requirements: 5.4_

- [ ] 6. Final checkpoint — full integration verification
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- All code changes are inside the single `index.html` `<script>` block — no new files except the optional test file
- Property tests use **fast-check** (`npm install --save-dev fast-check vitest jsdom`); run with `vitest --run`
- Each property test is tagged with `// Feature: game-effects-and-scoring, Property <N>: <text>`
- Checkpoints (tasks 3 and 6) are visual sanity checks, not automated tests
- The `dt` cap of 100 ms in the game loop prevents particle jumps when the tab is backgrounded

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "1.3"] },
    { "id": 2, "tasks": ["2.2"] },
    { "id": 3, "tasks": ["4.1", "4.2", "4.3"] },
    { "id": 4, "tasks": ["4.4", "5.1"] },
    { "id": 5, "tasks": ["5.2"] },
    { "id": 6, "tasks": ["5.3"] }
  ]
}
```
