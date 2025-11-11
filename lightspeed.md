# Light Speed - Game Implementation Prompt

You are to implement a single-file HTML5 real-time strategy game called **"Light
Speed"**. The game explores the concept of the speed of light as a strategic
constraint: players command an empire where information and orders travel at a
finite speed. You see the universe as it *was*, not as it *is*.

**Tech Stack:**

*   **HTML5/JS:** Single file, ES6+ features allowed.
*   **Rendering:**
    [PixiJS v7.3.2](https://cdnjs.cloudflare.com/ajax/libs/pixi.js/7.3.2/pixi.min.js)
    (via CDN).
*   **Styling:** [Tailwind CSS](https://cdn.tailwindcss.com) (via CDN) for UI
    overlays.
*   **Minimap & Graphs:** Native HTML5 `<canvas>` 2D context.

--------------------------------------------------------------------------------

## **I. Core Concept: The Light Speed Delay (LSD)**

The game engine must maintain two distinct states:

1.  **True State:** The actual, real-time state of the galaxy (unit counts,
    fleet positions, ownership). This updates immediately in the game loop.
2.  **Perceived State:** What the player (and AI factions) see, based on the
    time it takes for light to travel from an event to their **Homeworld**.

**The Golden Rule:**

> `Perceived Time = Current Time - (Distance to Observer's Homeworld / Speed of
> Light)`

*   **Visuals:** The player *only* sees the Perceived State.
*   **Commands:** Orders issued by the player travel from the Homeworld to the
    target at the speed of light. They only take effect once they "arrive" in
    the True State.

--------------------------------------------------------------------------------

## **II. Game Entities & Mechanics**

### **1. Suns (Stars)**

Suns are the production nodes.

*   **Attributes:** `id`, `pos` (x,y), `owner` (Faction ID), `level` (0-4),
    `units` (float), `upgradeTimer`.

*   **History Buffer:** To render the Perceived State correctly, every Sun must
    maintain a history of its state.

    *   *Implementation:* Store `{ time, units, owner, level, rate, upgrading }`
        in a circular buffer (size ~50).
    *   *Interpolation:* Find the checkpoint just before `Perceived Time` and
        simulate production forward to `Perceived Time`.

*   **Production Levels:**

    *   **Level 0:** Rate 0, Cap 50, Cost 0 (Neutral/Unoccupied).
    *   **Level 1:** Rate 0.4/sec, Cap 100, Cost 80.
    *   **Level 2:** Rate 1.0/sec, Cap 250, Cost 250.
    *   **Level 3:** Rate 2.4/sec, Cap 600, Cost 600.
    *   **Level 4:** Rate 6.0/sec, Cap 1500, Cost 0 (Max).
    *   **Neutral Rule:** Neutral Suns (Owner 0) do **not** produce units unless
        they are at least **Level 2**.

*   **Upgrading:**

    *   Takes **25 seconds**.
    *   Cost is deducted immediately.
    *   If the Sun is captured or hostile units arrive during upgrade, the
        upgrade timer is reset to 0 (cancelled).

### **2. Fleets**

Fleets are groups of units moving between Suns.

*   **Attributes:** `id`, `owner`, `sourcePos`, `targetSunId`, `units`,
    `launchTime`, `arrivalTime`, `targetPos`, `redirected` (boolean).

*   **Movement:** Linear interpolation from Source to Target.

*   **Visibility:** A fleet is only drawn if the current time falls within its
    "Perceived Visibility Window":

    *   Start: `Launch Time + (Distance(Source, Homeworld) / C)`
    *   End: `Arrival Time + (Distance(Target, Homeworld) / C)`
    *   *Grace Period:* Add a small buffer (e.g., 0.5s) to the end time to
        prevent flickering on arrival.

*   **Interpolation:** You must calculate the fleet's position at `Perceived
    Time`.

### **3. Commands**

*   **Attack:** Sends a fleet from a source Sun to a target Sun.

    *   **Delay:** The command signal travels from Homeworld to Source Sun. The
        fleet launches only when the signal arrives.

*   **Redirect:** Changes a fleet's destination mid-flight.

    *   **Delay:** Signal travels from Homeworld to the fleet's *future
        intercept position*.
    *   *Implementation:* To find the intercept point, iteratively approximate
        the time where `Distance(Homeworld, FleetPos(t)) / C + CurrentTime ==
        t`. Use 3 iterations for sufficient accuracy.
    *   *Execution:* When the command arrives at the intercept point:
        1.  The old fleet is marked as `redirected` (stops fighting/arriving).
        2.  A **NEW** fleet is created starting from that exact position,
            heading to the new target.
        3.  Visually, the old fleet "vanishes" (or stops) and the new one
            appears. In code, set the old fleet's `targetPos` to its current
            position and `arrivalTime` to `now` to effectively terminate it.

*   **Upgrade:** Signal travels from Homeworld to Sun. Upgrade starts on
    arrival.

### **4. Combat**

*   **Logic:** Happens in **True State** instantly when a fleet arrives at a
    hostile Sun.

*   **Resolution:** Simultaneous damage.

    *   Sort forces by size (descending).
    *   Subtract the size of the *smallest* force from *all* present forces.
    *   Repeat until only one faction remains.

*   **Visuals:** Combat explosions must be rendered at the **Perceived Time** of
    the battle. Store `battleEvents` (timestamps) on the Sun and keep them for
    at least 2 minutes to ensure they are seen even with max delay.

### **5. Scoring & Win Condition**

*   **Score:** Sum of units on owned Suns + units in active Fleets (excluding
    `redirected` fleets).
*   **Win:** Player is alive (score > 0) AND all AI factions are eliminated
    (score <= 0).
*   **Defeat:** Player is eliminated (score <= 0).

### **6. Map Generation**

*   **Distribution:** Procedural generation within `GALAXY_RADIUS`.

    *   *Density:* Higher density near the center. Rejection sampling:
        `threshold = (4 - 3*r/R)/4`.
    *   *Spacing:* Ensure `MIN_SUN_SPACING` between Suns.

*   **Sun Types:**

    *   20% Chance: Level 2, 60 Units.
    *   30% Chance: Level 1, 30 Units.
    *   50% Chance: Level 1, 15 Units.

*   **Homeworlds:**

    *   Placed at `GALAXY_RADIUS * HOMEWORLD_RIM_FACTOR`.
    *   Equally spaced angles around the galaxy.
    *   The nearest generated Sun is converted to a Homeworld (Level 1, 50
        Units, Owner set to Faction).

--------------------------------------------------------------------------------

## **III. Visuals & Rendering (PixiJS)**

The aesthetic is "Minimalist Space Ambient". Background is black (`#000000`).

### **1. Visual Time Clamping**

*   **Important:** When the game speed is very high (e.g., > 5x), visual effects
    like explosions and solar flares can become too fast to see.
*   **Implementation:** Calculate a `visualDt` for rendering effects.
    *   `visualDt = (GAME_SPEED > 5.0) ? dt * (5.0 / GAME_SPEED) : dt`
    *   Use this `visualDt` for updating explosion animations and solar flare
        lifecycles.

### **2. Sun Rendering**

*   **Glow:** Create a texture using a separate HTML Canvas with a
    `createRadialGradient` (white to transparent). Use `PIXI.Sprite` with
    `BLEND_MODES.ADD`.

*   **Core:** A `PIXI.Graphics` polygon. Use sine waves (noise) on the radius to
    make it wobble/animate over time.

    *   *Formula:* `radius + sin(theta*5 + time)*0.5 + sin(theta*11 -
        time*1.5)*0.25`

*   **Rings:** `PIXI.Graphics` rings that appear when upgrading (Yellow, pulsing
    alpha).

*   **Solar Flares:** For non-neutral suns, randomly spawn quadratic curves
    erupting from the surface. Animate their life/height using `visualDt`.

*   **Swarm:** Orbiting particles (`PIXI.Sprite`) representing units.

    *   *Count:* `Math.min(Math.floor(units / 2), 50)`.
    *   *LOD:* Hide individual particles if Zoom is low (e.g., < 0.3). Show only
        the number label.

*   **Label:** Text showing unit count (integer).

### **3. Fleet Rendering**

*   **Visual:** A swarm of particles trailing behind a center point.

    *   *Implementation:* Use a `PIXI.Container` and a manual particle pool
        (array) to avoid garbage collection.
    *   *Movement:* Particles should drift *opposite* to the fleet's movement
        vector to create a trail effect.
    *   *Count:* `Math.min(Math.floor(units), 80)`.

*   **Label:** Text showing fleet size.

*   **Color:** Matches the owner's faction color.

### **4. UI Overlays (PixiJS)**

*   **Selection:** Draw rings around selected Suns/Fleets.
*   **Command Lines:** When a command is issued, draw a line/dot traveling from
    the Homeworld to the target to visualize the signal delay.
*   **Explosions:** `PIXI.Graphics` circle that expands and fades out (shockwave
    effect). Use `visualDt` for the animation.

### **5. Minimap (Canvas 2D)**

*   Render a simplified view of the galaxy in the bottom-right corner.
*   Show the camera viewport rectangle.
*   Allow clicking/dragging on the minimap to move the camera.
*   Label below minimap: "PERCEIVED REALITY".

--------------------------------------------------------------------------------

## **IV. User Interface (HTML/Tailwind)**

Use a dark theme (`bg-gray-900`, `text-blue-400`, `border-blue-900`) for all UI
panels.

### **1. Title Screen**

*   Title: "LIGHT SPEED"
*   Description of the mechanics.
*   **Controls List:** WASD/Pan, Scroll/Zoom, Drag Select, etc.
*   **Settings Sliders:**

    *   **Stars:** Range 0-100. Formula: `Math.round(20 + (val/100)^2 * 980)`.
    *   **Opponents:** Range 1-20.
    *   **Game Speed:** Range 0-100. Formula: `0.1 * 500^(val/100)`.

*   **Start Button.**

### **2. HUD**

*   **Leaderboard (Top-Left):** List factions by score. Update every 0.5s. Show
    "eliminated" status if score < 1.
*   **Help Panel (Toggle 'H'):** Quick reference for controls.
*   **Speed Indicator:** Fades in/out when speed changes.

### **3. Pause / Game Over Screens**

*   **Score Graph:** A `<canvas>` chart showing unit counts over time for all
    factions.

    *   *Data Collection:* Record scores every **1.0 second** in the game
        engine.
    *   *Drawing:* Draw axes, time labels (X), score labels (Y).
    *   *Interaction:* Implement mouse hover detection. Find the closest data
        point to the mouse cursor. Draw a circle at that point and a tooltip box
        showing "Faction: Score".
    *   *Colors:* Player line width 4, AI line width 2.
    *   *Implementation:* Use `mousemove` and `mouseleave` event listeners on
        the canvas to track `graphHoverData` and trigger redraws.

*   **Buttons:**

    *   **Pause:** Resume, Restart, Main Menu.
    *   **Game Over:** Play Again, Continue Observing (if lost), Main Menu.

### **4. Debug Menu (Shift+D)**

*   Buttons: "Force Win", "Force Lose".

--------------------------------------------------------------------------------

## **V. Controls & Input**

*   **Mouse:**

    *   **Left Click:** Select Sun. (Threshold: Drag distance < 5px).
    *   **Left Drag:** Box Select Suns.
    *   **Double Click:** Attack Target with **100%** of units from selected
        Suns. (Implement by checking `Date.now() - lastClickTime < 300` in a
        `pointertap` or `pointerup` handler).
    *   **Right Click:**
        *   On Empty Space: Pan Camera (if dragged).
        *   On Sun (with selection): Attack (send **50%** units) or Redirect
            Fleets.
    *   **Shift + Click:** Add/Remove from selection.
    *   **Scroll:** Zoom In/Out (centered on mouse).

*   **Keyboard:**

    *   **WASD / Arrow Keys:** Pan Camera.
    *   **U:** Upgrade selected Suns.
    *   **F:** Toggle Fullscreen. (Implement edge panning: if mouse is within
        15px of edge, pan camera).
    *   **H:** Toggle Help.
    *   **L:** Toggle Leaderboard.
    *   **M:** Toggle Pause Menu.
    *   **+/-:** Adjust Game Speed (Support Numpad keys).

*   **Events:** Use `pointerdown`, `pointermove`, `pointerup` on the Pixi stage
    for unified touch/mouse handling. Prevent default context menu.

--------------------------------------------------------------------------------

## **VI. AI Logic**

The AI operates in the **True State** but uses **Perceived State** for decision
making.

*   **Update Interval:** Randomly every 3-7 seconds.

*   **Upgrade:** If it has a safe Sun with `Units > 1.5 * Cost` and `Level < 3`,
    upgrade.

*   **Attack:** If a Sun has `> 30` units:

    *   **Target Scoring:**
        *   Base Score: `-Distance` (closer is better).
        *   Bonus: `+500` if target is Neutral.
        *   Penalty: `-TargetUnits * 10`.
    *   **Attack Condition:** `SourceUnits > TargetUnits * 1.2 + 10`.
    *   **Action:** Attack best target with **60%** of units.

--------------------------------------------------------------------------------

## **VII. Constants & Configuration**

*   `GALAXY_WIDTH/HEIGHT`: 2000
*   `GALAXY_RADIUS`: 950
*   `LIGHT_SPEED_C`: 60
*   `UNIT_SPEED`: 20
*   `ZOOM_START`: 1.5
*   `MIN_ZOOM`: 0.5
*   `MAX_ZOOM`: 10.0
*   `LOD_ZOOM_THRESHOLD`: 2.5
*   `UPGRADE_DURATION`: 25.0
*   `MIN_SUN_SPACING`: 80
*   `HOMEWORLD_RIM_FACTOR`: 0.9
*   `COLORS`:

    *   Neutral: `0xffffff`
    *   Player: `0x3b82f6` (Blue-500)
    *   AI Colors (Fixed): `[0xef4444, 0x22c55e, 0xa855f7, 0xf97316, 0xeab308,
        0x06b6d4, 0xec4899, 0x6366f1, 0x14b8a6, 0xf43f5e, 0x84cc16, 0x8b5cf6]`
    *   *Fallback:* For >12 AIs, generate colors using HSL (Hue = index * 137.5
        deg). Implement a helper `hslToHex` function for this.

--------------------------------------------------------------------------------

## **VIII. Implementation Strategy**

1.  **Class Structure:**

    *   `Sun`, `Fleet`, `Command` (Data classes).
    *   `GameEngine` (Logic, True State, History).
    *   `GameRenderer` (PixiJS, Perceived State).
    *   `AIController` (One per AI faction).

2.  **Loop:** Use `requestAnimationFrame`.

    *   Calculate `realDt` (wall clock time) for UI/Camera updates.
    *   Calculate `dt = realDt * CONSTANTS.GAME_SPEED` for game logic updates.

3.  **Performance:**

    *   Use manual object pooling for fleet particles (array of sprites) to
        avoid garbage collection.
    *   Implement LOD for Sun labels/particles based on zoom.

4.  **State Management:**

    *   `Sun.history` is critical. Push checkpoints on significant events
        (production, battle, upgrade).
    *   `getPerceivedState(observerPos, time)` method on Sun class is the core
        of the "Light Speed" mechanic.

**Output:** Provide the complete, runnable HTML file code.
