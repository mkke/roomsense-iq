# TODO

## MQTT reconnect watchdog

Restart the unit after 10 minutes without an MQTT connection, in
`components/ha_mqtt/ha_mqtt.c`:

- record the time MQTT goes down (`MQTT_EVENT_DISCONNECTED` and start-up),
  clear it on `MQTT_EVENT_CONNECTED`;
- a periodic `esp_timer` checks once a minute and, past
  `MQTT_WATCHDOG_TIMEOUT_S` (600), logs the reason and sends
  `MQTT_APP_MSG_RESTART_CPU`, which saves the occupancy state before
  `esp_restart()`;
- arm only when MQTT credentials were loaded from flash, so a unit in
  setup (own access point, default broker) does not restart every 10
  minutes.

Test: stop the test broker for more than 10 minutes; expect exactly one
restart, then a reconnect once the broker is back.

## Last will on the availability topics

The last will points at `esp_discovery_number_max_macro_range.status_topic`.
It has to set the 19 `<Variable>/<id>/status` topics `offline`, so Home
Assistant marks a dead unit unavailable. MQTT allows one will per
connection, so this needs a single device-level availability topic that
the discovery configs reference.

## Release

Once both items are implemented and tested on the dev board: push
`main` to `origin`, then update one unit in service over the air and
watch it before updating the second.
