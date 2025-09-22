# Light Speed — Full LLM Prompt

**Goal:** Create a single HTML file with embedded JavaScript and CSS that
implements a 2D web game called **"Light Speed"** using **PixiJS** for
rendering. The implementation must be modular, well-documented, and tuned for
performance while strictly following the mechanics below.

> **High-level rules:** - The engine maintains a single authoritative **True
> State** (all game logic resolved here). Each faction (player + AIs) has its
> own **Perceived State**, which is a delayed view of the True State computed
> from that faction's Homeworld using light-travel delays. - All timings and
> visual delays are computed with an explicit `LIGHT_SPEED_C` constant;
> distances are in game spatial units.

--------------------------------------------------------------------------------

## CONSTANTS (top-level, tweakable)

Place these near the top of the implementation for easy tuning.

```js
const CONSTANTS = {
  GALAXY_WIDTH: 1000,
  GALAXY_HEIGHT: 1000,
  GALAXY_RADIUS: 500,
  LIGHT_SPEED_C: 16.67,      // units / sec (~1000 / 60)
  UNIT_SPEED: 8.33,         // units / sec (default 0.5 * C)
  ZOOM_START: 5.0,
  LOD_ZOOM_THRESHOLD: 3.0,
  SPRITE_CAP_PER_SUN: 200,
  SPRITE_CAP_PER_FLEET: 400,
  UPGRADE_DURATION: 5.0,    // seconds in True State
  RANDOM_SEED: null,        // optional deterministic seed for maps
  SOUND_ENABLED: false,
  MIN_SUN_SPACING: 20,
  NUM_SUNS: 200,
  HOMEWORLD_RIM_FACTOR: 0.95 // homeworld radius fraction of GALAXY_RADIUS
};
```

--------------------------------------------------------------------------------

# I. Game Objective & Aesthetic

The objective is strategic domination: capture/own all Suns (production
centers). The visual style is minimalist and ambient on a black, starry void.
All rendering must use **PixiJS** (CDN include allowed). Prefer WebGL and
batching where possible.

**Core mechanic — The Light Speed Delay (LSD):** Maintain a True State where all
logic (production, movement, combat, upgrades) occurs instantly in that
authoritative model. Each faction views a **Perceived State**, which is a
time-delayed replay of True-State events according to light travel time to that
faction's Homeworld.

**Observer Point:** A faction's Homeworld is its observation origin for LSD. All
delays for commands and perceived events are computed relative to Homeworld
positions.

--------------------------------------------------------------------------------

# II. Core Entities (Detailed)

## A. Suns (Production Centers / Bases)

-   **Static Entities** placed within the galaxy. Each Sun has:

    -   `position` (x,y), `level`, `owner` (FactionID or Neutral),
        `orbitingUnits` (integer), `productionRate` (u/s based on level),
        `maxCapacity`.

-   **Visual Design:** Glowing, procedurally-generated core + pulsing concentric
    rings. Homeworlds have distinct ring animation/saturation. On minimap, suns
    are colored dots by owner.

-   **Always-visible numerical display:** A numeric label above each Sun shows
    the true orbiting unit count (in the Perceived State view this count is
    shown as the perceived value). This label must always remain readable
    regardless of zoom (scale the label inversely with zoom or use a
    fixed-screen-space overlay).

-   **States & Ownership:**

    -   **Player-Owned** (Blue)
    -   **Enemy-Owned** (Red, Green — AI)
    -   **Neutral** (Grey/White) — neutral Suns have initial garrisons and do
        not produce units (unless base level > 1).

## B. Units (Army / Resource)

-   **No per-unit HP.** Units are discrete, identical resources.
-   **Orbiting behavior:** When idle, orbiting units form a visual "swirl"
    around their Sun. Implementation hint: base angular velocity + Perlin noise
    radial jitter.
-   **Commanded movement:** When launched, a group becomes a *fleet* traveling
    in a straight line from Source Sun to Target Sun at `UNIT_SPEED`. Fleets do
    not interact in deep space; combat only resolves at Suns.

### Visual LOD rules for Units/Fleets

-   Use **Zoom Factor** `z` semantics (see Section V for mapping). If `z >=
    LOD_ZOOM_THRESHOLD` show detailed sprites; otherwise hide sprites and show
    aggregated icons.
-   **Detailed View (z >= threshold):** Render up to `min(unitCount,
    SPRITE_CAP_PER_X)` sprites; when unitCount ≤ cap, instantiate that many
    sprites (one-to-one). When unitCount > cap, render the cap number of sprites
    plus a density/fill visual and always show the numeric label.
-   **Simplified View (z < threshold):** Hide individual sprites; show a single
    fleet icon with numeric count.

## C. Commands (In Transit)

-   Visual command icons are **spawned at the issuing faction's Homeworld** and
    travel at `LIGHT_SPEED_C` to the Source Sun.
-   **Only the issuing faction sees its outgoing command icons in transit.**
    Other factions will not see commands in transit; they only see events when
    those events are perceived (subject to LSD).

--------------------------------------------------------------------------------

# III. Player Interaction & Controls (Command System)

## Navigation

-   **Zoom:** Mouse wheel zooms centered on cursor. Zoom factor `z` maps to
    visible width = `GALAXY_WIDTH / z`. Start at `z = ZOOM_START` centered on
    player's Homeworld. LOD toggles at `z >= LOD_ZOOM_THRESHOLD`.
-   **Panning:** Edge-of-screen pan + WASD/arrow keys. Implement smoothing.
-   **Minimap:** Shows the player's **Perceived State**. Clicking the minimap
    recenters the main camera **immediately** (UI action, not delayed). An
    optional `minimapAnimatedJump` toggle can animate the pan instead of
    instant.

## Selection

-   Click to select a single player-owned Sun. Click-and-drag to select
    multiple.
-   Visual confirmation: bright colored circle around selection. Selection is an
    instantaneous UI action and is not subject to the light-speed delay, which
    only applies to commands (see below).

## Issuing Commands & LSD (ordered steps)

1.  Select one or more Source Suns (player-owned).
2.  Issue a command type:
    -   **Attack/Capture:** click Target Sun → deploy 50% of units from selected
        Source Sun(s).
    -   **Double-click Target Sun:** deploy 100%.
    -   **Upgrade:** click-and-hold on Source Sun for 0.5 sec to queue an
        upgrade.
3.  A command icon spawns at the Player's Homeworld and travels to the Source
    Sun at `LIGHT_SPEED_C`.
4.  The command **takes effect in the True State** when the icon reaches the
    Source Sun at:

    `t_effect_true = t0 + distance(Homeworld, SourceSun) / LIGHT_SPEED_C`

5.  Upon command effect (True State): units launch from the Source Sun toward
    Target Sun at `UNIT_SPEED` (for Attack) or the upgrade begins (for Upgrade).

6.  Any subsequent events (arrivals, captures) are timestamped in True State and
    will be perceived by an observer with Homeworld H at:

    `t_perceived = t_event + distance(eventLocation, H) / LIGHT_SPEED_C`

7.  Command icons are visible only to the issuing faction while in transit.

**Note:** Fleets in transit are visible in the issuing faction's Perceived State
(subject to rendering LOD). Other factions only see the consequences when
perceived.

--------------------------------------------------------------------------------

# IV. Core Gameplay Mechanics

## A. Information Delay

-   **True State vs Perceived State:** The server or authoritative engine
    resolves all mechanics in True State. Each faction maintains a Perceived
    State buffer computed by replaying True-State events delayed by `distance /
    C` from the event location to the faction's Homeworld.
-   Both main view and minimap use the Perceived State for display.

## B. Combat: deterministic multi-faction resolution

-   **Combat only at Suns.** When a fleet arrives at a Sun that contains
    opposing orbiting units (or other fleets that have also arrived at the same
    True-state timestamp), resolve combat atomically in True State.

### Simultaneous Arrivals & Multi-faction Combat (algorithm)

1.  At a True-state timestamp where arrivals occur, compute current unit counts
    per faction at that Sun (orbiting units + all fleets that arrived at that
    timestamp).
2.  Loop until ≤1 faction has units > 0:
    -   Sort factions by descending current unit count.
    -   Let A be top faction, B be second. Remove `m = min(units[A], units[B])`
        from both.
3.  Remaining faction with units > 0 (if any) becomes owner; surviving units
    orbit.
4.  If zero factions remain, Sun becomes Neutral with 0 units.
5.  **Tie-breaker:** stable faction ID ordering (e.g., Blue < Red < Green)
    resolves equal counts deterministically.

This generalizes 1-for-1 exchange to multi-faction arrivals and ensures
reproducible outcomes.

## C. Capturing a Sun

-   Capturing occurs when a faction eliminates all other units at the Sun in
    True State. Surviving attackers occupy the Sun and orbit.
-   No separate capture cooldown or cost — only the cost of units spent in
    combat.
-   Ownership changes are True-State events and will be perceived by each
    faction after the appropriate LSD.

## D. Upgrading a Sun (explicit)

-   On upgrade-command arrival in True State:
    -   Immediately **consume** the specified units from the Sun's orbiting
        pool.
    -   Start an `UPGRADE_DURATION` second timer in True State.
    -   Production rate/capacity change only applies on completion.
    -   If the Sun is captured during the upgrade window, the upgrade is
        canceled and consumed units are lost.
-   Default behavior: production continues during upgrade (configurable via
    CONSTANTS).

--------------------------------------------------------------------------------

# V. Game Balance & Setup

## A. Constants & Tuning (summary table)

-   Galaxy Width: 1000 units
-   Galaxy Height: 1000 units
-   Light Speed C: 16.67 units/sec
-   Unit Speed: 8.33 units/sec
-   Start zoom: `z = ZOOM_START = 5.0`
-   LOD threshold: `z >= 3.0`
-   Max Suns: `NUM_SUNS = 200`

## B. Sun Levels & Upgrade Costs (explicit)

Level | Prod. Rate (u/s) | Max Capacity | Upgrade Cost (to next)
----: | ---------------: | -----------: | ---------------------:
1     | 1                | 100          | 80 (→ Level 2)
2     | 2                | 150          | 160 (→ Level 3)
3     | 4                | 225          | 320 (→ Level 4)
4     | 8                | 350          | — (max level)

## C. Neutral Sun types

-   **Small:** initial garrison = 15, base level = 1
-   **Medium:** initial garrison = 30, base level = 1
-   **Large:** initial garrison = 60, base level = 2

## D. Map generation (precise radial distribution)

-   Galaxy radius `R = GALAXY_RADIUS` (500). Desired density ratio:
    `density(r=0) = 4 × density(r=R)`.
-   Use radial PDF: `p(r) ∝ (4 - 3 * r / R)`.

**Sampling recipe (accept/reject):** 1. Sample `r_candidate` uniformly from `[0,
R]`. 2. Sample `u` uniformly from `[0,1]`. 3. Accept `r_candidate` if `u <= (4 -
3 * r_candidate / R) / 4`, else reject and repeat. 4. Sample `θ` uniformly from
`[0, 2π)` and convert to `(x,y)`. 5. If `x,y` violates `MIN_SUN_SPACING` to
existing Sun centers, reject and retry.

-   Place `NUM_SUNS = 200` using the above sampler.

## E. Starting conditions & Homeworld placement

-   **Player (Blue):** Start with one Level 1 Sun with 50 units placed on the
    rim at radius ≈ `HOMEWORLD_RIM_FACTOR * R` and an arbitrary angle.
-   **AIs (Red, Green):** Each starts with one Level 1 Sun with 50 units on the
    rim at angles approx. +120° and −120° relative to player (small angle jitter
    allowed).
-   `homeworldPlacementMode` constant may be `rim` (default), `center`, or
    `random`.

## F. AI Opponent Behavior (clarified)

-   Each AI keeps a Perceived State based on its Homeworld and applies LSD.
-   Decision loop: evaluate actions every `T_ai` seconds, jittered uniformly in
    `[5,10]` seconds, plus tiny micro-jitter to de-sync AIs.
-   Simple heuristics:
    -   **Expand:** attack nearest Neutral Sun if local combined perceived
        forces > 1.5 × garrison.
    -   **Upgrade:** upgrade if perceived units > 1.8 × upgrade cost and no
        perceived threats within travel time window.
    -   **Attack:** combine nearby Suns if combined perceived force > 1.2 ×
        perceived enemy units.
-   AI uses the same command dispatch (its commands are delayed by light travel
    time from its Homeworld to Source Sun).

--------------------------------------------------------------------------------

# VI. Implementation Notes & Performance

## Rendering Framework

-   Use **PixiJS** (include via single `<script>` tag CDN). Use
    `ParticleContainer` / batching where possible.
-   Use shader-based procedural core for Suns if feasible; fallback to
    sprite-based cores otherwise.

## LOD & Sprite Caps

-   Use `CONSTANTS.SPRITE_CAP_PER_SUN` and `CONSTANTS.SPRITE_CAP_PER_FLEET` to
    limit sprites.
-   When `unitCount > SPRITE_CAP`, render capped sprites + aggregated particle
    emitter or density blob; numeric label must show true count.

## Determinism

-   Timestamp all True-State events as floating-second timestamps and resolve in
    timestamp order.
-   For deterministic replays/tests, supply `RANDOM_SEED` and use stable sorting
    and seeded RNG for tie-breaking.

## Sound & Rally Lines (placeholders)

-   Add `AudioManager` stub and `SOUND_ENABLED` toggle. Use procedural WebAudio
    tones by default; no external audio files.
-   Add `RallyLineManager` stub API; leave implementation for later.

## Large-scale performance considerations

-   Target performance to handle up to 50,000 logical units distributed across
    Suns and fleets, but **never** instantiate that many sprites. Use
    aggregation, particle emitters, and numeric overlays.
-   Use `visible = false` on sprite containers when outside LOD.

--------------------------------------------------------------------------------

# Developer Hints & Clarifications (explicit)

-   **True State vs Perceived State:** Always make this distinction explicit in
    code. True-state timers control game logic; Perceived-state is a replay
    buffer subscribed to True-state events filtered by `distance / C` delay per
    observer.
-   **Command visibility:** Only issuing faction sees command icons while in
    transit.
-   **Combat resolution:** Use the multi-faction deterministic algorithm above
    to support simultaneous arrivals.
-   **Homeworld rim placement:** This is intentional to create LSD asymmetries.
    For center-based playtests, change `homeworldPlacementMode`.
-   **Testing hooks:** Expose `CONSTANTS.RANDOM_SEED`,
    `CONSTANTS.LIGHT_SPEED_C`, `CONSTANTS.UNIT_SPEED`, and zoom constants to
    allow automated tests and tuning.

--------------------------------------------------------------------------------

# Deliverable Requirements

-   A single HTML file with embedded JS & CSS (modular code sections allowed
    within the file). Must be executable by opening in a browser.
-   Use PixiJS for all rendering.
-   Well-commented code with a short top-level README block in-code explaining
    how to change constants and run the file.
-   Include test-mode toggles: `DEBUG_SHOW_TRUE_STATE`, `RANDOM_SEED` input,
    `SOUND_ENABLED`, `minimapAnimatedJump`.
