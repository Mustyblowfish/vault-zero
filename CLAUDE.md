# Vault Zero — Claude Code Project Notes

## The Game
Single-file HTML5 post-apocalyptic survival game. All code lives in `index.html`.
- Terminal/CRT green aesthetic, hex map, turn-based resource management
- Play it live: https://mustyblowfish.github.io/vault-zero/

## Working File
Always edit: `/Users/jay/Documents/GitHub/vault-zero/index.html`
After edits, user pushes via GitHub Desktop (Commit → Push).

## Architecture
- Single HTML file — embedded CSS + JS, no frameworks, no build step
- Web Audio API for procedural sounds (blocked until user gesture — init on click)
- Canvas hex grid: flat-top, pan/drag, pixel-to-hex math
- Save/load via localStorage (auto-save every 15s)

## Key Systems

### People & Movement
- Each person gets `actionsLeft: 1` and `movesLeft: 1` per turn — reset in `nextTurn()`
- Moving costs `movesLeft`, acting costs `actionsLeft` — independent, either order
- Scouts move 2 hexes, all others move 1 hex (`getMoveRange()`)
- `canPersonReachHex(person, hex)` — checks movesLeft + distance
- `getPersonPosition(person)` — returns current hex {q,r} (vault = 0,0 if no missionHex)
- Move mode: `pendingMovePerson` state, `enterMoveMode()` / `cancelMoveMode()`

### Person Data Shape
```js
{
  id, name, role, trait, arrived,
  status: 'vault' | 'mission',
  missionHex: null | 'q,r',
  isPlayer: bool,
  actionsLeft: 1,
  movesLeft: 1,
  starvingDays: 0,
  stats: { speed, recon, combat, morale }
}
```

### Resources
- **Food**: passive cost per person per turn. Hitting 0 → starvingDays++. Second turn at 0 → death.
- **Water**: passive cost per person per turn
- **Power (kWh)**: consumed by buildings/base operations passively. NOT spent to move/act.
- **Metals + Wood**: construction resources for buildings
- Moving/acting costs NO resources — people can always move+act freely

### Starvation
- `food <= 0` → each person's `starvingDays` increments
- `starvingDays >= 2` → person dies (removed from G.people + G.scoutUnits)
- All dead → `showGameOver('starvation')` → score screen

### Turn-end delta display
- Food and water charts show `+X` / `-X` next to their value
- Red = will hit zero this turn, Amber = depleting, dim = stable

### Building System
- Build panel opened via `[ ⚙ BUILD ]` button above resource charts
- Build placement mode: `pendingBuildType`, cursor → crosshair, ESC cancels
- `handleBuildPlacement(hex)` — validates hex, deducts resources, places building
- Buildings also placeable at vault hex (0,0) via hex-info panel

### Hex Map
- Flat-top hexagons, `hKey(q,r)` = string key, `hDist()` = distance
- Amber/gold outlines on all hexes reachable by any person with movesLeft > 0
- `isReachableForScout(hex)` — any person can reach it this turn
- Auto-opens hex-info panel on arrival if person has actionsLeft > 0

### Audio
- `initAudio()` — creates AudioContext on first user gesture
- `playBootBeep(type)` — 'step', 'alert', 'complete'
- `playMoveSound()` — plays on dispatch
- All audio gated behind `isMuted` flag + audioCtx null-check

### Game Over
- `showGameOver(cause)` — displays score screen overlay (#game-over)
- Score = turns×50 + peakPop×200 + explored×30 + buildings×100 + survivors×150
- `doRestart()` — clears localStorage save + reloads page

## Game State (G object — key fields)
```js
G.phase        // 0=boot 1=restore 2=scan 3=contact 4=expand
G.turn, G.daysPassed
G.food, G.water, G.power, G.metals, G.wood
G.population, G.maxPopulation
G.people[]     // array of person objects
G.scoutUnits[] // animation units on map
G.world        // hex map { 'q,r': hexObj }
G.mapUnlocked  // bool — whether map canvas is visible
G.scoutCooldown // bool — brief lock after dispatch
```

## Phases
- **0** Boot/splash sequence (click to start)
- **1** Restore systems (water, comms)
- **2** Scanner online — can explore hex map
- **3** Contact — survivors arrive, resource management begins
- **4** Expansion

## Dev Skip
`devSkipToMap()` — jumps straight to phase 3 with a scout for testing

## Workflow for Claude
1. Read `index.html` before making any edits
2. Edit in place — no build step needed
3. Tell user to open GitHub Desktop → Commit → Push when done
4. User can then play at https://mustyblowfish.github.io/vault-zero/
