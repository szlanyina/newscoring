# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **standalone, zero-dependency UI prototype** for a radial drag-to-score 501 darts scorer. Everything lives in one **`index.html`** — no build step, no npm, no framework. Purpose: tune the gesture feel, layout, and colors on a real iPhone before integrating into the main "Darts is Fun" app. Solo play (one live player) with a static opponent panel kept structurally identical for later 2-player wiring.

## Developing

There is no build, lint, or test step. Edit `index.html` and reload.

```bash
npx serve . -l 5180     # or any static file server
```

Deploy: push to `main` on `github.com/szlanyina/newscoring` → Vercel auto-deploys to `newscoring.vercel.app`.

Supporting files: `manifest.webmanifest` + `icon.svg` make it an installable PWA (display `fullscreen`, black theme) so "Add to Home Screen" launches dark and chromeless.

## Architecture

`index.html` has three logical sections: **HTML** (header scoreboard + `svg#scorer` dial + footer buttons), **CSS** (all inline, design tokens in `:root`), and **JS** (no modules).

### Dial geometry — the data structures

- **`T`** — all geometry constants (ellipse radii `ringOuterRx/Ry`, `ringInnerRx/Ry`, `discRx/Ry`; `bubbleR`; `segCount: 20`; `startAngleDeg: -81`). **Tune the dial here first.** ViewBox is `600×790`, center `(300, 395)`. `startAngleDeg: -81` (not -90) puts a *cell boundary* on the vertical axis so the ring splits 10 numbers left / 10 right.
- **`RANGES`** — 10 score buckets, e.g. `"150-169"`, `"0-9"`, `"170-180"`. **`RANGES[i]` maps 1:1 to `BUBBLE_POS[i]`.** Order is arbitrary/hand-tuned; the comments in `BUBBLE_POS` are historical and may not match — trust the index, not the comment.
- **`BUBBLE_POS`** — hand-placed SVG coords per bubble (index 0 = centre, the rest ring around). Edit these to move bubbles.
- **`RING`** — computed once at load by `computeRing()`: `equalArcPoints()` divides each of the inner & outer ellipses into 20 **equal arc-length** boundary points (starting at top). Cell `i` connects `RING.inner[i]/[i+1]` to `RING.outer[i]/[i+1]`. This gives visually uniform cells (equal parametric angle would not, on an ellipse).

### Render loop

`render()` wipes `svg.innerHTML` and redraws everything from `STATE` each call — no virtual DOM, direct SVG. Called on every state-changing pointer event. Draw order: (1) ring cells via `roundedCellPath()` (rounded-corner trapezoids, `fill-opacity 0.7`), (3) transparent inner disc, (4) ring numbers **only while dragging** (the previewed cell's number is omitted — it shows on the bubble instead), (5) bubbles (active bubble drawn last so it floats on top), (6) the header "actual throw" value.

### Number / value placement

- **Ring numbers**: each cell's value sits at the centroid of its four `RING` corners (`cellCorners(i)`). Single-line vs two-line bubble labels use specific `dy` offsets (e.g. `dy="0.05em"` single, `-0.45em`/`1.0em` two-line) because iOS Safari ignores `dominant-baseline`.
- **`buildSegmentValues(vals, rangeStr)`** maps a bucket's values onto the 20 ring slots. Default: lower half on the **upper** arc (left→right), higher half on the **bottom** arc. `0-9` is all-bottom; `170-180` puts 170–179 on top + 180 at the first bottom slot. Bubble labels replace the trailing 0 with `*` (10→"1*", 0→"*").

### Gesture engine

Unified Pointer Events with `setPointerCapture`. Flow: **`pointerdown` on a bubble** (`bubbleAt()`, Euclidean distance × `GRAB_SLOP`) arms that range and the bubble jumps to the finger → **`pointermove`** updates `dragPos` (bubble follows finger) and calls `segmentAt()` → **`pointerup`** (`endDrag`) commits. `segmentAt()` is **point-in-polygon** against each cell quad (not angle math). On a scoring drop the bubble **lingers 700ms** at the drop spot showing the recorded throw, then snaps home (non-scoring release snaps immediately).

### Scoring state & operations (`STATE`)

- `remaining` counts down from `START` (501); `visits` is the full log of `{score, bust, after}`.
- **`recompute()`** replays all visit scores from `START` to refresh every `bust`/`after` and `remaining` — call it after any edit/undo/redo.
- **Edit**: clicking a player throw box sets `editIndex`; the next dial commit overwrites that visit instead of appending.
- **REM mode** (`remMode`, toggled by the REM button): the dialed value is treated as the *new remaining*; recorded throw = `remaining − dialed` (invalid over-dials are ignored).
- **Undo/redo** (`undoStack`/`redoStack`): each op is `{type:'add'}` or `{type:'edit', index, oldScore}`. Undo reverses the last op (an edit-undo restores the old score, not deletes a visit); redo re-applies it; any fresh commit clears `redoStack`.

### Header scoreboard

Grid `auto 1fr auto`: **player panel (magenta) left, opponent panel (gray/blue) right**, sets/legs in the middle. Each panel = a fixed-width `main-col` (big remaining + actual-throw box) and a scrollable `hist-col` (last throws, newest on top, dart-count badge per box). Side panels are pinned to the scoreboard edges so they don't shift as throws are added. The player side (`#playerRemaining`, `#scoreVal`, `#actualDarts`, `#histCol`) is wired to `STATE`; the opponent side has mirror ids (`#oppRemaining`, `#oppActualVal`, `#oppActualDarts`, `#oppHistCol`) with static placeholder values, ready for later wiring. An empty player box is detected via the absence of a `data-vi` attribute.

## iOS / touch gotchas baked in

- `touch-action: none` on `svg` and `-webkit-touch-callout: none` on `*` stop iOS hijacking the drag.
- `.frame { height: 100dvh; overflow: hidden }` locks to the viewport.
- `dominant-baseline` is unreliable on iOS Safari — vertical centering uses explicit `dy` offsets.
- The screenshot/preview tool often shows a 1-frame-stale image; verify state via DOM/`STATE` reads, not only screenshots.

## Verifying changes on the mobile layout

The header is tuned to fit a 375px-wide phone without horizontal overflow — after any header size change, check `document.querySelector('.scoreboard').scrollWidth <= 375`. Many interactions can only be exercised by dispatching synthetic PointerEvents on `#scorer` (convert viewBox coords to screen via `svg.getBoundingClientRect()`); the dial only renders ring numbers while `STATE.dragging` is true.
