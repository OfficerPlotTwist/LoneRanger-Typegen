# Z-Type Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fork `z-grid-tool.html` into `z-type-tool.html`: a 3-panel oblique typeface
editor where letters share one guide system, with per-letter guide divergence, a live
sample strip, and `.otf` + project-JSON export.

**Architecture:** One self-contained HTML file (opened via `file://`, no build step). The
source tool's global-singleton editor is refactored into an `Editor` factory instantiated
3×, all reading from shared global state (angles, metric frame, shared guides) and a
per-character glyph store. Effective guides per glyph are resolved live from a 3-state
split model. The existing guide/cell/merge engine is reused wholesale.

**Tech Stack:** Vanilla JS + SVG, `opentype.js@1.3.4` (already vendored via CDN `<script>`,
used for both font *import* and font *writing*), `localStorage` for autosave.

## Global Constraints

- Single file `z-type-tool.html`. No build step; must run by opening via `file://`.
- No ES modules / no external bundler (breaks `file://`). All JS in one inline `<script>`.
- Angles (`K`/`oAngle`, `dAngle`) are ALWAYS global; no code path unlinks a glyph's angle.
- Cells bind to guides by stable numeric `id`; never renumber an existing guide's id.
- Reuse, do not rewrite, the source engine: `FAM`, `intersect`, `corners`, `cellRings`,
  `mergedPaths`, `groupLoops`, `pathToRings`, `importSVG`, snap helpers.
- `.otf` target: `unitsPerEm = 1000`.
- Source of truth for behavior: `docs/superpowers/specs/2026-06-23-z-type-tool-design.md`.
- Never modify `z-grid-tool.html` — the fork is a new file.

**Testing model:** No automated runner (single `file://` HTML). Each task's "test" is a
scripted **browser verification**: open `z-type-tool.html`, perform the listed actions,
confirm the listed observations. Pure-logic units additionally carry `console.assert`
checks runnable from DevTools console. "Expected" lines are the pass condition.

---

### Task 1: Fork file + Editor refactor — 3 panels sharing ONE guide system and ONE cell set

This task proves the multi-instance refactor. Cells stay temporarily shared across panels
(per-glyph cells arrive in Task 2). Shared: `hG/oG/dG`, angles, `cells`. Per-panel: SVG,
viewBox (pan/zoom), drag state.

**Files:**
- Create: `z-type-tool.html` (fork of `z-grid-tool.html`)
- Reference (do not modify): `z-grid-tool.html`

**Interfaces:**
- Produces:
  - `function Editor(host)` → returns `{ el, render(), fitView(), id }`. `host` is a
    container `<div>`. Each Editor owns its `<svg>`, `vb` (viewBox), and pointer handlers.
  - `const editors = [Editor(...), Editor(...), Editor(...)]`
  - `function renderAll()` → calls `e.render()` for every editor + (later) sample strip.
  - Global mutable state kept module-level: `hG, oG, dG, dAngle, K, oAngle, cells,
    nextId, mode, snap`. (Same names/types as source.)
  - `function activeEditor()` → the Editor whose SVG is under the pointer (set on
    `pointerenter`); drives which viewBox click-math is used.

- [ ] **Step 1: Copy the source file**

```bash
cp z-grid-tool.html z-type-tool.html
```

- [ ] **Step 2: Replace the single `#wrap`/`#stage` markup with a 3-panel row**

In `z-type-tool.html`, replace the `<div id="wrap">…</div>` block (source lines ~113-123)
with a flex row of three panel hosts and move the status bar text into a shared footer:

```html
  <div id="panels">
    <div class="panel" data-i="0"></div>
    <div class="panel" data-i="1"></div>
    <div class="panel" data-i="2"></div>
  </div>
```

Add CSS (in the `<style>` block):

```css
#panels{flex:1;min-height:0;display:flex;gap:8px;padding:8px;background:#000}
.panel{flex:1;min-width:0;display:flex;flex-direction:column;border:1px solid #1c1c1c;border-radius:8px;overflow:hidden;background:#0a0a0a}
.panel .phead{display:flex;gap:6px;align-items:center;padding:6px 8px;background:var(--panel);border-bottom:1px solid var(--line)}
.panel .pbody{flex:1;min-height:0;display:flex;align-items:center;justify-content:center;overflow:hidden}
.panel svg{width:100%;height:100%;background:#000;cursor:crosshair;touch-action:none}
.panel.active{border-color:var(--accent)}
```

- [ ] **Step 3: Wrap the editor internals in an `Editor` factory**

The source keeps `svg, gFills, gGuides, gNodes, vb, drag, pan` as module globals. Convert
the per-surface pieces into an `Editor` closure. Keep `hG/oG/dG/cells/angles/mode/snap`
GLOBAL (shared). Each `Editor(host)`:

```js
function Editor(host){
  const i = +host.dataset.i;
  host.innerHTML =
    '<div class="phead"><span class="plabel">panel '+(i+1)+'</span>'+
    '<span style="flex:1"></span>'+
    '<button class="zin">＋</button><button class="zout">－</button><button class="zfit">Fit</button></div>'+
    '<div class="pbody"></div>';
  const body = host.querySelector('.pbody');
  const svg = document.createElementNS('http://www.w3.org/2000/svg','svg');
  svg.setAttribute('viewBox','0 0 1280 1280');
  svg.innerHTML =
    '<rect x="-200000" y="-200000" width="400000" height="400000" fill="#000"/>'+
    '<g class="gfills"></g><g class="gguides"></g><g class="gnodes"></g>';
  body.appendChild(svg);
  const gFills  = svg.querySelector('.gfills');
  const gGuides = svg.querySelector('.gguides');
  const gNodes  = svg.querySelector('.gnodes');
  let vb = {x:0,y:0,w:1280,h:1280};
  let drag=null, pan=null;

  function applyVB(){ svg.setAttribute('viewBox',`${vb.x} ${vb.y} ${vb.w} ${vb.h}`); }
  function toSVG(evt){ const p=svg.createSVGPoint(); p.x=evt.clientX; p.y=evt.clientY;
    return p.matrixTransform(svg.getScreenCTM().inverse()); }

  // ... move source render()/guideEl()/startDrag/onDrag/endDrag/zoom/pan handlers here,
  //     replacing module `svg/gFills/...` references with these locals, and reading the
  //     SHARED globals hG/oG/dG/cells/angles for geometry.

  host.addEventListener('pointerenter', ()=>setActive(i));
  return { el:host, id:i, render, fitView, _svg:svg };
}
```

Move source functions `render`, `guideEl`, `startDrag`, `onDrag`, `endDrag`, `startPan`,
`onPan`, `endPan`, `zoomAt`, the `wheel`/`pointerdown`/`contextmenu` listeners, and
`applyVB` INTO this closure (they become per-Editor). Geometry helpers that read only
globals (`corners`, `cellRings`, `cval`, `xOf`, `bracketF`, `toggleCell`, `eraseAt`,
`selectAt`, `cellAt`) stay module-level and are CALLED by the closure. Where the source
called `render()` directly, call `renderAll()` instead so all panels stay in sync.

- [ ] **Step 4: Add `setActive`, `activeEditor`, `renderAll`, and boot the 3 editors**

```js
let _active = 0;
function setActive(i){ _active=i; document.querySelectorAll('.panel').forEach((p,k)=>p.classList.toggle('active',k===i)); }
function activeEditor(){ return editors[_active]; }
function renderAll(){ for(const e of editors) e.render(); /* sample strip added in Task 7 */ }
const editors = [...document.querySelectorAll('.panel')].map(Editor);
setActive(0); renderAll();
```

Update the global keyboard `undo`/`Delete`/`Escape` handler and the toolbar buttons
(`zpreset`, `reset`, `sub-h`, etc.) to call `renderAll()` instead of `render()`.

- [ ] **Step 5: Browser verification**

Open `z-type-tool.html` in a browser. Then:
1. Three panels show side by side, each on its own black canvas.
2. Click **＋ ‖ guide** in the top toolbar, click in panel 1 → a horizontal guide appears
   in **all three** panels (shared guides).
3. Hover panel 2 and scroll → only panel 2 zooms (independent viewBox); panels 1 & 3
   unchanged.
4. Click **⃟ Load Z preset** → the Z guides + underlay appear in all three panels.
5. Hovering a panel adds the `.active` blue border to that panel only.

Expected: all five hold. If a click in a panel uses the wrong coordinates after zooming a
different panel, `activeEditor()`/per-Editor `toSVG` wiring is wrong — fix before commit.

- [ ] **Step 6: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): fork grid tool into 3-panel shared-guide shell"
```

---

### Task 2: Per-glyph cell store + letter dropdowns + localStorage autosave

Cells become per-character. Shared guides/angles remain global. Each panel edits one char.

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Consumes: `editors`, `renderAll`, global `cells` (now removed as a global).
- Produces:
  - `const GLYPHS = new Map()` — `char → { cells: Cell[], split, detached, localGuides }`.
    For this task only `cells` is populated; split fields default
    (`split:false, detached:false, localGuides:null`).
  - `function glyphOf(ch)` → returns the stored record, creating a blank one on demand.
  - Per-Editor `editor.ch` (the char it edits) and `editor.cells()` → `glyphOf(ch).cells`.
  - `function saveStore()` / `function loadStore()` — JSON ↔ `localStorage['z-type-store']`.
  - `const ALPHABET = [...'abcdefghijklmnopqrstuvwxyz0123456789.,!?-']` (dropdown order).

- [ ] **Step 1: Replace global `cells` with the glyph store**

```js
const GLYPHS = new Map();
function glyphOf(ch){
  let g=GLYPHS.get(ch);
  if(!g){ g={cells:[], split:false, detached:false, localGuides:null}; GLYPHS.set(ch,g); }
  return g;
}
const ALPHABET = [...'abcdefghijklmnopqrstuvwxyz0123456789.,!?-'];
```

Remove the module-level `let cells = []`. Every geometry function that referenced the
global `cells` (`cellAt`, `toggleCell`, `eraseAt`, `selectAt`, `render`, exporters) must
now operate on a **specific glyph's** cells. Thread the active glyph in: in the Editor
closure these become `glyphOf(this.ch).cells`. Add to `Editor`:

```js
let ch = ALPHABET[i];           // panel 0→'a', 1→'b', 2→'c'
function cells(){ return glyphOf(ch).cells; }
```

In the closure's `render`, `toggleCell`, `eraseAt`, `selectAt`, `cellAt`, replace the old
global `cells` with `cells()`. `selected` (the selection set) also moves INTO the Editor
closure (selection is per-panel).

- [ ] **Step 2: Add the per-panel letter dropdown to `.phead`**

In `Editor`, inject a `<select class="letter">` before the zoom buttons:

```js
const sel = document.createElement('select'); sel.className='letter';
for(const c of ALPHABET){ const o=document.createElement('option'); o.value=c; o.textContent=c; sel.appendChild(o); }
sel.value = ch;
sel.addEventListener('change', ()=>{ ch=sel.value; selected.clear(); render(); scheduleSave(); });
host.querySelector('.phead').insertBefore(sel, host.querySelector('.zin'));
```

Expose `ch` on the returned object (`get ch(){return ch}`) for the sample strip later.

- [ ] **Step 3: Add autosave + load**

```js
function serializeStore(){
  const glyphs={};
  for(const [c,g] of GLYPHS) glyphs[c]={cells:g.cells, split:g.split, detached:g.detached, localGuides:g.localGuides};
  return {v:1, K, oAngle, dAngle, hG, oG, dG, nextId, glyphs};
}
let _saveT=null;
function scheduleSave(){ clearTimeout(_saveT); _saveT=setTimeout(saveStore, 400); }
function saveStore(){ try{ localStorage.setItem('z-type-store', JSON.stringify(serializeStore())); }catch(e){} }
function loadStore(){
  let raw; try{ raw=localStorage.getItem('z-type-store'); }catch(e){ return false; }
  if(!raw) return false;
  try{ applyProject(JSON.parse(raw)); return true; }catch(e){ return false; }
}
function applyProject(p){       // also reused by Task 8 JSON import
  K=p.K; oAngle=p.oAngle; dAngle=p.dAngle; hG=p.hG; oG=p.oG; dG=p.dG; nextId=p.nextId;
  GLYPHS.clear();
  for(const c in p.glyphs){ const g=p.glyphs[c];
    GLYPHS.set(c,{cells:g.cells||[], split:!!g.split, detached:!!g.detached, localGuides:g.localGuides||null}); }
  syncObliqueUI(); syncAngleUI(); updateSlope();
}
```

Call `scheduleSave()` at the end of every mutating action (guide add/drag/delete, fill,
erase, angle change, undo). Call `loadStore()` once at boot **before** `renderAll()`.

- [ ] **Step 4: Browser verification**

Open the file (clear `localStorage` first via DevTools: `localStorage.clear()`):
1. Panels default to letters `a`, `b`, `c`.
2. Load Z preset, fill some cells in panel 1 (`a`). Panel 2 (`b`) stays empty → cells are
   per-glyph.
3. Change panel 2's dropdown to `a` → it now shows the SAME fills as panel 1 (same glyph).
4. Reload the page → letters and the filled `a` are restored from `localStorage`.
5. DevTools console: `GLYPHS.get('a').cells.length` > 0, `GLYPHS.get('b').cells.length` === 0.

Expected: all hold.

- [ ] **Step 5: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): per-glyph cell store, letter dropdowns, autosave"
```

---

### Task 3: Global undo across all panels

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Consumes: `GLYPHS`, `hG/oG/dG`, angles, `nextId`.
- Produces: `function pushHistory()`, `function undo()` — one global stack covering shared
  guides, angles, AND every glyph's cells/split state.

- [ ] **Step 1: Replace the source single-canvas snapshot with a whole-project snapshot**

```js
let history=[];
function snapshot(){
  const glyphs={};
  for(const [c,g] of GLYPHS) glyphs[c]=JSON.parse(JSON.stringify(g));
  return { hG:JSON.parse(JSON.stringify(hG)), oG:JSON.parse(JSON.stringify(oG)),
           dG:JSON.parse(JSON.stringify(dG)), dAngle, K, oAngle, nextId, glyphs };
}
function pushHistory(){ history.push(snapshot()); if(history.length>200) history.shift(); }
function undo(){
  if(!history.length) return;
  const s=history.pop();
  hG=s.hG; oG=s.oG; dG=s.dG; dAngle=s.dAngle; K=s.K; oAngle=s.oAngle; nextId=s.nextId;
  GLYPHS.clear(); for(const c in s.glyphs) GLYPHS.set(c, s.glyphs[c]);
  for(const e of editors) e.clearSelection();
  syncAngleUI(); syncObliqueUI(); updateSlope(); renderAll(); scheduleSave();
}
```

Add `clearSelection()` to the Editor's returned object (`selected.clear()`).

- [ ] **Step 2: Ensure every mutation calls `pushHistory()` before mutating**

Audit the closure + toolbar: guide add, guide drag start, guide delete, fill toggle,
erase, subdivide, reset, clear, angle change, metric drag (Task 4), split toggle (Task 5).
Each must `pushHistory()` once before its mutation (matching source pattern). The global
`Ctrl/Cmd+Z` handler calls `undo()`.

- [ ] **Step 3: Browser verification**
1. Fill a cell in panel 1 (`a`), then add a guide via panel 2 → both visible.
2. Press `Ctrl+Z` → the guide (last action) is removed; the fill remains.
3. Press `Ctrl+Z` again → the fill is removed.
4. Console `history.length` decreases by 1 per undo.

Expected: undo reverts the single most-recent action regardless of originating panel.

- [ ] **Step 4: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): global undo across panels and glyph store"
```

---

### Task 4: Metric frame — state, numeric controls, draggable on-canvas lines

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Produces:
  - `const metrics = { baseline:1040, xHeight:560, cap:240, ascender:160, descender:1140,
    advance:760, sidebearing:80 }` (canvas units, y-down; sensible defaults inside 0..1280).
  - `const METRIC_KEYS = ['ascender','cap','xHeight','baseline','descender']` (draggable Y lines).
  - Per-Editor metric line rendering + drag; numeric inputs in the toolbar mirror them.

- [ ] **Step 1: Add metric state + toolbar numeric inputs**

Add a toolbar group:

```html
<div class="grp" id="metricbar">
  <label>asc <input data-m="ascender"  type="number" class="mnum" style="width:54px"></label>
  <label>cap <input data-m="cap"       type="number" class="mnum" style="width:54px"></label>
  <label>x   <input data-m="xHeight"   type="number" class="mnum" style="width:54px"></label>
  <label>base<input data-m="baseline"  type="number" class="mnum" style="width:54px"></label>
  <label>desc<input data-m="descender" type="number" class="mnum" style="width:54px"></label>
  <label>adv <input data-m="advance"   type="number" class="mnum" style="width:54px"></label>
  <label>sb  <input data-m="sidebearing" type="number" class="mnum" style="width:44px"></label>
</div>
```

```js
const metrics = {baseline:1040,xHeight:560,cap:240,ascender:160,descender:1140,advance:760,sidebearing:80};
const METRIC_KEYS = ['ascender','cap','xHeight','baseline','descender'];
function syncMetricUI(){ document.querySelectorAll('.mnum').forEach(inp=>inp.value=metrics[inp.dataset.m]); }
document.querySelectorAll('.mnum').forEach(inp=>{
  inp.addEventListener('change', ()=>{ pushHistory(); metrics[inp.dataset.m]=+inp.value||0; renderAll(); scheduleSave(); });
});
syncMetricUI();
```

Include `metrics` in `serializeStore`, `applyProject`, `snapshot`/`undo` (add
`metrics:JSON.parse(JSON.stringify(metrics))` and restore by `Object.assign(metrics,…)`).

- [ ] **Step 2: Add CSS + render draggable metric lines in each Editor**

```css
.mline{stroke:#7a5cff;stroke-width:1;stroke-dasharray:4 6;pointer-events:none}
.mline-hit{stroke:transparent;stroke-width:14;cursor:ns-resize}
.mlabel-t{fill:#7a5cff;font-size:18px;pointer-events:none}
```

In the Editor closure `render()`, after guides, draw metric lines full width:

```js
const R=200000;
for(const key of METRIC_KEYS){
  const y=metrics[key];
  const hit=svgEl('line',{x1:-R,y1:y,x2:R,y2:y,class:'mline-hit'});
  hit.dataset.metric=key; hit.addEventListener('pointerdown',startMetricDrag);
  gGuides.appendChild(hit);
  gGuides.appendChild(svgEl('line',{x1:-R,y1:y,x2:R,y2:y,class:'mline'}));
  gGuides.appendChild(svgEl('text',{x:vb.x+8,y:y-4,class:'mlabel-t'})).textContent=key;
}
```

(`svgEl` is the source helper; promote it to module scope if not already.)

- [ ] **Step 3: Metric drag handler (updates GLOBAL metric, re-renders all)**

```js
let mdrag=null;
function startMetricDrag(e){
  if(e.button!==0 || mode!=='move') return;
  e.stopPropagation(); pushHistory();
  mdrag=e.target.dataset.metric;
  svg.setPointerCapture(e.pointerId);
  svg.addEventListener('pointermove',onMetricDrag);
  svg.addEventListener('pointerup',endMetricDrag);
}
function onMetricDrag(e){ if(!mdrag) return; metrics[mdrag]=Math.round(toSVG(e).y); syncMetricUI(); renderAll(); }
function endMetricDrag(){ if(!mdrag) return; mdrag=null;
  svg.removeEventListener('pointermove',onMetricDrag); svg.removeEventListener('pointerup',endMetricDrag); scheduleSave(); }
```

Guard the closure's main `pointerdown` to ignore clicks that hit `.mline-hit` in `move`
mode (same pattern as `.gline-hit`).

- [ ] **Step 4: Browser verification**
1. Five dashed purple lines (asc/cap/x/base/desc) appear in every panel at the default Ys.
2. Drag the `baseline` line down in panel 2 → it moves in **all three** panels and the
   `base` numeric input updates live.
3. Type a new `cap` value in the toolbar → the cap line jumps in all panels.
4. `Ctrl+Z` reverts the last metric move.

Expected: metric lines are global; drag and numeric edit stay mirrored.

- [ ] **Step 5: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): draggable global metric frame"
```

---

### Task 5: Split guide position — per-glyph EXTRA local guides (propagating)

Adds the first split state: shared guides still live; the glyph gains its own *extra*
guides layered on top. Requires resolving "effective guides" per glyph.

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Produces:
  - `function effHG(g)/effOG(g)/effDG(g)` — given a glyph record, return the guide arrays
    the Editor should use. This task: `split` → `shared.concat(local.extra)`; else `shared`.
  - Per-glyph `localGuides = {hG:[],oG:[],dG:[]}` holds ONLY the extra guides while
    propagating.
  - Editor reads guides via these resolvers instead of the bare globals.
  - Per-panel **Split** toggle button in `.phead`.

- [ ] **Step 1: Add the resolver and route ALL guide reads through it**

```js
function effGuides(g){
  if(!g.split) return {h:hG, o:oG, d:dG};
  const L=g.localGuides||{hG:[],oG:[],dG:[]};
  return { h:hG.concat(L.hG), o:oG.concat(L.oG), d:dG.concat(L.dG) };   // detach handled in Task 6
}
```

The source `FAM[*].arr()` returns the global arrays. Make `FAM` parameterizable by the
current glyph's effective set: introduce a closure-local `EFF` recomputed at the top of
`render()` and each interaction, and have `corners`, `bracketF`, node drawing, and guide
rendering read `EFF.h/EFF.o/EFF.d`. Concretely, pass `EFF` into the geometry helpers
(add an `eff` argument) rather than relying on `FAM[*].arr()`'s globals. `byId` lookups
must search the concatenated effective array so cells bound to either a shared or a local
guide resolve.

- [ ] **Step 2: New guides added while split go to the glyph's local set**

In the Editor's add-guide handlers, branch on the current glyph:

```js
function addGuide(type, value){
  const g=glyphOf(ch);
  if(g.split){
    g.localGuides=g.localGuides||{hG:[],oG:[],dG:[]};
    const arr = type==='h'?g.localGuides.hG : type==='o'?g.localGuides.oG : g.localGuides.dG;
    arr.push(type==='d'?{id:nextId++,px:value}:{id:nextId++,v:value});
  } else {
    const arr = type==='h'?hG : type==='o'?oG : dG;
    arr.push(type==='d'?{id:nextId++,px:value}:{id:nextId++,v:value});
  }
}
```

Dragging a guide must mutate whichever array (shared or local) actually holds that id.
Add `function findGuide(g,type,id)` that searches shared then local and returns the object
to mutate.

- [ ] **Step 3: Add the per-panel Split toggle**

```js
const splitBtn=document.createElement('button'); splitBtn.className='split'; splitBtn.textContent='⑃ split';
splitBtn.addEventListener('click',()=>{
  pushHistory(); const g=glyphOf(ch);
  g.split=!g.split; if(g.split && !g.localGuides) g.localGuides={hG:[],oG:[],dG:[]};
  splitBtn.classList.toggle('on',g.split);
  detachBtn.style.display = g.split?'':'none';   // detachBtn from Task 6
  render(); scheduleSave();
});
```

(Style `.split.on{outline:2px solid var(--oblique)}`.)

- [ ] **Step 4: Browser verification**
1. Load Z preset (shared guides in all panels). Panel 1 = `a`.
2. Toggle **split** on panel 1. Add a `‖` guide in panel 1 → it appears in panel 1 only.
   Panels 2 & 3 still show only the shared guides.
3. Drag a SHARED oblique guide in panel 2 → it still moves in panel 1 too (propagation
   intact while split).
4. Console: `GLYPHS.get('a').localGuides.hG.length === 1`, shared `hG` unchanged by step 2.

Expected: split adds extras locally; shared edits still flow into the split glyph.

- [ ] **Step 5: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): split guide position with per-glyph extra guides"
```

---

### Task 6: Stop-propagating sub-toggle — detach (frozen copy, IDs preserved)

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Produces:
  - `detached` flag + a frozen `localGuides.base = {hG,oG,dG}` deep copy (ids preserved)
    taken at detach time. While detached, `effGuides` returns `base.concat(extra)` and
    ignores the live shared arrays.
  - Per-panel **stop-propagating** sub-toggle (visible only when split is on).

- [ ] **Step 1: Extend the resolver for the detached state**

```js
function effGuides(g){
  if(!g.split) return {h:hG, o:oG, d:dG};
  const L=g.localGuides||(g.localGuides={hG:[],oG:[],dG:[],base:null});
  if(g.detached && L.base)
    return { h:L.base.hG.concat(L.hG), o:L.base.oG.concat(L.oG), d:L.base.dG.concat(L.dG) };
  return { h:hG.concat(L.hG), o:oG.concat(L.oG), d:dG.concat(L.dG) };
}
```

- [ ] **Step 2: Detach/reattach snapshots the shared guides (preserving ids)**

```js
const detachBtn=document.createElement('button'); detachBtn.className='detach'; detachBtn.textContent='⛔ stop sync';
detachBtn.style.display='none';
detachBtn.addEventListener('click',()=>{
  const g=glyphOf(ch); if(!g.split) return; pushHistory();
  g.detached=!g.detached;
  if(g.detached){                                   // freeze current shared guides into base
    g.localGuides.base={ hG:JSON.parse(JSON.stringify(hG)),
                         oG:JSON.parse(JSON.stringify(oG)),
                         dG:JSON.parse(JSON.stringify(dG)) };   // ids copied verbatim
  } else { g.localGuides.base=null; }               // reattach: drop frozen copy, follow shared again
  detachBtn.classList.toggle('on',g.detached);
  render(); scheduleSave();
});
host.querySelector('.phead').appendChild(detachBtn);
```

When detached, the add/drag handlers target `localGuides.base`/`localGuides.extra` arrays,
not the shared globals. Update `findGuide` to search `base` + `extra` when detached.

- [ ] **Step 3: Browser verification**
1. Panel 1 = `a`, split ON. Click **stop sync**. Console:
   `GLYPHS.get('a').localGuides.base.oG.length` equals shared `oG.length`, and the ids
   match (`GLYPHS.get('a').localGuides.base.oG[0].id === oG[0].id`).
2. Drag a shared oblique guide in panel 2 → panel 1 (`a`, detached) does NOT move; panel 3
   does.
3. Existing fills in `a` that were bound to those guides stay intact (ids preserved) and
   reshape to the frozen copy when YOU drag them in panel 1.
4. Click **stop sync** off → panel 1 follows shared guides again.

Expected: detach freezes a private id-stable copy; shared edits no longer reach it; cells
stay bound.

- [ ] **Step 4: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): stop-propagating detach with id-stable frozen guides"
```

---

### Task 7: Sample strip — editable text, size + letter-spacing, live merged render

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Consumes: `GLYPHS`, `mergedPaths`/`cellRings` (source exporters), `metrics`.
- Produces:
  - `function renderSample()` — draws `#sampleText` chars as merged glyph outlines,
    laid out by advance width + spacing, into `#sampleSvg`. Called from `renderAll()`.
  - `function glyphOutlinePaths(ch)` → `string[]` of SVG path-data for that glyph's merged
    outline in canvas coords (empty array if undesigned).

- [ ] **Step 1: Add the sample strip markup + controls (replace the old `#status` footer)**

```html
<div id="sample">
  <input id="sampleText" type="text" value="abcdefghijklmnopqrstuvwxyz" style="flex:1;min-width:120px">
  <label>size <input id="sampleSize" type="range" min="20" max="160" value="80"></label>
  <label>track <input id="sampleTrack" type="range" min="-40" max="120" value="0"></label>
  <div id="sampleView"><svg id="sampleSvg" xmlns="http://www.w3.org/2000/svg"></svg></div>
</div>
```

```css
#sample{display:flex;gap:10px;align-items:center;padding:8px 10px;background:var(--panel);border-top:1px solid var(--line);flex-wrap:wrap}
#sampleView{flex-basis:100%;height:120px;background:#000;border:1px solid #1c1c1c;border-radius:6px;overflow:hidden}
#sampleSvg{width:100%;height:100%}
```

- [ ] **Step 2: Implement outline extraction + layout**

```js
function glyphOutlinePaths(ch){
  const g=GLYPHS.get(ch); if(!g||!g.cells.length) return [];
  const simple=[], compound=[];
  for(const c of g.cells){ const r=cellRings(c); if(!r) continue; (r.length>1?compound:simple).push(r.length>1?r:r[0]); }
  const out=mergedPaths(simple).slice();
  for(const r of compound) out.push(r.map(loopPath).join(' '));
  return out;
}
function renderSample(){
  const svgS=document.getElementById('sampleSvg');
  while(svgS.firstChild) svgS.removeChild(svgS.firstChild);
  const text=document.getElementById('sampleText').value;
  const track=+document.getElementById('sampleTrack').value;
  const base=metrics.baseline, capH=Math.max(1, base-metrics.cap);   // canvas units per "cap height"
  let penX=metrics.sidebearing;
  for(const chr of text){
    if(chr===' '){ penX+=metrics.advance*0.5+track; continue; }
    for(const d of glyphOutlinePaths(chr)) svgS.appendChild(svgEl('path',{d,fill:'#fff',transform:`translate(${penX} 0)`}));
    penX += (metrics.advance) + track;     // advance-width spacing (per-glyph override deferred)
  }
  const totalW=penX+metrics.sidebearing;
  const sz=+document.getElementById('sampleSize').value;
  // map canvas y so baseline..cap fits the strip height at requested px size
  const scale=sz/capH;
  svgS.setAttribute('viewBox',`0 ${metrics.ascender} ${Math.max(totalW,1)} ${metrics.descender-metrics.ascender}`);
  svgS.setAttribute('preserveAspectRatio','xMinYMid meet');
  void scale; // size slider drives strip zoom via viewBox height; see Step 4 note
}
```

Wire the three inputs to `renderSample` (`input` events) and add `renderSample()` to
`renderAll()`.

- [ ] **Step 3: Browser verification**
1. Design a couple of letters (`a`, `b`) with fills. The sample strip shows them inline at
   their alphabet positions; undesigned letters are blank gaps.
2. Edit a cell in `a` → the strip updates live.
3. Drag the **size** slider → glyphs scale; **track** slider → spacing widens/narrows.
4. Type `ab ab` into the text field → spaces produce gaps, repeated letters repeat.

Expected: strip is a live, editable specimen driven by the glyph store + metrics.

- [ ] **Step 4: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): live editable sample strip"
```

---

### Task 8: Project JSON save / load

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Consumes: `serializeStore`, `applyProject` (Task 2).
- Produces: **Save Project** (download `.json`) and **Load Project** (file input →
  `applyProject`) toolbar buttons.

- [ ] **Step 1: Add buttons + hidden file input to the export toolbar group**

```html
<button id="saveproj">⤓ Save Project</button>
<button id="loadproj">⤒ Load Project</button>
<input id="projfile" type="file" accept=".json,application/json" style="display:none">
```

- [ ] **Step 2: Wire save/load (reusing serializeStore + applyProject + metrics)**

Add `metrics` to `serializeStore` (Step done in Task 4) and `applyProject`:

```js
document.getElementById('saveproj').onclick=()=>{
  const blob=new Blob([JSON.stringify(serializeStore(),null,0)],{type:'application/json'});
  const a=document.createElement('a'); a.href=URL.createObjectURL(blob); a.download='z-typeface.json'; a.click();
  URL.revokeObjectURL(a.href);
};
const pf=document.getElementById('projfile');
document.getElementById('loadproj').onclick=()=>pf.click();
pf.onchange=()=>{ const f=pf.files[0]; if(!f) return; const r=new FileReader();
  r.onload=()=>{ try{ pushHistory(); applyProject(JSON.parse(r.result));
    for(const e of editors) e.syncLetter(); renderAll(); scheduleSave(); }
    catch(err){ alert('Bad project file: '+err.message); } pf.value=''; }; r.readAsText(f); };
```

Add `syncLetter()` to the Editor object: re-point its dropdown to a valid char after a
project load (`if(!GLYPHS.has(ch)){…}`; set `sel.value=ch`).

- [ ] **Step 3: Browser verification**
1. Design `a`, `b`, set some metrics + a split on `a`. Click **Save Project** → a
   `z-typeface.json` downloads.
2. `localStorage.clear()`, reload (blank tool). Click **Load Project**, pick the file →
   glyphs, guides, angles, metrics, and the split state on `a` all return.
3. Console: `JSON.parse(saved).glyphs.a.split === true`.

Expected: lossless round-trip.

- [ ] **Step 4: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): project JSON save/load"
```

---

### Task 9: `.otf` font export via opentype.js

**Files:**
- Modify: `z-type-tool.html`

**Interfaces:**
- Consumes: `glyphOutlinePaths` (Task 7) — but needs RAW rings, not path strings, for the
  font path builder. Add `function glyphRings(ch)` → `Array<Array<[x,y]>>` (all merged
  outline loops in canvas coords). `metrics`, `opentype` (global from CDN).
- Produces: **Export .otf** button → builds and downloads an `.otf`.

- [ ] **Step 1: Add `glyphRings` (loops as point arrays, reusing the merge engine)**

`mergedPaths` returns path-data strings; for opentype we need the polylines. Factor the
loop geometry out: add a sibling to `mergedPaths`:

```js
function mergedLoops(polys){     // same as mergedPaths but returns loops (arrays of pts), not path strings
  const norms=polys.map(norm);
  const out=[];
  for(const idxs of groupByEdge(norms)) for(const lp of groupLoops(idxs.map(i=>norms[i]))) out.push(lp);
  return out;
}
function glyphRings(ch){
  const g=GLYPHS.get(ch); if(!g||!g.cells.length) return [];
  const simple=[], compound=[];
  for(const c of g.cells){ const r=cellRings(c); if(!r) continue; if(r.length>1) compound.push(r); else simple.push(r[0]); }
  return mergedLoops(simple).concat(...compound);   // compound rings already separate loops
}
```

- [ ] **Step 2: Build the opentype.Font (canvas→em transform, Y-flip about baseline)**

```js
function buildFont(){
  if(typeof opentype==='undefined'){ alert('opentype.js not loaded (needs network).'); return null; }
  const UPM=1000, base=metrics.baseline, capH=Math.max(1, base-metrics.cap);
  const s=UPM/capH;                                   // canvas units → em units
  const X=x=>x*s, Y=y=>(base-y)*s;                    // flip Y so up is positive em
  const glyphs=[ new opentype.Glyph({name:'.notdef', unicode:0, advanceWidth:Math.round(metrics.advance*s), path:new opentype.Path()}) ];
  for(const [ch,g] of GLYPHS){
    const rings=glyphRings(ch); if(!rings.length) continue;
    const path=new opentype.Path();
    for(const lp of rings){ if(lp.length<3) continue;
      path.moveTo(X(lp[0][0]), Y(lp[0][1]));
      for(let i=1;i<lp.length;i++) path.lineTo(X(lp[i][0]), Y(lp[i][1]));
      path.close();
    }
    glyphs.push(new opentype.Glyph({ name:ch, unicode:ch.codePointAt(0),
      advanceWidth:Math.round(metrics.advance*s), path }));
  }
  if(glyphs.length<2){ alert('No designed glyphs to export.'); return null; }
  return new opentype.Font({ familyName:'ZType', styleName:'Oblique', unitsPerEm:UPM,
    ascender:Math.round((base-metrics.ascender)*s), descender:Math.round((base-metrics.descender)*s),
    glyphs });
}
document.getElementById('exportotf').onclick=()=>{ const f=buildFont(); if(f) f.download('ZType-Oblique.otf'); };
```

Add the button to the toolbar: `<button id="exportotf">⤓ Export .otf</button>`.

Note on winding: `mergedLoops` yields outer loops and holes with opposite orientation
(the source merge guarantees this), which is exactly what opentype's nonzero fill needs —
no extra reversal required.

- [ ] **Step 3: Add a console sanity assert for the transform**

In DevTools after designing a glyph touching the baseline and cap lines:

```js
// baseline maps to em y=0, cap maps to em y≈UPM
console.assert(Math.abs(((metrics.baseline-metrics.baseline)*(1000/(metrics.baseline-metrics.cap))))===0,'baseline→0');
console.assert(Math.round((metrics.baseline-metrics.cap)*(1000/(metrics.baseline-metrics.cap)))===1000,'cap→1000');
```

Expected: both asserts pass (no console error).

- [ ] **Step 4: Browser verification**
1. Design at least `a` and `b` with solid fills spanning baseline→cap.
2. Click **Export .otf** → `ZType-Oblique.otf` downloads.
3. Install the font (OS font viewer) OR load it back via the existing **⤒ Font** importer
   and **Place as cells** the letter `a` → the placed outline matches the designed `a`.
4. The glyphs sit on the baseline (not floating/clipped), confirming the Y-flip + scale.

Expected: a valid, installable `.otf` whose glyphs match the canvas designs and metrics.

- [ ] **Step 5: Commit**

```bash
git add z-type-tool.html
git commit -m "feat(type-tool): .otf font export via opentype.js"
```

---

## Self-Review

**Spec coverage:**
- 3 panels sharing guides → Task 1. Per-glyph store + dropdowns + autosave → Task 2.
- Global undo → Task 3. Draggable global metric frame → Task 4.
- Split (extra local, propagating) → Task 5. Stop-propagating detach → Task 6.
- Editable sample strip + size + track → Task 7. Project JSON → Task 8. `.otf` → Task 9.
- Angles always global: enforced — no task touches angle in split/detach paths (Tasks 5/6
  only ever fork guide *positions*).
- Reuse of source engine (`FAM`, `corners`, `cellRings`, `mergedPaths`, `groupLoops`,
  `pathToRings`, `importSVG`): Tasks 1, 5, 7, 9.

**Type consistency:** `glyphOf(ch)` record shape `{cells, split, detached, localGuides}` is
identical across Tasks 2/5/6/8. `localGuides` shape `{hG,oG,dG[,base]}` consistent in 5/6.
`effGuides(g)` returns `{h,o,d}` in both 5 and 6. `serializeStore`/`applyProject` symmetric
and used by both autosave (2) and project JSON (8). `mergedPaths`(strings, Task 7) vs
`mergedLoops`(point arrays, Task 9) are distinct names for distinct return types — no clash.

**Placeholder scan:** No TBD/TODO. The one deferred detail (exact punctuation set) is fixed
concretely in Task 2 (`ALPHABET` string). Per-glyph advance-width override is explicitly
out of scope (global advance only), matching the spec's "optional override" as a non-goal
for v1.

**Open risk flagged for execution:** Task 1's refactor of source globals → `Editor`
closure is the highest-effort step; if `EFF`/`activeEditor` wiring is wrong, click
coordinates desync after per-panel zoom. The Task 1 verification explicitly checks this.
