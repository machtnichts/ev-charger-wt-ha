# EV-CHARGER-WT-HA

A small, plant-specific charging controller: charge the car from PV surplus and
cheap tariff windows, with the control logic living in Home Assistant.

It is deliberately hard-wired to *this* installation (SolarEdge Hybrid inverter +
go-e Charger HOME+) rather than being a general-purpose tool. Everything vendored
in from a generic EV charging app is the behaviour, not the code.

## Why there are two pieces

The SolarEdge inverter accepts **one** Modbus/TCP client and starts producing
errors when several poll it. So:

    1. modbus-proxy/  - the single Modbus client. Polls the inverter on a fixed
                        schedule, caches the registers, and serves any number of
                        readers. Validates every block so a flaky inverter
                        response cannot silently corrupt a reading.
    2. ha-app/        - the charging logic (reads the proxy, drives the go-e),
                        packaged as a Home Assistant add-on, with a web UI, a
                        REST API and MQTT discovery.

    SolarEdge 192.168.178.84:1502
             |
             |  one persistent connection, one request at a time
             v
      muxproxy  :1503 (Modbus)  :1504 (status)
        |        |
        |        +--> Home Assistant Modbus integration / the charging app / any tool
        v
      ha-app  :7080 (web UI + REST)  -->  go-e Charger 192.168.178.22
             |
             +--> MQTT discovery + state --> Home Assistant entities

## Status

Working and verified against the live hardware:

* proxy: 890+ downstream requests with 0 upstream errors, 0 timeouts,
  0 reconnects, 0 failed poll cycles (measured over 17 minutes); a burst of 120
  client reads cost exactly **1** inverter read (120x reduction).
* register map: verified digit-for-digit against the plant's live readings (SOC 84.4000015258789,
  battery power -581/-582 W, grid import/export counters, currents).
* controller: 22/22 unit tests, plus a live dry run against the real plant.
* MQTT: verified both directions against a loopback broker (18 discovery configs,
  state publishing, inbound commands).

**Control is disabled by default.** Until `control_enabled` is switched on, the
service only observes and logs what it *would* do. Switching it on while another controller is
still managing the charger will fight it for the charger — see the cutover below.

## Settings that matter

    mode             off | now | minpv | pv | cheap_hours | manual
                     pv          = follow the PV surplus only; ignores the cheap window.
                                   Exception: an active plan (below) may import from the
                                   grid to meet its deadline - a plan is a commitment,
                                   not a tariff choice.
                     cheap_hours = the cheap_hours window below, plus PV when the sun shows up
    min/max current  6 .. 14 A   (the wallbox/car limit on this plant)
    enable/disable delays   60 s / 180 s
    priority_soc     55 %    below this the house battery has priority: its charging
                             share stays with it (the car runs on the real export, not
                             blocked) and a drain into the car with nothing exported
                             stops the charge; from this SOC up the car gets that share
    buffer_soc       80 %    above this the reserve above the buffer may carry a RUNNING
                             charge: pv/minpv holds the car at the minimum current
                             instead of switching it off, until the SOC is back at the
                             buffer. A charge is never started on battery energy.
    cheap_hours      00:00-05:00  (grid 0.18 vs 0.28 EUR/kWh) - used by mode cheap_hours only
    plan_energy_kwh + plan_deadline   "charge 10 kWh by 07:30", continuous top-up
    residual_power_w        deliberate reserve held back below the surplus; 0 on this
                            plant, because the meter reading already carries the house

Settings changed in the web UI or through /api/settings are now persisted back into the
config file they were loaded from (written through a temp file + os.replace, the
previous content kept as config.json.bak), so a restart no longer reverts mode, limits or
control_enabled. The Home Assistant add-on options file is not written - the supervisor
owns it.

The PV row shows two numbers, `SE W / garage W`: the SolarEdge value comes from its own
registers (`pv_power_w` = inverter DC bus + battery), the garage array is read from HA
(`garage_pv.entity`, default `sensor.garage_pv_leistung`, every 60 s, with the reading's
age in the tooltip). The HA read is display only - it can fail without touching the
charge, and it is deliberately *not* added to the surplus, because the grid meter (model
203) already sees the whole installation behind it. Credentials come from the process
environment or `~/.hermes/.env`; the token is only ever sent as an Authorization header.

## Manual mode

`manual` hands the charger to you. The service keeps reading and reporting, but
issues **no writes at all**: no start, no stop, no current change. Switching to
manual while the car is charging leaves that charge running at whatever the wallbox
is set to — it is a handover, not a stop.

The guarantee lives in one place: `hardware_actions()` returns `[]` before anything
else can add an action, above the `control_enabled` (dry-run) gate. The only two
charger-write call sites in the service are in the loop that applies those actions,
so manual mode means zero writes, also with control enabled. `tests/test_controller.py`
pins this down ("an ongoing manual charge is not stopped", "full surplus yields no
commands").

The UI shows an amber `manual` badge, how long manual has been active, and what the
charger is actually set to. Leaving manual hands control back on the next cycle.

Note: manual mode muzzles *this* app only. Anything else talking to the charger —
the go-e app — is unaffected.

## Following the sun, and the 6 A floor

**Inside 6..max A the current follows the surplus immediately.** A rise or a fall is
applied the moment it is measured: inside the window the surplus *is* the target, so a
step-limited ramp would only make the car trail the sun. (There used to be a 1 A per
control cycle limit in both directions - on a cloud it meant the car kept drawing from
the grid or the house battery until the ramp caught up.)

**Hysteresis exists only at the floor.** Below the minimum current the car cannot be
charged at all, so that is where the decision needs damping: even when the surplus is
gone the app holds the charge at 6 A and waits out `disable_delay_s` (180 s), so a
cloud, a kettle or a passing load never switches the car off. If the surplus returns
inside the window the timer is cleared and the current jumps straight back to it; if
not, the charge is stopped once. `enable_delay_s` (60 s) damps the start the same way.

**Phases come from the cable, not from the car.** The cable's pilot signal decides which
conductors the box energizes, so both cars here charge on a single phase whenever the
1-phase cable is in use - one car has a 3-phase on-board charger, the other a 2-phase
one, and neither can use more than the cable offers. Measured once current flows:

| cable | on-board charger | phases seen | minimum |
| --- | --- | --- | --- |
| 1p | 2p or 3p | 1 - `[x, 0.1, 0.1]` | 1380 W |
| 3p | 2p | 2 - `[x, x, 0.1]` | 2760 W |
| 3p | 3p | 3 - `[x, x, x]` | 4140 W |

That is why the app assumes one phase before a charge starts (it is what lets it begin
at ~1.3 kW instead of waiting for 4.1 kW) and switches to the measured count on the
first cycle that carries current. The 1p cable is the normal case for PV surplus; the
3p cable is for winter or for loading the car quickly at night, when the surplus does
not matter.

**Three battery bands, as specified for this plant.** Below `priority_soc` the house battery has
priority: what it is *taking* stays with it and the car charges on the real export (in
which the battery's draw is already missing, because the meter measures it) - so a grey day
with a half-full battery charges the car instead of pushing the sun into the grid, and the
car is **not** blocked. From `priority_soc` up that share is added back to the surplus, so
the car gets it too, outranks the battery's charging and settles the battery at its level
instead of letting it run to 100 % (past `max_current` the car cannot take it all, so the
remainder still goes to the battery or the grid). Above `buffer_soc` the reserve above the
buffer may also *carry* a running charge: pv/minpv holds the car at the minimum current
instead of switching it off when the sun goes, until the SOC is back at the buffer.

Two limits always hold: the battery's discharge is otherwise subtracted from the surplus
(the battery's contribution is not solar, so the car never quietly drains the house
battery), and a charge is never *started* on battery energy - only one that is already
running is carried. `residual_power_w` is a deliberate reserve below the surplus and is
0 on this plant - the signed grid reading already carries the house's own consumption, so
subtracting an estimate of it a second time would count the base load twice and hold the
charge about an amp low.

The remaining grace time appears under the reason row:

    grace   stopping charging in 2 min 40 s if it doesn't get better

`tools/grace_timer_check.py` exercises this on demand: it collapses the surplus for
about a minute and restores it, which is enough to watch the timer start and clear
without ever interrupting a running charge.

## Talking to the wallbox (go-e HTTP API v1)

This box (firmware 041.0) speaks the **v1** API only:

    read    GET /status
    write   GET /mqtt?payload=<key>=<value>       e.g. /mqtt?payload=amx=8

`/api/set` (v2) and `/set` (the legacy short form) both return HTTP 404 here. A
local setter answers with the charger's *full status JSON*, not the cloud API's
`{"success":true,...}`; a bad payload answers `{"success":false,"error":...}`. The
driver tries `/mqtt` first, then `/api/set` and `/set` for other firmware
generations, and remembers the form that answered.

Current limits go out as **amx**, never **amp**: go-e's spec says amx "will not be
written on flash ... Recommended for PV charging". `amp` persists every write and PV
charging changes the limit constantly, so amp would wear the controller's flash out.
One consequence: /status only echoes the persisted `amp`, so the driver reports the
last `amx` it set (and falls back to `amp` after a restart).

Phases come from `pha` (a bit field: phases before/after the contactor), never from
the voltages. This plant has a 3-phase wallbox connection but a 1-phase vehicle
cable, so the box reports three supply phases while the car can only ever draw on
one - and a controller that trusts the box waits for ~4.1 kW before it will start. Only the
bits *behind* the contactor describe the vehicle, and only once current flows, so the
controller starts from the configured `phases` (1, ~1.4 kW) and re-evaluates with the
measured count as soon as the car actually charges.

Hysteresis: the enable/disable delays (60 s / 180 s) plus a 0.6 x min-current hold, so
a passing cloud or a kettle does not switch the car off. Surplus is computed from the
*signed* grid power, so a car drawing from the grid counts as negative surplus (with
`exporting_w`, clamped at 0, the car's own draw looked like surplus and the delay
would never have engaged).

`tools/goe_write_probe.py` exercises the write path with a no-op (it sets `amx` to the
current limit) and reports which endpoint answered.

## Install

See `docs/INSTALL.md`. In short:

* proxy: user systemd unit (`modbus-proxy/systemd/`), needs `loginctl
  enable-linger` once so it survives logout;
* app: copy `ha-app/` into the Home Assistant add-on folder (`/addons/evcharge_wt`
  on HAOS, reachable via the SSH or Samba add-on), then install it from the
  add-on store as a local add-on.

## Tools

`modbus-proxy/tools/` holds the instruments used to build this, kept because they
are the fastest way to re-verify after any change — all of them run without any
other service being up:

    discover_sunspec.py       walk the SunSpec chain and dump the model map
    site_decode.py            authoritative decoder (used by the app)
    test_protocol_conformance.py  protocol/cache conformance against a stub
    check_cache_integrity.py  scale-factor stability across repeated reads
    probe_goe.py              go-e API discovery

Verifying the **running** plant goes through the charging app, never through a
foreign controller: `curl -s 127.0.0.1:7080/api/state` (its own view of the
plant), `ha-app/tests/*.py` (its logic), `curl -s 127.0.0.1:1504/status` (the
proxy's counters) and `make check` in the proxy repository.

## Caveats

* MQTT was tested against a loopback broker, not against a real Mosquitto —
  no broker is running on this network yet.
* Battery control registers (0xE00D / 0xE010) are implemented but unused and
  untested; the current strategy never writes to the inverter.
* The go-e `set` path (`/set?amp=`, `/set?frc=`) is implemented but has not been
  exercised against the charger, because another controller owned it at the time.

## Repositories

This tree is the project documentation plus the Python reference proxy. The two apps live
in their own repositories, so each can be versioned and read on its own:

| Repo | Contents |
|---|---|
| `modbus-proxy-rs` | the Rust proxy that holds the inverter's single Modbus session (`modbus-proxy-rs/`) |
| `evcharge` | the charging controller, its web UI/REST API and tests (`ha-app/`) |
| `ha-power-dashboard` | the garage-PV poller and the Home Assistant "Strom" dashboard (`~/HA-POWER-DASHBOARD`) |

Both app directories are in this repo's `.gitignore` and are not tracked here. `STATE.md`
remains the plant's handover document and refers to all three.
