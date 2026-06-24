# Corner Rounding + Guide Collision — Design Spec

**Date:** 2026-06-23
**File:** `loneranger-typesetter.html`
**Branch:** `corner-rounding` (from `type-tool` @ 17a2ebc)

Two features added to the typesetter.

## Feature A — Guide collision (min-gap push)

Within each guide family (horizontal `h`, oblique `o`, angled `d`), guides can never cross.
Dragging one toward a neighbor **pushes** the neighbor, cascading, so a **minimum gap** is
always preserved.

- **Uniform rule:** ALL guides in a family push and are pushed equally — including the
  metric guides (baseline/cap/ascender/descender). Dragging a fill guide into the baseline
  shoves the baseline. (User decision.)
- **Default gap:** `minGap` initialized at boot to `(metricY('baseline') − metricY('cap')) / 16`
  (cap height ÷ 16). Stored as an editable number (toolbar input); not auto-recomputed after
  init. Persisted.
- **Units:** the gap is enforced in each family's native scalar (h: `v`=y; o: `v`=c; d: `px`).
  Same `minGap` number in all three. (Visual gap varies slightly with angle for o/d; v1
  accepts this — documented.)
- **Scope of push:** operates on the active glyph's EFFECTIVE family set (so split/detached
  glyphs push within their own visible guides; non-split pushes the shared set, affecting all
  panels — consistent with shared guides). Guide objects are shared by reference, so mutating
  scalars writes through.
- **Undo:** drag already snapshots at start (`pushHistory` in `startDrag`); all pushed guides
  revert together.

### Cascade algorithm (per family, dragged guide `g`, gap `G`, scalar accessor `scal`)
```
const fam = [...EFF[family]];          // slice of the effective family array (shared refs)
fam.sort((a,b)=>scal(a)-scal(b));
const i = fam.indexOf(g);
for(let j=i+1;j<fam.length;j++) if(scal(fam[j]) < scal(fam[j-1])+G) setScal(fam[j], scal(fam[j-1])+G);
for(let j=i-1;j>=0;j--)         if(scal(fam[j]) > scal(fam[j+1])-G) setScal(fam[j], scal(fam[j+1])-G);
```
Runs in `onDrag` after the dragged guide's value is set (post-snap).

## Feature B — Corner rounding

Click a corner of a filled cell to toggle it between sharp and a rounded fillet. Four
sliders set the radius by corner POSITION.

- **Click target:** a cell-corner vertex; rounds **that cell's corner only** (two cells
  meeting at a point round independently). Scope: single-ring fill cells (guide-bound or free
  polygons). Imported glyph/compound cells (multi-ring) are excluded.
- **Four sliders:** global `cornerRadius = {tl,tr,bl,br}` (canvas units, default 60). A rounded
  corner's radius is chosen by classifying the vertex against its cell's centroid:
  `y<cy`→top, `x<cx`→left (canvas y-down) ⇒ tl/tr/bl/br.
- **Stored:** per cell `cell.rounded` = array of corner indices (into the cell's ring). Rides
  live as guides move (indices, not coordinates). Already inside the cell object, so it is
  snapshotted/serialized with the glyph automatically. `cornerRadius` global joins
  `snapshot`/`serializeStore`/`applyProject`.
- **Mode + handles:** new toolbar mode `round` (button `◜ Round corner`). In that mode, every
  filled single-ring cell shows small corner handles (hollow = sharp, filled = rounded);
  click within radius toggles. Other modes hide the handles.

### Fillet geometry
For a rounded vertex `cur` with neighbors `prev`,`next`:
`t = min(r, 0.5·|prev−cur|, 0.5·|cur−next|)` where `r=cornerRadius[classify(cur)]`;
`p_in = cur + unit(prev−cur)·t`, `p_out = cur + unit(next−cur)·t`.
Emit `L p_in  Q cur p_out` (quadratic Bézier through the original corner). Sharp vertices emit `L cur`.

Two builders:
- `cornerPathStr(ring, roundedSet)` → SVG path-data string (canvas, sample strip, SVG export).
- `cornerLoopPts(ring, roundedSet, steps)` → flattened `[x,y]` loop (OTF export); each `Q`
  is subdivided into `steps` segments.

### Rendering & export
- **Canvas / sample strip:** each simple cell drawn via `cornerPathStr(ring, cell.rounded)`.
  (Canvas already draws cells individually.)
- **SVG + OTF export — per-cell when rounded:** if a glyph has ANY rounded corner
  (`glyphHasRounding(g)`), emit ALL its simple cells as individual contours
  (`cornerPathStr` / `cornerLoopPts`) instead of the edge-merge; otherwise use the existing
  merge (`mergedPaths`/`mergedLoops`) unchanged. Compound/glyph cells always stay as today
  (evenodd, no rounding). The nonzero union of per-cell contours reads as one solid
  letterform with rounded corners; counters fall out as unfilled regions.
  Trade-off: rounded glyphs export per-cell contours (more path data) rather than one fused
  outline — visually identical for fill, and keeps "what you see = what exports."

## State plumbing summary
Add to `snapshot` / `serializeStore` / `applyProject` (with default-merge for old saves):
`cornerRadius` (object) and `minGap` (number). `cell.rounded` rides inside cells already.

## Out of scope (v1)
- Rounding imported glyph outlines / compound cells.
- Per-corner independent radius (radius is by position).
- A fused (merged) rounded outline — per-cell contours instead.
- Metric guides as walls (chose uniform push).
- Auto-recompute of `minGap` when cap/baseline change (init only).
