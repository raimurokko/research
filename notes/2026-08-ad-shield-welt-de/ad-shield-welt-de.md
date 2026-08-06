# Wenn der Schadcode legal ist: Ad-Shield auf welt.de

## Eine forensische Fallstudie zu server­seitiger Anti-Adblock-Injektion — und warum die Domain-Reputation vor der Code-Analyse kommt

*Erstdokumentation, August 2026*

---

## Kurzfassung

Beim Aufruf von `https://www.welt.de/` mit aktivem netzwerkbasiertem DNS-Filter erscheint eine gefälschte Fehlermeldung, die den Nutzer auffordert, die Domain `html-load.com` freizugeben. Der oberflächliche Befund — ein per Base64 verschleierter Redirect auf eine „Error-Report"-Seite, ausgelöst durch ein im HTML-Quelltext injiziertes, obfuskiertes Script — trägt sämtliche Merkmale einer Website-Kompromittierung.

Er ist keine.

Die verantwortliche Infrastruktur gehört **Ad-Shield**, einem kommerziellen „Adblock-Recovery"-Anbieter. Die Injektion ist eine bezahlte, vom Publisher bewusst eingebundene Funktion. Dieser Bericht dokumentiert den vollständigen Diagnoseweg — einschließlich der zunächst falschen Arbeitshypothese — und zieht daraus eine methodische Lehre: Bei verdächtigem Drittanbieter-Code gehört die Reputationsprüfung der beteiligten Domains an den **Anfang** der Analyse, nicht ans Ende.

---

## 1. Der Auslöser

Sichtbarer Einstiegspunkt war eine Weiterleitung auf eine URL folgender Struktur:

```
https://report.error-report.com/modal?url=<base64>&error=<base64>&domain=html-load.com
```

Die Base64-Parameter dekodieren zu:

- `url` → `https://www.welt.de/`
- `error` → *„Failed to load website properly since html-load.com is blocked. Please allow html-load.com"*

Das Muster ist ein klassischer Anti-Adblock-„Scare": Der Seitenaufbau wird sabotiert, ein Overlay erscheint, und dem Nutzer wird nahegelegt, seine Schutzmaßnahme abzuschalten, um die vermeintlich kaputte Seite zu „reparieren".

---

## 2. Erste Analyse: Konsole und Quelltext

### 2.1 Netzwerk-Konsole

Die Browser-Konsole zeigte den fehlgeschlagenen Request:

```
GET https://html-load.com/vendor.js    (index):4
net::ERR_NAME_NOT_RESOLVED
```

Zwei Beobachtungen:

- **Initiator `(index):4`** — der Request stammt aus Zeile 4 des HTML-Dokuments selbst, nicht aus einem nachgeladenen Werbeskript.
- **`ERR_NAME_NOT_RESOLVED`** — die Domain wird gar nicht erst aufgelöst; ein DNS-Filter greift bereits.

Weitere nicht auflösbare Requests (doubleclick, id5-sync, piano.io) bestätigten einen aktiven netzwerkweiten Tracker-/Ad-Filter.

### 2.2 Quelltext (`view-source`)

Der entscheidende Fund stand direkt im ausgelieferten HTML:

```html
<!-- level: light  trigger: api  domain: welt.de  ts: 1785961800150 -->
<script async id="PjDNysNXd" data-sdk="wp-l/1.1.6" data-cfasync="false"
        nowprocket src="https://html-load.com/vendor.js" charset="UTF-8"
        onload="(async()=>{var t,e,r,o,a,n,i,s,l=1 ...">
<script data-cfasync="false" nowprocket>
        (()=>{var e,t,n,x,s=(n,x,s)=>{for(x=x||n.length,s=s||x,e='',t=0;
        t<s;t++)e+=n[9397*(t+697)%x];return e};const o=['Jg','gX', ...
```

Bemerkenswert:

- Ein **Injektions-Marker** (`level` / `trigger` / `domain` / `ts`) mit tagesaktuellem Unix-Timestamp in Millisekunden → dynamische Einbindung pro Request.
- Ein **obfuskierter Inline-Decoder** im Folge-Tag, der zur Laufzeit Strings aus einem Index-Array rekonstruiert.
- Die Attribute `nowprocket` und `data-cfasync="false"` — gezielt gesetzt, um WP-Rocket und den Cloudflare Rocket Loader daran zu hindern, das Script anzufassen.

An dieser Stelle lautete die Arbeitshypothese: **server­seitige Kompromittierung** im Stil bekannter WordPress-Injection-Kampagnen (Sign1 / Balada). Diese Hypothese war plausibel — und, wie sich zeigte, falsch. Sie prägte jedoch den nächsten, aufwändigen Analyseschritt.

---

## 3. Ausschluss lokaler Ursachen

Da das Script im vom Server empfangenen HTML stand (nicht clientseitig ins DOM geschrieben), und Browser-Extensions Response-Bodies nicht umschreiben können, blieben drei Hypothesen: lokaler MITM-Proxy mit eigener Root-CA, serverseitige Kompromittierung, oder ein Cache-Artefakt. Der frische Timestamp entkräftete Letzteres sofort.

Die lokale Umgebung wurde systematisch geprüft — alle Befunde sauber:

| Prüfung | Befund |
|---|---|
| `scutil --proxy`, `networksetup -get*proxy` | Kein Proxy aktiv |
| System-Keychain (`security find-certificate`) | Nur Apple-CAs |
| Login-Keychain | ProtonVPN Root CA, eigenes Developer-ID-Zertifikat — beide erwartbar |
| `security dump-trust-settings` | Keine manipulierten Trust-Settings |
| `systemextensionsctl list` | ProtonVPN (WireGuard, Split-Tunneling), LuLu — legitim |
| `profiles list` | Keine Konfigurationsprofile |
| `/etc/hosts` | Unverändert |
| `dig www.welt.de` (lokal & `@1.1.1.1`) | Korrekt auf Akamai (`edgekey.net`) |
| LaunchAgents / LaunchDaemons | Nur Microsoft, Docker, Google Keystone, Steam |

Zusätzlich: **Der Effekt trat in Safari und Chrome gleichermaßen auf.** Damit fielen sowohl die Extension- als auch die Proxy-Hypothese praktisch aus.

---

## 4. Der entscheidende Test: unabhängige Verifikation

Wenn das Problem nicht lokal ist, muss es von außen reproduzierbar sein. Zwei unabhängige Scans über **urlscan.io** — von dessen eigener Infrastruktur, nicht vom betroffenen Endgerät [1]:

```
Request Chain 0
  https://html-load.com/vendor.js   HTTP 302
  https://stg.html-load.com/vendor.js
```

Der Befund war identisch reproduzierbar. Ein Redirect `html-load.com → 302 → stg.html-load.com` belegte, dass die Infrastruktur global aktiv antwortet. Das lokale `ERR_NAME_NOT_RESOLVED` kam also **ausschließlich vom eigenen DNS-Filter**, der die Domain bereits auf einer Blockliste führte.

Zwischenstand: Die Injektion kommt von der welt.de-Auslieferung selbst — nicht vom Endgerät. Zu diesem Zeitpunkt schien die Kompromittierungs-These bestätigt.

---

## 5. Die Wende: OSINT vor Code

Bevor eine Incident-Meldung verschickt wurde, erfolgte der Schritt, der eigentlich an den Anfang gehört hätte — eine **Reputationsrecherche der Domain**. Ergebnis:

- **Joe Sandbox** klassifiziert `html-load.com` als Adware mit täuschender Präsentation: Security-Branding nach außen, obfuskierter Code zur Umgehung von Adblockern und Privacy-Tools nach innen [2].
- **Filterlisten-Projekte** führen die Domain samt Verwandten (`html-load.cc`, `css-load.com`, `js-load.com`, `content-load.com`, `content-loader.com`) als **Ad-Shield**-Infrastruktur — dokumentiert u. a. im BadBlock-Tracker, mit Verweisen auf frühere Debunking-Arbeit von 1Hosts und uAssets, nachdem das Script versucht hatte, Adblock-Nutzern die Schuld an „kaputten" Seiten zuzuschieben [3].
- **AdGuard** dokumentiert exakt die beobachtete URL-Struktur (`report.error-report.com/modal?...&domain=html-load.com`) wiederholt als Anti-Adblock-Script — u. a. auf apkmirror.com (März 2025) und gamewith.jp (Dezember 2025); ein koreanisches Issue von Januar 2024 benennt den dahinterstehenden Anbieter explizit als „Ad-Shield" (애드쉴드) [4][5][6].
- Auch die **Gegenrichtung** ist dokumentiert: Im Malwarebytes-Forum beschweren sich Nutzer über einen vermeintlichen „False Positive", weil Websites bei geblockter `html-load.com` nicht mehr richtig laden — exakt der Effekt, den das Script bei blockierter Domain erzeugt, um den Blocker zu diskreditieren [7].
- **AlienVault OTX** führt die Domain als Threat-Indicator mit Community-Pulses [8].

Damit fügte sich jedes einzelne Puzzleteil neu zusammen — und ergab ein völlig anderes Bild:

| Indikator | Kompromittierungs-These (falsch) | Ad-Shield-These (korrekt) |
|---|---|---|
| Server­seitige Injektion, frischer Timestamp | Angreifer in der Lieferkette | Bewusste Publisher-Integration, dynamisch gerendert |
| Marker `level/trigger/domain` | Angreifer-Signatur | Ad-Shield-Konfiguration |
| Obfuskierter Decoder | Malware-Tarnung | Anti-Adblock-Verschleierung |
| `nowprocket` / `data-cfasync` | Umgehung von Schutzmechanismen | Schutz des bezahlten Scripts vor Publisher-Optimierern |
| Fake-Fehlerseite | Scareware / Tech-Support-Scam | Dokumentiertes „Blame-the-Adblocker"-Gaslighting |
| DNS-Block greift | Zufall | Domain steht **wegen** Adware-Klassifizierung auf Filterlisten |

---

## 6. Wer ist Ad-Shield?

Ad-Shield tritt als kommerzieller „Adblock-Recovery"-Anbieter auf: Publisher binden ein Script ein, das mit erheblichem technischen Aufwand — Obfuskierung, wechselnde Domains, Umgehung von Filterregeln — Werbung auch bei aktivem Adblocker ausliefert. Die Filterlisten-Community dokumentiert die zugehörige Infrastruktur seit Jahren und beschreibt einen fortlaufenden Rüstungswettlauf mit den Listen-Autoren [3]. Über die Verbreitung gibt der urlscan-Befund selbst Auskunft: `html-load.com` führt einen Cisco-Umbrella-Rang von ~7.400 — die Domain gehört damit zu den 10.000 meistabgerufenen weltweit, was auf ein Deployment über eine große Zahl von Publishern hindeutet. Das Redirect-Ziel `stg.html-load.com` ist dagegen erst rund drei Monate alt, und das ausgelieferte `vendor.js` trug einen `Last-Modified`-Zeitstempel vom selben Abend — die Infrastruktur und das Payload werden aktiv weiterentwickelt [1]. Auf welt.de weist der Marker `data-sdk="wp-l/1.1.6"` zusammen mit dem per Request generierten Injektions-Kommentar auf eine server­seitige Integrationsvariante hin, die clientseitigen Blockern möglichst wenig Angriffsfläche bietet.

Die betriebswirtschaftliche Logik dahinter ist nachvollziehbar: Publisher verlieren erhebliche Werbeeinnahmen an Adblocker, und „Adblock Recovery" verspricht, diese Reichweite zurückzuholen. Die **Methode** jedoch — obfuskierter Code, gefälschte Fehlermeldungen, das gezielte Beschuldigen der Schutzsoftware des Nutzers — verlässt den Boden des Fairen. Und sie hat einen forensischen Nebeneffekt: Der `<script>`-Tag, den ein Publisher einbindet, ist von einer echten Kompromittierung technisch kaum zu unterscheiden. Genau das macht die Attribution schwierig — und die Reputationsrecherche unverzichtbar.

---

## 7. Indicators (IOCs)

**Domains**
```
html-load.com
stg.html-load.com
error-report.com / report.error-report.com
html-load.cc, css-load.com, js-load.com,
content-load.com, content-loader.com   (verwandte Ad-Shield-Infrastruktur)
```

**Injiziertes Markup**
```html
<!-- level: light  trigger: api  domain: <host>  ts: <unix-ms> -->
<script async id="<random>" data-sdk="wp-l/1.1.6" data-cfasync="false"
        nowprocket src="https://html-load.com/vendor.js" ...>
```

**Redirect-Kette**
```
html-load.com/vendor.js  →  302  →  stg.html-load.com/vendor.js
```

**Payload (Stand 05.08.2026, 21:54 UTC; wird laufend neu gebaut)**
```
vendor.js  SHA-256:
dbfc43dea28cc36ee79e547f7dda5731fb4236e58d88b79dcc50bbf4cc8cb408
(ausgeliefert via Cloudflare, Cache-Control: private, Last-Modified 05.08.2026 20:00 UTC)
```

**Scareware-Modal**
```
report.error-report.com/modal?url=<b64>&error=<b64>&domain=html-load.com
```

---

## 8. Gegenmaßnahmen

Ein **harter DNS-Block** ist kontraproduktiv: Er triggert genau das Fake-Fehler-Overlay, weil das Script das Fehlschlagen des Requests als Anlass nimmt, das CSS zu zerstören und den Nutzer zu belästigen.

Wirksamer sind zwei Ansätze:

1. **DNS-Sinkhole statt Block.** Statt die Domain ins Leere laufen zu lassen, wird sie auf einen lokalen Mini-Webserver umgeleitet, der auf jede Anfrage mit einer leeren, gültigen Antwort (`200 {}`) reagiert. Das Script erhält eine „erfolgreiche" Antwort, verzichtet auf das Overlay — und lädt dennoch keine Werbung. (Community-erprobt u. a. via NGINX auf dem Router.)

2. **Extension-basierter Adblocker.** Regelbasierte Blocker im Browser (uBlock Origin, AdGuard-Extension) können die Injektion clientseitig neutralisieren, bevor der Decoder greift — was ein reiner Netzwerk-Filter nicht leistet.

---

## 9. Methodische Lehre

Der lehrreichste Teil dieses Falls ist der Irrweg. Die Reihenfolge der Analyse war:

1. Symptom beobachtet
2. Code analysiert → Kompromittierungs-These gebildet
3. Lokale Ursachen ausgeschlossen (aufwändig, aber korrekt)
4. Extern verifiziert → These scheinbar bestätigt
5. **Domain-Reputation geprüft → These widerlegt**

Schritt 5 hätte Schritt 2 sein müssen. Sämtliche technischen Indikatoren — server­seitige Injektion, Obfuskierung, Schutz-Umgehungs-Attribute, frischer Timestamp — waren mit **beiden** Erklärungen kompatibel. Was die beiden Hypothesen trennte, war nicht der Code, sondern der **Ruf der beteiligten Domains** — eine Information, die in dreißig Sekunden abrufbar ist.

**Merksatz:** Bevor verdächtiger Drittanbieter-Code als Kompromittierung gewertet wird, gehört die Reputation jeder beteiligten Domain geprüft. Ein einzeiliger `<script src>` einer kommerziellen Adtech-Firma sieht im Quelltext exakt aus wie eine Injektion durch einen Angreifer. Der Unterschied liegt nicht in der Technik, sondern im Absender.

Die gute Nachricht am Rande: Die gesamte lokale Forensik behält ihren Wert. Der geprüfte Rechner war zu jedem Zeitpunkt sauber — und der einzige Grund, warum er je verdächtig wirkte, war ein DNS-Filter, der genau seinen Job tat.

---

## Referenzen

1. urlscan.io — Scans von www.welt.de, 05.08.2026: <https://urlscan.io/result/019fd3eb-fb80-73d9-8cc2-bd1d7fcaeaca/> (21:54 UTC) und <https://urlscan.io/result/019fd3f7-4965-7628-bd2a-a03aa120ccf8/> (22:07 UTC)
2. Joe Sandbox — Automated Malware Analysis Report für http://html-load.com: <https://www.joesandbox.com/analysis/1875962/0/html>
3. BadBlock Issue #78 — „[BLOCK] html-load.com Adware (Related domains)", OSINT-Sammlung zur Ad-Shield-Infrastruktur: <https://github.com/celenityy/BadBlock/issues/78>
4. AdGuard Filters Issue #200730 — report.error-report.com auf apkmirror.com (März 2025): <https://github.com/AdguardTeam/AdguardFilters/issues/200730>
5. AdGuard Filters Issue #221234 — report.error-report.com auf gamewith.jp (Dezember 2025): <https://github.com/AdguardTeam/AdguardFilters/issues/221234>
6. AdGuard Filters Issue #170923 — info.error-report.com, Anbieter als „Ad-Shield" (애드쉴드) benannt (Januar 2024): <https://github.com/AdguardTeam/AdguardFilters/issues/170923>
7. Malwarebytes-Forum — „False positive for html-load.com" (September 2024): <https://forums.malwarebytes.com/topic/316679-false-positive-for-html-loadcom/>
8. AlienVault OTX — Domain-Indicator html-load.com: <https://otx.alienvault.com/indicator/domain/html-load.com>

*Alle Referenzen zuletzt abgerufen am 06.08.2026.*

---

*Dieser Bericht dokumentiert eine öffentlich beobachtbare Konfiguration zum genannten Zeitpunkt. Ad-Shield ist ein legitim registriertes Unternehmen; die hier geübte Kritik richtet sich gegen die eingesetzten Methoden (Obfuskierung, irreführende Fehlermeldungen), nicht gegen die Rechtmäßigkeit des Geschäftsbetriebs. Konfigurationen können sich ändern.*
