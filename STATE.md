# EV-CHARGER-WT-HA — state, rules, open points

As of: 2026-09-19. **Where there are contradictions, the code and the tests apply, not this file.**
This file is the handover: it says what is running, why it is so, what comes next.

**Language rule, for this file and for everything else the project produces:** whatever language
the owner writes in (German, English, Russian), every artefact is written in **English** - source
code, comments and docstrings, documentation, commit messages, UI strings, log messages. Exactly
two things stay as they are: a **verbatim quote of the owner** (keep the quote in his words and put
an English gloss right after it - the quote is evidence), and **names that belong to a device or to
Home Assistant** (entity names, automation titles, what a device's own display shows).

## What it is about (and why)

The owner has replaced the third-party controller used earlier (Docker container `evcc`) with
his own, independent apps — it is **no longer started**, it would fight over the wallbox.
Reason (verbatim): he does not
want his apps to depend on something — "in diesem Fall von Python. if the system
changes, it can blow up my setup" (in this case on Python). End goal: **static Rust binaries with minimal
dependencies**. The Python versions remain unchanged; Rust is additive, and
porting happens layer by layer, once a layer is frozen. The
control plan is deliberately frozen ("i probably will never need the plan - so let it
as it is now").

Working style he expects: he checks the maths himself and follows up ("bist du sicher?" (are you sure?)),
wants measured value and assumption separated, writes German and expects terse, evidenced
answers with concrete numbers. Requirements for this project:

* **No additional poll load on the inverter (SE5000H, 192.168.178.84:1502).**
  Diagnosis through observation, not through more requests.
* A device that tolerates only one session has **exactly one** session (SE inverter,
  Deye logger). Never two clients at the same time.
* On the wallbox **only `amx` is written, never `amp`** ("amp will turn the flash of
  the wb to trash very soon").
* Manual mode = **zero write accesses**.
* Announce changes to the real plant beforehand, afterwards document the state with evidence.
* Electrical installation is not switched for test purposes — fuses are tested
  with injected times/stubs, not on the car.

## What is running now

| What | Unit / path | State |
|---|---|---|
| Charge controller | `systemctl --user status evcharge-wt.service` | active, Web UI `127.0.0.1:7080`, interval 30 s (car connected) / 300 s (empty) |
| Modbus proxy (Rust) | `systemctl --user status muxproxy-rs.service` | active, listens on `0.0.0.0:1503`, status `:1504`, build `abda451e49a433ff0b42df8635e512491dba3b0f` |
| Deye-PV logger (Rust) | `powerdash-deye-pv-rs` | active, different device, not part of the charging chain |
| evcc | Docker container | **decommissioned** — do not restart; the charge controller drives the wallbox |

Addresses: wallbox go-e `192.168.178.22` (FW 041.0, HTTP API v1) · inverter
`192.168.178.84:1502` · proxy address for consumers `192.168.178.44:1503` · HA
`http://homeassistant:8123` (HAOS 2026.9.2).

The read path to the inverter has proven to be sensitive and is frozen:
**three small windows per cycle** — `(40071,32)` inverter, `(40190,53)` meter,
`(57716,18)` vendor, together 103 registers. The range `40111..40189` is deliberately
**never** touched. The model was the old openHAB configuration (`se4k.things`): two
small windows every 10 s, never a single register on its own.

The proxy returns **no expired frames** (decided this way). Within
`ondemand_ttl` (10 s) two readers share one device read; after that the value is
expired, and if nothing new could be read, the client gets the **fault**
(exception `0x0B`) instead of an old frame — that would look like a fresh
measurement on the wire and the charging app would have no chance of recognising it. Consequence for the app: a
failed read yields no decision at all (no cycle), and a running
charge is ended after `site_stale_s` (600 s) — never started or
readjusted on the basis of old numbers. The Rust proxy and the Python reference behaved the
same **on the wire** — 14/14 identical responses, and the conformance test pins the expiry
rule (`an_expired_read_is_reported_instead_of_served_stale`). The **Python implementation
itself no longer exists** (deleted 2026-10-03, see the layout note under Repositories):
what came out of it and stayed is its wire-protocol suite, which `make check` still runs
against the Rust binary.

The wallbox is read via the **measured** `nrg` offsets (FW 041.0, the documentation is
off by one position): `nrg[0..2]` volts · `nrg[3]` constant 1 · `nrg[4..6]` currents
(0.1 A) · `nrg[7..9]` powers (0.1 kW) · `nrg[11]` total power (10 W) · `nrg[12..14]`
power factor. The number of phases comes from the currents; `pha` (63) is contactor fitment and
is never believed.

## Rules in the controller (with reasoning)

* **Surplus** = grid power (signed) + car power − battery discharge,
  plus the battery **intake**, once the battery has reached `priority_soc`. No
  base-load constant: the house load is already in the meter value, a second subtraction
  would count it twice and would keep the charge ~1 A too low (`residual_power_w` is a
  deliberate reserve and here 0). The discharge always remains subtracted — the car
  never discharges the house battery.
* **Three bands for the battery** (decided this way; behaviour taken from a generic EV charging app):
  **below `priority_soc` (55 %)** the battery has priority: its charging share stays with it,
  the car charges from the real export (in which the share is already missing, the meter measures it
  too) — but it is **not** blocked. A grey day with a half-full battery thus charges the car
  instead of pushing the sun into the grid. In addition the charge stops when the
  battery feeds the car and nothing is exported.
  **from `priority_soc`** the battery's charging share is added on top: the car overtakes
  the battery when charging, the battery stays at its level instead of running to 100 %.
  More than `max_current` (14 A) the car cannot take over — the rest continues into
  the battery or into the grid respectively.
  **above `buffer_soc` (80 %)** the reserve above the buffer may carry a **running**
  charge: pv/minpv keeps the car at 6 A instead of switching off
  as the sun fades; that lasts until the battery is back at 80 %. A charge is
  **never** started from the battery.
  The battery's discharge otherwise remains subtracted — the car does not discharge the house battery.
  `battery_boost` and `buffer_start_soc` are **removed** (remnants of the old
  buffer block; for "Akku ins Auto entladen" (discharging the battery into the car) the owner takes the `manual` mode).
  All three are in the UI in the tooltip of the "battery" row and on the buffer/priority SOC fields.
* **Current follows the surplus immediately** in the range 6 A…maximum. Hysteresis exists only at
  the lower limit, and it is **asymmetric** (since 2026-09-21): a **start** requires the
  lower limit **+100 W** (`enable_threshold_w`, set this way by the owner — a late start
  gives away sun), a **running** charge is **held** down to **−300 W** below it
  (`disable_threshold_w`, deliberately larger: an unnecessary stop costs a session) and only
  switched off below that. At 6 A / 1 phase therefore: start from **1480 W**, hold until **1080 W**.
  In the band in between the state stays as it is — exactly the window in which the app
  previously started and stopped on a minute-by-minute basis.
* **House-battery guard hangs on the decision, not on the mode name**: with
  surplus charging, charging does not happen below `buffer_soc` (and the reason is displayed); in the
  cheap-tariff window the battery is not the topic, because there the grid pays.
* **Cheap-tariff window (`cheap_hours`)**: inside the window everything is irrelevant — maximum and
  continuously until the end, no waiting, no battery block, no regulating down, no
  meter-freshness check. At the end **no** switching off, but seamless handover to the
  PV rules; those apply before, during and after it, so that the mode can stay on
  permanently. A running charge must be **held** inside the window — exactly that was
  the fault that switched every ~5 minutes for a whole night (69 edges).
  **Fixed on 19.09. at 07:46** (`controller.py`: branch without `and not charger.charging`);
  three checks in `test_controller.py` pin it („a running charge in the window stays at
  maximum", „no dwell timer or battery block interrupts it", „and it emits no on/off
  edge"). In `pv`/`minpv` the window remains ineffective — at night nothing happens there,
  and that is intended.
* **Phases**: after plugging in **1 phase** is assumed; the meter is only
  believed once ~20 s of current has flowed, and then the **highest** value seen is remembered until
  unplugging.
* **Safety latch**: The app counts the **stops** (`alw=0` after an `alw=1`) that the wallbox
  actually saw, rolling over 30 minutes, and shows them in the UI. At 5 a
  **fault** latches in: switch the charge on once, afterwards **no** more write accesses. Reason:
  "OBC eines Autos zu reparieren kostet Tausende Euros" (repairing a car's OBC costs thousands of euros). **Only stops, not both
  directions** (owner's decision, 2026-09-21): every start is the counterpart
  of a stop; counting both directions reported an interrupted session as two
  events and halved the tolerance. `amx=` writes count **never** — regulating is
  not switching. **Manual release — and a restart is such an action:
  `systemctl --user restart evcharge-wt.service` acknowledges the fault on purpose**
  (owner's decision, 2026-09-21). The counter lives only in memory, and that is
  intended: whoever restarts has looked at the cause beforehand. **Not** intended is the
  opposite — that the fault clears itself during the run because the counter
  ages; in operation the latch persists.
* **Session energy is measured twice** (since 2026-09-22). The app computes the
  go-e session from the phase measurements of the wallbox — over the **measured** cycle time
  (not `interval_s`, that is only the target) and with a **reset on unplugging**, so that
  "session" really means a plug-in session. In parallel it keeps a second figure over
  the **SDM630** in the garage branch (`ha-app/evcharge/session_meter.py`): on plugging in
  its kWh counter is remembered, during the session the difference is the session, on
  unplugging it is frozen as "last session" and **written into `logs/sdm_sessions.csv`**
  (one line per session, with the go-e value for comparison). The SDM630 measures
  the garage branch, into which the garage PV also feeds — the figure is therefore the **view of the
  meter** (car minus PV share); export is carried along so that the PV share remains visible.
  **Three values, as requested** (2026-09-22): (1) go-e, (2) SDM = `import − export`,
  (3) **SDM + garage PV** = `import − export + PV` — the balance of the branch, because exactly that
  is what the car drew and the meter did not see. Value 3 is the inverter's
  opinion and **only as good as its meter**: over the window 21.09. 05:00–17:00Z
  there is a **gap of up to 8 %** (inverter 4.18 kWh against
  3.88 kWh that the SDM saw flow out) — **upper bound, not a measurement**: the branch carries
  permanently the router, gate and go-e standby, and the SDM cannot **count** such small loads
  (starting current 0.04 A), so its export counter reads exactly too little of what they consume.
  The 0.30 kWh gap is 25 W continuous load over 12 h; the owner's estimate (router ~10 W
  + go-e ~5 W) already covers 0.18 kWh of it. The real error of the inverter thus lies
  between ~0 % and +8 %. And after every wake-up the register briefly reports
  **0.00 kWh** (poller journal 05:16:53) — such values are discarded and counted
  (`pv_artefacts`), otherwise the next real number would appear as a ~279 kWh "correction".
  If the meter is missing entirely (at night the logger sleeps, the PV is then really 0), the
  correction is **0** and the CSV line says so; if it arrives in the middle of the session, the baseline
  is caught up and the line calls the correction "partial".
  **Freshness:** a counter that does not change is **not** rewritten by HA — so its
  age says nothing. That is why the age of the **power** decides on "stale"
  (`entity_live` for the SDM, `entity_pv_power` for the Deye), and with outdated values
  the session **waits** instead of inventing 0 kWh.
* **The inverter is only read, never written** (2026-09-23, owner's
  decision: „Ich will nicht Akku steuern" (I do not want to control the battery)). The SE driver contained two **never called**
  write functions (battery mode `0xE00D`, discharge limit `0xE010`) and the Modbus client the
  write primitives — all **removed**, together with the likewise dead export-limit constants
  (`0xE000..0xE002`). This is now **structurally verified** (`test_solaredge_decode`): the
  client has no write method, the driver nothing that controls the battery, and the
  register addresses may not reappear in the *code* (in the module documentation they
  remain, so that it is clear *why* they are gone). **Live guard:** the proxy counts
  `upstream_writes`, and the app classifies any number > 0 as
  **"PROXY WROTE TO THE DEVICE" (bad)**. Status: **0 write accesses** at 59k poll cycles. Only the
  **wallbox** is written (current, enable, neutral position) — and only via the one write channel.
* **House time is not host time**: The host runs UTC, desired wall-clock times are expressed
  in `Europe/Berlin` (`Settings.timezone`, IANA name, daylight-saving-safe).

## PV forecast — step 1: display only (2026-09-23)

The owner wants the **car** to have priority in the morning and the **battery** in the
afternoon — decided by weather, not by time of day. This requires a local forecast.
**What is built is step 1: the forecast is displayed and logged daily; it controls
NOTHING.** That way he can first judge whether such a forecast is any good for his roof.

* **Source:** Open-Meteo, `global_tilted_irradiance` per roof surface, without a key, one request
  per hour and surface. Three surfaces, as the owner corrected them: **4.48 kWp east
  (az −90, 14 modules)**, **1.60 kWp west roof (az +90, 5 modules)** and **1.92 kWp west dormer
  (az +90, 6 modules, flatter than the roof)**. The location is **the exact point of the
  plant** — it is in Home Assistant and in the local `config.json` and deliberately **not**
  in this repo (previously the postcode centre stood here, which was 820 m off and today forecast 0.2 kWh
  less). Calculation: `GTI (W/m²) x kWp = Wh` per hour (DC side), `x PR 0.85`
  = AC expectation.
* **Evidenced against seven days with 15-minute exports from the SE portal** (2026-09-24; each file
  sums exactly to the portal day value, the timestamps are local time — checked by correlating the
  day shape against the forecast, not assumed). Forecast against the afternoon
  (12–18 h, from the 15-minute values):
  05.09 **33.1 kWh forecast / 18.8 kWh afternoon**; 06.09 **33.2 / 19.2**; 08.09 **31.3 / 17.9**
  — three days above **88 % of the daily ceiling**, three times a **strong** afternoon.
  21.09 25.4 / 14.1; 07.09 26.4 / 13.3; 10.09 27.8 / **8.9** — three days at **71–78 %**,
  three times medium to weak. **Between 14.1 and 17.9 kWh there is not a single observation** —
  the two groups do not overlap. From this follows the threshold: not 70 %, but
  **~88–90 %**. With 90 % of the 30-day best value the rule gets **7 of 7** days right.
* **Refuted: the day-factor correction** (that is, the earlier idea in this section). 08.09 and
  10.09 are mirror images: on 08.09 the morning was **dead** (2.56 of 6.88 kWh) and the
  afternoon **strong** (1.13-fold); on 10.09 the morning was **exactly as forecast** (0.98)
  and the afternoon collapsed to **0.51**. The morning therefore says nothing about the afternoon —
  in *no* direction. A day factor formed at 12 o'clock would have said "all is well" on 10.09
  and let the battery run empty in favour of the car. The factor is now only **recorded along**,
  no longer followed.
* **Side finding that explains the calibration:** over 23 days the forecast day total looked
  unbiased (mean 1.008) — but the **evening** (battery discharge, 1.13- to 1.58-fold) concealed
  that the **afternoon** on average delivers only **0.71** of the forecast. For this rule the
  afternoon counts, not the day total: **the day total works as a display, not as a statement about the
  day shape.** That is why the day shape (morning/midday/afternoon/evening) is now part of every
  day record.
* **The shape of the measured values comes from the hourly log** (`logs/pv_hourly.csv`, cumulative per
  hour) — it exists precisely for this reason, before the rule exists.
* **Pin (the promise of step 1):** the controller **does not know the word `forecast`**
  (`tests/test_pv_forecast.py` checks this, plus: the module has no actuator vocabulary). The
  forecast *cannot* switch anything as long as this check is green.
* **The factor `factor()`** = measured today / forecast for exactly this window, only with
  a real basis: below **0.05 kWh** measurement there is **no** factor (the day has not yet
  started — silent), outside **0.25–1.60** a warning and likewise none. No factor
  means: the later charge controller falls back to the sun-position-relative fallback, never to
  guessed numbers.
* **Data — all on disk, nothing only in memory** (a restart must not cost the history):
  `logs/pv_forecast_today.json` is continuously overwritten (a restart continues the day),
  `logs/pv_forecast.csv` gets **one row per completed day**: forecast (total **and** in
  four day sections), both measured values (AC and array side), both factors,
  house/car/battery energy, SOC range, `samples` — and the rule columns `best30_kwh` (reference),
  `best30_threshold_kwh`, `best30_pct`, `rule_best_says` (rule B) and `rule_margin_says` (rule A).
  Plus `logs/pv_days_seed.csv` with the **22 day values from the PV portal export**, so that the
  30-day reference exists immediately after a restart (currently the best value **33.498 kWh** from
  06.09).
  **Since 25.09 the garage meter is also in the row** (`sdm_import_kwh`, `sdm_export_kwh`
  and the day difference `sdm_import_day_kwh`/`sdm_export_day_kwh`) — that is the owner's
  reference for the car. Reason: on **23.09 19.38 kWh went into the car**, while the service was not yet
  running; the day record books `car_kwh 0.0` there. Carrying the meter along in the same row
  makes such gaps visible instead of finding them by hand weeks later (sum 22.–25.09:
  **29.22 kWh** at the meter against 7.63 kWh in the app count). The day difference arises from the
  last completion; on the first day it stays empty instead of guessed.
* **`samples` is important:** the day values are a zero-order-hold reading, they inherit the
  inverter's read cadence (with car every ~5 s, without car deliberately throttled — the
  rule "no additional poll traffic" also applies here). With stale measured values
  the charge controller integrates **nothing** (stale gate), instead of continuing to count old values.
* **Production now comes from the inverter meter, not from an integral**
  (2026-09-23): SunSpec model 101 `WH` (word 22/23, scale factor at 24) lies **within
  the window that is read anyway** — so costs **zero** additional Modbus traffic (a test pins: there
  remain three read operations with 103 registers) — and is exact: it loses nothing while the
  service is up, and it counts the later **battery discharge along** (that is the number that the
  monitoring app calls "production", and it is fair: what the
  inverter delivered is counted once). Proven live: over the same 3.5 minutes the meter rose by
  **0.0200 kWh** and the app's AC integral by **0.0200 kWh** — identical. Lifetime reading
  23.09: **29,267.336 kWh** (scale factor 0).
* **Caution when comparing after a restart:** the integral is **continued** from the day file,
  the meter start point only if the day file has one — directly after
  a restart the two numbers can therefore cover **different windows**. Comparing
  means: **deltas over the same window**, never the totals (the first look therefore showed
  0.100 against 0.032 kWh and was not an error). A start point that was not set at midnight
  is marked in the row as `se_partial`.
* **Rule B is built — as a display with history** (2026-09-24): the forecast is set against the
  **best complete day of the last 30**, threshold **90 %**; above it "car priority",
  below it "battery priority". The reference comes from the plant's own measurement series, so that it grows with the
  season (June ~52 kWh, September ~33.5 kWh). Both rules are **recorded daily
  along**, the history later decides which was right. In the UI both are shown:
  "forecast vs best day (30d)" and "rule B (season) would say". **It still controls nothing** —
  the controller pin is unchanged green.
* **Open (control, waiting for the collected days):** only wire it up when the history
  confirms the rule. The estimated quantities of rule A (`house_reserve_kwh`, margin 1.3) remain
  placeholders and will presumably not be needed for rule B.

## Recently fixed (2026-10-03)

**The actual cause of the lost cycles: the device closes every connection after ~330 s — fixed
by not carrying a connection across the idle stretch.**

* **Measured:** of 844 failed reads whose upstream-connection age is known, **809 sat on
  connections 5–6 minutes old** — median and p90 exactly **330.3 s = 11 × the app's 30 s
  cycle**. The raw log shows the loop: client connects, one upstream connection is opened, 11
  read cycles run over it, the 12th read (at 330 s) gets EOF, the client is served exception
  `0x0B` and loses that cycle, 30 s later it reconnects. So the device closes a connection
  after ~330 s *whether or not it is used*; the proxy was carrying one connection across those
  11 cycles and letting the device close it under a read.
* **The fix: `upstream.idle_close_s`** (default 0 = old behaviour, set to 10 in service) — the
  proxy closes an upstream connection once it has been quiet that long, so each burst opens
  its own and no connection ever reaches 330 s. A connect costs **~1 ms** here (median 1 ms,
  max 32 ms from client arrival to upstream connect), and this device has already taken far
  more: 10,474 connects in ~1.5 days (≈7000/day) during the old consumer's era. The new
  counter **`upstream_idle_closes`** grows while `upstream_errors` stays flat — that is the
  proof it is housekeeping, not a fault. It is deliberately *not* counted in
  `upstream_reconnects` (the app's card shows that as a fault signal) and does not enter the
  failure backoff.
* **Verified offline:** 2 new conformance tests (an idle stretch produces exactly one new
  connection and the reads *within* a burst still share it; 0 keeps the old behaviour),
  44 cargo tests, 17/17 cross-check, **14/14 differential against the baseline** (the wire is
  byte-for-byte unchanged), 13/13 poll-range config.
* **Deployed 2026-10-03 at 18:07:01 host time (20:07 in the house):** stop → `make static
  install` → start, the installed binary (`9f552c32`) checked against the build, and the
  static build also put through `make prodcheck-static` (13/13) before it was started. The
  first minutes show what was intended: `upstream_idle_closes 1` while `upstream_errors` and
  `upstream_reconnects` stay 0, and in the log a fresh `upstream connected` **per burst
  without a new `client connected` in between** (18:07:39 and 18:08:39) — the app's own client
  connection stays open, only the device session is renewed. The app's proxy card reads
  `PROXY OK`, 6 reads, 0 errors, site value fresh (671.8 W).
* **Still to do:** (a) watch `upstream_errors` over the next days — it must stay flat while
  `upstream_idle_closes` climbs, one per read burst. A burst lands every **~60 s**, not every
  30 s as `interval_s` suggests: measured 2026-10-03, upstream connections 59.98 / 60.08 /
  60.02 s apart, because the app's "due" test is measured from the *end* of the previous read
  and that read takes ~2.5 s (three windows at the proxy's `min_request_gap` of 1 s). So the
  counter should reach ~1440/day, not the ~2880 a true 30 s cadence would give — and that same
  ~2.5 s is why `idle_close_s` has to sit above ~3 s; (b) the baseline in
  `baseline/muxproxy` is still the 19.09 build, i.e. this change's yardstick — once the
  counters have been clean for a day, `make baseline` moves it to this build; (c)
  `upstream_idle_closes` is in `/metrics` but not surfaced in the app's card yet.

**The proxy's `response_timeout` stood at 5.0 while every note about it said 1 s — now 1.0, with
a reason that still exists.**

* **The old reason no longer holds.** 1 s was chosen because a SunSpec model scan probed address
  50000, which this inverter never answers, and a longer proxy timeout let that client give up
  first (`i/o timeout`, then `not a SunSpec device`). Measured in the logs of 14.09.–03.10.:
  **not one read in the 50000–56999 range**. That client is gone. (An earlier count of "4609
  hits" was a wrong grep of mine — my pattern also matched the 57xxx vendor blocks.)
* **The failure picture, by date — and it is 95 % history, not this app.** The three logs hold
  19,508 `failed:` (EOF) attempts, but **19,036 of them fall on 14.–17.09.**, on the read pattern
  of the *old* consumer: `2@57716` 5011, `105@40190` 3948, `2@57732` 3780, `50@40071` 3493,
  `4@57722` 1229, `4@57718` 1022 — full SunSpec models and single registers, none of which this
  app ever asks for. From 18.09. (the charging app becomes the client) that pattern drops to
  **zero** and what remains is the app's own three windows: `32@40071` (964), `53@40190` (14),
  `18@57716` (15). Day by day in the app era: 18.09. 39, then 1–3/day to 24.09., a burst of
  40/72/239/241/70/71/85 on 25.09.–01.10., then 3 and 1 on 02./03.10. — roughly **1–3 lost
  cycles per day** out of ~1440 site reads (the app reads the site every ~60 s, see below),
  and each one costs the app the cycle.
* **The mechanism, from the code — not a register count.** The proxy takes the upstream answer
  with `read_exact` (`src/upstream.rs:230/244`); a short frame followed by close surfaces as
  Rust's `failed to fill whole buffer` (UnexpectedEof). A device that *refuses* a quantity
  answers with a Modbus exception — a complete frame — it does not close the socket. So this is
  the device **closing an idle connection**, and the *first* read of the following cycle is the
  one that discovers it; that is why `32@40071`, the first of the three windows, is the one that
  fails (the other two follow on the same connection and are fine). The proxy then answers the
  client with exception `0x0B` instead of an expired frame and leaves the device alone for 5 s —
  long enough for the app's own 0.25 s retry to fail too, so the cycle is lost rather than
  recovered. **Splitting a read does not address this:** a 2-register read fails the same way
  (5011 times above), and 32 → 16+16 would move the hit to the second half, double the requests
  and mix two ages inside one SunSpec window (the AC power and its scale factor). If it is to be
  attacked, the targets are (a) TCP keepalive on the upstream socket, so a dead connection is
  replaced before a client asks instead of being discovered by the first read, or (b) not
  applying the 5 s "leave the device alone" to a plain EOF, since a closed connection is a
  reconnect case, not a flaky device — then the app's built-in retry recovers the cycle.
* **Two other failure kinds, both rare and both elsewhere:** `read N@M timed out` (>5 s response)
  appears 531 times, **all on 14.–17.09. and all on the old consumer's blocks** (`50@40071` 298,
  `4@57718` 208) — in the app era **no read has ever exceeded 5 s**. And 5 ×
  `connect: connection timed out` (evenings of 14./15./16./23./28.09.): the proxy could not reach
  the device at all; the last two hit the app's `32@40071` and cost it a cycle each. Since the
  bound is now 1 s, a read in the 1–5 s window becomes visible as `timed out` for the first time.
* **The number that was missing:** the app's Modbus client waits **6 s** (`drivers/modbus.py:20`,
  nothing overrides it), so a 5 s proxy timeout sat only 1 s below it — the app could have timed
  out before the proxy answered, exactly the confusion the old note described.
* **Decision: 1.0**, for two reasons that hold today: the proxy holds the plant's *only* device
  connection, so one hung read stalls every reader — 1 s bounds that; and 1 s keeps the proxy
  clearly under the app's 6 s, so the app always receives the proxy's clean exception `0x0B`
  instead of its own socket error. 1 s is ~10x the measured median (50–100 ms, p95 161 ms).
* **Checked:** restarted 17:41:56. After three app cycles `upstream_timeouts 0`,
  `upstream_errors 0`, no warning in the log, app card "PROXY OK", site values fresh (666 W).
  The single `site read failed: connection closed` at 17:42:38 is the restart itself.
  **Still to watch:** this counter must stay 0 over the coming days; if it rises, 1 s is too
  tight for this inverter.

**The buttons in the "Mode & settings" card seemed sluggish to respond — it was the display,
not the switching.**

* **Measured:** the web server responds in **~0.8 ms** (`/api/state` 5.9 KB, five measurements
  0.77–0.93 ms), the browser sees **2–3 ms** per request. The click itself therefore never arrived too
  late — the log shows the writes of the last days to the second.
* **The core:** `state["mode"]` and `state["control_enabled"]` were **only set in the cycle**
  (`cycle()`, `main.py` line 501/502). `update_settings()` wrote only
  `state["settings"]`. But the button reads exactly the two *other* fields:
  mode badge, highlighted mode button and the label "Enable/Disable control"
  (and DRY RUN) — the input fields next to them read `state["settings"]` and were immediately
  correct. Consequence: a click looked like nothing for up to **one cycle**.
* **The number for it:** the loop runs with `interval_s: 30`, measured **32.2 s** between two
  cycles (`cycles`/`last_cycle` read along over 40 s) — plus up to 3 s page poll. So
  **on average ~16 s, in the worst case ~33 s**, until the button responded. Exactly this
  pattern is also in the log: the same mode twice within seconds
  (26.09 11:58:06 and 11:58:13 `manual`; 03.10 14:59:59 and 15:00:05 `cheap_hours`).
* **Fix (2 lines, `update_settings`):** `state["mode"]` and `state["control_enabled"]`
  are now set **also** on change. `cycle()` still writes them every cycle
  — so there are not two truths, only one that is earlier. Proven offline (service with
  stub drivers, real `update_settings`/`cycle`): before, `mode` after the click was `pv` and
  only in the next cycle `manual`, now immediately `manual`.
* **Pinned** in `tests/test_service_smoke.py` ("a settings change is visible in the state
  snapshot at once"): mode and control flag are directly in the snapshot, `settings`
  matches, an unknown mode is still rejected and leaves nothing behind.
  All **13 suites green** (511 → 516 checks).
* **Not yet live:** the running service is from 30.09 — the two lines take effect only
  after `systemctl --user restart evcharge-wt.service`. A restart resets, as always, the
  day bases (SE meter, garage PV) as a "partial day" and the go-e session counter to 0;
  the car is currently connected with `complete`, no charging is running.
* **Lesson:** when a button seems "sluggish", first measure the server (milliseconds against
  seconds) and then ask **which** field the display reads. The state was never old,
  only the copy of it.

**And the second reason why a field "jumps back" (same day): the page overwrote it
itself.**

* **Observation by the owner:** "wenn ich etwas, z. B. 1 in das Feld plan kwh schreibe,
  dann wird es beim nächsten update zyklus auf 0 zurückgesetzt." (when I write something, e.g. 1 into the field plan kwh, then it is reset to 0 on the next update cycle.)
* **Reproduced, and the server was uninvolved:** the 3-second refresh wrote
  **every** settings field unconditionally anew from `state["settings"]`. A typed `1` was
  back to `0` after **4 s** (measurement on the real page), `config.json` stood at `0.0` the whole
  time — so nothing was ever saved and nothing discarded, only the
  display overwritten. The same pattern would have happened to every field.
* **Fix in JavaScript (`UI_HTML`):** the fields are now filled only when they are **not**
  currently being edited (`document.activeElement`) and **not** marked as `dirty`.
  The first keystroke sets the mark, a submitted save clears it
  (`clearDirtySettings()`) — afterwards the fields follow the server again, a rejected
  value therefore visibly jumps back to the real state instead of looking "saved".
  Untouched fields continue to be tracked.
* **First checked offline, then rolled out:** a small server in `/tmp` serves the
  **real** `UI_HTML` against a dummy of `/api/state` + `/api/settings`. With that:
  a typed `1` survives **three polls**; an untouched field still follows an external
  writer (`min_current` 6 → 9); after save, `1` is in the display **and** in the server.
  Then restart (16:46:52) and the same check repeated **live**: a typed `1` after
  **9 s / 3 polls** still there, `plan_energy_kwh` on the server unchanged `0` — nothing
  saved, nothing touched on the plant. Three source pins in
  `tests/test_service_smoke.py` (13 suites green).
* **`plan kWh` / `plan by` were removed from the code on 2026-10-03 at his request**
  ("ich brauche das nicht" [I do not need that]). Gone are: the fields `plan_energy_kwh`/`plan_deadline` in
  `Settings`, the decision fields `planned_kwh`/`plan_active`/`plan_wait`, the plan branch in
  `decide()` including `_plan_required_w`/`_plan_start_in_s`, the `session_kwh` argument of
  `decide()` (only the plan ever used it), the two input fields and the plan text in the
  countdown of the page, the HA number entity "EV charge plan energy" and the
  `set/plan_deadline` command in the MQTT client, the add-on options and the schema in
  `config.yaml`, the two keys in `config.json`/`config.example.json`, the README line
  and the old local `local-tools/mqtt-via-ha-rest` twin.
  **Reason (measured, before):** the branch was reachable only in `pv`, `minpv` and
  `cheap_hours` outside the cheap-tariff window — in `now`, `off`, `manual`, without car and in the
  cheap-tariff window the earlier `return` silenced it. And while it was waiting for a distant target, it
  **suppressed a good solar surplus**: 3000 W export, mode `pv`,
  without plan **13 A**, with plan **0 A**.
* **The removal is behaviour-neutral — proven, not claimed.** A/B against the old version
  from `git` (HEAD) with the **live-set** values (`plan_energy_kwh` 0, `plan_deadline` empty):
  **1152 scenarios** (six modes × time of day × grid power × SOC × plugged in/charging) with
  **0 differences** in `charge`, `target_current`, `surplus_w`, `blocked_by`, the
  grace counters, `manual`, `cheap_now` and `phases` — and **576/576 identical
  justification texts** (after normalisation of the separate wording change that was already
  uncommitted in the tree before).
* **Pinned so it does not come back:** `tests/test_controller.py` checks that `Settings` and
  `Decision` no longer have a plan field, that the word "plan" no longer occurs in the controller
  module (word-boundary-aware, so that `plant`/`plant_now` do not trigger) and that `decide()` no
  longer accepts `session_kwh`; `test_service_smoke.py` checks the page (no `set_plan` fields,
  no "plan:" in the countdown, no plan keys in the add-on options); `test_mqtt_loopback.py`
  checks that no plan entity and no plan command is left in the MQTT client. 13 suites green.
* **HA side — also tidied up:** Discovery messages are **retained** in the broker, so a
  removed entity stays in Home Assistant. Checked and fixed: the topic
  `homeassistant/number/evcharge_wt/plan_energy_kwh/config` was still there (18 Discovery topics),
  deleted with an **empty retained message** → **17 topics**, the plan no longer appears in any
  message, and HA no longer holds `number.ev_charger_wt_charge_plan_energy`
  (HTTP 404 instead of previously `state 0.0`). Checked beforehand: **no** reference to it in
  `entity_references.json` (zero hits on `evcharge` in any board or automation).
  New tool for this: `tools/mqtt_retained_audit.py` (lists the retained Discovery topics
  of this service; `--clear <topic>` deletes one — without emitting any credentials).

## Recently fixed (2026-10-02)

**New ZHA socket "Fliegengrill" — measures, switches, hangs on button and night automation.**

* **The socket itself is verified.** Tuya `_TZ3000_gjnozsaz` / **TS011F**, IEEE
  `a4:c1:38:02:08:5c:ff:ff`, `device_id d5a254ad4af3f63eaf15f456ff1db999`. Measured with the kettle
  as load: **1971.0 W / 8.547 A** and **voltage 237 → 228 V** at the moment of
  switching on (the inrush current pulls the line down). It calculates correctly. The question
  connected with this is thereby answered: **the socket reports small load correctly**, below
  its reporting threshold, however, a Fliegengrill (2–8 W) can remain at 0.0 W — for "is it
  running?" the **switch state** is the reliable indication, not the power.
* **Renamed and filed** (`config/device_registry/update`, read back):
  device **"Fliegengrill"**, area **`wohnzimmer`** (to where "Steckdose Kühlschrank" sits).
  The 13 entities are now called "Fliegengrill Leistung / Spannung / Stromstärke / Summe
  verbraucht / Kindersicherung". The **technical entity_ids remain generic**
  (`switch.tz3000_gjnozsaz_ts011f_11`) — deliberately: renaming the IDs is a breaking
  intervention, the name on the device is what the UI shows.
* **The button: IEEE `a4:c1:38:4b:8a:95:40:dc`** — a HOBEIAN `ZG-101ZL`, which had been listed
  as dead for **370 days** and transmits again after re-pairing (`avail=True`, LQI 172,
  battery 100 %). It was already on the network (`zha/devices` stayed at 64 devices) — ZHA attached
  it to its **existing** entry based on the IEEE. Renamed to
  **"Button Fliegengrill"**, area `wohnzimmer`. **Physical marking: "05"** — the owner
  calls it that; until today the button had no function, now it switches the Fliegengrill.
* **The trap that would have cost a lot of time:** The button device was set to **`disabled_by: user`**,
  and therefore all seven entities had `disabled_by: device` — they respond with **HTTP 404**
  and are invisible in the UI. The automation ran anyway, because **ZHA processes `zha_event` even for
  disabled devices**. The repair is to enable the **device** (`disabled_by: null`),
  not the entities: a pass over the entities found "nothing to do", and ~25 s later
  the values were there. Note: read `disabled_by` on the **device** before touching entities.
* **Three automations, all `state=on`:**
  * `fliegengrill_an` — 00:00 → `switch.turn_on`
  * `fliegengrill_aus` — 05:00 → `switch.turn_off`
  * `fliegengrill_taster` — `zha_event` with **filter in the trigger** (`event_data.device_ieee`),
    condition `command == 'toggle'`, action `switch.toggle`, `mode: single`
  Both time automations carry the condition
  `{{ now().month >= 4 and now().month <= 11 }}` (April–November). **HA runs on
  `Europe/Berlin`** — 00:00/05:00 are house time, not server time. "Von 0 bis 5" (from 0 to 5) is the
  owner's specification (Fliegengrill at night).
* **Proven end-to-end, not claimed:** an artificial event via
  `POST /api/events/zha_event` switched the socket (`last_triggered` set), and afterwards the
  **real** button switched **seven times** in a row (17:52:44 … 17:54:17) — every
  movement registers. Only with this is the radio path evidenced, not just the automation.
* **Trap when reading back the automation:** HA names the fields in the config view in the **plural**
  (`triggers` / `conditions` / `actions`), not `trigger`/`condition`/`action`. A read-back with
  the singular keys shows `None` for a completely correct entry — that looked like a
  failed write and was not one.
* **Side finding:** The kettle dropped at 17:47:19 from 1964 W back to 0.0 W, the voltage
  to 237 V. And `zha_event` reports for the **Eingangstür** (`00:15:8d:00:8b:bb:3c:8c`) at
  17:55:18/17:55:25 `attribute_updated` — so it lives on.

## Recently fixed (2026-09-25)

* **Battery round: four devices back, and one misinterpretation corrected.** Treppe Keller, Erstes
  Geschoss Treppe, TreppeEG Rechts and Button Altar came back by themselves after a cell change.
  **Button Mascha PC** additionally after **re-pairing in ZHA** — the entity
  `switch.hobeian_zg_101zl_4` stays unchanged in the process (ZHA attaches to the
  existing device based on the IEEE address: no `_5`, no dead entry, automations stay valid). A single press of the button
  does not wake a device without network registration, no matter how often. Fifth button of the same type and
  intact: `switch.button_pc_wt` ("Button PC WT", switches the PC off, battery reports freshly).
* **"stumm seit 19.09." (silent since 19.09.) was a misinterpretation.** That is only the timestamp that HA writes
  onto the entities on restart. The **last real contact** is in ZHA per device (`zha/devices`
  → `last_seen`, `lqi`, `available`): all still-silent battery devices had last sent **88 to
  369 days** ago — **no** device failed on 19.09. Triage rule from this: under ~8
  days = real candidate for a cell, months/years = dead entry (replacement, dismantled) — no
  battery helps there.
* **Three dead entries disabled** (nameless ZG-101ZL `…a2:81:f4` and `…95:40:dc` as well as the
  nameless `_TZ3000_zutizvyk TS0203`) via `config/device_registry/update` with `disabled_by: user`
  — reversible with `null`. Evidence: entities in HA **605 → 590**, read-back `disabled_by=user`. The
  silence alarm no longer names them: it has no hard-wired names but scans dynamically,
  and disabled entities leave the state machine. **WasserSensor Heizung** (88 d) re-registered
  the same evening after a cell change **by itself** (100 %, LQI 148) — so a re-pairing is not always
  needed; **Eingangstür** (121 d) deliberately stays (supposedly in
  operation, assignment still open: next to it there is the living twin `AqaraSensorSZTür` — its
  opening entity permanently reads **offen**, because the bedroom door is practically always tilted
  open; that is correct and **not** a defect, temperature and battery report freshly). Newly
  noticed and **clarified on 26.09. — see the entry on Ralf's TRV further below** (silent since
  20.02.2026, 217 days, no `climate` entity any more).
* **EINGANGSTÜR SENSOR BACK IN OPERATION (28.09., 17:47) — after 124 days of radio silence.**
  Last message was 27.05.2026. Sequence, measured in this order:
  * **Cell changed + button pressed** → device came back at **17:31:23**: `available: True`,
    LQI 136, battery 69.5 %, temperature 31 °C. **It then sent nothing more.**
  * **Two open-close cycles and a directly applied magnet** → **not a single radio message**,
    `last_seen` stayed at 17:31:23. The LED lit up — it only proves local current.
    This **ruled out mounting distance and cell polarity** (with wrong polarity it would
    not have come back at all).
  * **Deleted + re-paired (17:44:19 → 17:45:23) → fully functional:**
    ```
    15:47:25  binary_sensor.door_offnung -> on    (ZHA last_seen 17:47:22, lqi 144)
    15:47:29  binary_sensor.door_offnung -> off   (ZHA last_seen 17:47:25, lqi 132)
    ```
  * **The cell was fine all along** — the obvious suspicion was wrong.
  * **Registry unchanged:** device ID `1b4cd99bc3c312b75088e774d5c5bc22`, name "Eingangstür",
    area `eingang`, all four entity IDs identical (ZHA keys via the IEEE). The board
    "Fenster/Türen" needed **no** adjustment — the previous warning was unfounded.
  * **Note (important):** A freshly connected device can be **half-finished paired** —
    reachable, good LQI, a set of plausible values, then silence. Invisible from outside.
    After every pairing **trigger a real event** and demand a **new** `last_seen` *plus* a
    state change in the history. "It reported during pairing" is the failure case,
    not the proof.
  * **Tools** (local, gitignored): `local-tools/zha_device_health.py` (device inventory, dead calibrated against
    living) and `local-tools/door_join_watch.py` (listener, 4-s cadence, changes only).
  * **`zha.permit` can last max. 254 s**; a `504 Gateway Timeout` does **not** mean that the window is
    closed. `last_seen` comes as an ISO string, not as a number.
  * **Still dead (permanent state):** HOBEIAN ZG-101ZL (370 d), `_TZ3000_zutizvyk TS0203` (372 d),
    ElektroHeizungKeller (150 d), Leuchte Ecke Wohnzimmer (80 d).
* **Two water sensors now have a fat Telegram alarm** (HA-native, same bot and
  same group as the battery alarms): `wasser_leck_heizung` (`binary_sensor.wassersensor_heizung`,
  HOBEIAN ZG-222Z) and `wasser_leck_waschmaschine` (`binary_sensor.tz3000_upgcbody_snzb_05`), plus
  `wasser_entwarnung` for both. Behaviour: immediately on wetness (5 s debounced), then **every 5
  minutes again, as long as wet** (max. 24 rounds = 2 h), and an all-clear when it dries out —
  the latter only after real wetness (`trigger.from_state == 'on'`), not on HA restart.
  Evidence: test trigger 17:58:34 UTC (heating) and 18:01:54 UTC (washing machine) — both times
  the timestamp of the group notify entity moved along one second later.
  **Pitfall:** `automation.trigger` **waits** for the end of the automation — a multi-hour
  automation thus runs into the timeout of the HTTP request. Check success therefore via `last_triggered`
  and the timestamp of the notify entity, not on the return value of the trigger.
  **Tools** (local, gitignored): `local-tools/wasser_alarm_bauen.py` creates or changes the three automations
  (idempotent), `local-tools/wasser_alarm_zeigen.py` renders the stored
  texts for checking. Both read the token from `~/.hermes/.env` and contain no secrets.
* **Aqara inventory (19 devices)** — 9 `lumi.weather` (climate) and 10 `lumi.sensor_magnet.aq2`
  (door/window); 18 alive, reported within 8–47 min, counters healthy. **The only dead one remains
  `Eingangstür`** (121 d, cell ordered). Two findings requiring action:
  * **`ToiletteTemp` and `TempSensorBad` have no battery entity** — ZHA does not know their
    manufacturer/model (their remaining entities are called `sensor.unk_manufacturer_unk_model_*`),
    therefore a battery was never created. **The battery alarm cannot see them**, their
    cells could die unnoticed. **Correction (25.09.):** the path first recommended, "in ZHA
    erneut interviewen" (interview again in ZHA), **does not exist** — the ZHA websocket API only knows
    `zha/devices/reconfigure`, and 16 calls over five minutes with a demonstrably awake device
    (whose `last_seen` and values advanced during the action) did not change the manufacturer.
    Manufacturer and model come from the Node Descriptor, which ZHA reads **when pairing**. The real
    path: **remove + re-pair** — evidenced in this own house, because `Fenster Sensor Toilette` carries
    the same dead entries and next to them correct entities. Price: new entity suffixes and dead entries;
    check beforehand what refers to it. **Still open.**
  * **Reference check before the re-pairing (25.09.): zero hits.** Scan across all automations,
    scenes, state attributes and **all 12 Lovelace boards** (Standard, Karte, Lampe, Mein Zuhause,
    Treppe-Alarm, Meine Energie, Garage SDM 360, Fenster/Türen, Klima, Heizung, CO2, Strom) — not a
    single one of the 12 entities (and neither of the two device_ids) is referenced anywhere. The
    re-pairing therefore cannot break anything. **YAML configuration is not readable via API** (template
    sensors, scripts, recorder exclusions) — that is not checked.
  * **Old IDs are recorded**, so that they can be assigned after the re-pairing:
    `local-tools/ids_vor_neukoppeln.json` (gitignored). Search terms: IEEE
    `00:15:8d:00:8b:ba:8c:27` (ToiletteTemp, device_id `a844a6db54c28aff22aa2a0677e72e21`) and
    `00:15:8d:00:8b:bd:8d:86` (TempSensorBad, device_id `c88b0d9a758cd7121af2fde02a3b5c8b`);
    entities `sensor.toilettetemp_{temperatur,luftfeuchtigkeit,druck}`,
    `sensor.tempsensorbad_{temperatur,luftfeuchtigkeit,druck}`, both `…_identifizieren` plus the
    `unk_manufacturer_unk_model_{rssi,lqi}` dead entries. **The old entities are NOT deleted** —
    so the history and long-term statistics of the two sensors remain.
  * **Re-pairing of both sensors successful (25.09. evening).** Both now have manufacturer `LUMI`
    and one **battery entity** each: `sensor.toilette_toilettetemp_batterie` = **55.5 %** (numeric,
    so the battery alarm counts it) and `sensor.bad_tempsensorbad_batterie` still `unknown`
    (fills on the next report). **The measurement entities kept their IDs**
    (`sensor.{toilettetemp,tempsensorbad}_{temperatur,luftfeuchtigkeit,druck}`) — the recorder hangs
    on the ID, so **history and long-term statistics continue without a break**; only the new
    battery entities are history-less. The reference check beforehand (zero hits) was thereby
    confirmed: there was nothing to repair. Before-after assignment:
    `local-tools/ids_vor_neukoppeln.json`.
    Open and purely cosmetic: seven `unk_manufacturer…` dead entries, the crooked battery IDs, and the
    two sensors are missing on the hand-maintained boards `Klima`/`Heizung`.
  * **These three cosmetic points were done the same evening:** battery entities renamed to
    `sensor.toilettetemp_batterie` (55.5 %) and `sensor.tempsensorbad_batterie` (still `unknown`);
    all seven `unk_manufacturer…` dead entries `hidden_by: user` (the one still active additionally
    disabled); on `dashboard-klima` and `dashboard-heizung` one card **"Bad & Toilette"**
    appended each (Klima 2→3, Heizung 5→6 cards, remaining content unchanged byte for byte). **Near-mistake
    in the process:** the new card was built before the renaming and pointed to the old battery IDs —
    noticed during the reference check against `/api/states` (2 unknown entities per board), corrected,
    afterwards 0 unknown references. The appearance itself is not verified (no HA login).
* **Window/door board extended (25.09.):** the view `fenster` of the board `fenster-turen`
  now has a **“Batteries”** section at the bottom — a heading card with `mdi:battery-40` plus the list
  of all ten unit batteries, in the **same room order** as the opening list above.
  Style follows `treppe-alarm/0` (there: heading “Batteries” + plain entity list). All ten
  units have a battery entity; `sensor.door_batterie` (entrance door) stands, as expected,
  at `unavailable` until the CR1632 arrives. 20 references checked, **none** unknown, the remaining
  view content unchanged byte-for-byte. Next cells by current reading: basement party room 66 %,
  toilet 69.5 %, balcony door 73 %. **Unverified:** whether a heading card outside of
  sections renders — the view uses no sections; if it does not appear, a
  card title replaces the heading.
* **Heating board: the “Delta flow–return” gauge wrapped (26.09.).** The tile reported
  “entity is non-numeric” because `sensor.heizung_differenz_vor_rucklauf` returns `unknown`: it
  is a template helper (config entry “Heizung: Differenz Vor- Rücklauf”, domain `template`,
  state `loaded`) over the two flow-monitor sensors, and the ESP `esp32_c3_web_a8dfa8` is
  **intentionally off** (no heating season). The sensor is therefore not faulty but honest — a
  value of 0 would be invented. **Fix:** the gauge now sits in a `conditional` card with two
  conditions (`state_not: unknown`, `state_not: unavailable`) and reappears by itself
  once heating resumes. **Evidence:** the ESPHome integration is healthy (the second ESP
  `esp_wroom_32_keller` delivers 3/3 values), all 18 references of the view valid, 6 cards before
  as after. Unverified: the rendering (no HA login). The card “Heating overview” deliberately still
  shows “not available” — it states why the difference is missing.
* **ESP test passed on 26.09. — and the Dach-CO₂ construction site is closed with it.** The user switched
  on the flow monitor **and** the Dach-CO₂ ESP. Evidence (from the own listener, accurate to the second):
  `wifi_status` of the flow monitor `OFFLINE → ONLINE` at **07:50:25**, first values **07:49:28** —
  flow 23.06 °C, return 22.19 °C, **difference 0.9 °C**. The prediction “near 0” was exactly right:
  the water largely stands still because there is no heating. **User correction (26.09.):** the
  flow monitor is attached to the **gas heating**, which has
  **nothing to do with** the Daikin heat pump `dach_ap22393`. Its 0 W compressor power is therefore **no**
  confirmation for the heating circuit but a separate plant — the assistant had conflated the two (both
  carry “Dach” in their name) and sold that as confirmation; **retracted**. The measurements
  themselves and the fix remain unaffected by it. The condition of the wrapped
  tile is thus met — **the user has to check the rendering**, that is the only remaining open item.
  The **Dach-CO₂ sensor**, `unavailable` since 19.09., was **not a fault** but the
  switched-off device: now 751 ppm / 44 % / 21.7 °C. All four ESPHome devices (Flow, WZ-CO₂,
  Dach-CO₂, Keller-CO₂) are thus healthy. **With that the item “Dach-CO₂ unavailable” is done.**
* **The silent TRV in Ralf's party basement clarified (26.09.) — it is a Tuya TS0601, not the SONOFF.**
  **Important, to avoid confusion:** the card **“Ralfs TRV”** on the heating board is
  **`climate.sonoff_trvzb_thermostat`** — `mode=heat`, `action=idle`, target **7.0 °C** (frost protection),
  actual **20.9 °C**. The device **is alive** and is a **different** one than the silent one. The two automations
  `trv_kellerparty_fenster_auf_heizung_zu` and `trv_kellerparty_fensterlogik_profi` are attached to the SONOFF.
  The **silent** device is the **Tuya TS0601** `Thermostat-Ralf-Keller-Party-Z`
  (`a4:c1:38:8e:bf:be:09:0e`), which the assistant initially followed; evidently the **predecessor**,
  superseded by the SONOFF. ZHA lists it as
  `available=False`, `last_seen = 20.02.2026 20:07 UTC`. The field is calibrated (living devices:
  socket Mascha minutes, button 07:41, window sensors 07:21/07:17) and therefore reliable.
  **Additional finding:** for the Tuya TS0601 (`a4:c1:38:8e:bf:be:09:0e`) there is **no
  `climate` entity** — only `rssi` and `lqi` (both `disabled_by: integration`) plus the
  firmware update entity. It is therefore currently **not controllable**, even if it came back.
  **Cause open:** empty battery, removed, or deactivated. The user makes clear: the
  basement lamps are **not routers** (and are attached to the wall switch, so are usually off) — the
  route explanation therefore does not hold, the light-switch test I proposed is moot.
  **What is decisive is the time order:** the TRV went silent on **20.02.**, i.e. **before** all other
  failures in this corner (`ElektroHeizungKeller` 02.05., entrance door 27.05., corner
  living room light 11.07.). The cause therefore lies with the **device itself**.
  The ZHA network is healthy (413 entities, 247 with a value, most recent report seconds old) — the silence
  is on the device side, not network-wide.
  **Silent devices, as of 26.09. (7+ days):** corner living room light 11.07. (77 d), entrance door
  27.05. (121 d, cell ordered), ElektroHeizungKeller 02.05. (147 d), Tuya TS0601
  (Ralf-Keller-Party-Z) 20.02. (217 d),
  HOBEIAN ZG-101ZL 14.06. (103 d) and 23.09.2025 (368 d), TS0203 21.09.2025 (369 d) — the last
  three are the previously deactivated dead entries. `Steckdose PC Lea` has been without
  contact since 23.09. — **intentionally: it is not connected** (user), so no fault. The earlier
  labelling of these two as “Router offline” (25.09.) is thus at least for Lea's socket
  **wrong**; whether `ElektroHeizungKeller` is a router is open.
  **Incidentally:** the four Daikin splits are named `climate.dach_ap22393`, `kevin_ap02845`,
  `lea_ap27941`, `wzr_ap86576` (all `off`) — only the SONOFF TRVZB on the heating board is `heat`.
  **Why the Tuya has no entities (checked on 26.09.):** its fingerprint
  `_TZE284_noixx2uz` does **not** occur in the **entire** quirks repo `zigpy/zha-device-handlers` (branch `dev`)
  — hence only rssi/lqi. **zigbee2mqtt** also does not have it in the device database,
  there three open “External Converter” requests are running (#29450, #30906, #31060). The community
  collection `dlnraja/com.tuya.zigbee` lists it as **`radiator_valve`** — so it is a
  radiator valve. **A viable path, if it is to be used:** the quirk `tuya/tuya_trv.py`
  already knows six `_TZE284` TRVs (`c6wv4xyo`, `ne4pikwm`, `o3x45p96`, `ogx8u5z6`, `p3dbf6qs`,
  `ymldrmzx`), all following the pattern `TuyaThermostat` + `MODELS_INFO` — a local copy with its
  fingerprint in `/config/zha_quirks/` would be a few lines, but the **data points** would have to be verified against the
  z2m converter threads. **Prerequisite in any case:** the device has
  not been on the network for 217 days and must be **re-paired**. Order: first re-pair (cheap,
  perhaps ZHA recognises more by now), then the quirk if needed, otherwise dead entry.
* **Local quirk for the Tuya valve written (26.09.) — `local-tools/ts0601_trv_noixx2uz.py`**
  (gitignored, **not** in the public repo; 84 lines, syntax checked). Structure:
  `TuyaQuirkBuilder("_TZE284_noixx2uz", "TS0601")` with the data points **2** (system_mode),
  **3** (running_state), **4** (setpoint, ×10), **5** (actual temperature, ×10), **7** (child lock),
  **36** (frost protection). These six are **doubly attested**: the z2m community converter for *exactly
  this* fingerprint (thread #29450) and the 16-fingerprint family block in
  `zhaquirks/tuya/tuya_trv.py` agree exactly on 2/3/4/5/7 (checked, not assumed).
  **Deliberately omitted:** calibration (family DP 47, converter DP 114) and fault/battery warning
  (family DP 35) — not attested for this device, otherwise risk of wrong values.
  **Installation (open, his hand):** file to `/config/zha_quirks/`, in `configuration.yaml`
  `zha:` → `custom_quirks_path: /config/zha_quirks/`, **HA restart**, **afterwards** re-pair the valve
  — the quirk must already be active at pairing. **Then check:** actual temperature against a
  known thermometer, write the setpoint and check on the device. The user has several
  radiators and uses valves for “window open → radiator closed”, so the path is worthwhile. **If it
  works, a PR to `zigpy/zha-device-handlers` would be the next step** (one line in the
  family, plus deleting the local copy).
* **Quirk demonstrably loaded (26.09.) — checked before pairing.** Evidence: `zhaquirks/__init__.py`
  sets `loaded = True` **only** in the `else` branch after a faultless `exec_module` (line 611) and then logs
  at 618 “Loaded custom quirks…”; in the system log (WebSocket `system_log/list` — **not**
  `/api/error_log`, that endpoint no longer exists, HTTP 404) this WARNING stands at **08:13:39**
  from today, and there is **no** entry “Unexpected exception importing custom quirk”. The file
  is therefore imported faultlessly. **This also proves that the `configuration.yaml` route
  holds:** the options of the ZHA config entry are `null`, the path came from there.
  Incidentally from the same log: the flow-monitor ESP has IP **192.168.178.28** (at 08:13:41 a
  one-off aioesphomeapi connection warning, restart race — afterwards it delivers); at 08:13:44 a
  template warning **`'batt_low' is undefined`** (somewhere a template references an undefined
  variable, not yet found — no file access to /config); plus Modbus
  unit warnings for `sensor.sdm630_*` (`VAR`, `kvarh`, empty `power_factor`) — cosmetic.
  **The method is secured as a skill reference:**
  `home-assistant-integration/references/unsupported-zigbee-device.md`.
  **Next step:** pair the valve, then cross-check actual temperature and setpoint.
* **The quirk works — Tuya valve is back on the network after 217 days (26.09., 08:19).** Hard evidence:
  ZHA reports **`quirk_applied: True`** and `quirk_class: 'zhaquirks.tuya.builder:(_TZE284_noixx2uz /
  TS0601)'`; the device is `available: True`, **`last_seen` 08:20:14** (seconds old), **LQI 156**,
  RSSI −61, `nwk` 61126 — so it is transmitting. **The same device identifier as before**
  (`5bc08943e7c3a0290c370097e41e7edf`) — **no duplicate entry**, the old name
  “Thermostat-Ralf-Keller-Party-Z” and the area `keller_party` were preserved.
  **The quirk has created exactly the promised entities:**
  `climate.thermostat_ralf_keller_party` (modes off/heat, 5–30 °C),
  `switch.…_frostschutz` (**DP 36**), `switch.…_kindersicherung` (**DP 7**), plus
  `sensor.…_hlk_aktion`, `…_pi_warmebedarf`, `…_quelle_der_sollwertanderung`, `…_zeitstempel`
  (the extra entities of the family `TuyaThermostatV2` — that too is proof that the right
  class was loaded). **Still open:** the values stand at `unavailable`
  (`soll=None`, `ist=None`), because a battery device must first wake up — waking by button press
  or a setpoint write (ZHA queues commands for sleeping devices).
  **Final check step:** actual temperature against a known thermometer, write the setpoint and
  check **on the device**. Only then is the data-point mapping verified.
* **Interim status valve (26.09., ~08:35): entity hangs, reloading ZHA is the next step.**
  The quirk took effect (see above), but the entities never came up: `climate.thermostat_ralf_keller_party`
  remained `unavailable` (`soll/ist=None`), likewise `sensor.…_hlk_aktion`; the other sensors `unknown`.
  The creation history shows the reason: `unknown` **08:19:14** → `unavailable` **08:19:27** — the
  entity was created while the device was not yet ready (known Tuya/ZHA pattern).
  The device itself is healthy: `available: True`, `last_seen` advances (08:21:24), LQI 156.
  **Tried:** (1) `homeassistant.update_entity` on all entities — brought the sensors from
  `unavailable` to `unknown`, but **no data points**: the `off` values of the two switches may
  be defaults, a device report is **not** proven. (2) Deactivating/reactivating via
  the registry (`disabled_by: user` → `None`) — result: the entities are *enabled* again,
  but only appear after a **reload of the ZHA integration**. That is now the pending
  step and waits for his okay (Zigbee drops out for ~30 s during this, alarms and buttons are blind
  for that time). The user has set **20 °C on the valve**; in HA that is not yet visible — as soon as
  the entity lives, **20.0** must stand there (that is the DP-4 test from the device side).
  **Near-accident, documented:** A generated script accidentally contained a
  `homeassistant.turn_off` call **without a target** — in HA that switches off *all* switchable devices.
  Noticed and removed before execution. Rule now in the skill reference
  `home-assistant-integration/references/service-call-safety.md`.
* **Valve: what the *working* community converter says (26.09.).** Thread #29450 contains four
  versions; the **last** (comment 04.02.) is the one with which the author reports “already great results” —
  he has **eight** of these valves. Their data points: **2 preset** (auto/manual/leave),
  **3 running_state**, **4 setpoint ÷10**, **5 actual temperature** (signed, ÷10),
  **6 battery**, **7 child lock** (`LOCK:false, UNLOCK:true` — **inverted**),
  **28–34 weekly schedule**. **Decisive:** DP 4 and 5 match my quirk mapping —
  the **core mapping is right**. Differences: DP 2 is *preset* there instead of system_mode (the
  values 0/1/2 fit together), DP 6 (battery) and the weekly schedule are missing from mine.
  **The author describes exactly our symptom:** the devices go “every two weeks” into a
  **calibration loop**, and **only removing the battery** brings them back. That fits the
  blinking **“CL”** on his display — child lock *or* calibration, both conceivable.
  **Status:** the valve transmits (`last_seen` seconds old, LQI 160) but delivers **no** values;
  the system log shows **no** warning about unknown data points. Tried without effect:
  `homeassistant.update_entity`, deactivating/reactivating the entities, a read command on the
  standard thermostat cluster 0x0201 (timeout, then HTTP 500). Listener runs until ~08:30.
  **Next steps:** (a) wait and see whether the lock/calibration state resolves; (b) otherwise
  evaluate the debug log (re-enable debug via `logger.set_level`, then search for `Tuya`
  — I cannot reach `/config`, the user must do that); (c) extend the quirk with DP 6 and the
  weekly schedule. **Debug is off again** (reset to `warning`).
* **Valve: a whole hour of observation, not a single value (26.09., 06:30–07:30).** The listener
  (20 s interval) logged **not a single state change**; `soll`/`ist` remained throughout
  `None`. The device was throughout healthy and even got better: `last_seen` in each case
  seconds old, **LQI 160 → 176**, **RSSI −60 → −56** (excellent link). **Important correction
  (user rightly pressed):** it does **not** follow from this that “no data arrive”. There are
  two different faults — **(A)** the device sends nothing at all (only network packets), **(B)** it sends
  data but with **different data-point numbers** than in my quirk, or **(C)** the right
  data arrive and the entity does not process them. `last_seen` proves only **reachability**,
  not data transfer — a sleeping device keeps it fresh with pure radio polls. And my earlier
  reasoning (“no entry in the system log”) was **worthless**: in the source
  (`zhaquirks/tuya/__init__.py`) an unknown Tuya frame is logged with **`_LOGGER.debug`**
  (“Unrecognised command: %x”), **not** as a warning — in the system log it therefore
  cannot appear at all. **The debug listening continues** (`zigpy.zcl` and `zhaquirks.tuya` at
  `debug`, `homeassistant.components.zha` back to `warning`), so that the log stays clear;
  **then reset everything to `warning`**.
  **Open steps:** (1) user looks under Settings → System → Logs for `Tuya` and
  sends a screenshot: lines present → adjust the mapping; no lines → cleanly re-pair.
  (2) Open question: does “CL” still stand on the display (child lock; clear with **+ and −** together).
* **Valve: the day's finding in brief (26.09., until ~10:40).** The device is **reachable and
  reports in**, but delivers **no data** and accepts **none**. Evidence:
  - From the log (search for the IEEE) there are **four** lines, all at **26.09. 08:31:41 and
    08:32:02**: `Device 0xeec6 (a4:c1:38:8e:bf:be:09:0e) joined the network` — the **join after
    battery change/restart**, after that **nothing**. Important: these are **INFO** lines; zigpy does
    not log pure radio polls at all, so "nothing further" only means: **no ZCL traffic**.
  - **`last_seen` moves in 15-minute steps** (10:19:30 → 10:34:21) — that is the valve's polling
    interval. So it is awake and polling.
  - **Two write commands from HA with no effect at all:** 10:19:30 `set_temperature 11.5` and
    ~10:35 `set_temperature 12.5` — entities stayed `unknown`, 15-minute listener without a single
    change, display kept showing 15. **So `last_seen` moves while nothing arrives and nothing gets
    through.**
  - User reports: display shows **15** (presumably the setpoint; he had earlier set 11) and earlier
    **"CL"** (child lock) — the converter's author describes exactly such states.
  **Important lesson for the search:** zigpy names devices in debug lines by the **short address**
  (`0xeec6`), not by the full IEEE — that is why the IEEE search found only the four lines.
  **Open:** search for `eec6` and `0xef00` around 10:19/10:35 in the log (shows whether the command
  goes out at all and whether a reply comes back). Debug is still set to `debug` for `zigpy.zcl` and
  `zhaquirks.tuya` — **reset it afterwards**.
* **BREAKTHROUGH: the cause is found (26.09., ~10:45) — the valve needs the "data query" spell.**
  The log sent by the user shows for `0xEEC6` (nwk 61126 = the valve) **over the whole period only a
  single kind of traffic**: every ~15 minutes a `set_time_request` (Tuya time sync), which zigpy
  answers with `DefaultResponse(SUCCESS)` — **nothing else**, **not a single datapoint report**.
  In addition: **not a single write attempt** for my two `set_temperature` calls (10:19:30 / ~10:35)
  → **my commands did not even go out**; the **15** on the display is therefore **not** from me.
  **The mechanism** (in `zhaquirks/tuya/__init__.py`): class `BaseEnchantedDevice` —
  ``tuya_spell_data_query: bool = False  # additional spell needed for some devices to send data``.
  The spell (`spell_data_query()` → `tuya_cluster.command(TUYA_QUERY_DATA)`, **0x03**) is cast **once
  at device configuration**. In the modern builder `.tuya_enchantment(data_query_spell=True)` switches
  it on (creates `EnchantedDeviceV2(CustomZigpyDevice, BaseEnchantedDevice)`); **the default is
  `False`** — and **neither the 16-family nor my quirk ever switched it on**.
  **Fix implemented:** `local-tools/ts0601_trv_noixx2uz.py` now contains `.tuya_enchantment(data_query_spell=True)`
  (89 lines, syntax checked, still gitignored).
  **Next steps for the user:** replace the file → **restart HA** → if still nothing comes, **re-pair**
  the valve (the spell runs at configuration). **Proof in the log:** the debug line `Executing data
  query spell on Tuya device a4:c1:38:8e:bf:be:09:0e` — it contains the **IEEE** and is therefore
  directly searchable. A manual attempt to send `0x03` via `zha.issue_zigbee_cluster_command` ran into
  an HTTP 504 (service waits for a reply, device asleep) — the command may be stuck in the queue.
* **Correction and new order (26.09., ~10:55) — the spell is cast only ONCE.** The restart at 10:47
  came to nothing: the entities were indeed rebuilt (10:47:48), but the valve was **asleep** during it
  (`last_seen` 10:34 → 10:49), LQI/rssi initially empty. The magic spell is a **command to the device**
  and is cast **once at configuration** — it is **not repeated**. Just like the manual attempt, which
  ran into a **timeout**. **Conclusion: the restart fundamentally cannot achieve this if the device is
  asleep during it. During *pairing* the device is awake — that is where the spell is delivered.** So:
  replace the file → **remove the device and re-pair** (not just restart).
  **Additionally corrected:** in my version `.tuya_enchantment(data_query_spell=True)` stood *very
  early* in the chain; all existing Tuya quirks (e.g. `tuya_trv.py`, `tuya_sensor.py`, `ty0201.py`)
  set it **shortly before `skip_configuration()`/`add_to_registry()`**. After examining the builder,
  `device_class()` is only a simple setter (no reset) — so the early position was **presumably
  harmless**, but the convention is now followed (93 lines, syntax checked, still gitignored).
  Documented explicitly as a **precaution**, not as a proven cause. According to the source,
  `skip_configuration` affects only the **reporting configuration**, not the magic spells.
* **CONFIRMED in the log of 26.09. (10:55) — the call point was the error.** The user's file shows for
  the valve (now short address **0xAC57**, IEEE unchanged → rejoined):
  ```
  10:54:32  [zha.zigbee.device] [0xAC57](TS0601): started configuration
  10:54:32  [zha.zigbee.device] [0xAC57](TS0601): applying quirks custom device configuration
  10:54:32  [zigpy.device] [0xac57] Executing attribute read spell on Tuya device a4:c1:38:8e:bf:be:09:0e
  10:54:32  [0xAC57:1:0x0000] Sending request: Read_Attributes(attribute_ids=[4, 0, 1, 5, 7, 65534])
  ```
  (twice, 10:54:32 and 10:54:46). **So the attribute-read spell fired — with exactly the six attributes
  the test requires — but `Executing data query spell` is completely missing.** This proves: the device
  class **was** enchanted, but had the **defaults** (`read_attr_spell=True`, `data_query_spell=False`).
  **So my `.tuya_enchantment(data_query_spell=True)` at the first position in the chain did not take
  effect** — the suspicion about the position was right, not merely caution.
  **Second, important finding: device configuration does NOT run only during pairing.** It ran at
  **10:54** (after the restart at 10:47), as soon as the valve was reachable, and ZHA applies
  `applying quirks custom device configuration` in the process. **Consequence: with the corrected file
  a restart suffices — the configuration (and thus the data query) follows once the valve is awake.
  Re-pairing is no longer mandatory.**
  **Proof for the next run:** the line `Executing data query spell on Tuya device
  `a4:c1:38:8e:bf:be:09:0e` must then appear in the log.
* **IT WORKS (26.09., 11:03–11:06) — the data-query spell was the solution.** With the corrected file
  + restart the entities filled in:
  ```
  11:02:59  state=unknown   soll=None  ist=22.0            (start of the values)
  11:03:00  state=heat_cool soll=None  ist=22.0  action=heating
  11:05:24  state=heat_cool soll=None  ist=25.0  action=heating
  ```
  **User's counter-test:** he **breathed on** the valve → display and HA both show the new value
  (22 → 25). That proves **datapoint 5 (actual temperature) including conversion**. Further running
  entities: `kindersicherung` = off, `frostschutz` = off, `hlk_aktion` = heating.
  **Still open: the setpoint (`soll = None`).** A write command from HA (11:07, `set_temperature
  17.0`, service reports ok) had not yet arrived after 90 s — expectable, the device is asleep and
  accepts commands at the next awake moment. **To check: whether the display then briefly shows 17.**
  **Finding from the display photo:** next to "24" a **key symbol** is visible — **the child lock is
  active**. **CORRECTED (11:11 — measured, not inferred): the mapping is NOT inverted.** The history
  of the entity shows `on` at **11:03:00** — exactly when the key symbol was on the display, and as a
  **device-reported** change (not as a written value). So my quirk maps DP 7 **correctly**; `on` =
  locked. My earlier note ("DP 7 is inverted, minor correction needed") was a **false inference from
  the z2m converter** — its `lookup({LOCK: false, UNLOCK: true})` does not apply to every device of
  this family. Lesson: **always** check the polarity **on the device's display**, never on the
  converter. Assistant's error: at 11:08:43 it sent `switch.turn_on` and thereby **locked** the valve
  instead of unlocking it.
  After the user switched to `off` (11:11:14) still open: **does the display still show the key, and
  does a turned setpoint travel to HA?**
  Likewise on the display: radio symbol (connected), hand and clock symbol (manual/time-programme
  indicator).
  **The case is thus essentially solved**: fingerprint quirk + **`tuya_enchantment(data_query_spell=True)`**
  at the correct position in the chain is the proven solution for this valve.
* **FULLY SOLVED (26.09., 11:28–11:30) — the operating mode was the last error.** After the mode fix
  (datapoint 2 no longer on `SystemMode.Auto`, but **fixed on `Heat`**) everything ran:
  ```
  11:24:04  Restart (connection gone)
  11:24:37  climate=heat_cool  soll=None   ← old file still active
  11:28:22  climate=heat       soll=19.5   ← ★ fix takes effect: mode heat, setpoint VISIBLE
  11:28:37  climate=heat       soll=14.5   ← setpoint moves (device turn)
  11:29:34  Write test from HA: set_temperature 18.0 -> ok
  11:29:49  climate=heat       soll=18.0   ← ★ write command ARRIVED and stayed
  ```
  **Diagnosis correction to my own approach:** the repeated "soll = None" was a **measurement error**.
  Only `attributes.temperature` was read; in the operating mode `heat_cool` (automatic) this field is
  **always empty** for a range entity, the value lives in `target_temp_low`. So the setpoint was
  **never lost** — merely invisible. **Rule: for a `climate` entity always also read
  `target_temp_low`/`target_temp_high`, not just `temperature`.**
  Evidence from the user's ZHA diagnostics file (`...TZE284_noixx2uz_TS0601_5bc08943e.json`):
  * `occupied_heating_setpoint = 1700` in the thermostat cluster = **17.0 °C**, so the real setpoint.
  * `ctrl_sequence_of_oper = 2` = **heating only** — the "Auto" mapping contradicted the device.
  * `child_lock` = cluster attribute **`0xef07`** → **datapoint 7** (confirmed, `inverted: false`).
  * `frost_protection` = cluster attribute **`0xef24`** → **datapoint 36** (confirmed).
  **Mnemonic for future guessed datapoints: attribute = `0xEF00` + datapoint number.**
  Permanently empty entities (device does not send these DPs): `pi_heating_demand`,
  `setpoint_change_source`, `setpoint_change_source_timestamp` — cosmetic only.
* **SELF-FOUND MAPPING ERROR (26.09., 11:48) — `running_state` was inverted.** The user asked why the
  entity reports "heating mode" when setpoint is 11.4 and actual is 23 — a correct question, it
  uncovered an error. History as proof:
  ```
  11:40:36  soll=35.0  ist=22.0  action=idle      ← 35 above 22, should be HEATING
  11:46:06  soll=11.4  ist=23.0  action=heating   ← 11.4 below 23, should be IDLE
  ```
  Cause: **the sibling family `tuya_trv.py` contains BOTH variants for datapoint 3** — two fingerprints
  use `Heat_State_On if x`, two use `if not x`. The **inverted** one was copied; correct is `if x`
  (confirmed by the observed correlation). Changed in `local-tools/ts0601_trv_noixx2uz.py` (line 46,
  one word).
  **Important:** `hvac_action` is **not** calculated by HA, but is the valve state reported by the
  device (DP 3). HA therefore cannot go to idle "by itself" — it shows what the valve claims.
  **Lesson (as rule 10 in the skill reference):** never adopt from a family the variant that happens
  to be in view, but check against the **observed correlation**.
* **BREAKTHROUGH ON THE WRITE DIRECTION (26.09., 11:44–11:49) — the child lock blocks external write
  commands.** The user put it himself: "die Übertragung TRV → HA klappt super, aber andersrum scheint
  es zu stocken" (the transfer TRV → HA works great, but the other way round seems to stall) and
  "ich MUSS erst Kindersicherung ausmachen und dann drehen" (I MUST first turn the child lock off and
  then turn). A write test **with the valve unlocked** held:
  ```
  11:44:38  before SOLL=35.0
  set_temperature 11.4 -> ok
  +20s … +80s   SOLL=11.4  action=idle
  +100s … +240s SOLL=11.4  action=heating   ← 4 minutes stable, NO fallback
  ```
  Previously (with the valve locked) every written value tipped back to the device value after ~70 s.
  The pattern "value appears in HA, disappears on the next device report" is therefore the **lock**,
  not a wrong address. **But:** the separation is not yet clean, because it is not logged whether the
  valve was locked at the moment of each failed attempt. **Cleaner test:** user unlocks on the device,
  does **not** touch it again, then writes; afterwards additionally check the `child_lock` switch from
  HA and on the display whether **HA can unlock at all** (after the `turn_on` at 11:08:43 he still had
  to physically press and hold). **That is the key question for the window automation**: if HA can
  only lock but not unlock, the sequence "unlock → value → lock" needs another route.
  Side finding from the same test: the valve motor needs ~100 s until `running_state` flips from
  "heating" to "idle" — the delay is mechanics, not a fault.
* **TIME WINDOW CONFIRMED (26.09., 11:52–12:00) — write commands only take effect shortly after
  unlocking.** Two write commands four minutes apart, no intervention on the device in between:
  ```
  11:52  set_temperature 11.4   ->  120 s observed, value held (valve reported nothing)
  11:56:35  set_temperature 18.0 ->  +0…+20 s SOLL=18.0, then +30 s SOLL=11.4 and it stayed there
                                     (170 s observed, no further change)
  ```
  **The fall-back to 11.4 is the key:** the device fell back to the value it had last **accepted**.
  This proves: **11.4 arrived** (no HA echo — the value came back *after* the 18.0), **18.0 was
  rejected**. The window was already closed between 11:52 and 11:56. **Signature for future tests:**
  a fall-back names the last successfully written value — not a random value.
  **And:** the `switch.…_kindersicherung` indicator read `off` during the whole test, although the
  valve was locking. **The switch says nothing about the real lock.**
  **Open key question:** can Home Assistant unlock the valve at all (switch), or only the long press
  on the device? Not answered. Own result: my `switch.turn_on` at 11:08:43 **locked** the valve (did
  not unlock it), the user still had to physically press and hold.
  **Next step (user):** manual — is there a **permanent mode** for the child lock? Without a permanent
  mode, "window open → heating off" cannot be automated via this route.
* **CORRECTION IS LIVE (2026-09-26, ~11:57) — proven by a change at the *same* data point.**
  The broad listener shows: before the rollout, at `soll=35.0 / ist=22.0` the action was `idle`,
  after the restart (11:52:53), at `soll=11.4 / ist=23.0` the action is `heating` — **the reverse of
  the old mapping for the same data-point meaning**. The entity now therefore interprets the device
  value differently, i.e. **the corrected file is loaded and the device configuration has run through**.
  No further restart needed.
  Current device state: `running_state` = „Ventil offen" (`heating`), although the setpoint 11.4
  lies below the room temperature 23. Two possible explanations — valve still mechanically open, or
  the device reports its own state independently of the setpoint. Only checkable at the radiator.
  Incidental finding: the entity `sensor.…_hlk_aktion` changes synchronously with `hvac_action` — both
  map the same data point 3.
* **CHILD LOCK SOLVED — permanently off (2026-09-26, ~12:05, via manual).** The user found the
  **permanent mode** in the manual: **CL is now permanently off, turning works immediately.** That
  removes the blocker for "window open → heating off" — write commands should now take effect at any
  time.
  **But immediately afterwards the *other* direction jams:** the user sets **20 °C** on the device and
  **HA keeps showing 11.4**. The entity was last updated at **11:53:17** — on the restart or the
  device configuration — and has since reported **nothing for 13 minutes**, although the device
  was turned. Before the restart all turns arrived within seconds.
  **To investigate:** is the device no longer sending (long sleep after the configuration? does the
  spell work only once?), or is it sending and HA not accepting it (mapping after the change)?
  Observation is running; if it stays silent, the log settles the question (`0xef00` lines).
  **Important for the automation:** the *write direction* is now free, but the *read direction* must
  be reliable, otherwise the controller regulates on stale values.
* **BATTERY DATA POINT + RE-PAIRING (2026-09-26, ~12:15–12:25).** In the log, **data point 6 = 43**
  appeared as the **only real measured value** of the valve (battery level). The quirk was extended
  with `.tuya_battery(dp_id=6)`; the file now maps **seven** data points:
  **2, 3, 4, 5, 6, 7, 36** (syntax checked, chain `adds` → `tuya_enchantment` → `skip_configuration`
  → `add_to_registry` correct).
  **The user's planned sequence (deliberately in this order):** replace the file → **HA restart**
  (so the quirk is loaded) → **remove** the valve in ZHA → **re-pair**.
  **Purpose:** not the quirk (that has long been in), but the **device configuration** — and with it
  the **data-retrieval spell**, which has not been cast since the restart around 11:53.
  **Expectation after re-pairing:** setpoint (12/13), actual ≈ 23 °C, mode `heat`, action `idle`,
  battery 43 %.
  **Still open** is the user's 0.5-step theory (the device works in half degrees; 11.4 could
  be invalid). **Neither confirmed nor refuted** — as long as the valve stays silent, a
  "standing" value can prove nothing (rule 11). The test succeeds only once the device reports again.
  **Incidental finding:** the 13.0 write command held for 150 s without reverting — with a **silent**
  device that is **no** proof of success.
  **Note:** HA log files are named in **UTC** (`10-09-17` = 12:09 Berlin).
* **✅ SOLVED (2026-09-26, 12:27) — re-pairing cast the spell again, everything works.**
  Inventory **after** the Re-Pair:
  ```
  climate:  modus=heat  SOLL=5.0 → (write command 11.4) → device shows 11  ist=24.0  action=idle  sperre=off
  new:      sensor.…_z_batterie = 100.0   (the added data point 6)
  ```
  **The user confirmed at the display that the value arrived** ("du hast es auf 11 gestellt bekommen,
  war vorher 5" — you managed to set it to 11, it was 5 before). This proves the **write direction**
  *at the device* for the first time — not merely as a display in HA.
  **His 0.5-step theory is CONFIRMED:** written 11.4 → the device shows **11** and rounds
  to its steps. Odd values are therefore **not** wrong, they are rounded.
  **And the `running_state` correction works:** with SOLL 11 < Ist 24 the action reports **`idle`** —
  previously the inverted mapping would have said `heating`. Exactly the logic the user had
  demanded.
  **Complete final state:** mode `heat`, setpoint/battery/actual are transmitted, writing
  works, child lock permanently off, battery 100 %.
  **Path for the window automation:** a lock switch is **not** necessary — it is enough to write a
  low setpoint (e.g. 5 °C) on window opening. That is now evidenced.
  **What the re-pairing achieved:** the file had long been correct; all that was missing was the
  **device configuration**, which casts the data-retrieval spell. Note: **An HA restart does not
  trigger it, a Re-Pair does.**
* **✅ COMPLETE (2026-09-26, 12:34) — half degrees confirmed, child lock does not disturb the radio path.**
  The user checked at the display: **`set_temperature 19.5` → display shows `19.5`**.
  * **The device works in *half* degrees.** Written 11.4 → the display showed **11** (odd values
    are *rounded*), written 19.5 → the display shows **19.5**. My earlier claim
    "ganze Grad" (whole degrees) was concluded from *one* rounded value — too little.
  * **The child lock blocks only the *buttons* on the housing, not the radio path.** With an *active*
    lock (key visible in the display), 14.0 and 19.5 each held for minutes, and 19.5 arrived
    **at the display**. That is exactly the desired behaviour: **the child cannot get at it, HA can.**
    My earlier deduction ("die Sperre verwirft Schreibbefehle" — the lock discards write commands)
    relied on reverts that can also come from the *device report* — **not reliable**.
  * **Important for repeating:** a **Re-Pair switches the child lock back on**
    (in the listener around **12:24:39**). After every re-pairing the permanent mode must be set again.
  * **The HA switch `…_kindersicherung` does not show the real state** — it stood continuously at
    `off` while the device was locked. Only the display is authoritative.
  **Final state:** mode `heat`, setpoint writable and readable (half degrees), actual temperature,
  valve state correct, battery %, child lock on and still remote-controlled. Case closed.
* **BEDROOM WINDOW AUTOMATION BUILT AND BOTH DIRECTIONS EVIDENCED (2026-09-26, 12:47–12:52).**
  The new valve is intended for the **Schlafzimmer Eltern** (parents' bedroom) and was moved there.
  * **Window contact:** `binary_sensor.aqarasensorsztur_offnung` („Schlafzimmer Fenster groß").
    The user expressly confirmed it.
  * **New automation:** `automation.schlafzimmer_trv_fenster_auf_zu` (id `1790419671862`),
    trigger window `on`/`off` **each 15 s**, actions `climate.set_temperature` 8 and 20 respectively.
  * **Measured runs:** 12:48:58 open → 12:49:13 fired → SOLL 8.0; 12:51:27 closed → 12:51:42
    fired → SOLL 20.0. **Each exactly 15 s after the state change.**
  * **Settling time works:** fast open-close-open-close (12:48:49–12:48:52) triggered **nothing**,
    likewise a 2-second opening at 12:46. Intended: protects motor and battery.
  * **NOT TOUCHED:** `automation.trv_kellerparty_fenster_auf_heizung_zu` (id `1772474316617`,
    trigger `binary_sensor.fenstersensorkellerparty_offnung`, controls `climate.sonoff_trvzb_thermostat`)
    — different room, different window, different valve. Follow-up check: unchanged, still controls SONOFF.
  * **Renaming (WebSocket API, the REST API returns 404 for the registries):**
    device `5bc08943e7c3a0290c370097e41e7edf` → `name_by_user` „Schlafzimmer Eltern", area
    `schlafzimmer`. Entity IDs migrated: `climate.schlafzimmer_eltern_thermostat`,
    `switch.schlafzimmer_eltern_kindersicherung`, `…_frostschutz`, `sensor.…_batterie`,
    `…_hlk_aktion`, `update.…_firmware`, `sensor.…_rssi`, `…_lqi`.
    **The display names follow the device name automatically** (the entities had `name = None`).
    **Checked beforehand:** no dashboard named the valve, only its own automation hung on it (adjusted).
    **Note:** a device rename changes all displays at once, the entity IDs must be
    carried along individually (`config/entity_registry/update`, `new_entity_id`).
* **SOLAREDGE: NEW 6-MINUTE ERROR SINCE ~2026-09-25 — NOT CAUSED ON OUR SIDE (2026-09-27).**
  The user saw errors in the **charging app**; his observation "die Tage davor gab es diesen Rhythmus
  nicht" (the days before, this rhythm did not exist) is **evidenced**. Evaluation of the logs:
  ```
  site read failed per day:  2026-09-25 11   2026-09-26 21   2026-09-27 16
  (before that only isolated outliers: 2026-09-12 4 · 2026-09-17 12 · 2026-09-20–24 1–4)
  Intervals of the last 12 errors: 24, 24, 24, 24, 24, 24, 24, 24, 24, 24, 24, 24 minutes
  Intervals at the start:          0, 0, 4, 6917, 0, 0, 0, 0, 1, 0, 4, 20 minutes  (completely irregular)
  ```
  **Out of isolated dropouts, a cadence emerged around 2026-09-25.** The error always hits
  `read 32@40071` (inverter block, model 101) — **never** the meter. The app polls every 30 s, so
  **every twelfth** attempt fails; ~0.33 % of all queries.
  **A 30-minute pause of the proxy (2026-09-26 22:06–22:36) changed NOTHING** — afterwards exactly the
  same 6-minute cadence. **That rules out overheating/overload.**
  **Software ruled out:** nothing was changed in the project around 2026-09-25/26 (the commits of
  those days are all valve documentation); the proxy configuration last on **2026-09-20 06:04**; the
  polling interval stands unchanged at `interval_s: 30` (configuration and example identical).
  **So device-side.** Open: the inverter's firmware version, indications in the SE portal around
  2026-09-25.
  **No pressure to act:** the app copes with it (it reconnects 30 s later and keeps running), the
  meter is not affected, the proxy still writes nothing (`upstream_writes: 0`).
  Log line for the installer: since 2026-09-25 the inverter delivers the block `32@40071`
  truncated every 6 minutes (`failed to fill whole buffer`), without a load peak, unchanged after a
  30 min pause.
* **SOLAREDGE FIRMWARE UPDATE — DATE FOUND: 2026-09-16/17 (2026-09-27).** The user is the
  installer and knew that the inverter received an update ("aber nicht gestern oder so" — but not
  yesterday or so), and referred to STATE.md or the session histories. Find in the sessions:
  **"The SolarEdge Modbus path is now HEALTHY again (firmware update + Modbus toggle fixed it) after
  the 2026-09-16/17 wedge"** (`@session:default/20260913_065549_918f4a`) and, a day later,
  **"inverter map unchanged after the firmware updates, meter block decodes cleanly"**
  (`@session:default/20260917_075117_cf75a1`).
  ```
  2026-09-16/17  Firmware update + Modbus toggle  (solved the fault, HEALTHY again)
  2026-09-19     Firmware read read-only: 0004.0025.0015   (model SE5000H-RWS00BNO4, SN 740745CE)
  2026-09-25     The 6-minute error cadence begins          <-- 8 days LATER
  2026-09-27     Firmware unchanged: 0004.0025.0015  (one query via the proxy, ~50 ms)
  ```
  **Result: the update is not the cause.** It lies eight days before the start of the cadence, and
  the version is identical to this day — so there was also **no second update** in between.
  Deliberately left open: a late-acting firmware effect cannot be entirely excluded, eight days
  would be unusual for that, however. **The SunSpec identification block (40004) contains no
  update date** — the date appears only in SetApp/portal or in the session histories.
  Note: `session_search` returns hits only for **simple** search terms; long AND chains
  (five terms) come back empty, a single word ("Firmware") finds the spot immediately.
* **PV METER PUZZLE SOLVED (2026-09-27, 15:26): the charging app calculates CORRECTLY — they are two different quantities.**
  The user compared "PV produced today (counter)" (13.59 kWh) with the SE app (19.9 kWh) and
  suspected an error. The SE app shows the day's **energy balance**:
  ```
  Production             19.9 kWh
    Into house            2.08 kWh (11 %)
    To battery            6.65 kWh (33 %)
    To grid              11.2  kWh (56 %)
  Consumption             4.45 kWh
  ```
  The inverter meter counts only what the inverter **delivered**:
  **2.08 + 11.2 = 13.28 kWh ≈ 13.59 kWh (charging app)** — matches to within 0.3 kWh.
  The **6.65 kWh that went into the battery are not yet discharged** (battery 99 %,
  battery mode „Time of Use", "Into house 0 kW"). They come back as output energy once the
  battery discharges — exactly what the tooltip says ("sie zählt die spätere Akku-Entladung mit" —
  it counts the later battery discharge too).
  **Only the name needs correcting:** "PV produced today (counter)" is **not** the PV generation,
  but the **output** of the inverter. The tooltip compares it with the monitoring app —
  that is misleading once a battery is in play.
  **Still open: the Deye is missing.** Garage today 2.7 kWh (SDM meter); the app reads it
  (`garage.pv_w` 849 W, `sdm.pv_energy_kwh`, `sdm.last.pv_kwh`) and does **not** add it to the
  PV meter.
  **Lifetime remains different:** app 29.36 MWh against SE app 66.5 MWh. At 6.59 MWh/year
  that is 4.5 against 10.1 years — open whether an earlier inverter is in the SE sum.
  **Note:** in a plant with a battery, "Production" (PV generation) ≠ "Output of the
  inverter". The difference is the battery charge and recovers later. A single
  day figure without this split is **no** proof of an error — the SE app shows the split under
  "energy balance".
  (The previous interpretation "die App zeigt zu wenig / Register falsch" (the app shows too little /
  register wrong) is thereby **withdrawn**.)
* **FIRST FULL DAY WITH THE NEW FACTOR (completed 2026-09-27 24:00 Berlin, read 2026-09-28 early).**
  ```
  2026-09-27  Forecast whole day               25.913 kWh
              Output, integral                 18.687 kWh
              Output, inverter meter           21.152 kWh
              Generation (pv_kwh, new)         22.882 kWh  = output + battery +7.983 − 3.789
              Factor new (against generation)  0.883
              Factor old (against output)      0.721      <- what the factor would have said before
              Factor array (DC)                0.73
  ```
  The **meter is the better source**: 21.152 against 18.687 from the integral — the latter loses
  every minute in which the service does not run (here 2.5 kWh).
  **At night the meter keeps running**, because the battery supplies the house *through* the
  inverter: at 05:27 there were 2.18 kWh output at SOC 35.9 % and without sun. Without the battery
  terms `pv_kwh` would have become **negative** there (−0.12) → **now limited to 0** (generation
  cannot be negative; `factor()` returns None for a flat day anyway, so the limit cannot
  make the factor look better).
  **Column rebuild proven:** "header rewritten to 39 columns, 5 old row(s) mapped by name" —
  `measured_pv_kwh` stands as **column 8** in the file, without shifting the old rows.
  **6-minute error unchanged:** 190 hits in the proxy log, still exactly every 6 minutes
  (03:16, 03:22). The pause and the app changes have changed nothing about it.
* **FORECAST FACTOR: THE BATTERY CHARGE WAS MISSING (27.09., 16:00) — the most important finding of the day.**
  Owner's note, verbatim: *„Wenn wir dabei Einspeichern in den Akku nicht berücksichtigen,
  dann wird unser Forecast bis 24:00 immer falsch sein"* (If we do not take charging into the battery into account, then our forecast until 24:00 will always be wrong). **He is right, and measurably so.**
  The factor compared **PV generation** (forecast) with the **inverter output** (AC integral).
  The difference is the battery charge — it is generated, but not delivered.
  ```
                       before    after
  measured              12.19  →   20.55 kWh    output + battery charge
  expected until now    19.93      22.44 kWh
  factor                 0.61  →    0.916       instead of 40 % yield shortfall
  rest of day            3.81  →    3.40 kWh    (corrected)
  ```
  **Cause in the code:** the comment at the integration claimed that at the AC node the
  battery needed "no separate term" — that holds for **house/car**, but **not** for the
  forecast comparison. At night the error flips into the opposite: the meter keeps rising while the
  battery discharges.
  **Implementation** (`ha-app/evcharge/main.py`, gitignored):
  * new day field **`pv_kwh` = `ac_kwh` + `charge_kwh` − `discharge_kwh`** (conservation law over the
    day: delivered + now in the battery − out of the battery = what the roof generated);
  * `factor_ac` now compares against `pv_kwh` (name stays, because the day record and the UI
    know it); additionally `factor_ac_delivered` for the old value and `measured_pv_kwh` as a column;
  * `measured_today_kwh` of the forecast takes `pv_kwh`; restore after restart reads
    `measured_pv_kwh`, falls back to `measured_ac_kwh` for old rows;
  * Tooltip of the forecast row now says "PV generation = inverter output + battery charge".
  * The CSV writer maps new columns **by name** (not by position) — so the new
    field lands in the file without any intervention.
  **Verified:** all 13 test suites green, service restarted, factor live from 0.61 to 0.916.
  **Note:** In a plant with a battery, the **AC output** is *not* the **PV generation**. Anyone who
  sets a PV forecast against measurements must include the battery term, otherwise the factor is
  too low during the day and too high at night.
* **GARAGE PV ADDED AS A SECOND NUMBER (27.09., 15:45).** Owner's request, verbatim:
  *„deye nicht addieren, hier ging es ja vor allem um akku, der nur von SE geladen werden kann.
  Aber so ähnlich wie bei leitung auch deyes produktion als zweiter zahl anzeigen"* (Do not add the deye, here it was mainly about the battery, which can only be charged by the SE. But similarly to the line, also show the deye's production as a second number).
  ```
  Row:  inverter output today / garage PV      13.59 / 2.70
  ```
  **Deliberately NOT added** — only the SolarEdge charges the house battery, so a sum would answer a
  different question. Pattern exactly as with the power (`spv.textContent = SE + " / " + garage`).
  **Implementation** (in `ha-app/evcharge/main.py`, file is **gitignored** — charging apps stay outside
  the repo):
  * `_garage_day` holds `{day, start_kwh, today_kwh, partial}`; the anchor is created on the first
    read of a day from `sdm_state["pv_energy_kwh"]` (= HA `sensor.garage_pv_energie`).
  * `_garage_day_update()` copies the structure of the SE anchor (`_fc_se_latch`), including
    partial-day marker and backward latch (a reset/counter rollover must not invent a number).
  * The value stands as `garage_day` in the state and is only displayed; **no** influence on the control.
  * The tooltip of the row now tells the truth: *"ATTENTION: this is NOT the PV generation, but
    what the inverter DELIVERED"* — the old claim ("exactly the number that your
    monitoring app shows as production") has been wrong since the battery was installed.
  ```
  2026-09-27 13:45:20  garage PV day baseline latched at 2813.300 kWh (HA counter) (partial day)
  garage_day: {"day_kwh": 0.0, "counter_kwh": 2813.3, "partial": true, "day": "2026-09-27"}
  ```
  **Verified:** 511 individual checks across all 13 suites green, `node --check` for the embedded
  JavaScript ok, service cleanly restarted.
  **Note:** The day boundary of the app is **midnight in the house timezone** — in the log it stands
  as **22:00 UTC**, because the server runs UTC and the log shows the house time. The
  day baselines from the log prove it: `29287.762 → 29314.126 → 29343.352` yield 26.364 and
  29.226 kWh — exactly the stored day values.
  **Open:** The lifetime comparison (app 29.36 MWh against SE app 66.5 MWh) remains unresolved.
* **REPORTING RHYTHM MEASURED (26.09., 12:14–12:54).** The long listener shows how often the valve
  sends values on its own:
  ```
  12:26:49  actual=24.0        ← last temperature report after the re-pair
  12:49:19  setpoint=8.0        ← command of the new automation (at 12:49:13)
  12:51:19  actual=23.0        ← ★ next temperature report, ~24 minutes later
  ```
  **Result:** The actual temperature comes **on its own, but rarely — on the order of half
  an hour.** The setpoint, by contrast, appears immediately (it is our own write command).
  **Consequence for the automation:** do not wait for the report-back. The controller writes and knows
  its setpoint; for the actual temperature it must reckon with values up to ~30 minutes old.
  Also visible: the `running_state` correction works (until 12:24 `heating`, afterwards `idle`, which
  with setpoint 8 < actual 24 is correct), and on the re-pair at 12:24:39 the child lock jumped
  to `on` for 30 seconds.
  * **Correction to an earlier claim:** the attic studio is **not** the weakest-radio corner.
    A dedicated router stands there (`Steckdose Mascha`, LQI 140), the house has **26 routers** against
    36 end devices, and the LQI values fluctuate strongly (Mascha's button 172 → 80 within an hour,
    `FensterSensorAQ` in the same room 164). The initially reported 60–68 were snapshots.
    **Why Mascha's button lost its network registration is not
    decided with the available data** (route or coordinator table) — do not present it as settled.
  * **Button → socket runs via HA**, not as a device binding: the automation "Button => Mascha PC"
    (`automation.button_mascha_pc`, id `1758434574653`) listens on `zha_event`, checks
    `device_ieee == a4:c1:38:d6:46:c0:09:e7` and `command == "toggle"` and switches
    `switch.steckdose_mascha`. Evidence for the success of the re-pairing: trigger 19:31:16, six
    minutes after the pairing. **Consequence:** as long as the button is not joined, the socket
    is not switchable, even if its LED lights up. The trigger is unfiltered (`zha_event` for all
    devices) — it works, but an `event_data` filter on the IEEE would be cleaner and, with
    `mode: single`, would also avoid discarding a press during another event.
  Next cells: SZTemp 48 %, WohnzimmerTemp 55.5 %, TempSensorTreppe 59 %.
* **Three button automations switched to a filtered trigger** (25.09.): `Button => Steckdose mein
  PC` (id 1758393492176), `Button => Mascha PC` (1758434574653) and `Button => Steckdose Altar`
  (1758434943261) listened to **every** `zha_event` in the house and filtered only in the condition; now
  `event_data: {device_ieee: …}` stands directly in the trigger. **Only** the trigger was changed —
  condition, action and mode were compared byte-wise after writing and are identical, all three
  still `on`. The filter is the same value that the condition checks anyway (which demonstrably
  works, see the 19:31 trigger). **Proven end-to-end the same evening** (button press 20:40):
  Mascha 20:40:04 → `switch.steckdose_mascha` off; Altar 20:40:44 → `switch.tz3000_gjnozsaz_ts011f`
  ("Steckdose Wohnzimmer Altar") on at 20:40:48. **Important here:** the `switch` entity of the button
  itself does **not** change when pressed (both stayed at their old timestamp) — it is the
  on/off cluster state of the device, not a press counter. Proof of reception is `zha_event`, not this
  entity; the earlier note in the skill `home-assistant-state-forensics` was wrong on that point and
  has been corrected.
* **Two dead automations found:** `automation.goe_nachtladen_start_2` and
  `automation.goe_nachtladen_stop` stand at `unavailable`, for both **no**
  configuration exists anymore (REST config view: 404). Dead entries from the go-e/night-charging era; they can
  no longer trigger anything and therefore do not collide with our own charge controller. Cleanup open.
* **"Fenster Bad"** (`lumi.sensor_magnet.aq2`, previously only the factory name): device name set via the register
  **Entity IDs deliberately NOT renamed** — they are in the Lovelace dashboard
  `fenster-turen`, a rename would have broken the tile. Before any ID rename, first reference
  (automations + all storage boards), that prevented exactly this error here. Inclusion of the
  sensor documented: it reported a 2-second short open/close as both edges.
* **ZHA is the Zigbee binding, not Zigbee2MQTT** (config entry "Sonoff Zigbee 3.0 USB Dongle
  Plus"); `zigbee2mqtt/#` at the broker is empty. On `zha_event`: a **joined button** generates
  one when pressed (documented: `attribute_updated on_off` from Mascha's IEEE seconds after the
  pairing, and **no** event during 5–10 presses before — the clean before/after proof).
  A sleeping presence/battery sensor, by contrast, generates none. An empty event stream proves
  nothing about sensors; the state change of the entity remains authoritative.

## Recently fixed (2026-09-23)

* **HA's location was still on the factory setting Amsterdam** (52.3731/4.8903, elevation 0 m) —
  every sun-based HA automation and later the fallback of our forecast rule would thereby have
  computed for the wrong city. Set via `homeassistant.set_location` to
  **49.1278/8.4076, 105 m** (postcode centre of Linkenheim; the elevation from Open-Meteo, the same
  elevation model as the forecast). Timezone was already `Europe/Berlin`, country `DE`.
  **Evidenced threefold:** `/api/config`, `zone.home` and, as an independent witness, the
  sunset — `sun.sun` jumps from 17:36 UTC (19:36 Berlin) to **17:24 UTC
  (19:24 Berlin)**, 12 minutes earlier, exactly the jump from 52.37° to 49.13° North.
  Subsequently adjusted to the **exact point of the plant** (the owner
  sent it): for that, the elevation was fetched again from the elevation model (111 m) and the sunset
  taken as a plausibility check (820 m shift ⇒ a few seconds, measured 2 s).
  **Privacy rule:** the exact coordinates are **only** in Home Assistant and in the
  local `config.json` (excluded via `.gitignore`, checked with `git check-ignore`) —
  they do **not** belong in this repo or in the docs. Publicly, only the postcode level is here.
  The app also now computes with the same point (freshly retrieved: 28.86 kWh, independently
  recomputed 28.86 kWh).
* **The inverter is demonstrably read-only** — see the rule above: the unused
  battery write functions and the Modbus write primitives are out, structurally pinned,
  and the proxy still counts `upstream_writes: 0`.
* **PV forecast step 1** built and live (see its own section above).

## Recently fixed (2026-09-22)

* **The Deye poller now falls silent at night and wakes on the meter** (different project:
  `HA-POWER-DASHBOARD/deye-pv-rs`). Before: a failed attempt every 33 s, each with a log line
  **and** an `offline` to HA — around 2600 lines and 2600 messages per night for a
  device that sleeps as expected (the SolarMAN logger module is attached to the inverter;
  proof: it was back on its own at around **05:16 UTC**, port 8899 open). Now: doubling
  of the interval up to `--backoff-max` (900 s), log line only on the first error of a series and
  then every eighth step, `offline` only on the state change (once per failure), and the
  recovery line names the number of attempts. **The owner's idea is the wake-up call:**
  during the backoff the poller reads `sensor.sdm630_total_kwh` (grows in both
  directions, so it moves exactly when energy flows in the garage branch — export
  or into the car) and polls again immediately when the meter moves; one early
  attempt per backoff period, so that a logger broken during the day is not hammered.
  Pinned by 6 new unit tests (schedule, throttling, once-per-failure, wake gating),
  33 in the binary + 13 conformance tests green.
* **Third session value: SDM + garage PV** (`ha-app/evcharge/session_meter.py`, owner's
  request; rules above). Two traps closed in the process, each with a test: a **0.00** of the
  inverter meter as the *base* would have turned the next real number into a ~279 kWh correction,
  and a session that begins while the logger sleeps now catches up the base
  afterwards (correction then "partial"). `test_session_meter` **68 checks**, 12 suites green.
* **Measurements at the garage branch** (22.09.2026 — important when reading all the numbers):
  * Three nights, ~11.5 h each: SDM **import and export exactly 0.000 kWh**. The branch is therefore
    at night not "load-free", but **below the counting threshold** of the device (datasheet:
    starting current 0.4 % of Ib = **0.04 A**, specified only from 5 % Ib = 0.5 A; measured:
    0.41 A / 97 VA / **−97 var** / PF −0.20 on L2, meter stays put anyway). Router, gate and
    go-e standby are therefore not counted along — the session number is clean of that.
  * The garage hangs practically **single-phase on L2** (L1/L3 measure 0.00 A); there garage PV,
    go-e, router, gate.
  * The **inverter meter** is too high by **at most 8 %** against the SDM export
    (4.18 against 3.88 kWh in 12 h) — and this gap is **no proof of a fault of the
    inverter**: 0.30 kWh in 12 h are exactly **25 W continuous load on the branch**, and that
    exists there (router, gate, go-e standby) — they are only invisible to the meter because it
    cannot **count** them. The real fault therefore lies between ~0 % (at ~25 W continuous load)
    and +8 %; an inline plug in front of the router would clarify it. After waking up,
    the register briefly reads 0.00.
  * **The meter counts in 0.1 kWh, not in 0.01** (corrected 22.09.2026): our poller read
    it **10× too small** (279.56 kWh instead of 2795.6). Decided **without** the app, via the
    self-consistency of meter and power: in the window 21.09. 05:00–17:00Z the register ran
    **42 steps** further, while the logged AC power yielded **4.18 kWh** → 0.0995
    kWh per step. The Deye app confirms it from the other side (2.79 MWh after 739
    operating days ≈ 3.8 kWh/day, matching the measured daily yield). Consequence: the **third
    session value** too would have been too small by a factor of 10. The poller also publishes
    no backward-running meter anymore — HA reads a decrease under `total_increasing` as a
    meter reset and **adds** the new value, so it would have booked the whole reading as
    generation in the morning.
  * Open: the register is **16 bit** and overflows at **6553.5 kWh** (~2.7 years at
    this yield) — then the backward protection freezes the value, before that the high word
    must be checked.
## Recently fixed (2026-09-21)

* **The app sent `alw=0` even though there was enough sun** (`controller.py: _finalize`).
  Measured on 20.09.: `11:28:36 alw=0` at **2358 W** surplus, `11:30:37 alw=0` at
  **1901 W** — the owner saw it from within the app (2.85 kW production at
  2.66 kW consumption, 99 % solar+battery). Cause: the **start delay hung on
  `charger.charging`** (whether the car is drawing), but the write decision on
  **`charger.enabled`** (whether the wallbox is enabled). With a plugged-in but not
  drawing car (full / departure time), `charge and not charging` was true in **every** cycle
  → the 60 s grace was re-armed every cycle → the decision was forced to
  `charge=False` → the write path (which compares against `enabled`) switched the
  wallbox off. **One stop per minute with plenty of surplus**, five of them in the
  safety window — exactly that triggered the fault. Both delays now hang on
  the same quantity as the write path (`enabled`); a waiting grace writes **nothing**.
  Pinned by „a plugged car that is not drawing must not be stopped every cycle"
  (10 cycles, 0 `alw`), „the start grace waits on the wallbox and writes nothing while it
  waits" and „the stop grace holds first and writes exactly one stop afterwards".
* **Hysteresis 300 W at the lower limit** (`enable_threshold_w` / `disable_threshold_w`,
  `controller.py: _floor_w`). Until then `disable_threshold_w` was **declared and never
  read** — a setting that did nothing. Now: start from lower limit **+300 W**,
  hold until lower limit **−300 W**; the **start hysteresis** the owner lowered on the same day
  to **100 W** (config + restart), the hold limit stayed at 300 W. Live visible
  in the reasoning: „below minimum (**1080 W**, 1p)" as long as the wallbox enables,
  „(**1480 W**, 1p)" when it is off.
* **Safety only counts stops now** (`safety.py`) — rule above, `amx` never counted.
* **Test double corrected** (`tests/test_controller.py`): `car()` sets `enabled` matching
  `charging`. Before, it described states the hardware does not produce (current flows
  without the wallbox enabling) — and thereby masked exactly this fault.
* Live evidence after the restart on 21.09. 06:40: one `amx=6` (re-adjusting while holding), then
  **exactly one** `alw=0` after the 180 s grace expired, then quiet; counter 0/5, no fault.
  Status: **12 suites green** (`test_controller` 94, `test_safety` 33, `test_session_meter`
  68 checks).

## Recently fixed (2026-09-19)

* **Cheap-tariff window stopped running charges** (`controller.py`). Before, the branch read
  `if mode == cheap_hours and cheap_now and not charger.charging`: as soon as the car actually
  drew current, it fell into the surplus logic, was backed off to 6 A and after the
  180 s grace switched off — whereupon the window restarted it 60 s later. **Signature of the
  night of 18/19.09. (00:00–03:25 local): 40× `alw=0` and 41× `alw=1` in the log, reasoning
  oscillated between `cheap tariff window` and `surplus -1117 W below minimum (4140 W, 3p)`;
  each round 31 s at 14 A, 181 s at 6 A, 88 s off.** Now the branch holds every running
  charge until the window ends (a comment in the code explains it, the three checks from the rule
  above pin it). Live evidence for „the fix is running": the process start must be **younger** than
  `controller.py` — `ps -o lstart= -p $(systemctl --user show evcharge-wt.service -p MainPID --value)`.
* **Crash loop of the charging app** (11:13–15:19 local blind, **1182 restarts at ~12 s**,
  port 7080 dead). `proxy.py:summary()` did not normalise `now`, while `main.py` calls it as
  `proxy_summary(pstats, failures=…)` **without** a clock. The line runs only when the
  proxy reports an error timestamp — the new proxy build (09:59) delivers
  `last_upstream_error_at`, the first upstream error at 11:13 turned that into `float(None)`
  in every cycle. Fixed by normalising in `summary()`; covered by
  „the production call shape: summary() without an explicit clock" in `test_proxy_card.py`
  **plus** a live check against `/status`. Both states were demonstrated: fix out → test
  aborts, fix in → green. **Lesson for the future: after every rebuild of the proxy check
  the field set of `/status` and run the app tests** — a new field is a
  new code path, and even a pure display path drags the control down with it.

## Open items

2. The HA Lovelace dashboard (`/strom-verbrauch`) was **never** visually checked in HA itself
   (login wall); the substitute is `docs/preview.html`. This is the biggest open
   uncertainty in the dashboard part.
3. The **Rust port of the charging app has not been started** — it is the next candidate,
   but only once its logic is settled (the plan deliberately left unchanged).
4. **SolarEdge reports too high single-phase above ~4600 W (peak 5533 W)** — the owner's
   suspicion: that was a wrongly detected phase count, not the inverter. The
   phase routine has been stricter since then (believed only after ~20 s of flowing current, then the
   highest value until unplugging; `phases_checked=false` as long as no current flows) →
   **observe** whether the value returns.
5. **The hold in the cheap-tariff window is proven only by tests, never seen at night on the real
   car.** At the next use of `cheap_hours` (winter; the mode is currently set to
   `pv`, where the window has no effect), expect: **one** start at opening, then
   **0 stops** until the window ends, charging throughout at `max_current` — the stop counter
   in the UI must stay at 0. If ~12 stops per hour occur again, the old condition
   has returned (signature in „Recently fixed") and it is code, not hardware; the
   safety latches itself at 5 stops and then writes nothing more.
6. **MQTT is on — done on 2026-09-25.** The owner entered the credentials from the
   Mosquitto add-on himself (`set_mqtt_login.py`: prompts masked with `getpass`,
   writes directly into `ha-app/config.json`, permissions 600, backup as `.bak`; the file is
   gitignored). Proven: direct CONNACK test with exactly the app client → **code 0 = accepted**,
   `mqtt_connected: true`, and in HA **18 entities under „EV Charger WT", none without a value**.
   On the backstory: anonymously the broker accepts nothing (CONNACK 5) and the app client is
   demonstrably correct (MQTT-3.1.1-CONNECT checked) — the owner's first data attempt was
   rejected with **code 5 = not authorised**, so the broker did not know this combination.
   **Two faults came to light in the process — both in code that had never run before:**
   * The discovery templates for `binary_sensor` „EV charging" and `switch` „control enabled"
     emitted Jinja booleans (`False`) — HA recognises in them neither `ON/OFF` nor `false`, both
     entities stayed permanently `unknown`. Now written out, pinned in
     `tests/test_mqtt_loopback.py` against the source text.
   * **Eight orphaned retained discovery messages** were lying in the broker: an older version had
     published `mode`, `max_current`, `min_current`, `buffer_soc`, `priority_soc`, `plan_energy_kwh` and
     `decision` as *sensors* (later they became *numbers*) plus a `binary_sensor`
     instead of the `switch`. They create entities without a value in HA. Deleted with
     `mqtt_discovery_audit.py` (empty payload, `retain=True`) — HA entities: **26 → 18**.
     *Mnemonic:* a retained discovery message survives every code change (the same mechanics
     as with the evcc remnant in item 7, only in our own device). After every change to
     `publish_discovery()` the audit run is worth it.
7. **The evcc remnant in HA stays — the owner cleans it up himself, „irgendwann mal" (some time or other)**
   (decision 2026-09-23, explicitly: do *not* touch it, do not offer it again).
   For reference, what is lying there: the integration entry **`evcc_intg` is on
   `setup_retry`** (HA keeps knocking on a dead server — the only remnant still
   working), plus **97 `evcc_*` entities, of which 96 `unavailable`/`restored`** (59 of them are
   the go-e entities from evcc's MQTT discovery). **There is no evcc automation any more** — the
   earlier note „Automation EVCC PV Laden ab 8 Uhr an" (Automation EVCC PV charging on from 8 o'clock) was outdated (0 evcc automations,
   checked on 23.09.). If he delegates it after all later: the way would be
   `DELETE /api/config/config_entries/entry/<entry_id>` (demonstrably exists — checked with an
   invented ID, clean 404 „Invalid entry specified" instead of 405); the 59
   MQTT entities would **not** go away with that, they hang as retained discovery messages in the
   broker and need emptied topics (`mqtt.publish` with empty payload and `retain` — possible via
   HA itself, without a broker login).
8. **Switch over dashboard rows** (after item 6): point the dead `sensor.evcc_*` rows in the
   HA POWER DASHBOARD at the entities of our own app that will then exist.
9. **The Deye poller should also be switched to MQTT later** (the owner's wish,
   2026-09-20). Today it publishes via HA itself (`POST /api/services/mqtt/publish`,
   token from `~/.hermes/.env`, routines in `powerdash/deye_pv.py` and `deye-pv-rs/src/ha.rs`)
   — that needs no broker login, but can only send. Switch-over when the broker login
   is accessible (item 6), so that **one** pattern applies in the house for all publishers.
   The path prepared for that and then discarded again (HA REST publish, read-only, no
   broker login) lies parked in `ha-app/local-tools/mqtt-via-ha-rest/` — it was not
   taken, because with it the **control out of HA is lost** (the app can via
   this path only send, not receive; mode, current limits and SOC thresholds live in
   its own web UI and via `set/#` topics).

11. ~~SDM630 polling~~ **resolved (2026-09-22): the SDM630 is polled perfectly.** The
    old timestamps are correct — a meter that does not change is **not** re-written by
    HA. Proof: the voltage sensors (`sdm630_l1/l2/l3_spannung`) and the
    frequency change **every ~15 s** (237.83 -> 237.41 V in 75 s). **Lesson for the
    freshness check: it must be a moving value** — voltage yes, power/current **no**
    (at night 0.00 W / 0.0 A, is never re-written, looks like a dead meter).
    `sdm.entity_live` is therefore set to `sensor.sdm630_l1_spannung`.
    **Also normal:** the **garage PV (Deye, 192.168.178.33:8899) is not reachable at night**
    — a micro-inverter is supplied by the sun. Last successful
    record 21.09. **17:38 UTC**, Berlin sunset was 19:40 CEST = 17:40 UTC; it comes
    back by itself after sunrise. **Minor open item:** the poller then writes
    every night every 33 s `ERROR ... Host is unreachable` (~2600 lines) — throttle to one line per
    hour or stay silent between dusk and sunrise.

## Known measurement anomalies of the environment

Not our concern, but bear in mind when reading numbers: `sensor.garage_pv_energie`
drops out 8×; `sensor.evcc_battery_power` is `unavailable` (remnant of the decommissioned control, in HA `restored`); the Deye value is ~4 min
old; `ElektroHeizungKeller` + sensors `unavailable`; the Rust Deye poller cannot do
https to the HA connection.

## How to check (proven commands)

```sh
cd /home/adermake/EV-CHARGER-WT-HA/ha-app
for t in test_safety test_controller test_phase_probe test_service_smoke test_proxy_card \
         test_goe_driver test_ha_read test_site_cadence test_cheap_hours; do
  python3 tests/$t.py; done
python3 /tmp/health.py                      # live situation in ~10 lines
curl -s 127.0.0.1:7080/api/state            # the same situation as JSON
curl -s 127.0.0.1:1504/status               # proxy statistics
cd ../modbus-proxy-rs && make check         # cargo + conformance + cross-comparison + differential (vs baseline) + poll-range config
```

Important when installing the proxy: `make static install` fails with "Text file
busy" as long as it runs → **stop, install, start**. And: check the installed binary
against the build (`sha256sum bin/muxproxy`), do not assume. Order for any proxy change:
`make check` → **`make baseline`** (saves the build that is now leaving service) →
`make static install` → `make check` again (the differential then compares the new build
against that baseline, 14/14 expected).

## Rollback to the direct path (consumer without proxy)

```sh
systemctl --user disable --now muxproxy-rs.service
```

Then on the consumer (charge controller or whoever reads the meters) have the meter configuration
point back to `192.168.178.84:1502` and restart it — **stop the consumer beforehand
and switch all meters in one step**, otherwise it validates a half-changed
configuration against the device and does not save it.

The earlier tool for this (`set_evcc_meter_host.py`, wrote directly into the SQLite of the old
controller) has been removed from the proxy repo, because it served exclusively that; it lies locally under
`modbus-proxy-rs/local-tools/` and is not versioned.

## Repositories (public, MIT)

Four repos, all initialised **in place** (`git init` in the existing
directories) so that the running units keep their paths. Branch `main`, identity only
local per repo (`trwa <me@home>` — does *not* link the commits to the GitHub account),
remote prepared. **All four have been public on GitHub since 2026-09-20** (account
`machtnichts`, MIT, copyright `nixda`):

| Repo | Path | Commit | Files |
|---|---|---|---|
| `modbus-proxy-rs` | `modbus-proxy-rs/` | `778db32` | 30 |
| `evcharge` | `ha-app/` | `a381fdb` | 35 |
| `ha-power-dashboard` | `~/HA-POWER-DASHBOARD` | `e3884ee` | 72 |
| `ev-charger-wt-ha` | this directory | `ff71f9f` | 26 |

Verified by a fresh clone from GitHub (file inventory, licence, no unwanted files);
`evcharge` additionally with a complete test run from the clone (76 checks green). Rust builds from
the clone were **not** run — a `cargo build` run is missing here for that, the
conformance tests in the repo itself remain the reference.

Further changes as usual: `git add` / `git commit` / `git push` in the respective
directory; the working copies track `origin/main`.

**Layout change on 2026-10-03, first step:** the Python reference proxy moved from this repo
(`modbus-proxy/`) into the proxy repo as `modbus-proxy-rs/reference/` — together with its
instruments and its sample config. This repo is now **documentation only** (`STATE.md`,
`README.md`, `docs/`, `LICENSE`). Nothing running changed: the live proxy was always the Rust
binary with its own `config/muxproxy.json`. The Python implementation was **never installed as
a service anywhere** — its systemd unit was deleted on 2026-10-03 (it had been disabled since
the Rust proxy took over the port).

**Second step, same day: the Python implementation itself was deleted.** Reason, measured
before deleting: it was no longer the same version. Side by side against the same dummy
upstream both reported the same 16 counters, but the Python was missing
`upstream_backoff_s`, `last_upstream_error_at` and `cache_registers`, and it never
incremented `validation_failures` — it would have answered **"0 validation failures" while
failures were happening**. Exactly those fields are what the charging app reads for its
proxy card (`ha-app/evcharge/proxy.py`), so a folder named "reference" that disagrees with
the binary in service about the status surface is worse than no folder. It had also simply
not been touched since the port; nothing kept the two in step. What was kept:

* the **wire-protocol suite** `tools/python/test_protocol_conformance.py`, still run against
  the Rust binary by `tools/cross_check_python_suite.py` (17 checks) — that suite was written
  against the protocol, not against an implementation, which is what makes it an oracle;
* the four hand instruments and the measured map: `site_decode.py`, `discover_sunspec.py`,
  `check_cache_integrity.py`, `probe_goe.py`, `sunspec_map.json`;
* the Python-era **poll-range config** as `config/poll-ranges.json` (19 ranges,
  `expect_header`, `sf_offsets`) — kept because it is the only thing that exercises the
  polling and validation path; the config in service has **no poll ranges at all**.

**The differential test changed sides with it.** It used to compare Python against Rust. From
now on it compares **the fresh build against the last known-good build**: `make baseline`
copies the binary in service (`bin/muxproxy` → `baseline/muxproxy`, both unversioned, ignored
by git), and `make differential` puts that build and the new one in front of their own stub
and compares 14 response PDUs byte for byte. Same strength, no second implementation to
maintain — and the first baseline is exactly the binary that had already been proved equal to
the Python. `make check` tolerates a missing baseline (a fresh clone cannot have one) with a
loud note; an explicit `make differential` fails instead of quietly comparing nothing.

**Verified after the change:** `make check` green — 38 cargo tests, 17/17 cross-check,
**14/14 differential** (this build vs the baseline of 19.09), 13/13 poll-range config. The
paths were followed through every place that named the old location (this file, `README.md`,
`docs/INSTALL.md`, the Rust `README.md`, `INSPECT`/tools, `.gitignore`).

**Deliberately not in the repo**: `bin/` (built binary), `target/`, `.venv/`, `logs/`,
`__pycache__/`, the controller's live `config.json` (instead `config.example.json`) and
`NOTES-local.md` in all four repos.

**evcc references**: completely removed from the **charge controller** (comments, docstrings,
UI tooltips, test labels); the substance — three battery bands, `bufferSoc`/`prioritySoc`,
the 3-phase minimum — is in `ha-app/NOTES-local.md` (not committed). The **proxy** was
likewise generalised ("Modbus consumer", `config/muxproxy.json`, the three evcc tools to
`local-tools/`), and the **tools of this repo** that needed evcc's API or CSVs lie
now likewise in `local-tools/` (not versioned) — evcc never runs again.

**Verification now runs against the own app**, not against a third-party controller:
`curl -s 127.0.0.1:7080/api/state` (state) · `tests/` of the app · `curl -s 127.0.0.1:1504/status`
(proxy) · `make check` in the proxy repo (conformance against the stub).

**As of 2026-09-20: all four are published and the history is smoothed** —
**one** commit per repo (`evcharge a381fdb`, `modbus-proxy-rs 8fbd4c1`, `ev-charger-wt-ha
a4d0806`, `ha-power-dashboard e3884ee`), pushed with `--force` after orphan-branch rewrite,
old objects removed locally via `reflog expire` + `gc --prune=now`. `modbus-proxy-rs` was
**deleted and recreated empty** on 20.09 and pushed fresh, because its first
import commit was still retrievable on the server by exact SHA; afterwards it no longer was.
Something like that can only be re-checked with the exact hash:

```sh
git -C /home/adermake/EV-CHARGER-WT-HA/modbus-proxy-rs fetch --depth=1 origin 778db32 && echo "noch da" || echo "weg"
```

A fresh clone of the repo must also **build and test itself**:
`git clone … && cd modbus-proxy-rs && cargo test --offline` → 12 tests, 0 errors (the repo
is dependency-free).

## Where the truth lies

* `README.md` (project), `docs/INSTALL.md`, `docs/REGISTERS.md` (register map, 203 =
  grid meter), `docs/preview.html` (dashboard preview).
* Logs: `logs/evcharge.log` (heartbeats + switching writes), `logs/evcharge.stdout`
  (tracebacks). The proxy writes to stdout into a file, not to the journal.
* `ha-app/config.json` — intervals, reserve, `phases`, `proxy_status`, `safety`, and the
  persisted settings (`.bak` is retained).
* The instruments kept from the Python era: `modbus-proxy-rs/tools/python/`
  (`site_decode.py`, `discover_sunspec.py`, `sunspec_map.json`,
  `test_protocol_conformance.py`, `check_cache_integrity.py`, `probe_goe.py`). The Python
  implementation itself is **gone**; the poll-range config it ran with is
  `modbus-proxy-rs/config/poll-ranges.json`, the config in service is
  `modbus-proxy-rs/config/muxproxy.json`.
* The yardstick for proxy changes: `modbus-proxy-rs/baseline/muxproxy` — the last build that
  was in service, saved by `make baseline`, unversioned. The differential test compares the
  fresh build against it.
* Skills (procedural knowledge, load as needed): `ev-charging-control`,
  `modbus-single-client-proxy`, `goe-charger-http-api`, `solaredge-sunspec-modbus`,
  `home-assistant-integration`, `port-verification`.

## The one rule for new sessions

First read this file, then `python3 /tmp/health.py`, only then change anything. The
inverter is sensitive, the car expensive, and the owner notices when numbers
are not backed by evidence.
