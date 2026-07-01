# Satyanweshi

**See what the internet knows about you — then clean it up.**

Satyanweshi (সত্যান্বেষী, *"truth-seeker"*) is a personal exposure audit tool. You give it the identifiers you actually use — a name, usernames, email addresses, a phone number — and it fans out across public sources to show you exactly what is discoverable about you online. It then turns that into a prioritised, actionable cleanup plan: which accounts to delete, which passwords to rotate, which data brokers to opt out of.

The name comes from Sharadindu Bandyopadhyay's Byomkesh Bakshi, who refused to be called a detective and insisted on *satyanweshi* instead. That is the spirit here: this is not a tool for investigating strangers. It is a tool for finding the truth about yourself.

> **Status:** early development. The CLI is the first deliverable; a web dashboard follows. Expect breaking changes until `v0.1.0`.

---

## Why this exists

Most people have no idea how much of their footprint is public: a decade-old forum account tied to a username they still use, a breached password that is also their banking password, a people-search site listing their home address, EXIF GPS coordinates in a photo they posted once. The tools to find this already exist in the OSINT community, but they are built for investigators, scattered across dozens of repos, and stop at *finding*. Satyanweshi stitches the best of them together behind one command and, crucially, tells you what to do about each result.

## What it does

A scan runs in four stages:

1. **Seed** — you provide identifiers (name, usernames, emails, phone, optional photos and a domain you own). Email and phone seeds must be verified before they are scanned; see [Ethics & safety](#ethics--safety).
2. **Collect** — pluggable collectors query public sources in parallel, each wrapping a proven open source project.
3. **Normalise & score** — every result becomes a uniform `Finding` with a category, confidence level, evidence, and severity. Findings roll up into an **exposure score** per category and overall.
4. **Remediate** — each finding is mapped to a concrete action with a direct link: account deletion page, password-reset URL, data-broker opt-out form, or a how-to for things like stripping image metadata. Re-running a scan later shows what actually disappeared.

### Finding categories

| Category | Examples |
|---|---|
| **Identity** | Full name associated with usernames, profile photos, bios |
| **Accounts** | Services where a username or email is registered |
| **Breaches** | Appearances in known data breaches, exposed passwords |
| **Contact** | Emails, phone numbers, and the services linking them |
| **Location** | Addresses on people-search sites, EXIF GPS, geotagged posts |
| **Footprint** | Search-engine mentions, archived pages, domain WHOIS records |

## Quick start

```bash
pipx install satyanweshi          # or: pip install satyanweshi
satya init                         # one-time: configure optional API keys (HIBP etc.)
satya scan --username prasunc --email you@example.com
satya report --format html --open  # open the latest report in your browser
```

A scan with no API keys still works using the free sources. Adding a [Have I Been Pwned](https://haveibeenpwned.com/API/Key) key unlocks breach lookups.

### Example output

```
$ satya scan --username prasunc --email you@example.com

  Satyanweshi · scan 2026-10-08T10:42Z

  Exposure score  ████████░░  62 / 100   HIGH

  Breaches   3 findings   ● 1 critical    password exposed in 2 breaches
  Accounts  14 findings   ● 4 high        9 accounts match username, 5 confirmed via email
  Location   1 finding    ● 1 high        address listed on 1 people-search site
  Footprint  6 findings   ○ low

  Top actions
   1. Rotate password on gmail.com, enable 2FA        → https://myaccount.google.com/security
   2. Opt out of spokeo.com listing                   → https://www.spokeo.com/optout
   3. Delete unused account on last.fm (difficulty: easy) → https://www.last.fm/settings/delete

  Full report: ~/.satyanweshi/reports/2026-10-08T1042.html
```

## Architecture

```
satyanweshi/
├── cli/            # Typer-based CLI: init, scan, report, verify
├── core/
│   ├── models.py   # Seed, Finding, Action, ScanResult (pydantic)
│   ├── scoring.py  # severity weights → category & overall exposure score
│   └── engine.py   # orchestrates collectors, dedup, normalisation
├── collectors/     # one adapter per source; all implement Collector protocol
│   ├── base.py
│   ├── maigret.py
│   ├── holehe.py
│   ├── hibp.py
│   └── ...
├── remediation/    # maps finding types → Actions using open datasets
│   ├── rules.py
│   └── data/       # vendored JustDeleteMe + data-broker opt-out lists
├── report/         # JSON, terminal, and HTML renderers
└── verify/         # ownership verification for emails / phones
```

**Design principles**

- **Collectors are adapters, not forks.** Each wraps an existing project through its Python API or as a subprocess with JSON output, so upstream improvements flow in and licences stay isolated.
- **One schema in, one schema out.** Every collector emits `Finding` objects; everything downstream (scoring, remediation, reporting) only ever sees that schema.
- **Confidence is first-class.** Username collisions are common. A match on `jsmith` is reported as *possible*, a match confirmed by email probe as *confirmed*. Scores weight accordingly.
- **Remediation is data, not code.** Action mappings live in versioned datasets so they can be updated without a release.

## Sources & credits

Satyanweshi stands on the shoulders of the OSINT community. Collectors wrap or consume:

| Source | Used for | Licence |
|---|---|---|
| [Maigret](https://github.com/soxoj/maigret) | Username enumeration across thousands of sites | MIT |
| [Sherlock](https://github.com/sherlock-project/sherlock) | Username enumeration (alternate engine) | MIT |
| [WhatsMyName](https://github.com/WebBreacher/WhatsMyName) | Site-pattern dataset | CC BY-SA 4.0 |
| [holehe](https://github.com/megadose/holehe) | Email → registered services | GPL-3.0 (subprocess) |
| [ignorant](https://github.com/megadose/ignorant) | Phone → registered services | GPL-3.0 (subprocess) |
| [h8mail](https://github.com/khast3x/h8mail) | Breach aggregation | BSD-3 |
| [Have I Been Pwned](https://haveibeenpwned.com/API/v3) | Breaches & Pwned Passwords (k-anonymity) | API |
| [GHunt](https://github.com/mxrch/GHunt) | Google account footprint | AGPL-3.0 (subprocess) |
| [theHarvester](https://github.com/laramies/theHarvester) | Search-engine harvesting | GPL-2.0 (subprocess) |
| [ExifTool](https://exiftool.org/) | Image metadata | Perl Artistic |
| [waybackpy](https://github.com/akamhy/waybackpy) | Archived pages | MIT |
| [JustDeleteMe](https://github.com/jdm-contrib/jdm) | Account deletion directory | MIT |
| [Big Ass Data Broker Opt-Out List](https://github.com/yaelwrites/Big-Ass-Data-Broker-Opt-Out-List) | Data-broker removal procedures | CC BY 4.0 |

GPL/AGPL tools are invoked as separate processes and are never imported into the Satyanweshi codebase, which keeps the core MIT-licensed.

## Ethics & safety

This tool can only be legitimately used on **yourself**. To keep it that way:

- **Ownership verification.** Email and phone seeds must be verified (a code is sent to them) before any collector touches them. Username and name seeds carry a mandatory self-attestation and are rate-limited.
- **No stored raw breach data.** Breach lookups use HIBP's range-query API; plaintext passwords are never retrieved or stored.
- **Local by default.** Scan results live in `~/.satyanweshi/` on your machine, encrypted at rest. Nothing is uploaded unless you explicitly export it.
- **Polite collection.** Collectors respect rate limits and `robots.txt`, and use only the probe techniques their upstream projects already use.

If you find this tool being misused, or find a safety gap, please open an issue or email the maintainer privately.

## Roadmap

- [ ] `v0.1` — CLI with Maigret, holehe, HIBP collectors; JSON + terminal report; basic remediation rules
- [ ] `v0.2` — HTML report, exposure scoring, ownership verification, JustDeleteMe + broker opt-out datasets
- [ ] `v0.3` — Phone, image-metadata, archive and search-engine collectors; re-scan diffing
- [ ] `v0.4` — FastAPI backend + web dashboard; scheduled re-scans
- [ ] Later — plugin API for third-party collectors, localisation

## Contributing

Issues and PRs are welcome. The most useful contributions right now are new collector adapters and remediation rules. See `CONTRIBUTING.md` (coming with `v0.1`) for the collector interface and testing conventions.

## Licence

MIT © Prasun Chakraborty. Third-party tools retain their own licences as listed above.
