# Fiskespel — Project Overview

## Vision

**Fiskespel** is a Swedish fishing game with the long-term goal of delivering a realistic, data-driven experience across Sweden's lakes, rivers, and coastal waters. Players will fish from shore, boat, and kayak in authentic environments shaped by real geography, species populations, weather, seasons, and time of day.

The current repository contains a **playable prototype** themed around **Munksjön** — a mobile-first demo that validates core gameplay, progression, and UX before a full production build.

---

## Current State (Prototype)

| Attribute | Detail |
|-----------|--------|
| **Repository** | Single-file web app (`index.html`) |
| **Language** | Swedish UI |
| **Platform target** | Mobile browser (max-width 480px layout) |
| **Rendering** | Canvas 2D for fishing; Three.js r128 for shop rod preview |
| **Persistence** | None — state resets on page refresh |
| **Backend** | None — fully client-side |
| **Version control** | Git with initial commit; remote at `github.com/HagemanEric/Fiskespel` |

The prototype is best understood as an **interactive game design specification**: it proves the fun of setup → cast → bite → fight → reward, not as production-ready game architecture.

---

## Product Pillars (Target Product)

These pillars guide all future development. The prototype implements simplified versions of pillars 1, 4, and 5.

1. **Authentic Swedish waters** — Real locations with accurate shorelines, depths, and access types (shore, boat, kayak).
2. **Real fish species** — Species tied to ecological regions, with behavior influenced by habitat, temperature, season, and time of day.
3. **Real-world data integration** — Population density, regulations, records, and environmental data from authoritative Swedish sources.
4. **Dynamic environment** — Weather, seasons, water temperature, daylight, and ice cover affecting fish activity.
5. **Deep fishing simulation** — Methods (mete, spinn, vertikal, trolling), gear selection, and skill-based fight mechanics.
6. **Scalable content** — Data-driven architecture supporting hundreds or thousands of fishing locations without hand-building each scene.

---

## Core Gameplay Loop

The prototype implements a complete loop that should be preserved in future versions:

```
Setup (spot · method · time) → Cast → Wait/Retrieve/Jig → Bite → Hookset → Fight → Land → Reward
```

### Player-facing screens

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

Three layers per species, designed to anchor fantasy to reality:

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

These abstractions will later map to **habitat tags** and **environment state vectors** on real geography.

---

## Stakeholders and Audience

| Audience | Interest |
|----------|----------|
| **Players** | Accessible mobile fishing with depth and Swedish authenticity |
| **Developers** | Clear migration path from prototype to scalable engine |
| **Content authors** | Data-driven lakes, species, and gear without code changes |

---

## Success Criteria by Stage

### Prototype (current)

- [x] Playable full loop on mobile browser
- [x] Progression, shop, and catch log
- [x] Method-specific mechanics and fight system
- [ ] Persistence across sessions
- [ ] Boat purchase flow in shop

### Alpha (future)

- One 3D reference lake (Munksjön)
- Save system and core sim logic in tested modules
- Shore fishing in 3D with ported odds/fight model

### Beta (future)

- Multiple Swedish water bodies via content pipeline
- Fish behavior beyond random bite rolls
- Weather and seasonal modifiers

### Release (future)

- Stable content delivery for many locations
- Regulations and seasonal rules
- Polished audio, performance, and platform targets

---

## Key Risks

| Risk | Mitigation |
|------|------------|
| Prototype code cannot scale to 3D open worlds | Treat prototype as GDD reference; plan phased rewrite |
| Real-world data licensing | Validate SLU, SMHI, Lantmäteriet terms early |
| Scope creep on simulation realism | Phase deliverables with explicit cut lines per milestone |
| Solo/small team maintenance | Modular architecture and automated content validation |

---

## Related Documentation

| Document | Contents |
|----------|----------|
| [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) | Current and target system design |
| [DEVELOPMENT_ROADMAP.md](./DEVELOPMENT_ROADMAP.md) | Phased plan from stabilization to release |

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

*Last updated: June 2025 — reflects prototype state at initial commit.*
