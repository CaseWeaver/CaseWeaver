# CaseWeaver Code Documentation (v1.001)

Technical documentation of the single-file architecture, the embedded data model, and every
feature subsystem — written for code review and hand-off to a human developer. All code
lives in `CaseWeaver.html`; there is no build step, no bundler, and no external request at
runtime.

---

## 1. File anatomy

`CaseWeaver.html` contains, in order:

| Section | What it is |
|---|---|
| `<head>` styles | All CSS, hand-written, theme-driven via CSS custom properties (`--bg`, `--accent`, …) |
| `<script>` #1 | **vis-network** (bundled, minified) — graph rendering & physics |
| `<script>` #2 | **jsPDF** (bundled, minified) — PDF report generation |
| `<body>` markup | Header, sidebar, graph container, canvas toolbar, detail pane, import screen, modals (editor, settings, report, CSV view, about), selection bar, presentation bar, toast |
| `<script id="case-data" type="application/json">` | **The case itself.** A single JSON document; rewritten in place on every save |
| `<script>` #3 | The application (~2,000 lines, unminified, section-commented) |

The app script is organised in commented sections, top to bottom: brand constants →
selector/entity/theme/shape/icon/emoji libraries → data model & normalisation → undo →
graph build → sidebar renderers → bulk selection & buckets → link drawing → redaction →
annotations → PNG export → storyboard/present → filters/stats → detail pane → dossier →
search → view buttons → layout engine → lens → CSV import → dashboard import → editor →
settings → save/auto-save → PDF report → toast → about → init → CSV view.

### Save-in-place mechanism

At load, before the app mutates the DOM:

```js
const PRISTINE_HTML = "<!DOCTYPE html>\n" + document.documentElement.outerHTML;
```

`buildFileHTML()` regex-replaces the `#case-data` block inside `PRISTINE_HTML` with
`serializeDataJSON()` (all `<` escaped to `<` so no value can terminate the script
block). The result is a byte-faithful copy of the original file with only the data swapped
— this is what Save writes.

## 2. Data model (`DATA`)

Parsed from `#case-data` at load; `normalizeData()` guarantees every field below exists so
downstream code never checks for `undefined`. Unknown fields are preserved (forward
compatibility); missing fields are defaulted (backward compatibility — this is what makes
old-version dashboards importable).

```jsonc
{
  "config": {
    "caseId": "CASE-XXX", "caseTitle": "…", "subtitle": "…", "sourceNote": "…",
    "summary": "",              // executive summary for the PDF
    "currency": "$",            // amount unit; alphabetic units render as suffixes
    "theme": "dark",
    "hideBrand": false,
    "report": {},               // report dialog state (title, banner, cover options…)
    "savedAt": 0,               // Date.now() at save; newer-copy detection for snapshots
    "autosave": { "on": false, "interval": 300000 },   // OFF by default; ms (60s minimum)
    "spacing": 1.5              // physics spacing multiplier, 0.5–3
  },
  "clusters": { "c1": { "name": "Entities", "color": "#4363d8" } },
  "nodes": [{
    "id": "jane_doe",           // slugified label, unique
    "label": "Jane Doe",
    "cluster": "c1",            // PRIMARY cluster — sets the colour
    "clusters": ["c4"],         // EXTRA memberships (never contains the primary)
    "type": "P",                // one of NTYPES: P C H A M G L F E D
    "shape": "", "icon": "", "emoji": "", "photo": "",   // visual overrides (photo = data URL)
    "role": "", "sig": "", "desc": "",
    "tags": ["mule"],
    "sel": { "addr": [], "email": [], "phone": [], "domain": [], "id": [], "ip": [],
             "device": [], "wallet": [], "hash": [], "url": [], "user": [], "mac": [] },
    "pinned": false,            // anchored in place (physics can't move it)
    "x": 0, "y": 0              // remembered position — only meaningful for pinned nodes
  }],
  "edges": [{ "from": "id", "to": "id", "label": "controls", "weak": 0,
              "amount": 262000, "date": "2026-05-02" }],
  "bookmarks": [ /* full scene snapshots captured by the storyboard */ ],
  "annotations": [{ "id": "note_1", "text": "…", "x": 0, "y": 0,
                    "anchor": "jane_doe" }],   // anchor optional → note follows that node
  "buckets": [{ "id": "bucket:1", "name": "Money mules",
                "members": ["id", …], "collapsed": true, "x": 0, "y": 0 }]
}
```

Integrity rules enforced by `normalizeData()` / `syncFromData()`:

- every node's `cluster` exists (else reassigned to the first cluster); `clusters[]` is
  filtered to existing clusters and never contains the primary;
- bucket members must reference existing nodes; empty buckets are dropped;
- note `anchor`s pointing at deleted nodes are removed;
- edge `amount` must be a positive finite number, else `null`.

## 3. Runtime state

`DATA` is the single source of truth; everything else is derived and rebuilt by `build()`:

| Global | Meaning |
|---|---|
| `network, nodes, edges` | vis-network instance + its two DataSets (view model, NOT persisted) |
| `N, E, CLUSTERS, SEL, SIG` | flat derivations of `DATA` (`syncFromData()`) |
| `degree, degRanked, bigSet, midSet` | connectivity ranking → node sizing |
| `index, shared` | selector index: `type|normalizedValue → {ids}`; `shared` = entries with ≥2 ids |
| `active, activeTags, selTypesOn` | current filter state (visible clusters / tags / selector types) |
| `connMin, flowMin, flowFrom, flowTo` | numeric filters |
| `phys, currentLayout, lensOn, selMode, selFocus, traceOn, REDACT, presenting` | mode flags |
| `undoStack, DIRTY` | undo snapshots; unsaved-changes flag for auto-save |
| `fileHandle` | remembered File System Access handle (persisted in IndexedDB) |

Conventions: vis node ids prefixed `note_` are annotations, `bucket:` are bucket container
nodes (the `:` cannot appear in slugified entity ids, so no collision); everything else is
an entity. Private per-node view fields are underscored (`_cl`, `_cls`, `_type`, `_raw`,
`_pinned`, `_note`, `_bucket`). Edge `_kind` is `"rel"`, `"sel"`, or `"note"`.

## 4. The build pipeline — `build(preserve, posOverride)`

`build()` tears down and recreates the vis network from `DATA`. **Every structural change
funnels through it**, which is what keeps feature interactions simple.

`preserve` (added in v1.001) is the important part:

- `build(false)` — fresh layout: physics stabilisation runs, camera fits, filters reset.
  Used for imports and initial load.
- `build(true)` — **scene-preserving rebuild**: captures node positions, camera
  (`getViewPosition`/`getScale`), physics pause state, active clusters, and active tags
  *before* teardown, then reapplies them: every surviving node is recreated at its old
  x/y, stabilisation is disabled, the camera is `moveTo`'d back, physics resumes only if
  it was running, and `applyFilters()` re-applies the filter state. New clusters created
  during the edit start visible. `posOverride` lets undo restore the *pre-change*
  positions. Used by every editing path.

Build steps, in order:

1. `syncFromData()` — derive flats, enforce integrity.
2. Degree ranking → node sizes (top 8% big, next 15% medium).
3. **Bucket folding** — members of collapsed buckets are skipped; one `bucket:` node is
   added per collapsed bucket (label `📦 name (n)`).
4. Node DataSet — explicit colour from the primary cluster (see §12 for why there is
   deliberately **no vis `group` property**), shape resolution (photo → icon → emoji →
   shape → type default), `_cls` membership array, `fixed:{x,y}` + border highlight for
   pinned nodes.
5. Annotation nodes (`shape:"text"`); anchored ones get `physics:true` plus a `note`-kind
   leash edge (`length:70`, dashed) to their anchor.
6. Relationship edges — length `SPACING × clamp(75..300, 60+11·(deg(a)+deg(b)))`; width by
   `sqrt(amount/maxAmount)`; weak edges dashed. Edges touching a collapsed bucket are
   re-pointed to the bucket node and aggregated per `from|to` pair (count + summed amount
   label); edges fully inside one bucket are dropped.
7. Shared-selector index & edges — values normalised (`norm()`: phones → last 10 digits,
   MACs → hex only), groups of ≤6 get a full mesh, larger groups a star (k−1 edges);
   endpoints are bucket-mapped and de-duplicated. Hidden until `selMode`.
8. Network creation — barnesHut physics scaled by `DATA.config.spacing`
   (`gravitationalConstant`, `springLength`), stabilisation/`improvedLayout` only when not
   preserving, LOD tweaks for >250 nodes (auto-pause physics, hide edge labels when zoomed
   out, hide edges while dragging).
9. Event wiring — click routes to entity detail / bucket detail / note; double-click to
   editor / bucket expand / note delete; `dragEnd` persists positions for notes, buckets,
   and pinned nodes and marks the case dirty.
10. Sidebar renderers: `buildLegends`, `buildRootSel`, `renderShared`, `renderTags`,
    `renderBookmarks`, `renderBuckets`, `updateStats`, `updateSelBar`.

## 5. Undo (`pushUndo`, `doUndo`)

Snapshot-based: `pushUndo(label)` is called **before** any mutation of `DATA` and pushes
`{label, json: JSON.stringify(DATA), pos: capturePositions()}` (max 50, FIFO eviction).
`doUndo()` (Ctrl/Cmd-Z or the toolbar button) restores the JSON, re-normalises, and calls
`build(true, snapshot.pos)` so both the data *and* the arrangement roll back. The keyboard
handler ignores keystrokes while typing in inputs or while a modal is open; the button's
tooltip carries the pending label. Redo is intentionally out of scope. Snapshotting a
full case is O(size of JSON) — instant for cases in the thousands of nodes; photos are the
only heavy payload and are capped at 256 px at upload.

Every mutation path pushes: editor apply/delete, CSV/dashboard import, sample load,
settings apply, link draw, bulk cluster/tag/shape, bucket create/add/remove/expand/
collapse/dissolve/rename, note add/delete, pin/unpin.

## 6. Auto-save (`autoSaveTick`, `syncAutoSaveTimer`)

Design constraints (deliberate): **off by default**, **timer-driven only** (never
per-interaction — rewriting a >1 MB file on every click would tank the graph), **fully
silent** (a background timer may not open pickers — browsers require a user gesture
anyway).

- `markDirty()` flips `DIRTY` on every mutation (all `pushUndo` calls, plus drags);
  Save clears it.
- `autoSaveTick()` — no-ops unless `autosave.on && DIRTY`. Save order: (1) the remembered
  `fileHandle` *if* `queryPermission() === "granted"` (never `requestPermission` — that
  would prompt); (2) `storeSnapshot()` into `localStorage` keyed by case id. Success
  clears `DIRTY` and toasts.
- `syncAutoSaveTimer()` — reconciles the `setInterval` (min 60 s) and the header button
  UI; called at init, on toggle, and after Settings changes. State persists in
  `DATA.config.autosave` (`interval` ms: 60000–300000 slider / 3600000 / 86400000).

## 7. Manual save (`saveFile`)

Priority chain: remembered handle (re-request permission if needed) → `showSaveFilePicker`
once, then remember the handle in IndexedDB (`fhDB`/`fhRemember`/`fhRecall` — survives
reloads) → browsers without the File System Access API store a `localStorage` snapshot
(`storeSnapshot`; on load, a snapshot newer than the file's `savedAt` wins) →
Shift+Save or storage failure falls back to a download.

## 8. Buckets

Buckets are **data-model folding, not vis clustering** — the vis `cluster()` API fights the
filter pipeline, so instead `build()` simply doesn't materialise members of collapsed
buckets and adds a synthetic container node (§4.3/4.6 cover node/edge folding).

`bucketCreate / bucketAddMembers / bucketRemoveMember / bucketExpand / bucketCollapse /
bucketDissolve / bucketLabels` all mutate `DATA.buckets` (with `pushUndo`) and rebuild with
`build(true)`. A node belongs to at most one bucket (enforced at create/add). UI surfaces:
the sidebar **Buckets** panel (`renderBuckets`), the container node itself (click →
`showBucketDetail`, the side-pane explorer with per-member remove/copy/rename/dissolve;
double-click → expand), and the selection bar's **Bucket…** select.
`showDetail2Data` renders a read-only mini-dossier for members while hidden.
Buckets ignore filters (always visible) and are excluded from stats.

## 9. Pinning

`setPinned(ids, pin)` writes `pinned` (+ current x/y) into `DATA.nodes` and flips vis
`fixed:{x:true,y:true}` live. `clearFixed()` (layout switches) and `setPositions()` (manual
layouts) both skip pinned nodes, so pins survive every layout run. `dragEnd` re-records a
pinned node's position after hand-moves. Entry points: selection bar (bulk), detail pane
(single).

## 10. Annotations & anchored notes

`DATA.annotations` entries with an `anchor` field are tied to a node: `build()` gives the
note `physics:true` and a dashed `_kind:"note"` leash edge (`na_<noteId>`), so the physics
engine keeps the note beside its node — zero per-frame bookkeeping. Free notes stay
`physics:false` at fixed x/y. `addNote(anchorId)` anchors when exactly one entity is
selected (or from the detail pane button). `applyFilters` shows a leash edge iff its node
is visible; deleting a node silently unanchors its notes (integrity pass); deleting a note
removes its leash.

## 11. Multi-cluster membership

`node.cluster` (primary — colour, layout grouping) + `node.clusters[]` (extras). The vis
node carries `_cls = [primary, ...extras]`; `applyFilters` shows a node if **any** membership
is active. The editor renders "Also in clusters" checkboxes; the selection bar has both
*Move* (primary, removes the target from extras to avoid duplication) and *Also add*
(extras only, "do not remove from previous clusters"). Settings cluster deletion re-points
primaries to the first remaining cluster and strips dangling extras. Legends count
primaries only.

## 12. The vis-network group-colour pitfall (fixed bug — do not regress)

The bundled vis-network 10.x **re-applies default group palette colours over a node's
explicit `color`** whenever that node passes through `nodes.update({hidden})` while it has
a `group` property — turning nodes pure yellow/magenta on any filter toggle. CaseWeaver
therefore sets **no `group` property at all**; cluster colours are always written
explicitly into each node's `color` object (`build()` step 4, and the bulk cluster-move
handler). If you ever reintroduce `group`, filter toggling will corrupt colours again.

## 13. Filters & derived UI

`applyFilters()` is a single batched pass (one `nodes.update` + one `edges.update` — per-item
updates are O(seconds) at 1k nodes): node visibility = any-cluster ∧ tag ∧ min-connections;
notes and buckets bypass filters; rel edges additionally respect money-flow bounds; sel
edges require `selMode` + type chip + optional focused selector; note leashes follow their
node. `updateStats()` recomputes the header tallies from visible elements.

## 14. Imports & exports

- **CSV import** (`importCSV(text, "replace"|"merge")`) — header-alias resolution
  (`SEL_ALIAS`, `NTYPE_LOOKUP`), relationship vs selector rows, node de-dup by slug,
  per-run `importN` tag. `parseCSV` is a hand-rolled RFC-4180-ish parser (quotes, commas,
  newlines).
- **Dashboard import** (`importDashboard(html)`, v1.001) — extracts the `#case-data` JSON
  from **any** saved CaseWeaver file via `extractDashboardJSON` (regex + JSON.parse +
  shape validation); replace or merge (nodes by id, edges de-duped by `from|to|label`,
  cluster-key collisions renamed side-by-side, annotations by id). Version differences are
  absorbed by `normalizeData()` defaulting.  The dropzone accepts `.html` and sniffs
  `<!doctype`/`<html` for content-based detection.
- **CSV view / export** (`caseCSVRows`, `renderCsvTable`) — canonical type words
  (`CSV_TYPE_WORD`) chosen so `typeCode()` round-trips them; live filter; clipboard copy
  and download.
- **Copy labels** (`copyTextList`) — `navigator.clipboard` with a `prompt()` fallback.
- **PNG** (`graphImageDataURL`) — clones visible nodes/edges at current positions into an
  off-screen fixed network at export resolution, captures the canvas.
- **PDF** (`openReportModal` → jsPDF) — cover, summary, key figures, graph capture,
  storyboard scenes, per-entity dossiers.

## 15. Layout engine

Manual layouts (`layoutSpaced/Grid/Radial/Org/Time`) compute a position map and apply it
via `setPositions` (fixes nodes, physics off, pinned skipped); `layoutForce` clears fixed
(pins survive) and re-enables physics. `assignOrgLevels`/`bfsDepth` derive hierarchy for
org/radial. The layout select and physics button stay in sync via
`syncLayoutBtns`/`syncPhysicsBtn`.

## 16. Other subsystems, briefly

- **Shared-selector focusing** — `focusSelector(key)` selects/zooms the sharing entities
  and activates their clusters. `moreList`/`bindMoreToggles` implement the capped "+N
  more" name lists (text-node replacement, XSS-safe).
- **Trace** — `traceFrom(start, forward)` BFS over directional rel edges, honouring active
  money filters, recolouring the walked chain.
- **Lens** — `neighborhood()` BFS to depth 1–2, dims non-members.
- **Redaction** — display-layer only: `disp()`/`dispSel()` mask label text at render
  points; the data is never altered. As of v1.001 node images/shapes are untouched.
- **Storyboard** — `captureBookmark` serialises the full scene (positions, layout, camera,
  filters, highlights); `applyBookmark` restores it; Present mode pages through bookmarks.
- **Themes** — `applyTheme` writes CSS variables and `GT` (graph colour tokens), then
  recolours the live DataSets.
- **Responsive toolbar** — `syncCramped()` toggles `body.cramped` below 1480 px; CSS
  collapses `#canvasBar` to a ⚙ handle that expands on hover; `#detail` sits above it in
  z-order.
- **Toast / About / branding** — `toast()` transient messages; `openAbout`;
  `applyBrandVisibility` honours `hideBrand`.

## 17. Performance notes

- >250 nodes: physics auto-pauses after stabilisation, edge labels LOD-hide when zoomed
  out, edges hide during drag/zoom, `improvedLayout` off, stabilisation iterations scaled
  down.
- Shared-selector groups >6 members use star topology (k−1 edges, not k²).
- All filter work is batched (§13). Undo snapshots are stringify-based (§5).
- Photos are downscaled to ≤256 px before being stored in the file.

## 18. Extending CaseWeaver

- **New entity type:** add to `NTYPES` (label, tag, shape, aliases) — import aliases,
  legend, editor, and CSV round-trip pick it up; add its canonical word to
  `CSV_TYPE_WORD`.
- **New selector type:** add to `SELTYPES` + `SEL_PLACEHOLDER` + `SEL_ALIAS`; everything
  else (index, chips, editor fields, redaction) is generated from `SELKEYS`.
- **New cyber icon:** add to `CYBERICONS` — `{name, p:[{d:"<svg path>", f?:1}]}` on a
  48×48 viewBox; stroke styling and colourisation are applied by `iconToDataURL`.
- **New theme:** add to `THEMES` with the full `css` variable map and `graph` colour block.
- **Anything that mutates `DATA`:** call `pushUndo(label)` first, then `build(true)` (or a
  targeted `nodes.update` for pure-visual tweaks), and rely on `markDirty()` via
  `pushUndo` for auto-save. Keep the no-`group` rule (§12).

## 19. Testing

The release was verified with headless-Chromium (Playwright) suites covering: scene
preservation across edits, undo (API + Ctrl-Z), pin/unpin, bulk shape/cluster/tag ops,
label copy (clipboard), the full bucket lifecycle, anchored notes, redaction behaviour,
auto-save timer semantics (off-by-default, dirty-gated, snapshot path), settings controls,
legacy-HTML import, spacing defaults, responsive collapse, colour stability across
`hidden` updates (§12 regression), CSV-view/report modals, and a full save → reopen
round-trip of the rewritten file. No console errors under any exercised path.
