# Spacing & Kerning — Design Spec

**Date:** 2026-06-23
**File:** `loneranger-typesetter.html`
**Branch:** `kerning` (from `master` @ ffa3763)

Replace the uniform per-glyph advance with real per-glyph sidebearings plus flat pair
kerning, edited visually in the sample strip, exported to `.otf` (advance widths solid;
kern pairs best-effort).

## Current state (what this changes)
The sample strip and `buildFont()` give every glyph `metrics.advance` and lay glyphs out
`penX += metrics.advance + track`. There is no per-glyph width and no kerning. `metrics`
holds `sidebearing` (a global value, currently only used as the strip's left margin).

## Spacing model
Each glyph's advance becomes `LSB + inkWidth + RSB`:
- **inkWidth** = horizontal extent of the glyph's actual ink, from its drawn outline.
- **LSB / RSB**: default to the global `metrics.sidebearing`; a glyph may carry manual
  overrides `g.lsb` / `g.rsb` (numbers, in canvas units) in its glyph record.
- **Empty glyphs** (space, or a char with no ink): advance = `metrics.advance` (fixed),
  no outline drawn.

### Helpers
```
function glyphInk(ch){
  // bbox over the glyph's drawn outline points (reuse the same geometry as export, so ink
  // matches what's rendered, incl. corner rounding). Returns {minx,maxx,width} or null if empty.
}
function glyphSpacing(ch){
  const g=GLYPHS.get(ch), ink=glyphInk(ch);
  if(!ink) return {empty:true, advance:metrics.advance, lsb:0, rsb:0, ink:null};
  const lsb=(g&&typeof g.lsb==='number')?g.lsb:metrics.sidebearing;
  const rsb=(g&&typeof g.rsb==='number')?g.rsb:metrics.sidebearing;
  return {empty:false, lsb, rsb, ink, advance: lsb+ink.width+rsb};
}
```
`glyphInk` should derive points from the glyph's outline loops (the same per-cell rounded /
merged geometry the exporters use), so ink bounds honor rounding and effGuides.

## Kerning
Global `const kerning = {}` keyed by the two characters concatenated (`"AV"`, `"To"`),
value in canvas units (typically negative = tighter). Applied between an ordered pair.

## Sample-strip layout (renderSample)
For each character index `i` in the sample text (`prev` = previous char):
1. If `i>0`, apply kern: `penX += kerning[prev+ch] || 0`.
2. `sp = glyphSpacing(ch)`.
3. If `sp.empty`: draw nothing; `penX += sp.advance + track`.
4. Else: draw the glyph translated by `tx = penX + sp.lsb - sp.ink.minx` (left ink lands at
   `penX + LSB`); `penX += sp.advance + track`.
The global `track` slider stays as an extra uniform preview spacing. Build a per-glyph
placement list `[{i, ch, x0, x1, inkx0, inkx1}]` during render for hit-testing.

## Visual editing (interactive sample strip)
The sample `<svg>` gets `tabindex="0"` and a click handler:
- **Click on a glyph** (cursor x within its advance box) → `sampleSel = {type:'glyph', i, ch}`;
  focus the SVG; highlight the glyph.
- **Click in the gap between two glyphs** → `sampleSel = {type:'pair', left, right}` (the
  chars on each side); highlight the gap.
- A status line shows the selection and its live value (LSB/RSB or kern).

Keydown on the focused sample SVG (NOT the text input — so typing still works there):
- **→ / ←**: glyph → RSB ±STEP (space after); pair → kern ±STEP.
- **Shift+→ / ←**: glyph → LSB ±STEP (space before). (Pairs ignore Shift.)
- **Backspace/Delete**: glyph → clear `g.lsb`/`g.rsb` (back to auto); pair → delete the kern entry.
- `STEP = 5` canvas units (Alt = 1 for fine). Each keydown `pushHistory()` once, then
  `renderAll()` + `scheduleSave()`.

Overrides write `g.lsb`/`g.rsb` on the glyph record; kern writes/deletes `kerning[left+right]`.

## Export (buildFont)
- **Advance widths (solid):** per glyph, `advanceWidth = round(glyphSpacing(ch).advance * s)`
  (s = canvas→em scale). Position each glyph's path so its left ink sits at LSB: per-glyph
  x-offset `xoff = (sp.lsb - sp.ink.minx)`, i.e. `X(x) = (x + xoff) * s` for that glyph.
  Empty glyphs: `advanceWidth = round(metrics.advance * s)`, no path. (Today buildFont uses a
  single global advance and `X=x=>x*s`; make advance and the x-offset per-glyph.)
- **Kern pairs (best-effort):** attempt to emit a `kern` table via opentype.js — build glyph
  index pairs → em values and set whatever opentype.js's writer accepts. If 1.3.4's writer
  cannot emit kerning, leave a clear comment, keep kerning in-app + JSON, and report it. Do
  NOT block the advance-width export on this.

## Persistence
- Per-glyph `lsb`/`rsb`: add to the EXPLICIT `serializeStore` per-glyph object
  (`{cells,split,detached,localGuides,lsb,rsb}`) and to `applyProject`'s glyph
  reconstruction. (snapshot already deep-clones whole glyph records.)
- Global `kerning`: add to `snapshot` (clone), `undo` (replace), `serializeStore`,
  `applyProject` (default `{}`). Keep the live `kerning` object identity stable on
  undo/apply by clearing+assigning, not rebinding, OR rebind is fine since few hold it —
  prefer `clear+assign` to match `metrics`/`cornerRadius` patterns where a reference is held.

## Out of scope (v1)
Class/contextual kerning, GPOS features, vertical metrics, auto-kern suggestions. Manual
sidebearing overrides are absolute values (not deltas). `track` stays preview-only.
