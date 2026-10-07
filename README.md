# PAC-KIRO 🟣

A retro Pac-Man inspired browser game built with vanilla HTML5 Canvas and JavaScript — starring the Kiro logo as the playable character.

![Pac-Kiro Game](kiro-logo.png)

## 🎮 How to Play

- **Arrow keys** to move Kiro through the maze
- Eat all the dots to complete the level
- Eat a **power pellet** (glowing purple orb) to turn ghosts blue and eat them for bonus points
- Avoid ghosts — you have **3 lives**
- Press any arrow key to **start / restart**

## ✨ Features

- **Kiro logo sprite** as the player character with mouth animation and glow ring
- **4 ghosts** with homing AI and 20% random deviation so they're beatable
- **Power pellets** that trigger ghost scared mode (blue ghosts, random movement)
- **Particle system** — purple trail, dot sparkles, explosion bursts, and confetti on a new high score
- **Persistent high score** via `localStorage`
- **Blinking NEW HIGH SCORE label** with confetti burst
- **Screen overlays** for Game Over, Level Complete, and Start screens
- Dark purple Kiro brand theme (`#790ECB`)

## 🚀 Running the Game

Just open `index.html` in any modern browser — no build step, no dependencies.

```bash
# Option 1: double-click index.html
# Option 2: serve locally
npx serve .
# or
python -m http.server 8080
```

Then open `http://localhost:8080` (or `8000` for Python).

## 🏗️ Project Structure

```
proyecto_juego/
├── index.html       # Entire game — HTML, CSS, and JS in one file
├── kiro-logo.png    # Player sprite
└── README.md
```

## 🎯 Scoring

| Action              | Points |
|---------------------|--------|
| Eat a dot           | 10     |
| Eat a power pellet  | 50     |
| Eat a scared ghost  | 200    |

## 🛠️ Built With

- Vanilla HTML5 Canvas API
- Pure JavaScript (no frameworks, no dependencies)
- CSS for HUD and layout

## 🏆 Built at AWS re:Invent with Kiro AI

This game was created during an AWS re:Invent workshop using [Kiro](https://kiro.dev), an AI-powered development environment.
