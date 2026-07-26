# Contributing

Thanks for helping make CaseWeaver better.

## Ground rules

- **One file.** The product is a single self-contained HTML file. No build step, no npm, no external requests at runtime. Contributions that add network calls or split the app into served assets will be declined.
- **Offline first.** Everything must work from `file://` on an air-gapped machine.
- **Case data stays put.** The `#case-data` JSON block and the Save round-trip are load-bearing; changes to serialization need a migration note.

## Developing

1. Open `CaseWeaver.html` in a browser. That's the dev environment.
2. Edit with any text editor; the app code lives at the bottom of the file, below the bundled libraries.
3. Test with `samples/sample-case.csv` and with a saved copy from an older version (round-trip compatibility).

## Pull requests

- Keep diffs surgical — this is a large single file.
- Describe the investigator-facing behavior change, with before/after screenshots for UI work.
- Verify: import, Save round-trip, PDF report, PNG export, redaction, Present mode, both light and dark themes.

## Reporting bugs

Use the issue templates. Include browser + OS, and if possible a minimal CSV that reproduces the problem (never real case data).
