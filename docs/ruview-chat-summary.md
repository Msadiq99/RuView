# RuView — Quick Run & Diagnostics (paste to ChatGPT)

Purpose
- Short, copy-pasteable instructions to run the RuView demo, host the sensing HTTP API, and verify the dashboard.

What this does
- Runs `examples/ruview_live.py` in demo mode (no serial devices) and writes `/tmp/ruview-last-feature.json` every 1s.
- Hosts a tiny HTTP API + dashboard at `http://<HOST_IP>:30000/` served by `scripts/ruview-sensing-server.py`.

Commands (run on the host machine)

1. Install runtime dependency (pyserial):

```bash
python3 -m pip install --user pyserial
```

2. Start the sensing server (binds to all interfaces on port 30000):

```bash
# from repo root
PORT=30000 python3 scripts/ruview-sensing-server.py
```

3. Run the RuView live demo writer (writes /tmp/ruview-last-feature.json every 1s):

```bash
# demo (no hardware)
python3 examples/ruview_live.py --csi none --mmwave none --duration 0 --interval 2 --mode vitals
```

Notes:
- Using `--duration 0` keeps the script running until you Ctrl-C; change to a number for seconds.
- To use real devices, replace `--csi none` with the serial path (macOS examples: `/dev/cu.usbmodem1101` or `/dev/tty.usbserial-XXXX`).

Verify the server and data

1. Check health (returns `feature_age_s`):

```bash
curl http://<HOST_IP>:30000/health
```

2. Get the latest sensing JSON:

```bash
curl http://<HOST_IP>:30000/api/v1/sensing/latest
```

3. Open dashboard in browser:

- `http://<HOST_IP>:30000/` (simple polling UI)

Why you might see "No data"
- The feature file `/tmp/ruview-last-feature.json` is missing or stale (older than the server's staleness limit, default 10s).
- The ruview writer is not running or is not writing to the expected path. Ensure `RUVIEW_FEATURE_JSON` environment variable is consistent.

Quick troubleshooting steps

```bash
# ensure the feature file exists and is fresh
ls -l /tmp/ruview-last-feature.json
cat /tmp/ruview-last-feature.json
# restart the sensing server on port 30000 if needed
pkill -f ruview-sensing-server.py || true
PORT=30000 python3 scripts/ruview-sensing-server.py &
# start the demo writer
pkill -f ruview_live.py || true
python3 examples/ruview_live.py --csi none --mmwave none --duration 0 --mode vitals &
```

Security & network
- The server exposes `Access-Control-Allow-Origin: *` for convenience on local networks. If exposing outside your LAN, add TLS and authentication (Caddy/NGINX or a tunnel with auth).

Flash a supported ESP32 board
- Supported boards: **ESP32-S3 with 8 MB flash** or **ESP32-C6**.
- Unsupported board: **ESP32-C3 / Core 3** will not run the CSI firmware.

1. Verify the board type:

```bash
python3 -m pip install --user esptool
python3 -m serial.tools.list_ports
PORT=/dev/cu.usbmodem1101
python3 -m esptool --port "$PORT" chip_id
```

If it reports `esp32c3` or `esp32`, stop and switch to ESP32-S3 or ESP32-C6. `esp32-csi-node` does not support those chips.

Boot check (verify board is alive):

```bash
python3 -m serial.tools.miniterm "$PORT" 115200
```

Press `Ctrl+]` to exit. You should see boot messages like `ets Jun  8 2016 00:22:57` or `ESP32-S3 CSI Node`. If nothing appears, the board is not responding — try a different cable, port, or board.

Common failure modes before flashing:
- wrong serial port: use `python3 -m serial.tools.list_ports` and choose the device that appears when you plug in USB.
- chip type mismatch: `esp32c3` / `esp32` will never run this firmware.
- USB driver / cable issue: try a different cable or USB port if the board does not appear.

2. Erase the board flash (optional, only if you want a clean reinstall):

```bash
PORT=/dev/cu.usbmodem1101
python3 -m esptool --port "$PORT" erase_flash
```

3. Reset the board:

- Unplug the USB cable, wait 2 seconds, then plug it back in.
- If your board has a reset button, press it once.

4. Recover the board with a simple known-good image (recommended after erase):

```bash
cd firmware/esp32-hello-world
idf.py set-target esp32c3 build
idf.py -p "$PORT" flash monitor
```

If you are on ESP32-C3, do not reinstall `esp32-csi-node` — that firmware is unsupported on C3. Instead use the hello-world image to confirm the board is alive, then switch to ESP32-S3 or ESP32-C6 for the CSI node.

5. Flash prebuilt firmware (fastest, ESP32-S3):

```bash
PORT=/dev/cu.usbmodem1101
python3 -m esptool --chip esp32s3 --port "$PORT" --baud 460800 \
  write_flash --flash_mode dio --flash_size 8MB \
  0x0     firmware/esp32-csi-node/release_bins/bootloader.bin \
  0x8000  firmware/esp32-csi-node/release_bins/partition-table.bin \
  0xf000  firmware/esp32-csi-node/release_bins/ota_data_initial.bin \
  0x20000 firmware/esp32-csi-node/release_bins/esp32-csi-node.bin
```

If you have an ESP32-C6 node, build from source and then flash it using `idf.py set-target esp32c6`.

3. Provision WiFi credentials:

```bash
python3 firmware/esp32-csi-node/provision.py --port "$PORT" \
  --ssid "YourSSID" --password "YourPass" --target-ip 192.168.1.20
```

4. Start the sensing server and open the dashboard:

```bash
PORT=30000 python3 scripts/ruview-sensing-server.py
```

Then visit `http://<HOST_IP>:30000/`.

If you prefer to build from source with ESP-IDF:

```bash
cd firmware/esp32-csi-node
idf.py set-target esp32s3 build
# or for ESP32-C6:
# idf.py set-target esp32c6 build
idf.py -p /dev/cu.usbmodem1101 flash monitor
```

Optional next steps (pick one)
- Make the dashboard prettier and add live charts (client-side JS).
- Run with real devices and confirm `examples/ruview_live.py` opens the serial ports.
- Expose via a secure tunnel (ngrok) for remote testing.

---

If you want a different format or extra fields (HR/BR example JSON, launchd plist, or ngrok config), tell me which and I'll add it.