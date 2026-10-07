# Design Document: game-effects-and-scoring

## Overview

This document describes the technical design for adding **high score persistence** and **four particle/animation systems** to Pac-Kiro — a single-file HTML5 Canvas Pac-Man clone.

All new code lives inside the existing `index.html` `<script>` block. There are no external dependencies, build steps, or module boundaries. The design therefore favours plain object literals, small focused functions, and a single shared particle pool that drives every effect type from one unified update/draw loop.

The two main additions are:

- **ScoreManager** — a thin wrapper around `localStorage` that loads, compares, and persists the high score while guarding against corrupt stored data.
- **ParticleSystem** — a unified pool of particle objects, each carrying a `type` discriminator, that handles spawning, updating (physics + alpha decay), drawing, and pruning for all four effect flavours: trail, explosion, sparkle, and confetti.

A secondary concern is extending the **HUD** to display the high score and a timed, blinking "NEW HIGH SCORE!" label.

---

## Architecture

The design follows a **data-oriented, flat-object** style that matches the existing codebase. There are no classes, no inheritance hierarchies, and no event buses. New state is appended to the existing global scope alongside `score`, `lives`, and `ghosts`.

```
index.html
└── <script>
    ├── constants (unchanged + new particle config constants)
    ├── ScoreManager          ← new: localStorage read/write, hi-score tracking
    ├── ParticleSystem        ← new: pool[], spawn functions, update(), draw()
    ├── HUD helpers           ← extended: hud(), newHiScoreLabel state
    ├── maze / player / ghost logic (unchanged)
    └── game loop (loop())    ← extended: call particle update/draw each frame
```

**Data flow per frame:**

```
loop()
  ├─ frame++
  ├─ cx.clearRect(...)
  ├─ drawMaze()
  ├─ [S_PLAY] updatePlayer()   → calls ScoreManager.check(score)
  │                             → calls spawnSparkle / spawnExplosion
  ├─ [S_PLAY] updateGhosts()
  ├─ [S_PLAY] checkCollision() → calls spawnExplosion
  ├─ ParticleSystem.update()   ← advances physics, prunes dead particles
  ├─ drawPlayer()
  ├─ drawGhosts()
  ├─ ParticleSystem.draw(cx)   ← renders all live particles
  └─ overlay / HUD label drawing
```

**Rationale for draw-after-gameplay:** Trail and explosion particles should render on top of the maze and ghosts but below overlays (game-over, win), so `ParticleSystem.draw()` is placed after `drawPlayer()` / `drawGhosts()` and before any screen-covering overlay.

---

## Components and Interfaces

### ScoreManager

A plain object with four properties and three methods, all living in global scope.

```js
// Initialised once; referenced globally.
const ScoreManager = {
  hi: 0,              // in-memory high score for the current session
  _newHiTriggered: false,  // prevents duplicate confetti bursts per crossing
  _labelTimer: 0,     // countdown in milliseconds for NEW HIGH SCORE! label
  _labelVisible: false,

  /** Call once at page load / game init. */
  load() { … },

  /**
   * Call every frame (or after every score change).
   * Updates hi, writes localStorage, sets label state, triggers confetti.
   * @param {number} currentScore
   */
  check(currentScore) { … },

  /** Call when game-over or level-complete overlay is shown. */
  hideLabel() { … }
};
```

### ParticleSystem

A plain object wrapping a flat array pool.

```js
const ParticleSystem = {
  pool: [],   // Particle[]

  /** Advance all particles by dt milliseconds; prune dead ones. */
  update(dt) { … },

  /** Render all live particles onto context ctx. */
  draw(ctx) { … },

  // ── spawn helpers ──────────────────────────────────────────────────────────
  spawnTrail(x, y) { … },
  spawnExplosion(x, y, scared) { … },
  spawnSparkle(x, y, isPower) { … },
  spawnConfetti() { … }
};
```

### HUD

The existing `hud()` function is extended to write a third element (the high score). A separate overlay drawing step handles the `NEW HIGH SCORE!` label so it renders above particles but below game-over/win screens.

```js
// Existing DOM refs extended:
const hiEl = document.getElementById('hi');   // new <span> in HTML

function hud() {
  scEl.textContent = score;
  lvEl.textContent = lives;
  hiEl.textContent = ScoreManager.hi;
}

function drawHiScoreLabel(ctx) {
  // Only called when ScoreManager._labelVisible is true.
  // Blink at 500 ms on/off using ScoreManager._labelTimer.
}
```

---

## Data Models

### Particle (shared fields for all types)

Every particle in the pool is a plain JS object with these mandatory fields:

| Field | Type | Description |
|---|---|---|
| `type` | `'trail'` \| `'explosion'` \| `'sparkle'` \| `'confetti'` | Discriminator used by draw() |
| `x` | `number` | Current canvas x position (px) |
| `y` | `number` | Current canvas y position (px) |
| `vx` | `number` | Horizontal velocity (px/frame) |
| `vy` | `number` | Vertical velocity (px/frame) |
| `alpha` | `number` | Current opacity [0, 1] |
| `age` | `number` | Milliseconds since emission |
| `lifetime` | `number` | Total lifespan in milliseconds |
| `dead` | `boolean` | Set to `true` when the particle should be pruned |

### TrailParticle (extra fields)

| Field | Type | Description |
|---|---|---|
| `radius` | `number` | Initial radius in px, randomly chosen in [3, 6] |
| `color` | `string` | Always `'#790ECB'` |

### ExplosionParticle (extra fields)

| Field | Type | Description |
|---|---|---|
| `radius` | `number` | Initial radius in px, randomly chosen in [4, 8] |
| `color` | `string` | One of `['#790ECB','#c084fc','#ef4444','#ffffff']` (normal) or `['#790ECB','#c084fc','#a5f3fc']` (scared kill) |

### SparkleParticle (extra fields)

| Field | Type | Description |
|---|---|---|
| `radius` | `number` | Initial radius in px, randomly chosen in [2, 5] |
| `color` | `string` | Always `'#c084fc'` |

### ConfettiParticle (extra fields)

| Field | Type | Description |
|---|---|---|
| `w` | `number` | Rectangle width in px, randomly chosen in [6, 10] |
| `h` | `number` | Rectangle height in px, randomly chosen in [4, 7] |
| `angle` | `number` | Current rotation angle in radians |
| `spin` | `number` | Rotation speed in rad/frame, in [−0.1, 0.1] |
| `color` | `string` | One of `['#790ECB','#c084fc','#ffffff','#a78bfa','#fbbf24']`, fixed at spawn |

Confetti `lifetime` is effectively infinite (not time-based): `dead` is set to `true` only when `y > H` (canvas height). The `lifetime` field is set to `Infinity` to signal this to the update loop.

### ScoreManager state

| Field | Type | Description |
|---|---|---|
| `hi` | `number` | Current session high score (non-negative integer) |
| `_newHiTriggered` | `boolean` | `true` after confetti fires for the current crossing; reset when score drops (new game) |
| `_labelTimer` | `number` | Milliseconds remaining for `NEW HIGH SCORE!` label; ≤ 0 means hidden |
| `_labelVisible` | `boolean` | Derived convenience flag: `_labelTimer > 0` |

### localStorage schema

| Key | Value type | Notes |
|---|---|---|
| `pac-kiro-highscore` | String representing a base-10 non-negative integer | e.g. `"1500"`. Corrupt or non-numeric values are replaced with `"0"`. |

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: High score load round-trip

*For any* non-negative integer value written to `localStorage['pac-kiro-highscore']` before `ScoreManager.load()` is called, the resulting `ScoreManager.hi` SHALL equal that integer.

**Validates: Requirements 1.1**

### Property 2: Corrupt data falls back to zero

*For any* string stored in `localStorage['pac-kiro-highscore']` that is not a valid non-negative integer (empty string, `'NaN'`, alphabetic text, negative numbers), `ScoreManager.load()` SHALL set `ScoreManager.hi` to `0` and overwrite the key with `"0"`.

**Validates: Requirements 1.6**

### Property 3: High score update is monotonically non-decreasing

*For any* sequence of `ScoreManager.check(s)` calls with scores `s₁, s₂, …, sₙ`, `ScoreManager.hi` at step `i` SHALL equal `max(hiInitial, s₁, …, sᵢ)` — it can never decrease during a session.

**Validates: Requirements 1.4**

### Property 4: Confetti fires exactly once per high score crossing

*For any* sequence of `ScoreManager.check(s)` calls within a single game session where the score crosses the previous high score threshold exactly once, `ParticleSystem.spawnConfetti()` SHALL have been called exactly once across the entire sequence.

**Validates: Requirements 5.5**

### Property 5: Particle alpha reaches zero at end of lifetime

*For any* TrailParticle, ExplosionParticle, or SparkleParticle `p` emitted at time `t₀`, at time `t₀ + p.lifetime` the particle's `alpha` SHALL equal `0.0` (within floating-point tolerance of 0.001) and `p.dead` SHALL be `true`.

**Validates: Requirements 2.3, 3.4, 4.5**

### Property 6: Confetti removed only when below canvas

*For any* ConfettiParticle `p`, `p.dead` SHALL be `false` for every frame in which `p.y ≤ H` and SHALL be set to `true` on the first frame in which `p.y > H`.

**Validates: Requirements 5.4**

### Property 7: Sparkle cap preserves existing particles

*For any* call to `spawnSparkle` that would push the active sparkle count above 100, the particles already in the pool SHALL remain untouched (none removed), and only the newly requested sparkles SHALL be silently dropped until the count falls below the cap.

**Validates: Requirements 4.6**

---

## Error Handling

| Scenario | Handling strategy |
|---|---|
| `localStorage` read throws (private browsing, quota) | Wrap `ScoreManager.load()` and `check()` in `try/catch`; silently fall back to `hi = 0` for the session without writing anything |
| `localStorage` write throws | Silently swallow the exception; in-memory `hi` stays correct even if persistence fails |
| Corrupt `pac-kiro-highscore` value | `parseInt(val, 10)` with `isNaN` / `< 0` guard → reset to `0` and overwrite |
| Canvas 2D context unavailable during particle draw | Guard with `if (!ctx) return;` at the top of `ParticleSystem.draw()` and each `spawn*` helper |
| Canvas context unavailable during explosion spawn | Silently skip emission (Requirement 3.7) |
| `requestAnimationFrame` `dt` spike (tab backgrounded) | Cap `dt` at 100 ms in the `loop()` delta calculation to prevent particles jumping across the canvas on resume |

---

## Testing Strategy

This feature is implemented entirely in vanilla JS with no module system, so tests must either run inside the browser or use a lightweight test runner that can evaluate the logic functions in isolation (e.g., Vitest or Jest with jsdom, or a simple in-page test harness).

### Unit tests (example-based)

Focus on the pure logic layer that does not require a live canvas or real DOM:

- `ScoreManager.load()` with a valid stored value → `hi` equals that value
- `ScoreManager.load()` with no stored value → `hi` is `0`
- `ScoreManager.load()` with a corrupt value (`"abc"`, `"NaN"`, `"-5"`) → `hi` is `0`, localStorage reset to `"0"`
- `ScoreManager.check(score)` where `score > hi` → `hi` updated, localStorage written, `_labelTimer` set to 3000
- `ScoreManager.check(score)` called twice above threshold → confetti only spawned once (`_newHiTriggered` guard)
- `ScoreManager.hideLabel()` → `_labelTimer` set to `≤ 0`
- Particle alpha interpolation helper: `alphaAt(age, lifetime)` → `1 - age/lifetime`, clamped to [0, 1]

### Property-based tests

The feature involves logic (score comparison, alpha decay, capping) that holds universally across all valid inputs, making it a good candidate for property-based testing. Use **fast-check** (JS/TS) as the PBT library, configured with at least 100 iterations per property.

Each test is tagged with a comment:
`// Feature: game-effects-and-scoring, Property <N>: <property text>`

**Property 1 — High score load round-trip**
Generate: random non-negative integer `n`
Setup: `localStorage.setItem('pac-kiro-highscore', String(n))`
Assert: after `ScoreManager.load()`, `ScoreManager.hi === n`
*(100 iterations, arbitrary non-negative integers)*

**Property 2 — Corrupt data falls back to zero**
Generate: arbitrary strings that do not parse as non-negative integers (e.g. `fc.string()` filtered to exclude valid integers)
Assert: `ScoreManager.hi === 0` and `localStorage.getItem('pac-kiro-highscore') === '0'`

**Property 3 — High score monotonically non-decreasing**
Generate: array of non-negative integers representing a sequence of score values
Assert: at each step `i`, `ScoreManager.hi === max(initial, s[0..i])`
*(100 iterations, sequences of length 1–50)*

**Property 4 — Confetti fires exactly once per crossing**
Generate: initial `hi` value and a sequence of scores that cross it exactly once
Assert: mock `spawnConfetti` call count === 1
*(100 iterations)*

**Property 5 — Alpha reaches zero at end of lifetime**
Generate: random lifetime in [300, 600] ms and sample `age === lifetime`
Assert: `alphaAt(age, lifetime) ≈ 0` (within 0.001) and particle marked dead
*(100 iterations)*

**Property 6 — Confetti removed only when below canvas**
Generate: sequence of y-positions for a confetti particle (increasing)
Assert: `dead === false` while `y ≤ H`, `dead === true` on first frame `y > H`
*(100 iterations)*

**Property 7 — Sparkle cap preserves existing particles**
Generate: existing pool size (0–100) and a new sparkle batch size
Assert: if `existingSparkleCount + batch > 100`, only `max(0, 100 - existingCount)` new sparkles are added; existing sparkles are untouched
*(100 iterations)*
