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
    *   **Enemy-Owned (Red, Green, Purple, etc.):** Functions identically to a
        player Sun but for an AI opponent. They are primary targets for attack.
        Visible on the minimap as a dot of their respective color.
    *   **Neutral (Grey):** Possess a dull grey core and do not produce units
        (unless upgraded to level 2+). They start with a garrison of units that
        must be defeated to capture the Sun. Visible on the minimap as a grey
        dot.
*   **State History (Crucial for LSD):** To correctly render the perceived
    state, each Sun must maintain a history buffer (at least 50 entries) of
    state checkpoints. Each checkpoint stores the timestamp, unit count, owner,
    level, and production rate.

#### **B. Units (Army / Resource)**

Units are the mobile, commandable entities.

*   **Visual Design & Level of Detail (LOD):** The rendering of units must adapt
    to the camera's zoom level for visual clarity and performance.
    *   **Detailed View (Zoom Level >= 2.5):** When the camera is zoomed in to
        level 2.5 or higher, Units are rendered as small, glowing motes of light
        (simple circles in PixiJS), matching the color of their parent Sun. The
        number of visible circles orbiting a Sun must exactly equal the unit
        count for that Sun (capped at 50 for visual clarity).
    *   **Simplified View (Zoom Level < 2.5):** When the view is zoomed out
        below level 2.5, individual unit sprites must be hidden. The presence of
        units is indicated solely by the numerical display above the Sun.
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
    *   **Detailed View (Zoom Level >= 2.5):** The fleet is rendered as a swarm
        of individual, glowing motes of light, matching their faction color.
    *   **Simplified View (Zoom Level < 2.5):** When the view is zoomed out
        below level 2.5, the individual unit sprites are hidden. Instead, a
        single colored icon representing the fleet is displayed, accompanied by
        a numerical display indicating the total number of units in that fleet.
*   **Fleet Visibility Window:** A fleet launched at time `T_launch` from Source
    (dist `D_src` from observer) to Target (dist `D_tgt` from observer) arriving
    at `T_arrival` is only visible to the observer during the specific time
    window: `[T_launch + D_src/C, T_arrival + D_tgt/C]`.
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
    (from one edge to the opposite) is calibrated to be approximately 33 seconds
    (2000 / 60). The delay to any given Sun is proportional to its distance from
    the Player's Homeworld.
*   **Icons:**
    *   **Attack/Capture:** A sharp, arrow-like icon.
    *   **Upgrade:** A swirling, nova-like icon.
*   **Light Speed Delay:** The command only takes effect once the icon reaches
    the *Source Sun*.

### **III. Player Interaction & Controls (The Command System)**

*   **Setup Screen:**
    *   Before the game starts, a title screen allows the player to configure
        the game.
    *   **Number of Stars:** Slider to adjust the total number of suns (approx
        0-200+).
    *   **Number of Opponents:** Slider to adjust AI count (1-20).
    *   **Game Speed:** Slider to adjust the simulation speed multiplier.
*   **Navigation:**
    *   **Zoom:** Scrolling the mouse wheel zooms the view in and out, centered
        on the current mouse cursor position. The zoom level is a numerical
        scale where a zoom of 1.0 shows the entire 2000x2000 unit galaxy. On
        initialization, the view is centered on the player's Homeworld at an
        initial zoom level of 1.5. This control also governs the Level of Detail
        (LOD) switch for unit rendering (threshold 2.5).
    *   **Panning:** Moving the mouse cursor to the edges of the screen will pan
        the camera in that direction. The camera can also be panned using the
        WASD or arrow keys, or by dragging with the Right Mouse Button.
    *   **Minimap:** Shows a scaled-down representation of the entire galaxy,
        with colored dots indicating the presence and ownership of Suns.
        Clicking on a location in the minimap instantly jumps the main view to
        that area.
*   **Selection:**
    *   **Single Sun:** A single click on a player-owned Sun selects it. This is
        visually confirmed by a bright circle of the player's color appearing
        around the Sun.
    *   **Multiple Suns:** Left-click and drag to create a selection box that
        encompasses all desired friendly Suns. Implement a movement threshold
        (e.g., 5 pixels) to distinguish between a click and the start of a drag.
    *   **Add/Remove Selection:** Hold Shift while clicking to toggle selection
        of individual Suns.
*   **Issuing Commands & Light Speed (Homeworld Model):**
    1.  Select one or more player-owned Source Sun(s).
    2.  Issue a command:
        *   **Attack/Capture:**
            *   **Click Target Sun:** Deploys 50% of units from selected Source
                Sun(s).
            *   **Double-Click Target Sun:** Deploys 100% of units from selected
                Source Sun(s). Implement a timing threshold (e.g., 300ms) to
                detect double-clicks.
        *   **Upgrade:** Press the 'U' key.
    3.  A command icon is dispatched from the *Player's Homeworld* towards the
        *Source Sun(s)* at light speed.
    4.  The action (Unit Deployment or Upgrade) will only commence *after* the
        command icon reaches the Source Sun.
    5.  **Unit Deployment:** Upon command arrival at the Source Sun, the
        specified units launch towards the Target Sun at the defined Unit Speed.
    6.  **Upgrade:** Upon command arrival, the upgrade process begins at the
        Source Sun.
*   **Hotkeys:**
    *   **'U':** Upgrade selected Suns.
    *   **'F':** Toggle Full Screen.
    *   **'H':** Toggle Help panel.
    *   **'L':** Toggle Leaderboard.

### **IV. Core Gameplay Mechanics: The Strategic Actions**

#### **A. Information Delay**

*   **True State vs. Perceived State:** The game engine maintains a single,
    authoritative "True State" which is always current. All game logic (unit
    production, movement, combat outcomes) is resolved instantly within this
    state.
*   **Delayed Visibility (Interpolation):** The player does not see the True
    State. They see a "Perceived State" which is a delayed version of reality.
    To render this, calculate the perceived time `T_perceived = T_current -
    (distance_to_homeworld / C)`. Find the latest historical checkpoint for the
    Sun before `T_perceived` and linearly interpolate the unit count forward to
    `T_perceived` using the production rate at that checkpoint.
*   **Minimap Delay:** The minimap also reflects these information delays.

#### **B. Combat: A War of Attrition**

*   **Trigger:** Combat occurs exclusively at Suns. When a fleet of units
    arrives at a Sun that has units of an opposing faction (either orbiting or
    from another arriving fleet), a battle is initiated in the True State.
*   **Resolution Loop:** Combat is resolved instantly in the True State.
    1.  Identify all factions present at the Sun.
    2.  Sort factions by unit count (descending).
    3.  If more than one faction remains, subtract the unit count of the second
        strongest from the strongest. Eliminate the second strongest.
    4.  Repeat until 0 or 1 faction remains.
    5.  The player sees the battle play out visually with the corresponding LSD.

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
*   **The Process:** Upon command arrival, the upgrade timer starts (25
    seconds).
*   **Cancellation:** If hostile units arrive at the Sun during the upgrade
    timer, the upgrade is immediately cancelled and the timer resets.
*   **The Reward:** After the timer completes, the Sun's level increases,
    boosting production rate and maximum unit capacity. All visual cues and
    production changes are subject to LSD to the Player's Homeworld.

### **V. Game Balance & Setup**

#### **A. Constants:**

Parameter         | Value          | Notes
----------------- | -------------- | ---------------------------------
Galaxy Width      | 2000 units     |
Galaxy Height     | 2000 units     |
Galaxy Radius     | 950 units      | Suns generated within this radius
Light Speed (C)   | 60 units/sec   |
Unit Speed        | 20 units/sec   |
Upgrade Duration  | 25 seconds     | Time to complete an upgrade
Min Sun Spacing   | 80 units       |
**Sun Levels**    | **Rate (u/s)** | **Capacity**
Level 0 (Neutral) | 0              | 50
Level 1           | 0.4            | 100
Level 2           | 1.0            | 250
Level 3           | 2.4            | 600
Level 4           | 6.0            | 1500

#### **B. Map Generation:**

*   **Shape:** Circular region of radius 950 centered in a 2000x2000 square.
*   **Number of Suns:** Adjustable via slider (default 150).
*   **Players:** 1 Player (Blue) vs Adjustable AI (1-20).
*   **Distribution:** Suns are distributed within the circular galaxy. The
    distribution of Suns should be higher at the center and decrease linearly
    towards the edge.
*   **Starting Conditions:**
    *   The Player (Blue) starts with one Level 1 Sun and 50 units, located on
        the outer rim of the galaxy (radius * 0.9).
    *   AI players start with one Level 1 Sun and 50 units, also located on the
        outer rim, distributed evenly.
    *   Neutral suns are randomly assigned levels and unit counts.

#### **C. AI Opponent Behavior:**

*   **Goal:** Expand and eliminate others.
*   **Decision Triggers:** AI evaluates actions every 5-10 seconds.
*   **Priorities:**
    1.  **Upgrade:** Upgrade a Sun if unit count > 1.5x upgrade cost and level
        < 3.
    2.  **Attack:** Attack weakest enemy Sun within 800 units range if combined
        forces > 1.2x enemy units + 10. Prioritize neutral suns.
*   **LSD Compliance:** AI commands and information are ALSO subject to Light
    Speed Delay, calculated from their own Homeworld. The AI does not cheat; it
    does not have access to the "True Game State". It must make all decisions
    based on its own "Perceived State" of the galaxy.

### VI. Implementation Notes

*   **Rendering Framework:** To render the potentially large number of units (up
    to 50,000) performantly, the project must use the **PixiJS** 2D WebGL
    rendering framework. PixiJS can be included via a single `<script>` tag from
    a CDN.
*   **Particle Pooling:** To maintain performance with thousands of unit
    particles, implement an object pool for the PixiJS sprites used for units.
    Pre-allocate a pool of sprites and reuse them instead of creating/destroying
    them every frame.
*   **Level of Detail (LOD) Implementation:** The visibility of unit sprites
    must be dynamically managed based on the camera's zoom level (the scaling
    factor of the PixiJS stage/viewport). When the scale drops below a
    predefined threshold (2.5), the visibility of the unit sprites should be
    disabled.
*   **Sound:** Optional.
