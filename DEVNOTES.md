# Vault Zero — Developer Notes

> Living reference for Claude sessions. Update this file at the end of each session when new systems are added.
> Last updated: session covering vb-intro mobile fix, end-day map refresh, iOS perf fixes, FEATURES flags, dev panel.

---

## File Structure

Single file: `index.html` (~7600 lines). No build step. Deploy by pushing to GitHub Pages.

```
index.html
├── <head>          Fonts, all CSS
├── <body>
│   ├── #header         Top bar (always visible): VAULT ZERO title, phase tag, TEST btn, MENU btn
│   ├── #splash         Terminal narrative boot sequence (typewriter)
│   ├── #char-create    Survivor registration form (name, pronoun, role)
│   ├── #vault-boot     Emergency boot mini-game (4 modules: Power/Water/Scavenge/Comms)
│   ├── #scanner-boot   Full-screen lever pull → hex map transition
│   ├── #menu-overlay   Pause menu (save/load/mute/quit)
│   └── #main           Game proper
│       ├── #terminal-panel   Left: action log + buttons
│       └── #map-panel        Right: hex canvas (#gc) + HUD
└── <script>        All JS
```

CSS custom properties (`:root`):
```css
--green: #00ff41      --green-dim: #00aa2a    --green-dark: #003310
--green-faint: #001a08  --amber: #ffb000       --red: #ff3030
--bg: #020602         --scanline: rgba(0,255,65,0.03)
```

---

## Game State Object `G`

Declared at ~line 3058. All persistent state lives here (saved/loaded as JSON).

```javascript
const G = {
  // ── Meta ──────────────────────────────────────────────────────
  phase: 0,         // 0=boot | 1=vault-boot | 2=surface scan | 3=contact | 4=expand
  turn: 0,          // day counter (increments each END DAY)
  daysPassed: 0,    // turn * 7

  // ── Resources ─────────────────────────────────────────────────
  power: 0,         // 0–100 %
  food: 0,
  water: 0,
  metals: 0,
  wood: 0,

  // ── Population ────────────────────────────────────────────────
  population: 0,
  maxPopulation: 0,
  scouts: 0,
  people: [],       // array of Person objects (see Person schema below)

  // ── Vault systems (unlocked during vault-boot mini-game) ──────
  systems: {
    power: false,
    water: false,
    comms: false,
    scanner: false,
    fabricator: false,
  },

  // ── Map ───────────────────────────────────────────────────────
  mapUnlocked: false,
  scanRange: 0,     // how many hex rings revealed around vault
  world: {},        // key: "q,r" → Hex object (see Hex schema below)
  units: {},        // scout/enemy units on map

  // ── Narrative / progression ───────────────────────────────────
  transmissionIdx: 0,
  discoveries: 0,   // hexes explored (triggers phase transitions)
  survivors: 0,
  cooldowns: {},    // button cooldown timers

  // ── History (last 10 turns, for resource charts) ──────────────
  resourceHistory: { power:[], food:[], water:[], population:[] },
};
```

### Person schema (`G.people[i]`)
```javascript
{
  id, name, role,       // role: 'scout'|'soldier'|'medic'|'engineer'|'farmer'
  isPlayer: bool,
  arrived: turn,        // turn they joined
  status: 'vault'|'scouting'|'injured'|'dead',
  actionsLeft: 1,       // reset each turn
  movesLeft: 1,         // reset each turn
  // pronoun: set via G.people pronouns — stored separately
}
```

### Hex schema (`G.world["q,r"]`)
```javascript
{
  q, r,
  terrain: 'plains'|'forest'|'ruins'|'mountain'|'water'|'vault',
  revealed: bool,
  explored: bool,
  building: null | 'SOLAR'|'FURNACE'|'GROWLIGHT'|'COLLECTOR'|'ANIMAL_PEN'|...,
  loot: {},
}
```

---

## Phase Flow

```
phase 0  → startGame() fires, terminal narrative plays
           player watches typewriter sequence

phase 1  → restorePower() → restoreWater()/restoreComms() → checkSystemsProgress()
           → showVaultBoot()
           4-module emergency boot mini-game (Power/Water/Scavenge/Comms)
           → completeVaultBoot() → showScannerBoot()
           Lever pull → sbootDone() → enterSurfaceMap()

phase 2  → surface map active, hexes revealed, scouting begins
           Trigger: G.discoveries >= 2 && G.turn >= 5 → triggerContact()

phase 3  → contact phase (survivors arrive, more content unlocked)
           Trigger: G.population >= 5 && G.discoveries >= 4 → triggerExpansion()

phase 4  → expansion phase (full game loop)
```

**Phase tag** (`#phase-tag`) text matches phase:
`INITIALISING` → `RESTORING` → `SCANNING` → `CONTACT` → `EXPANDING`

---

## Screen Flow & Key Functions

| Screen | Show function | Hide/transition |
|--------|--------------|----------------|
| Splash (terminal) | `startGame()` L3837 | auto → `showVaultBoot()` after sequence |
| Char create | shown by `startGame()` before terminal | `confirmCharacter()` → hides it |
| Vault-boot | `showVaultBoot()` L6626 | `completeVaultBoot()` → `showScannerBoot()` |
| Scanner boot | `showScannerBoot()` L7242 | `sbootDone()` → `enterSurfaceMap()` |
| Hex map | `enterSurfaceMap()` L7112 | stays for rest of game |
| Menu overlay | `toggleMenu()` | `toggleMenu()` |

---

## Key Systems & Where to Find Them

### Audio (Web Audio API)
- `initAudio()` ~L2629 — creates AudioContext, calls startAmbience(). Called once on first user interaction.
- `startAmbience()` ~L2648 — 4-layer ambient: 60Hz hum, bandpass fan noise, random clicks, sub-bass pulses
- `masterGainNode` — global, controls master volume (0.06). Used by `toggleMute()`
- `tryResumeAudio()` ~L6341 — handles suspended/closed AudioContext on iOS return from background
- SFX functions: `sndBreaker()`, `sndPowerEngage()`, `sndWaterChoice()`, `sndScavRoom()`, `sndCommsBtn()`, `sndTermBtn(freq)`, `sndVaultAccess()`, `sndBeginSeq()`, `sndMapBootScan()`, `sndLeverStart/Update/Stop()`

### Hex Map
- Canvas: `#gc`, context `ctx2`, dimensions `W2 × H2`
- `HEX_SIZE` — hex radius in px, initial 20 (mobile) / 30 (desktop), range 12–60
- `camX, camY` — camera offset; `tCamX, tCamY` — animated target
- `drawMap()` — main render; guarded by `!G.mapUnlocked || mapBooting`
- `h2p(q, r)` — hex coords → world px; `w2s(x, y)` → screen px; `s2w(sx, sy)` → world px
- `generateWorld()` ~L4613 — procedural terrain, R=12 ring radius
- `revealAround(q, r, range)` — reveals hex ring

### Vault-boot mini-game
All functions prefixed `vb*`. State in `vbState` object.
Module order: Power (breakers) → Water (valve choices) → Scavenge (room search) → Comms (requirements)
Completion: `completeVaultBoot()` transitions to scanner boot.

### Scanner-boot lever
All functions prefixed `sboot*`. State: `sbootActive`, `sbootLvProg` (0–1).
`sbootLvApply(p)` — drives all visual updates including lever drone sound.
Completion: `sbootLvEngage()` → `sbootDone()` → `enterSurfaceMap()`.

### Map boot reveal
`startMapBootReveal()` — canvas getImageData snapshot, scans top→bottom over 2.2s.
`mapBooting` flag suppresses `drawMap()` during reveal.

### Buttons / Actions
`setActions(defs[])` — populates the action button bar. Each def: `{ id, label, style, action, glow, disabled }`.
`buttons{}` — live map of id → button element.
`setCooldown(id, ms, onDone?)` — disables button, runs progress bar, calls onDone when done.

### Saves
`saveGame()` / `loadGame()` — localStorage key `vaultZeroSave`. Saves `G` as JSON.

### CYOA Encounters
`triggerEncounter(hex)` — opens encounter overlay for a hex.
Encounter definitions in `ENCOUNTERS[]` array.

---

## Dev Tools (in-game)

**TEST button** (top-right header) — toggles the dev panel.

Dev panel skip functions:
- `devSkipToSplash()` — restarts terminal narrative
- `devSkipToCharCreate()` — shows character creation form
- `devSkipToVaultBoot()` — vault-boot mini-game with clean state
- `devSkipToScannerBoot()` — lever screen with all systems online
- `devSkipToMap()` — hex map with full game state (phase 3)

---

## CSS Naming Conventions

| Prefix | Area |
|--------|------|
| `.vb-*` / `#vb-*` | Vault-boot mini-game |
| `.vbm-*` | Vault-boot module headers |
| `.vbi-*` | Vault-boot intro overlay |
| `.sboot-*` / `#sboot-*` | Scanner-boot lever screen |
| `.cc-*` / `#cc-*` | (unused so far — reserved for char-create) |
| `.char-*` / `#char-*` | Character creation |
| `.btn` | Action buttons (base class) |
| `.btn-glow` | Pulsing green glow (next recommended action) |
| `.btn-glow-next` | Brighter green glow (unlocked after sequence) |
| `.btn-glow.amber` | Amber glow variant |

---

## Feature Flags (`FEATURES` object, top of JS ~L3111)

Toggle entire systems on/off without deleting code. Set `false` to disable WIP features.

```javascript
const FEATURES = {
  enemies:    true,   // enemy spawning + movement
  encounters: true,   // CYOA encounter overlays on hex explore
  building:   true,   // build panel / structure placement
  saving:     true,   // localStorage save/load
};
```

When adding a new half-built system: add a flag here, wrap the feature entry points in `if (FEATURES.myFeature)`, set to `false` until ready.

---

## Known Issues

> Update this list as bugs are found/fixed. Format: `- [ ] description` (open) or `- [x] description (fixed vXXXXXXX)`.

- [ ] iOS audio: still investigating full reliability of resume after long background periods
- [ ] Hex map: initial camera position after boot reveal may not centre perfectly on all devices
- [x] Vault-boot mobile: had to scroll to reach BEGIN SEQUENCE (fixed a537a34)
- [x] Char-create mobile: UNSEAL AIRLOCK hidden below fold (fixed a537a34)
- [x] Hex map pinch zoom: iOS intercepting gesture (fixed a537a34)
- [x] End-day map not refreshing: character highlights didn't appear until next tap (fixed this session)

---

## Planned Features

> Update this list as features are discussed. Format: `- [ ] description` (not started) or `- [~] description` (in progress).

- [ ] More encounter types (CYOA) — wilderness, ruins, survivor camps
- [ ] Enemy behaviour: different enemy types with varied movement patterns
- [ ] Building upgrades — upgrade SOLAR/GROWLIGHT etc. for better yields
- [ ] Scout mission system — assign scouts to multi-turn missions with outcomes
- [ ] Radio transmissions / story events that evolve by phase
- [ ] Fog of war reveal animations
- [ ] Win condition / end-game sequence

---

## Things to Watch / Known Limitations

- **Single file**: No hot-reload. Refresh browser after each save. Use git commits per feature.
- **iOS audio**: `tryResumeAudio()` handles suspend/close on background. If audio still breaks, check `audioCtx.state` in dev panel. Audio schedulers now check `audioCtx.state !== 'running'` before creating nodes.
- **drawMap throttle**: capped at ~60fps via timestamp check. If map looks stale, the throttle window is 14ms — adjust `_drawMapLast` check.
- **Mobile layout breakpoint**: `≤700px` triggers compact vault-boot (2×2 grid) and compact vb-intro. Char-create compacts at `≤600px` OR `≤720px` height.
- **Map canvas sizing**: `resizeMap()` must be called after any layout change that affects `#canvas-wrap` dimensions.
- **`mapBooting` flag**: Must be `false` before `drawMap()` can run. If map goes blank, check this flag in dev panel.
- **G.phase gating**: Many features check `G.phase`. When adding new content, decide which phase gates it.
- **Audio node GC on iOS**: `schedulePulse` now disconnects nodes via `onended` callback. All new short-lived SFX nodes should do the same to prevent WebAudio memory growth.

---

## Commit History (recent)

| Hash | What |
|------|------|
| `a537a34` | Fix 3 iPhone issues: char-create layout, audio resume, map pinch |
| `79398a8` | Fix vault-boot mobile layout and iOS audio resume |
| `4d28fa2` | Add SFX system, map boot reveal, TEST button shortcut |
| `0a651ae` | Add scanner-boot lever screen |
| `d6f5ceb` | Terminal boot: sequential locks, button glows, header visible |
| `f5aacae` | Improve vault-boot module text legibility |
