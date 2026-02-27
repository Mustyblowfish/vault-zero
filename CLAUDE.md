# Vault Zero — Developer Context

Single-file HTML5 post-apocalyptic survival game. All code lives in `index.html`.
- Terminal/CRT green aesthetic, hex map, turn-based resource management
- Play it live: https://mustyblowfish.github.io/vault-zero/
- `face-demo.html` is a standalone pixel portrait playground (keep in sync with `drawFacePortrait` in main file)

## Workflow
1. Read `index.html` before making any edits (context is lost between sessions — always re-read)
2. Edit in place — no build step needed
3. Commit via git, user pushes via GitHub Desktop → https://mustyblowfish.github.io/vault-zero/

---

## File layout (`index.html`)

Sections in order:
1. `<head>` — Google Fonts (Share Tech Mono, VT323), meta
2. `<style>` — CSS variables, layout, all component styles
3. `<body>` HTML — splash → char-create → game (terminal panel + map panel) → overlays
4. `<script>` — constants → G state → utility fns → game logic → UI fns → event listeners

---

## G — global state (single source of truth)

```js
G = {
  phase,          // 0=boot 1=restore 2=scan 3=map/main game
  turn,           // current day number (increments on End Day)
  power, food, water, metals, wood,
  population,     // cached G.people.length — keep in sync manually after push/filter
  scanRange,
  systems: { power, water, comms, scanner, fabricator },  // booleans
  world,          // { "q,r": hexObj } — the full map
  people,         // Person[]
  scoutUnits,     // ScoutUnit[] — animated map tokens for people
  enemyUnits,     // EnemyUnit[]
  nextScoutId, nextEnemyId,
  scoutCooldown,  // bool — blocks new dispatches during animation
  mapUnlocked,
  discoveries,    // count of explored hexes (triggers phase events)
  transmissionIdx,
  resourceHistory,
  cooldowns,      // { [buttonId]: timeoutId }
}
```

> `G.scoutUnits` and `G.enemyUnits` are initialised **after** the G literal — search "SCOUT UNITS on the map".

---

## Key object shapes

### Person
```js
{
  id,           // "person_<timestamp>_<rand>"
  name, role,   // role: 'scout' | 'farmer' | 'soldier' | 'doctor'
  trait,        // flavour string
  arrived,      // turn joined
  status,       // 'vault' | 'mission'
  missionHex,   // "q,r" or null (null = at vault)
  isPlayer,
  actionsLeft,  // 0 or 1 — resets each turn
  movesLeft,    // 0 or 1 — resets each turn
  starvingDays,
  wounds,       // 0–3; 3 = death
  fatigue,      // 0–5; 5 = exhausted → auto-returns to base
  stats: { speed, recon, combat, morale }  // all 1–5
}
```

### Hex
```js
{
  q, r,
  terrain,      // key into HEX_TERRAIN: WASTE RUIN FOREST WATER GRASS MOUNTAIN BUNKER
  building,     // key into HEX_BUILDINGS or null
  revealed,     // fog-of-war lifted
  explored,     // a person has physically visited
  eventSeen,    // CYOA encounter already triggered — don't repeat
  resources: { food, metals, wood, water }
}
```

### ScoutUnit / EnemyUnit
```js
// Scout
{ personId, q, r, tq, tr, moving, progress, animX, animY }
// Enemy
{ id, type, q, r, hp, maxHp, moving, tq, tr, animX, animY }
```

---

## Turn flow (`nextTurn`) — ORDER MATTERS

1. Reset `actionsLeft` / `movesLeft` for all people
2. Consume food & water
3. Starvation threshold checks → possible deaths
4. Resource generation (buildings, farmers at vault)
5. HUD update + cycle log header
6. Low-resource warnings
7. Doctor passive heal (most-wounded at vault)
8. **Fatigue tick** — field +1, vault −2, outpost −1
9. **Auto-return** exhausted characters (`autoReturnExhausted`)
10. `maybeSpawnEnemy` + `setTimeout(moveEnemies, 400)`

---

## Key functions

| Function | What it does |
|---|---|
| `log(text, type, delay)` | Queued terminal log with typing effect |
| `logGap()` | Blank separator line in terminal |
| `nextTurn()` | Advance one game day |
| `drawMap()` | Redraw hex canvas — always call after state changes |
| `updateHUD()` | Refresh resource bars + end-day glow |
| `openHexInfo(hex)` | Show right-side hex-info panel |
| `dispatchScoutTo(hex, person)` | Move person to hex (animates, triggers arrival logic) |
| `autoReturnExhausted(person)` | Force-move toward nearest base |
| `resolveCombat(person, enemy)` | Exchange damage; sets actionsLeft=0 |
| `maybeSpawnEnemy()` | 35% chance/turn, max 4 enemies |
| `generatePerson(forceRole?)` | Create new survivor object |
| `drawFacePortrait(canvas, name, role)` | Seeded pixel portrait — call AFTER canvas is in DOM |
| `showEncounter(enc, hex, person)` | Launch CYOA encounter overlay |
| `hKey(q,r)` | Returns `"q,r"` string |
| `hDist(q1,r1,q2,r2)` | Hex distance (axial) |
| `revealAround(q, r, range)` | Lift fog in radius |
| `toast(text)` | Brief floating on-screen message |
| `setActions(arr)` | Set terminal action buttons |
| `setCooldown(id, ms)` | Button cooldown timer |

---

## Constants

| Constant | Purpose |
|---|---|
| `HEX_TERRAIN` | Terrain types with color/label/desc |
| `HEX_BUILDINGS` | All buildings. OUTPOST is field-only (skip in vault build loop) |
| `HEX_ACTIONS` | terrain → action key array |
| `ACTION_DEFS` | label, icon, yields, base amount per action |
| `ROLE_DEFS` | Role stats, icons, descriptions |
| `ENEMY_TYPES` | raider (turn 5+), mutant (turn 15+) |
| `ENCOUNTER_POOLS` | CYOA content keyed by terrain type |
| `ENCOUNTER_CHANCE` | Per-terrain probability of triggering |
| `MAX_FATIGUE` | 5 |
| `FACE_PALETTES` | Role-based colour schemes for pixel portraits |

---

## CSS conventions

- **Variables**: `--green` (#00ff41) · `--green-dim` (#005500) · `--green-dark` (#003300) · `--red`
- **Font**: `'Share Tech Mono', monospace`
- **Mobile breakpoint**: `@media (max-width: 700px)`
- **z-index layers**: map=1, hex-info=20, map-log=21, map-end-day=22, combat-popup=30, roster=50, encounter=200

## Log types
`'sys'` `'story'` `'action'` `'alert'` `'error'` `'header-line'`

Stagger delays within an event: 0 / 200 / 400 / 600 ms.

---

## Audio
- `initAudio()` — creates AudioContext on first user gesture
- `playMoveSound()` — on dispatch
- `playBootBeep(type)` — 'step' | 'alert' | 'complete'
- All gated behind `isMuted` + `audioCtx` null-check

---

## Current features
- Hex map, fog of war, axial coords, pan/drag
- Fatigue (MAX_FATIGUE=5); wounds (0–3); doctor heals passively + actively in field
- Enemy units: raiders + mutants; combat consumes the action for that turn
- Pixel face portraits — seeded per name
- Buildings: Animal Pen, Rain Collector, Solar Panel, Furnace, Grow Lights, Forge, Outpost
- CYOA encounter system — triggers on first hex visit, keyed by terrain type, suppresses hex-info auto-open
- Splash boot sequence, character creation, mobile support (iOS tested)

## Dev skip
`devSkipToMap()` — jumps to phase 3 with a scout. Initialises `G.enemyUnits`, `G.scoutUnits`. Good for testing mid-game features. Triggered by the red "TEST" button on the splash.

---

## Critical gotchas

**`innerHTML +=` destroys canvas** — serialises subtree to HTML string and back. Always `createElement` / `appendChild` when a canvas is nearby.

**`nextTurn()` order is load-bearing** — don't reorder without reading the full function.

**`animateScout` / `animateEnemy` are RAF loops** — `anyActionPending` in `drawMap()` blocks End Day while animations run. Any new animated unit type needs adding to that check.

**`addLine()` / typing queue** — calling `log()` during an active encounter can stack badly. Use `setTimeout` offsets.

**Seeded random** — `G.world` generation must stay deterministic. For per-character randomness use `faceRng(nameToFaceSeed(name))`.

**`G.population`** — always sync manually with `G.people.length` after push/filter.

**`dispatchScoutTo` arrival callback** — multiple systems hook here: encounter trigger, combat check, phase triggers, cooldown reset. Read it before touching.

---

## Planned next
- More CYOA encounter content (rare handcrafted "black hex" stories)
- Canvas mini-games for specific hex types (radio tower signal lock, etc.)
- Module split into source files (when file exceeds ~7k lines)
