# Plink

**A fair Galton board is a Gaussian factory. Break it, and you can watch the bell curve die.**

### [▶ Play it live](https://xy808yx.github.io/plink/)

![Plink demo](demo.gif)

*(The GIF above is a placeholder. Replace it with a screen recording of the Lab levers in action.)*

Plink is a single, self-contained HTML file: an interactive [Galton board](https://en.wikipedia.org/wiki/Galton_board) that teaches probability, and then quietly teaches the thing most probability lessons skip, **fat tails**.

Drop marbles through a triangle of pegs. Each one bounces left or right at every row, and no one can say where a single marble will land. Drop a thousand and they always pile into the same smooth hill: the **bell curve**. That is the everyday miracle of the Central Limit Theorem, and for kids that is the whole toy. It even makes a satisfying *plink*.

But the bell curve is not a law of nature. It is a **consequence of three quiet assumptions**:

1. Each bounce is **independent** of the last.
2. Each step is the **same fixed size** (one peg left or right).
3. No single step can dominate the sum.

Break any one of those and the bell dies. That is the real lesson living behind the **Lab** toggle.

## Break the machine, reveal Extremistan

Flip on **Lab** and you get levers that rig the machine. Each one attacks a different assumption, and each one drags you out of the tidy world of averages (Benoit Mandelbrot's **Mediocristan**) into the world where one freak event swamps everything else (**Extremistan**).

- **Bias** leans the pegs. The hill slides left or right, but it stays a bell. A loaded coin *shifts* the peak; it does not fatten the tails. This is the control case, and the "beyond 3σ" readout barely moves, which is exactly the point.
- **Streaks (momentum)** makes each bounce lean the way the last one went. Now the bounces are no longer independent. Marbles spill *past* the curve into the edges, and that overspill **is** the fat tail.
- **Wild (Lévy jumps)** lets a rare bounce abandon the fixed step size and take one enormous, heavy-tailed leap. A few marbles rocket off the chart while the average becomes hostage to the single biggest jump. This is the mathematical signature of a market crash, a viral hit, a war.

Two readouts keep the adults honest:

- **beyond 3σ** measures how much mass lands past three standard deviations of the bell the machine *should* produce. Under a fair board it sits near the Gaussian 0.3%. Rig it and it climbs into double digits.
- **farthest** reports how many standard deviations out the single most extreme marble was. In Extremistan this number is absurd, 50σ, 100σ, events a Gaussian says should never happen in the lifetime of the universe.

Toggle **Log Y** and the punchline lands: on a log axis the bell nose-dives at the edges like a parabola falling off a cliff, while a fat tail refuses to, its bars staying stubbornly tall far out. And **Side by side** races a fair board against a rigged one on a shared x-scale, so the middles line up and the tails visibly diverge.

## Who this is for

Plink is layered by depth on purpose:

- **Kids** get a marble toy that makes a nice sound and always builds the same hill.
- **Parents and educators** get a clean, honest demonstration of the Central Limit Theorem.
- **Anyone who has read Taleb** gets a hands-on machine for the single most important idea in applied probability: the bell curve is fragile, and most of the consequential world does not live under it.

## Credits and lineage

Plink stands on three shoulders:

- **[Francis Galton](https://en.wikipedia.org/wiki/Francis_Galton)** built the original quincunx (the "bean machine") in 1877 to show how independent random steps sum into a normal distribution.
- **[Benoit Mandelbrot](https://en.wikipedia.org/wiki/Benoit_Mandelbrot)** spent a career showing that real prices, coastlines, and cotton markets are wilder than the bell allows, and gave us the Mediocristan / Extremistan distinction and the mathematics of heavy tails.
- **[Nassim Nicholas Taleb](https://en.wikipedia.org/wiki/Nassim_Nicholas_Taleb)** turned fat tails into a way of seeing the world in *Fooled by Randomness*, *The Black Swan*, and *Antifragile*.

Galton built the machine that makes the bell. Mandelbrot and Taleb spent their lives warning you not to trust it. Plink lets you do both in one place.

## Running it

There is nothing to build. It is one file with zero dependencies (vanilla JavaScript and Canvas).

- **Open it:** double-click `index.html`, or
- **Serve it:** `python3 -m http.server` and visit the printed URL.

## Deploying

`index.html` lives at the repo root, so GitHub Pages serves it at the site root. Enable Pages on the `main` branch with path `/` and the board is live at the Pages URL. A `.nojekyll` file is included so Pages serves everything verbatim.

## For future work

The code is a single `index.html` with the seams marked in comments: `CONFIG`, `BOARD`, `BOUNCE RULE`, `BIN DOMAIN`, `TAIL METRIC`, and `ARCHITECTURE`. See [CLAUDE.md](CLAUDE.md) for the design notes and the layered-depth philosophy.

## License

[MIT](LICENSE).
