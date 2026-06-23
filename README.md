# PillQare Firmware Releases (PUBLIC)

Public hosting for PillQare OTA firmware. The device polls `manifest.json` and
downloads the `.bin` it points to (a GitHub **release asset**).

## ⚠️ Security rules

- **The `.bin` published here MUST be a GENERIC build** (built with
  `main/device_config.h.generic` — placeholder uuid/key). NEVER publish a `.bin`
  built with a real `device_config.h` → it would leak one device's API key into
  the image flashed on every device.
- **NEVER** put a `.elf` here. ELFs contain debug symbols → keep them PRIVATE in
  `../firmware_elfs/` (not pushed anywhere public). They are required to decode
  crash coredumps of that exact version.
- The `.bin` is verified on-device by the ESP image's appended **SHA-256**
  (integrity). App signing is deferred to a later phase (before production).

## Manual release process (per version)

From the firmware repo (`hardwar-pillquare/pillqv2`):

1. Bump `PILLQARE_FW_VERSION` in `main/device_config.h` (and in the generic template).
2. `cp main/device_config.h.generic main/device_config.h`   # GENERIC build
3. `idf.py build`  → produces `build/<project>.bin` and `build/<project>.elf`
4. `sha256sum build/<project>.bin`   (or `Get-FileHash` on Windows)
5. Update `manifest.json` here: `version`, `url`, `sha256`, `size_bytes`, notes.
6. Create the GitHub release with the `.bin` as an asset:
   `gh release create vX.Y.Z build/<project>.bin --repo <OWNER>/pillqare-firmware-releases`
7. **Archive the `.elf`** into `../firmware_elfs/vX.Y.Z.elf` (private).

### The 3 manual pitfalls
- SHA-256 not regenerated → device rejects the download.
- Firmware version ≠ manifest version → update not detected.
- `.elf` not archived → that version's coredumps become undecodable.

## manifest.json schema
```json
{ "version": "1.0.1",
  "url": "https://github.com/<OWNER>/pillqare-firmware-releases/releases/download/v1.0.1/pillqare.bin",
  "sha256": "<hex>",
  "is_critical": false,
  "release_notes": "…",
  "size_bytes": 0 }
```
Same shape as the CRM `/firmware` response → migrating to the CRM later only
changes the base URL the device polls (`OTA_MANIFEST_URL` in `ota_service.h`).
