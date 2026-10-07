# Requirements Document

## Introduction

This feature adds score persistence and visual particle effects to Pac-Kiro, a single-file HTML5 Canvas Pac-Man style game. The enhancements cover two areas: (1) saving game history by persisting the high score across browser sessions using localStorage, and (2) enriching gameplay feel with four particle/animation systems — a movement trail behind the player, explosions on ghost collisions, sparkles when eating dots or power pellets, and a confetti celebration when a new high score is achieved.

All code lives in a single `index.html` file with vanilla JavaScript and the HTML5 Canvas 2D API. No build tools or external frameworks are used.

## Glossary

- **Game**: The Pac-Kiro HTML5 Canvas game running in the browser.
- **Player**: The Kiro-logo sprite controlled by the user.
- **Ghost**: An enemy sprite that chases the Player through the maze.
- **Dot**: A small collectible pellet worth 10 points placed throughout the maze.
- **PowerPellet**: A large collectible pellet worth 50 points that temporarily makes Ghosts frightened.
- **ScoreManager**: The logical component responsible for tracking, comparing, and persisting scores.
- **ParticleSystem**: The logical component responsible for spawning, updating, and drawing particle effects on the Canvas.
- **TrailParticle**: A short-lived particle emitted continuously at the Player's position while moving.
- **ExplosionParticle**: A burst of particles emitted at the collision point when the Player is hit by a Ghost.
- **SparkleParticle**: A short-lived shimmer particle emitted when the Player eats a Dot or PowerPellet.
- **ConfettiParticle**: A falling rectangular particle emitted across the Canvas when the player achieves a new high score.
- **HUD**: The on-screen display showing SCORE and LIVES.
- **localStorage**: The browser's Web Storage API used to persist data between sessions.

---

## Requirements

### Requirement 1: High Score Persistence

**User Story:** As a player, I want my best score to be saved between browser sessions, so that I can track my personal record even after closing the tab.

#### Acceptance Criteria

1. WHEN the Game initialises, THE ScoreManager SHALL read the stored high score value from localStorage key `pac-kiro-highscore` and hold it in memory as the current session high score.
2. IF no value exists in localStorage under `pac-kiro-highscore`, THEN THE ScoreManager SHALL initialise the in-memory high score to `0`.
3. THE HUD SHALL display the high score alongside the current score and lives count in the format `HI: <value>`, where `<value>` is a non-negative integer with no leading zeros (e.g. `HI: 0`, `HI: 1500`).
4. WHEN the player's current score exceeds the in-memory high score, THE ScoreManager SHALL update the in-memory high score to equal the current score and write the new value to localStorage key `pac-kiro-highscore` as a base-10 integer string.
5. WHEN the Game is reset after a game-over, THE HUD SHALL retain and continue to display the high score loaded from localStorage, not reset it to `0`.
6. IF the value read from localStorage under `pac-kiro-highscore` is not a valid non-negative integer (e.g. corrupt data, `NaN`, non-numeric string), THEN THE ScoreManager SHALL fall back to `0`, overwrite the corrupt entry in localStorage with `"0"`, and proceed normally.

---

### Requirement 2: Player Movement Trail

**User Story:** As a player, I want to see a glowing trail behind Kiro as it moves, so that fast movement feels dynamic and satisfying.

#### Acceptance Criteria

1. WHILE the Player is moving (current direction vector is non-zero), THE ParticleSystem SHALL emit between 1 and 3 TrailParticles per rendered frame at the Player's current canvas position.
2. THE ParticleSystem SHALL render each TrailParticle as a filled circle with an initial radius between 3 px and 6 px (assigned randomly at emission time), using the brand purple colour `#790ECB` with an alpha value that decreases linearly from `0.6` to `0` over the particle's lifetime of 300 milliseconds.
3. THE ParticleSystem SHALL remove each TrailParticle from the active pool after it has existed for exactly 300 milliseconds, measured from its emission timestamp.
4. WHILE the Player is stationary (direction vector is zero), THE ParticleSystem SHALL NOT emit new TrailParticles.
5. THE ParticleSystem SHALL support at least 200 simultaneously active TrailParticles without causing the frame rate to drop below 30 FPS on a browser running on a device with at least a dual-core CPU at 2 GHz and 4 GB RAM.
6. THE ParticleSystem SHALL assign each TrailParticle a small random drift velocity (magnitude between 0 px/frame and 1 px/frame in a random direction) so that stacked particles spread slightly and remain visually distinct.

---

### Requirement 3: Ghost Collision Explosion

**User Story:** As a player, I want to see an explosion when Kiro gets hit by a ghost, so that the collision feels impactful and I immediately understand what happened.

#### Acceptance Criteria

1. WHEN the Player occupies the same maze cell as a non-frightened Ghost, THE ParticleSystem SHALL emit a burst of between 12 and 20 ExplosionParticles centred on the collision canvas position.
2. THE ParticleSystem SHALL assign each ExplosionParticle an initial velocity with a direction uniformly distributed across 360° and a speed between 2 px/frame and 5 px/frame.
3. THE ParticleSystem SHALL render each ExplosionParticle as a filled circle with an initial radius between 4 px and 8 px using colours sampled uniformly from `['#790ECB', '#c084fc', '#ef4444', '#ffffff']`, with alpha decreasing linearly from `1.0` to `0.0` over the particle's lifetime.
4. THE ParticleSystem SHALL assign each ExplosionParticle a lifetime chosen randomly between 300 milliseconds and 600 milliseconds at spawn time, and SHALL remove it from the active pool when that lifetime expires.
5. WHEN the Player eats a frightened Ghost, THE ParticleSystem SHALL emit a burst of between 12 and 20 ExplosionParticles using only the colours `['#790ECB', '#c084fc', '#a5f3fc']` to visually distinguish it from a lethal collision.
6. WHEN a second collision event fires while a previous ExplosionParticle burst is still active, THE ParticleSystem SHALL emit the new burst independently and allow both bursts to coexist until their respective lifetimes expire.
7. IF the canvas 2D rendering context is unavailable when an explosion is triggered, THE ParticleSystem SHALL silently skip particle emission without throwing an unhandled exception.

---

### Requirement 4: Dot and Power Pellet Sparkle

**User Story:** As a player, I want sparkle effects when I eat dots and power pellets, so that collecting them feels rewarding.

#### Acceptance Criteria

1. WHEN the Player eats a Dot, THE ParticleSystem SHALL emit between 3 and 5 SparkleParticles at the Dot's last known canvas position.
2. WHEN the Player eats a PowerPellet, THE ParticleSystem SHALL emit between 8 and 12 SparkleParticles at the PowerPellet's last known canvas position.
3. THE ParticleSystem SHALL render each SparkleParticle as a filled circle with an initial radius between 2 px and 5 px (assigned randomly at emission time) using colour `#c084fc`, with alpha decreasing linearly from `1.0` to `0.0` over the particle's full lifetime.
4. THE ParticleSystem SHALL assign each SparkleParticle a velocity with direction uniformly distributed across 360° and speed between 1 px/frame and 3 px/frame.
5. THE ParticleSystem SHALL remove each SparkleParticle from the active pool after exactly 400 milliseconds, measured from its emission timestamp, regardless of current frame rate.
6. WHEN the total number of active SparkleParticles would exceed 100, THE ParticleSystem SHALL drop newly emitted sparkles (not remove existing ones) until the count falls below the cap.

---

### Requirement 5: New High Score Confetti

**User Story:** As a player, I want a confetti celebration when I beat my high score, so that achieving a new record feels rewarding and memorable.

#### Acceptance Criteria

1. WHEN the player's current score first exceeds the previous session high score during active gameplay, THE ParticleSystem SHALL emit a burst of between 60 and 100 ConfettiParticles with initial x-positions distributed uniformly across the full canvas width and initial y-position set to `0`.
2. THE ParticleSystem SHALL assign each ConfettiParticle an initial vertical velocity between 2 px/frame and 5 px/frame (downward), a horizontal drift between -1 px/frame and 1 px/frame, and a rotation speed between -0.1 rad/frame and 0.1 rad/frame.
3. THE ParticleSystem SHALL render each ConfettiParticle as a filled rectangle of width between 6 px and 10 px and height between 4 px and 7 px, using one colour assigned uniformly at random from `['#790ECB', '#c084fc', '#ffffff', '#a78bfa', '#fbbf24']` at spawn time; the colour SHALL NOT change during the particle's lifetime.
4. THE ParticleSystem SHALL remove each ConfettiParticle from the active pool after its y-position exceeds the canvas height, and SHALL NOT remove it before that condition is met.
5. WHEN the score crosses a new persisted high score value on the first frame of that crossing, THE ParticleSystem SHALL emit exactly one confetti burst; it SHALL NOT emit additional bursts while the score continues to increase beyond that same threshold.
6. THE HUD SHALL display a blinking text label `NEW HIGH SCORE!` in colour `#fbbf24` immediately when a new high score is achieved; the label SHALL blink at a 500 ms on / 500 ms off interval and SHALL disappear after 3 seconds, measured from the moment the new high score was first detected.
7. WHEN the game ends (game-over or level complete) before the 3-second label display period expires, THE HUD SHALL immediately hide the `NEW HIGH SCORE!` label.
8. WHEN a second new high score event fires while the `NEW HIGH SCORE!` label is already visible, THE HUD SHALL reset the 3-second countdown from the moment of the new event without any visible interruption to the label display.
