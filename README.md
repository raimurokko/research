# Novum Analytica — Research Notes

Field notes from security research at [Novum Analytica GmbH](https://novum-analytica.example) — incident forensics, OSINT attribution, vulnerability research, and the occasional methodological lesson learned the hard way.

Notes are published in English where relevant to an international audience; German originals are included where they exist. Each note ships with its indicators (IOCs) in machine-readable form under `evidence/`.

## Notes

| Date | Title | Languages |
|---|---|---|
| 2026-08 | [One Shot: What a Single-Use Gate Actually Protects](notes/2026-08-clickfix-single-use-gate/) — a single-use gate puts a clock on an investigation rather than a wall; includes a published correction after the payload was recovered two days later | [EN](notes/2026-08-clickfix-single-use-gate/README.md) · [DE](notes/2026-08-clickfix-single-use-gate/clickfix-single-use-gate.md) |
| 2026-08 | [When the Malware Is Legal: Ad-Shield on welt.de](notes/2026-08-ad-shield-welt-de/) — server-side anti-adblock injection on a top German publisher, and why domain reputation should precede code analysis | [EN](notes/2026-08-ad-shield-welt-de/README.md) · [DE](notes/2026-08-ad-shield-welt-de/ad-shield-welt-de.md) |

The two 2026-08 notes are a matched pair: near-identical surface evidence — an obfuscated script injected into served HTML — with opposite conclusions. Read together they make the case that page source alone cannot separate a compromise from a paid integration.

## Classification

**Everything in this repository is TLP:CLEAR.** Notes, indicators and evidence may be
redistributed without restriction, including publicly. No note here carries a more
restrictive marking; if a case cannot be published at TLP:CLEAR, it does not go in this
repository.

TLP:CLEAR is the current designation under [FIRST TLP 2.0](https://www.first.org/tlp/) for
what older documents call TLP:WHITE. The two mean the same thing; TLP:WHITE has been
deprecated since 2022 and is not used here.

## Scope & responsible use

These notes document publicly observable behaviour, based on passive observation and open sources. No systems were accessed beyond what any visitor's browser performs. Indicators are shared to support defenders, filter-list maintainers, and fellow researchers.

**Affected sites are named only when they are the subject of the finding and not a victim
of it.** A publisher that deliberately integrates anti-adblock scareware is named. A school
whose website was compromised by a third party is not, and neither are the other
compromised sites appearing in a campaign — they are kept out of the indicator files
entirely, because blocking them punishes their own users.

Corrections and additional sightings are welcome — please open an issue.

## License

Text and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) unless noted otherwise. Indicator lists (`evidence/`) are released without restriction (CC0) to allow unencumbered use in blocklists and detection tooling.
