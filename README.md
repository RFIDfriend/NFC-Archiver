# NFC Archiver

**RFIDfriend** hardware companion: an ESP32 + NXP PN5180 BLE ISO 15693 reader/writer for [Magic NFC](https://github.com/RFIDfriend) and third-party apps.

> **Kurz (DE):** NFC Archiver ist ein BLE-fähiger ISO-15693-Reader/Writer (ESP32 + PN5180) für die Magic-NFC-App und eigene Integrationen.

## What it is

NFC Archiver turns a **Wemos D1 mini ESP32-WROOM** board with a **PN5180** NFC frontend into a wireless ISO 15693 (Vicinity / SLIX-class) tool over Bluetooth Low Energy. Use it with the Magic NFC iOS app, or implement the public BLE protocol in your own client.

This repository includes:

- Hardware and feature documentation
- The BLE protocol for third-party implementers
- A browser-based Web Serial flasher
- Released firmware binaries (CI artifacts)

## Features

- BLE control of the NFC RF field (on / off)
- Transparent raw ISO 15693 command path
- Optional multi-tag inventory (capability-gated)
- Autonomous **dumper mode** with on-device dump storage
- Privacy password list for protected tags
- Battery level (BAS) and Device Information (DIS)
- WS2812 status LED
- Firmware updates via **desktop Web Serial** or **BLE OTA** (Magic NFC app)

See [docs/features.md](docs/features.md) for details.

## Documentation

| Doc | Description |
|-----|-------------|
| [docs/hardware.md](docs/hardware.md) | BOM, pinout, wiring |
| [docs/features.md](docs/features.md) | Feature reference |
| [docs/ble-protocol.md](docs/ble-protocol.md) | Full public BLE GATT / command protocol |
| [docs/flashing.md](docs/flashing.md) | Web Serial + BLE OTA flashing |
| [flash/](flash/) | Browser flasher (GitHub Pages) |
| [firmware/](firmware/) | Released binaries + `version.json` |

## Flash via Web Serial

1. Open **[https://rfidfriend.github.io/NFC-Archiver/flash/](https://rfidfriend.github.io/NFC-Archiver/flash/)** in **Chrome** or **Edge** (desktop).
2. Connect the board over USB.
3. Click **Install** and follow the Espressif web installer prompts.

Details: [docs/flashing.md](docs/flashing.md). Local copy: [flash/index.html](flash/index.html).

## Firmware updates

| Method | Where | Binary | Notes |
|--------|--------|--------|--------|
| **Web Serial** (USB) | Desktop browser → [flash/](flash/) | `nfc-archiver.factory.bin` (merged, offset `0`) | Full flash / first install; erases as prompted |
| **BLE OTA** | Magic NFC app (unlocked session) | `nfc-archiver.bin` (app image) | Slow over BLE; unlock required; see protocol OTA opcodes `0x20`–`0x23` |

## Version source

Published releases are described by:

- [`firmware/version.json`](firmware/version.json) — version, build id, changelog, artifact names
- [`firmware/nfc-archiver.bin`](firmware/nfc-archiver.bin) — application image (OTA / app-only)
- [`firmware/nfc-archiver.factory.bin`](firmware/nfc-archiver.factory.bin) — merged factory image for USB install (bootloader + partitions + app)

Binaries appear after CI publishes them. Until then, `firmware/` may contain only placeholders — see [firmware/README.md](firmware/README.md).

## License

Documentation and web flasher assets: **MIT** (see [LICENSE](LICENSE)).

Firmware binaries: provided **AS IS** by RFIDfriend — proprietary / as-is; no warranty.
