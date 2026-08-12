# When the Malware Is Legal: Ad-Shield on welt.de

## A forensic case study in server-side anti-adblock injection — and why domain reputation should come before code analysis

*First documentation, August 2026*

---

## TL;DR

Visiting `https://www.welt.de/` — one of Germany's largest news sites — with a network-level DNS filter active produces a fake error message urging the user to unblock the domain `html-load.com`. The surface-level evidence — a Base64-obfuscated redirect to an "error report" page, triggered by an obfuscated script injected into the page's HTML source — carries every hallmark of a website compromise.

It isn't one.

The infrastructure behind it belongs to **Ad-Shield**, a commercial "adblock recovery" vendor. The injection is a paid feature, deliberately integrated by the publisher. This report documents the full diagnostic path — including the initially wrong working hypothesis — and draws a methodological lesson from it: when analysing suspicious third-party code, checking the reputation of the involved domains belongs at the **beginning** of the investigation, not at the end.

---

## 1. The trigger

The visible entry point was a redirect to a URL of the following structure:

```
https://report.error-report.com/modal?url=<base64>&error=<base64>&domain=html-load.com
```

The Base64 parameters decode to:

- `url` → `https://www.welt.de/`
- `error` → *"Failed to load website properly since html-load.com is blocked. Please allow html-load.com"*

The pattern is a classic anti-adblock scare: sabotage the page rendering, display an overlay, and nudge the user into disabling their own protection to "fix" the supposedly broken site.

---

## 2. Initial analysis: console and page source

### 2.1 Network console

The browser console showed the failing request:

```
GET https://html-load.com/vendor.js    (index):4
net::ERR_NAME_NOT_RESOLVED
```

Two observations:

- **Initiator `(index):4`** — the request originates from line 4 of the HTML document itself, not from a lazily loaded ad script.
- **`ERR_NAME_NOT_RESOLVED`** — the domain never resolves; a DNS filter is already intercepting it.

Further unresolvable requests (doubleclick, id5-sync, piano.io) confirmed an active network-wide tracker/ad filter.

### 2.2 Page source (`view-source`)

The decisive finding sat directly in the served HTML:

```html
<!-- level: light  trigger: api  domain: welt.de  ts: 1785961800150 -->
<script async id="PjDNysNXd" data-sdk="wp-l/1.1.6" data-cfasync="false"
        nowprocket src="https://html-load.com/vendor.js" charset="UTF-8"
        onload="(async()=>{var t,e,r,o,a,n,i,s,l=1 ...">
<script data-cfasync="false" nowprocket>
        (()=>{var e,t,n,x,s=(n,x,s)=>{for(x=x||n.length,s=s||x,e='',t=0;
        t<s;t++)e+=n[9397*(t+697)%x];return e};const o=['Jg','gX', ...
```

Notable details:

- An **injection marker** (`level` / `trigger` / `domain` / `ts`) carrying a same-day Unix timestamp in milliseconds → the tag is inserted server-side, not written into the DOM by client-side code. (The original reading of this marker — that the tag is rendered *per request* — did not survive the evidence review; see the correction in section 10.)
- An **obfuscated inline decoder** in the adjacent tag, reconstructing strings from an index array at runtime.
- The attributes `nowprocket` and `data-cfasync="false"` — deliberately set to prevent WP-Rocket and Cloudflare Rocket Loader from touching the script.

At this point the working hypothesis was: **server-side compromise** in the style of well-known WordPress injection campaigns (Sign1 / Balada). That hypothesis was plausible — and, as it turned out, wrong. It nevertheless shaped the next, labour-intensive step of the analysis.

---

## 3. Ruling out local causes

Since the script appeared in the HTML *as received from the server* (not written into the DOM client-side), and browser extensions cannot rewrite response bodies, three hypotheses remained: a local MITM proxy with its own root CA, a server-side compromise, or a cache artefact. The timestamp in the marker was read as "fresh" and taken to eliminate the cache option — **that step was wrong**, and it is corrected in section 10. What actually ruled out a local cache was the independent reproduction from urlscan's infrastructure in section 4.

The local environment was checked systematically — every result came back clean:

| Check | Result |
|---|---|
| `scutil --proxy`, `networksetup -get*proxy` | No proxy active |
| System keychain (`security find-certificate`) | Apple CAs only |
| Login keychain | ProtonVPN root CA, own Developer ID certificate — both expected |
| `security dump-trust-settings` | No tampered trust settings |
| `systemextensionsctl list` | ProtonVPN (WireGuard, split tunneling), LuLu — legitimate |
| `profiles list` | No configuration profiles |
| `/etc/hosts` | Untouched |
| `dig www.welt.de` (local & `@1.1.1.1`) | Correctly resolving to Akamai (`edgekey.net`) |
| LaunchAgents / LaunchDaemons | Only Microsoft, Docker, Google Keystone, Steam |

Additionally: **the effect occurred identically in Safari, Chrome and Firefox** — three independent engines (WebKit, Blink, Gecko), not merely three applications. That effectively eliminated both the extension and the proxy hypotheses: no extension is installed across all three, and a local proxy would have had to forge certificates for three separate trust stores. Captures exist for Chrome and Firefox (`evidence/`); the Safari observation was not screenshotted.

---

## 4. The decisive test: independent verification

If the problem isn't local, it must be reproducible from the outside. Two scans via **urlscan.io** — executed from urlscan's own infrastructure, not the affected device [1]:

```
Request Chain 0
  https://html-load.com/vendor.js   HTTP 302
  https://stg.html-load.com/vendor.js
```

The finding reproduced identically. A redirect `html-load.com → 302 → stg.html-load.com` confirmed that the infrastructure responds globally. The local `ERR_NAME_NOT_RESOLVED` therefore came **exclusively from the local DNS filter**, which already carried the domain on a blocklist.

Interim conclusion: the injection originates from welt.de's own delivery chain — not from the endpoint. At this point, the compromise hypothesis appeared confirmed.

---

## 5. The turn: OSINT before code

Before any incident report went out, the step that should have come first was finally taken — a **reputation check on the domain**. The results:

- **Joe Sandbox** classifies `html-load.com` as adware with a deceptive presentation: security-themed branding on the surface, obfuscated code designed to circumvent ad blockers and privacy tools underneath [2].
- **Filter-list projects** track the domain and its relatives (`html-load.cc`, `css-load.com`, `js-load.com`, `content-load.com`, `content-loader.com`) as **Ad-Shield** infrastructure — documented in the BadBlock tracker, with references to earlier debunking work by 1Hosts and uAssets after the script had tried to make adblock users blame their own blocker for "broken" pages [3].
- **AdGuard** has repeatedly documented the exact observed URL structure (`report.error-report.com/modal?...&domain=html-load.com`) as an anti-adblock script — including on apkmirror.com (March 2025) and gamewith.jp (December 2025); a Korean issue from January 2024 explicitly names the operator behind it as "Ad-Shield" (애드쉴드) [4][5][6].
- The **inverse perspective** is documented as well: in the Malwarebytes forums, users complained about a supposed "false positive" because websites stopped loading properly with `html-load.com` blocked — precisely the effect the script produces when its domain is blocked, in order to discredit the blocker [7].
- **AlienVault OTX** lists the domain as a threat indicator with community pulses [8].

With that, every piece of the puzzle rearranged itself — into an entirely different picture:

| Indicator | Compromise hypothesis (wrong) | Ad-Shield hypothesis (correct) |
|---|---|---|
| Server-side injection, timestamped marker | Attacker in the supply chain | Deliberate publisher integration, rendered server-side |
| Marker `level/trigger/domain` | Attacker signature | Ad-Shield configuration |
| Obfuscated decoder | Malware camouflage | Anti-adblock obfuscation |
| `nowprocket` / `data-cfasync` | Evasion of protection mechanisms | Protecting the paid script from the publisher's own optimisers |
| Fake error page | Scareware / tech-support scam | Documented "blame the adblocker" gaslighting |
| DNS block firing | Coincidence | Domain is on filter lists **because** it is classified as adware |

---

## 6. Who is Ad-Shield?

Ad-Shield operates as a commercial "adblock recovery" vendor: publishers embed a script that — through considerable technical effort involving obfuscation, rotating domains, and filter-rule evasion — delivers ads even with an active ad blocker. The filter-list community has documented the associated infrastructure for years and describes an ongoing arms race with list maintainers [3]. The urlscan data itself speaks to the scale of deployment: `html-load.com` carries a Cisco Umbrella rank of roughly 7,400 — placing it among the 10,000 most-requested domains worldwide, indicating deployment across a large number of publishers. The redirect target `stg.html-load.com`, by contrast, is only about three months old, and the served `vendor.js` carried a `Last-Modified` timestamp from the same evening — both the infrastructure and the payload are under active development [1]. On welt.de, the marker `data-sdk="wp-l/1.1.6"` together with the server-side injection comment points to an integration variant designed to give client-side blockers as little attack surface as possible.

The business logic is understandable: publishers lose substantial ad revenue to blockers, and "adblock recovery" promises to reclaim that reach. The **method**, however — obfuscated code, fabricated error messages, deliberately blaming the user's own protection software — leaves fair ground behind. And it has a forensic side effect: the `<script>` tag a publisher embeds is technically almost indistinguishable from a genuine compromise. That is precisely what makes attribution hard — and reputation research indispensable.

---

## 7. Indicators (IOCs)

The full list, with the confirmation status of each entry, is in
[`evidence/iocs.txt`](evidence/iocs.txt).

**Domains — directly observed**
```
html-load.com            fb.html-load.com
content-loader.com       fb.content-loader.com
fuel.norexceptdrug.com
stg.html-load.com
error-report.com         report.error-report.com
```

`fuel.norexceptdrug.com` is worth singling out: a nonsense-word host with
exactly one urlscan submission ever (2026-08-05), serving the same two files
as `html-load.com` — freshly rotated infrastructure.

**Domains — inherited from filter lists, NOT confirmed here**
```
html-load.cc, css-load.com, info.error-report.com
content-load.com   — no urlscan scans; on ALL-INKL shared hosting rather than
                     Cloudflare like every confirmed host. Likely a false
                     positive; blocking it risks an uninvolved customer.
js-load.com        — no DNS delegation at all. Not verifiable.
```

**Serving pattern** (more durable than any single domain)
```
<host>/vendor.js  and  <host>/main.js
fb.<host>/app.js
```

**Injected markup**
```html
<!-- level: light  trigger: api  domain: <host>  ts: <unix-ms> -->
<script async id="<random>" data-sdk="wp-l/1.1.6" data-cfasync="false"
        nowprocket src="https://html-load.com/vendor.js" ...>
```

**Redirect chain**
```
html-load.com/vendor.js  →  302  →  stg.html-load.com/vendor.js
```

**Payload (as of 2026-08-05, 21:54 UTC; rebuilt continuously)**
```
vendor.js  SHA-256:
dbfc43dea28cc36ee79e547f7dda5731fb4236e58d88b79dcc50bbf4cc8cb408
(served via Cloudflare, Cache-Control: private, Last-Modified 2026-08-05 20:00 UTC)
```

**Two distinct endpoints**
```
# shown to the user
report.error-report.com/modal?url=<b64>&error=<b64>&domain=html-load.com

# silent callback, seen in the DevTools console
error-report.com/report?type=loader_light&url=<b64>&error=<b64>&request_id=<b64>
```
The apex host and the `request_id` distinguish the second one. Its value
`loader_light` matches the `level: light` field in the injected marker — the
integration tier is reported back.

---

## 8. Countermeasures

A **hard DNS block** is counterproductive: it triggers exactly the fake error overlay, because the script treats the failing request as a cue to break the page's styling and harass the user.

Two approaches work better:

1. **DNS sinkhole instead of block.** Rather than letting the domain resolve to nothing, redirect it to a small local web server that answers every request with an empty, valid response (`200 {}`). The script receives a "successful" reply, skips the overlay — and still loads no ads. (Community-proven, e.g. via NGINX on a router.)

2. **Extension-based ad blocker.** Rule-based blockers inside the browser (uBlock Origin, the AdGuard extension) can neutralise the injection client-side before the decoder runs — something a pure network filter cannot do.

---

## 9. The methodological lesson

The most instructive part of this case is the detour. The order of the investigation was:

1. Symptom observed
2. Code analysed → compromise hypothesis formed
3. Local causes ruled out (labour-intensive, but correct)
4. Externally verified → hypothesis seemingly confirmed
5. **Domain reputation checked → hypothesis refuted**

Step 5 should have been step 2. Every single technical indicator — server-side injection, obfuscation, protection-evasion attributes, the timestamped marker — was compatible with **both** explanations. What separated the two hypotheses was not the code but the **reputation of the involved domains** — information retrievable in thirty seconds.

**Rule of thumb:** before suspicious third-party code is classified as a compromise, the reputation of every involved domain deserves a check. A one-line `<script src>` from a commercial adtech firm looks, in the page source, exactly like an injection by an attacker. The difference lies not in the technique but in the sender.

The good news on the side: all of the local forensics retains its value. The examined machine was clean at every point — and the only reason it ever looked suspicious was a DNS filter doing exactly its job.

---

## 10. Correction (2026-08-06)

Reviewing the captured evidence for publication turned up an error in this note's own reasoning. It is left visible rather than quietly edited out, because it is the same failure mode section 9 is about.

**The claim.** Sections 2.2 and 3 read the `ts` field in the injection marker as a *fresh, per-request* timestamp, and section 3 used that freshness to rule out a cache artefact.

**The evidence.** Two captures of the served source exist: Firefox at 21:53 UTC (`ts: 1785961800120`, script id `FhEukiprShLX`) and Chrome at 22:46 UTC (`ts: 1785961800150`, script id `PjDNysNXd`). Fifty-three minutes apart, both carry the same second — 2026-08-05 20:30:00 UTC — differing only by 30 ms.

**The correction.** `ts` marks when the response was **generated or cached**, not when it was requested. It therefore cannot rule out a cache artefact; it is in fact consistent with one. The script `id` *does* differ between the two responses, so the tag is generated in variants — but the timestamp does not track the request.

**What this changes.** Nothing in the conclusion. The injection is Ad-Shield's, and that rests on domain reputation (section 5) and on independent reproduction from urlscan's infrastructure (section 4) — not on the timestamp. What it changes is the honesty of the chain: one supporting step was an over-reading of a field whose meaning had not been established. Which is, precisely, the lesson of section 9 — applied this time to the author.

---

---

## 11. Re-verification (2026-08-07)

Re-checked two days after publication, prompted by an observation that "the content on
welt.de is gone" and a question whether it had been malware after all.

It is not gone, and it was not malware. The integration is still served, from the same
origin, with the same construction:

| | 2026-08-05 | 2026-08-07 |
|---|---|---|
| Script host | `html-load.com` | `html-load.com` — unchanged |
| Path | `/vendor.js` | `/main.js` |
| `data-sdk` | `wp-l/1.1.6` | **`wp-l/1.1.8`** |
| Element `id` | `PjDNysNXd` | `sOwIET` — randomised per response, as before |
| `nowprocket`, `data-cfasync="false"` | present | present |
| Obfuscated inline decoder | index-array reconstruction | same construction, different constants |
| `level` / `trigger` / `domain` / `ts` marker | present | **not found** |

**Why it looked gone.** The scareware modal only fires when `html-load.com` fails to load.
From this vantage point the domain now resolves normally (Cloudflare, `104.18.20.31` /
`104.18.21.31`), so nothing breaks, so no overlay appears. The trigger changed, not the
page. That is precisely the mechanism described in section 1, seen from the other side —
and a reminder that "the symptom stopped" is not evidence about the cause.

**Two SDK minor versions in two days.** This is the part worth keeping. Compromises do not
ship incremented release numbers. A vendor maintaining a product does. The version bump,
the path rename and the rotated decoder constants are all consistent with routine
deployment by a commercial supplier and inconsistent with an injection someone left behind
on a breached host — which strengthens the conclusion of section 5 by a route that was not
available when it was written.

**The marker is gone.** The `level` / `trigger` / `domain` / `ts` object no longer appears.
Section 10 already withdrew the reading that it was rendered per request; its
disappearance is consistent with that withdrawal, and it removes the artefact that
produced the original wrong hypothesis in the first place.

Nothing in sections 1–10 needs correcting as a result. The conclusion holds and is now
supported by a second, independent observation two days apart.

## References

1. urlscan.io — scans of www.welt.de, 2026-08-05: <https://urlscan.io/result/019fd3eb-fb80-73d9-8cc2-bd1d7fcaeaca/> (21:54 UTC) and <https://urlscan.io/result/019fd3f7-4965-7628-bd2a-a03aa120ccf8/> (22:07 UTC)
2. Joe Sandbox — Automated Malware Analysis Report for http://html-load.com: <https://www.joesandbox.com/analysis/1875962/0/html>
3. BadBlock issue #78 — "[BLOCK] html-load.com Adware (Related domains)", OSINT collection on Ad-Shield infrastructure: <https://github.com/celenityy/BadBlock/issues/78>
4. AdGuard Filters issue #200730 — report.error-report.com on apkmirror.com (March 2025): <https://github.com/AdguardTeam/AdguardFilters/issues/200730>
5. AdGuard Filters issue #221234 — report.error-report.com on gamewith.jp (December 2025): <https://github.com/AdguardTeam/AdguardFilters/issues/221234>
6. AdGuard Filters issue #170923 — info.error-report.com, operator named as "Ad-Shield" (애드쉴드) (January 2024): <https://github.com/AdguardTeam/AdguardFilters/issues/170923>
7. Malwarebytes forums — "False positive for html-load.com" (September 2024): <https://forums.malwarebytes.com/topic/316679-false-positive-for-html-loadcom/>
8. AlienVault OTX — domain indicator html-load.com: <https://otx.alienvault.com/indicator/domain/html-load.com>

*All references last accessed 2026-08-06.*

---

*This report documents a publicly observable configuration at the time stated. Ad-Shield is a legally registered business; the criticism expressed here targets the methods employed (obfuscation, misleading error messages), not the legality of the operation. Configurations may change.*
