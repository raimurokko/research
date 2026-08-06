# Novum Analytica — Research Notes

Field notes from security research at [Novum Analytica GmbH](https://novum-analytica.example) — incident forensics, OSINT attribution, vulnerability research, and the occasional methodological lesson learned the hard way.

Notes are published in English where relevant to an international audience; German originals are included where they exist. Each note ships with its indicators (IOCs) in machine-readable form under `evidence/`.

## Notes

| Date | Title | Languages |
|---|---|---|
| 2026-08 | [One Shot: a ClickFix Chain That Cannot Be Analysed Twice](notes/2026-08-clickfix-single-use-gate/) — a single-use gate makes a macOS ClickFix payload unretrievable after first contact, and what that does to the order of an investigation | [EN](notes/2026-08-clickfix-single-use-gate/README.md) · [DE](notes/2026-08-clickfix-single-use-gate/clickfix-single-use-gate.md) |
| 2026-08 | [When the Malware Is Legal: Ad-Shield on welt.de](notes/2026-08-ad-shield-welt-de/) — server-side anti-adblock injection on a top German publisher, and why domain reputation should precede code analysis | [EN](notes/2026-08-ad-shield-welt-de/README.md) · [DE](notes/2026-08-ad-shield-welt-de/ad-shield-welt-de.md) |

The two 2026-08 notes are a matched pair: near-identical surface evidence — an obfuscated script injected into served HTML — with opposite conclusions. Read together they make the case that page source alone cannot separate a compromise from a paid integration.

## Scope & responsible use

These notes document publicly observable behaviour, based on passive observation and open sources. No systems were accessed beyond what any visitor's browser performs. Indicators are shared to support defenders, filter-list maintainers, and fellow researchers.

Corrections and additional sightings are welcome — please open an issue.

## License

Text and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) unless noted otherwise. Indicator lists (`evidence/`) are released without restriction (CC0) to allow unencumbered use in blocklists and detection tooling.
