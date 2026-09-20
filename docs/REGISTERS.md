# SolarEdge + go-e wiring notes (reverse-engineered and verified on this plant)

Everything here was established empirically against the live plant and
SolarEdge's "SunSpec Logging" Technical Note v3.2 (June 2025). Where a value was
verified, the check that proved it is written down — do not change these numbers
without repeating the check.

## Device

    SolarEdge Hybrid inverter, Modbus/TCP 192.168.178.84:1502, unit id 1
    Only ONE Modbus client is supported (SolarEdge states this explicitly).
    TCP server idle timeout is 2 minutes.

## SunSpec chain (PDU / base-0 addresses, as returned by this device)

    SunS marker       40067
    model 101 inverter  header 40069, len 50,  data 40071..40120
    model 1   common    header 40121, len 65,  data 40123..40187
    model 203 meter     header 40188, len 105, data 40190..40294
    models 701..713     header 40295 .. 41264  (SolarEdge vendor storage models)

Note the chain order (101 first, then common, then meter) is unusual; do not
assume a spec-like order. The vendor models are contiguous, so the whole region
40069..41300 can be polled as one block sequence.

## Model 101 (inverter), data indices

    0 A 1 A_SF 2..4 AphA/B/C 5..7 PPVphAB/BC/CA 8..10 PhVphA/B/C 11 V_SF
    12 W 13 W_SF 14 Hz 15 Hz_SF 22/23 WH(acc32) 24 WH_SF
    25 DCA 26 DCA_SF 27 DCV 28 DCV_SF 29 DCW 30 DCW_SF 36 Status

Verified: DCW 3764 W and DCV 410 V match the app's PV readings; inverter AC W ~3709 W.

## Model 203 (three-phase meter = grid), data indices

SolarEdge implements a superset of the SunSpec meter model here. Critically, the
real-power block is NOT at the spec offset 9 — reading offset 9 yields line-to-line
voltage, which is why a spec-based reader appears to "work" but reports nonsense.

    0 M_AC_Current (sum) 1..3 A/B/C 4 M_AC_Current_SF
    5 M_AC_Voltage_LN (avg) 6..8 A/B/C 9..12 LL/AB/BC/CA 13 M_AC_Voltage_SF
    14 M_AC_Freq 15 M_AC_Freq_SF
    16 M_AC_Power 17..19 A/B/C 20 M_AC_Power_SF        <-- the power block
    21 M_AC_VA 22..24 25 VA_SF
    26 M_AC_VAR 27..29 30 VAR_SF
    36/37 M_Exported (acc32)   44/45 M_Imported (acc32)   52 M_Energy_W_SF

Proof used: per-phase powers sum to the total (22588 + 9826 - 26687 = 5727 ~= 5726
raw), and the decoded values agree with the app to the digit:
    battery SOC 84.4000015258789 (identical), PV 4449 vs 4444 W,
    grid import/export counters 31378.734 / 32700.105 kWh (identical),
    grid currents within 0.02 A.

### Scale factors are DYNAMIC

M_AC_Power_SF was observed as -2, -3 and -4 within minutes; current/voltage scale
factors change too. Consequences:
  * never hard-code a scale factor;
  * always read the value and its scale factor in the SAME block (the proxy does);
  * a client that reads the value and the SF in separate requests can silently
    mis-scale by 10x.

### Sign convention

The meter reports export as positive. The previous controller (template `solaredge-hybrid`, grid
usage) applies `scale: -1`, so:

    grid_power (import positive) = -1 * (M_AC_Power)

Confirmed by a controller refresh epoch where the raw register read -839.8 W while the app
reported +839.8 W.

### Energy counters

0.01 Wh units: kWh = acc32 * 10^(M_Energy_W_SF - 3). With SF -2 that is 1e-5.
Verified: 3137873400 * 1e-5 = 31378.734 kWh == the app's grid energy exactly.

## Battery (SolarEdge vendor registers)

    0xE174 float32  instantaneous power, POSITIVE = charging (the app negates it)
    0xE184 float32  state of energy / SOC in %
    0xE176 uint64   lifetime discharged counter, 0.001 kWh
    0xE17A uint64   lifetime charged counter, 0.001 kWh
    0xE00D uint16   StorageRemoteCtrl_CommandMode  (7 self-consumption, 3 charge)
    0xE010 float32  StorageRemoteCtrl_DischargeLimit (W)

    Floats are stored with the 16-bit words SWAPPED (struct ">HH" with lo,hi).
    Readable blocks observed: 0xE000, 0xE040, 0xE100, 0xE140, 0xE170, 0xE180.
    Holes that time out if read as part of a larger span: 0xE080, 0xE0C0, 0xE1C0.

Verified: SOC 84.4000015258789 and battery power -581 W vs -582 W in the app (exact).

## go-e Charger HOME+ (HTTP API v1, firmware 041.0)

    /status returns JSON; /api/status (v2) is NOT enabled on this device.

    nrg[0..2]   phase voltages (V)
    nrg[3..5]   phase currents (0.1 A)
    nrg[6..8]   phase power (0.1 kW)
    nrg[11]     total power (10 W)      -> 135 == 1350 W == the app's charge power
    nrg[12]     power factor (0.01)
    eto         lifetime energy (0.1 kWh)  -> 200370 == 20037.0 kWh
    car         1 idle, 2 charging, 3 waiting, 4 complete
    amp         present current limit (A)
    alw         1 = charging allowed
    amp/tma/tmp temperatures, fwv firmware, sse serial

    Control: /set?amp=<6..16> and /set?frc=<0 neutral | 1 stop | 2 start>

Empirical quirk: while the vehicle charges single-phase at 5.9 A the currents array
reads [0.1, 5.9, 0.0] — the vehicle load appears on the *second* channel. Use the
largest phase current rather than a fixed phase index.

## Known flakiness

The inverter intermittently answers with a shifted or stale block. The proxy
therefore validates polled ranges (`expect_header`, `sf_offsets` in config.json)
and never lets a failing block overwrite good cached data. Without that check a
single bad response silently corrupts every downstream reading (observed as wild
grid values of +-1.8 kW while the real flow was a few tens of watts).
