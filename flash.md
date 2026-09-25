# Building and distributing firmware

How to produce a `.bin` other people can flash to their own ESP32, and how they
flash it.

Nothing user-specific is compiled in — Wi-Fi credentials, the ntfy server,
topic, login, both passphrases and the watchlist live in NVS on the device,
entered through the setup portal on first boot. That is what makes the binary
shareable.

---

## 1. Prerequisites (you, the builder)

[arduino-cli](https://arduino.github.io/arduino-cli/) plus the ESP32 core
(3.x — 3.3.10 tested) and one library:

```bash
arduino-cli core install esp32:esp32
```

```bash
arduino-cli lib install ArduinoJson
```

ArduinoJson must be **7.x**. `WebServer`, `DNSServer`, `WiFiClientSecure`,
`HTTPClient` and `Preferences` all ship with the core.

## 2. Build

Bump `SG_VERSION` in `StockGraberAlerts/StockGraberAlerts.ino` first, then:

```bash
arduino-cli compile --clean --fqbn "esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs" --export-binaries StockGraberAlerts
```

The artifacts land in `StockGraberAlerts/build/esp32.esp32.esp32/`:

| File | Size | Purpose |
| --- | --- | --- |
| `StockGraberAlerts.ino.bin` | ~1.1 MB | the application |
| `StockGraberAlerts.ino.bootloader.bin` | 25 KB | second-stage bootloader |
| `StockGraberAlerts.ino.partitions.bin` | 3 KB | partition table |
| `boot_app0.bin` | 8 KB | OTA selector |
| `StockGraberAlerts.ino.merged.bin` | 4 MB | all of the above, pre-combined |

The application is 58% of the 1.9 MB app partition. `--export-binaries` is what
writes these into the sketch folder; without it arduino-cli builds into a
temporary directory and throws the result away.

The partition scheme (`min_spiffs`, see README §3) is baked into
`partitions.bin`, so it travels with the firmware as long as whoever flashes it
writes the partition table too — both methods below do.

The FQBN targets a **4 MB** board (ESP32 DevKitC and most ESP-WROOM-32
modules). `merged.bin` is a whole-flash image and assumes 4 MB; rebuild with a
matching `FlashSize` for anything else.

### Package the release

```bash
mkdir -p dist
```

```bash
cp StockGraberAlerts/build/esp32.esp32.esp32/StockGraberAlerts.ino.merged.bin dist/stockgraber-v1.0.0-esp32-4mb.bin
```

```bash
cd dist && shasum -a 256 stockgraber-v1.0.0-esp32-4mb.bin > stockgraber-v1.0.0-esp32-4mb.bin.sha256
```

## 3. Confirm the binary carries no secrets

Do this before publishing:

```bash
strings dist/stockgraber-v1.0.0-esp32-4mb.bin | grep -iE "your-ssid|your-wifi-password|your-topic|your-passphrase"
```

Substitute your own network name, password, topic and passphrase. Expect **no
output**. Grep for the actual values, not the word "password" — the settings
form contains `<input type=password …>` markup.

Sanity-check that `strings` actually read the file. This should print `1`:

```bash
strings dist/stockgraber-v1.0.0-esp32-4mb.bin | grep -c "StockGraber Alerts v"
```

---

## 4. Instructions for the person flashing

They need [esptool](https://github.com/espressif/esptool) — no Arduino
toolchain, no source:

```bash
pip install esptool
```

### Find the port

| OS | Typically |
| --- | --- |
| macOS | `/dev/cu.usbserial-0001`, `/dev/cu.SLAB_USBtoUART` or `/dev/cu.wchusbserial*` |
| Linux | `/dev/ttyUSB0` |
| Windows | `COM3`, `COM4`, … |

If nothing appears, the USB-serial driver is missing — CP210x for Silicon Labs
bridges, CH34x for WCH. A charge-only USB cable produces the same symptom, so
try another cable before chasing drivers.

### Verify the download

```bash
shasum -a 256 -c stockgraber-v1.0.0-esp32-4mb.bin.sha256
```

### Erase, then write

Erasing first avoids inheriting stale settings from whatever was on the board
before. **Skip the erase when upgrading a StockGraber node** — it would wipe the
settings and watchlist.

```bash
esptool --port /dev/cu.usbserial-0001 erase-flash
```

```bash
esptool --port /dev/cu.usbserial-0001 --baud 460800 write-flash 0x0 stockgraber-v1.0.0-esp32-4mb.bin
```

The merged image already contains the bootloader (`0x1000`), partition table
(`0x8000`), OTA selector (`0xe000`) and application (`0x10000`). Writing the
bare application to `0x0` instead produces a board that does not boot.

Older esptool (v4 and earlier) uses underscores: `erase_flash`, `write_flash`.
If the connect stalls at `Connecting....____`, hold **BOOT**, tap **EN**,
release **BOOT** and retry.

### First boot

1. Join the Wi-Fi network **`StockGraber-Setup-XXXX`** (suffix differs per
   board).
2. The setup page should open automatically. If not, browse to
   **http://192.168.4.1**.
3. Fill in the form → **Save & restart** (details in README §4.1). Wi-Fi is
   **2.4 GHz only**; give each node its **own** ntfy topic.

The node joins the network and publishes a startup notice to the topic:

```
StockGraber - node started
Reset: power-on
IP: 192.168.1.42
Settings: http://192.168.1.42
Firmware: v1.0.0
Watchlist: 4 of 50 symbols
Heap: 158 KB free
```

The settings page stays available at that address, protected by the admin
password (username **`admin`**). Send `<passphrase> help` to the topic for the
command list. Serial runs at **115200 baud** if anything goes wrong.

---

## 5. Notes

- **Reflashing does not erase settings.** NVS (`0x9000`, 20 KB) lives outside
  the application partition, so an upgrade keeps the configuration and the
  watchlist.
- **Returning a node to setup mode** needs no USB: press **EN**, release, then
  press **BOOT** within 2 s; or send `<passphrase> factoryreset` over ntfy
  (keeps the watchlist).
- **To wipe settings and watchlist over USB** without reflashing:

  ```bash
  esptool --port /dev/cu.usbserial-0001 erase-region 0x9000 0x5000
  ```
