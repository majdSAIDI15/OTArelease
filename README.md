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

## Audio bank (`sdcard/` + `audio_manifest.json`)

The device self-heals its SD audio files (French voice prompts). On boot / WiFi
reconnect it reads `audio_manifest.json` (served **raw**, so these files ARE
tracked in git — unlike the firmware `.bin` which is a release asset) and
re-downloads any file that is missing or the wrong size to the SD card.

- **Layout:** the WAVs live under `sdcard/`, mirroring the device paths.
  `sdcard/validation.wav` → `/sdcard/validation.wav`,
  `sdcard/audio/*.wav` → `/sdcard/audio/*.wav`.
- **`audio_manifest.json`** (repo root) lists each file's `path` + expected
  `size` (bytes). `base_url` + `path` = the raw download URL. `size` is the
  integrity gate (catches a truncated download); an optional `sha256` per file
  is verified when present.
- Consumed by `main/services/audio_provision` in the firmware
  (`AUDIO_MANIFEST_URL`).
- **To change a prompt:** replace the WAV under `sdcard/`, update its `size` in
  `audio_manifest.json`, commit + push. Devices pick it up on the next check.

```json
{ "base_url": "https://raw.githubusercontent.com/majdSAIDI15/OTArelease/main/sdcard/",
  "files": [ { "path": "validation.wav", "size": 373326 },
             { "path": "audio/remind.wav", "size": 57644 } ] }
```
