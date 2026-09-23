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
  pro Stunde und Fläche. Zwei Flächen: **4 kWp Ost (az −90) + 4 kWp West (az +90)**, Neigung 25°
  (die Neigung geht bewusst als Beobachtungsfaktor auf). Standort ist **der genaue Punkt der
  Anlage** — er steht in Home Assistant und in der lokalen `config.json` und absichtlich **nicht**
  in diesem Repo (vorher stand hier der PLZ-Mittelpunkt, der 820 m daneben lag und heute 0,2 kWh
  weniger prognostizierte). Rechnung: `GTI (W/m²) x kWp = Wh` je Stunde (DC-Seite), `x PR 0,85`
  = AC-Erwartung.
* **Belegt:** die App zeigt für heute **28,86 kWh** am genauen Standort (der PLZ-Mittelpunkt
  ergab 28,66) — eine unabhängige Direktrechnung mit demselben Aufruf ergibt **28,86 kWh**
  (identisch). Gegen echte SE-Tage: 22.09 Prognose 27,9 vs 28,5 gemessen (**Faktor 1,02**),
  21.09 **0,74**, 20.09 **0,73**; 23.09 Prognose 28,86 gegen **26,8 kWh** (Besitzer, ~19 Uhr —
  der Wert steigt noch, weil die Akku-Entladung mitzählt) → **Faktor ~0,93**. Also: klarer Tag
  punktgenau, trübe Tage ~27 % zu hoch — deshalb Marge 1,3 im geplanten Regler.
* **Pin (die Zusage von Schritt 1):** der Controller **kennt das Wort `forecast` nicht**
  (`tests/test_pv_forecast.py` prüft das, plus: das Modul hat kein Aktuator-Vokabular). Die
  Prognose *kann* nichts schalten, solange diese Prüfung grün ist.
* **Der Faktor `factor()`** = gemessen heute / Prognose für genau dieses Fenster, nur mit
  echter Basis: unter **0,05 kWh** Messung gibt es **keinen** Faktor (der Tag hat noch nicht
  angefangen — still), außerhalb **0,25–1,60** eine Warnung und ebenfalls keinen. Kein Faktor
  heißt: der spätere Regler fällt auf die sonnenstands-relative Notlösung zurück, nie auf
  geratene Zahlen.
* **Daten:** `logs/pv_forecast_today.json` wird laufend überschrieben (ein Neustart setzt den
  Tag fort), `logs/pv_forecast.csv` bekommt je **fertigem Tag eine Zeile**: Prognose, beide
  Messwerte (AC-Seite und Array-Seite), beide Faktoren, Haus-/Auto-/Akku-Energie, SOC-Bereich und
  `samples`.
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
* **Offen (Schritt 2, wartet auf Daten):** die Regel selbst — Rest-Prognose (korrigiert) minus
  erwarteter Hausverbrauch gegen den Akku-Bedarf bis `priority SOC`; die Schätzgrößen
  (`house_reserve_kwh` 3 kWh, Marge 1,3) sind Platzhalter, bis ein paar geloggte Tage sie ersetzen.

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
6. **MQTT-Ausbau vertagt (Stand 20.09.2026, Besitzer nicht vor Ort).** Der Broker ist da
   (HA-Add-on `Mosquitto broker`, `192.168.178.126:1883` offen), anonym nimmt er nichts an,
   und die App steht bereit (`mqtt.host/port` gesetzt, `enabled: false`). **Es fehlt allein
   das Login.** Es ist **nicht** per HA-API zu holen — belegt am 20.09.: `/api/hassio/addons`
   → `HTTP 401` (Add-on-Optionen liegen im Supervisor, der Token hat keine Admin-Rechte),
   `/api/config/config_entries/entry?domain=mqtt` liefert die Integration „Mosquitto broker",
   aber **leere `data`** (HA gibt gespeicherte Zugangsdaten nie heraus), und ein HA-Benutzer
   als Broker-Login wäre nur als bcrypt-Hash vorhanden. Klartext gibt es nur im Add-on-UI.
   **Wenn der Besitzer zu Hause ist:** *Einstellungen → Add-ons → Mosquitto broker →
   Konfiguration → `logins`* nachsehen (oder einen Eintrag anlegen) und die Werte mit
   `getpass` in `ha-app/config.json` schreiben (landet weder in der Shell-History noch im
   Chat), dann `mqtt.enabled: true`, `systemctl --user restart evcharge-wt.service`, und in
   HA prüfen, ob `sensor.ev_charging_power` & Co. auftauchen.
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