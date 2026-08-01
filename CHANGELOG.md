# Changelog

## v1.001 — 2026-08-01

This release folds in every change from the previously-unreleased internal builds (the 2.x
line, through **CaseWeaver-2_3_2**) and adds a major batch of new workflow features on top.
The in-app version string is now `v1.001` to match the repository.

### Carried over from internal build 2_3_2 (never published to GitHub)

- **CSV view** — tabular view of the whole case (top-bar button): filter rows live, copy
  them to the clipboard, or download the case as a round-trippable CSV.
- **Round-trip CSV** — downloaded CSVs re-import cleanly; every import run is tagged
  (`import1`, `import2`, …) so successive merges stay traceable.
- **Reworked save system** — save-in-place via the File System Access API with a remembered
  file handle (the picker appears only once; later saves rewrite the same file), an
  in-browser snapshot fallback for Safari/Firefox (reopening the same file restores your
  edits), and Shift+Save for a portable downloaded copy.
- **Connections filter Clear button.**

### New features

- **Auto-save** — toggle in the top bar next to **Save case**, **off by default**. Saves on
  a timer only — never after every interaction, so it cannot bog the graph down. Interval is
  set in Settings: a 1–5 minute slider, 1 hour, or 1 day. Uses the remembered file handle
  when available, otherwise the browser snapshot; it never opens dialogs on its own.
- **Undo** — multi-level undo (up to 50 steps) for every data mutation: entity edits and
  deletes, imports, cluster moves, tags, buckets, notes, pins, settings. `Ctrl/Cmd-Z` or the
  ↩ Undo button in the graph toolbar.
- **Stable graph arrangement** — editing a dossier, drawing a link, or any other change no
  longer resets node positions, camera, zoom, physics pause state, or active filters. The
  scene picks up exactly where you left it.
- **Pin / anchor nodes** — pin any selection (or a single node from its detail pane) so it
  holds its spot while physics keeps arranging everything else. Pinned positions save with
  the file.
- **Buckets** — collapse a multi-selection into a single expandable container node.
  Explore a bucket's contents in the side pane, remove members individually or dissolve the
  whole bucket, expand it back onto the graph and re-collapse it, and copy every member
  label as a pasteable list. Edges from bucketed nodes re-route to the bucket (aggregated,
  with link counts and summed amounts).
- **Copy labels** — copy the labels of any multi-selection (or of a bucket) to the
  clipboard as a plain pasteable list.
- **Bulk shape change** — set the node shape for an entire selection at once.
- **Multi-cluster membership** — a node can now belong to several clusters. Adding it to
  another cluster never removes it from its existing ones; the primary cluster keeps
  setting the colour. Editable per-node (checkboxes in the editor) or in bulk from the
  selection bar. Cluster filtering shows a node if *any* of its clusters is active.
- **Notes anchored to nodes** — `+ Note` with a node selected (or from its detail pane)
  ties the note to that node with a short leash so it follows the node around. Free-floating
  notes still work as before.
- **Six new cyber icons** — VOIP infrastructure, soft phone, cell phone, landline phone,
  SIP gateway, and ISP, with matching CSV type aliases (`voip`, `sip_gateway`, `isp`, …).
- **Wider default spacing** — default edge length and repulsion raised (1.5×) so nodes
  overlap far less out of the box, plus a 0.5×–3× spacing slider in Settings for hairball
  cases that previously required pausing physics and hand-placing nodes.
- **Compact shared-selector lists** — both the sidebar list and the detail pane now cap the
  entity-name lists at three with a "+N more" expander, so heavily-shared selectors no
  longer flood the panels.
- **Cleaner relationship view** — the detail pane groups connections into Outgoing /
  Incoming with counts, de-duplicates multiple links to the same counterpart, and collapses
  long lists behind a "show more".
- **Redaction fix** — redaction now masks only the visible label (and selectors); the
  node's photo, icon, and shape are left untouched.
- **Responsive graph toolbar** — on narrow windows the graph-settings bar collapses to a
  slim ⚙ handle that expands on hover, so it can no longer cover the entity name in the
  detail pane; the detail pane also always stacks above it.
- **Import from older CaseWeaver dashboards** — drop any saved CaseWeaver `.html` file
  (back to the first release) onto the import screen to load it, or merge its entities,
  links, clusters, and notes into the current case.
- **New documentation** — [USER_GUIDE.md](USER_GUIDE.md) (full walkthrough of every
  feature, with screenshots) and [CODE_DOCUMENTATION.md](CODE_DOCUMENTATION.md)
  (architecture, data model, and embedded logic for code review and dev hand-off).

### Fixed

- **Cluster colours no longer corrupt on filter changes.** The bundled vis-network 10.x
  re-applies default group colours over explicit ones whenever a node is shown/hidden;
  nodes now carry explicit colours only, so toggling clusters, tags, or filters can no
  longer turn parts of the graph yellow/magenta.
- Deleting an entity cleans up bucket memberships and note anchors that referenced it.

## v1.0 — 2026-07-26

Initial public release.

- Instant graph from a CSV edge list (relationship + selector rows, natural column aliases)
- 10 entity types, 12 selector types, smart matching (last-10-digit phones, separator-blind MACs)
- Shared-selector engine: identical values on 2+ entities auto-flag as hidden connections
- Hand-built dossiers: role, significance, notes, tags, photos, emoji/cyber icons, relationships with amounts & dates
- Money-flow analysis: edge width by amount, in/out totals, min-amount & date filters, fund tracing
- Six layouts (force, spaced clusters, grid, radial, org chart, timeline) + lens mode
- Storyboard bookmarks & full-screen Present mode
- Redaction toggle (applies to exports)
- Multi-page PDF report with fully customizable cover, and PNG graph export
- 8 colour themes; floating-panel workspace with canvas toolbar
- Save writes all case data back into the HTML file itself
