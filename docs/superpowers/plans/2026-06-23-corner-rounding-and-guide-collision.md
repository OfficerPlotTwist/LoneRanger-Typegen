# Corner Rounding + Guide Collision — Implementation Plan

> **For agentic workers:** implement task-by-task; each ends with a `node --check`-verifiable
> deliverable. Browser/visual checks are human-deferred (single `file://` HTML, no runner).

**Goal:** Add (A) min-gap push collision for guides and (B) per-cell corner rounding to
`loneranger-typesetter.html`, on branch `corner-rounding`.

**Architecture:** Single self-contained `file://` HTML, one inline `<script>`, no ES modules,
opentype.js via CDN. Reuse the existing guide/effGuides/cell/export engine.

**Spec:** docs/superpowers/specs/2026-06-23-corner-rounding-and-guide-collision-design.md

## Global Constraints
- Single file `loneranger-typesetter.html`; no build; `file://`-runnable; one inline `<script>`; no ES modules.
- Angles always global; guide ids stable (incl. sentinel metric ids `-1..-4`); never renumber.
- Reuse engine: `effGuides`, `corners`, `cellRings`, `ringPath`, `mergedPaths`, `mergedLoops`, `groupLoops`, `findGuide`, `snapY`, `metricY`.
- Verify each task: extract inline `<script>` → temp `.js` → `node --check`; report cmd+output. Do NOT fabricate browser runs.
- Commit only `loneranger-typesetter.html` (`.superpowers/` gitignored).

---

### Task 1 — Guide collision (min-gap push)

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:**
- `let minGap` (number) + toolbar numeric input `#mingap`.
- `function pushFamily(famKey, draggedGuide)` — enforces the gap within the active glyph's effective family set.

- [ ] **Step 1: minGap state + numeric input + persistence**
  - Add `let minGap = 0;` and initialize at boot AFTER metric guides seed:
    `minGap = Math.max(1, (metricY('baseline') - metricY('cap'))/16);`
  - Add toolbar input in a sensible group: `<label>min gap <input id="mingap" type="number" min="0" step="1" style="width:54px"></label>`. Wire `change` → `pushHistory(); minGap = Math.max(0, +e.target.value||0); render-all; scheduleSave();`. Add `function syncGapUI(){ document.getElementById('mingap').value = Math.round(minGap); }` and call it at boot, in `undo`, and in `applyProject`.
  - Add `minGap` to `snapshot()` (capture/restore), `serializeStore()` (include), `applyProject()` (`if(typeof p.minGap==='number') minGap=p.minGap;` else keep default).

- [ ] **Step 2: `pushFamily` cascade**
  - Family scalar accessors already exist via `FAM` (`FAM.h.val`, `FAM.o.val`, `FAM.d.val`) and the guides store `v` (h/o) or `px` (d). Implement:
```js
function pushFamily(famKey, g){
  if(!(minGap>0)) return;
  const eff = effGuides(glyphOf(activeCh()));          // active glyph's effective set; use the controller's current-glyph accessor
  const arr = (famKey==='h'?eff.h:famKey==='o'?eff.o:eff.d);
  const scal = famKey==='d' ? (x=>x.px) : (x=>x.v);
  const setS = famKey==='d' ? ((x,v)=>x.px=v) : ((x,v)=>x.v=v);
  const fam = [...arr].sort((a,b)=>scal(a)-scal(b));
  const i = fam.indexOf(g); if(i<0) return;
  for(let j=i+1;j<fam.length;j++) if(scal(fam[j]) < scal(fam[j-1])+minGap) setS(fam[j], scal(fam[j-1])+minGap);
  for(let j=i-1;j>=0;j--)        if(scal(fam[j]) > scal(fam[j+1])-minGap) setS(fam[j], scal(fam[j+1])-minGap);
}
```
  - NOTE: replace `activeCh()`/`glyphOf(...)` with the actual current-glyph accessor used elsewhere when a guide is dragged — the drag happens inside an `Editor` closure that knows its `ch`; call `pushFamily` from there passing the closure's effective set, OR pass the eff set in. Simplest: compute `pushFamily` inside the Editor's `onDrag` where `glyphOf(ch)` and `effGuides` are in scope. Adapt the signature to receive the eff family array directly if that's cleaner: `pushFamily(arr, scalKind, g)`.

- [ ] **Step 3: call it during drag**
  - In the Editor `onDrag` for guide drags: after the dragged guide's `v`/`px` is set (post-snap, via the existing `findGuide` path), call the push for that guide's family, then `renderAll()`. Determine the family from `drag.type` (`'h'|'o'|'d'`). Metric guides are in `hG` → they participate automatically (uniform push). Do NOT exempt them.
  - Keep `snapY(p.y, drag.id)` self-snap exclusion intact.

- [ ] **Step 4: verify + commit**
  - Extract `<script>` → `node --check`. Commit: `feat(typesetter): min-gap push so guides never cross`.
  - Human-deferred browser checks: dragging an h guide into a neighbor pushes it; cascade through several; metric guides push/are pushed; gap = capHeight/16 by default; editing `#mingap` changes it; undo reverts all pushed guides; split/detached glyph pushes only its own guides.

---

### Task 2 — Corner rounding: model, mode, handles, canvas + sample render, sliders

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:**
- `const cornerRadius = {tl:60,tr:60,bl:60,br:60}`; 4 toolbar sliders.
- `function classifyCorner(pt, centroid)` → `'tl'|'tr'|'bl'|'br'`.
- `function cornerPathStr(ring, roundedSet)` → SVG path-data string.
- `function cellRoundedSet(cell)` → Set of rounded indices (empty if none / not eligible).
- Mode `'round'` + corner handles + click toggle.

- [ ] **Step 1: state + sliders + persistence**
  - `const cornerRadius={tl:60,tr:60,bl:60,br:60};`
  - Toolbar group with 4 range inputs `data-c="tl|tr|bl|br"` class `cnum`, min 0 max 300 value 60, each labelled (⌜ ⌝ ⌞ ⌟). `input` → `cornerRadius[c]=+val; renderAll(); scheduleSave();` (push history on first input of a drag is optional; simplest: pushHistory on `change`). `function syncCornerUI(){...}` set at boot/undo/applyProject.
  - Add `cornerRadius` to `snapshot`/`serializeStore`/`applyProject` (default-merge: `if(p.cornerRadius) Object.assign(cornerRadius, p.cornerRadius)`).

- [ ] **Step 2: geometry helpers**
```js
function classifyCorner(p, c){ return (p[1]<c[1]?'t':'b') + (p[0]<c[0]?'l':'r'); } // -> 'tl'|'tr'|'bl'|'br'
function centroidOf(ring){ let x=0,y=0; for(const p of ring){x+=p[0];y+=p[1];} return [x/ring.length,y/ring.length]; }
function cellRoundedSet(cell){
  // eligible: single-ring fill cell (guide-bound or free). Not compound/glyph (.rings) cells.
  if(cell.rings) return null;
  return Array.isArray(cell.rounded) ? new Set(cell.rounded) : new Set();
}
function cornerPathStr(ring, rounded){
  if(!rounded || !rounded.size) return ringPath(ring);   // existing sharp builder
  const c = centroidOf(ring), n = ring.length;
  const sub=(a,b)=>[a[0]-b[0],a[1]-b[1]], add=(a,b)=>[a[0]+b[0],a[1]+b[1]],
        mul=(a,s)=>[a[0]*s,a[1]*s], len=a=>Math.hypot(a[0],a[1]);
  const unit=a=>{const L=len(a)||1;return [a[0]/L,a[1]/L];};
  const seg=[]; // build commands
  for(let i=0;i<n;i++){
    const cur=ring[i], prev=ring[(i-1+n)%n], next=ring[(i+1)%n];
    if(rounded.has(i)){
      const r=cornerRadius[classifyCorner(cur,c)];
      const t=Math.min(r, 0.5*len(sub(prev,cur)), 0.5*len(sub(next,cur)));
      const pin=add(cur,mul(unit(sub(prev,cur)),t)), pout=add(cur,mul(unit(sub(next,cur)),t));
      seg.push({pin,cur,pout,round:true});
    } else seg.push({cur,round:false});
  }
  // emit: start at first segment's entry point
  const f=2, fix=p=>p.map(v=>+v.toFixed(f));
  let d, start = seg[0].round ? seg[0].pin : seg[0].cur;
  d='M'+fix(start).join(' ');
  for(let i=0;i<n;i++){ const s=seg[i];
    if(s.round){ d+=' L'+fix(s.pin).join(' ')+' Q'+fix(s.cur).join(' ')+' '+fix(s.pout).join(' '); }
    else d+=' L'+fix(s.cur).join(' ');
  }
  return d+' Z';
}
```
  (The leading `L` to `seg[0].pin` duplicates the M point harmlessly; acceptable. Ensure the first rounded corner's `Q` still renders — the wrap is handled because we iterate all `n` and close.)

- [ ] **Step 3: canvas + sample render use the rounded builder**
  - Canvas: where the Editor renders a cell's path for single-ring cells (today via `ringPath(corners)`), use `cornerPathStr(ring, cellRoundedSet(cell))`. Multi-ring/compound cells unchanged.
  - Sample strip `glyphOutlinePaths(ch)`: for SIMPLE cells, if `glyphHasRounding(g)` emit each simple cell via `cornerPathStr(ring, cellRoundedSet(cell))`; else keep `mergedPaths(simple)`. (Define `function glyphHasRounding(g){ return g.cells.some(c=>!c.rings && Array.isArray(c.rounded) && c.rounded.length); }`.) Compound cells unchanged.

- [ ] **Step 4: round mode + handles + click toggle**
  - Add mode button `<button id="m-round" class="mode">◜ Round corner</button>` to the mode group; wire into the existing mode switching (`modes`/`mlabel`).
  - In the Editor render, when `mode==='round'`, draw corner handles: for each eligible single-ring cell, for each corner index, append a small `circle` (r≈7 in current viewBox units, class `chandle` / `chandle on` if rounded). Handles are pointer-targets.
  - In the Editor `pointerdown` for `mode==='round'`: hit-test the click against eligible cells' corners (nearest corner within ~14 units). On hit: `pushHistory()`; toggle that index in `cell.rounded` (init `cell.rounded=[]` if absent; add/remove index); `renderAll(); scheduleSave()`. Use the cell's CURRENT corners (`cellRings(cell, eff)[0]` / `corners(cell, eff)`) to locate vertices.
  - CSS: `.chandle{fill:#000;stroke:var(--accent);stroke-width:2}` `.chandle.on{fill:var(--accent)}`.

- [ ] **Step 5: verify + commit**
  - `node --check`. Commit: `feat(typesetter): corner rounding model, round mode, 4 position sliders, canvas+sample render`.
  - Human-deferred: round-mode shows handles on filled cells; clicking toggles round/sharp; the 4 sliders change radius by corner position; rounding persists & undoes; non-fill (glyph) cells show no handles.

---

### Task 3 — Corner rounding export (SVG + OTF)

**Files:** Modify `loneranger-typesetter.html`.

**Interfaces produced:** `function cornerLoopPts(ring, roundedSet, steps)`; per-cell export branch in SVG export and `glyphRings`.

- [ ] **Step 1: flattened rounded loop for OTF**
```js
function cornerLoopPts(ring, rounded, steps=8){
  if(!rounded || !rounded.size) return ring.slice();
  const c=centroidOf(ring), n=ring.length;
  const sub=(a,b)=>[a[0]-b[0],a[1]-b[1]], add=(a,b)=>[a[0]+b[0],a[1]+b[1]],
        mul=(a,s)=>[a[0]*s,a[1]*s], len=a=>Math.hypot(a[0],a[1]), unit=a=>{const L=len(a)||1;return[a[0]/L,a[1]/L];};
  const out=[];
  for(let i=0;i<n;i++){
    const cur=ring[i], prev=ring[(i-1+n)%n], next=ring[(i+1)%n];
    if(rounded.has(i)){
      const r=cornerRadius[classifyCorner(cur,c)];
      const t=Math.min(r,0.5*len(sub(prev,cur)),0.5*len(sub(next,cur)));
      const pin=add(cur,mul(unit(sub(prev,cur)),t)), pout=add(cur,mul(unit(sub(next,cur)),t));
      out.push(pin);
      for(let k=1;k<=steps;k++){ const u=k/steps, mt=1-u;            // quadratic Bézier pin->cur->pout
        out.push([mt*mt*pin[0]+2*mt*u*cur[0]+u*u*pout[0], mt*mt*pin[1]+2*mt*u*cur[1]+u*u*pout[1]]); }
    } else out.push(cur);
  }
  return out;
}
```

- [ ] **Step 2: SVG export per-cell when rounded**
  - In the SVG export builder, today simple cells go through `mergedPaths(simple)`. Change: if `glyphHasRounding`-equivalent across the cells being exported (any simple cell has rounding), emit each simple cell as its own `<path>` via `cornerPathStr(ring, cellRoundedSet(cell))`; else keep `mergedPaths(simple)`. Compound cells unchanged (evenodd). (The export iterates all glyphs/cells — apply the rounded-vs-merge choice per the set of simple cells being emitted; simplest: if ANY simple cell across the export has rounding, emit ALL simple cells per-cell so abutment stays consistent.)

- [ ] **Step 3: OTF export per-cell when rounded**
  - In `glyphRings(ch)`: today `return mergedLoops(simple).concat(...compound)`. Change to:
    `if(glyphHasRounding(g)) return simple.map(c=>cornerLoopPts(cellRingFor(c), cellRoundedSet(c))).concat(...compound); else return mergedLoops(simple).concat(...compound);`
    where `cellRingFor(c)` is the single ring used today to build `simple` (the same ring pushed into `simple`). Ensure `simple` retains the cell reference so `cellRoundedSet`/ring are available — adjust the simple-collection loop to keep `{cell, ring}` pairs if needed.

- [ ] **Step 4: verify + commit**
  - `node --check`. Commit: `feat(typesetter): corner rounding flows into SVG and .otf export`.
  - Human-deferred: export an SVG of a rounded glyph → corners are rounded; export `.otf`, install → rounded corners present; un-rounded glyphs still export the merged outline as before; counters/holes still correct.

## Self-Review checklist (run after writing each task)
- No `metrics.{baseline,cap,ascender,descender}` reintroduced (still `metricY`).
- `cornerRadius`/`minGap` added to all of snapshot/serializeStore/applyProject.
- Rounding eligible only for single-ring cells; compound/glyph cells untouched.
- Un-rounded glyphs keep existing merged export (no behavior change when nothing is rounded).
- Push uses the active glyph's effective family set; metric guides included; undo reverts via the drag-start snapshot.
