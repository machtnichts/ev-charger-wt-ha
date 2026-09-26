# EV-CHARGER-WT-HA — Zustand, Regeln, offene Punkte

Stand: 2026-09-19. **Bei Widersprüchen gilt der Code und die Tests, nicht diese Datei.**
Diese Datei ist die Übergabe: sie sagt, was läuft, warum es so ist, was als Nächstes ansteht.

## Worum es geht (und warum)

Der Besitzer hat den früher eingesetzten Fremd-Controller (Docker-Container `evcc`) durch
eigene, unabhängige Apps ersetzt — er wird **nicht mehr gestartet**, er würde um die Wallbox
kämpfen. Grund (wörtlich): er will
nicht, dass seine Apps von etwas abhängen — "in diesem Fall von Python. if the system
changes, it can blow up my setup". Endziel: **statische Rust-Binaries mit minimalen
Abhängigkeiten**. Die Python-Versionen bleiben unverändert bestehen; Rust ist additiv, und
portiert wird Schicht für Schicht, sobald eine Schicht eingefroren ist. Der
Steuerungsplan ist bewusst eingefroren ("i probably will never need the plan - so let it
as it is now").

Arbeitsweise, die er erwartet: er rechnet selbst nach und hakt nach ("bist du sicher?"),
will Messwert und Vermutung getrennt, schreibt Deutsch und erwartet knappe, belegte
Antworten mit konkreten Zahlen. Vorgaben für dieses Projekt:

* **Keine zusätzliche Poll-Last auf dem Wechselrichter (SE5000H, 192.168.178.84:1502).**
  Diagnose durch Beobachtung, nicht durch mehr Anfragen.
* Ein Gerät, das nur eine Sitzung verträgt, hat **genau eine** Sitzung (SE-Wechselrichter,
  Deye-Logger). Nie zwei Clients gleichzeitig.
* An der Wallbox wird **nur `amx` geschrieben, nie `amp`** ("amp will turn the flash of
  the wb to trash very soon").
* Handmodus = **null Schreibzugriffe**.
* Änderungen an der echten Anlage vorher ankündigen, danach den Zustand belegen.
* Elektroinstallation wird nicht zu Testzwecken geschaltet — Sicherungen werden mit
  eingeschleusten Zeiten/Stubs getestet, nicht am Auto.

## Was jetzt läuft

| Was | Unit / Weg | Zustand |
|---|---|---|
| Laderegler | `systemctl --user status evcharge-wt.service` | aktiv, Web-UI `127.0.0.1:7080`, Intervall 30 s (Auto dran) / 300 s (leer) |
| Modbus-Proxy (Rust) | `systemctl --user status muxproxy-rs.service` | aktiv, hört `0.0.0.0:1503`, Status `:1504`, Build `abda451e49a433ff0b42df8635e512491dba3b0f` |
| Deye-PV-Logger (Rust) | `powerdash-deye-pv-rs` | aktiv, anderes Gerät, nicht Teil der Ladekette |
| evcc | Docker-Container | **stillgelegt** — nicht neu starten; der Laderegler steuert die Wallbox |

Adressen: Wallbox go-e `192.168.178.22` (FW 041.0, HTTP-API v1) · Wechselrichter
`192.168.178.84:1502` · Proxy-Adresse für Consumer `192.168.178.44:1503` · HA
`http://homeassistant:8123` (HAOS 2026.9.2).

Der Lesepfad zum Wechselrichter hat sich als empfindlich erwiesen und ist eingefroren:
**drei kleine Fenster pro Zyklus** — `(40071,32)` Wechselrichter, `(40190,53)` Zähler,
`(57716,18)` Vendor, zusammen 103 Register. Der Bereich `40111..40189` wird bewusst
**nie** angefasst. Vorbild war die alte openHAB-Konfiguration (`se4k.things`): zwei
kleine Fenster alle 10 s, nie ein Register einzeln.

Der Proxy gibt **keine abgelaufenen Frames** zurück (so entschieden). Innerhalb von
`ondemand_ttl` (10 s) teilen sich zwei Leser einen Geräte-Read; danach ist der Wert
verfallen, und wenn nichts Neues gelesen werden konnte, bekommt der Client den **Fehler**
(Exception `0x0B`) statt eines alten Frames — der sähe auf der Leitung wie eine frische
Messung aus und die Lade-App hätte keine Chance, ihn zu erkennen. Folge für die App: ein
fehlgeschlagener Read ergibt gar keine Entscheidung (kein Zyklus), und eine laufende
Ladung wird nach `site_stale_s` (600 s) beendet — nie auf Basis alter Zahlen gestartet oder
nachgeregelt. Rust-Proxy und Python-Referenz verhalten sich gleich, der Konformitätstest
pinnt es (`an_expired_read_is_reported_instead_of_served_stale`).

Die Wallbox wird über die **gemessenen** `nrg`-Offsets gelesen (FW 041.0, die Doku liegt
um eine Stelle daneben): `nrg[0..2]` Volt · `nrg[3]` konstante 1 · `nrg[4..6]` Ströme
(0,1 A) · `nrg[7..9]` Leistungen (0,1 kW) · `nrg[11]` Gesamtleistung (10 W) · `nrg[12..14]`
Leistungsfaktor. Phasenzahl kommt aus den Strömen; `pha` (63) ist Kontaktorbestückung und
wird nie geglaubt.

## Regeln im Regler (mit Begründung)

* **Überschuss** = Netz-Leistung (vorzeichenbehaftet) + Auto-Leistung − Batterie-Abgabe,
  dazu die Batterie-**Aufnahme**, sobald der Akku `priority_soc` erreicht hat. Keine
  Grundlast-Konstante: die Hauslast steht schon im Zählerwert, eine zweite Subtraktion
  zählte sie doppelt und hielt die Ladung ~1 A zu niedrig (`residual_power_w` ist eine
  bewusste Reserve und hier 0). Die Abgabe bleibt immer abgezogen — das Auto entlädt
  niemals die Hausbatterie.
* **Drei Bänder für die Batterie** (so entschieden; Verhalten aus einer generischen EV-Ladeapp übernommen):
  **unter `priority_soc` (55 %)** hat die Batterie Vorrang: ihr Ladeanteil bleibt bei ihr,
  das Auto lädt aus dem echten Export (in dem der Anteil schon fehlt, der Zähler misst ihn
  mit) — es wird aber **nicht** gesperrt. Ein grauer Tag mit halbvollem Akku lädt das Auto
  also, statt die Sonne ins Netz zu schieben. Zusätzlich stoppt die Ladung, wenn die
  Batterie das Auto speist und nichts exportiert wird.
  **ab `priority_soc`** wird der Ladeanteil der Batterie aufgeschlagen: das Auto überholt
  die Batterie beim Laden, der Akku bleibt auf seinem Stand statt auf 100 % zu laufen.
  Mehr als `max_current` (14 A) kann das Auto nicht übernehmen — der Rest geht weiter in
  den Akku bzw. ins Netz.
  **über `buffer_soc` (80 %)** darf die Reserve oberhalb des Buffers eine **laufende**
  Ladung tragen: pv/minpv hält das Auto bei 6 A, statt bei nachlassender Sonne
  abzuschalten; das geht so lange, bis der Akku wieder auf 80 % ist. Eine Ladung wird
  **nie** aus der Batterie gestartet.
  Die Abgabe der Batterie bleibt sonst abgezogen — das Auto entlädt die Hausbatterie nicht.
  `battery_boost` und `buffer_start_soc` sind **entfernt** (Reste der alten
  Buffer-Sperre; für "Akku ins Auto entladen" nimmt der Besitzer den Modus `manual`).
  Alles drei steht im UI im Tooltip der Zeile "battery" und an den Feldern buffer/priority SOC.
* **Strom folgt dem Überschuss sofort** im Bereich 6 A…Maximum. Hysterese gibt es nur an
  der Untergrenze, und sie ist **asymmetrisch** (seit 21.09.2026): ein **Start** braucht die
  Untergrenze **+100 W** (`enable_threshold_w`, so vom Besitzer gesetzt — ein später Start
  verschenkt Sonne), eine **laufende** Ladung wird bis **−300 W** darunter **gehalten**
  (`disable_threshold_w`, bewusst größer: ein unnötiger Stopp kostet eine Sitzung) und erst
  darunter abgeschaltet. Bei 6 A / 1 Phase also: Start ab **1480 W**, Halten bis **1080 W**.
  Im Band dazwischen bleibt der Zustand, wie er ist — genau das Fenster, in dem die App
  vorher im Minutentakt gestartet und gestoppt hat.
* **Hausbatterie-Wächter hängt an der Entscheidung, nicht am Modusnamen**: beim
  Überschussladen wird unter `buffer_soc` nicht geladen (und der Grund angezeigt); im
  Billigfenster ist die Batterie nicht das Thema, weil dort das Netz zahlt.
* **Billigzeitfenster (`cheap_hours`)**: im Fenster ist alles egal — Maximum und
  durchgehend bis zum Ende, kein Warten, keine Batteriesperre, kein Herunterregeln, keine
  Zählerfrische-Prüfung. Am Ende **kein** Abschalten, sondern nahtlose Übergabe an die
  PV-Regeln; die gelten davor, währenddessen und danach, damit der Modus dauerhaft an
  bleiben kann. Eine laufende Ladung muss im Fenster **gehalten** werden — genau das war
  der Fehler, der eine Nacht lang alle ~5 Minuten geschaltet hat (69 Flanken).
  **Behoben am 19.09. um 07:46** (`controller.py`: Zweig ohne `and not charger.charging`);
  drei Checks in `test_controller.py` pinnen es („a running charge in the window stays at
  maximum", „no dwell timer or battery block interrupts it", „and it emits no on/off
  edge"). In `pv`/`minpv` bleibt das Fenster wirkungslos — nachts passiert dort nichts,
  und das ist so gewollt.
* **Phasen**: nach dem Anstecken wird **1 Phase** angenommen; der Zähler wird erst
  geglaubt, wenn ~20 s Strom geflossen ist, und dann der **höchste** gesehene Wert bis zum
  Abstecken gemerkt.
* **Sicherung**: Die App zählt die **Stopps** (`alw=0` nach einem `alw=1`), die die Wallbox
  wirklich gesehen hat, gleitend über 30 Minuten, und zeigt sie im UI. Bei 5 rastet eine
  **Störung** ein: Ladung einmal einschalten, danach **keine** Schreibzugriffe mehr. Grund:
  "OBC eines Autos zu reparieren kostet Tausende Euros". **Nur Stopps, nicht beide
  Richtungen** (Entscheidung des Besitzers, 21.09.2026): jeder Start ist das Gegenstück
  eines Stopps, das Zählen beider Richtungen meldete eine unterbrochene Sitzung als zwei
  Ereignisse und halbierte die Toleranz. `amx=`-Schreibungen zählen **nie** — Nachregeln ist
  kein Schalten. **Freigabe von Hand — und ein Neustart ist eine solche Handlung:
  `systemctl --user restart evcharge-wt.service` quittiert die Störung mit Absicht**
  (Entscheidung des Besitzers, 21.09.2026). Der Zähler lebt nur im Speicher, und das ist so
  gewollt: wer neu startet, hat die Ursache vorher angesehen. **Nicht** gewollt ist das
  Gegenteil — dass die Störung sich während des Laufs von selbst löst, weil der Zähler
  altert; im Betrieb bleibt der Riegel bestehen.
* **Session-Energie wird doppelt gemessen** (seit 22.09.2026). Die App rechnet die
  go-e-Session aus den Phasenmessungen der Wallbox — über die **gemessene** Zykluszeit
  (nicht `interval_s`, das ist nur das Ziel) und mit **Reset beim Abstecken**, damit
  „Session" wirklich eine Ansteck-Sitzung meint. Parallel führt sie eine zweite Zahl über
  den **SDM630** im Garagenstrang (`ha-app/evcharge/session_meter.py`): beim Anstecken
  werden dessen kWh-Zähler gemerkt, während der Sitzung ist die Differenz die Session, beim
  Abstecken wird sie als „letzte Session" eingefroren und **in `logs/sdm_sessions.csv`
  geschrieben** (eine Zeile je Session, mit dem go-e-Wert zum Vergleich). Der SDM630 misst
  den Garagenstrang, in den auch die Garage-PV einspeist — die Zahl ist also die **Sicht des
  Zählers** (Auto minus PV-Anteil); Export wird mitgeführt, damit der PV-Anteil sichtbar bleibt.
  **Drei Werte, so gewünscht** (22.09.2026): (1) go-e, (2) SDM = `import − export`,
  (3) **SDM + Garage-PV** = `import − export + PV` — die Bilanz des Strangs, denn genau das
  hat das Auto gezogen und der Zähler nicht gesehen. Wert 3 ist die Meinung des
  Wechselrichters und **nur so gut wie dessen Zähler**: über das Fenster 21.09. 05:00–17:00Z
  besteht eine **Lücke von bis zu 8 %** (Wechselrichter 4,18 kWh gegen
  3,88 kWh, die der SDM hinausfließen sah) — **Obergrenze, keine Messung**: der Strang trägt
  dauerhaft Router, Tor und go-e-Standby, und der SDM kann so kleine Lasten nicht **zählen**
  (Startstrom 0,04 A), sein Exportzähler liest also genau das zu wenig, was sie verbrauchen.
  Die 0,30 kWh Lücke sind 25 W Dauerlast über 12 h; die Schätzung des Besitzers (Router ~10 W
  + go-e ~5 W) deckt davon schon 0,18 kWh. Der echte Fehler des Wechselrichters liegt damit
  zwischen ~0 % und +8 %. Und nach jedem Aufwachen meldet das Register kurz
  **0,00 kWh** (Poller-Journal 05:16:53) — solche Werte werden verworfen und gezählt
  (`pv_artefacts`), sonst würde die nächste echte Zahl als ~279 kWh „Korrektur" erscheinen.
  Fehlt der Zähler ganz (nachts schläft der Logger, die PV ist dann wirklich 0), ist die
  Korrektur **0** und die CSV-Zeile sagt es; kommt er mitten in der Session, wird die Basis
  nachgeholt und die Zeile nennt die Korrektur „teilweise".
  **Frische:** ein Zähler, der sich nicht ändert, wird von HA **nicht** neu geschrieben — sein
  Alter sagt also nichts. Deshalb entscheidet das Alter der **Leistung** über „stale"
  (`entity_live` für den SDM, `entity_pv_power` für den Deye), und bei veralteten Werten
  **wartet** die Session, statt 0 kWh zu erfinden.
* **Der Wechselrichter wird nur gelesen, nie geschrieben** (23.09.2026, Entscheidung des
  Besitzers: „Ich will nicht Akku steuern"). Der SE-Treiber enthielt zwei **nie aufgerufene**
  Schreibfunktionen (Akkumodus `0xE00D`, Entladegrenze `0xE010`) und der Modbus-Client die
  Schreibprimitive — alles **entfernt**, samt der ebenfalls toten Export-Limit-Konstanten
  (`0xE000..0xE002`). Das ist jetzt **strukturell geprüft** (`test_solaredge_decode`): der
  Client hat keine Schreibmethode, der Treiber nichts, was den Akku steuert, und die
  Registeradressen dürfen im *Code* nicht wieder auftauchen (in der Modul-Doku stehen sie
  weiter, damit klar ist, *warum* sie weg sind). **Live-Wächter:** der Proxy zählt
  `upstream_writes`, und die App stuft jede Zahl > 0 als **„PROXY WROTE TO THE DEVICE" (bad)**
  ein. Stand: **0 Schreibzugriffe** bei 59k Poll-Zyklen. Geschrieben wird ausschließlich die
  **Wallbox** (Strom, Freigabe, Neutralstellung) — und nur über den einen Schreibkanal.
* **Hauszeit ist nicht Hostzeit**: Der Host läuft UTC, gewünschte Wanduhrzeiten sind in
  `Europe/Berlin` ausgedrückt (`Settings.timezone`, IANA-Name, sommerzeitfest).

## PV-Prognose — Schritt 1: nur Anzeige (23.09.2026)

Der Besitzer will vormittags das **Auto** bevorzugt haben und nachmittags den **Akku** —
entschieden nach Wetter, nicht nach Uhrzeit. Dafür braucht es eine lokale Prognose.
**Gebaut ist Schritt 1: die Prognose wird angezeigt und täglich protokolliert; sie steuert
NICHTS.** So kann er erst beurteilen, ob so eine Prognose für sein Dach taugt.

* **Quelle:** Open-Meteo, `global_tilted_irradiance` je Dachfläche, ohne Schlüssel, ein Abruf
  pro Stunde und Fläche. Drei Flächen, wie der Besitzer sie korrigiert hat: **4,48 kWp Ost
  (az −90, 14 Module)**, **1,60 kWp West-Dach (az +90, 5 Module)** und **1,92 kWp West-Gaube
  (az +90, 6 Module, flacher als das Dach)**. Standort ist **der genaue Punkt der
  Anlage** — er steht in Home Assistant und in der lokalen `config.json` und absichtlich **nicht**
  in diesem Repo (vorher stand hier der PLZ-Mittelpunkt, der 820 m daneben lag und heute 0,2 kWh
  weniger prognostizierte). Rechnung: `GTI (W/m²) x kWp = Wh` je Stunde (DC-Seite), `x PR 0,85`
  = AC-Erwartung.
* **Belegt gegen sieben Tage mit 15-Minuten-Exporten aus dem SE-Portal** (24.09.2026; jede Datei
  summiert sich exakt auf den Portal-Tageswert, die Zeitstempel sind Ortszeit — geprüft, indem die
  Tagesform gegen die Prognose korreliert wurde, nicht angenommen). Prognose gegen den Nachmittag
  (12–18 Uhr, aus den 15-Minuten-Werten):
  05.09 **33,1 kWh Prognose / 18,8 kWh Nachmittag**; 06.09 **33,2 / 19,2**; 08.09 **31,3 / 17,9**
  — drei Tage über **88 % der Tagesobergrenze**, dreimal ein **starker** Nachmittag.
  21.09 25,4 / 14,1; 07.09 26,4 / 13,3; 10.09 27,8 / **8,9** — drei Tage bei **71–78 %**,
  dreimal mittel bis schwach. **Zwischen 14,1 und 17,9 kWh liegt keine einzige Beobachtung** —
  die beiden Gruppen überschneiden sich nicht. Daraus folgt die Schwelle: nicht 70 %, sondern
  **~88–90 %**. Mit 90 % vom 30-Tage-Bestwert trifft die Regel **7 von 7** Tagen richtig.
* **Widerlegt: die Tagesfaktor-Korrektur** (also die frühere Idee in diesem Abschnitt). 08.09. und
  10.09. sind Spiegelbilder: am 08.09. war der Vormittag **tot** (2,56 von 6,88 kWh) und der
  Nachmittag **stark** (1,13-fach); am 10.09. war der Vormittag **exakt wie prognostiziert** (0,98)
  und der Nachmittag brach auf **0,51** ein. Der Vormittag sagt über den Nachmittag also nichts —
  in *keine* Richtung. Ein um 12 Uhr gebildeter Tagesfaktor hätte am 10.09. „alles bestens" gesagt
  und den Akku zugunsten des Autos leerlaufen lassen. Der Faktor wird nur noch **mitgeschrieben**,
  nicht mehr verfolgt.
* **Nebenbefund, der die Kalibrierung erklärt:** über 23 Tage sah die Prognose-Tagessumme
  unverzerrt aus (Mittel 1,008) — aber der **Abend** (Akku-Entladung, 1,13- bis 1,58-fach) verdeckt,
  dass der **Nachmittag** im Mittel nur **0,71** der Prognose liefert. Für diese Regel zählt der
  Nachmittag, nicht die Tagessumme: **die Tagessumme taugt als Anzeige, nicht als Aussage über die
  Tagesform.** Deshalb ist die Tagesform (Vormittag/Mittag/Nachmittag/Abend) jetzt Teil jeder
  Tageszeile.
* **Die Form der Messwerte kommt aus dem Stundenschrieb** (`logs/pv_hourly.csv`, kumulativ je
  Stunde) — er existiert genau deshalb, bevor die Regel existiert.
* **Pin (die Zusage von Schritt 1):** der Controller **kennt das Wort `forecast` nicht**
  (`tests/test_pv_forecast.py` prüft das, plus: das Modul hat kein Aktuator-Vokabular). Die
  Prognose *kann* nichts schalten, solange diese Prüfung grün ist.
* **Der Faktor `factor()`** = gemessen heute / Prognose für genau dieses Fenster, nur mit
  echter Basis: unter **0,05 kWh** Messung gibt es **keinen** Faktor (der Tag hat noch nicht
  angefangen — still), außerhalb **0,25–1,60** eine Warnung und ebenfalls keinen. Kein Faktor
  heißt: der spätere Regler fällt auf die sonnenstands-relative Notlösung zurück, nie auf
  geratene Zahlen.
* **Daten — alles auf Platte, nichts nur im Speicher** (ein Neustart darf die Historie nicht
  kosten): `logs/pv_forecast_today.json` wird laufend überschrieben (ein Neustart setzt den Tag
  fort), `logs/pv_forecast.csv` bekommt je **fertigem Tag eine Zeile**: Prognose (gesamt **und** in
  vier Tagesabschnitten), beide Messwerte (AC- und Array-Seite), beide Faktoren,
  Haus-/Auto-/Akku-Energie, SOC-Bereich, `samples` — und die Regel-Spalten `best30_kwh` (Referenz),
  `best30_threshold_kwh`, `best30_pct`, `rule_best_says` (Regel B) und `rule_margin_says` (Regel A).
  Dazu `logs/pv_days_seed.csv` mit den **22 Tageswerten aus dem PV-Portal-Export**, damit die
  30-Tage-Referenz nach einem Neustart sofort existiert (aktuell der Bestwert **33,498 kWh** vom
  06.09.).
  **Seit 25.09. steht auch der Garagenzähler in der Zeile** (`sdm_import_kwh`, `sdm_export_kwh`
  und die Tagesdifferenz `sdm_import_day_kwh`/`sdm_export_day_kwh`) — das ist die Referenz des
  Besitzers fürs Auto. Anlass: am **23.09. gingen 19,38 kWh ins Auto**, während der Dienst noch
  nicht lief; die Tageszeile bucht dort `car_kwh 0.0`. Die Zählermitführung in derselben Zeile
  macht solche Lücken sichtbar, statt sie Wochen später von Hand zu finden (Summe 22.–25.09.:
  **29,22 kWh** am Zähler gegen 7,63 kWh in der App-Zählung). Die Tagesdifferenz entsteht aus dem
  letzten Abschluss; am ersten Tag bleibt sie leer statt geraten.
* **`samples` ist wichtig:** die Tageswerte sind eine Zero-Order-Hold-Auslesung, sie erben die
  Lesekadenz des Wechselrichters (mit Auto alle ~5 s, ohne Auto bewusst gedrosselt — die
  Regel „kein zusätzlicher Poll-Verkehr" gilt auch hier). Bei veralteten Messwerten integriert
  der Regler **nichts** (Stale-Gate), statt alte Werte weiterzuzählen.
* **Die Produktion kommt jetzt aus dem Wechselrichter-Zähler, nicht aus einem Integral**
  (23.09.2026): SunSpec model 101 `WH` (Wort 22/23, Skalenfaktor bei 24) liegt **im ohnehin
  gelesenen Fenster** — kostet also **null** zusätzlichen Modbus-Verkehr (ein Test pinnt: es
  bleiben drei Lesevorgänge mit 103 Registern) — und ist exakt: er verliert nichts, während der
  Dienst steht, und er zählt die spätere **Akku-Entladung mit** (das ist die Zahl, die die
  Monitoring-App „Produktion" nennt, und sie ist fair: gezählt wird einmal, was der
  Wechselrichter abgegeben hat). Live belegt: über dieselben 3,5 Minuten stieg der Zähler um
  **0,0200 kWh** und das AC-Integral der App um **0,0200 kWh** — identisch. Lebensdauerstand
  23.09.: **29.267,336 kWh** (Skalenfaktor 0).
* **Achtung beim Vergleichen nach einem Neustart:** das Integral wird aus der Tagesdatei
  **fortgesetzt**, der Zähler-Startpunkt nur dann, wenn die Tagesdatei einen hat — direkt nach
  einem Neustart können die beiden Zahlen also **verschiedene Fenster** abdecken. Vergleichen
  heißt: **Deltas über dasselbe Fenster**, nie die Gesamtwerte (der erste Blick zeigte deshalb
  0,100 gegen 0,032 kWh und war kein Fehler). Ein Startpunkt, der nicht um Mitternacht gesetzt
  wurde, ist in der Zeile als `se_partial` markiert.
* **Regel B ist gebaut — als Anzeige mit Historie** (24.09.2026): die Prognose wird gegen den
  **besten vollständigen Tag der letzten 30** gestellt, Schwelle **90 %**; darüber „Auto-Vorrang",
  darunter „Akku-Vorrang". Die Referenz kommt aus der eigenen Messreihe, damit sie mit der
  Jahreszeit mitwächst (Juni ~52 kWh, September ~33,5 kWh). Beide Regeln werden **täglich
  mitgeschrieben**, die Historie entscheidet später, welche Recht hatte. Im UI stehen beide:
  „forecast vs best day (30d)" und „rule B (season) would say". **Es steuert weiterhin nichts** —
  der Controller-Pin ist unverändert grün.
* **Offen (Steuerung, wartet auf die gesammelten Tage):** erst verdrahten, wenn die Historie die
  Regel bestätigt. Die Schätzgrößen von Regel A (`house_reserve_kwh`, Marge 1,3) bleiben
  Platzhalter und werden für Regel B voraussichtlich nicht gebraucht.

## Zuletzt behoben (25.09.2026)

* **Batterie-Runde: vier Geräte zurück, und eine Fehldeutung korrigiert.** Treppe Keller, Erstes
  Geschoss Treppe, TreppeEG Rechts und Button Altar kamen nach Zellwechsel von selbst zurück.
  **Button Mascha PC** zusätzlich nach **Neuanlernen in ZHA** — die Entität
  `switch.hobeian_zg_101zl_4` bleibt dabei unverändert (ZHA hängt anhand der IEEE-Adresse an das
  bestehende Gerät: kein `_5`, keine Leiche, Automationen bleiben gültig). Ein Tastendruck allein
  weckt ein Gerät ohne Netzanmeldung nicht, egal wie oft. Fünfter Taster derselben Bauart und
  intakt: `switch.button_pc_wt` („Button PC WT", schaltet den PC ab, Batterie meldet frisch).
* **„stumm seit 19.09." war eine Fehldeutung.** Das ist nur der Zeitstempel, den HA beim Neustart
  auf die Entitäten schreibt. Der **letzte echte Kontakt** steht geräteweise in ZHA (`zha/devices`
  → `last_seen`, `lqi`, `available`): alle noch stummen Batteriegeräte hatten zuletzt vor **88 bis
  369 Tagen** gesendet — **kein** Gerät ist am 19.09. ausgefallen. Triage-Regel daraus: unter ~8
  Tagen = echter Kandidat für eine Zelle, Monate/Jahre = Karteileiche (Ersatz, abgebaut) — da hilft
  keine Batterie.
* **Drei Karteileichen deaktiviert** (namenlose ZG-101ZL `…a2:81:f4` und `…95:40:dc` sowie das
  namenlose `_TZ3000_zutizvyk TS0203`) über `config/device_registry/update` mit `disabled_by: user`
  — reversibel mit `null`. Beleg: Entitäten in HA **605 → 590**, Rücklesen `disabled_by=user`. Der
  Stumm-Alarm nennt sie nicht mehr: er hat keine fest verdrahteten Namen, sondern scannt dynamisch,
  und deaktivierte Entitäten verlassen die Zustandsmaschine. **WasserSensor Heizung** (88 d) ist am
  selben Abend nach Zellwechsel **von selbst** wieder eingebucht (100 %, LQI 148) — es braucht also
  nicht immer ein Neuanlernen; **Eingangstür** (121 d) bleibt absichtlich stehen (angeblich in
  Betrieb, Zuordnung noch offen: es gibt daneben den lebenden Zwilling `AqaraSensorSZTür` — dessen
  Öffnungs-Entität steht dauerhaft auf **offen**, weil die Schlafzimmertür praktisch immer gekippt
  ist; das ist korrekt und **kein** Defekt, Temperatur und Batterie melden frisch). Neu
  aufgefallen und noch ungeklärt: **`Thermostat-Ralf-Keller-Party-Z`** (still seit 20.02.2026).
* **Zwei Wassersensoren haben jetzt einen fetten Telegram-Alarm** (HA-nativ, gleicher Bot und
  gleiche Gruppe wie die Batterie-Alarme): `wasser_leck_heizung` (`binary_sensor.wassersensor_heizung`,
  HOBEIAN ZG-222Z) und `wasser_leck_waschmaschine` (`binary_sensor.tz3000_upgcbody_snzb_05`), dazu
  `wasser_entwarnung` für beide. Verhalten: sofort bei Nässe (5 s entprellt), danach **alle 5
  Minuten erneut, solange nass** (max. 24 Runden = 2 h), und eine Entwarnung beim Trockenwerden —
  letztere nur nach echter Nässe (`trigger.from_state == 'on'`), nicht beim HA-Neustart.
  Beleg: Testauslösung 17:58:34 UTC (Heizung) und 18:01:54 UTC (Waschmaschine) — beide Male
  wanderte der Zeitstempel der Gruppen-Notify-Entität eine Sekunde später mit.
  **Pitfall:** `automation.trigger` **wartet** auf das Ende der Automation — eine mehrstündige
  Automation läuft damit in den Timeout der HTTP-Anfrage. Erfolg deshalb über `last_triggered`
  und den Zeitstempel der Notify-Entität prüfen, nicht am Rückgabewert der Auslösung.
  **Werkzeuge** (lokal, gitignored): `local-tools/wasser_alarm_bauen.py` legt die drei Automationen
  an bzw. ändert sie (idempotent), `local-tools/wasser_alarm_zeigen.py` rendert die gespeicherten
  Texte zur Kontrolle. Beide lesen den Token aus `~/.hermes/.env` und enthalten keine Geheimnisse.
* **Aqara-Inventur (19 Geräte)** — 9 `lumi.weather` (Klima) und 10 `lumi.sensor_magnet.aq2`
  (Tür/Fenster); 18 leben, gemeldet innerhalb 8–47 min, Zähler gesund. **Der einzige Tote bleibt
  `Eingangstür`** (121 d, Zelle bestellt). Zwei Befunde mit Handlungsbedarf:
  * **`ToiletteTemp` und `TempSensorBad` haben keine Batterie-Entität** — ZHA kennt bei ihnen
    Hersteller/Modell nicht (ihre übrigen Entitäten heißen `sensor.unk_manufacturer_unk_model_*`),
    deshalb wurde nie eine Batterie angelegt. **Der Batterie-Alarm kann sie nicht sehen**, ihre
    Zellen könnten unbemerkt sterben. **Korrektur (25.09.):** der zuerst empfohlene Weg „in ZHA
    erneut interviewen" existiert **nicht** — die ZHA-Websocket-API kennt nur
    `zha/devices/reconfigure`, und 16 Aufrufe über fünf Minuten bei nachweislich wachem Gerät
    (deren `last_seen` und Werte während der Aktion vorrückten) haben den Hersteller nicht geändert.
    Hersteller und Modell kommen aus dem Node Descriptor, den ZHA **beim Koppeln** liest. Wirklicher
    Weg: **entfernen + neu koppeln** — belegt am eigenen Haus, denn `Fenster Sensor Toilette` trägt
    dieselben Leichen und daneben richtige Entitäten. Preis: neue Entitäts-Suffixe und Leichen;
    vorher prüfen, was darauf verweist. **Noch offen.**
  * **Referenzprüfung vor dem Neukoppeln (25.09.): null Treffer.** Scan über alle Automationen,
    Szenen, Zustandsattribute und **alle 12 Lovelace-Boards** (Standard, Karte, Lampe, Mein Zuhause,
    Treppe-Alarm, Meine Energie, Garage SDM 360, Fenster/Türen, Klima, Heizung, CO2, Strom) — keine
    einzige der 12 Entitäten (und keine der beiden device_ids) wird irgendwo referenziert. Das
    Neukoppeln kann also nichts brechen. **YAML-Konfiguration ist per API nicht lesbar** (Template-
    Sensoren, Skripte, recorder-Ausschlüsse) — dort ist nicht geprüft.
  * **Alte IDs sind festgehalten**, damit nach dem Neukoppeln zugeordnet werden kann:
    `local-tools/ids_vor_neukoppeln.json` (gitignored). Suchende Begriffe: IEEE
    `00:15:8d:00:8b:ba:8c:27` (ToiletteTemp, device_id `a844a6db54c28aff22aa2a0677e72e21`) und
    `00:15:8d:00:8b:bd:8d:86` (TempSensorBad, device_id `c88b0d9a758cd7121af2fde02a3b5c8b`);
    Entitäten `sensor.toilettetemp_{temperatur,luftfeuchtigkeit,druck}`,
    `sensor.tempsensorbad_{temperatur,luftfeuchtigkeit,druck}`, beider `…_identifizieren` plus die
    `unk_manufacturer_unk_model_{rssi,lqi}`-Leichen. **Die alten Entitäten werden NICHT gelöscht** —
    so bleiben Historie und Langzeitstatistik der beiden Sensoren erhalten.
  * **Neukoppeln beider Sensoren erfolgreich (25.09. abends).** Beide haben jetzt Hersteller `LUMI`
    und je eine **Batterie-Entität**: `sensor.toilette_toilettetemp_batterie` = **55,5 %** (numerisch,
    damit zählt der Batterie-Alarm sie mit) und `sensor.bad_tempsensorbad_batterie` noch `unknown`
    (füllt sich beim nächsten Bericht). **Die Mess-Entitäten behielten ihre IDs**
    (`sensor.{toilettetemp,tempsensorbad}_{temperatur,luftfeuchtigkeit,druck}`) — der Recorder hängt
    an der ID, also laufen **Historie und Langzeitstatistik ohne Bruch weiter**; nur die neuen
    Batterie-Entitäten sind historienlos. Die Referenzprüfung vorher (null Treffer) hat sich damit
    bestätigt: es war nichts zu reparieren. Vorher-Nachher-Zuordnung:
    `local-tools/ids_vor_neukoppeln.json`.
    Offen und rein kosmetisch: sieben `unk_manufacturer…`-Leichen, die krummen Batterie-IDs, und die
    beiden Sensoren fehlen auf den handgepflegten Boards `Klima`/`Heizung`.
  * **Diese drei Kosmetikpunkte sind noch am selben Abend erledigt:** Batterie-Entitäten umbenannt zu
    `sensor.toilettetemp_batterie` (55,5 %) und `sensor.tempsensorbad_batterie` (noch `unknown`);
    alle sieben `unk_manufacturer…`-Leichen `hidden_by: user` (die eine noch aktive zusätzlich
    deaktiviert); auf `dashboard-klima` und `dashboard-heizung` je eine Karte **„Bad & Toilette"**
    angehängt (Klima 2→3, Heizung 5→6 Karten, übriger Inhalt byteweise unverändert). **Beinahe-Fehler
    dabei:** die neue Karte wurde vor dem Umbenennen gebaut und zeigte auf die alten Batterie-IDs —
    aufgefallen beim Referenzcheck gegen `/api/states` (2 unbekannte Entitäten je Board), korrigiert,
    danach 0 unbekannte Referenzen. Das Aussehen selbst ist nicht verifiziert (kein HA-Login).
* **Fenster/Türen-Board erweitert (25.09.):** die Ansicht `fenster` des Boards `fenster-turen`
  hat unten einen Bereich **„Batterien"** — Überschriftskarte mit `mdi:battery-40` plus die Liste
  aller zehn Einheiten-Batterien, in **derselben Raumreihenfolge** wie die Öffnungsliste darüber.
  Stil nach `treppe-alarm/0` (dort: heading „Batterien" + schlichte Entitätsliste). Alle zehn
  Einheiten haben eine Batterie-Entität, `sensor.door_batterie` (Eingangstür) steht wie erwartet
  auf `unavailable`, bis die CR1632 da ist. 20 Referenzen geprüft, **keine** unbekannt, übriger
  Ansichtsinhalt byteweise unverändert. Nächste Zellen laut Stand: Keller Partyraum 66 %,
  Toilette 69,5 %, Balkontür 73 %. **Ungeprüft:** ob eine Überschriftskarte außerhalb von
  Sections rendert — die Ansicht nutzt keine Sections; falls sie nicht erscheint, ersetzt ein
  Kartentitel die Überschrift.
* **Heizungs-Board: die Gauge „Delta Vorlauf–Rücklauf" verpackt (26.09.).** Die Kachel meldete
  „Entität ist nicht-numerisch", weil `sensor.heizung_differenz_vor_rucklauf` `unknown` liefert: es
  ist ein Template-Helper (Config-Entry „Heizung: Differenz Vor- Rücklauf", domain `template`,
  state `loaded`) über die beiden Flow-Monitor-Sensoren, und der ESP `esp32_c3_web_a8dfa8` ist
  **absichtlich aus** (keine Heizsaison). Der Sensor ist also nicht defekt, sondern ehrlich — ein
  0-Wert wäre erfunden. **Fix:** die Gauge steckt jetzt in einer `conditional`-Karte mit zwei
  Bedingungen (`state_not: unknown`, `state_not: unavailable`) und erscheint von selbst wieder,
  wenn geheizt wird. **Beleg:** die ESPHome-Integration ist gesund (der zweite ESP
  `esp_wroom_32_keller` liefert 3/3 Werte), alle 18 Referenzen der Ansicht gültig, 6 Karten vorher
  wie nachher. Ungeprüft: das Rendern (kein HA-Login). Die Karte „Heizung Übersicht" zeigt bewusst
  weiter „nicht verfügbar" — sie sagt, warum die Differenz fehlt.
* **ESP-Test am 26.09. bestanden — und die Dach-CO₂-Baustelle ist damit zu.** Der Nutzer schaltete
  Flow-Monitor **und** Dach-CO₂-ESP ein. Beleg (aus dem eigenen Mithörer, Sekunden genau):
  `wifi_status` des Flow-Monitors `OFFLINE → ONLINE` um **07:50:25**, erste Werte **07:49:28** —
  Vorlauf 23,06 °C, Rücklauf 22,19 °C, **Differenz 0,9 °C**. Die Vorhersage „nahe 0" traf genau zu:
  das Wasser steht weitgehend, weil nicht geheizt wird. **Korrektur des Nutzers (26.09.):** der
  Flow-Monitor hängt an der **Gasheizung**, die mit der Daikin-Wärmepumpe `dach_ap22393`
  **nichts zu tun hat**. Deren 0 W Kompressorleistung ist also **keine** Bestätigung für den
  Heizkreis, sondern eine getrennte Anlage — der Assistent hatte beide zusammengezogen (beide
  tragen „Dach" im Namen) und das als Bestätigung verkauft; **zurückgenommen**. Die Messwerte
  selbst und der Fix bleiben davon unberührt. Damit ist die Bedingung der verpackten
  Kachel erfüllt — **das Rendern hat der Nutzer zu prüfen**, das ist der einzige offene Rest.
  Der **Dach-CO₂-Sensor**, seit 19.09. `unavailable`, war **kein Defekt**, sondern das
  ausgeschaltete Gerät: jetzt 751 ppm / 44 % / 21,7 °C. Alle vier ESPHome-Geräte (Flow, WZ-CO₂,
  Dach-CO₂, Keller-CO₂) sind damit gesund. **Damit ist der Punkt „Dach-CO₂ unavailable" erledigt.**
  * **Korrektur zu einer früheren Behauptung:** das Dachstudio ist **nicht** die funkschwächste Ecke.
    Es steht dort ein eigener Router (`Steckdose Mascha`, LQI 140), das Haus hat **26 Router** gegen
    36 Endgeräte, und die LQI-Werte schwanken stark (Maschas Button 172 → 80 innerhalb einer Stunde,
    `FensterSensorAQ` im selben Raum 164). Die zunächst gemeldeten 60–68 waren Momentaufnahmen.
    **Warum Maschas Taster seine Netzanmeldung verlor, ist mit den vorliegenden Daten nicht
    entschieden** (Route oder Koordinator-Tabelle) — nicht als geklärt darstellen.
  * **Button → Steckdose läuft über HA**, nicht als Gerätebindung: die Automation „Button => Mascha PC"
    (`automation.button_mascha_pc`, id `1758434574653`) hört auf `zha_event`, prüft
    `device_ieee == a4:c1:38:d6:46:c0:09:e7` und `command == "toggle"` und schaltet
    `switch.steckdose_mascha`. Beleg für den Erfolg des Neuanlernens: Auslösung 19:31:16 Uhr, sechs
    Minuten nach dem Anlernen. **Folge:** solange der Taster nicht eingebucht ist, ist die Steckdose
    nicht schaltbar, auch wenn seine LED leuchtet. Der Trigger ist ungefiltert (`zha_event` für alle
    Geräte) — er funktioniert, aber ein `event_data`-Filter auf die IEEE wäre sauberer und würde bei
    `mode: single` auch das Verwerfen eines Drucks während eines anderen Ereignisses vermeiden.
  Nächste Zellen: SZTemp 48 %, WohnzimmerTemp 55,5 %, TempSensorTreppe 59 %.
* **Drei Taster-Automationen auf gefilterten Trigger umgestellt** (25.09.): `Button => Steckdose mein
  PC` (id 1758393492176), `Button => Mascha PC` (1758434574653) und `Button => Steckdose Altar`
  (1758434943261) hörten auf **jedes** `zha_event` im Haus und filterten erst in der Bedingung; jetzt
  steht `event_data: {device_ieee: …}` direkt im Trigger. Geändert wurde **nur** der Trigger —
  Bedingung, Aktion und Mode sind nach dem Schreiben byteweise verglichen und identisch, alle drei
  weiter `on`. Der Filter ist derselbe Wert, den die Bedingung ohnehin prüft (die nachweislich
  funktioniert, siehe 19:31-Auslösung). **End-to-end belegt am selben Abend** (Tastendruck 20:40):
  Mascha 20:40:04 → `switch.steckdose_mascha` aus; Altar 20:40:44 → `switch.tz3000_gjnozsaz_ts011f`
  („Steckdose Wohnzimmer Altar") um 20:40:48 an. **Wichtig dabei:** die `switch`-Entität des Tasters
  selbst ändert sich beim Drücken **nicht** (beide blieben auf ihrem alten Zeitstempel) — sie ist der
  On/Off-Cluster-Zustand des Geräts, kein Druckzähler. Empfangsnachweis ist `zha_event`, nicht diese
  Entität; die frühere Notiz im Skill `home-assistant-state-forensics` war in dem Punkt falsch und
  wurde korrigiert.
* **Zwei tote Automationen gefunden:** `automation.goe_nachtladen_start_2` und
  `automation.goe_nachtladen_stop` stehen auf `unavailable`, zu beiden existiert **keine**
  Konfiguration mehr (REST-Config-View: 404). Karteileichen aus der go-e-/Nachtladen-Zeit; sie können
  nichts mehr auslösen und kollidieren daher nicht mit dem eigenen Lade-Regler. Aufräumen offen.
* **„Fenster Bad"** (`lumi.sensor_magnet.aq2`, bisher nur Werksname): Gerätename über das Register
  gesetzt. **Entitäts-IDs absichtlich NICHT umbenannt** — die stehen im Lovelace-Dashboard
  `fenster-turen`, ein Rename hätte die Kachel zerlegt. Vor jedem ID-Rename erst referenzieren
  (Automationen + alle Storage-Boards), das hat hier genau den Fehler verhindert. Aufnahme des
  Sensors belegt: er meldete ein 2 Sekunden kurzes Auf/Zu als beide Flanken.
* **ZHA ist die Zigbee-Anbindung, nicht Zigbee2MQTT** (Config-Entry „Sonoff Zigbee 3.0 USB Dongle
  Plus"); `zigbee2mqtt/#` am Broker ist leer. Zu `zha_event`: ein **eingebuchter Taster** erzeugt
  beim Drücken eines (belegt: `attribute_updated on_off` von Maschas IEEE Sekunden nach dem
  Anlernen, und **kein** Ereignis während 5–10 Drücken davor — der saubere Vorher/Nachher-Beweis).
  Ein schlafender Präsenz-/Battersensor erzeugt dagegen keines. Ein leerer Ereignisstrom beweist
  also nichts über Sensoren; maßgeblich bleibt die Zustandsänderung der Entität.

## Zuletzt behoben (23.09.2026)

* **HAs Standort stand noch auf der Werkseinstellung Amsterdam** (52,3731/4,8903, Höhe 0 m) —
  jede sonnenbasierte HA-Automatik und später der Rückfall unserer Prognose-Regel hätte damit
  für die falsche Stadt gerechnet. Gesetzt über `homeassistant.set_location` auf
  **49,1278/8,4076, 105 m** (PLZ-Mittelpunkt Linkenheim; die Höhe aus Open-Meteo, demselben
  Höhenmodell wie die Prognose). Zeitzone war bereits `Europe/Berlin`, Land `DE`.
  **Belegt dreifach:** `/api/config`, `zone.home` und als unabhängiger Zeuge der
  Sonnenuntergang — `sun.sun` springt von 17:36 UTC (19:36 Berlin) auf **17:24 UTC
  (19:24 Berlin)**, 12 Minuten früher, genau der Sprung von 52,37° auf 49,13° Nord.
  Anschließend auf den **genauen Punkt der Anlage** nachgezogen (der Besitzer hat ihn
  geschickt): dazu die Höhe erneut aus dem Höhenmodell geholt (111 m) und den Sonnenuntergang
  als Plausibilitätsprobe genommen (820 m Verschiebung ⇒ wenige Sekunden, gemessen 2 s).
  **Datenschutz-Regel:** die genauen Koordinaten stehen **nur** in Home Assistant und in der
  lokalen `config.json` (per `.gitignore` ausgeschlossen, mit `git check-ignore` geprüft) —
  sie gehören **nicht** in dieses Repo oder in die Doku. Öffentlich ist hier nur die PLZ-Ebene.
  Auch die App rechnet jetzt mit demselben Punkt (frisch abgerufen: 28,86 kWh, unabhängig
  nachgerechnet 28,86 kWh).
* **Der Wechselrichter ist nachweislich lesend** — siehe die Regel oben: die ungenutzten
  Batterie-Schreibfunktionen und die Modbus-Schreibprimitive sind raus, strukturell gepinnt,
  und der Proxy zählt weiter `upstream_writes: 0`.
* **PV-Prognose Schritt 1** gebaut und live (siehe eigener Abschnitt oben).

## Zuletzt behoben (22.09.2026)

* **Der Deye-Poller schweigt jetzt nachts und wacht am Zähler auf** (anderes Projekt:
  `HA-POWER-DASHBOARD/deye-pv-rs`). Vorher: ein Fehlversuch alle 33 s, jeder mit Logzeile
  **und** einem `offline` nach HA — rund 2600 Zeilen und 2600 Nachrichten je Nacht für ein
  Gerät, das erwartungsgemäß schläft (das SolarMAN-Logger-Modul hängt am Wechselrichter;
  Beweis: es war um **05:16 UTC** von selbst wieder da, Port 8899 offen). Jetzt: Verdopplung
  vom Intervall bis `--backoff-max` (900 s), Logzeile nur beim ersten Fehler einer Serie und
  dann jedem achten Schritt, `offline` nur beim Zustandswechsel (einmal je Ausfall), und die
  Erholungszeile nennt die Zahl der Versuche. **Die Idee des Besitzers ist der Weckruf:**
  während des Backoffs liest der Poller `sensor.sdm630_total_kwh` (wächst in beide
  Richtungen, bewegt sich also genau dann, wenn im Garagenstrang Energie fließt — einspeisen
  oder ins Auto) und pollt sofort wieder, wenn der Zähler sich bewegt; ein vorzeitiger
  Versuch je Backoff-Periode, damit ein tagsüber defekter Logger nicht gehämmert wird.
  Gepinnt durch 6 neue Unit-Tests (Schedule, Drosselung, einmal-je-Ausfall, Weck-Gating),
  33 im Binary + 13 Konformitäts-Tests grün.
* **Dritter Session-Wert: SDM + Garage-PV** (`ha-app/evcharge/session_meter.py`, Wunsch des
  Besitzers; Regeln oben). Zwei Fallen dabei geschlossen, jede mit Test: ein **0,00** des
  Wechselrichter-Zählers als *Basis* hätte die nächste echte Zahl in eine ~279-kWh-Korrektur
  verwandelt, und eine Session, die beginnt, während der Logger schläft, holt die Basis jetzt
  nach (Korrektur dann „teilweise"). `test_session_meter` **68 Prüfungen**, 12 Suiten grün.
* **Messungen am Garagenstrang** (22.09.2026 — wichtig beim Lesen aller Zahlen):
  * Drei Nächte, je ~11,5 h: SDM **Import und Export exakt 0,000 kWh**. Der Strang ist also
    nachts nicht „lastfrei", sondern **unter der Zählschwelle** des Geräts (Datenblatt:
    Startstrom 0,4 % von Ib = **0,04 A**, spezifiziert erst ab 5 % Ib = 0,5 A; gemessen:
    0,41 A / 97 VA / **−97 var** / PF −0,20 auf L2, Zähler stehen trotzdem). Router, Tor und
    go-e-Standby werden also nicht mitgezählt — die Session-Zahl ist davon sauber.
  * Die Garage hängt praktisch **einphasig auf L2** (L1/L3 messen 0,00 A); dort Garage-PV,
    go-e, Router, Tor.
  * Der **Wechselrichter-Zähler** liegt gegen den SDM-Export um **höchstens 8 %** zu hoch
    (4,18 gegen 3,88 kWh in 12 h) — und diese Lücke ist **kein Beweis für einen Fehler des
    Wechselrichters**: 0,30 kWh in 12 h sind genau **25 W Dauerlast am Strang**, und die gibt
    es dort (Router, Tor, go-e-Standby) — sie sind nur für den Zähler unsichtbar, weil er sie
    nicht **zählen** kann. Der echte Fehler liegt daher zwischen ~0 % (bei ~25 W Dauerlast)
    und +8 %; ein Zwischenstecker vor dem Router würde es klären. Nach dem Aufwachen liest
    das Register kurz 0,00.
  * **Der Zähler zählt in 0,1 kWh, nicht in 0,01** (korrigiert 22.09.2026): unser Poller las
    ihn **10× zu klein** (279,56 kWh statt 2795,6). Entschieden **ohne** die App, über die
    Selbstkonsistenz von Zähler und Leistung: im Fenster 21.09. 05:00–17:00Z lief das Register
    **42 Schritte** weiter, während die protokollierte AC-Leistung **4,18 kWh** ergab → 0,0995
    kWh je Schritt. Die Deye-App bestätigt es von der anderen Seite (2,79 MWh nach 739
    Betriebstagen ≈ 3,8 kWh/Tag, passend zum gemessenen Tagesertrag). Folge: auch der **dritte
    Session-Wert** wäre um Faktor 10 zu klein gewesen. Der Poller veröffentlicht außerdem
    keinen rückwärts laufenden Zähler mehr — HA liest ein Absinken bei `total_increasing` als
    Zählerreset und **addiert** den neuen Wert, hätte also morgens den ganzen Stand als
    Erzeugung gebucht.
  * Offen: Das Register ist **16 Bit** und läuft bei **6553,5 kWh** über (~2,7 Jahre bei
    diesem Ertrag) — dann friert der Rückwärts-Schutz den Wert ein, vorher muss das hohe Wort
    geprüft werden.

## Zuletzt behoben (21.09.2026)

* **Die App hat `alw=0` geschickt, obwohl genug Sonne da war** (`controller.py: _finalize`).
  Gemessen am 20.09.: `11:28:36 alw=0` bei **2358 W** Überschuss, `11:30:37 alw=0` bei
  **1901 W** — der Besitzer hat es aus der App heraus gesehen (2,85 kW Produktion bei
  2,66 kW Verbrauch, 99 % solar+battery). Ursache: die **Anlaufverzögerung hing an
  `charger.charging`** (ob das Auto zieht), die Schreibentscheidung aber an
  **`charger.enabled`** (ob die Wallbox freigegeben ist). Bei einem angesteckten, aber nicht
  ziehenden Auto (voll / Abfahrtszeit) war `charge and not charging` in **jedem** Zyklus wahr
  → die 60-s-Gnade wurde jeden Zyklus neu aufgezogen → die Entscheidung wurde auf
  `charge=False` gezwungen → der Schreibpfad (der gegen `enabled` vergleicht) schaltete die
  Wallbox ab. **Ein Stopp pro Minute bei reichlich Überschuss**, fünf davon im
  Sicherungsfenster — genau das löste die Störung aus. Beide Verzögerungen hängen jetzt an
  derselben Größe wie der Schreibpfad (`enabled`); eine wartende Gnade schreibt **nichts**.
  Gepinnt durch „a plugged car that is not drawing must not be stopped every cycle"
  (10 Zyklen, 0 `alw`), „the start grace waits on the wallbox and writes nothing while it
  waits" und „the stop grace holds first and writes exactly one stop afterwards".
* **Hysterese 300 W an der Untergrenze** (`enable_threshold_w` / `disable_threshold_w`,
  `controller.py: _floor_w`). `disable_threshold_w` war bis dahin **deklariert und nie
  gelesen** — eine Einstellung, die nichts tat. Jetzt: Start ab Untergrenze **+300 W**,
  Halten bis Untergrenze **−300 W**; die **Starthysterese** hat der Besitzer am selben Tag
  auf **100 W** gesenkt (Config + Neustart), die Haltegrenze blieb bei 300 W. Live sichtbar
  in der Begründung: „below minimum (**1080 W**, 1p)" solange die Wallbox freigibt,
  „(**1480 W**, 1p)" wenn sie aus ist.
* **Sicherung zählt nur noch Stopps** (`safety.py`) — Regel oben, `amx` zählte nie.
* **Test-Attrappe korrigiert** (`tests/test_controller.py`): `car()` setzt `enabled` passend
  zu `charging`. Vorher beschrieb sie Zustände, die die Hardware nicht hergibt (Strom fließt,
  ohne dass die Wallbox freigibt) — und verdeckte damit genau diesen Fehler.
* Live-Beleg nach dem Neustart am 21.09. 06:40: ein `amx=6` (Nachregeln beim Halten), dann
  **genau ein** `alw=0` nach Ablauf der 180-s-Gnade, danach Ruhe; Zähler 0/5, keine Störung.
  Stand: **12 Suiten grün** (`test_controller` 94, `test_safety` 33, `test_session_meter`
  68 Prüfungen).

## Zuletzt behoben (19.09.2026)

* **Billigfenster stoppte laufende Ladungen** (`controller.py`). Vorher lautete der Zweig
  `if mode == cheap_hours and cheap_now and not charger.charging`: sobald das Auto wirklich
  Strom zog, fiel es in die Überschusslogik, wurde auf 6 A zurückgenommen und nach der
  180-s-Gnade abgeschaltet — worauf das Fenster es 60 s später neu startete. **Signatur der
  Nacht 18./19.09. (00:00–03:25 lokal): 40× `alw=0` und 41× `alw=1` im Log, Begründung
  pendelte zwischen `cheap tariff window` und `surplus -1117 W below minimum (4140 W, 3p)`;
  je Runde 31 s bei 14 A, 181 s bei 6 A, 88 s aus.** Jetzt hält der Zweig jede laufende
  Ladung bis zum Fensterende (Kommentar im Code erklärt es, die drei Checks aus der Regel
  oben pinnen es). Live-Beleg für „der Fix läuft": der Prozessstart muss **jünger** sein als
  `controller.py` — `ps -o lstart= -p $(systemctl --user show evcharge-wt.service -p MainPID --value)`.
* **Absturzschleife der Lade-App** (11:13–15:19 lokal blind, **1182 Neustarts à ~12 s**,
  Port 7080 tot). `proxy.py:summary()` normalisierte `now` nicht, während `main.py` es als
  `proxy_summary(pstats, failures=…)` **ohne** Uhr aufruft. Die Zeile läuft nur, wenn der
  Proxy einen Fehler-Zeitstempel meldet — der neue Proxy-Build (09:59) liefert
  `last_upstream_error_at`, der erste Upstream-Fehler um 11:13 machte daraus `float(None)`
  in jedem Zyklus. Gefixt durch Normalisieren in `summary()`; abgedeckt durch
  „the production call shape: summary() without an explicit clock" in `test_proxy_card.py`
  **plus** einen Live-Check gegen `/status`. Beide Zustände wurden belegt: Fix raus → Test
  bricht ab, Fix rein → grün. **Lehre für die Zukunft: nach jedem Rebuild des Proxys den
  Feldsatz von `/status` prüfen und die App-Tests laufen lassen** — ein neues Feld ist ein
  neuer Codepfad, und auch ein reiner Anzeigepfad reißt die Steuerung mit.

## Offene Punkte

2. Das HA-Lovelace-Dashboard (`/strom-verbrauch`) wurde **nie** im HA selbst visuell
   geprüft (Login-Wand); Ersatz ist `docs/preview.html`. Das ist die größte offene
   Unsicherheit im Dashboard-Teil.
3. Der **Rust-Port der Lade-App ist nicht begonnen** — sie ist der nächste Kandidat,
   aber erst, wenn ihre Logik stillsteht (Plan bewusst unverändert gelassen).
4. **SolarEdge meldet oberhalb ~4600 W einphasig zu hoch (Spitze 5533 W)** — Verdacht des
   Besitzers: das war eine falsch erkannte Phasenzahl, nicht der Wechselrichter. Die
   Phasenroutine ist seitdem strenger (erst nach ~20 s fließendem Strom geglaubt, dann der
   höchste Wert bis zum Abstecken; `phases_checked=false`, solange kein Strom fließt) →
   **beobachten**, ob der Wert wiederkommt.
5. **Das Halten im Billigfenster ist nur durch Tests belegt, nie nachts am echten Auto
   gesehen.** Beim nächsten Einsatz von `cheap_hours` (Winter; der Modus steht derzeit auf
   `pv`, dort ist das Fenster wirkungslos) zu erwarten: **ein** Start beim Öffnen, danach
   **0 Stopps** bis zum Fensterende, Ladung durchgehend auf `max_current` — der Stopp-Zähler
   im UI muss bei 0 bleiben. Treten wieder ~12 Stopps pro Stunde auf, ist die alte Bedingung
   zurückgekommen (Signatur in „Zuletzt behoben") und es ist Code, nicht Hardware; die
   Sicherung rastet bei 5 Stopps selbst ein und schreibt dann nichts mehr.
6. **MQTT ist an — erledigt am 25.09.2026.** Der Besitzer hat die Zugangsdaten aus dem
   Mosquitto-Add-on selbst eingetragen (`set_mqtt_login.py`: fragt mit `getpass` verdeckt ab,
   schreibt direkt in `ha-app/config.json`, Rechte 600, Sicherung als `.bak`; die Datei ist
   gitignored). Belegt: direkter CONNACK-Test mit genau dem App-Client → **Code 0 = angenommen**,
   `mqtt_connected: true`, und in HA **18 Entitäten unter „EV Charger WT", keine ohne Wert**.
   Zur Vorgeschichte: anonym nimmt der Broker nichts an (CONNACK 5) und der App-Client ist
   nachweislich korrekt (MQTT-3.1.1-CONNECT geprüft) — der erste Datenversuch des Besitzers wurde
   mit **Code 5 = nicht autorisiert** abgelehnt, diese Kombination kannte der Broker also nicht.
   **Zwei Fehler kamen dabei ans Licht — beide in Code, der nie zuvor gelaufen war:**
   * Die Discovery-Templates für `binary_sensor` „EV charging" und `switch` „control enabled"
     gaben Jinja-Booleans aus (`False`) — HA erkennt darin weder `ON/OFF` noch `false`, beide
     Entitäten blieben dauerhaft `unknown`. Jetzt ausgeschrieben, in
     `tests/test_mqtt_loopback.py` am Quelltext festgenagelt.
   * **Acht verwaiste retained Discovery-Nachrichten** lagen im Broker: eine ältere Fassung hatte
     `mode`, `max_current`, `min_current`, `buffer_soc`, `priority_soc`, `plan_energy_kwh` und
     `decision` als *Sensoren* publiziert (später wurden es *numbers*) plus einen `binary_sensor`
     statt des `switch`. Sie erzeugen in HA Entitäten ohne Wert. Gelöscht mit
     `mqtt_discovery_audit.py` (leere Payload, `retain=True`) — HA-Entitäten: **26 → 18**.
     *Merksatz:* eine retained Discovery-Nachricht überlebt jede Code-Änderung (dieselbe Mechanik
     wie beim evcc-Rest in Punkt 7, nur im eigenen Gerät). Nach jeder Änderung an
     `publish_discovery()` lohnt der Audit-Lauf.
7. **Der evcc-Rest in HA bleibt liegen — der Besitzer räumt ihn selbst auf, „irgendwann mal"**
   (Entscheidung 23.09.2026, ausdrücklich: *nicht* anfassen, nicht nochmal anbieten).
   Zum Nachschlagen, was dort liegt: der Integrationseintrag **`evcc_intg` steht auf
   `setup_retry`** (HA klopft weiter an einen toten Server — der einzige Rest, der noch
   arbeitet), dazu **97 `evcc_*`-Entities, davon 96 `unavailable`/`restored`** (59 davon sind
   die go-e-Entities aus evccs MQTT-Discovery). **Es gibt keine evcc-Automation mehr** — die
   frühere Notiz „Automation EVCC PV Laden ab 8 Uhr an" war veraltet (0 evcc-Automationen,
   geprüft am 23.09.). Falls er es später doch delegiert: der Weg wäre
   `DELETE /api/config/config_entries/entry/<entry_id>` (existiert nachweislich — mit einer
   erfundenen ID geprüft, sauberes 404 „Invalid entry specified" statt 405); die 59
   MQTT-Entities gingen damit **nicht** weg, die hängen als retained Discovery-Nachrichten im
   Broker und brauchen geleerte Topics (`mqtt.publish` mit leerer Payload und `retain` — über
   HA selbst möglich, ohne Broker-Login).
8. **Dashboard-Zeilen umstellen** (nach Punkt 6): die toten `sensor.evcc_*`-Zeilen im
   HA-POWER-DASHBOARD auf die dann vorhandenen Entities der eigenen App zeigen lassen.
9. **Auch der Deye-Poller soll später auf MQTT umgestellt werden** (Wunsch des Besitzers,
   20.09.2026). Heute publiziert er über HA selbst (`POST /api/services/mqtt/publish`,
   Token aus `~/.hermes/.env`, Routinen in `powerdash/deye_pv.py` und `deye-pv-rs/src/ha.rs`)
   — das braucht keinen Broker-Login, kann aber nur senden. Umstellung, wenn der Broker-Login
   zugänglich ist (Punkt 6), damit im Haus **ein** Muster für alle Veröffentlicher gilt.
   Der dafür vorbereitete, wieder verworfene Weg (HA-REST-Publish, read-only, kein
   Broker-Login) liegt geparkt in `ha-app/local-tools/mqtt-via-ha-rest/` — er wurde nicht
   genommen, weil damit die **Steuerung aus HA heraus verloren geht** (die App kann über
   diesen Weg nur senden, nicht empfangen; Modus, Stromgrenzen und SOC-Schwellen leben in
   der eigenen Web-UI und über `set/#`-Topics).

11. ~~SDM630-Polling~~ **geklärt (22.09.2026): der SDM630 wird einwandfrei gepollt.** Die
    alten Zeitstempel sind korrekt — ein Zähler, der sich nicht ändert, wird von HA **nicht**
    neu geschrieben. Beweis: die Spannungssensoren (`sdm630_l1/l2/l3_spannung`) und die
    Frequenz ändern sich **alle ~15 s** (237,83 -> 237,41 V in 75 s). **Lehre für die
    Frische-Prüfung: ein bewegter Wert muss es sein** — Spannung ja, Leistung/Strom **nein**
    (nachts 0,00 W / 0,0 A, wird nie neu geschrieben, sieht wie ein toter Zähler aus).
    `sdm.entity_live` steht deshalb auf `sensor.sdm630_l1_spannung`.
    **Ebenfalls normal:** die **Garage-PV (Deye, 192.168.178.33:8899) ist nachts nicht
    erreichbar** — ein Mikro-Wechselrichter wird von der Sonne versorgt. Letzte erfolgreiche
    Zeile 21.09. **17:38 UTC**, Sonnenuntergang Berlin war 19:40 MESZ = 17:40 UTC; sie kommt
    nach Sonnenaufgang von selbst zurück. **Kleiner offener Punkt:** der Poller schreibt dann
    jede Nacht alle 33 s `ERROR ... Host is unreachable` (~2600 Zeilen) — auf eine Zeile je
    Stunde drosseln oder zwischen Dämmerung und Sonnenaufgang schweigen.

## Bekannte Messanomalien der Umgebung

Nicht unsere Baustelle, aber beim Lesen von Zahlen bedenken: `sensor.garage_pv_energie`
fällt 8× aus; `sensor.evcc_battery_power` ist `unavailable` (Rest der stillgelegten Steuerung, in HA `restored`); der Deye-Wert ist ~4 min
alt; `ElektroHeizungKeller` + Sensoren `unavailable`; der Rust-Deye-Poller kann kein
https zur HA-Verbindung.

## Wie man prüft (erprobte Befehle)

```sh
cd /home/adermake/EV-CHARGER-WT-HA/ha-app
for t in test_safety test_controller test_phase_probe test_service_smoke test_proxy_card \
         test_goe_driver test_ha_read test_site_cadence test_cheap_hours; do
  python3 tests/$t.py; done
python3 /tmp/health.py                      # Live-Lage in ~10 Zeilen
curl -s 127.0.0.1:7080/api/state            # dieselbe Lage als JSON
curl -s 127.0.0.1:1504/status               # Proxy-Statistik
cd ../modbus-proxy-rs && make check         # cargo + Konformität + Kreuzvergleich + Differential + Produktivconfig
```

Wichtig beim Installieren des Proxys: `make static install` scheitert mit "Text file
busy", solange er läuft → **stoppen, installieren, starten**. Und: installierte Binärdatei
gegen den Build prüfen (`sha256sum bin/muxproxy`), nicht annehmen.

## Rollback auf den direkten Weg (Consumer ohne Proxy)

```sh
systemctl --user disable --now muxproxy-rs.service
```

Danach beim Consumer (Laderegler bzw. wer auch immer die Meter liest) die Meter-Konfiguration
zurück auf `192.168.178.84:1502` zeigen lassen und ihn neu starten — **Consumer vorher
stoppen und alle Meter in einem Schritt umstellen**, sonst validiert er eine halb geänderte
Konfiguration gegen das Gerät und speichert sie nicht.

Das frühere Werkzeug dafür (`set_evcc_meter_host.py`, schrieb direkt in die SQLite des alten
Controllers) ist aus dem Proxy-Repo entfernt, weil es ausschließlich ihn bediente; es liegt lokal unter
`modbus-proxy-rs/local-tools/` und wird nicht versioniert.

## Repositories (öffentlich, MIT)

Vier Repos, alle **an Ort und Stelle** initialisiert (`git init` in den bestehenden
Verzeichnissen), damit die laufenden Units ihre Pfade behalten. Branch `main`, Identität nur
lokal pro Repo (`trwa <me@home>` — verknüpft die Commits *nicht* mit dem GitHub-Konto),
Remote vorbereitet. **Alle vier sind seit 20.09.2026 öffentlich auf GitHub** (Account
`machtnichts`, MIT, Copyright `nixda`):

| Repo | Pfad | Commit | Dateien |
|---|---|---|---|
| `modbus-proxy-rs` | `modbus-proxy-rs/` | `778db32` | 30 |
| `evcharge` | `ha-app/` | `a381fdb` | 35 |
| `ha-power-dashboard` | `~/HA-POWER-DASHBOARD` | `e3884ee` | 72 |
| `ev-charger-wt-ha` | dieses Verzeichnis | `ff71f9f` | 26 |

Verifiziert per frischem Klon von GitHub (Dateibestand, Lizenz, keine unerwünschten Dateien);
`evcharge` zusätzlich mit komplettem Testlauf aus dem Klon (76 Checks grün). Rust-Builds aus
dem Klon wurden **nicht** ausgeführt — dafür fehlt hier ein `cargo build`-Lauf, die
Konformitätstests im Repo selbst bleiben die Referenz.

Weitere Änderungen wie gewohnt: `git add` / `git commit` / `git push` in dem jeweiligen
Verzeichnis; die Arbeitskopien verfolgen `origin/main`.

**Bewusst nicht im Repo**: `bin/` (gebautes Binary), `target/`, `.venv/`, `logs/`,
`__pycache__/`, die Live-`config.json` des Reglers (stattdessen `config.example.json`) und
`NOTES-local.md` in allen vier Repos.

**evcc-Bezüge**: aus dem **Laderegler** vollständig entfernt (Kommentare, Docstrings,
UI-Tooltips, Test-Labels); die Substanz — drei Batterie-Bänder, `bufferSoc`/`prioritySoc`,
das 3-Phasen-Minimum — steht in `ha-app/NOTES-local.md` (nicht committet). Der **Proxy** wurde
ebenso generalisiert („Modbus consumer", `config/muxproxy.json`, die drei evcc-Werkzeuge nach
`local-tools/`), und die **Werkzeuge dieses Repos**, die evccs API oder CSVs brauchten, liegen
jetzt ebenfalls in `local-tools/` (nicht versioniert) — evcc läuft nie wieder.

**Verifikation läuft ab jetzt gegen die eigene App**, nicht gegen einen Fremd-Controller:
`curl -s 127.0.0.1:7080/api/state` (Lage) · `tests/` der App · `curl -s 127.0.0.1:1504/status`
(Proxy) · `make check` im Proxy-Repo (Konformität gegen den Stub).

**Stand 20.09.2026: alle vier sind veröffentlicht und die Historie ist geglättet** — je
**ein** Commit pro Repo (`evcharge a381fdb`, `modbus-proxy-rs 8fbd4c1`, `ev-charger-wt-ha
a4d0806`, `ha-power-dashboard e3884ee`), gepusht mit `--force` nach Orphan-Branch-Umschrieb,
alte Objekte lokal per `reflog expire` + `gc --prune=now` entfernt. `modbus-proxy-rs` wurde
am 20.09. **gelöscht und leer neu angelegt** und frisch gepusht, weil sein erster
Import-Commit beim Server noch per exaktem SHA abrufbar war; danach war er es nicht mehr.
Nachprüfen lässt sich so etwas nur mit dem exakten Hash:

```sh
git -C /home/adermake/EV-CHARGER-WT-HA/modbus-proxy-rs fetch --depth=1 origin 778db32 && echo "noch da" || echo "weg"
```

Ein frischer Klon des Repos muss außerdem **selbst bauen und testen**:
`git clone … && cd modbus-proxy-rs && cargo test --offline` → 12 Tests, 0 Fehler (das Repo
ist dependency-frei).

## Wo die Wahrheit liegt

* `README.md` (Projekt), `docs/INSTALL.md`, `docs/REGISTERS.md` (Registerkarte, 203 =
  Netz-Zähler), `docs/preview.html` (Dashboard-Vorschau).
* Logs: `logs/evcharge.log` (Herzschläge + Schalt-Schreibzugriffe), `logs/evcharge.stdout`
  (Tracebacks). Der Proxy schreibt nach stdout in eine Datei, nicht ins Journal.
* `ha-app/config.json` — Intervalle, Reserve, `phases`, `proxy_status`, `safety`, und die
  persistierten Einstellungen (`.bak` bleibt erhalten).
* Skills (Prozedurwissen, laden sich bei Bedarf): `ev-charging-control`,
  `modbus-single-client-proxy`, `goe-charger-http-api`, `solaredge-sunspec-modbus`,
  `home-assistant-integration`, `port-verification`.

## Die eine Regel für neue Sitzungen

Erst diese Datei lesen, dann `python3 /tmp/health.py`, dann erst etwas ändern. Der
Wechselrichter ist empfindlich, das Auto teuer, und der Besitzer merkt es, wenn Zahlen
nicht belegt sind.