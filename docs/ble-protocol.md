# BLE protocol — NFC Archiver (public)

Public wireless interface for **NFC Archiver** (RFIDfriend). Suitable for Magic NFC and third-party implementers.

Endianness: multi-byte integers on the command/response wire are **little-endian (LE)** unless noted. Unlock magic in device info is a hex string; the unlock **command** sends the 16-bit value as **high byte then low byte**.

## Device identity

| Field | Value |
|-------|--------|
| BLE advertised name | `NFC Archiver` |
| Manufacturer | RFIDfriend.com |
| Product | NFC Archiver |

## GATT — NFC service

UUID prefix: `47414D42-5249-5553-4E46-4341524348`

| Role | UUID |
|------|------|
| Service | `47414D42-5249-5553-4E46-4341524348A0` |
| Command (write) | `47414D42-5249-5553-4E46-4341524348A1` |
| Response (notify / read) | `47414D42-5249-5553-4E46-4341524348A2` |
| Status (read) | `47414D42-5249-5553-4E46-4341524348A3` |

Enable notifications on the **Response** characteristic (`CCC` / BLE2902) before issuing commands.

### Status characteristic (JSON)

UTF-8 JSON, example:

```json
{"rf":false,"dumper":true,"ble":true,"unlocked":false,"dumps":3}
```

| Field | Type | Meaning |
|-------|------|---------|
| `rf` | bool | NFC field on |
| `dumper` | bool | Dumper mode enabled |
| `ble` | bool | BLE connected |
| `unlocked` | bool | Unlock window active |
| `dumps` | number | Stored dump count |

## GATT — Info / debug / capabilities

| Role | UUID |
|------|------|
| Service | `47414D42-5249-5553-4E46-4341524348B0` |
| Device info (JSON, read) | `47414D42-5249-5553-4E46-4341524348B1` |
| Debug status (JSON, read/notify) | `47414D42-5249-5553-4E46-4341524348B2` |
| Capabilities (JSON, read) | `47414D42-5249-5553-4E46-4341524348B3` |

### Device info JSON

```json
{
  "name": "NFC Archiver",
  "fw": "0.2.0+260723001",
  "build": "260723001",
  "hw": "ESP32-WROOM+PN5180",
  "serial": "…",
  "mac": "…",
  "unlockMagic": "0xCDEF",
  "maxDumps": 512
}
```

`unlockMagic` is required for unlock / OTA / reset. Parse as a 16-bit hex value (e.g. `"0xCDEF"` → bytes `0xCD`, `0xEF`).

### Capabilities JSON

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

- Missing flag ⇒ treat feature as unavailable.
- `"bleOta": true` — BLE OTA opcodes `0x20`–`0x23` are supported.
- `"iso15693InventoryAll": true` — command `0x11` / response `0x91` available.

### Debug JSON (selected fields)

Includes boot state, `rf`, `pn`, `dumper`, `pwCount`, `dumps`, `batt`, counters (`rx`/`tx`/`ok`/`err`/`to`), `last` status, `unlocked`, `fw`, `build`, `serial`, `mac`, `up` (uptime seconds).

## Standard services

| Service | UUID | Notes |
|---------|------|--------|
| Device Information (DIS) | `0x180A` | Manufacturer `0x2A29`, model `0x2A24`, FW rev `0x2A26`, serial `0x2A25`, HW rev `0x2A27` |
| Battery Service (BAS) | `0x180F` | Battery Level `0x2A19` (uint8 percent, read/notify) |

## Framing

- **Command**: write to Command characteristic. First byte = opcode; remaining bytes = payload.
- **Response**: notify on Response characteristic. Typical: `[responseOpcode][status][…]`.
- Prefer write **with response** for reliability; respect ATT MTU for large OTA chunks.

## Status codes

| Code | Name | Meaning |
|------|------|---------|
| `0x00` | OK | Success |
| `0x01` | General error | Bad length, failed operation, etc. |
| `0x02` | Hardware not ready | PN5180 / boot not ready |
| `0x03` | Field off | RF required but off |
| `0x04` | Timeout / no tag | No card or RF timeout |
| `0x05` | Locked | Unlock required or wrong magic |
| `0x06` | Full | Storage full |

Unknown opcodes may respond with opcode `0xFF` and status `0x01`.

## Commands (app → device)

| Opcode | Name | Unlock? | Payload | Response |
|--------|------|---------|---------|----------|
| `0x01` | Field on | No | — | `0x81` `[status]` |
| `0x02` | Field off | No | — | `0x82` `[status]` |
| `0x03` | Reset | **Yes** | — | `0x83` `[status]` then reboot |
| `0x04` | Unlock | No | `magicHi`, `magicLo` (2 bytes) | `0x84` `[status]` |
| `0x05` | Set dumper mode | No | `enabled` (`0`/`≠0`) | `0x85` `[status][enabled]` |
| `0x06` | Set password list | No | `count` + `count×4` password bytes | `0x86` `[status][count]` |
| `0x07` | Get dumps | No | — | `0x97` meta then `0x98` chunks |
| `0x08` | Clear dumps | No | — | `0x88` `[status][removed u16 LE]` |
| `0x10` | Raw ISO 15693 | No | `len` + `len` request bytes | `0x90` `[status][len][data…]` |
| `0x11` | Inventory all | No* | optional — see below | `0x91` — see below |
| `0x20` | OTA begin | **Yes** | `u32 size LE`, `u32 crc32 LE` | `0xA0` — see OTA |
| `0x21` | OTA data | **Yes** | `u32 offset LE` + data | `0xA1` — see OTA |
| `0x22` | OTA abort | **Yes** | — | `0xA2` `[status]` |
| `0x23` | OTA commit | **Yes** | — | `0xA3` `[status]` then reboot |

\* `0x11` only if capability `iso15693InventoryAll` is true.

### Unlock (`0x04`)

```
TX: 04 | magicHi | magicLo
RX: 84 | status
```

Unlock remains valid for a limited window (~30 s). Disconnect clears unlock. Required for **reset** and **OTA**.

### Dumper (`0x05`)

```
TX: 05 | enabled
RX: 85 | status | enabledConfirmed
```

### Passwords (`0x06`)

```
TX: 06 | count | pw0[4] | pw1[4] | …
RX: 86 | status | count
```

`count` ≤ 16. Total write length must be `2 + count×4`.

### Get dumps (`0x07`)

No immediate simple ACK. Device notifies:

**Meta `0x97`:**

```
97 | dumpCount_lo | dumpCount_hi | maxDumps_lo | maxDumps_hi
```

**Chunk `0x98`:**

```
98 | dumpIndex_lo | dumpIndex_hi | chunkIndex | totalChunks | partLen | partBytes…
```

Concatenate `partBytes` per dump index (UTF-8 JSON). Chunks are small (compatible with default ATT MTU).

### Clear dumps (`0x08`)

```
RX: 88 | status | removed_lo | removed_hi
```

### Raw ISO 15693 (`0x10`)

```
TX: 10 | len | request[len]
RX: 90 | status | len | response[len]
```

Requires RF field on. Response payload capped (implementation typically ≤ 128 bytes).

### Inventory all (`0x11`) — optional

Allowed write lengths (including opcode): **1**, **2**, or **3** bytes.

| Length | Layout |
|--------|--------|
| 1 | `11` — firmware defaults |
| 2 | `11 \| max_tags` |
| 3 | `11 \| max_tags \| inventory_flags` |

Response:

```
91 | status | count | { DSFID | UID[8] } × count
```

Each UID is **8 bytes, LSB first**. On OK with `count == 0`, frame is at least three bytes.

## BLE OTA (`0x20`–`0x23`)

Requires unlock and capability `"bleOta": true`. Transfer the **application** image (`nfc-archiver.bin`), not the merged factory image.

**Note:** BLE OTA is **slow**. Prefer USB Web Serial for full installs and large updates.

CRC32: IEEE / zlib polynomial over the entire image (same family as typical ESP image tooling — match the publisher’s `version.json` / CI). Confirm against published artifacts if you implement your own packager.

### `0x20` OTA begin

```
TX: 20 | size[4] LE | crc32[4] LE
RX: A0 | status | maxChunk[2] LE
```

On success, `maxChunk` is the maximum data bytes allowed per subsequent `0x21` write (not counting opcode + offset). If locked: status `0x05`.

### `0x21` OTA data

```
TX: 21 | offset[4] LE | data…
RX: A1 | status | offset[4] LE
```

`offset` is the absolute byte offset into the image. Keep `len(data) ≤ maxChunk`. Response echoes the offset for that chunk (or the next expected offset — treat non-OK status as failure and abort).

### `0x22` OTA abort

```
TX: 22
RX: A2 | status
```

Cancels an in-progress OTA session.

### `0x23` OTA commit

```
TX: 23
RX: A3 | status
```

On success the device validates size/CRC and **reboots** into the new image. On failure it stays on the previous app (status ≠ `0x00`).

### Suggested host sequence

1. Read info + capabilities; ensure `bleOta`.
2. Unlock (`0x04`).
3. `0x20` begin with image size and CRC32.
4. Stream `0x21` chunks until complete; respect `maxChunk` and unlock window (re-unlock if needed).
5. `0x23` commit; wait for reboot / reconnect.
6. On error at any step: `0x22` abort.

## UID byte order

ISO 15693 UIDs in inventory / dump JSON hex strings follow the PN5180 path: typically **LSB first** on the wire. Convert to MSB only for display if your UX expects that.

## Change process (implementers)

1. Read capabilities before using optional opcodes.
2. Keep status-code handling central.
3. Do not assume MTU > 23 unless negotiated; dump chunks and OTA `maxChunk` already account for limits.
