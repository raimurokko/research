# Evidence

Material behind [the note](../README.md). All observations are from
2026-08-05 (UTC); local capture times were CEST (UTC+2).

## Indicators

- `iocs.txt` — indicators, CC0. Every uncommented line is a bare domain, so
  `grep -v '^#' iocs.txt` yields a clean list. Entries that could not be
  confirmed from our own data are commented out on purpose.
- `requests-adshield.csv` — the 149 requests to Ad-Shield hosts, extracted
  from a full-session HAR (3295 entries). Columns: time, method, host, path,
  status, mime, initiator, transfer size.

## Captures

| File | Shows |
|---|---|
| `01_modal-scareware_chrome_2026-08-05T2058Z.png` | The scareware dialog: *"Please allow ads on this site."* |
| `02_modal-scareware_chrome_2026-08-05T2059Z.png` | Same, one minute later |
| `03_console-error-report-endpoint_chrome_2026-08-05T2109Z.png` | DevTools console; the tooltip exposes the `error-report.com/report?type=loader_light…` callback |
| `04_view-source-marker_firefox_2026-08-05T2153Z.png` | Served source in Firefox — injection marker, `ts: …120`, script id `FhEukiprShLX` |
| `05_view-source-marker_chrome_2026-08-05T2246Z.png` | Served source in Chrome — same marker, `ts: …150`, script id `PjDNysNXd` |
| `06_devtools-network-capture_chrome_2026-08-05T2246Z.png` | The DevTools Network session the HAR was taken from |
| `served-source_www.welt.de_2026-08-05T2246Z.html` | Chrome's rendered `view-source:` page for the Chrome capture. The served HTML is HTML-escaped and wrapped in highlighting markup — strip tags and unescape before grepping. |

Screenshots are cropped to the browser window. Nothing inside the window was
altered; the desktop, window switcher, sidebar and dock were removed because
they show unrelated personal content. Unedited originals are held offline.

## Not in this repository

The full HAR (`www.welt.de.har`, 65 MB) is deliberately not committed. It is
mostly headers and timings: only 186 of 3295 entries carry a response body,
and every one of those is JavaScript. It contains no HTML document, and the
strings `html-load.com`, `data-sdk` and the injection marker appear in none of
its bodies — the central evidence is in the served-source file and the
screenshots instead. `requests-adshield.csv` preserves what the HAR actually
proved, at 11 KB. The HAR is available on request.

## A note on the timestamp

The injected marker carries `ts: 1785961800xxx` = 2026-08-05 20:30:00 UTC.
Both captures show that same second — `…120` in Firefox at 21:53 UTC and
`…150` in Chrome at 22:46 UTC, 53 minutes apart. The timestamp therefore marks
when the response was **generated or cached**, not when it was requested. The
script `id`, by contrast, differs between the two responses.

This matters for the note's section 3, which treats the "fresh timestamp" as
grounds for ruling out a cache artefact. That step does not hold: the
timestamp is stable across captures. The conclusion is unaffected — the
injection is Ad-Shield's, confirmed by domain reputation, not by the
timestamp — but the reasoning step was an over-reading.
