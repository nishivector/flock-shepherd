# Flock Shepherd — Design Document
Round 14

---

## IDENTITY

**Game name:** Flock Shepherd
**Tagline:** *Stillness is the only weapon that matters.*
**What is the player:** A lone electromagnetic pulse node — a faint blue-white ring, 14px diameter, drifting through an open atmospheric sky. You have no body. You are a gravity well.

**World feel:** You are suspended high above a cloudscape seen from directly above, as if looking down from orbit onto a slow-moving Ghibli atmosphere — warm thermals, vast silence, noon light filtered through distant haze. The world is not hostile. It is indifferent. The predator clouds don't hunt *you* — they hunt the flock. You are not in danger. Your flock is.

**Emotional experience:** *Held breath.*

**Reference games:**
- **R-Type** — You read cloud formations the way a pilot reads enemy squadrons; you commit before the window opens, not after it closes. Timing is pre-cognitive.
- **Metal Slug** — All threat is spatially legible at a glance. A tightening predator spiral is never ambiguous. You see it, you feel it, you respond. No abstraction.
- **Flower (thatgamecompany)** — Movement creates beauty; restraint creates power. The flock is alive in your hands without being yours to command.

---

## VISUAL SPEC

**Background color:** `#4A7FA5` — noon-sky teal, broad atmospheric field
**Primary color (flock):** `#F0C842` — gold-amber, warm sunlit glow
**Secondary color (player node):** `#D0E8FF` — pale ice-blue-white
**Accent color (predator cloud):** `#E05535` — ember-red, deep orange-red edge
**Barrier color:** `#2A5A78` — darker teal, 30% opacity fill + solid `#1A3A50` outline

**Bloom:** Yes. Strength: 0.9. Threshold: 0.55. Applied to flock particles and player node only. Predator clouds have NO bloom — they are matte, absorptive. The contrast between glowing flock and flat predator is the primary visual language.

**Vignette:** Yes. Soft radial, 20% opacity darkening at edges only. Never intrudes on center play area.

**Camera:** Top-down orthographic. Fixed. Canvas is 900×900px logical (adapts to viewport). No scrolling. The world is exactly one screen. Distance feel achieved by particle scale (flock particles are 4–6px, player node is 14px ring, predators are 3–5px).

**Player silhouette:** Thin pulsing ice-white ring
(14px diameter, 1.5px stroke, pulses from 0.7 to 1.0 opacity at 1.2s period, inner fill transparent)

---

## SOUND SPEC

**Music:** A slow, dry sub-bass heartbeat — single sine wave at 55Hz, 72 BPM, no reverb. Over it: a shimmering texture of overlapping high-pitched sine tones (800–2400Hz range) that slowly drift in pitch like distant birdsong or wind chimes in thermal air. Four to six sine oscillators detuned ±8 cents from each other, fading in and out with 3–8 second envelopes. Total dynamic range: quiet. This is not a hype track.

**Music BPM:** 72

**Music responds to:**
- Flock cohesion level (0–100%): as flock compresses toward comet, high shimmer tones resolve toward a single pitch (from scattered to unison over 4 seconds). At comet release, the shimmer locks to one pure tone for 1.5 seconds.
- Predator proximity: sub-bass pulse doubles to 144 BPM when any predator is within 120px of flock centroid.
- Player stillness: after 2 seconds of stillness, a slow low pad fades in at -18dB, growing to -10dB at 4 seconds (comet charge). This is the only "musical" moment.

**Sound effects (6):**

1. **Flock particle death** — single short sine burst, 400Hz → 200Hz in 80ms, -20dB, reverb tail 0.3s.
   `new Tone.Synth({ oscillator: { type: "sine" }, envelope: { attack: 0.001, decay: 0.08, sustain: 0, release: 0.1 } }).toDestination()`
   Trigger at `"F4"` for 0.08s. Volume: -20dB.

2. **Comet charge build** — rising pad sweep, D3 → D4 over 4000ms, volume -18dB → -8dB.
   `new Tone.Synth({ oscillator: { type: "triangle" }, envelope: { attack: 4.0, decay: 0, sustain: 1.0, release: 0.3 } }).toDestination()`
   Triggered at stillness onset, released at comet fire.

3. **Comet release** — deep resonant whoosh-burst. Bass thud (80Hz sine, attack 0.001, decay 0.4) + high shimmer sweep (1200Hz → 3000Hz NoiseSynth, 0.2s). -6dB total.
   `new Tone.NoiseSynth({ noise: { type: "pink" }, envelope: { attack: 0.005, decay: 0.2, sustain: 0, release: 0.1 } }).connect(new Tone.Filter(1200, "highpass")).toDestination()`

4. **Predator approach warning** — low pulse at 55Hz, amplitude modulated at 4Hz as predator enters 150px radius. Tone.js: `new Tone.LFO(4, 0, 0.3).connect(synth.volume)`. Fades in over 500ms.

5. **Goal zone reached** — 3-note rising arpeggio (D4, F#4, A4), each 100ms, bell-like MetalSynth. Full flock arrival: full 3 notes. Partial: first note only.
   `new Tone.MetalSynth({ frequency: 293.66, envelope: { attack: 0.001, decay: 0.3, sustain: 0, release: 0.2 }, harmonicity: 5.1, modulationIndex: 32, resonance: 4000 }).toDestination()`

6. **Comet hits predator cloud** — crack-scatter sound: short MetalSynth burst + noise burst.
   `new Tone.MetalSynth({ frequency: 600, envelope: { attack: 0.001, decay: 0.12, sustain: 0, release: 0.05 } }).toDestination()` triggered at predator scatter.

---

## MECHANIC SPEC

**Core loop:** Move your node to influence 80 flock particles through proximity, hold still to compress them into a comet that pierces barriers and scatters predator clouds, then guide the flock through the goal zone to advance.

**Primary input:**
- `pointermove`: Player node tracks pointer/touch position. Smooth lerp, factor 0.15 per frame. If pointer is off-screen, node drifts toward center at 30px/sec.
- `pointerdown`: No special action — holding still is done by not moving, not by pressing.
- `pointerup`: No special action.
- No buttons needed. This is a pure pointer/touch follow game. Works on mobile by touching and dragging.

**Key timing values:**

*Flock physics:*
- Particle count per level: 80 (start) — never increases, only decreases when predators eat them
- Flock inertia: velocity lerp half-life 2000ms (each frame: `vel = lerp(vel, targetVel, 1 - 0.9997^deltaMs)`)
- Player speed thresholds (measured in px/sec based on node's velocity):
  - **Attract** (< 8 px/sec): particles within 220px radius drift toward node at 35px/sec
  - **Orbit** (8–45 px/sec): particles orbit node tangentially at 28px/sec at 90px radius, scaling with distance
  - **Scatter** (> 45 px/sec): particles flee away from node at 55px/sec
- Influence radius: 220px (attract), 180px (orbit), 260px (scatter)
- Cohesion meter: 0–100% = what % of flock is within 60px of node when still. Displayed as small arc around node.

*Comet system:*
- Stillness threshold: player speed < 5px/sec for 4000ms continuously
- Visual: thin arc around node grows from 0 to full circle over 4000ms while charging
- At 4000ms: all particles within 300px snap into formation (tight 20px radius cluster) directly behind node
- Comet fires in last non-still movement direction (or straight up if never moved)
- Comet speed: 420px/sec
- Comet duration: 1600ms or until it exits the canvas
- Comet radius: 18px (for collision detection with barriers and predators)
- After comet fires: particles released at comet's current position, resume normal flock physics

*Predator clouds:*
- Each cloud: 12–20 small ember-red discs (3–5px radius each), rotating in a loose spiral
- Spiral rotation: 35°/sec
- Hunt speed: cloud centroid moves toward flock centroid at 22px/sec
- Kill: if predator disc overlaps flock particle (within 8px), flock particle removed
- Scattered by comet: any predator within 80px of comet path gets launched away at 180px/sec for 2000ms, then returns

**Win condition:** 30 or more flock particles reach the goal zone (glowing `#F0C842` ring, 50px radius) simultaneously. All remaining particles "land" with a ripple. Level advances.

**Lose condition:** Total flock count drops below 30 at any time. Game over screen appears.

**Score:** `score += particles_delivered × level_multiplier × combo_multiplier`
- `combo_multiplier` = how many levels completed without hitting the minimum threshold (bonus for keeping flock fat)

---

## LEVEL DESIGN

### Level 1 — Open Sky
**What's new:** Tutorial. Just the flock and a goal zone. No predators. No barriers.
**Parameters:** 80 particles, 1 predator cloud (disabled), goal zone center-right, player starts center-left
**Goal:** Guide 30+ particles to goal zone
**Duration:** ~30–60 seconds
**Teaching:** Discovers attract/scatter mechanic naturally. First stillness-comet charge is optional but rewarded.

### Level 2 — The Circling
**What's new:** 1 predator cloud introduced, orbiting the play field at the edge. It slowly spirals inward.
**Parameters:** 80 particles, 1 predator (15 discs, hunt speed 18px/sec), spiral starts at 400px from center
**Goal:** 30+ particles to goal zone before predator eats you below 30
**Duration:** ~60–90 seconds
**Teaching:** Predator threat. First time comet is necessary to scatter the predator.

### Level 3 — The Narrows
**What's new:** 2 rectangular barriers across the screen (not touching edges — 80px gaps on each side). Goal zone is on the far side.
**Parameters:** 80 particles, 1 predator (18 discs, 20px/sec), 2 barriers (60px wide, 180px tall)
**Goal:** 30+ particles through barrier gaps OR via comet punch-through
**Teaching:** Comet as barrier-breaker is introduced here. First wow moment.

### Level 4 — Double Spiral
**What's new:** 2 predator clouds simultaneously. They orbit each other as they hunt.
**Parameters:** 80 particles, 2 predators (15 discs each, 22px/sec), 1 barrier
**Goal:** 30+ particles to goal
**Duration:** ~90–120 seconds
**Teaching:** Comet choice — scatter left cloud or right cloud first? The skill ceiling starts here.

### Level 5 — The Gauntlet
**What's new:** 2 predators + 3 barriers arranged in a needle corridor. Goal zone is at far end.
**Parameters:** 80 particles, 2 predators (20 discs each, 25px/sec), corridor is 90px wide
**Goal:** 30+ particles delivered
**Teaching:** The full mastery test. Comet must be used as slingshot through the corridor. Double spiral predator management and corridor threading simultaneously.

---

## THE MOMENT

Level 3: The player discovers the barrier for the first time. Their flock hits it and scatters back. They try to guide particles through the 80px gap — it works, but slowly. Then a predator cloud is closing in from the left. In desperation they hold still. The comet charges. They fire it directly at the barrier. The comet punches through, particles scatter to the far side, and the predator cloud gets grazed — half its discs flee. The player just cleared the level in one move they didn't plan. They will try to replicate this on purpose for the rest of the game.

---

## EMOTIONAL ARC

**First 30 seconds:** The flock moves like smoke. You chase it, it scatters. You stop, and it drifts back toward you. It feels alien and alive. You don't know if you're doing it right. Then the comet charge ring appears around your node — and you realize: this game rewards stillness. That's the inversion.

**After 2 minutes:** You've found the groove. You're gliding slowly, the flock orbiting like a small galaxy. You can feel the 2-second inertia — you're no longer responding to where the flock is, you're responding to where it *will* be. Predators feel like a puzzle, not a threat.

**Near the win (Level 5):** You have 35 particles left. Two predator clouds are tightening from opposite sides. The corridor is 90px wide. You position yourself at the corridor entrance, hold still. The ring charges. Four seconds. The flock compresses behind you like a golden teardrop. You fire. The comet threads the needle, both predators scatter, and the particles pour through. You delivered 31.

---

## THIS GAME'S IDENTITY IN ONE LINE

"This is the game where you win by doing nothing — and learning to do nothing well."

---

## START SCREEN

**Idle animation (Three.js canvas):**
- 80 gold flock particles slowly drifting in lazy orbits at 15px/sec around a fixed node at canvas center
- Node pulses at 1.2s period (0.7→1.0 opacity, no movement)
- One loose predator cloud (12 discs) spirals lazily at screen edge — not threatening, just atmospheric
- No barriers, no goal zone — pure flock beauty
- Every ~8 seconds, node "holds still" for 4 seconds: comet ring charges and fires to the right, flock sweeps through, then disperses and reforms
- Particle glow cycles: #F0C842 at full saturation, ambient bloom 0.9

**SVG overlay spec (Options A + B):**

**Option A — SVG title:**
```
Text: "FLOCK SHEPHERD"
Font: monospace, bold, 36px
Fill: #F0C842 (gold)
Filter: feGaussianBlur stdDeviation=6, feComposite over
Letter spacing: 0.15em
Animation: fadeIn + letter-spacing 0.3em → 0.15em over 1.2s
```

**Option B — SVG iconic silhouette (flock + node):**
```
A central small circle (node, 8px radius, stroke #D0E8FF, no fill)
Surrounded by 7 tiny circles (flock particles, 3px radius, stroke #F0C842, no fill)
arranged in a rough orbit pattern at 30px radius from center
One particle at 0°, 50°, 100°, 155°, 210°, 265°, 320°
Total SVG: 80×80px, centered above title
Animation: iconPulse 3s ease-in-out infinite — the 7 particles
slowly orbit the center node at 20°/sec, creating a slow rotation
```

Position: Option B icon above Option A title, stacked vertically.
Below title: "Tap to begin" in monospace 12px, #D0E8FF, blinking at 1s.
