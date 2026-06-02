# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **standalone, zero-dependency UI prototype** for a radial drag-to-score 501 darts scorer. Single file: `index.html`. No build step, no npm, no framework. Purpose: tune the gesture feel and layout on a real iPhone before integrating into the main "Darts is Fun" app.

## Developing

Open `index.html` directly in a browser, or serve with any static file server:

```bash
npx serve . -l 5180
```

Deploy to production: push to `main` on `github.com/szlanyina/newscoring` → Vercel auto-deploys to `newscoring.vercel.app`.

## Architecture

Everything lives in one `index.html` with three logical sections:

**HTML** — three elements: `.scoreboard` header (static placeholder), `.radial-wrap > svg#scorer` (the dial, fully JS-generated), `.back` button.

**CSS** — inline styles only. Design tokens in `:root`. Key rules: `height: 100dvh; overflow: hidden` on `.frame` locks layout to viewport. `touch-action: none` on `svg` + `-webkit-touch-callout: none` on `*` prevents iOS from stealing touch events during drag.

**JS** — no framework, no modules. Three data structures drive everything:

- `T` — all geometry constants (ellipse radii, segment count, center). **Tune here first** — the whole dial reshapes from these values. ViewBox is `600×790`, center at `(300, 395)`.
- `RANGES` — the 9 score buckets. `RANGES[i]` maps 1:1 to `BUBBLE_POS[i]`.
- `BUBBLE_POS` — hand-placed SVG coordinates for each bubble. `RANGES[0]` (`0-19`) is the center bubble; indices 1–8 ring around it clockwise from top.

**Render loop** — `render()` blows away `svg.innerHTML` and redraws everything from `STATE` on every frame. There is no virtual DOM; direct SVG DOM manipulation. Called on every pointer event that changes state.

**Gesture engine** — unified Pointer Events (`pointerdown/move/up/cancel`) with `setPointerCapture` so drags don't lose tracking. Flow: `pointerdown` on a bubble → activates that range → `pointermove` calls `segmentAt()` to find which of the 20 ring slots is under the finger → `pointerup` commits the value.

**Hit-testing** — `bubbleAt()` uses Euclidean distance with `GRAB_SLOP` multiplier (forgiving on mobile). `segmentAt()` normalizes the pointer to a unit circle (compensating for the ellipse) then uses `atan2` to find the angular sector. `SELECT_THRESHOLD` (0.80) is the normalized radius below which no segment fires — the dead zone.

**Number placement** — each ring number sits at `lineIntersect(innerLeft, outerRight, innerRight, outerLeft)` — the true geometric center of each trapezoid cell, not the arc midpoint.

## Key tuning knobs

| Constant | Effect |
|---|---|
| `T.ringOuterRx/Ry` | Overall dial size |
| `T.ringInnerRx/Ry` | Width of blue number band (smaller = wider band) |
| `T.discRx/Ry` | Green oval shape |
| `T.bubbleR` | Bubble radius |
| `BUBBLE_POS` | Individual bubble placement (viewBox coords) |
| `SELECT_THRESHOLD` | How far finger must reach into the ring to arm |
| `GRAB_SLOP` | How forgiving the initial bubble grab is |

## Phase status

- **Phase 1** ✅ Static layout
- **Phase 2** ✅ Drag gesture (pick range → drag → live preview → commit)
- **Phase 3** 🔲 501 countdown (subtract on commit, bust <0, win at 0, visit history, undo)
