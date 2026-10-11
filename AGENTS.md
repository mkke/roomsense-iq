# roomsense-iq (fork)

Fork of `roomsense/roomsense-iq`, the ESP-IDF firmware of the RoomSense
IQ room sensor (ESP32-S3, 8 MB flash, two 3 MB OTA app slots). The two
units in service (bedroom `9038`, living room `D95C`) feed
weather-backend over MQTT.

Remotes: `origin` is the fork `git@github.com:mkke/roomsense-iq.git`,
`upstream` is the vendor repository. `main` tracks `origin/main`.

Open work goes in `TODO.md`, which exists only while there is some. No
commit is pushed to `origin` until every item there is implemented and
tested on the dev board.

## Build

The vendor CI builds with ESP-IDF `release-v4.4`; use the same image:

```bash
docker run --rm -v "$HOME/git/roomsense-firmware:/project" -w /project/firmware espressif/idf:release-v4.4 idf.py build
```

The app image is `firmware/build/firmware.bin`; it goes into `ota_0` at
`0x10000`. Secure boot, flash encryption and app rollback are off, so a
self-built image boots and a failed one is not rolled back.

## Dev board

A base board without the sensor module: MAC `80:65:99:a3:10:34`, device
ID `1034`, on `/dev/cu.usbserial-10` through a USB-to-UART bridge.
Without the sensor module the climate tasks log I2C errors and carry on.

- esptool lives in `~/.venvs/esptool`. The shell hook refuses running it
  from an agent session, so the user runs the flash and monitor commands.
- Use 115200 baud. At 460800 the bridge corrupts transfers
  (`Invalid SLIP escape`).
- A full 8 MB backup of the board's factory flash is
  `~/git/roomsense-firmware-factory-1034.bin`.

Flash the app slot:

```bash
cd ~/git/roomsense-firmware/firmware/build && ~/.venvs/esptool/bin/esptool --chip esp32s3 --port /dev/cu.usbserial-10 -b 115200 write-flash --flash-mode dio --flash-freq 80m --flash-size 8MB 0x10000 firmware.bin
```

Serial log (`Ctrl-T Ctrl-R` twice resets the board, `Ctrl-]` quits; the
monitor holds the port):

```bash
~/.venvs/esptool/bin/python -m serial.tools.miniterm --raw --rts 0 --dtr 0 /dev/cu.usbserial-10 115200 | tee /private/tmp/roomsense-1034.log
```

## Test setup

- **Broker:** mosquitto on this Mac, port `1884`, anonymous, with
  `log_type all` so every publish and disconnect is logged. Keep test
  boards off the production broker: every unit announces the Home
  Assistant device identifier `RoomSense`, so Home Assistant would merge
  a test board into the live device.
- **Wi-Fi:** SSID `L10_2G` has a MAC allow list in the UniFi controller;
  a new board's MAC has to be added there first.
- **Wi-Fi drop:** disassociate the board on its AP over SSH, each call
  announced and confirmed first. ap4 (Qualcomm):
  `hostapd_cli -i ath3 disassociate <mac>`. ap3 (MediaTek, Dropbear)
  needs `-o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa`.
- **Firmware version of a unit:** `curl -X POST -d ota_update_status http://<ip>/OTAstatus`
  returns the running image's `version` (`git describe` of the tree it
  was built from), its build time, and `elf_sha256`, the first 8 hex
  digits of the ELF's SHA-256.

## Firmware behaviour to know

- The web page's Wi-Fi connect does not disconnect from the current
  network first, so it hangs on "Connecting..." and stores nothing.
  Press Disconnect, then connect through the unit's own access point.
- The last will is a retained `offline` on `RoomSense/<id>/status`, the
  availability topic every discovery config references. The unit
  publishes `online` there on connect.
- `MQTT_EVENT_DATA` is ignored and nothing is subscribed, so a unit
  cannot be told to restart over MQTT.
- The unit restarts itself on an `MQTT_EVENT_ERROR` with socket errno 11
  or 23, and after 600 s without an MQTT connection when its MQTT
  credentials came from flash.
- Published temperature and humidity come from the SCD40
  (`main/climatesense.c`), with a 12.1 C offset stored in the sensor's
  EEPROM. Each start of its periodic measurement makes the temperature
  read about 9 C high, settling with a time constant of about 60 s. A
  boot leaves a running sensor running, and after a power-up temperature
  and humidity are held back for 240 s.
