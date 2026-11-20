# XHouse EGA1800 gate → local with ESPHome

*Español: [README.md](README.md)*

Convert an **XHouse EGA1800** swing gate (board `EGA-18`, "WiFi+BLE Swing Gate")
from a **cloud-dependent** device into a **100% local** one, controlled from Home
Assistant with [ESPHome](https://esphome.io).

No hardware swap required: the "Smart" module the gate ships with **is already an
ESP32-C2**, so it is **reflashed in place** with ESPHome. It stops talking to the
manufacturer's cloud (Alibaba Cloud IoT) and talks directly, over UART, to the
STM32 that drives the motors.

## Why

- **Speed.** The command travels ESP ↔ STM32 over a wire; it opens instantly. No
  round-trip to a server overseas.
- **Cloud independence.** It works even with no internet, even if the manufacturer
  shuts down the service or changes the app. Your data never leaves your network.
- **Native integration.** It shows up in Home Assistant as a `cover` (gate),
  pedestrian/vehicle buttons, backup-battery voltage and per-leaf state.
  Automatable like any local entity.

## ⚠️ Before you start: back up the stock firmware

Flashing ESPHome **erases** the module's factory firmware. If you ever want to go
back (or something goes wrong), **you need that backup**. There is no public
download of the ESP firmware, so **this is your only copy**. Do it first.

With a **3.3 V** USB-UART adapter connected to the ESP32-C2 (TXD0/RXD0, EN/reset,
3V3, GND) and **GPIO9 to GND** to enter download mode:

```bash
esptool.py --port /dev/ttyUSB0 read_flash 0 0x200000 esp_stock_backup.bin
```

Keep `esp_stock_backup.bin` (2 MB) somewhere safe, off the machine. To restore the
original firmware later:

```bash
esptool.py --port /dev/ttyUSB0 write_flash 0 esp_stock_backup.bin
```

> The physical details (board photo, Smart connector, pin tap and download mode)
> are in [`docs/hardware.en.md`](docs/hardware.en.md).

## How it works (software)

```
Home Assistant  ──ESPHome API──►  ESP32-C2 (this firmware)  ──UART 115200──►  STM32  ──►  motors
```

- The **STM32** is the gate's brain: it drives motors, limit switches, the backup
  battery (24 V), the display and the RF remotes. **It is not touched.**
- The **ESP32-C2** ("Smart" module) normally adds WiFi+BLE and talks to the cloud.
  We replace it with ESPHome, which speaks the **same UART protocol** the STM32
  expects.
- Control command: `55 AA 01 <channel> 02` (emulates one remote-control press,
  step by step). State and battery: the ESP polls with `55 AA 32` every 3 s and
  parses the reply.

The full protocol (frames, bytes, state codes) is in
[`docs/protocol-uart.en.md`](docs/protocol-uart.en.md). It was obtained by reverse
engineering and verified on a real EGA1800.

## What you get in Home Assistant

| Entity              | Type            | What it does                                  |
|---------------------|-----------------|-----------------------------------------------|
| Gate                | `cover` (gate)  | open / close / stop, with % open              |
| Pedestrian          | `button`        | presses the pedestrian channel (one leaf)     |
| Vehicle             | `button`        | presses the full-gate channel                 |
| Battery             | `sensor` (V)    | backup battery voltage                        |
| Battery present     | `binary_sensor` | whether a backup battery is connected         |
| Model / Firmware    | `text_sensor`   | what the STM32 reports (e.g. EGA1800 / 1.3.0) |

## Installation

1. **Install ESPHome** (Home Assistant add-on, or the
   [CLI](https://esphome.io/guides/installing_esphome)).
2. **Back up the stock firmware** (see above). Don't skip it.
3. Copy the example files:
   ```bash
   cp esp-ega1800.example.yaml esp-ega1800.yaml
   cp secrets.yaml.example secrets.yaml
   ```
4. Edit `secrets.yaml` with your WiFi and an API key (instructions are inside the
   file itself).
5. Build and flash. The **first time** must be over **serial cable** (3.3 V
   USB-UART adapter to the ESP, GPIO9 to GND):
   ```bash
   esphome run esp-ega1800.yaml
   ```
   Later updates go **over WiFi (OTA)**, no need to open the enclosure.
6. Add the device in Home Assistant: **Settings → Devices & Services** → it should
   appear via auto-discovery (ESPHome).

## Important notes

- The module is an **ESP32-C2** with a **26 MHz crystal**: the YAML already ships
  the required `sdkconfig_options`. Without them, WiFi won't connect.
- Keep `power_save_mode: none` on WiFi: the C2 is single-core and power-save
  disrupts the UART.
- Control is **step by step** (each press cycles open→stop→close). The YAML's
  `cover` already has the logic so HA's open/close/stop do the expected thing.
- The STM32 **does not report state on its own**: it must be polled. The YAML polls
  every 3 s.

## Other models

The control command usually works the same, but the **state-frame layout may
differ** between variants (EGA18, EGB18, EGA15, EGB1900…). If you convert another
model, see the last section of
[`docs/protocol-uart.en.md`](docs/protocol-uart.en.md) to capture and adjust the
byte offsets.

## Contributing

Did you get it working on your gate, convert another variant, or find something to
improve? You're welcome: **open an issue** to report a problem, ask a question or
share frame captures from another model, or **send a pull request** with your
changes. Every contribution helps more people move their gate to local control.

## License

[MIT](LICENSE).

## Disclaimer

This involves reflashing hardware that controls a motorized gate. You do it at your
own risk: you can brick the module (hence the backup), and a misconfigured gate can
be dangerous. Always test with the gate's path clear and the power cutoff within
reach. Not affiliated with XHouse/Giigle.
