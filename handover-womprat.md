# Womprat - Design Handover & Blueprint

A reference for building a new game (e.g. the **AT-ST game**) in the same mould as Womprat.
Womprat is a 3D wireframe "blueprint" Death Star trench-run shooter. Follow the conventions below
so the new game shares its look, feel, and tech.

---

## 1. What Womprat is

- Single self-contained `.html` file. No build step, no bundler, no dependencies to install.
- 3D wireframe **blueprint aesthetic**: deep navy background, white/cyan line art, a few hot accent colours.
- On-rails-ish shooter: you fly an X-wing down an endless trench, dodge/shoot turrets and TIEs,
  fight bosses, chase a high score.
- Synthesised audio only (no copyrighted samples).
- Deployed as one file to a live domain; version-controlled on GitHub.

**Source of truth:** `/Users/john/Desktop/deathstar-trench.html`
**Deploy copy + repo:** `/Users/john/Desktop/womprat/` (`index.html` is a copy of the source; `deploy.sh` ships it).

---

## 2. Tech stack

- **Three.js 0.160.0**, loaded from unpkg via an ES-module **importmap** (no npm):
  ```html
  <script type="importmap">{ "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/" } }</script>
  <script type="module"> ... game ... </script>
  ```
- `WebGLRenderer` + `EffectComposer` / `RenderPass` / a custom `ShaderPass` (palette pass).
- Plain ES modules. Everything lives in one `<script type="module">`. No framework.
- **Important:** because it's a module, top-level vars are NOT on `window`. For debugging in a preview,
  temporarily attach `window.__dbg = {...}` and strip it before shipping.

---

## 3. Visual identity (keep this exact)

- **Background:** `#0b2545` (also `palette[0]` = rgb 7,24,47) so empty space maps to itself.
- **Line work:** pure white (`0xffffff`) "ink" via `LineBasicMaterial`. Cyan/blue tints for depth.
- **18-colour palette quantization.** A final `ShaderPass` snaps every rendered pixel to the nearest
  of 18 authored colours: 12 blues, 2 reds/oranges (player lasers, explosions), 2 greens
  (Imperial bolts), 2 whites. This is what gives the cohesive "plotter blueprint" look.
- **CRITICAL colour-management setup** (without this the palette renders washed-out):
  ```js
  THREE.ColorManagement.enabled = false;
  renderer.outputColorSpace = THREE.LinearSRGBColorSpace;   // no extra gamma encode
  // palette ShaderPass is the LAST pass -> straight to screen, no OutputPass
  ```
- **No em dashes** in any visible copy - use a spaced hyphen ` - `. **No emojis** unless asked.

### Wireframe-with-fill rendering

Pure wireframe lets you see through walls. Two helpers solve this:

```js
function edges(geom, mat = wireMat) {   // solid dark fill (occludes) + white wireframe on top
  const g = new THREE.Group();
  g.add(new THREE.Mesh(geom, fillMat));                       // fillMat = dark-blue MeshLambert
  g.add(new THREE.LineSegments(new THREE.EdgesGeometry(geom), mat));
  return g;
}
function wire(geom, mat = wireMat) {     // wireframe only - for decals / far / open shapes
  return new THREE.LineSegments(new THREE.EdgesGeometry(geom), mat);
}
```
- Use `edges()` for anything that should block what's behind it (hull, walls, turrets).
- Use `wire()` for sky objects, etched detail, open rings.
- **Glows / solid accents** (engine cores, eyes, pickups) = `MeshBasicMaterial` (flat colour, ignores
  light, gets palette-quantized). Additive `MeshBasicMaterial` discs make soft halos.

### Draw-call optimisation

`optimizeStatic(group)` merges all the `fillMat` meshes and all the wireframe `LineSegments` in a group
into one of each (via `BufferGeometryUtils.mergeGeometries`). Cuts a model from dozens of draw calls to
~2. Call it on static models (turrets, scenery). Add animated/coloured children (eyes, flames, glows)
**after** the merge so they keep their own material.

### lookAt gotcha (cost me a bug)

`THREE.Object3D.lookAt()` aims the object's **+Z** axis at the target (NOT -Z like a camera). So build
any model that uses `lookAt` with its **nose / front / gun-barrels at +Z**, or it flies/aims backwards.

---

## 4. Audio (synthesised only - no samples)

- One `AudioContext`, a `masterGain` (~0.45) -> destination. Unlock on first real keypress
  (`initAudio(); resumeAudio();`). `M` toggles mute by zeroing `masterGain`.
- **SFX** = short oscillator + gain-envelope functions (blaster = downward pitch sweep, explosion =
  filtered noise burst + sub, R2 = bandpassed warble, etc.). Pattern: build osc/noise -> filter ->
  gain envelope -> masterGain, `start`/`stop`.
- **Engine** = looping noise through an LFO-modulated low-pass + a quiet sub (a *moving* bed, not a
  static tone - static drones get annoying fast). Opens up (brighter/louder) on boost.
- **Music bed** = a lookahead step-scheduler called each frame (`updateMusic()`), scheduling short
  notes ahead of `actx.currentTime` on a dedicated bus. Slow ominous minor pulse; faster/tenser during
  bosses. Keep it quiet - it's a bed.
- Copyright: **never** use real Star Wars audio. Synthesise everything.
- Preview limitation: synthetic key events don't unlock browser audio, so you **cannot verify audio by
  ear in the headless preview** - confirm it loads without error, then ask the user to listen.

---

## 5. Game architecture

- A single `state` object holds everything (score, shields, dist, alive, started, all the `next*Dist`
  spawn gates, boss flags, combo, camShake, timers...).
- `update(dt)` runs the sim; gated `if (!state.started || !state.alive) return;`.
- `loop()` = `const dt = Math.min(clock.getDelta(), 0.05); update(dt); ...background... ; composer.render();
  requestAnimationFrame(loop);`. **dt is clamped** so a throttled/backgrounded tab can't fast-forward timers.
- **World scroll model:** the player ship stays near a fixed Z; the *world* (trench segments, turrets,
  pickups) scrolls toward the camera by `scroll = forward * 0.25`. Recycle trench segments when they pass.
- **Distance-gated spawners:** `if (state.dist >= state.nextXDist) { spawnX(); state.nextXDist = state.dist + gap; }`.
- **HUD** is plain HTML/CSS overlaid on the canvas, refreshed by `setHUD()` each frame.
- `restart()` removes every pooled object (loop over the arrays), resets `state`, re-seeds.
- **Verify every UI/visual change with a screenshot** (preview MCP). The preview's canvas sometimes
  loads at the wrong size - dispatch a `resize` event to force a full-frame render.

### Boss pattern (reuse for AT-ST set-pieces)

Two boss flavours, both: set an `xActive`/`xDone` flag pair, **ease the world to a crawl** (`state.curSpeed`
lerps to ~0.12-0.2), pause normal spawners, show a boss HUD bar + a `bannerFlash()` title, then resume +
award a bonus.
- **HP boss** (mega turrets): take N shots; EMP does chunk damage; die at 0 HP.
- **Timed boss** (Boba Fett): survive a 45s countdown bar; hits shave the timer; he leaves at 0.
- A roaming boss gets a **focus halo** + a screen **vignette** (`#bossdim`) so it never gets lost in the
  wireframe.

### Reusable systems already built (lift these wholesale)

scoring + **combo multiplier** (`scoreKill(base)`, x1-x5, resets on hit) · shields/health bar ·
EMP screen-clear super · **pickups** (shield repair + spare EMP, collectible icons) · **camera shake**
(`addShake()`, rotation-only, decays) · `bannerFlash()` centre announcements · `#bossdim` vignette ·
**localStorage leaderboard** (key `womprat_lb`, name entry, default name **LUKE**) · cameo fly-bys ·
background dogfight. Off-screen threat arrows were considered but not built.

---

## 6. Deploy & repo

- The whole game is **one HTML file**. Deploy = copy source -> `index.html` -> SFTP upload.
- `deploy.sh` (gitignored) does it in one command via an **SSH key** (`~/.ssh/<key>`, chmod 600).
- **Never commit secrets** - keep `deploy.sh` and any key out of git (`.gitignore` them). Passwords that
  get pasted in chat should be rotated.
- Commit cadence: batch ~4 visible changes per commit; can deploy more often than you commit.
- GitHub repo holds `index.html` + `deploy.sh` (ignored) + this handover. Womprat lives at
  github.com/runphp-john/womprat and deploys to womprat.net (20i / StackCP).

**For the AT-ST game:** stand up an analogous repo + domain, copy `deploy.sh` and point it at the new
target, keep the single-file structure.

---

## 7. Building the AT-ST game in this mould

Keep: single-file Three.js, blueprint palette + colour-management setup, `edges()`/`wire()`/`fillMat`/
`optimizeStatic`, synth audio, the `state`/`update`/`loop` architecture, HUD/leaderboard/shields/combo/
pickups/shake/boss patterns, deploy flow.

Change for an AT-ST (Imperial walker) game - decide these up front:
- **Viewpoint & movement.** AT-ST is a ground walker, not a flier. Options: cockpit/gunner view over a
  scrolling battlefield (Hoth snowfield / Endor forest), or 3rd-person behind the walker. Likely a
  ground-scroll instead of a trench; add a walker "stomp" bob to the camera.
- **Hero model.** Chicken-walker: a boxy command head with two chin guns + the side "ears", on two
  angled legs. Build it with `edges()`; barrels at +Z; add a glowing red/green viewport accent.
- **Targets/threats.** Snowspeeders (with tow cables?), rebel infantry, turrets, trees/AT-ATs as scenery
  or bosses. Reuse the spawner + boss machinery.
- **Terrain.** Swap the trench walls for ground + parallax horizon/treeline; keep wireframe + fill so it
  still reads as blueprint. Snow = lighter palette tints; forest = more greens (already in palette).
- **Set-pieces.** An AT-AT boss (HP), or a timed "hold the line" wave (timed). Reuse vignette/banners.

Pitfalls to remember: `lookAt` aims +Z; palette quantises ALL colours (author to the 18); disable colour
management or it washes out; verify visuals with screenshots; keep engine/ambient audio *moving* not droning.
