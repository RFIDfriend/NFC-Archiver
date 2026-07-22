# Firmware artifacts

This folder holds **CI-published binaries** for NFC Archiver.

| File | Role |
|------|------|
| `version.json` | Version, build id, changelog, artifact names |
| `nfc-archiver.bin` | Application image (BLE OTA / app-only updates) |
| `nfc-archiver.factory.bin` | Merged factory image (bootloader + partitions + app) for USB Web Serial at offset `0` |

## Placeholder note

Until the first CI run uploads release artifacts, only `version.json` and this README may be present. The `.bin` files appear after CI publishes them to this path.
