# RuView Core 3 Firmware Fix

## Problem

If your board is an ESP32-C3 (Core 3), this repository's CSI firmware will not work.

The `wifi-densepose`/`ruview` firmware is designed for:
- **ESP32-S3** (recommended for CSI processing)
- **ESP32-C6** (supported in newer firmware builds)

## Why it fails

- The project documentation and hardware setup repeatedly state that **ESP32-C3 is not supported**.
- ESP32-C3 is a single-core chip and cannot run the CSI DSP pipeline required by `firmware/esp32-csi-node`.
- The repo includes an example `firmware/esp32-hello-world`, but the main CSI node firmware is built only for `esp32s3`/`esp32c6`.

## Evidence from the repo

- `CLAUDE.md` and setup docs explicitly say:
  - `ESP32-C3` is unsupported
  - `ESP32` and `ESP32-C3` are single-core and can’t run the CSI DSP pipeline
- `firmware/esp32-csi-node/README.md` only documents `esp32s3` and `esp32c6` build targets.

## Fix / recommendation

### Best fix
Use a supported board:
- `ESP32-S3` (8MB or 4MB)
- `ESP32-C6`

If you have a Core 3 device, it will not display the CSI firmware UI or respond correctly when flashed with `esp32-csi-node`.

### If you want to verify the board itself
Use the simple hello world firmware:

```bash
cd firmware/esp32-hello-world
idf.py set-target esp32c3 build
idf.py -p /dev/cu.usbmodemXXXX flash monitor
```

If that works, the board is alive, but it still cannot run the main RuView CSI firmware.

## If you want to keep using this repo
Use one of the supported boards and flash the correct target:

```bash
cd firmware/esp32-csi-node
idf.py set-target esp32s3 build
idf.py -p /dev/cu.usbmodemXXXX flash monitor
```

For C6:

```bash
cd firmware/esp32-csi-node
idf.py set-target esp32c6 build
idf.py -p /dev/cu.usbmodemXXXX flash monitor
```

## Important note

There is no quick patch in this repository that will make ESP32-C3 work with the CSI firmware.
Porting it would require a new single-core CSI DSP implementation and hardware-specific support for the board — that is a separate engineering effort.

## Quick check for your board

Use `esptool.py` or `idf.py` to confirm the chip type:

```bash
python3 -m pip install --user esptool
python3 -m esptool chip_id
```

If it reports `esp32c3`, you must switch to a supported board for the RuView firmware.
