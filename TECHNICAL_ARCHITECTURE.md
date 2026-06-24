# Fiskespel — Technical Architecture

This document describes the **current prototype architecture** and the **target production architecture** for a scalable 3D Swedish fishing game. It is intended for developers planning the migration from the single-file demo to a data-driven simulation platform.

---

## Table of Contents

1. [Current Architecture](#1-current-architecture)
2. [System Components (Prototype)](#2-system-components-prototype)
3. [Data Model (Prototype)](#3-data-model-prototype)
4. [Rendering Pipeline (Prototype)](#4-rendering-pipeline-prototype)
5. [Input and Platform (Prototype)](#5-input-and-platform-prototype)
6. [Simulation Logic (Prototype)](#6-simulation-logic-prototype)
7. [Known Limitations](#7-known-limitations)
8. [Target Architecture](#8-target-architecture)
9. [Target Data Schema](#9-target-data-schema)
10. [Technology Recommendations](#10-technology-recommendations)
11. [Migration Mapping](#11-migration-mapping)

---

## 1. Current Architecture

### 1.1 High-level diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      index.html (monolith)                   │
├─────────────────────────────────────────────────────────────┤
│  HTML structure  │  Inline CSS  │  Inline JavaScript        │
├──────────────────┴──────────────┴───────────────────────────┤
│                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│   │  Tab Router  │   │  Game State  │   │ Static Config│    │
│   │  (DOM tabs)  │   │  G, SC, lure │   │ SP,METHODS…  │    │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘    │
│          │                  │                  │            │
│          ▼                  ▼                  ▼            │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│   │  UI Layer    │   │ State Machine│   │ Odds Engine  │    │
│   │  panels/modal│   │ SC.mode      │   │ bite/size    │    │
│   └──────────────┘   └──────┬───────┘   └──────────────┘    │
│                             │                                │
│              ┌──────────────┼──────────────┐                │
│              ▼              ▼              ▼                │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│   │ Canvas 2D    │  │ Fight System │  │ Three.js     │     │
│   │ drawScene()  │  │ fightTick()  │  │ Shop preview │     │
│   └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
   Browser APIs                         CDN: Three.js r128
   Canvas, WebGL, Vibration            Google Fonts
```

### 1.2 Architectural characteristics

| Property | Value |
|----------|-------|
| Pattern | Monolithic SPA, no module bundler |
| State | Global mutable objects (`G`, `SC`, runtime vars) |
| Persistence | None |
| Testing | None |
| Build | None — static file |
| Deployment | Any static host |

---

## 2. System Components (Prototype)

### 2.1 Application shell

- **Container**: `.app` — max-width 480px, full viewport height
- **Navigation**: Three tabs (`fiska`, `shop`, `logg`) via class toggling
- **Persistent chrome**: Level badge, XP bar, kr, pearls

### 2.2 Fishing screen components

| Component | DOM ID | Role |
|-----------|--------|------|
| Game canvas | `#cv` | 2D scene rendering |
| Fight HUD | `#hud` | Tension and stamina bars |
| Reel pad | `#reelpad` | Vertical drag → reel input |
| Action buttons | `#liftBtn`, `#strikeBtn` | Jig lift, hookset |
| Dock | `#dock` | Odds bar, cast/setup buttons |
| Setup panel | `#panel` | Spot, method, time chips |
| Hint | `#fhint` | Contextual instruction text |
| Modal | `#modal` | Catch result / line snap |
| Toast | `#toast` | Transient feedback |

### 2.3 Shop screen components

| Component | Role |
|-----------|------|
| `#shop3d` | Three.js WebGL rod viewer |
| `#shelf` | Horizontal equipment list with generated thumbnails |
| `#sinfo` | Selected item stats and buy/equip CTA |
| `#vtog` | Studio vs water view toggle |

### 2.4 Game loop

```javascript
requestAnimationFrame(frame)
  → updateScene(dt, now)   // logic, state transitions
  → drawScene(now)           // canvas render (if fishing tab active)
```

A separate always-on loop renders the shop Three.js scene when initialized.

---

## 3. Data Model (Prototype)

### 3.1 Static configuration (immutable at runtime)

| Object | Keys | Purpose |
|--------|------|---------|
| `SP` | species id | Names, colors, density, unlock, weights, prices, grow span |
| `ORDER` | array | Species display/hierarchy order |
| `METHODS` | method id | Name, unlock, boat flag, mechanic type, species multipliers |
| `TIMES` | time id | Name, sky/water/shore colors, glitter, species multipliers |
| `SPOTS` | spot id | Name, unlock, species multipliers |
| `RODS` | array | Gear stats, 3D mesh parameters |
| `SKINS` | array | Cosmetic material overrides |
| `HAP` | species id | Haptic patterns for bite and fight |

### 3.2 Tunable constants

| Constant | Default | Role |
|----------|---------|------|
| `TAK` | 0.48 | Bite probability ceiling |
| `K` | 85 | Odds saturation factor |
| `GOLV` | 0.4 | Predator level bonus per level |
| `CAP` | 25 | Maximum rod gear bonus (%) |

### 3.3 Player state (`G`)

```javascript
G = {
  level, xp, kr, pe,           // progression & currency
  spot, method, time,          // current setup
  rod, skin, boat,             // equipment
  ownedRods: Set,
  ownedSkins: Set,
  cat, selRod, selSkin, view,  // shop UI state
  log: { [speciesId]: LogEntry }
}
```

**LogEntry** per species: `count`, `pb`, `cls`, `spelRek`, `beatSpel`, `beatVerk`, `seen`.

### 3.4 Scene state (`SC`)

```javascript
SC = {
  mode,      // state machine phase
  t,         // elapsed time
  _nibbleAt, // fake nibble scheduler
  _bite,     // pre-rolled bite probability for current cast
  _wait      // mete wait timer
}
```

### 3.5 State machine (`SC.mode`)

```
ready → charging → cast → mete_wait | spin | jig
                              ↓
                            bite → fight → surface | result
                              ↓
                            ready
```

---

## 4. Rendering Pipeline (Prototype)

### 4.1 Canvas 2D (gameplay)

**Initialization**: `ResizeObserver` on `#cv` → `resizeScene()` → `buildStatic()`.

**Static scenery** (rebuilt on resize):
- Shoreline polygon (`SHORE`)
- Cloud positions (`CLOUDS`)
- Reed stalks (`REEDS`)

**Per-frame** (`drawScene`):
1. Sky gradient (from `TIMES[G.time]`)
2. Sun/moon glow
3. Clouds (non-night)
4. Shore fill
5. Water gradient + glitter
6. Wave lines
7. Ripples
8. Fish, lure, fishing line
9. Reeds and foreground bank
10. Rod (bezier with bend)
11. Cast preview and power meter (charging)
12. Splash particles

**Fish drawing**: Procedural 2D ellipses scaled by weight, animated by fight state.

### 4.2 Three.js (shop only)

- **Primary renderer**: Full rod model with grip, reel, guides, tip
- **Thumbnail renderer**: Offscreen 120×76 PNG via `preserveDrawingBuffer`
- **Interaction**: Pointer drag rotates `pivot` group
- **Auto-rotate**: Studio view idle rotation after 90 frames

**Version**: Three.js r128 from cdnjs (2021-era API).

---

## 5. Input and Platform (Prototype)

### 5.1 Input mapping

| Input | Handler | Active when |
|-------|---------|-------------|
| Tab clicks | `.tab onclick` | Always |
| Canvas pointer down/move/up | Charge-cast | `ready`, active methods |
| Cast button | `meteCast()` | `ready`, mete |
| Reel pad drag | `reelAccum` | spin, jig, fight |
| Strike button | `strike()` | `bite` |
| Lift button | `jigLift()` | `jig`, rest phase |
| Shop canvas drag | Pivot rotation | Shop tab |

### 5.2 Mobile considerations

- `touch-action: none` on interactive surfaces
- `{ passive: false }` on touch handlers to prevent scroll
- `100dvh` viewport height
- Vibration API with iOS screen-shake fallback
- `prefers-reduced-motion` disables strike pulse animation

### 5.3 Reel input normalization

```javascript
reelInput(dt) = clamp(reelAccum / dt / 650, 0, 1.3)
```

Vertical drag distance accumulated per frame; consumed in fight, spin, and jig modes.

---

## 6. Simulation Logic (Prototype)

### 6.1 Odds engine

**Raw weight per species:**
```
raw(k) = tat(k) × method.m(k) × time.t(k) × spot.s(k) + predatorBonus(k)
predatorBonus = rov ? level × GOLV : 0
```

**Bite probability:**
```
sum = Σ raw(k) for available species
bite = min(TAK, TAK × sum / (sum + K)) × (1 + 0.01 × level) × gearG()
```

**Quality** (for size, not frequency):
```
quality(k) = currentSetupMultiplier(k) / bestPossibleMultiplier(k)
```

### 6.2 Size selection

Uses level growth `sizeReach(k)` and setup `quality(k)` to weight stor / bra / liten rolls. Trophy class (`stor`) gated until sufficient level growth — prevents record fish at low level.

### 6.3 Fight simulation

| Variable | Behavior |
|----------|----------|
| `F.tension` | Rises when reeling, especially during fish runs |
| `F.stamina` | Depletes while reeling; win at zero |
| `F.running` | Periodic bursts; player must give line |
| `rodBend` | Visual feedback tied to tension |

**Loss**: `tension >= 1` → snap.  
**Win**: `stamina <= 0` → surface animation → reward.

### 6.4 Progression

```javascript
xpNeed(level) = round(14 × level^1.4)
```

Level factor `lf()` scales cast range, hookset window, fight ease, and contact sensitivity.

---

## 7. Known Limitations

| Area | Issue |
|------|-------|
| Architecture | Single 566-line file; no separation of concerns |
| Persistence | All progress lost on refresh |
| Scalability | One implicit lake; four abstract spots |
| 3D | Gameplay entirely 2D |
| Fish AI | Probabilistic bites; no agents or habitat |
| Environment | Time-of-day color swap only; no weather/season/temp |
| Boat | Flag exists; purchase flow incomplete |
| Security | `innerHTML` for dynamic UI |
| Dependencies | CDN Three.js without SRI; outdated version |
| Testing | No unit or integration tests |
| Audio | None |
| Thumbnails | Regenerated PNG data URLs on each shelf render |

---

## 8. Target Architecture

### 8.1 Layered production stack

```
┌─────────────────────────────────────────────────────────────┐
│                        Game Client                           │
│  (Unity / Unreal / Godot — TBD in Phase 3)                  │
├─────────────────────────────────────────────────────────────┤
│  Presentation  │  Scene graph, UI, audio, VFX, input        │
├────────────────┼────────────────────────────────────────────┤
│  Simulation    │  Fish AI, bite model, fight, environment    │
├────────────────┼────────────────────────────────────────────┤
│  Content       │  Location loader, species registry, gear    │
├────────────────┼────────────────────────────────────────────┤
│  Platform      │  Save/load, analytics, remote config        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Content & Services                       │
├──────────────────┬──────────────────┬───────────────────────┤
│  CDN / Storage   │  Backend API     │  Authoring Tools      │
│  location packs  │  saves, leader-  │  lake import, species │
│  meshes, audio   │  boards, config  │  validation pipeline  │
└──────────────────┴──────────────────┴───────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     External Data Sources                    │
│  SLU · SMHI · Lantmäteriet · regulation databases           │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 Recommended repository layout (target)

```
fiskespel/
├── docs/                    # Project documentation
├── packages/
│   ├── core/                # Shared simulation logic (portable, tested)
│   ├── client/              # Game engine project
│   └── tools/               # Content pipeline CLI
├── data/
│   ├── species/
│   ├── locations/
│   ├── gear/
│   ├── methods/
│   └── regulations/
├── assets/                  # Art, audio, engine-specific resources
└── tests/
```

### 8.3 Core services (target)

| Service | Responsibility |
|---------|----------------|
| `BiteSimulator` | Port of `odds()`, `pickAndSize()` — unit tested |
| `FightController` | Species-specific fight profiles |
| `EnvironmentSim` | Time, season, weather, water temperature |
| `FishPopulation` | Density and spawn rules per habitat cell |
| `FishAgent` | Individual fish state and behavior |
| `ProgressionService` | XP, unlocks, records |
| `ContentLoader` | Stream location packs from CDN |
| `SaveService` | Local + cloud persistence |

### 8.4 Spatial model (target)

```
Region
  └── WaterBody (lake | river | coastal)
        └── SubArea (bay, stretch, depth zone)
              └── HabitatCell (grid or navmesh polygon)
                    └── tags: vass, rev, djup, open_water, ...
```

Prototype spots map directly to **habitat tags** on spatial cells.

---

## 9. Target Data Schema

### 9.1 Species definition (JSON)

```json
{
  "id": "perca_fluviatilis",
  "nameSv": "Abborre",
  "nameLatin": "Perca fluviatilis",
  "predator": true,
  "baseDensity": 0.30,
  "unlockLevel": 1,
  "weightClasses": {
    "liten": [0.05, 0.2],
    "bra": [0.2, 0.7],
    "stor": [0.7, 2.5]
  },
  "realRecordKg": 2.6,
  "sellPricePerKg": 18,
  "growthLevels": 28,
  "behaviorProfile": "perch_aggressive",
  "methodAffinity": { "mete": 1.5, "spinn": 1.8, "vertikal": 1.6, "troll": 0.6 }
}
```

### 9.2 Location definition (JSON)

```json
{
  "id": "munksjon",
  "nameSv": "Munksjön",
  "type": "lake",
  "region": "SE-F",
  "centroid": [14.156, 57.782],
  "boundsGeoJson": "…",
  "bathymetryRef": "munksjon_depth.tif",
  "habitats": ["vass", "open_water", "rock_reef", "deep_edge"],
  "speciesPopulation": [
    { "speciesId": "perca_fluviatilis", "density": 0.30, "source": "SLU" }
  ],
  "accessModes": ["shore", "boat"],
  "regulationsRef": "se-freshwater-default"
}
```

### 9.3 Environment state (runtime)

```json
{
  "timeOfDay": "skymning",
  "season": "host",
  "airTempC": 12,
  "waterTempC": 14,
  "windMs": 3.5,
  "cloudCover": 0.6,
  "precipitation": "none",
  "moonPhase": 0.25,
  "iceCover": 0
}
```

Each field feeds multiplier tables — extending the prototype's `TIMES` pattern.

---

## 10. Technology Recommendations

| Concern | Recommendation | Notes |
|---------|----------------|-------|
| Game engine | **Unity (URP)** or **Unreal Engine 5** | Water plugins, GIS tooling, multi-platform |
| Lightweight alternative | **Godot 4** | Lower overhead; fewer water/GIS middleware options |
| Web-only path | **Three.js + TypeScript + Vite** | Viable for simplified 3D; harder at scale |
| Simulation core | **TypeScript or C#** shared library | Extract and test before engine binding |
| Backend | **Supabase** or **Firebase** | Auth, cloud saves, remote config |
| Spatial data | **PostGIS** + GeoJSON export | Lake boundaries, query by region |
| Content delivery | **CDN** (Cloudflare R2, S3) | Location packs, lazy loading |
| CI/CD | **GitHub Actions** | Build, test, deploy content packs |

**Decision gate**: Engine choice is finalized in Phase 3 after a one-week technical spike on water rendering and shore casting.

---

## 11. Migration Mapping

| Prototype artifact | Target destination |
|--------------------|-------------------|
| `SP`, `ORDER` | `data/species/*.json` |
| `METHODS` | `data/methods/*.json` + animation controllers |
| `TIMES` | Environment sim diurnal/season tables |
| `SPOTS` | Habitat tags on spatial grid |
| `RODS`, `SKINS` | `data/gear/` + 3D asset references |
| `odds()`, `raw()`, `quality()` | `BiteSimulator` (unit tested) |
| `pickAndSize()` | `CatchResolver` |
| `fightTick()` | `FightController` + species profiles |
| `gainXP()`, `lf()` | `ProgressionService` |
| `drawScene()` | Reference for mood/color; replaced by 3D |
| Three.js `buildRod()` | GLTF modular rod system |
| Shop UI flow | Retained UX pattern on new renderer |
| `HAP` haptics | Platform haptics + audio layer |
| Tab navigation | Engine UI framework or retained web shell |

---

## Appendix: External Dependencies (Prototype)

| Dependency | Source | Version |
|------------|--------|---------|
| Three.js | cdnjs.cloudflare.com | r128 |
| Fraunces | Google Fonts | variable |
| Inter | Google Fonts | 400–700 |
| IBM Plex Mono | Google Fonts | 500–600 |

---

*Last updated: June 2025 — reflects prototype at initial commit and planned target architecture.*
