Create a single HTML file with embedded JavaScript and CSS to implement a 2D web
game called "Light Speed". The game should function according to the mechanics
outlined below.

These instructions are long, and the implementation will be complicated, so make
sure that your implementation is modular and well-commented.

### **I. Game Objective & Aesthetic**

The fundamental objective is strategic domination. The player must use their
colored units to conquer all neutral and enemy "Suns" on the map, ultimately
eliminating all other colored factions to be the last one standing. The game's
aesthetic is minimalist and ambient, set against a black, star-filled void. The
only elements are glowing, celestial-like bodies (Suns) and their swarms of
particle-like Units. Use PixiJS for all rendering.

**The Light Speed Delay (LSD):** A core mechanic is the adherence to the speed
of light. The game engine will maintain a "True State" of the galaxy, which is
updated in real-time. However, the player will only see a "Perceived State".
Events that occur in the True State (battles, captures, unit movements) are
queued and only become visible to the player after a delay corresponding to the
time it would take for light from that event to reach the player's Homeworld.
This means the player is always viewing the past.

**Observer Point:** The player's starting Sun is their "Homeworld". All Light
Speed Delays for commands issued by the player, and information received by the
player, are calculated based on the distance to or from this Homeworld. Each AI
player also has its own Homeworld and its own Perceived State of the galaxy.

### **II. Core Entities: A Detailed Look**

#### **A. Suns (Production Centers / Bases)**

Suns are the static, central pillars of gameplay.

*   **Visual Design:** Each Sun consists of a glowing, procedurally generated
    core (ideally implemented with shaders for performance) surrounded by one or
    more faint, pulsing concentric rings. The size of the core and the
    number/prominence of these rings indicate its upgrade level. A faction's
    Homeworld is visually distinct, perhaps with a brighter, more prominent set
    of rings or a unique core pulsation, to differentiate it from other owned
    Suns. On the minimap, Suns will appear as colored dots corresponding to
    their owner.
*   **Unit Capacity:** Each Sun has a maximum number of units it can support in
    orbit. Production halts when this capacity is reached. Upgrading the Sun
    increases this capacity.
*   **Numerical Display:** A small, non-intrusive number is displayed above each
    Sun, indicating the current number of orbiting units. **This display must
    always be visible, regardless of zoom level,** and serves as the sole
    indicator of strength when individual units are hidden.
*   **States & Ownership:**
    *   **Player-Owned (Blue):** Actively and automatically produces blue Units
        up to its capacity. Can be selected by the player to issue commands or
        to be upgraded. Visible on the minimap as a blue dot.
    *   **Enemy-Owned (Red, Green):** Functions identically to a player Sun but
        for an AI opponent. They are primary targets for attack. Visible on the
        minimap as a dot of their respective color.
    *   **Neutral (Grey/White):** Possess a dull grey core and do not produce
        units. They start with a garrison of grey/white units that must be
        defeated to capture the Sun. These garrisons do not regenerate. Visible
        on the minimap as a grey/white dot.

#### **B. Units (Army / Resource)**

Units are the mobile, commandable entities.

*   **Visual Design & Level of Detail (LOD):** The rendering of units must adapt
    to the camera's zoom level for visual clarity and performance.
    *   **Detailed View (Zoom Level >= 3):** When the camera is zoomed in to
        level 3 or higher, Units are rendered as small, glowing motes of light
        (simple circles in PixiJS), matching the color of their parent Sun. The
        number of visible circles orbiting a Sun must exactly equal the unit
        count for that Sun.
    *   **Simplified View (Zoom Level < 3):** When the view is zoomed out below
        level 3, individual unit sprites must be hidden. The presence of units
        is indicated solely by the numerical display above the Sun.
*   **Default Behavior (Orbiting):** When not under a command, Units produced by
    a Sun will form a chaotic, swirling cloud of light motes around it, up to a
    Sun's capacity. The movement should be random and swirly, not rigid circular
    orbits. (This behavior is only visually apparent when zoomed in).
*   **Commanded Behavior (Movement):** When directed to a target, the selected
    group of Units moves in a straight line from their Source Sun towards the
    Target Sun's core. They do not engage enemies while in transit; combat only
    occurs upon arrival at a destination Sun. Fleets from opposing factions will
    pass through each other harmlessly in deep space. The rendering of these
    in-transit fleets must adhere to the same Level of Detail (LOD) rules as
    units orbiting a Sun:
    *   **Detailed View (Zoom Level >= 3):** The fleet is rendered as a swarm of
        individual, glowing motes of light, matching their faction color.
    *   **Simplified View (Zoom Level < 3):** When the view is zoomed out below
        level 3, the individual unit sprites are hidden. Instead, a single
        colored icon representing the fleet is displayed, accompanied by a
        numerical display indicating the total number of units in that fleet.
*   **Properties:** Units have no individual health. They are discrete entities:
    they either exist or they are destroyed. They serve as a single resource for
    combat, capture, and upgrading.

#### **C. Commands (In Transit)**

*   **Visual Representation:** When a command is issued, a unique icon
    representing the command type (e.g., Attack, Upgrade) emanates from the
    *player's Homeworld* and travels towards the *Source Sun* at the speed of
    light. This visual cue indicates the command is in transit. Players cannot
    see enemy commands in transit.
*   **Maximum Delay:** The time it takes for light to cross the entire galaxy
    (from one edge to the opposite) is calibrated to be approximately 60
    seconds. The delay to any given Sun is proportional to its distance from the
    Player's Homeworld.
*   **Icons:**
    *   **Attack/Capture:** A sharp, arrow-like icon.
    *   **Upgrade:** A swirling, nova-like icon.
*   **Light Speed Delay:** The command only takes effect once the icon reaches
    the *Source Sun*.

### **III. Player Interaction & Controls (The Command System)**

*   **Navigation:**
    *   **Zoom:** Scrolling the mouse wheel zooms the view in and out, centered
        on the current mouse cursor position. The zoom level is a numerical
        scale where a zoom of 1.0 shows the entire 1000x1000 unit galaxy. A zoom
        of 2.0 shows a 500x500 unit area (a quarter of the map), a zoom of 4.0
        shows a 250x250 area, and so on. On initialization, the view is centered
        on the player's Homeworld at an initial zoom level of 5.0. This control
        also governs the Level of Detail (LOD) switch for unit rendering.
    *   **Panning:** Moving the mouse cursor to the edges of the screen will pan
        the camera in that direction. The camera can also be panned using the
        WASD or arrow keys. screen. This shows a scaled-down representation of
        the entire galaxy, with colored dots indicating the presence and
        ownership of Suns. Clicking on a location in the minimap instantly jumps
        the main view to that area.
*   **Selection:**
    *   **Single Sun:** A single click on a player-owned Sun selects it. This is
        visually confirmed by a bright circle of the player's color appearing
        around the Sun.
    *   **Multiple Suns:** Click and drag to create a selection box that
        encompasses all desired friendly Suns.
*   **Issuing Commands & Light Speed (Homeworld Model):**

    1.  Select one or more player-owned Source Sun(s).
    2.  Issue a command:
        *   **Attack/Capture:**
            *   **Click Target Sun:** Deploys 50% of units from selected Source
                Sun(s).
            *   **Double-Click Target Sun:** Deploys 100% of units from selected
                Source Sun(s).
        *   **Upgrade:** Click-and-Hold on the selected Source Sun for 0.5
            seconds.
    3.  A command icon is dispatched from the *Player's Homeworld* towards the
        *Source Sun(s)* at light speed.
    4.  The action (Unit Deployment or Upgrade) will only commence *after* the
        command icon reaches the Source Sun.
    5.  **Unit Deployment:** Upon command arrival at the Source Sun, the
        specified units launch towards the Target Sun at the defined Unit Speed.
    6.  **Upgrade:** Upon command arrival, the upgrade process begins at the
        Source Sun.

*   **Rally Lines:** Not implemented in this version.

### **IV. Core Gameplay Mechanics: The Strategic Actions**

#### **A. Information Delay**

*   **True State vs. Perceived State:** The game engine maintains a single,
    authoritative "True State" which is always current. All game logic (unit
    production, movement, combat outcomes) is resolved instantly within this
    state.
*   **Delayed Visibility:** The player does not see the True State. They see a
    "Perceived State" which is a delayed version of reality. When an event
    (e.g., a battle starting, a sun being captured) happens at a location in the
    True State, that event is timestamped. It will only be rendered on the
    player's screen after a delay equal to `distance_from_event_to_Homeworld /
    C`. This means the player is effectively watching historical replays of
    events that have already concluded in the True State.
*   **Minimap Delay:** The minimap also reflects these information delays.

#### **B. Combat: A War of Attrition**

*   **Trigger:** Combat occurs exclusively at Suns. When a fleet of units
    arrives at a Sun that has units of an opposing faction (either orbiting or
    from another arriving fleet), a battle is initiated in the True State.
*   **Resolution:** Combat is a direct, numerical exchange. For every attacking
    unit, one defending unit is destroyed. It is a strict 1-for-1 trade until
    one side is eliminated. This is resolved instantly in the True State; the
    player sees the battle play out visually with the corresponding LSD.

#### **C. Capturing a Sun (Expansion)**

*   **Instant Conversion:** Capturing a Sun is a direct result of winning a
    battle at that location. If an attacking force defeats all defending units
    at a Sun (whether neutral or enemy-owned), the Sun is instantly converted to
    the attacker's faction.
*   **No Additional Cost:** There is no separate "capture cost" or process. The
    cost of capture is simply the cost of winning the battle.
*   **Surviving Units:** Any attacking units that survive the battle will
    immediately begin orbiting their newly acquired Sun.
*   **Notification:** The change in the Sun's ownership and color is an event in
    the True State. Notification of this change is subject to LSD to the
    Player's Homeworld.

#### **D. Upgrading a Sun (Economic Investment)**

*   **The Cost:** Units are consumed from the orbiting swarm to upgrade.
*   **The Process:** Consumed units are absorbed, bright "nova" effect.
*   **The Reward:** Increased production rate and maximum unit capacity. All
    visual cues and production changes are subject to LSD to the Player's
    Homeworld.

### **V. Game Balance & Setup**

#### **A. Constants:**

Parameter        | Value           | Notes
---------------- | --------------- | ----------------------------
Galaxy Width     | 1000 units      | Arbitrary units for distance
Galaxy Height    | 1000 units      | Arbitrary units for distance
Light Speed (C)  | 16.67 units/sec | Galaxy Width / 60 seconds
Unit Speed       | 8.33 units/sec  | 0.5 * C
**Sun Levels**   | **Lvl 1**       | **Lvl 2**
Prod. Rate (u/s) | 1               | 2
Max Capacity     | 100             | 150
Upgrade Cost     | -               | 80 units
**Neutral Suns** | **Small**       | **Medium**
Initial Garrison | 15              | 30
Base Level       | 1               | 1

#### **B. Map Generation:**

*   **Shape:** The game takes place in a 1000x1000 unit square area. However,
    Suns are only generated within a circular region of radius 500 centered in
    this square.
*   **Number of Suns:** 200
*   **Players:** 1 Player (Blue) vs 2 AI (Red, Green)
*   **Distribution:** Suns are distributed within the circular galaxy. The
    distribution of Suns should be four times higher at the exact center of the
    galaxy than at the outermost edge (radius 500). The density should decrease
    linearly with the distance from the center.
*   **Minimum Distance:** Maintain a minimum distance of 20 units between Sun
    centers.
*   **Starting Conditions:**
    *   The Player (Blue) starts with one Level 1 Sun and 50 units, located on
        the outer rim of the galaxy.
    *   The two AI players (Red and Green) each start with one Level 1 Sun and
        50 units, located on the outer rim, on opposite sides of the galaxy from
        each other, and roughly equidistant from the player's start.
    *   Neutral suns are randomly assigned Small, Medium, or Large types.

#### **C. AI Opponent Behavior:**

*   **Goal:** Expand and eliminate others.
*   **Decision Triggers:** AI evaluates actions every 5-10 seconds.
*   **Priorities:**
    1.  **Expand:** Attack nearest Neutral Sun if local forces > 1.5x garrison.
    2.  **Upgrade:** Upgrade a Sun if unit count > 1.8x upgrade cost and no
        immediate threats.
    3.  **Attack:** Attack weakest enemy Sun if combined forces from nearby
        Suns > 1.2x enemy units.
*   **LSD Compliance:** AI commands and information are ALSO subject to Light
    Speed Delay, calculated from their own Homeworld. The AI does not cheat; it
    does not have access to the "True Game State". It must make all decisions
    based on its own "Perceived State" of the galaxy, which is just as delayed
    and potentially out-of-date as the player's.

### VI. Implementation Notes

*   **Rendering Framework:** To render the potentially large number of units (up
    to 50,000) performantly, the project must use the **PixiJS** 2D WebGL
    rendering framework. PixiJS can be included via a single `<script>` tag from
    a CDN.
*   **Level of Detail (LOD) Implementation:** The visibility of unit sprites
    must be dynamically managed based on the camera's zoom level (the scaling
    factor of the PixiJS stage/viewport). When the scale drops below a
    predefined threshold (zoomed out), the visibility of the unit sprites should
    be disabled (e.g., setting a container's `visible` property to `false`).
    This prevents rendering thousands of sub-pixel sprites, drastically
    improving performance.
*   A tranquil, procedurally generated soundscape can be attempted using the Web
    Audio API for subtle background ambience and simple tones for events (e.g.,
    command issued, battle, capture). No external audio files.
