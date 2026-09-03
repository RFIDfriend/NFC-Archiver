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
| 1 | Silicon diode **1N4007** (or equivalent ~0.6–0.7 V drop) | Required for direct battery power — see [Power supply](#power-supply) |
| — | Solder bridge / short jumper | Bridge PN5180 **3.3 V** ↔ **5 V** pins — see [Power supply](#power-supply) |
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

## Power supply

These steps are **required** for a reliable start of common PN5180 breakout boards with this ESP32 setup.

### 1. Bridge PN5180 `3.3V` and `5V` pins

On most PN5180 modules the **3.3 V** and **5 V** supply pins sit next to each other. For the NFC Archiver, feed the board from the ESP32 **3.3 V** rail only and **short those two pins** (solder bridge or wire jumper), so **3.3 V is applied to both**.

| Without bridge | With bridge |
|----------------|-------------|
| Often only one supply domain is powered → module does not boot / RF never comes up | Both module supply pins see 3.3 V → board starts |

Connect:

```
Wemos 3.3V  ──►  PN5180 3.3V  ──(short)──  PN5180 5V
Wemos GND   ──►  PN5180 GND
```

SPI and control lines stay at **3.3 V logic** (see pinout above). Do **not** feed 5 V into ESP32 GPIOs.

> **Note:** Dedicated 5 V on the module’s `5V` pin can improve RF range when a true 5 V rail is available (e.g. USB host). For battery / ESP32‑only builds, bridging both pins to **3.3 V** is the working configuration used here.

### 2. Battery power — series silicon diode (1N4007)

A full LiPo can sit around **4.0–4.2 V** (e.g. 4.09 V). That is still often “tolerated” by the ESP32 for a while, but it is **above the ~3.6 V limit** of the PN5180 digital/interface supply. Direct battery → Wemos **3.3 V** pin then causes the PN5180 to fail to start or to behave erratically.

Place a **silicon diode (1N4007)** in series between the battery positive terminal and the Wemos **3.3 V** pin. Forward drop is typically **~0.6–0.7 V**, so ~4.09 V becomes ~3.3–3.5 V on the shared 3.3 V rail (ESP32 + bridged PN5180).

```
Battery (+) ──►|── (1N4007, stripe/cathode toward ESP32) ──► Wemos 3.3V
Battery (−) ───────────────────────────────────────────────► Wemos GND
```

| Polarity | Meaning |
|----------|---------|
| Anode (no stripe) | Toward **battery +** |
| Cathode (stripe) | Toward **Wemos 3.3 V** |

Also connect the battery sense path to **GPIO34** (via your divider) as before.

**USB flashing / USB power:** You can leave the diode in place for battery operation; when running from USB alone, the Wemos onboard regulator supplies 3.3 V. Do not connect a charging pack in a way that fights USB 5 V without a proper charge/protection board.

**Alternatives (optional):** Power the pack into **5 V / VIN** through a proper LiPo charge module so the onboard LDO regulates 3.3 V. The series **1N4007** into the **3.3 V** pin remains the documented fix for direct‑to‑rail battery wiring used with this project.

## Notes

- Logic levels are **3.3 V**. Do not drive the ESP32 or PN5180 with 5 V IO.
- GPIO34 is input-only — suitable for ADC, not for outputs.
- Keep SPI wires short; poor wiring shows up as flaky NFC or bus hangs.
- USB on the Wemos is used for the [Web Serial flasher](../flash/) and serial logs (typically 115200 baud).
- If the ESP32 boots but NFC never works: confirm the **3.3 V↔5 V bridge** on the PN5180 and, for battery packs, the **1N4007** drop — see [Power supply](#power-supply).

## Related

- [features.md](features.md) — software features that use this hardware
- [flashing.md](flashing.md) — how to load firmware
- [ble-protocol.md](ble-protocol.md) — wireless API once flashed
