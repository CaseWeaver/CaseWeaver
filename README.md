<p align="center">
  <img src="docs/screenshots/02-graph-evidence-room.png" alt="CaseWeaver — link analysis in one file" width="820"/>
</p>

<h1 align="center">CaseWeaver</h1>
<p align="center"><b>The case file that draws itself.</b><br/>
Self-contained link-analysis dashboard in a single HTML file. No install, no server, no account, no network calls.</p>

---

## What it is

Drop a CSV edge list onto `CaseWeaver.html` and it renders an interactive network of your case — people, companies, infrastructure, malware, accounts, groups, locations, financial vehicles, events, and evidence. Work every entity by hand (dossiers, contact selectors, relationships, photos, tags, money flows), then hit **Save** and everything writes back into the file itself.

The graph engine (vis-network) and PDF library (jsPDF) are bundled inside, so it runs identically on a locked-down corporate laptop, an air-gapped forensics box, or a field machine with no internet.

**Who it's for:** fraud and AML investigators, OSINT researchers, corporate security and due-diligence teams, journalists, and cyber threat analysts — anyone who needs to see who connects to whom and hand the answer to someone else without standing up i2, Maltego, or a graph database.

## Quick start

1. Download [`CaseWeaver.html`](CaseWeaver.html) and open it in any modern browser.
2. Click **Load sample case** to explore a worked fraud → mule → laundering operation, or drop in your own CSV.
3. Click any node to open its dossier; **Shift+drag** from a node to draw a new link.
4. Hit **Save case** — the browser downloads an updated copy of the file with all your data inside. Replace your working copy with it. That file *is* the case: copy it for snapshots, drop it on a shared drive, or hand it to counsel.

> 🔐 **Store the file encrypted at rest.** Everything you enter is saved inside the HTML in plain text — keep it on full-disk-encrypted storage (BitLocker, FileVault, LUKS) and share it only over encrypted channels.

## CSV format

One flat file, two kinds of rows:

```csv
source,target,link,source_type,target_type,source_cluster,target_cluster,amount,date
Jane Doe,Acme Holdings LLC,controls,person,company,Operators,Shells,,
PayVehicle Inc.,Offshore Fund Ltd,layering transfer,company,financial,Shells,Shells,262000,2026-05-02
Jane Doe,"1 Main St, Anytown",,person,address,,,,
```

- **Relationship rows** — `source` and `target` are two entities; `link` becomes the edge label. Types: `person`, `company`, `infrastructure`, `account`, `malware`, `group`, `location`, `financial`, `event`, `document` (blank = company). Optional `amount`/`date` enable money-flow analysis.
- **Selector rows** — set `target_type` to a contact field (`address`, `email`, `phone`, `domain`, `id`, `ip`, `device`, `wallet`, `hash`, `url`, `user`, `mac`) and the value attaches to the source entity instead of becoming a node.
- Natural column aliases are accepted (`from`/`to`, `src`/`dst`, `md5`, `handle`, …); repeated names merge into one node; values containing commas go in quotes.

See [`samples/sample-case.csv`](samples/sample-case.csv) for a complete example exercising every field.

## Features

- **Shared-selector engine** — identical values on two-plus entities auto-flag as hidden connections: two shells registered at the same address, two personas reusing a handle, two campaigns beaconing to the same C2 IP. Smart matching (last-10-digit phones, separator-blind MACs).
- **Hand-built dossiers** — role, significance, notes, tags, relationships with amounts and dates.
- **Money flow analysis** — edge thickness by amount, in/out totals per entity, min-amount and date filters, upstream/downstream fund tracing.
- **Six layouts + lens** — force, spaced clusters, grid, radial, org chart, timeline; lens mode spotlights any node's neighborhood.
- **Storyboard & Present mode** — bookmark whole scenes (camera, filters, highlights) and step through them full-screen for briefings.
- **Redaction toggle** — one click masks names and selectors everywhere, including exports.
- **PDF report** — multi-page report (customizable cover, executive summary, key figures, graph, per-entity dossiers) plus PNG graph captures.
- **8 themes** — Evidence room, Counsel copy (print-friendly light), Neon Grid, Ghost Shell, Night Ops, Radar Room, Microfiche, Thermal.
- **Scales past a thousand entities.**

## Screenshots

| | |
|---|---|
| ![Import screen](docs/screenshots/01-import-screen.png) | ![Counsel copy theme](docs/screenshots/03-theme-counsel-copy.png) |
| ![Report dialog](docs/screenshots/04-report-dialog.png) | ![Evidence room](docs/screenshots/02-graph-evidence-room.png) |

## Privacy by design

No telemetry, no cloud, no CDN, no fonts fetched — the file never phones home. Open the network tab and watch nothing happen.

## License

MIT — see [LICENSE](LICENSE). Bundled third-party libraries are listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Contributing

Bug reports and PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). Security issues: see [SECURITY.md](SECURITY.md).
