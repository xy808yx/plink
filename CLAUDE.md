# CLAUDE.md — Plink

Context for future sessions working on this repo.

## What Plink is

A single self-contained `index.html`: an interactive Galton board that teaches probability and,
crucially, **fat tails**. "Plink" is the kid-facing name (the sound the marbles make). The
fat-tails depth lives behind a **Lab** toggle. The teaching arc is: a *fair* board is a Gaussian
factory; break independence or the fixed step size and the bell dies (Mediocristan to Extremistan).

Credits the project leans on: **Galton** (the 1877 quincunx), **Mandelbrot** (heavy tails,
Mediocristan/Extremistan), **Taleb** (fat tails as a worldview). See README, which leads with the
"break the machine to reveal Extremistan" story.

## Hard constraints (do not violate)

- **One file.** Dependency-free vanilla JS + Canvas, single `index.html`. If you ever must bundle,
  use a tiny esbuild step that still emits a single `index.html`. The ONE allowed exception is the
  PWA sidecar (static assets, no code): `manifest.webmanifest`, `icon.svg` (the cyan fat-tailed
  curve — vector source of truth), `icon-192.png` / `icon-512.png` (any + maskable), and
  `apple-touch-icon.png` (180, iOS home screen). Regenerate PNGs from `icon.svg` with
  `qlmanage -t -s 1024 -o <dir> icon.svg` then `sips -z <n> <n>`. Head wires favicon (SVG + PNG
  fallback), apple-touch-icon, manifest, and `theme-color`. App CODE stays single-file.
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
