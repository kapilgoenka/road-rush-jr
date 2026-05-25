# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Road Rush Jr.** is a browser-based arcade driving game built with vanilla HTML, Canvas, and Web Audio API. The entire game is contained in a single `index.html` file (~1100 lines) with no external dependencies or build system.

## Quick Start

**Preview the game:**
1. Run `python3 preview_start Python HTTP Server` (or use any HTTP server on port 3000)
2. Open http://localhost:3000 in your browser
3. Or simply open `index.html` directly in any browser

**Edit the game:**
- All code is in `index.html` (lines 1–1103)
- Sections are marked with `// ─── Section Name` headers for easy navigation
- No build step; changes apply immediately when you refresh the browser

## Code Architecture

The game is structured as a single-file canvas application with the following major systems:

### 1. **Canvas & Rendering (Lines 25–39)**
- Game dimensions: 480×720px (vertical/portrait orientation)
- Responsive scaling to fit window size
- Canvas 2D context for all drawing

### 2. **Audio System (Lines 41–122)**
- Web Audio API for sound effects and continuous engine hum
- Tone generator with frequency sweeps, envelope shaping, and oscillator types
- Sound functions: `sndStar()`, `sndHit()`, `sndTierUp()`, etc.
- `muted` flag controls audio globally; mute toggle in top-right corner

### 3. **State Machine (Lines 124–127)**
- Three states: `START` (title screen), `PLAYING` (active game), `GAME_OVER` (end screen)
- Single `gameState` variable; `setState(s)` for transitions
- Screens and input handling vary per state

### 4. **Input System (Lines 129–207)**
- **Keyboard:** ArrowLeft/ArrowRight or A/D for movement; Space/Enter to start/restart
- **Mouse/Touch:** Click mute button (top-right); click start button (on title screen); on-screen LEFT/RIGHT buttons below the road (mobile)
- Touch button logic converts canvas coordinates from viewport coords

### 5. **Game Objects**

#### Player Car (Lines 231–244)
- Single car sprite with `x`, `y` velocity/acceleration
- Lane-based movement (3 lanes: x ≈ 80, 240, 400px)
- Colors randomized per game (e.g., red, blue, yellow, purple)

#### Obstacles (Lines 246–258)
- Spawned randomly in each of 3 lanes with configurable spawn rates
- Each obstacle has: `x` (lane), `y` (position), `w` (width), `h` (height), `speed`
- Collision detection: simple AABB (axis-aligned bounding box)

#### Stars (Lines 260–272)
- Bonus pickups that increment score and tier progress
- Spawn rate scales with difficulty tier
- Removed on collision with player

#### Particles & Effects (Lines 274–334, 873–887)
- Explosion particles on obstacle hit: spawn as small circles, fade out over time
- Screen shake on hit: temporarily offset canvas position for visual impact
- Bounce animation on collision: brief upward velocity dampening

### 6. **Difficulty Progression (Lines 357–383)**
- Three tiers: Tier 1 (base), Tier 2 (higher speed), Tier 3 (even faster)
- `tierUpThreshold`: star count needed to advance to next tier
- Each tier increases obstacle speed and spawn rate

### 7. **Game Loop (Lines 384–505)**
- `update()`: advance physics, spawn objects, check collisions, update game state
- Main loop: `requestAnimationFrame(gameLoop)` for 60 FPS
- On collision: decrement lives, trigger bounce/screen shake, emit particles
- Game over when lives reach 0

### 8. **Rendering (Lines 506–915)**
- **Road:** scrolling background with lane markings
- **Obstacles:** rectangles colored by lane
- **Player car:** outlined polygon (rotated rectangle) with dynamic color
- **HUD:** lives (hearts), score, tier indicator
- **Touch buttons:** left/right navigation buttons during gameplay
- **Tier-up banner:** flash animation when advancing difficulty

## Key Functions

| Function | Purpose |
|----------|---------|
| `startGame()` | Initialize new game: reset lives, score, obstacles, player state; set PLAYING state |
| `update()` | Main game logic: update positions, spawning, collision checks, state transitions |
| `gameLoop()` | Request animation frame; call `update()` and all draw functions |
| `draw*()` (multiple) | Render specific game elements: road, obstacles, player, HUD, particles |
| `playTone()` | Play single sound effect with frequency sweep and envelope |
| `snd*()` | High-level sound effect wrappers (star, hit, tier-up) |
| `ensureAudio()` | Lazy-init Web Audio context (browser requires user interaction) |

## Tuning Game Parameters

Common values to adjust for gameplay balance:

| Variable | Line | Current | Effect |
|----------|------|---------|--------|
| `GAME_W`, `GAME_H` | 28–29 | 480, 720 | Canvas size (aspect ratio) |
| `livesStart` | 358 | 3 | Initial lives on game start |
| `spawnRate` | 363 | 0.02 | Probability obstacle spawns per frame (increase = more frequent) |
| `spawnRateIncrease` | 365 | [0.02, 0.03, 0.04] | Spawn rate per tier |
| `speed` | 361 | [3, 4.5, 6] | Obstacle speed per tier |
| `tierUpThreshold` | 359 | 10 | Stars needed to advance tier |
| `carLaneX` | 232–234 | [80, 240, 400] | X-coordinates of 3 driving lanes |
| `playerSpeed` | 242 | 5 | Player lateral (lane-change) acceleration |

Adjust these to change difficulty curve, spawn frequency, or lane layout.

## Development Notes

### Adding Features
- **New sound effects:** Add a `snd*()` function calling `playTone()` with frequency, duration, waveform type
- **New visual effects:** Add to particle system or screen shake logic in `update()`; render in main draw functions
- **New game mechanics:** Implement in `update()` and render in corresponding draw function
- **Obstacle types:** Create new spawn logic in `update()`; add new draw function for rendering

### Collision Detection
- Currently AABB (bounding box). Coordinates are in game space (0–480 width, 0–720 height)
- Player: `[playerX - 15, playerY, 30, 40]`
- Obstacles: stored as `{ x, y, w, h }`

### Testing
- No test framework; manual testing via browser
- Test on both desktop (keyboard) and mobile (touch controls, responsive scaling)
- Use browser DevTools console for any debug output

### Performance
- Single canvas layer; no DOM manipulation beyond initial setup
- Obstacle/particle count is bounded: obstacles ~15–30 visible, particles ~100 max
- Audio: continuous engine hum + occasional one-shot effects

## Commit Strategy

Development follows numbered features (KGO-N) per commit, e.g.:
- KGO-14: Mobile touch controls
- KGO-15: Visual polish — particles, screen shake & car color variety

Each commit typically adds one feature or polish pass. No separate branches; work directly on `main`.

## Browser Requirements

- Canvas 2D context
- Web Audio API
- CSS flexbox
- Touch Events API (for mobile)
- `requestAnimationFrame`

Tested on modern Chrome, Safari, Firefox. No polyfills needed.

## Running the Game Locally

`.claude/launch.json` is configured with a Python HTTP server preset. Run via:
```bash
preview_start "Python HTTP Server"
```
This serves the game at http://localhost:3000. Useful for testing on mobile devices or verifying final visuals before committing.
