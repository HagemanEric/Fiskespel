# Fiskespel — Project Overview

## Vision

**Fiskespel** is a Swedish fishing game with the long-term goal of delivering a realistic, data-driven experience across Sweden's lakes, rivers, and coastal waters. Players will fish from shore, boat, and kayak in authentic environments shaped by real geography, species populations, weather, seasons, and time of day.

The **real game** will be built in **Godot 4**. The repository currently contains an **HTML balance prototype** (`index.html`) — a test machine for tuning fishing feel, odds, and progression before those rules are reimplemented in Godot.

---

## Two Artifacts, Two Roles

| Artifact | Role | Technology |
|----------|------|------------|
| **`index.html`** | Balance and feel test bed; design specification for rules | Single-file HTML, Canvas 2D, minimal Three.js in shop only |
| **Godot game** *(future)* | Shippable product — 3D environments, full content | Godot 4, GDScript |

The prototype is **not** the production codebase. We will **not** refactor it into Vite/TypeScript or build a web-based 3D engine.

---

## Current State (Prototype)

| Attribute | Detail |
|-----------|--------|
| **Game file** | `index.html` (only playable file) |
| **Language** | Swedish UI |
| **Platform** | Mobile browser (max-width 480px layout) |
| **Rendering** | Canvas 2D for fishing; Three.js r128 for shop rod preview only |
| **Purpose** | Tune bite odds, fight feel, size model, XP curve |
| **Code structure** | Nine commented JS sections (KONFIGURATION → EVENTHANTERING) |
| **Backend** | None — fully client-side |
| **Persistence** | Planned / partial — local save recommended for playtesting |

---

## Product Pillars (Target Product — Godot)

These pillars guide the Godot build. The prototype validates simplified versions of pillars 4 and 5.

1. **Authentic Swedish waters** — Real locations with accurate shorelines, depths, and access types (shore, boat, kayak).
2. **Real fish species** — Species tied to ecological regions, with behavior influenced by habitat, temperature, season, and time of day.
3. **Real-world data integration** — Population density, regulations, records, and environmental data from authoritative Swedish sources.
4. **Dynamic environment** — Weather, seasons, water temperature, daylight, and ice cover affecting fish activity.
5. **Deep fishing simulation** — Methods (mete, spinn, vertikal, trolling), gear selection, and skill-based fight mechanics.
6. **Scalable content** — Data-driven architecture supporting hundreds or thousands of fishing locations.

---

## Core Gameplay Loop

The prototype implements the loop that Godot must preserve:

```
Setup (spot · method · time) → Cast → Wait/Retrieve/Jig → Bite → Hookset → Fight → Land → Reward
```

### Player-facing screens (prototype)

| Screen | Purpose |
|--------|---------|
| **Fiska** | Main fishing experience with canvas scene and touch controls |
| **Shop** | Purchase and equip rods; buy cosmetic skins with pearls |
| **Logg** | Species discovery, catch history, and three-tier records |

### Progression and economy

- **Levels & XP** — Curve tuned for ~25 minutes from level 1 to 5; unlocks species, spots, methods, and boat access.
- **Swedish kronor (kr)** — Earned by selling fish; spent on rods.
- **Pearls (◈)** — Earned by beating lake and real-world records; spent on cosmetic skins.

### Record system

| Tier | Description | Reward |
|------|-------------|--------|
| Personal best | Player's largest catch | XP bonus |
| Lake record | Beating the in-game lake record | Title + pearls |
| Real record | Matching the documented real-world record | Trophy + pearls |

---

## Species (Prototype)

Five freshwater species represent Munksjön's starter ecology:

| Species | Swedish name | Unlock level | Real record (kg) |
|---------|--------------|--------------|------------------|
| Mort | Mört | 1 | 1.0 |
| Abborre | Abborre | 1 | 2.6 |
| Braxen | Braxen | 3 | 7.0 |
| Gadda | Gädda | 5 | 15.0 |
| Gos | Göös | 12 | 8.0 |

Each species has weight tiers (liten / bra / stor), sell price, habitat preferences via odds multipliers, and species-specific haptic feedback during bites and fights.

---

## Fishing Methods (Prototype)

| Method | Mechanic | Notes |
|--------|----------|-------|
| Mete | Passive float fishing | One-button cast, timed wait |
| Spinn | Active retrieve | Charge-cast + vertical reel pad |
| Vertikal · jigg | Lift-and-drop jig | Lift button + reel pad; bites on drop |
| Troll (båt) | Boat trolling | Requires boat unlock at level 15 |

---

## Environment Setup (Prototype)

**Spots** within Munksjön: Vassvik, Öppet vatten, Stenrev, Djupkant.

**Times of day**: Dag, Förmiddag, Gryning, Skymning, Natt — each affects sky/water rendering and species activity multipliers.

In Godot, these abstractions will map to **habitat tags** and **environment state** on real geography.

---

## Development Phases (Summary)

See [DEVELOPMENT_ROADMAP.md](./DEVELOPMENT_ROADMAP.md) for full detail.

| Phase | Status | Focus |
|-------|--------|-------|
| 1 — Stabilize prototype | ✅ Done | Save, hygiene, docs |
| 2 — Structure code | ✅ Done | JS sections, English names, Swedish player text |
| 3 — Fishing feel & balance | **Active** | Bite, fight, odds, size, ~25 min to level 5 |
| 4 — Playtest with anglers | Planned | External calibration |
| 5 — Export rules | Planned | Formulas and data for Godot |
| 6 — Rebuild in Godot | Planned | Vertical slice from specs |

---

## Success Criteria

### Prototype (phases 1–4)

- [x] Playable full loop on mobile browser
- [x] Progression, shop, and catch log
- [x] Method-specific mechanics and fight system
- [x] Code organized in tunable sections
- [ ] Level 1→5 in ~25 minutes (validated by playtest)
- [ ] Balance values signed off by real anglers (phase 4)
- [ ] Rules exported in Godot-ready format (phase 5)

### Godot vertical (phase 6+)

- Core loop with same rules as frozen prototype specs
- 3D Munksjön reference environment
- Save/load and shippable input (touch + desktop)

### Long-term (post–phase 6)

- Multiple Swedish water bodies, real data, weather/seasons
- Boat, shore, and kayak fishing modes
- Scalable location content

---

## Key Risks

| Risk | Mitigation |
|------|------------|
| Over-investing in prototype code | Prototype stays single-file; no Vite/TS; retire when Godot vertical works |
| Rules lost in translation to Godot | Phase 5: explicit formula specs + JSON, not JS port |
| Balancing in two places | Freeze constants after phase 4; Godot reads spec, not live HTML |
| Real-world data licensing | Validate SLU, SMHI, Lantmäteriet terms before content scale-up in Godot |

---

## Related Documentation

| Document | Contents |
|----------|----------|
| [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) | Prototype architecture + Godot target |
| [DEVELOPMENT_ROADMAP.md](./DEVELOPMENT_ROADMAP.md) | Six-phase plan: prototype → Godot |

---

## Glossary

| Term | Meaning |
|------|---------|
| **Napp** | Bite — fish taking the lure or bait |
| **Mete** | Still-water bait fishing |
| **Spinn** | Spinning — active lure retrieval |
| **Pärlor (◈)** | Pearls — premium cosmetic currency |
| **Verk rekord** | Real-world documented record weight |
| **Sjörekord** | In-game lake record |

---

*Last updated: June 2025 — Godot as production target; HTML prototype as balance test bed.*
