# Security policy

## Threat model

CaseWeaver is a local, offline tool. It makes no network requests, has no server side, and stores everything you enter inside the HTML file itself, in plain text.

That means:

- **The file is the sensitive asset.** Store it on encrypted-at-rest storage (BitLocker, FileVault, LUKS) and share it only over encrypted channels.
- **Redaction is cosmetic for distribution copies** — a redacted export masks values, but the working file still contains the originals.
- **Opening untrusted case files:** a CaseWeaver file is HTML+JS. Treat a file you didn't create like any untrusted document — inspect or open it in an isolated browser profile.

## Reporting a vulnerability

Please open a private security advisory on GitHub (Security → Advisories → Report a vulnerability) rather than a public issue. Include a proof-of-concept CSV or file where relevant. We aim to acknowledge reports within 72 hours.

In scope: XSS via CSV/dossier fields or imported data, Save round-trip corruption, data exfiltration vectors, PDF generation issues.
