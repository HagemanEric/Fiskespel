# Fiskespel — Technical Architecture

This document describes the **HTML prototype architecture** (`index.html`) and the **target Godot architecture** for the real game. The prototype is a **balance test machine** — not production code and not a path to a web-based 3D engine.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prototype Structure (`index.html`)](#2-prototype-structure-indexhtml)
3. [Data Model (Prototype)](#3-data-model-prototype)
4. [Simulation Logic (Prototype)](#4-simulation-logic-prototype)
5. [Rendering and Input (Prototype)](#5-rendering-and-input-prototype)
6. [Known Limitations (Prototype)](#6-known-limitations-prototype)
7. [Target Architecture (Godot)](#7-target-architecture-godot)
8. [Rule Export Format (Phase 5)](#8-rule-export-format-phase-5)
9. [Migration Mapping: Prototype → Godot](#9-migration-mapping-prototype--godot)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  TODAY: index.html — balance & feel test bed                     │
│  Single file · Canvas 2D gameplay · rules in commented sections  │
└───────────────────────────────┬─────────────────────────────────┘
                                │ Phase 5: export rules (not port JS)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  TARGET: Godot 4 — shippable game                                │
│  3D environments · GDScript · data-driven content (later)        │
└─────────────────────────────────────────────────────────────────┘
```

**Explicitly out of scope for the prototype path:**

- Vite, TypeScript, or module bundlers
- Web-based 3D game engine or open-world renderer
- Unity or Unreal

Three.js in the prototype is **shop-only** (rod preview). It is not part of the production rendering plan.

---

## 2. Prototype Structure (`index.html`)

### 2.1 File layout

| Layer | Location |
|-------|----------|
| HTML structure | Lines 1–185 (unchanged during refactor) |
| CSS | Inline `<style>` (unchanged) |
| JavaScript | Single `<script>` block with nine sections |

### 2.2 JavaScript sections

| Section | Responsibility |
|---------|----------------|
| **KONFIGURATION** | `BITE_CEILING`, `ODDS_SATURATION`, `PREDATOR_LEVEL_BONUS`, `GEAR_CAP`, XP curve, haptics |
| **ARTDATA** | `SPECIES`, `METHODS`, `TIMES`, `SPOTS`, `RODS`, `SKINS` |
| **SPELSTATE** | `gameState`, `sceneState`, canvas/shop runtime vars |
| **FÅNGSTMOTOR** | Odds, species/size pick, cast, fight, `updateFishingScene` |
| **EKONOMI** | Landing rewards, shop purchases |
| **PROGRESSION** | `grantXp`, level unlocks |
| **FISKELOGG** | Per-species log and records |
| **UI** | Canvas draw, HUD, shop (Three.js), modals |
| **EVENTHANTERING** | Input bindings, `initGame()` |

Code identifiers are **English**; comments are **Swedish**; player-facing strings are **Swedish**.

### 2.3 High-level diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      index.html                              │
├─────────────────────────────────────────────────────────────┤
│  KONFIGURATION │ ARTDATA │ SPELSTATE                        │
├────────────────┴─────────┴──────────────────────────────────┤
│  FÅNGSTMOTOR (odds · bite · fight · scene tick)              │
├─────────────────────────────────────────────────────────────┤
│  EKONOMI │ PROGRESSION │ FISKELOGG                           │
├─────────────────────────────────────────────────────────────┤
│  UI (Canvas 2D + shop Three.js) │ EVENTHANTERING            │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 Architectural characteristics

| Property | Value |
|----------|-------|
| Pattern | Monolithic single-file SPA |
| State | Global objects (`gameState`, `sceneState`) |
| Build | None — open file in browser |
| Purpose | Tune balance; document rules for Godot |
| Tests | Manual playtest (no CI unit tests planned) |

---

## 3. Data Model (Prototype)

### 3.1 Tunable constants (KONFIGURATION)

| Constant | JS name | Default | Role |
|----------|---------|---------|------|
| TAK | `BITE_CEILING` | 0.48 | Max bite probability per cast |
| K | `ODDS_SATURATION` | 85 | Odds saturation — higher = sparser bites |
| GOLV | `PREDATOR_LEVEL_BONUS` | 0.4 | Predator weight bonus per level |
| CAP | `GEAR_CAP` | 25 | Max rod gear bonus (%) |
| XP base | `XP_BASE` | 14 | Progression tempo (`14 × level^1.4`) |

### 3.2 Static data (ARTDATA)

| Object | Purpose |
|--------|---------|
| `SPECIES` | Fish: density, unlock, weights, prices, records, growth |
| `METHODS` | Fishing methods: mechanic type, species multipliers |
| `TIMES` | Time of day: visuals + species multipliers |
| `SPOTS` | Location within lake: species multipliers |
| `RODS` / `SKINS` | Equipment and cosmetics |
| `HAPTIC_PROFILES` | Per-species vibration patterns |

### 3.3 Player state (`gameState`)

Progression, currencies (`kronor`, `pearls`), current setup (spot, method, time), equipment, shop UI state, and per-species `log` entries (personal best, lake record, size classes, etc.).

### 3.4 Scene state (`sceneState`)

State machine driven by `sceneState.mode`:

```
ready → charging → cast → mete_wait | spin | jig
                              ↓
                            bite → fight → surface | result
                              ↓
                            ready
```

---

## 4. Simulation Logic (Prototype)

These functions are the **primary export targets** for Phase 5 documentation.

### 4.1 Odds engine

**Raw weight per species:**
```
computeRawWeight(id) = density × methodMult × timeMult × spotMult + predatorBonus
predatorBonus = isPredator ? level × PREDATOR_LEVEL_BONUS : 0
```

**Bite probability:**
```
biteChance = min(BITE_CEILING, BITE_CEILING × sum / (sum + ODDS_SATURATION))
             × (1 + 0.01 × level) × gearMultiplier
```

**Setup quality** (size only, not bite frequency):
```
quality(id) = currentSetupMult(id) / bestPossibleMult(id)
```

### 4.2 Size selection

`pickSpeciesAndSize()` uses `computeSizeReach(id)` (level vs species `growthLevels`) and `computeSetupQuality(id)` to weight stor / bra / liten. Trophy class is gated at low level growth.

### 4.3 Fight simulation

| Variable | Behavior |
|----------|----------|
| `fightState.tension` | Rises when reeling, especially during runs |
| `fightState.stamina` | Depletes while reeling; win at zero |
| `fightState.running` | Periodic bursts — player must give line |
| Snap | `tension >= 1` |
| Land | `stamina <= 0` → surface → rewards |

### 4.4 Progression

```javascript
xpRequiredForLevel(level) = round(XP_BASE × level^1.4)
```

`getLevelFactors()` scales cast range, hookset window, fight ease, and retrieve contact sensitivity.

---

## 5. Rendering and Input (Prototype)

### 5.1 Gameplay rendering

**Canvas 2D** — stylized lake silhouette, not a production visual target. Used only to support feel testing (lure position, fight motion, tension feedback).

### 5.2 Shop rendering

**Three.js r128** (CDN) — rod preview and shelf thumbnails. Cosmetic only; **not ported to Godot as web code** — Godot will use its own 3D assets.

### 5.3 Input

Touch-first: reel pad, charge-cast on canvas, strike/lift buttons. Vibration API with screen-shake fallback on iOS.

### 5.4 Reel input normalization

```javascript
consumeReelInput(dt) = clamp(reelAccum / dt / 650, 0, 1.3)
```

---

## 6. Known Limitations (Prototype)

| Area | Limitation | Acceptable because |
|------|------------|-------------------|
| Single HTML file | No automated tests | Prototype is for manual balance tuning |
| Canvas 2D visuals | Not representative of final game | Godot handles all 3D presentation |
| Three.js shop | CDN dependency, extra WebGL | Shop UX reference only |
| No fish AI agents | Probabilistic bites | Agent sim comes in Godot, guided by exported odds rules |
| One abstract lake | Four spots, five species | Enough to validate rules before content scale |
| Persistence | May be incomplete | localStorage sufficient for playtests |

**Do not fix by expanding the prototype:** modular TypeScript, web 3D world, or content pipelines — those belong in Godot.

---

## 7. Target Architecture (Godot)

### 7.1 Engine and language

| Choice | Decision |
|--------|----------|
| Engine | **Godot 4** |
| Script | **GDScript** (C# optional for shared libs if needed later) |
| Prototype | Remains `index.html` until Godot vertical replaces it for playtests |

### 7.2 Godot project layout (initial)

```
fiskespel-godot/
├── project.godot
├── data/                    # Imported from Phase 5 JSON
│   ├── species.json
│   ├── methods.json
│   ├── times.json
│   ├── spots.json
│   └── balance_constants.json
├── scripts/
│   ├── sim/                 # bite_engine.gd, fight_engine.gd, progression.gd
│   ├── player/
│   └── ui/
├── scenes/
│   ├── main.tscn
│   ├── fishing/
│   └── ui/
└── assets/                  # 3D models, audio, textures
```

### 7.3 Layered systems (Godot)

```
Presentation   — 3D scene, camera, UI, audio, VFX
Simulation     — Bite, fight, progression (from Phase 5 specs)
Content        — Species, locations, gear (data files)
Platform       — Save/load, analytics (later)
```

Long-term vision (many Swedish locations, SMHI/SLU data, weather) is implemented **in Godot**, not by growing the HTML prototype.

### 7.4 Spatial model (future Godot content)

```
Region
  └── WaterBody (lake | river | coastal)
        └── SubArea
              └── HabitatCell (tags: vass, rev, djup, open_water, …)
```

Prototype `SPOTS` map to habitat tags when content work begins in Godot.

---

## 8. Rule Export Format (Phase 5)

Phase 5 produces documentation and data **outside** `index.html`:

| Artifact | Contents |
|----------|----------|
| `docs/specs/bite_engine.md` | Formulas, inputs, outputs, examples |
| `docs/specs/fight_engine.md` | Tension, stamina, runs, species profiles |
| `docs/specs/progression.md` | XP curve, unlocks, level factors |
| `docs/specs/state_machine.md` | Mode transitions |
| `docs/balance/constants.json` | All tunable numbers |
| `docs/balance/*.json` | Species, methods, times, spots tables |

Godot reads JSON/resources; it does **not** embed or transpile the prototype script.

---

## 9. Migration Mapping: Prototype → Godot

| Prototype (concept) | Godot destination |
|---------------------|-------------------|
| `computeBiteOdds()`, `pickSpeciesAndSize()` | `scripts/sim/bite_engine.gd` |
| `tickFight()`, haptic profiles | `scripts/sim/fight_engine.gd` |
| `grantXp()`, `getLevelFactors()` | `scripts/sim/progression.gd` |
| `sceneState.mode` transitions | `scripts/sim/fishing_state.gd` |
| `SPECIES`, `METHODS`, etc. | `data/*.json` → Godot Resources |
| Canvas `drawFishingScene` | 3D scene + camera (new implementation) |
| Three.js shop rod | Godot 3D preview scene |
| Tab UI / Swedish copy | Godot Control nodes — reuse text, not DOM |

**Migration principle:** Reimplement behavior from **specs**, not copy-paste JavaScript.

---

## Appendix: External Dependencies (Prototype Only)

| Dependency | Source | Used for |
|------------|--------|----------|
| Three.js r128 | cdnjs | Shop rod preview only |
| Google Fonts | fonts.googleapis.com | UI typography |

These dependencies do not carry over to Godot.

---

*Last updated: June 2025 — HTML prototype for balance; Godot 4 for production.*
