# Install and cutover

## 1. The proxy (on this host, knetzwerk)

The proxy must own the inverter connection before anything else reads the plant.

    # start it now
    python3 /home/adermake/EV-CHARGER-WT-HA/modbus-proxy/muxproxy.py

    # make it permanent (user service, no sudo needed)
    mkdir -p ~/.config/systemd/user
    cp /home/adermake/EV-CHARGER-WT-HA/modbus-proxy/systemd/*.service ~/.config/systemd/user/
    systemctl --user daemon-reload
    systemctl --user enable --now evcharge-modbus-proxy
    systemctl --user status evcharge-modbus-proxy

    # survive logout / reboot (needs sudo once, so the user has to run it)
    sudo loginctl enable-linger adermake

Health check:

    curl -s http://127.0.0.1:1504/          # JSON: status, counters, cache
    curl -s http://127.0.0.1:1504/metrics
    curl -s http://127.0.0.1:1504/ranges

## 2. Point the consumer at the proxy instead of the inverter

This is the important step: while a consumer talks to the inverter directly, it is a
second Modbus client, which is exactly what the inverter cannot tolerate.

The controller that ran here before kept its configuration in a SQLite file (there is no
YAML to edit). The templates point at host `192.168.178.84`, port `1502`, unit 1.
Change the three meters (grid, pv, battery) and the charger entry to host
`192.168.178.44` (this host) port `1503`, keeping unit 1 — then restart the consumer.

Doing it from the controller's own UI is the least error-prone route: Configuration -> each
meter/charger -> host field.

After the restart, confirm the consumer's numbers are unchanged and that only the proxy
holds a connection to the inverter:

    ss -tn | grep 1502        # expect exactly ONE established connection
    curl -s http://127.0.0.1:1504/ | grep -E "clients_total|upstream_errors"

## 3. Home Assistant

Option A (recommended): install the proxy's client as HA's Modbus integration and
let the add-on do the controlling.

* Enable "Advanced SSH & Web Terminal" or "Samba share" once — the add-on has to
  be copied into `/addons`.
* `cp -r /home/adermake/EV-CHARGER-WT-HA/ha-app /addons/evcharge_wt`
  then Settings -> Add-ons -> Add-on store -> three-dot menu -> Check for updates
  -> "EV Charge WT" -> Install -> Start.
* In the add-on configuration set `site_host` to this host's IP (192.168.178.44)
  and `charger_host` to 192.168.178.22. `control_enabled` stays false to start.
* Optionally install the "Mosquitto broker" add-on and set `mqtt_enabled: true`
  plus `mqtt_host: core-mosquitto` to get all entities auto-discovered.

Option B: run the same app here on the host via the systemd user unit
(`ha-app/systemd/evcharge-wt.service`) and skip the add-on entirely. Home
Assistant can then pull everything over MQTT or via the REST API.

Optional, if you want the raw numbers as native HA sensors regardless of the app:
add HA's built-in Modbus integration against `192.168.178.44:1503` (holding
registers). Remember the inverter's scale factors are dynamic, so read value and
scale factor together and compute in a template sensor — or simply rely on the
app's MQTT entities, which already do that correctly.

## 4. Cut over control

Only after the previous controller has stopped driving the charger:

1. Set the add-on option `control_enabled: true` (or use the web UI toggle).
2. Watch the decision for a while: the web UI shows surplus, target current and
   the reason string, e.g. "pv: surplus 2380 W -> 10.3 A".
3. Keep `mode: pv` initially and compare against what the plant did before.

To stop the app interfering instantly: toggle control off in the web UI, or set
`mode: off`. Nothing else in the plant is touched by software — the app never
writes to the inverter.

## Rollback

* Set `control_enabled: false` (app stops writing to the charger).
* Point the consumer back at 192.168.178.84:1502 and restart it.
* Stop `evcharge-modbus-proxy`; the inverter is then unowned again.
