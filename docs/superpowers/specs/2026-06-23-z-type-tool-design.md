# Z-Type Tool — Design Spec

**Date:** 2026-06-23
**File to build:** `z-type-tool.html` (fork of `z-grid-tool.html`)
**Status:** Approved, pending implementation plan

## Purpose

Fork the single-canvas oblique-grid glyph editor (`z-grid-tool.html`) into a
multi-letter typeface-design tool. The user designs a full alphabet on a shared
oblique guide system, editing three letters at a time, with a live sample strip
and real deliverables (`.otf` font + project JSON).

## Goals

- Design a whole alphabet (a–z, 0–9, punctuation) of consistent oblique letterforms.
- Edit **3 letters at once**, side-by-side, sharing one guide system so the
  letters stay visually consistent.
- Allow controlled per-letter divergence of guides **without** breaking the shared
  angle system.
- See all designed glyphs composited into editable sample text, live.
- Export a usable font and a re-openable project.

## Non-Goals (YAGNI)

- Per-glyph or contact-sheet **SVG** export (explicitly dropped).
- Separate OS windows / pop-out panels (panels live in one page).
- Kerning pairs, OpenType features, hinting. Advance-width spacing only.
- Curve tooling beyond what the existing engine already supports (cells are
  straight-edged polygons; imported/font curves are flattened as today).

## Architecture

Single forked HTML file. The current tool uses global singletons (one `svg`, one
`hG/oG/dG`, one `cells`, one `mode`). The fork refactors the editor surface into a
reusable **Editor** unit instantiated **3×**, all reading from shared global state.

### State layers

1. **Global, always shared**
   - Angles: oblique `K` / `oAngle`, angled `dAngle`. (Already global in the
     source tool.) Toggling split never unlinks angle.
   - **Metric frame:** `baseline`, `xHeight`, `cap`, `ascender`, `descender`
     (Y values in canvas units, 0..1280), plus default advance width / sidebearing.
   - **Shared guide positions:** `hG`, `oG`, `dG` (each guide `{id, v|px}`).

2. **Per-glyph** (stored per character)
   - `cells` — the filled cells (guide-bound or free polygons, as today).
   - Split config: `splitOn` (bool), `detached` (bool — "stop propagating"),
     `localGuides` = `{hG, oG, dG}` (the glyph's private guides / frozen copy).

3. **Alphabet store**
   - `GLYPHS`: `Map<char, perGlyphState>`. Autosaved to `localStorage` on change;
     loaded on startup.

### Effective-guides resolution (core mechanic)

For the glyph shown in a panel, the guides actually used are computed live:

| Split | Propagating | Effective guides |
|-------|-------------|------------------|
| off   | —           | shared guides only |
| on    | yes (default) | shared guides (live) **+** glyph's local *extra* guides |
| on    | no (stop-propagate) | private copy of guides (IDs preserved) **+** extras; shared edits no longer reach it |

- Cells bind to guides by **stable ID**, so all states reshape live via the
  existing `corners()` lookup.
- "Stop propagating" snapshots the current shared guides into the glyph's
  `localGuides`, **preserving their IDs** so already-placed cells stay bound.
- Re-enabling propagation (turning the sub-toggle off) re-links to shared guides;
  turning split fully off discards local guides (with a confirm if any cell binds
  to a local-only guide).

## Components

### Top toolbar (global)
- Tool mode: Move/Drag, +‖ guide, +╱ guide, +◹ angled, Fill, Erase, Pan
  (unchanged set from source).
- Oblique angle control (number + slider) → drives `K`, rotates ╱ family.
- Angled angle control (number + slider) → `dAngle`.
- Fill-pair selector (unchanged).
- Metric-frame controls: baseline / x-height / cap / ascender / descender Y
  (numeric inputs, mirrored by draggable on-canvas lines — see below), default
  advance width, default sidebearing.
- Snap / nodes / trace toggles (unchanged).
- Font import (reuse existing opentype.js glyph→cells importer, targets the
  focused panel).
- Export buttons: **Save Project (JSON)**, **Load Project (JSON)**, **Export .otf**.

### Editor panel (×3)
Each encapsulates its own `svg`, `viewBox` (independent pan/zoom), drag state, and
render loop. Panel header:
- **Letter dropdown** (a–z, 0–9, common punctuation). Switching saves the current
  glyph to `GLYPHS` and loads the selected char.
- **Split guide position** toggle.
- **Stop propagating guide edits** sub-toggle (visible only when split is on).
- Zoom in / out / fit.

The active tool is global; the panel under the pointer receives the action
(add/fill/erase/drag). Angle and metric changes re-render all three panels.

**Metric lines are draggable on-canvas.** Each panel draws the global metric frame
(baseline, x-height, cap, ascender, descender) as distinctly-styled horizontal
lines; dragging one in any panel updates the global metric value and re-renders all
panels, the numeric inputs, and the sample strip. They are global, so a drag in one
panel moves them everywhere.

### Sample strip (bottom)
- Editable text field, default `abcdefghijklmnopqrstuvwxyz`.
- **Size** slider and **letter-spacing** slider.
- Renders each character's merged glyph outline (reusing `mergedPaths`) inline,
  positioned by advance width + letter-spacing. Undesigned characters render faint
  or blank. Re-renders live on any glyph/guide/angle edit.

## Data flow

1. User edits in a panel → mutates that glyph's `cells` and/or guides (shared or
   local per the resolution table).
2. Edit triggers: re-render the editing panel, re-render the other two panels (in
   case they share the changed guide/angle), re-render the sample strip, and
   schedule a debounced `localStorage` autosave.
3. Switching a panel's letter flushes the current glyph into `GLYPHS` first, then
   loads the target.

## Export details

### Project JSON
Serialize `{ version, angles:{K,oAngle,dAngle}, metrics, sharedGuides:{hG,oG,dG},
nextId, glyphs: { char: { cells, splitOn, detached, localGuides } } }`.
Load validates and rehydrates, preserving guide IDs.

### .otf compile (opentype.js, already loaded)
- For each designed glyph: collect merged outline polygons (existing
  `mergedPaths` / `cellRings` logic).
- Transform canvas coords → font em units: flip Y about `baseline`, scale by
  `unitsPerEm / (cap - baseline)` (target `unitsPerEm = 1000`).
- Build an `opentype.Path` (moveTo/lineTo/close per ring). Winding from the
  existing merge already yields outer/hole opposite orientation → correct nonzero
  fill in the font.
- Advance width: per-glyph override if set, else glyph right-extent + global
  sidebearing.
- Assemble `new opentype.Font({familyName, styleName, unitsPerEm, ascender,
  descender, glyphs:[.notdef, ...designed]})` and `font.download()`.

## Reuse from source tool

Kept largely intact and shared across panels:
- Guide families `FAM`, `intersect`, `corners`, cell model, snap helpers.
- Fill/erase/select, subdivide.
- **Undo: one global history** spanning all panels, the shared guides, angles,
  metrics, and the alphabet store. Any edit in any panel pushes one global
  snapshot; undo reverts the most recent change regardless of which panel made it.
- SVG path import (`importSVG`) and font→cells (`placefont`).
- Edge-merge exporter geometry (`mergedPaths`, `groupLoops`) — feeds both .otf and
  sample rendering.

## Resolved decisions

- **Undo:** one global history across all panels (see Reuse section).
- **Metric frame lines:** draggable on-canvas, mirrored to numeric inputs (see
  Editor panel section).

## Open questions deferred to the implementation plan

- Exact punctuation set in the dropdown.
