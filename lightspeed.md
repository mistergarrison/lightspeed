# Light Speed — Design & Requirements

A single-file HTML5 real-time strategy game built on one idea: **information travels at a finite speed, so you command your empire from inside a delay.** You never see the galaxy as it is — only as it *was* when the light left. Orders you give now arrive later. Good play means reasoning about that gap and using it against an opponent who is reasoning about it too.

This document specifies the behavior the game must have and the numbers that define its balance and feel. It is deliberately quiet about *how* to build any of it. Where a mechanism is genuinely load-bearing (the perceived-time rule, deterministic simulation, combat resolution) it is stated precisely. Everything else — data structures, rendering technique, event plumbing, class layout, exact iteration counts — is the implementer's call. A short list of those open choices appears at the end.

**Constraints**

- One self-contained HTML file. ES6+ is fine.
- Rendering via PixiJS 7.3.2 (CDN). UI overlays via Tailwind (CDN). Minimap and graphs via the native 2D canvas.
- No build step, no server requirement for single-player.

---

## 1. The light-speed model

The engine keeps one authoritative **true state** — real positions, unit counts, and ownership, advanced by the simulation. Every observer (you and each AI) sees a **perceived state** derived from it, lagged by the light travel time from each event to that observer's home:

> perceived time at a point = current time − distance(point, observer's home) / C

Two consequences define the whole game:

- **You see the past.** A sun or fleet is rendered using its state at the perceived time for your home. Distant things are more stale than near ones. The same event looks different to two players whose capitals sit at different distances.
- **Your orders take time to arrive.** A command travels from your capital to where it needs to act, at speed C, and only changes the true state when it gets there. You are always acting on old information and your reaction is always late.

Both directions of delay are required. A build that lags the *visuals* but lets orders apply instantly (or vice versa) has missed the point.

Reconstructing perceived state means each sun must expose its state at an arbitrary past time. Keep enough history per sun to cover the worst-case galaxy-wide delay, and reconstruct intermediate moments by carrying production forward from the most recent checkpoint at or before the requested time. The buffer size, checkpoint cadence, and interpolation details are yours to choose; the requirement is that perceived reconstruction is correct back to the maximum possible delay.

---

## 2. Determinism (load-bearing)

The simulation advances in **fixed time steps** and all randomness derives from a single seed. This is not a stylistic preference — replay and multiplayer both depend on it, so it constrains the whole engine:

- The only consumers of randomness are **map generation** and **AI decisions**. Combat, production, movement, and command resolution are fully deterministic.
- Every command, from the player *and* from every AI, flows through one path and is recorded in a single ordered log with its issue and arrival times.
- No wall-clock time, `Math.random`, iteration-order ambiguity, or floating-point nondeterminism may leak into the simulation path.

Given this, the entire game is a pure function of `(seed, command log)`. That single fact is what makes the next two features cheap and correct:

- **Replay** is just the seed plus the log; re-simulating reproduces the game exactly.
- **Multiplayer** is lockstep: peers exchange commands and each simulates the same steps to the same result.

Treat any divergence between two runs of the same seed and log as a determinism bug, not a rounding quirk to be papered over.

---

## 3. Suns and the economy

Suns are the production nodes and the only thing you own. A sun has an owner, a position, a level, and a (fractional) unit count. Owned suns produce units up to a cap; production pauses at the cap.

| Level | Rate (units/s) | Cap | Cost to reach |
|------:|---------------:|----:|--------------:|
| 0 | 0 | 50 | — (neutral) |
| 1 | 0.4 | 100 | 80 |
| 2 | 1.0 | 250 | 250 |
| 3 | 2.4 | 600 | 600 |
| 4 | 6.0 | 1500 | — (max) |

- **Neutral suns** (unowned) sit at level 1 by default and do not grow. A neutral sun only produces if map generation seeded it at level 2 or higher. Capturing a sun lets it produce normally.
- **Upgrading** costs the listed amount immediately and takes 25 seconds. A hostile arrival during an upgrade cancels it (the spent units are already gone).
- Several techs modify a sun's rate, cap, or upgrade time (see §6). Those modifiers apply only to suns currently linked into your command network.

These numbers are the balance contract; keep them unless you are intentionally rebalancing.

---

## 4. Fleets and movement

A fleet is a group of units in transit from one sun to another. It moves in a straight line at the fleet speed and fights or reinforces when it arrives in the true state.

**Perceived visibility.** A fleet is only drawn to an observer during the window when its light would have reached them — roughly from (launch + distance(source, observer home)/C) to (arrival + distance(target, observer home)/C), with a small grace period at the end so arrivals don't flicker out. An observer can be unable to see a fleet that has, in truth, already arrived.

**Emission control ("quiet" fleets).** The player can launch fleets dark. A quiet fleet trades speed for stealth: it moves slower but enemies perceive it even later than light-lag alone would dictate. This is the seed of a whole strategic layer that the Deception and Forecast tech domains build on (§6). The base trade-off is required; its exact magnitudes are tunable.

---

## 5. Commands

All player and AI intents go through one ordered, logged path. The command set:

- **Attack** — send units from a source sun to a target. Double-click sends everything; a right-click send commits half. The order leaves your capital and the fleet launches only when the signal reaches the source.
- **Coordinated strike** — one order launching from several of your suns at once. With the Logistics capstone, the launches are timed so the fleets *arrive* together despite different distances.
- **Redirect** — retarget a fleet in flight. The order takes effect at the light-delayed point where it actually catches the fleet, not instantly; from there a new leg begins toward the new target. The interception is necessarily approximate — pick a method and accuracy; the requirement is that redirects respect the same signal delay as every other order.
- **Split** — divide an in-flight fleet so you can feint with one half. (Unlocked by a Logistics tech.)
- **Upgrade** — raise a sun's level; begins when the order arrives.
- **Dedicate** — flip owned suns between producing units and funding research (§6).
- **Set research** — choose the active tech.

Every command is gated by the command network: you can only issue it from or to suns you currently command (§6). The light delay on a command is the distance the signal must cover from your capital, so nearer orders land sooner — a real consideration when your frontier is far away.

---

## 6. The command network and relay range

You do not command every sun you own. You command the ones **linked back to your capital through space you currently control**, relayed sun to sun, where each hop is within a finite **relay range**. A sun you own but can't currently reach through the network is stranded: it still produces and defends, but you can't give it orders or route through it.

This turns territory into a connectivity problem and makes range a resource:

- The network is computed over **perceived** ownership — you route through the suns you *believe* you still hold, which may be stale.
- Base relay range is tied to how far apart stars are (with a floor); it is not infinite, and on an average map your reach falls short of the whole galaxy.
- **Signals research extends range** (Tightbeam ×1.4, Command Throughput a further ×1.2), and other techs cut command latency or jam the enemy's range. Reaching a contested star can require investing in range before you can act there at all — the gap is a real obstacle, not flavor.
- A toggle shows the network overlay so the player can see which suns are live and where the edge is.

This mechanic is required and is what the tutorial's research lesson is built around (§11).

---

## 7. Research and the tech tree

A second economy runs alongside units. You **dedicate** owned, linked suns to research: their production stops feeding unit growth and instead funds a shared research bank. Spend the bank on a tech tree; a tech completes when the bank reaches its cost.

The tree has **five domains**, each a spine → two forks → capstone. The spine unlocks the forks; the capstone requires a fork. Costs by tier: spine 120, fork 280, capstone 650. Most Industry and Logistics effects roll out per sun across your linked network rather than applying instantly everywhere.

The exact multipliers below are the current balance and are tunable; the *strategic role of each domain* is the requirement, because together they are the game's depth.

**Signals — reach and routing.** Tightbeam Relays (range ×1.4) · Command Throughput (range ×1.2 again) · Mesh Routing (orders ~15% sooner, resists jamming) · ★ Entangled Relay Pair (a zero-latency, jam-resistant lane from capital to frontier).

**Industry — output.** Refined Yields (+25% production) · Deep Reservoirs (+50% cap) · Rapid Assembly (upgrades in half the time) · ★ Industrial Core (a further +40% production and cap).

**Logistics — movement.** Tuned Drives (+30% fleet speed) · Fleet Division (enables splitting fleets) · Course Computers (redirect at greater range, tighter intercepts) · ★ Chronosynced Strike (multi-sun attacks arrive together).

**Deception — hiding your moves.** Emission Control (quiet fleets run darker, lose less speed) · Ghost Drives (quiet fleets go fully dark at full speed) · Holographic Spoofing (decoys delay an enemy spotting *any* of your fleets) · ★ Dead-Man's Beacon (captured suns keep broadcasting their old owner for a while).

**Forecast — seeing through the lag.** Light-Echo Forensics (extrapolate visible enemy fleet headings; see partly through emission control) · Predictive Telemetry (estimate an enemy sun's *present* strength, not its stale image) · Signal Triangulation (locate and mark enemy capitals) · ★ Interference Field (jam enemy command range unless they run Mesh Routing or an Entangled Pair).

Deception and Forecast are deliberately opposed: one degrades what the enemy can perceive, the other claws perception back. Keep that tension.

---

## 8. Combat, scoring, and victory

**Combat** happens in the true state the instant a hostile fleet arrives, and resolves as simultaneous mutual attrition: repeatedly remove the size of the smallest force present from every force, until one side remains. This rule is required — it makes reinforcement timing and combined arrivals matter. The resulting explosion is shown to each observer at *their* perceived time, which can be well after the battle actually happened.

**Score** is the sum of a faction's units on its suns and in its live fleets.

**Victory** is the player surviving while every AI faction is eliminated. **Defeat** is the player's score reaching zero. A defeated player may keep watching the galaxy play out.

---

## 9. Map generation

Generate stars procedurally inside a circular galaxy, denser toward the center, with a minimum spacing between them. Seed a minority of stars at higher level/garrison so some neutral territory is worth more and is harder to take. Place each faction's homeworld near the rim, spaced evenly by angle, by promoting the nearest star to a level-1 capital with a starting garrison.

The distribution shape (denser core), the spacing floor, and rim placement are the requirements. The exact density curve, the seed-richness probabilities, and the spacing value are balance dials — current defaults are in §13.

---

## 10. AI

Each AI plays on its own perceived state but acts on the true state, re-deciding at irregular intervals so factions don't move in lockstep. Its instincts: expand toward near, weak, or neutral targets it can actually reach and overwhelm; upgrade safe, wealthy suns; and invest in research over a match. Crucially, the AI issues through the same logged command path the player does — that is what lets a replay reproduce an AI-driven game exactly, so it is a determinism requirement, not just tidiness.

A single **difficulty dial** (roughly 0 to 1) scales how aggressive and how sharp the AI is, and at lower settings injects hesitation and mistakes (occasionally freezing a decision, mistargeting, or under-committing). The dial and its felt effect are required; the specific heuristics and thresholds are yours.

---

## 11. Game modes and the tutorial

The title is a hub. A landing screen offers four entries — **Tutorial**, **Solo Skirmish**, **Multiplayer**, **Watch a Replay** — plus a **Controls & hotkeys** page; every sub-screen returns to the landing. Solo Skirmish exposes the match settings (star count, opponents, game speed) before launch.

The in-match HUD shows a live leaderboard and a speed indicator, and toggles for help, the command-network overlay, and research. Pausing and game-over both surface a score-over-time graph for all factions and the appropriate navigation (resume/restart/menu, or play-again/observe/menu).

**The tutorial** is a guided solo match on a fixed seed that teaches the core ideas in order: select a sun; build and upgrade; capture a neighbor; *watch an order take time to arrive*; reveal the command network; and research. It has firm pedagogical requirements, because earlier drafts got these wrong:

- A step that asks for an action advances **only** when the player performs that action (or the outcome occurs). It must not offer a manual "next" that skips the task. Pure explanation steps may offer "Continue." The player can exit at any time.
- The research lesson must require **actually completing a tech that is needed to make progress** — not merely opening the tree. Concretely: the player holds an outpost stranded just beyond relay range, and only researching Tightbeam Relays brings it into the network so it can be commanded. The expected tech is highlighted in the tree while that step is active.
- Advancement keys off the player's *action*, not the resulting engine state, since light-lag means the consequence is delayed.

The number of steps and the exact copy are the implementer's. The teaching order and the two rules above are the requirements.

---

## 12. Multiplayer

Multiplayer reuses the deterministic engine: once a match starts, peers run **lockstep**, exchanging commands and simulating identically. The game owns the lobby and lifecycle; the network transport is a seam.

- **Lobby model.** A host holds the authoritative roster of slots, each slot a human, an AI, or open. Players reach a host by entering a code or by picking a host from a public game browser. The host assigns factions and starts the match; from there the engine drives the game over exchanged commands.
- **Transport is pluggable.** The spec requires a clean, documented seam for advertising/hosting/joining, the lobby message protocol, and handing the started match its transports — so a real transport (WebRTC, a relay, etc.) can be dropped in without touching game logic. Stubs that satisfy the seam are acceptable scaffolding; fakes that pretend to be networked are not.
- **Restrictive networks.** Plain peer-to-peer fails behind many corporate proxies; a usable build needs a relay/TURN path, not STUN alone. Note this in the seam.

The determinism contract in §2 is precisely what makes lockstep correct, which is why it is non-negotiable.

---

## 13. Replay

A finished game can emit a **replay code**: the seed, the match settings, and the full command log, encoded compactly. *Watch a Replay* takes such a code, re-creates the galaxy from the seed, and re-simulates by replaying the logged commands (the live AI is silent during playback — its decisions are already in the log). The guarantee is exact reproduction of the visible game.

A malformed code must fail gracefully rather than crash. Because replays re-simulate in the same engine, in-app playback is exact; sharing codes across genuinely different platforms is the one place floating-point reproduction is not guaranteed, which is worth a word in the UI if codes are ever shared.

---

## 14. Controls

| Input | Action |
|---|---|
| Left click | Select a sun |
| Left drag | Box-select |
| Shift + click | Add / remove from selection |
| Double-click target | Attack with 100% of selected units |
| Right-click target | Attack with 50%, or redirect selected fleets |
| Scroll | Zoom, centered on cursor |
| WASD / arrows | Pan (edge-pan in fullscreen) |
| U | Upgrade selected |
| X | Split selected fleet |
| N | Toggle command-network overlay |
| R | Toggle research / tech tree |
| G | Dedicate selected suns to research |
| Q | Toggle quiet (dark) orders |
| T | Toggle all fleet trajectories |
| V | Toggle signal-intel view (what a sun perceives of you) |
| H / L | Toggle help / leaderboard |
| M or Esc | Pause menu |
| F | Fullscreen |
| +/− | Game speed |
| Shift + D | Debug (force win/lose) |

Use unified pointer handling so touch and mouse both work, and suppress the browser context menu over the play area. The mapping is the UX contract; the input plumbing is yours.

---

## 15. Visual and audio direction

The look is minimalist ambient space: black background, factions distinguished by color, suns rendered as glowing bodies that subtly wobble and flare, fleets as trailing particle swarms, upgrades as pulsing rings. Level-of-detail by zoom (drop per-unit particles and fine labels when far out). A minimap in the corner shows the galaxy and viewport and is labeled to remind the player it, too, is **perceived reality**.

Two perceptual requirements matter more than any rendering choice:

- **Effects play at perceived time.** Explosions, flares, and command-signal pulses appear when their light reaches the viewer — not when the underlying event occurred in the true state.
- **Effects stay legible at speed.** At very high game speeds, slow the *visual* lifecycle of transient effects relative to the simulation so they remain visible. A reasonable approach is to scale effect animation time down once game speed passes a threshold.

How you build glows, particles, trails, and shockwaves is entirely open.

---

## 16. Reference constants (current defaults, tunable)

| Constant | Value | Constant | Value |
|---|---|---|---|
| Galaxy radius | 950 | Speed of light C | 60 |
| Fleet speed | 20 | Quiet-fleet speed ×| 0.6 |
| Min sun spacing | 105 | Homeworld rim factor | 0.9 |
| Upgrade duration | 25 s | Fixed sim step | 0.05 s |
| Zoom start / min / max | 1.5 / 0.5 / 10 | Battle-event retention | 120 s |
| Tightbeam range × | 1.4 | Throughput range × | 1.2 |
| Refined production × | 1.25 | Tuned fleet speed × | 1.3 |
| Interference (enemy range) × | 0.8 | Mesh latency × | 0.85 |

Faction colors: neutral white, player blue, AIs a fixed palette extended by generated hues past twelve factions. Spine/fork/capstone research costs: 120 / 280 / 650.

---

## 17. Left to the implementer

The following are intentionally unspecified — make reasonable engineering choices:

- All data structures, including how perceived history is stored and reconstructed, and the redirect interception method and its accuracy.
- Class and module layout, the game loop's internals, and any object pooling or performance work.
- Every rendering technique (glow, particles, trails, shockwaves, minimap drawing).
- Exact AI heuristics, thresholds, and scoring, beyond the behavior and difficulty dial in §10.
- The network transport behind the multiplayer seam, and the on-the-wire encoding of replay codes.
- Tutorial step count and copy, within the rules of §11.
- Final balance tuning of any number marked tunable here.

If a choice would break determinism (§2) or the two-directional light delay (§1), it is not a free choice — those are the spine of the game.
