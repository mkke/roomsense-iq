# TODO

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
