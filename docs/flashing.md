# Flashing — NFC Archiver

Two supported ways to load firmware.

## 1. Desktop Web Serial (USB)

Best for **first install**, recovery, and full flashes.

### Requirements

- Desktop **Google Chrome** or **Microsoft Edge** (Web Serial API)
- USB cable to the Wemos D1 mini ESP32
- Drivers if your OS does not enumerate the board’s USB-UART chip

Safari and most mobile browsers are **not** supported for Web Serial.

### Steps

1. Open the flasher:
   - GitHub Pages: [https://rfidfriend.github.io/NFC-Archiver/flash/](https://rfidfriend.github.io/NFC-Archiver/flash/)
   - Or open [`../flash/index.html`](../flash/index.html) from a local/static host that can load `manifest.json` and the binary (relative paths).
2. Put the board in a flashable state (USB connected; hold BOOT if your board requires it for first connect).
3. Click **Install** on the page (Espressif [esp-web-tools](https://esphome.github.io/esp-web-tools/)).
4. Select the serial port and confirm. The installer may erase flash when `new_install_prompt_erase` is set.

### What gets written

The flash manifest uses the **merged factory** image at offset **0**:

| Artifact | Path | Offset | Use |
|----------|------|--------|-----|
| Factory (merged) | `firmware/nfc-archiver.factory.bin` | `0` | Web Serial / full USB install |
| App only | `firmware/nfc-archiver.bin` | (OTA partition) | BLE OTA / app updates |

A full ESP32 flash normally needs bootloader + partition table + app. CI publishes:

- `nfc-archiver.factory.bin` — merged image for offset `0` (preferred for esp-web-tools)
- `nfc-archiver.bin` — application image for OTA

Version metadata: [`../firmware/version.json`](../firmware/version.json).

See [`../flash/manifest.json`](../flash/manifest.json).

## 2. BLE OTA (Magic NFC app)

Best for updates when USB is inconvenient.

1. Pair/connect to **NFC Archiver** in the Magic NFC app.
2. Unlock the device (device-specific magic from BLE info).
3. Start a firmware update; the app streams `nfc-archiver.bin` using OTA opcodes `0x20`–`0x23` (see [ble-protocol.md](ble-protocol.md)).
4. Wait for commit and reboot. Keep the phone close; **BLE OTA is slow**.

Requirements:

- Firmware with capability `"bleOta": true`
- Unlocked session for the whole transfer (re-unlock if the window expires)
- Matching CRC32 / size as published in `version.json`

Third-party apps can implement the same OTA sequence.

## Troubleshooting

| Issue | Hint |
|-------|------|
| No serial port | Cable, drivers, another app holding the port |
| Install fails mid-way | Retry erase install; try a different USB port/cable |
| Device boots but no BLE | Confirm you flashed the RFIDfriend NFC Archiver image; check LED / battery |
| BLE OTA fails locked | Unlock first; check `bleOta` capability |
| BLE OTA CRC error | Use the exact `nfc-archiver.bin` from the same `version.json` release |

## Related

- [hardware.md](hardware.md) — board and pins
- [features.md](features.md) — OTA feature overview
- [ble-protocol.md](ble-protocol.md) — OTA command details
