# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

First-person co-op game about hauling cargo across realistic, Expanse-style space. Travel is the gameplay; everything between A and B is an obstacle (attacks, breaches, repairs, mining stops).

**Pillars**
- **Realistic physics**: acceleration-based movement (flip and burn), gravity, slingshots, recoil, relative velocities, G limits on crew.
- **Fragile ships**: no shields; hulls breach anywhere they're hit; decompression can pull crew out.
- **First-person immersion**: outside is seen only through windows, scopes, and radar, but everything out there is real.
- **Resource pressure**: ammo, oxygen, reaction mass, coolant, heat.
- **Seamless**: no loading screens between ship, space, and asteroids.

**Core loop**: pick up cargo → travel a leg → survive attacks/breaches/repairs/mining → deliver → recruit and resupply → repeat.

**Setting**: a Jupiter-like gas giant moon system. Trips of ~300,000–2,000,000 km, ~20–40 min of play each. Quiet stretches run in compressed time during 3–5G burns (crew in crash couches; in co-op, only when everyone is strapped in). Any contact or failure drops back to real time.

## Current state

- Unreal Engine **5.8** C++ project, freshly created. One runtime module: `Source/FlipAndBurn`.
- Git + LFS initialized locally (no remote yet; file locking needs one).
- Current focus: **Phase 1, the sim core** (see Roadmap).

## Architecture rules

These are deliberate decisions. Don't work around them without asking.

1. **Standalone sim core.** A plain C++ library (no Unreal types) using `double` and a **fixed 60 Hz timestep** with interpolation for rendering. Owns orbits, ships, projectiles, pressure, heat, and damage logic. Unreal sits on top for rendering, characters, input, audio, and networking. Actors only *display* sim state.
2. **Nested reference frames.** Every object stores position, rotation, and velocity relative to a parent frame in a tree (Space → planets/asteroids/ships → crew/crates/debris). Crossing a boundary (out a hole, onto an asteroid, into another ship) **reparents while preserving world position and velocity**. Build this from the start; retrofitting is painful.
3. **Ship interior never moves in its own frame.** Thrust and rotation are applied as felt forces on free objects:
   `a = -A - α × r - 2ω × v - ω × (ω × r)` (thrust gravity, spin-up shove, Coriolis, centrifugal).
   Standing players use a custom gravity direction; floating players and loose objects get the full formula.
4. **Physics split.** Ship and macro motion run in the custom sim, **not Chaos** (Chaos can't predict trajectories, fast-forward, keep orbits accurate, or handle km/s speeds). Chaos only handles local collisions inside a frame (crates, debris, fractured panels, ragdolls), fed apparent gravity by the sim.
5. **One apply-force-at-point function** covers drive, RCS, recoil, and breach thrust.
6. **Projectiles are data, not physics bodies**: position, velocity, frame, stepped per tick with a short raycast. Firing replicates once as an event; ballistics are deterministic.
7. **Rendering far objects**: everything drawn camera-relative. Distant bodies use scaled proxies (same angular size, closer and smaller). Scopes are narrow-FOV cameras. LOD/impostors for tiny distant ships.
8. **Networking**: player-hosted listen server, **replicate events, not state**, keep logic server-authoritative so a dedicated server is possible later.
9. **Hot loops stay in C++** (sim, projectiles, voxels, pressure, heat, frames). Blueprints are fine for UI and event glue.
10. **Performance**: distance is free, object count costs. Distant ships follow simple predicted paths; encounters spawn in a bubble ahead of the route; moons use precomputed orbits.

## Key systems (design reference)

**Hull damage (Siege-style, not server physics destruction)**: panels with ~10 cm cell grids; hits mark cells by caliber; a damage texture drives holes; large holes rebuild simplified collision; heavy damage swaps to a pre-fractured panel. Same grid drives interior and exterior. Replicate as `(panel, position, caliber)` events. Hits are stored on the target ship in its own frame at any range and rebuilt visually on approach.

**Decompression**: pressure per compartment, flowing through doors and holes, scaling with hole area and pressure difference. Escaping air is thrust (force + torque on the ship). Large holes eject crew into the Space frame; small holes pin them (~800 N on a 10 cm hole) until plugged.

**Weapons by range band**: torpedoes (2,000+ km, guided), railguns (<~500 km, unguided, big heat), point defense (<~10 km, auto-tracking). Small arms work in vacuum; recoil pushes and spins you in zero G (spin matters more than push). Ammo types: frangible, fragmenting, armor-piercing; caseless is a separate weapon line.

**Heat is a core resource**: shed only by radiating. Sources: reactor, drive, railguns, electronics, sunlight. Sinks: radiators (slow, fragile), heat sinks (fast, fill up), coolant dumping (consumable). Heat = infrared visibility; running cold enables stealth.

**Sound**: nothing outside is audible unless it physically reaches you (through air, hull, or suit/boots). Ship computer sonifies distant events.

**Mining**: smooth density+material voxels (Deep Rock style), chunked per asteroid, each its own frame. Mass and center of mass update as material is removed. Weapons carve the same voxels. Disconnected islands (flood fill) become loose bodies. Consider Voxel Plugin for meshing.

**Crew**: classes (Pilot, Engineer, Gunner, Miner) are better at one job, but anyone can do anything. No AI crewmates. Respawn by recruiting.

**LLM NPCs (later)**: STT → small local model (1–4B, quantized) → TTS. **The game decides intent, leverage, and allowed actions; the model picks from a fixed list in a strict format; the game validates.** Never let the model invent facts. For CQB, build a Ready-or-Not-style compliance system first so combat works without the LLM.

## Roadmap

Goal: a vertical slice proving the loop is fun. One ship, one short route, one enemy type, breaches and patching, two-player co-op.

**Phase 1: Sim core** ← current
- [ ] C++ library with doubles and a fixed timestep
- [ ] Ship body: thrust, rotation, flip and burn, fuel
- [ ] Point gravity from one or two bodies
- [ ] Projectiles as data with deterministic ballistics
- [ ] Nested frame tree with reparenting that preserves position and velocity
- [ ] Simple debug view to tune feel

Phase 1 starting point: a cylinder in empty space with zero gravity, a starfield, a sun, and static reference objects. Per-body state = position, velocity, quaternion, angular velocity, mass, inertia, force and torque accumulators, owned by a world subsystem. Add debug lines and readouts.
**Validation tests**: constant velocity with no force; 9.8 m/s² reads 1.0G; flip and burn stops where calculated; a box tumbles about its intermediate axis. Next: one gravity source, a better integrator, a stable circular orbit.

**Phase 2: Unreal foundation**: Git + LFS setup; sim core as a module; graybox interior from kit pieces; first-person character with apparent gravity and zero-G mode; window/scope cameras showing a distant ship.

**Phase 3: Damage and combat**: hull damage grid with masked holes; compartment pressure, flow, ejection; patching; one enemy ship with a railgun and simple AI.

**Phase 4: Co-op**: listen server with event replication for shots and holes; two players crewing one ship over a short route.

**Later**: heat, torpedoes and point defense, voxel mining, compressed burns, classes, voice NPCs, reputation.

## Conventions

- **Git commands**: never run git commands that change the repo (commit, branch, checkout, merge, rebase, reset, stash, push, tag, LFS lock/unlock, config changes, etc.) unless explicitly told to. Read-only commands (`status`, `log`, `diff`, `check-attr`) are fine.
- **Version control**: Git with LFS for binaries, file locking for `.uasset`/`.umap`, One File Per Actor for levels. Ignore `Binaries/`, `Intermediate/`, `Saved/`, `DerivedDataCache/`, `.vs/`.
- **Editor**: VS Code via the generated `FlipAndBurn.code-workspace` (Editor Preferences → Source Code Editor; Tools → Refresh Visual Studio Code Project). Requires MSVC Build Tools.
- **Hardware**: Unreal work on the Windows PC (3060 Ti, 32 GB). The sim core should stay engine-independent so it can also be built and tested on macOS.
- Keep the sim core free of Unreal headers so it can be unit tested standalone.

## Biggest risks

1. Navigation UI: making burn planning, intercepts, and slingshots understandable in first person.
2. Enemy combat AI that flies and fights well under realistic physics.
3. Scope: many interacting systems multiply integration work. Favor the vertical slice.
