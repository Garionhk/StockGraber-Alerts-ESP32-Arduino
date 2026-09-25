# StockGraber Alerts v1.0.0 — ESP32

First release of the ESP32 node as a shareable binary.

## Highlights

- **No credentials in the firmware.** Wi-Fi, ntfy server/topic/login, command
  passphrase, admin password, report options and the watchlist are entered on a
  built-in web page and stored in flash (NVS). `sg_secrets.h` and the two-pass
  `FORCE_PROVISION` flash are gone.
- **Setup portal** — first boot opens the `StockGraber-Setup-XXXX` Wi-Fi with a
  captive page at `http://192.168.4.1`.
- **Settings page on the LAN** at the node's IP, login `admin`.
- **Wi-Fi fallback** — if the router is unreachable the setup network comes back
  (login required) while the node keeps retrying.
- **ntfy login** — username/password or access token, for protected servers.
- **`factoryreset`** command over ntfy; **BOOT-button** recovery on the board.
- Startup notice now includes the settings URL and firmware version.

## Download

| File | For |
| --- | --- |
| `stockgraber-v1.0.0-esp32-4mb.bin` | ESP32 (WROOM-32 / DevKitC), 4 MB flash — merged image, write at `0x0` |

SHA-256: `87ba46890b94cd275507446a233a775c1e6218d1200a19a36a87faf1406f4224`

```bash
esptool --port <PORT> --baud 460800 write-flash 0x0 stockgraber-v1.0.0-esp32-4mb.bin
```

Full instructions: `flash.md`.

## Upgrading a node set up with `sg_secrets.h`

Flash without erasing. Existing Wi-Fi, topic, passphrase and watchlist keep
working. The settings page will say *Settings locked* until an admin password is
set: press EN, release, press BOOT within 2 s, and save the form with a new
admin password.

## Build

ESP32 Arduino core 3.3.10, ArduinoJson 7.4.3,
`esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs` —
1,158,116 bytes (58% of the 1.9 MB app partition).

## Known limitations

- Build-verified; the web portal has not yet been exercised on hardware.
- Settings page is plain HTTP; the admin login crosses the LAN unencrypted.
- ntfy and Yahoo TLS certificates are not verified.
