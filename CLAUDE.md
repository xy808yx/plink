# CLAUDE.md — Plink

Context for future sessions working on this repo.

## What Plink is

A single self-contained `index.html`: an interactive Galton board that teaches probability and,
crucially, **fat tails**. "Plink" is the kid-facing name (the sound the marbles make). The
fat-tails depth lives behind a **Lab** toggle. The teaching arc is: a *fair* board is a Gaussian
factory; break independence or the fixed step size and the bell dies (Mediocristan to Extremistan).

Since the Learn push there are two top-level modes behind one `Learn | Play` seg at the top of
the dock: **Play** is the untouched sandbox; **Learn** is a six-chapter seaside story (waves =
bell, giant wave = scary tail, treasure ashore = lucky tail) driven by a thin controller over the
same engine. Sigma is taught exactly once, in chapter 2, as "steps from home".

Credits the project leans on: **Galton** (the 1877 quincunx), **Mandelbrot** (heavy tails,
Mediocristan/Extremistan), **Taleb** (fat tails as a worldview). See README, which leads with the
"break the machine to reveal Extremistan" story.

## Hard constraints (do not violate)

- **One file.** Dependency-free vanilla JS + Canvas, single `index.html`. If you ever must bundle,
  use a tiny esbuild step that still emits a single `index.html`. The allowed exceptions are static
  sidecars (assets, no code): `manifest.webmanifest`, `icon.svg` (now the **aurora ribbon over the
  Mokulua islets** — the fat-tailed curve reborn in the night world; vector source of truth),
  `icon-192.png` / `icon-512.png` (any + maskable), `apple-touch-icon.png` (180, iOS home screen),
  and the OPTIONAL **`theme.mp3`** background track (J supplies it; the app is silent and fully
  functional without it). Regenerate PNGs from `icon.svg` with `qlmanage -t -s 1024 -o <dir>
  icon.svg` then `sips -z <n> <n>`. Head wires favicon (SVG + PNG fallback), apple-touch-icon,
  manifest, and `theme-color` (now `#0A1220` night). App CODE stays single-file.
- **Preserve the glass-marble sprite renderer** (`makeMarble` / `sprite` / `drawMarble`,
  pre-rendered and blitted). Same pattern now also covers the aurora background (`makeAuroraBlobs`
  / `drawAurora`) and the peg glow (`pegSpriteFor`): pre-render once, blit every frame, never
  build gradients per frame in the hot loop.
- **Preserve the layered-depth UX**: kids get the simple toy; the Lab row (`#labrow`) is hidden
  until Lab mode reveals the levers.
- **Preserve the "one teaching line in the insight strip" pattern** (`setInsight`, the `#insight`
  element). Every lever change says one plain sentence there.

## Architecture (the seams, marked in comments)

The whole app is one IIFE in `index.html`. Grep for these markers:

- `=== CONFIG ===` — `S`, the shared view state plus the "rig" (bias / momentum / wild).
- `=== BOARD ===` — `makeBoard()`. Per-machine state (histogram, in-flight balls, effects,
  geometry, and its own copy of the rig). `boards` holds one board normally, two in compare mode.
- `=== BOUNCE RULE ===` — inside `spawnBall(board, o)`. The single source of truth for a ball's
  path. All four levers act here or on the resulting landing slot.
- `=== BIN DOMAIN ===` — histogram is a sparse `Map<slot, count>` over a fixed domain. When Wild is
  live the domain pads wider than the peg spread (`domainPad`) so outliers show instead of clipping.
  `binX(board, slot)` maps a slot to x; `displayCounts` folds off-domain outliers onto the edge
  columns for drawing while stats still use the true slot.
- `=== TAIL METRIC ===` — `boardStats()`. Mass beyond 3σ of the *naive* bell (mean `n*bias`, sd
  `sqrt(n*bias*(1-bias))`). Recentres with bias so pure bias reads ~0.3% (Gaussian), not a fake tail.
- `=== ARCHITECTURE ===` — closing note on the Board refactor and the shared x-scale.
- `LEARN (Push 2)` — `setLever(key,val)` (the ONLY sanctioned way to script a lever: keeps `S`,
  the slider value, the `--fill` paint, and `applyRig`/`buildBoards` in sync), the `CHAPTERS`
  array (copy + `setup`/`demo`/`tinkerSetup`/`begin`/`ready` per chapter), and the `learn`
  controller (`active/idx/phase/doneNow/completed/moved`).

## The world (the island rebuild — "Push A")

The old black-stage/aurora-blob look was fully replaced (J: "still ugly as hell") with a zen,
drawaurora.com-style island day/night dreamscape. The real-world referent is a specific Hawaiian
beach and its twin islets (the Mokes), but that **place name is deliberately never surfaced
anywhere** (J: "it should just be known") — copy, comments, and docs say "the islands" / "the
Mokes", not the town. Spec:
`~/.claude/plans/still-ugly-as-hell-dapper-comet.md`. Adapted from a 3-way scene-engine competition
(Scene B won: mirror-reflected sea + aurora). Key seams:

- **`Scene` IIFE** (grep `==================== SCENE`): a full-bleed fixed background canvas `#scene`
  (`z-index:0`) painting a keyframe day/night sky, celestial arc (sun by day, moon by night), the
  **two twin islands (the Mokes)** — Moku Nui larger-left, Moku Iki right — rendered by
  `islePoints`/`isleColor`/`fillIsle`/`rimIsle` in `drawLand` as CLEAR dark silhouettes (island
  colour is a darkened tint of the sky right above the sea, so they read against any hour) with a
  lit crest rim, alone on an open horizon (no ridges behind them); and a **mirror-reflected sea** (samples `scv` each
  paint, flips it in banded, plus a moonglitter path + night aurora-reflection columns). The old
  line-drawn palm frame was DELETED (read as a "spiderweb" — first thing to go). `KEYS[]` is the phase color table
  (Night/Dawn/Sunrise/Day/Golden/Dusk/Night). Public API: `resize/frame/begin/easeTo/release/
  setTime/addRipple`. Pre-render-once/blit pattern like the marble sprites; the reflection band loop
  + shimmer are the only per-paint heavy work.
- **The clock.** The app owns wall time. `Scene.frame(now,dt)` (called from the board's `frame()`)
  advances `sceneT` by `dt/DAY_LEN` (DAY_LEN=360s) only when `running && started && !document.hidden`,
  and **self-caps paint to ~30fps** (`lastPaint`) while the clock advances every call. `begin()` runs
  on the veil tap (Play drifts; Learn waits for its chapter). **Learn borrows the sky**: `gotoChapter`
  calls `Scene.easeTo(CHAPTER_SKY[idx], 4)` (shortest-direction ease, holds at the hour); `exitLearn`
  calls `Scene.release()` to resume the free-running day. `reduce` freezes at a golden still; `S.lite`
  (set by the board's ftAvg) drops the reflection/shimmer. **resize() clamps W/H to >=1** so a 0-size
  viewport never builds a 0-dim sprite (drawImage throws on those — real bug caught in review).
- **The board canvas is transparent now.** `draw()` paints ONLY pegs/marbles/histogram/bell; the
  frosted-glass `.cabinet` (`backdrop-filter:blur`) + the Scene behind it are the backdrop. Old
  aurora blobs / dust / vignette / body gradient are GONE. Marble rare-edge `CYAN`/`CYANS` `let`s
  keep their names but were repointed to **aurora green** (`7FE8A9`/`C9F5DE`); warm middle ramp
  (`HONEY`/`AMBER`/`AMBERD`) also warmed. The bell curve is now an **always-on faint ghost** (the
  Bell toggle just brightens it via `gk`).
- **Layering / pointer-events.** `.app{pointer-events:none}` with `.topbar/.cabinet/.insight/.dock/
  .veil` re-enabled to `auto`, so taps in the grid gaps fall through to `#scene` → `Scene.addRipple`
  (a quiet one-shot shimmer; `reduce` skips). The `.topbar` has a text-shadow so it stays legible on
  a bright day sky (it sits on the scene with no glass).
- **Veil + audio.** `#veil` (tap-to-begin, `z-index:50`, display serif title) unlocks audio and calls
  `Scene.begin()` on first pointerdown/keydown, then fades out. **Audio engine** (grep `audio`): a
  looping `theme.mp3` music BED (`fetch`+`decodeAudioData`, dual-source equal-power **crossfade loop**
  via `scheduleMusicLoop`/`playMusicOnce` setTimeout chain) + retuned quiet synth `plink`/`thud`/
  `chord` mixed on top (`sfxBus`). `S.music`/`S.sfx` replaced the old `S.sound`; music defaults ON,
  persisted in `plink.progress.v1` `audio:{music,sfx}` (default ON when the field is absent). The old
  synthesized sea pad (`startAmbient`) is GONE. Missing `theme.mp3` = graceful silence, plinks still
  work. `#musicbtn` (interim, in the Options row) toggles via `setMusic`. **A real `theme.mp3` now
  ships** (24.6s stereo loop J supplied); the crossfade loop (`XFADE=2.0`) covers the seam.

## Push B follow-ups (zen mode, fireworks, clear Mokes)

- **Zen (watch mode).** A `#zen` button in the Options row enters a drawaurora-style watch mode:
  `body.zen` fades the whole `.app` control layer to `opacity:0` (with `pointer-events:none` on the
  chrome so taps fall through to `#scene`), leaving only the living scene. A floating `#zenbar` pill
  (`z-index:40`) carries a **Fireworks** toggle + **Exit**; it auto-`.idle`-fades after ~4.2s of
  stillness and returns on `pointermove`/tap (`zenWake`). `enterZen` calls `stopModes()` so nothing
  pours behind the curtain; `exitZen` turns fireworks off. Escape exits. `#zen` is `grid-column:1/-1`
  (full-width in both dock grids); it lives only in Play's Options, so Learn never shows it.
- **Fireworks** (inside the `Scene` IIFE; public `Scene.setFireworks(on)`): `launchRocket` sends a
  rocket up from the horizon, `burst` sprays ~46-88 **pre-rendered glow-dot sparks** (`fwSprite`
  cache, one soft radial per colour, saturated purple/green/gold/pink with a small hot core — the
  drawaurora dot look, NOT streaks), `updateFireworks(dt)` runs gravity/drag/fade, `drawFireworks`
  blits additively. Drawn **before `drawSea`** so the mirror-calm water reflects every bloom. `dt`
  comes from `time-lastDrawT` inside `Scene.draw` (tied to the 30fps-capped paint, so it pauses with
  the tab). Sparks self-cap at 1000. Fireworks only auto-launch while `fwOn` (the zen toggle).
- **Clear Mokes.** The two islands were muddy bumps behind a tall ridge; now they are the hero
  silhouettes on an open horizon (see the `Scene` bullet above). If you re-tune, keep them clearly
  darker than the sky at EVERY hour (that is what `isleColor` guarantees) and keep a visible water
  gap between them.
- **Painterly scene pass (DONE, the "90s" fix).** The three offenders were rebuilt drawaurora-grade,
  all soft and pre-rendered, verified across every hour + a clean 4-dimension adversarial review:
  - **Aurora** (`makeAuroraSprite` + `drawAurora`): the hard vertical streaks are gone. It is now a
    soft continuous CURTAIN of dense overlapping radial dabs along a waving sine spine, tint walking
    **green -> teal -> violet** across x, with faint vertical drapes, all melted smooth by a heavy
    1/6 downscale/upscale blur. Blitted `lighter` into the **upper-LEFT third only** (right side stays
    dark and empty), **night-only** (gated by `starK`), alpha `starK*0.8`. It is now a **user TOGGLE**
    (`S.aurora`, default ON): the `#aurorabtn` in Options + the `#zenaurora` pill in the zenbar both
    call `setAurora` (kept in sync), persisted in progress `scene:{aurora}`. Forced OFF under `S.lite`;
    drift frozen under `reduce`. NOTE: the moon HALO was shrunk (`r*3.6`, alpha 0.32) and **night
    clouds fade out** (`drawClouds` scales alpha by `1-starK*0.9`) so the dark sky lets the aurora read.
  - **Glitter beam** (`drawGlitter`): the blocky `fillRect` specks are gone. Now a feathered trapezoid
    light column (widening with depth, `body.tint`) plus a few soft elongated GLINT dabs (`glintSprite`,
    blitted `lighter`, jittered). Glints skipped under `S.lite`; phase frozen under `reduce`.
  - **Sea reflection** (`drawSea`): the venetian-blind band loop is gone. Now ONE soft flipped blit:
    capture the sky+land band into `mirrorCanvas` (**source rect is DEVICE px** `W*DPR x horizonY*DPR`,
    dest is CSS px — the DPR gotcha), cheap-blur via `blurCanvas` (1/4 downscale), blit flipped once
    with `translate(_,horizonY*2)+scale(1,-1)` at alpha 0.42 + a gentle wobble/squash + a horizon haze
    wash. Skipped entirely under `S.lite` (base gradient + haze + glitter still pretty).
  - **Layered hills (REMOVED).** The painterly pass briefly added three receding sine-profile ridges
    behind the Mokes, but their crests read as extra islands (J: "there are 4 islands, we only want
    the 2 Mokes"), so `LAYERS`/`ridgeProfile`/`drawRidge`/`ridgeProfiles` were deleted. `drawLand`
    now paints ONLY the two Mokes on an open horizon (the authentic view: two offshore islets on open
    ocean). `RIDGE` stays — it is the land-green tint mixed into `isleColor`. If depth ever feels thin,
    add FLAT atmospheric haze at the horizon, never island-shaped humps. `nui`/`iki` cached in `resize()`.

## Weather, sand shoreline, day/night control, menu redesign (this pass)

A follow-up on J's asks ("sand shoreline like Lanikai", day/night control, rare shooting stars + rain +
lightning, kill the "Plink" intro wordmark, and "the non-minimized menu still looks like dogshit").

- **Foreground sand shoreline.** The sea no longer runs to the bottom: `resize()` computes `shoreY`
  (`round(H*0.845)`); the sea fills `horizonY..shoreY` (`drawSea` uses `seaBot=shoreY`, so `seaH=shoreY-
  horizonY`) and the **beach** fills `shoreY..H`. `buildShore()` (called in `resize`) precomputes
  `shoreBands[]` (4 sine-profile berms, back=wet near the water, front=dry at the bottom); `drawShore(P,
  time)` paints them back-to-front with `sandColor(P,f)` (blends `SAND_DAY` warm gold -> `SAND_NIGHT` cool
  moonlit by `P.starK`; each berm a shade lighter than the last), plus a wet-sand sheen, a breathing FOAM
  waterline, lit berm crests, and a bottom vignette. GOTCHA fixed: a sine berm dipping below the waterline
  left an uncovered dark seam, so `drawShore` FIRST lays a **flat wet-sand base** (`fillRect` shoreY..H)
  that the sine berms draw on top of. The mirror reflection is unaffected (it still samples `scv` `0..
  horizonY` with the DPR source-rect trick and is clipped to the sea rect).
- **Weather (all rare, ambient, gated by `S.weather` + `reduce`; motion only).** In the Scene: **shooting
  stars** (`meteors[]`/`spawnMeteor`/`updateMeteors`/`drawMeteors`, night-weighted by `starK`, a bright
  additive streak + glow head), **rain** (`rainDrops[]` built in `resize`, `updateRain` eases `rainLevel`
  toward `rainSpell`, `drawRain` thin slanted streaks, fewer under `S.lite`), and **lightning** (`flash`
  + one `reflash`, `drawFlash` a soft full-height additive wash — cozy, not a strobe). `updateWeather(dt,P)`
  is the scheduler (long dry spells, short showers; a rare flash while it rains fires distant thunder). It
  runs from `draw()` on the same `fdt`. **`thunder(delay)`** (audio section, near `chord`) is fully
  synthesized (`makeNoise` brown-noise buffer -> sweeping lowpass + sub-sine, slow swell + long tail, very
  quiet, gated by `S.sfx && S.weather && AC`, silent before the veil tap — no asset). **Weather toggle**
  `#weatherbtn` -> `S.weather`, persisted in progress `scene:{weather}` (default ON when absent).
- **Day/night control (drawaurora-style).** Scene gained `pinned`; `setTime(t)` grabs+holds the clock
  (`pinned=true`), `live()` hands it back (`pinned=false`, resumes if play+started), `isPinned()`. `frame()`
  time order is now `pinned > reduce > ease > running`. UI: `#tod` range (a fixed night->day->night gradient
  track) + `#todlive` "Live" toggle in Options ("Sky" section). Dragging pins (Live off); Live un-pins.
  `syncSky()` (called from the main `frame`, throttled 1-in-14, only when `#tod` is on screen) makes the
  thumb follow the live clock. `Scene.release()` (exitLearn) also clears `pinned`.
- **Intro veil.** The big serif "Plink" wordmark is GONE; the veil is now a tasteful inline-SVG **bell
  curve** (`.veil-bell`, gradient stroke that draws itself in) over "touch anywhere to start". The old
  `.veil-title`/`--display` serif path was removed.
- **Menu redesign.** Zen + Minimal left the Options panel; they now live in a **`.modesrow`** (Zen | Minimal,
  2-col) pinned at the bottom of the dock/rail, hidden in Learn (`margin-top:auto` for the tall rail PLUS
  `position:sticky;bottom:0` with an opaque frosted backing, so on a short landscape phone where the rail
  scrolls they stay reachable at the visible bottom instead of dropping below the fold). `#minimalbtn`
  is now just **"Minimal"**. Options gained `.opthead` section labels (Board / Sky), the Time meter, and a
  Weather toggle; it is forced to a 2-col grid in portrait (`.dock .optsrow`) so Bell|Paths, Music|Weather
  pair evenly. The stale `#zen{grid-column:1/-1}` rule was reduced to a border accent, then dropped
  entirely in the dock-chrome-polish pass below (Zen now reads identically to Minimal).
- **Minimal is now IMMERSIVE.** `body.minimal` hides dock/topbar/insight and collapses `.app` to a single
  full-bleed `"board"` area; `.cabinet` goes transparent/borderless so the near-full-screen Galton board
  floats directly on the living scene (a zen way to just watch it), with only the slim `minibar` at the
  bottom. Works in BOTH orientations (the `body.minimal .app` grid override out-specifies the portrait and
  landscape `.app` grids). The minibar still borrows `optsrow`/`labrow` into pop-up sheets (which now carry
  the new Time/Weather controls too).

## Dock chrome polish ("make the full dock feel like the minibar")

J: "polish the dock" (the non-minimized menu still looked like a grid of dark boxes bolted on the scene).
A CSS-only restyle that ports the minibar's clean language onto the full dock. Design-panel Workflow (3
directions -> synthesis) + a 4-lens adversarial-review Workflow that caught 3 real issues (all fixed).

- **Borderless-at-rest buttons.** Base `button` border went `1px solid var(--line)` -> `1px solid
  transparent` (kept the faint `var(--surface)` fill so text controls still read as tappable chips). This
  single change kills the grid-of-boxes; every dock/panel button is now a soft chip, not an outlined box.
  The amber `.primary` hero and `.minibar .mi` (own rules) are untouched.
- **Active toggle = the minibar's soft signal.** `button[aria-pressed="true"]` dropped the inset aurora
  ring for `background:var(--surface-2); color:var(--aurora); border-color:aurora 45%` (rain: amber 50% +
  `--honey` text). Mirrors `.minibar .mi[aria-pressed]` exactly.
- **Segmented pair (`.seg` = Learn|Play + Options|Lab)** got ONE group outline (`box-shadow:inset 0 0 0
  1px`) + a single hairline internal divider (first-child `border-right`). GOTCHA the review caught: the
  new aurora active-border collided with the divider (double hairline at the seam), so `.seg
  button[aria-pressed="true"]{ border-color:transparent }` was added AFTER the first-child rule (equal
  specificity, source-order wins) so inside a seg the active read is fill + aurora text only; the group
  outline + divider own the edges.
- **Container softened** slab -> floating card: `.dock` + `.insight` border to a hairline (`--line` 55%)
  and shadow from `--shadow-card` to a soft `0 14px 44px -30px`. Kept the `--glass` fill + blur, so day-sky
  legibility is unchanged (verified over a real noon sky).
- **Landscape void fix.** The rail was a full-height slab (`height:100%`) with a dead middle. Now
  `max-height:100%; height:auto; align-self:center` so it SHRINK-WRAPS to content and floats CENTERED in
  the column (scene above/below); when content overflows a short landscape phone it caps at 100% and
  scrolls with the SOLID `--glass` fill intact (legibility never breaks). GOTCHA the review caught: naively
  dropping the sticky footer regressed a deliberate prior fix (Zen|Minimal scrolled below the fold on a
  short landscape phone, worse with Options open). Restored `.dock .modesrow{ position:sticky; bottom:0;
  margin-top:auto }` with a `color-mix(--night 96%)` frosted backing (covers controls scrolling behind;
  margin-top:auto collapses to 0 when the card is content-sized, so the fits case stays a clean floating
  card). The opaque footer band is inherent to a sticky footer over a translucent scrolling card.
- **Quieter chrome.** `.eyebrow` green softened to `color-mix(--aurora 52%, --ink-faint)`. `.modesrow
  button` (Zen|Minimal) kept `background:var(--surface)` (NOT transparent) so they read as tappable chips
  on touch, where there is no `:hover` to reveal an outline (another review catch). `#zen` border-accent
  removed so Zen matches Minimal.

## Moke reshape, minimal stars, portrait fix, touch-first buttons (this pass)

J's asks off the reference photos of the real twin islets: the Mokes were not right, the stars looked
like "floating dust everywhere", the islands "bunch up like creepy mountains" in portrait, and one more
pass on the buttons for iPad / iPhone Pro first. Verified in-browser at day/golden/night and portrait +
landscape via the temp `window.__scene` hook (removed before ship); adversarial-review Workflow after.

- **Moke silhouettes rebuilt SMOOTH.** `islePoints(cx,w,h,baseY,kind)` now takes a `kind` (`'nui'`/`'iki'`)
  instead of a `lean` number, and defers the profile to `isleProfile(kind,f)` built from a small gaussian
  helper `bell(f,c,w)` (Scene-scoped, no collision with the `#bell` button or `S.showBell`). The craggy
  sine ripple is GONE (it read as jagged mountains). **Moku Nui** = a tall headland left-of-centre falling
  to a lower rounded shoulder on the right (`0.36*body + 0.72*bell(.34,.17) + 0.50*bell(.64,.20)`);
  **Moku Iki** = one clean gently-peaked dome (`0.52*body + 0.52*bell(.46,.30)`). `body =
  sin(f*PI)^0.80` keeps the base zero at both shores. Matches the sunset-silhouette reference: peak-left
  Nui, rounded-dome Iki.
- **Portrait bunching fixed (heights are WIDTH-driven, not sky-driven).** The old geometry keyed island
  HEIGHT to `horizonY`, so a tall portrait viewport stretched them into close, creepy spikes. Now in
  `resize()`: `uw=min(W*0.27,300)` is the Nui base width; `NUIh=min(uw*0.46, horizonY*0.34)` and
  `IKIh=min(uw*0.58*0.62, horizonY*0.26)` scale height to WIDTH with a horizonY cap (so a short landscape
  phone never looms). Positions spread to `NUIx=W*0.37`, `IKIx=W*0.70` with proportional widths so a clear
  water channel survives in BOTH orientations (verified ~46px gap at 375px portrait). If you retune, keep
  height width-driven and keep the channel; do NOT key height to horizonY again.
- **Minimal starfield (killed the dust).** `makeStarSprite` dropped the big-faint-halo branch entirely
  (that soft-blob branch WAS the "floating dust") and cut density `N=w*h/2600 -> w*h/11000`; every star is
  now a small crisp point (`s=0.7+1.5`, `bright=0.5+0.5`, high in the sky at `y<h*0.78`). `buildTwinkles`
  went `26 -> 9` gentle sparkles (`s=0.8+1.1`). `drawStars` unchanged (still reads `tw.s/x/y/sp/ph`).
- **Touch-first button pass (iPad / iPhone Pro first).** (1) Every `:hover` rule is now gated behind
  `@media (hover:hover)` (base button, primary, `.modesrow`, `.minibar .mi`) so hover styling never
  STICKS after a tap on iOS. (2) Active toggles got a glance-clear tinted FILL, not just tinted text:
  `button[aria-pressed="true"]` -> `background:color-mix(--aurora 15%, --surface-2)` + aurora border/text;
  the `.rain` variant uses amber; `.minibar .mi[aria-pressed]` mirrors the aurora tint so dock == minibar.
  The `.seg button[aria-pressed]{border-color:transparent}` divider guard still holds, but the review
  caught that the `border-color` shorthand ALSO zeroes the first tab's `border-right` (which IS the
  divider), so the hairline vanished whenever the LEFT tab was selected (Learn, or Options). Fixed with
  `.seg button:first-child[aria-pressed="true"]{ border-right-color:color-mix(--line 55%) }` to re-assert it.
  (3) Comfortable targets: `.livebtn` 32->36, `.zenbar button` 40->44, narrow-phone `.minibar .mi`
  min-height 42->44. `button:active` press deepened to `scale(.97)`. NOTE `color` is not in the button
  `transition` list, so verifying an aria-pressed toggle's FILL in a hidden preview tab shows the start
  colour (the background transition never advances when the tab is hidden) while the text snaps — bypass
  with `el.style.transition='none'` when probing computed styles.

## The four levers (all in Lab mode)

1. **Bias** (`S.bias`): P(bounce right). Shifts the peak, stays a bell. The control case.
2. **Momentum / Streaks** (`S.momentum`): each bounce blends toward the previous one (dependence).
   Mass spills to the edges; genuine fat tail relative to the naive bell.
3. **Wild / Lévy** (`S.wild`): per-BALL chance of going wild, spread across rows as a per-step
   probability (`1-(1-wild)^(1/n)`). A wild step takes a heavy-tailed signed jump (`levyJump`,
   Pareto tail alpha ~1.3). The ball walks the pegs, then visibly LEAPS to its outlier slot.
4. **Side by side** (`S.compare`): two boards, a locked Fair one and a Rigged one that mirrors the
   levers, laid out with the SAME dx and domain (`layoutAll`) so the tails line up.

Adult readouts: **far-out balls** (tail share beyond 3σ, `#tailstat`), **farthest Nσ**
(`board.farSigma`), and a **Log Y** toggle (`S.logScale`; linear packs marbles, log draws bars so
shapes read straight).

Kid-facing labels (IDs and math unchanged): Bias is **Lean** (`#bias`), Momentum is **Streaks**
(`#mom`), Wild is **Wild jumps** (`#wild`), Side-by-side is **Compare** (`#compare`), and the rules
reset is **Make it fair** (`#resetrules`). Do not rename the IDs.

## Gotchas

- The histogram is a `Map`, not an array, precisely so wild outliers never index out of range and
  domain changes never lose data. Don't "simplify" it back to a fixed array.
- Sprites are keyed by slot and cached (`spriteCache`); colour depends on `S.rows` and the fixed
  palette, so the cache is cleared when Rows changes. Sprite colour is by
  distance-from-centre only (board-independent) so both boards share sprites. An IN-FLIGHT ball is
  drawn with `ballColorSlot(b)`, NOT its landing slot: while it walks the pegs it wears its CURRENT
  column's colour (so a tail-bound marble is not blue until it earns it), revealing its true slot
  only as it drops in / leaps / settles. `ballColorSlot` MUST round+clamp to an integer and read
  `b.board.geo` (never a captured `g`/`primary`) so the cache stays integer-keyed and side-by-side
  boards colour independently.
- Motion (anti-"snake"): balls spawn scattered across the hopper mouth (`±dx*0.85`) with jittered
  start height and per-BALL hop speed, and every segment's duration is jittered in `startSeg`, so a
  poured crowd desyncs into a curtain instead of a single-file thread. `segTarget` adds a small
  lateral wobble to peg glances; the final drop into the bin stays exact so landings are honest.
- Motion juice (aurora pass): the final drop eases with `pow(t,2.2)` (heavier fall); a lone ball
  gets ONE bounce-settle micro-state (`b.bouncing`, gated to crowds < 60 and `!reduce && !S.lite`)
  before `settle()` runs, so counts land a beat later but are never changed. Every falling ball
  carries a short light trail (`b.trail`, 8 points; a wild leap gets 14) drawn as one additive
  two-stroke polyline coloured by `ballColorSlot`, gated to crowds < 150; when gated, tails burn
  off point by point. Landings push a `ground` ripple flash coloured by `binInfo(b.land).rgb`.
- Aurora + pegs: `drawAurora` blits 3-4 pre-rendered blobs (palette colours only) with `lighter`
  compositing, drifting on slow sines, concentrated in the upper board; reduce-motion freezes the
  drift. Pegs are one cached sprite per radius (`pegSpriteFor`), alpha-boosted per peg by
  `board.pegGlow` (Float32Array indexed `r*(r+1)/2+i`, set in `onPeg`, decayed in `tick`). A faint
  cyan sea-horizon line sits at `g.binTop` (the Push 2 seaside-story seed).
- Adaptive quality: `frame()` keeps a rolling frame-time average and flips `S.lite` on above ~22ms
  (off below 17ms); `S.lite` sheds trails and bounce-settle first. The per-frame body lives in
  `tick(dt)`, separate from the rAF plumbing, so verification (and future Learn scripting) can
  step the sim manually while the tab is hidden.
- Collision (`=== COLLISION ===`, `separate()` called in the frame loop): marbles are solid. A
  spatial-hash separation pass pushes overlapping in-flight balls apart via a decaying RENDER offset
  (`b.ox/b.oy`, sprung back each frame in `updateBall`), so a pour jostles like real marbles instead
  of ghosting through each other. It moves the render position ONLY; `settle()` stays time-based on
  `b.land`, so counts/stats are untouched (a fair pour still makes a clean bell). Vertical push is
  damped so it can't fight the fall; leaping wild balls skip it; offsets are capped and clamped to
  the board region so nothing bleeds across the side-by-side divider. ~0.7ms/frame at 250 balls.
- Palette (FIXED, no picker): the marble `let`s `HONEY/AMBER/AMBERD` (calm common ramp) and
  `CYANS/CYAN` (rare-edge pop) ARE the palette, mirrored into the `:root` CSS custom props that
  colour the UI chrome. A single fixed palette replaced the old 5-theme `Colors`/`applyTheme`/
  `#themebar` picker (dropped for a fixed-palette + no-emoji direction; shipped in the chrome
  redesign commit). Invariant: crowded middle stays CALM, rare edge POPS. Effect colours (trail,
  celebrate, chevrons, guess, labels) read the marble vars. Don't hardcode `84,236,220` etc back in,
  and don't reintroduce a theme picker without asking.
- Control chrome (redesigned): the dock is ONE token-driven control card in both orientations
  (`--sp-*` / `--r-ctl` / `--r-card` / `--shadow-card` / `--rail-w` / `--ease-settle`). Clusters are
  CSS grids — portrait `repeat(3,1fr)`, landscape a 2-col rail — with `.group{display:contents}`
  flattening buttons into them; Drop 1 is a full-width hero, Options|Lab a `.seg` segmented pair,
  sliders are meter rows (`grid` label / track / value), `.stat`s are readout rows, and `.railfoot`
  pins a quiet plaque to the bottom of the landscape rail. `#dockcount` mirrors the counter into
  the rail header. All controls keep >=44px targets + focus-visible outlines. Do NOT revert to
  flex-wrap: it went ragged in portrait and full-width lonely-word pills in landscape.
- Options / Lab are ONE accordion (`setPanel`): opening a panel closes its sibling, tapping the
  open one closes it, `aria-pressed` stays in sync. Do not revert to independent toggles (both
  panels used to stack open at once).
- Slider meters: the label column is FIXED (5.2em portrait, 4.5em in the landscape rail), never
  `auto`, so every track starts at the same x. The filled track is a CSS gradient driven by
  `--fill`, painted by `paintFill(el)` on init and input; call `paintFill` after any programmatic
  `.value` change (Make it fair does).
- Type: one legible system sans (`--font`) + tabular mono numbers (`--mono`). The serif face was
  removed; do not reintroduce it.
- Audio: all synthesized, off by default, context created only inside a user gesture. `plink`
  (peg tick) and `thud` (water-drop landing) pitch along a pentatonic map (`pent`); `chord()` is
  the rare-event chime with a sparkle partial; `startAmbient`/`stopAmbient` run the sea pad
  (detuned sines through a breathing lowpass + bandpassed surf noise) tied to the Sound toggle.
- Learn mode (the story journey): chapters BORROW real dock controls into `#learnstage`
  (`buildStage`/`stageClear` move the live nodes, listeners and ids travel with them, and
  `stageClear` restores them in reverse insertion order). Never clone controls for the stage:
  a clone would fork the listeners. Exactly ONE element carries `.spotlight` at a time. The
  `completeWhen` gate lives in `tick()` (cheap `ready()` predicates on the live boards) so it
  works under the manual stepper too; `Skip` never waits on it. Two gate honesty rules learned
  the hard way: chapter 4 gates on `board.wildLanded` (a counter bumped in `settle()` when a
  landed ball's path took a wild jump) because `farSigma>3` can be banked by a FAIR end-bin
  ball; chapter 5's `begin()` baselines settled + airborne + pending so the watch demo's
  still-falling balls can never complete the gate for a fast-tapping kid. Chapter narration strings live
  in `CHAPTERS` and are innerHTML: keep them kid-plain, seaside-threaded, bold ONE idea, and
  NEVER an em dash. In learn mode the sandbox `.controls` rows and the `.insight` strip are
  CSS-hidden (`.app.learn ...`), so lever handlers may still write insight text harmlessly.
  Landscape learn hides the railhead/railfoot and widens `--rail-w`; `startTinker` scrolls the
  instruction to the top of the dock so lever + tools + Next sit in view below it.
- Progress: localStorage `plink.progress.v1` = `{v:1, lastMode, chapter, completed[], levers{}}`,
  written on mode/chapter/phase/completion changes, all reads/writes try/catch wrapped. On boot
  `initLearn()` (runs after `buildBoards`, before the first frame) restores completion and, if
  `lastMode==='learn'`, resumes the story at the saved chapter (always in `watch` phase). A fresh
  or `play` user boots into the sandbox with default levers: stored `levers{}` are a snapshot for
  future use, deliberately NOT restored, so Play stays exactly the fresh sandbox.
- `preview_screenshot` caches an early pegs-only frame for this rAF canvas after a relayout. Verify
  rendering with `getImageData` pixel probes, not screenshots. NOTE: the preview tab is usually
  `document.hidden`, so rAF is throttled to zero and the canvas never animates on its own — to
  verify motion/settling, temporarily expose a manual stepper (`step(dt,k)` that runs the frame body
  minus the rAF reschedule), drive it from `preview_eval`, probe `board.bins`, then remove the hook.

## Local dev + deploy

- Serve: launch.json config name **`plink`** → `python3 -m http.server 8210 --directory` this folder.
  Local URL `http://localhost:8210/`.
- Deploy: GitHub Pages, `index.html` at repo root serves at the site root. `.nojekyll` is committed.
  Repo owner is **xy808yx**; live at `https://xy808yx.github.io/plink/` once Pages is enabled on
  `main` / path `/`.

## Style

- Commits use the anonymous identity `xy808yx <9j48dnjksk@privaterelay.appleid.com>`. Never leak a
  real name or email.
- No em dashes ANYWHERE: docs, chat, and (since the aurora pass) the in-app copy too. The old
  `&mdash;` voice was retired when the copy went kid-plain (short warm sentences a 7 to 10 year
  old can read on their own). Keep new insight strings in that voice: short sentences, plain
  words, bold only the one idea that matters.
