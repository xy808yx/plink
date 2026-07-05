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
  use a tiny esbuild step that still emits a single `index.html`.
- **Preserve the glass-marble sprite renderer** (`makeMarble` / `sprite` / `drawMarble`,
  pre-rendered and blitted).
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

Adult readouts: **beyond 3σ** (tail share), **farthest Nσ** (`board.farSigma`), and a **Log Y**
toggle (`S.logScale`; linear packs marbles, log draws bars so shapes read straight).

## Gotchas

- The histogram is a `Map`, not an array, precisely so wild outliers never index out of range and
  domain changes never lose data. Don't "simplify" it back to a fixed array.
- Sprites are keyed by slot and cached (`spriteCache`); colour depends on `S.rows`, so the cache is
  cleared when Rows changes. Colour is by distance-from-centre only (board-independent) so both
  boards share sprites.
- `preview_screenshot` caches an early pegs-only frame for this rAF canvas after a relayout. Verify
  rendering with `getImageData` pixel probes, not screenshots.

## Local dev + deploy

- Serve: launch.json config name **`plink`** → `python3 -m http.server 8210 --directory` this folder.
  Local URL `http://localhost:8210/`.
- Deploy: GitHub Pages, `index.html` at repo root serves at the site root. `.nojekyll` is committed.
  Repo owner is **xy808yx**; live at `https://xy808yx.github.io/plink/` once Pages is enabled on
  `main` / path `/`.

## Style

- Commits use the anonymous identity `xy808yx <9j48dnjksk@privaterelay.appleid.com>`. Never leak a
  real name or email.
- No em dashes in prose deliverables (README, this file, chat). NOTE: the in-app copy (insight
  strings) already uses `&mdash;` as its established typographic voice; match the surrounding code
  when editing app copy, but keep docs and chat em-dash-free.
