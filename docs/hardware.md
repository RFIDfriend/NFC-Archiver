# Hardware — NFC Archiver

Reference hardware for the public NFC Archiver firmware image.

## Overview

| Item | Value |
|------|--------|
| Product | NFC Archiver |
| Manufacturer | RFIDfriend.com |
| MCU board | Wemos D1 mini ESP32-WROOM (`ESP32-WROOM`) |
| NFC frontend | NXP PN5180 |
| Status LED | WS2812 / WS2812B (1 pixel) |
| Power sense | Battery voltage via ADC |

## Bill of materials (BOM)

| Qty | Part | Notes |
|-----|------|--------|
| 1 | Wemos D1 mini ESP32-WROOM | 3.3 V logic, USB for flash / serial |
| 1 | PN5180 NFC reader module | ISO 15693 / Vicinity; SPI interface |
| 1 | WS2812B LED (or strip segment) | Single pixel on data GPIO |
| 1 | LiPo / battery + divider (or board-specific sense) | Fed into ADC GPIO34 |
| — | Wiring / headers | Match pinout below |
| — | Antenna | Module antenna or external antenna per PN5180 design |

Exact resistor values for battery sensing depend on your pack and divider; firmware expects a usable ADC reading on GPIO34.

## Pinout

SPI and control lines between **ESP32-WROOM (Wemos D1 mini)** and **PN5180**, plus LED and battery ADC:

| Signal | ESP32 GPIO | Notes |
|--------|------------|--------|
| SPI SCK | **18** | PN5180 SCK |
| SPI MOSI | **23** | PN5180 MOSI |
| SPI MISO | **19** | PN5180 MISO |
| PN5180 CS | **5** | Chip select |
| PN5180 BUSY | **4** | Busy / IRQ-related handshake |
| PN5180 RST | **16** | Reset |
| WS2812 data | **14** | Board label often **TMS** |
| Battery ADC | **34** | ADC1, input-only |

### Quick reference

```
SCK  18
MOSI 23
MISO 19
CS    5
BUSY  4
RST  16
LED  14
BATT 34
```

## Notes

- Logic levels are **3.3 V**. Do not drive the ESP32 or PN5180 with 5 V IO.
- GPIO34 is input-only — suitable for ADC, not for outputs.
- Keep SPI wires short; poor wiring shows up as flaky NFC or bus hangs.
- USB on the Wemos is used for the [Web Serial flasher](../flash/) and serial logs (typically 115200 baud on private builds).

## Related

- [features.md](features.md) — software features that use this hardware
- [flashing.md](flashing.md) — how to load firmware
- [ble-protocol.md](ble-protocol.md) — wireless API once flashed
