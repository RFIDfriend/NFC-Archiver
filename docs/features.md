# Features — NFC Archiver

Feature reference for firmware **0.2.x** and the public BLE API. Capability flags are advertised over BLE; clients should check capabilities before assuming a feature exists.

## BLE NFC field control

Commands `0x01` (field on) and `0x02` (field off) enable or disable the PN5180 RF field.

- Field must be **on** before raw ISO 15693 traffic (`0x10`) succeeds; otherwise status `0x03` (field off).
- Connecting over BLE typically turns the field **off** so interactive sessions start clean.
- Status JSON (`rf`) and the LED reflect whether RF is active.

## Raw ISO 15693

Command `0x10` sends a transparent ISO 15693 request frame to the tag and returns the response bytes.

- Payload: `[len][request bytes…]`
- Response: `[0x90][status][len][response bytes…]`
- Suitable for inventory, system info, read/write blocks, vendor commands, etc.
- UIDs on the wire are typically **LSB first** (PN5180 path).

## Multi-tag inventory (optional)

When capability `iso15693InventoryAll` is true, command `0x11` runs an on-device multi-tag inventory and returns all found UIDs in one response (`0x91`). Prefer this over many raw inventory frames when several tags are in the field.

## Dumper mode

Autonomous dump when **not** connected over BLE:

- Enable/disable with `0x05` (`1` = on, `0` = off); persisted on device.
- While enabled and idle (no BLE), the device polls for tags, optionally tries privacy passwords, and stores dumps.
- LED: yellow when dumper is on and disconnected; green / purple for success / duplicate (see LED section).

## Dump storage

- On-device storage for captured dumps (up to **512** entries; see device info `maxDumps`).
- `0x07` — fetch all dumps (meta `0x97` + JSON chunks `0x98`).
- `0x08` — clear all dumps; response includes count removed.
- Status / debug JSON expose current `dumps` count.
- Status `0x06` (full) may apply when storage cannot accept more data.

Each dump chunk reassembles to JSON fields such as `seq`, `uid`, `dsfid`, `afi`, `icRef`, `numBlocks`, `bytesPerBlock`, `totalBytes`, `privacyPassword`, `memory` (hex).

## Passwords

Command `0x06` replaces the on-device privacy password list (up to **16** entries × **4** bytes).

- Used in dumper mode to unlock privacy-protected tags before dumping.
- Order = try order.
- Capability: `"passwords": true`.

## Battery

- Battery Service (BAS, UUID `0x180F`), characteristic Battery Level `0x2A19` (read + notify).
- ADC on **GPIO34**; percent also appears in debug JSON (`batt`).
- Capability: `"battery": true`.

## LED status (WS2812 on GPIO14)

Approximate meaning of the base colors (flashes override briefly):

| Situation | Color |
|-----------|--------|
| Boot (in progress) | Yellow blink |
| Idle, BLE disconnected, dumper **off** | White |
| Idle, BLE disconnected, dumper **on** | Yellow |
| BLE connected, RF off | Blue |
| BLE connected, RF on | Yellow |
| Dump in progress | Orange blink |
| Dumper success (tag still present) | Green |
| Dumper duplicate | Purple |
| Error / failure flash | Red |
| Short confirm flash | Green |

Capability: `"led": true`.

## Device Information (DIS) & Battery (BAS)

Standard BLE services:

| Service | UUID | Role |
|---------|------|------|
| Device Information | `0x180A` | Manufacturer, model, firmware, serial, hardware string |
| Battery Service | `0x180F` | Battery level percent |

Custom info/debug/capabilities live under the RFIDfriend info service (`…B0`) — see [ble-protocol.md](ble-protocol.md).

## Unlock & reset

- Sensitive operations (e.g. reset, BLE OTA) require **unlock** (`0x04`) with a device-specific magic from the info JSON (`unlockMagic`).
- Unlock window is time-limited (~30 s); disconnect locks again.
- Reset (`0x03`) reboots the ESP32 after a successful unlocked request.

## OTA (firmware update)

Two update paths:

1. **USB Web Serial** — full/merged factory image via the browser flasher ([flashing.md](flashing.md)).
2. **BLE OTA** — app image over BLE (`0x20` begin → `0x21` data → `0x23` commit), unlock required. Capability: `"bleOta": true`.

BLE OTA is intentionally **slow**; prefer USB for large or frequent updates.

## Capabilities JSON

Example shape (fields may grow):

```json
{
  "iso15693": true,
  "raw": true,
  "iso14443a": false,
  "battery": true,
  "led": true,
  "debug": true,
  "reset": true,
  "unlock": true,
  "dumper": true,
  "passwords": true,
  "dumpStore": true,
  "iso15693InventoryAll": true,
  "bleOta": true
}
```

Treat missing flags as **unsupported**.
