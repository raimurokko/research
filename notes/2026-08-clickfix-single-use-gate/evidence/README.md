# Evidence

- `iocs.txt` — indicators, CC0. Every uncommented line is a bare domain, so
  `grep -v '^#' iocs.txt` yields a clean list. Unconfirmed entries are
  commented out on purpose.

## What is not here, and why

**No screenshot of the lure page.** The capture shows the affected site's
domain in the address bar and twice in the page body. Redacting that leaves an
image that proves nothing, so it is withheld rather than published in a form
that would be worthless anyway. The site is a victim; see section 8 of the note.

**No copy of the notification sent to the site operator.** It contains the
organisation's postal address, phone number, mailboxes, and the name and work
address of an individual data protection officer — an uninvolved third party.

**No second stage.** It was never retrieved. The gate honours each token once,
so by the time the chain was decoded there was nothing left to fetch. This is
the subject of the note rather than a gap in it.

## Where the primary artefact is

The clipboard payload — 477 bytes, MD5 `2169ff5e7be77fc3ff72758f9fa50658` — is
attacker-controlled and published in full, together with the detection rules
written from it and the 29-domain cluster analysis:

**<https://github.com/raimurokko/macos-threat-tracking>**

It is kept there rather than duplicated here: two copies of the same indicator
set drift apart, and the YARA rules submitted to YARAhub already carry that
repository as their reference link. This note is the narrative; that repository
is the working material.
