# Spacing & Kerning — Implementation Plan

> Implement task-by-task on branch `kerning`. Each ends with a `node --check`-verifiable
> deliverable; browser/visual checks are human-deferred (single `file://` HTML, no runner).

**Goal:** Per-glyph sidebearings (auto-from-ink + manual override) + flat pair kerning,
edited visually in the sample strip, exported to `.otf` (advance widths solid; kern best-effort).

**Spec:** docs/superpowers/specs/2026-06-23-spacing-and-kerning-design.md

## Global Constraints
- Single file `loneranger-typesetter.html`; no build; `file://`-runnable; one inline `<script>`; no ES modules; opentype.js via CDN.
- Reuse engine: `glyphRings`/`glyphOutlineData`, `effGuides`, `cellRings`, `mergedLoops`, `buildFont`, `metrics`, `GLYPHS`, `renderSample`.
- Verify each task: extract inline `<script>` → temp `.js` → `node --check`; report cmd+output. Do NOT fabricate browser runs.
- Commit only `loneranger-typesetter.html` (`.superpowers/` gitignored).
- `metrics` keeps `sidebearing` (global default SB) + `advance` (empty-glyph width).

---

### Task 1 — Spacing model: ink bounds, per-glyph advance, kern map, sample layout, persistence

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:** `glyphInk(ch)`, `glyphSpacing(ch)`, `const kerning={}`; renderSample
uses per-glyph advance + kerning; lsb/rsb + kerning persisted.

- [ ] **Step 1: ink + spacing helpers (module scope, near glyphRings/glyphOutlineData)**
```js
function glyphInk(ch){
  // reuse the OTF outline geometry (canvas coords, honors rounding + effGuides)
  const g=GLYPHS.get(ch); if(!g) return null;
  const loops = glyphRings(ch); if(!loops || !loops.length) return null;
  let mn=Infinity, mx=-Infinity;
  for(const lp of loops) for(const p of lp){ if(p[0]<mn)mn=p[0]; if(p[0]>mx)mx=p[0]; }
  if(!isFinite(mn)) return null;
  return {minx:mn, maxx:mx, width:Math.max(0, mx-mn)};
}
function glyphSpacing(ch){
  const g=GLYPHS.get(ch), ink=glyphInk(ch);
  if(!ink) return {empty:true, advance:metrics.advance, lsb:0, rsb:0, ink:null};
  const lsb=(g&&typeof g.lsb==='number')?g.lsb:metrics.sidebearing;
  const rsb=(g&&typeof g.rsb==='number')?g.rsb:metrics.sidebearing;
  return {empty:false, lsb, rsb, ink, advance: lsb+ink.width+rsb};
}
const kerning={};
```
NOTE: confirm `glyphRings(ch)` works for any `ch` (it reads `glyphOf(ch)`/effGuides) and is
defined ABOVE these helpers, or hoisted (function declaration). If `glyphRings` is defined
below, move these helpers after it or rely on hoisting.

- [ ] **Step 2: rewrite renderSample layout to use per-glyph spacing + kerning**
Find `renderSample`. Replace the per-char placement so that, walking the sample text with
`prev` tracking the previous char:
```js
let penX = /* existing left start, e.g. */ metrics.sidebearing;
let prev=null;
for(const ch of text){
  if(prev!==null) penX += (kerning[prev+ch]||0);
  const sp=glyphSpacing(ch);
  if(sp.empty){ penX += sp.advance + track; prev=ch; continue; }
  const tx = penX + sp.lsb - sp.ink.minx;
  // draw this glyph's outline paths translated by (tx, 0)  [reuse existing per-glyph path build]
  penX += sp.advance + track;
  prev=ch;
}
```
Keep the existing outline-path construction per glyph (the `glyphOutlineData`/`cornerPathStr`
strings); only the x-translate and advance change. Preserve the size-slider viewBox logic and
`totalW = penX + metrics.sidebearing`.

- [ ] **Step 3: persistence of lsb/rsb + kerning**
- `serializeStore`: change the per-glyph object to include `lsb` and `rsb`:
  `glyphs[c]={cells:g.cells,split:g.split,detached:g.detached,localGuides:g.localGuides,lsb:g.lsb,rsb:g.rsb}`
  and add `kerning` to the returned top-level object.
- `applyProject`: in the glyph reconstruction add `lsb:g.lsb, rsb:g.rsb`; and restore
  kerning: `if(p.kerning){ for(const k in kerning) delete kerning[k]; Object.assign(kerning,p.kerning); }`.
- `snapshot`: add `kerning:JSON.parse(JSON.stringify(kerning))` (glyph lsb/rsb already
  ride in the whole-glyph clone). `undo`: restore via `if(s.kerning){ for(const k in kerning) delete kerning[k]; Object.assign(kerning,s.kerning); }`.

- [ ] **Step 4: node --check + commit** `feat(typesetter): per-glyph sidebearing spacing + kern map in sample strip`.
Human-deferred: letters now auto-space by ink width in the strip; global sidebearing affects gaps; reload persists.

---

### Task 2 — Interactive sample strip: select + nudge sidebearings/kern

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:** `sampleSel` state, placement list from render, click + key handlers.

- [ ] **Step 1: emit a placement list during renderSample**
While laying out (Task 1 loop), push `place.push({i, ch, x0:penXbeforeGlyph, x1:penXafter, isEmpty})`
into a module/closure-level `samplePlace=[]` (reset each render). `x0` = pen position where
this glyph's slot starts (before its advance, after kern), `x1` = after its advance. Store the
gap regions implicitly (between `place[k].x1` and `place[k+1].x0`+kern, or simply: a click
between two glyph boxes = the pair). Keep it simple: a click maps to the nearest glyph box it
falls inside (glyph select) else the boundary between the two nearest boxes (pair select).

- [ ] **Step 2: selection state + highlight + status line**
```js
let sampleSel=null; // {type:'glyph', i, ch} | {type:'pair', left, right}
```
Add a `#sampleStatus` element (in the sample bar). In renderSample, after layout, if
`sampleSel` draw a highlight (a translucent rect over the selected glyph's box, or a vertical
marker at the selected pair boundary) and set `#sampleStatus` text to the live value
(e.g. `a  LSB 80  RSB 80` or `kern "AV" -40`).

- [ ] **Step 3: click handling on the sample svg**
Give `#sampleSvg` `tabindex="0"`. On `pointerdown`, convert clientX to sample-svg user X
(account for the viewBox scale used by the size slider), find the placement: if X within a
glyph box → glyph select; else pick the boundary between the two surrounding boxes → pair
select (left=that left glyph's ch, right=the right glyph's ch). Set `sampleSel`, focus the
svg, `renderSample()` (re-highlight). Clicking empty area clears selection.

- [ ] **Step 4: keyboard nudge (on the focused sample svg only)**
Add a `keydown` listener on `#sampleSvg` (so it does NOT fire while typing in `#sampleText`):
```js
sampleSvg.addEventListener('keydown', e=>{
  if(!sampleSel) return;
  const STEP = e.altKey?1:5;
  let handled=true;
  if(e.key==='ArrowRight'||e.key==='ArrowLeft'){
    const dir = e.key==='ArrowRight'?1:-1;
    pushHistory();
    if(sampleSel.type==='glyph'){
      const g=glyphOf(sampleSel.ch);
      if(e.shiftKey){ g.lsb=(typeof g.lsb==='number'?g.lsb:metrics.sidebearing)+dir*STEP; }
      else          { g.rsb=(typeof g.rsb==='number'?g.rsb:metrics.sidebearing)+dir*STEP; }
    } else {
      const key=sampleSel.left+sampleSel.right;
      kerning[key]=(kerning[key]||0)+dir*STEP;
    }
    renderAll(); scheduleSave();
  } else if(e.key==='Backspace'||e.key==='Delete'){
    pushHistory();
    if(sampleSel.type==='glyph'){ const g=glyphOf(sampleSel.ch); delete g.lsb; delete g.rsb; }
    else delete kerning[sampleSel.left+sampleSel.right];
    renderAll(); scheduleSave();
  } else handled=false;
  if(handled) e.preventDefault();
});
```
(`glyphOf` is the controller-level accessor; the sample handlers live at controller scope
where `glyphOf`/`kerning`/`metrics`/`pushHistory`/`renderAll`/`scheduleSave` are visible.)

- [ ] **Step 5: node --check + commit** `feat(typesetter): visual sidebearing + kern editing in sample strip`.
Human-deferred: click a letter → arrows nudge its spacing; click a gap → arrows kern the pair; Backspace resets; undo works; typing in the text field still moves the caret (arrows there unaffected).

---

### Task 3 — Export: per-glyph advance widths in .otf + best-effort kern table

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:** buildFont per-glyph advance + x-offset; optional kern emission.

- [ ] **Step 1: per-glyph advance + x-position in buildFont**
In `buildFont`, today every glyph uses one advance and `X=x=>x*s`. Change per glyph:
```js
const sp=glyphSpacing(ch);
const xoff = sp.empty?0:(sp.lsb - sp.ink.minx);     // shift left ink to LSB
const GX = x => (x + xoff)*s;                         // per-glyph x transform (Y unchanged)
const advanceWidth = Math.round((sp.empty?metrics.advance:sp.advance)*s);
// build the opentype.Path using GX for x and the existing Y(y) for y; pass advanceWidth.
```
Empty/undesigned glyphs: still emit the glyph entry with `advanceWidth` and no contours (or
skip — match current behavior, but ensure designed glyphs get correct advance). Keep `.notdef`.

- [ ] **Step 2: best-effort kern table**
After building `glyphs`, attempt to attach kerning. opentype.js 1.3.4 write support for `kern`
is uncertain — implement defensively:
```js
// Map our char-pair kerning to glyph-index pairs in em units; attempt opentype kern.
// If opentype.js exposes no writable kern path, skip (kerning stays in-app + JSON).
try {
  const idx = {}; glyphs.forEach((gl,n)=>{ if(gl.unicode) idx[String.fromCodePoint(gl.unicode)]=n; });
  const pairs = {};
  for(const k in kerning){ const L=idx[k[0]], R=idx[k[1]]; if(L!=null&&R!=null) pairs[L+','+R]=Math.round(kerning[k]*s); }
  if(Object.keys(pairs).length) font.kerningPairs = pairs;   // opentype reads this; writer may ignore
} catch(e){ /* kerning stays in-app/JSON only */ }
```
Add a one-line note in the export (or a console.info) if kerning pairs exist but may not be
embedded, so the human knows to verify. Do NOT block download on kern support.

- [ ] **Step 3: node --check + commit** `feat(typesetter): per-glyph advance widths in .otf (kern best-effort)`.
Human-deferred: export `.otf`, install → letters carry their designed widths (i/m differ); verify whether kern pairs survive (report finding).

## Self-Review checklist
- `glyphInk` returns null for empty glyphs; spacing falls back to `metrics.advance`.
- renderSample: kern applied between pairs; glyph placed by `lsb - ink.minx`; track preserved.
- lsb/rsb in BOTH serializeStore and applyProject glyph objects; kerning in all 4 state sites.
- Arrow nudges only on focused sample svg (not while typing in `#sampleText`).
- buildFont per-glyph advance + x-offset; empty glyphs handled; .notdef intact.
- Un-spaced (all-default) glyphs: advance = 2·sidebearing + inkWidth — a deliberate change from the old uniform advance (expected; the whole point).
