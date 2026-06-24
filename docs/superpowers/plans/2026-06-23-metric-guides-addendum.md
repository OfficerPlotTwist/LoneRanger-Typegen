# Addendum: Metric lines function as horizontal guides

> Enhancement to `z-type-tool.html` after the 9-task build. Branch `type-tool`.

**Goal:** Make the metric lines **ascender, cap, baseline, descender** behave as real
horizontal (‖ / h-family) guides — fillable, bindable, and snap targets — while
**x-height** remains a reference-only line (visible, draggable, used for font metadata,
but NOT a fill/bind/snap guide).

## Decision: metric guides ARE entries in the shared `hG` array

The four functional metric lines become special, **undeletable**, tagged members of the
shared `hG` guide array, so they automatically participate in everything the h-family
already does (`bracketF` fill, cell binding via `corners`/`byId`, effGuides resolution,
per-panel render, propagation across panels). This is far less error-prone than threading
a parallel metric set through every guide consumer.

Guide shape for a metric guide: `{ id:<sentinel>, v:<y>, metric:'baseline'|'cap'|'ascender'|'descender' }`.
Use fixed sentinel ids that never collide with `nextId` (e.g. negative ids `-1..-4`, and
keep `nextId` minting positive ids). Ids must stay stable so bound cells survive.

## Single source of truth for the 4 metric positions

Remove `baseline`, `cap`, `ascender`, `descender` from the `metrics` object. Their live
positions now live ONLY in the tagged `hG` guides. Add a helper:

```js
function metricY(key){ const g=hG.find(g=>g.metric===key); return g?g.v:0; }
```

Replace every read of `metrics.baseline|cap|ascender|descender` with `metricY('...')`:
- `buildFont()` (`base`, `capH`, `ascender`, `descender` em math)
- `renderSample()` (`base`, `capH`)
- the numeric metric inputs for these 4 (their `change` writes to the guide's `v`, then
  `renderAll()`; `syncMetricUI()` reads `metricY(...)` for those 4)

`metrics` KEEPS `xHeight`, `advance`, `sidebearing` only.

## x-height stays reference-only

Keep `metrics.xHeight`. Render it as the current dashed purple reference line + label
(the existing `.mline`/`.mline-hit` style), draggable to update `metrics.xHeight`. It is
NOT added to `hG`, NOT bindable/fillable, NOT a snap target. Its numeric input stays.

## Rendering

In the per-Editor `render()` h-guide loop: if a guide has `.metric`, draw it with the
metric style (dashed accent + the metric name as a text label) instead of the normal
`gline-h` style. It keeps a `.gline-hit` so it stays draggable via the existing guide
drag. Remove baseline/cap/ascender/descender from the old `METRIC_KEYS` separate-line
rendering; x-height is the only remaining separately-rendered reference line.

## Undeletable

`delGuide` (and the dblclick/right-click delete paths) must no-op on a guide whose
`.metric` is set. They can be dragged but never removed.

## Snap targets

Extend `snapY(y)`: in addition to the existing grid snap, snap to any metric guide's `v`
(the 4 functional ones) when within the snap radius, so a new/dragged ‖ guide locks onto
baseline/cap/etc. (x-height is excluded.)

## Persistence / migration / undo

- Metric guides ride inside `hG`, so `snapshot`/`serializeStore`/`applyProject` already
  carry them. No new fields except they're now in `hG`.
- **Migration:** at `loadStore()`/`applyProject()`, after restoring `hG`, ensure the four
  metric guides exist. If an older save lacks them (no `g.metric` in `hG`), inject them —
  seeding `v` from the old `p.metrics.baseline|cap|ascender|descender` if present, else the
  current defaults — using the sentinel ids. Guarantees one of each, never duplicated.
- Boot: when starting with no save, seed the four metric guides into `hG` with defaults
  (baseline 1040, cap 240, ascender 160, descender 1140) and sentinel ids.

## Acceptance (human browser-verified)

1. baseline/cap/ascender/descender render as labelled horizontal guides in all 3 panels.
2. Fill mode can fill a cell bounded by cap→baseline (they bracket like ‖ guides); dragging
   the cap guide reshapes that cell.
3. Adding/dragging a normal ‖ guide near a metric line snaps onto it.
4. Right-click / double-click does NOT delete a metric guide; a normal guide still deletes.
5. x-height still shows as a reference line, draggable, but cannot be filled/bound/snapped.
6. `.otf` export and the sample strip still place glyphs correctly (now reading `metricY`).
7. Reload restores everything; an older project JSON still loads (migration injects metrics).

## Out of scope (YAGNI)

- Making x-height a functional guide.
- Per-glyph metric overrides.
- Adding new metric types beyond the existing five.
