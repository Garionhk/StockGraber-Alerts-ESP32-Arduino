# StockGraber Alerts — ESP32 (Arduino)

📺 Video walkthrough: https://youtu.be/AS1tXet5ko8

The C++ / Arduino version of the node: per-market end-of-day reports pushed to
ntfy, plus remote commands (`status`, `list`, `<CODE>`, `add`, `remove`, `help`)
answered from your phone. Same behaviour as the Raspberry Pi daemon.

> ⚠️ **Give this node its own ntfy topic** (e.g. `<your-topic>-esp`). On the
> Pi's topic you'd get duplicate reports and two bots answering every command.

## Prebuilt firmware

**v1.0.0** is in [`dist/`](dist/): `stockgraber-v1.0.0-esp32-4mb.bin`, a merged
image for a 4 MB ESP32 — no toolchain needed:

```
esptool --port /dev/cu.usbserial-0001 --baud 460800 write-flash 0x0 dist/stockgraber-v1.0.0-esp32-4mb.bin
```

Then set it up from your phone (§4.1). Checksum, release notes and the full
flashing guide: [`dist/RELEASE-v1.0.0.md`](dist/RELEASE-v1.0.0.md),
[`flash.md`](flash.md).

---

## 1. Hardware

Built for a plain **ESP32 Dev Module** (ESP32-WROOM-32 / DevKitC) — 4 MB flash,
**no PSRAM**.

| Spec | Value |
|---|---|
| SoC | ESP32-D0WD-V3 (or D0WDQ6) |
| Cores | Dual-core Xtensa LX6 @ 240 MHz |
| Flash | 4 MB quad SPI |
| PSRAM | **none** |
| SRAM | 520 KB total; ~320 KB DRAM usable as heap |
| Radio | Wi-Fi 2.4 GHz b/g/n, Bluetooth 4.2 + BLE |
| Crystal | 40 MHz |
| USB | CP2102 or CH340 **UART bridge** — a driver may be needed, see §5.0 |

### Living without PSRAM

This is the one thing that genuinely differs from an ESP32-S3 N16R8 build, and
it is all in `sg_yahoo.cpp`. A one-year Yahoo chart is 23–34 KB of JSON, and on
this board that buffer, the mbedTLS buffers and the parsed document all come out
of a single ~200 KB internal heap. Three things keep it inside:

- **The body buffer is reserved in one 48 KB block, before TLS connects.** Left
  to grow on its own it reallocs once per 1436-byte TCP read — about 25 times
  for a 34 KB body — and that churn against a heap already holding the TLS
  buffers is what fragments it into a failed allocation.
- **TLS memory is released before parsing.** mbedTLS takes two 16 KB content
  buffers (`MBEDTLS_SSL_MAX_CONTENT_LEN`) plus a session, ~40 KB, and it is
  always internal DRAM (`MBEDTLS_INTERNAL_MEM_ALLOC=y`) — PSRAM would not have
  helped here even if the board had it. The download and the parse therefore
  never peak at the same time.
- **A fetch that cannot fit is refused, not attempted.** `yahooFetch()` checks
  for 120 KB free and a 48 KB contiguous block first and returns
  `low memory (… KB free, largest block … KB)`. An out-of-memory abort partway
  through would reboot the board; this costs one symbol a `no data` line.

The same code still runs unmodified on an S3: `JsonAllocator` puts the document
in SPIRAM when `psramFound()`, and the IDF routes the 48 KB body buffer there
too. Nothing below needs changing to move back.

### Flash layout

Partition scheme `Minimal SPIFFS (1.9MB APP with OTA/128KB SPIFFS)`:

| Partition | Offset | Size | Holds |
|---|---|---|---|
| `nvs` | `0x9000` | 20 KB | settings from the web portal, watchlist |
| `otadata` | `0xe000` | 8 KB | OTA slot selector |
| `app0` | `0x10000` | 1.9 MB | the sketch (currently ~58% used) |
| `app1` | `0x1F0000` | 1.9 MB | second OTA slot (unused) |
| `spiffs` | `0x3D0000` | 128 KB | SPIFFS (unused) |
| `coredump` | `0x3F0000` | 64 KB | crash dumps |

The 4 MB default scheme (1.2 MB app) also fits, at 88% — but with no room to
grow and an equally tight OTA slot. §3 uses `min_spiffs` for that reason.

---

## 2. Software prerequisites

| Component | Version | Notes |
|---|---|---|
| ESP32 Arduino core | **3.3.10** | IDF 5.x. Older 2.x cores use a different watchdog API and will not build. |
| ArduinoJson | **7.x** (7.4.3 tested) | **v6 will not compile.** The sketch uses the v7 `Allocator` interface and bare `JsonDocument`. |
| Arduino IDE *or* arduino-cli | any current | Both paths documented below. |

Installing ArduinoJson v7 replaces v6 globally. If other sketches on this
machine parse JSON with the v6 API, they will need updating.

---

## 3. Board settings

*Tools → Board → esp32 → **ESP32 Dev Module***, then:

| Setting | Value | Why |
|---|---|---|
| **Partition Scheme** | **Minimal SPIFFS (1.9MB APP with OTA/128KB SPIFFS)** | TLS + ArduinoJson makes a big binary. The 1.2 MB default leaves only 12% headroom. |
| **PSRAM** | **Disabled** | A WROOM-32 has none. Enabling it just adds a failed probe at boot. |
| Flash Size | 4MB (32Mb) | matches a stock DevKitC; check yours if unsure (§5.0) |
| Upload Speed | 921600 | drop to 115200 if uploads fail on a CH340 |
| CPU Frequency | 240MHz (WiFi/BT) | default |
| Erase All Flash | **Disabled** | Enabling it wipes NVS — see §7 |

There is **no "USB CDC On Boot"** setting on this board and none is needed:
`Serial` goes out UART0 through the USB bridge chip, which is where the Serial
Monitor is already looking.

The equivalent FQBN for arduino-cli:

```
esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs
```

---

## 4. Where to change settings

**Nothing user-specific is compiled in.** Wi-Fi, ntfy, passphrases, the
report/command options and the watchlist are all entered on a small web page
the node serves itself, and stored in NVS. A `.bin` built from this sketch
carries no credentials and can be shared or flashed onto any board.

### 4.1 First-boot setup

On a board with no stored settings, the node starts its own Wi-Fi network:

1. Join **`StockGraber-Setup-XXXX`** (the suffix is the last four of its MAC)
   from a phone or laptop. The setup page should pop up on its own; if not, open
   **http://192.168.4.1**.
2. Fill in the form and press **Save & restart**. The node restarts, joins your
   network and announces itself on your ntfy topic.

| Section | Fields |
|---|---|
| Wi-Fi | network (picked from a live scan — 2.4 GHz only) and password |
| ntfy | server URL (`https://ntfy.sh`, or `http://192.168.1.5:8080` for a self-hosted one) and topic |
| ntfy login | username + password, or a token — blank for an open server. A token wins over username/password. |
| Passphrases | command passphrase, and an admin password for this page |
| Reports & commands | daily reports on/off, commands on/off, report delay (15–30 min), 52-week proximity %, poll interval |
| Watchlist | Yahoo codes separated by commas or spaces, up to 50 |

- **Topic: must be unique per node.** Two nodes on one topic = duplicate
  reports and two bots answering every command. ntfy.sh topics are readable by
  anyone who knows the name, so make it long and unguessable. Letters, digits,
  `-` and `_` only.
- **Command passphrase** prefixes every command (`<passphrase> status`) and is
  required for `add`/`remove`/`factoryreset`. Without it, anyone who finds your
  topic could edit the watchlist.
- **Admin password** protects the settings page (username **`admin`**). It must
  differ from the command passphrase, because that one travels through ntfy.
- **Watchlist** codes typed here are not checked against Yahoo the way `add`
  is, so a typo shows up later as a `no data` line. Aliases work (`gold` →
  `GC=F`).

Password fields are never shown back. Leaving one blank keeps the stored value;
tick *clear stored ntfy login* to remove the ntfy username/password/token.

The setup screen **never times out** — with no credentials there is nothing
else useful the node could be doing.

### 4.2 Changing settings later

| Situation | What to do |
|---|---|
| Node is online | Browse to its IP (it's in the startup notification) and log in as `admin` |
| Wi-Fi is down | It falls back to the `StockGraber-Setup-XXXX` network (portal at `192.168.4.1`, admin login required) while retrying your router every 10 s |
| Moved house, or forgot the admin password | Press **EN**, release it, then **immediately press and hold BOOT** for ~2 s — setup portal, no login required |
| Remotely | `<passphrase> factoryreset` over ntfy |

Every save restarts the node; there is no partial "apply live" state.

**Timing the BOOT button matters.** The ESP32 samples BOOT (GPIO0) *during*
reset — holding it through the EN press puts the chip into its serial
bootloader instead, and it just sits there. Press BOOT only after letting go of
EN; the node watches for it for 2 s (`BOOT_WINDOW_MS`) after `press BOOT now`
appears on serial. That window only runs on a provisioned board, so a normal
boot is ~2 s slower than it used to be.

`factoryreset` wipes the settings and reboots into setup. The watchlist is kept.

Physical access bypasses the admin password by design: anyone holding the board
can reflash it anyway, so a BOOT-button escape hatch costs no real security and
saves you from a lockout.

### 4.3 Boards set up with the old `sg_secrets.h` / `FORCE_PROVISION` flow

They keep working after this firmware is flashed: the Wi-Fi, topic, passphrase
and watchlist already in NVS are read as before. They have **no admin
password**, though, so the LAN settings page answers *Settings locked* until
one is set. Use the BOOT-button step above once, fill in an admin password
(other blank password fields keep their stored values), and save.

### Compile-time constants

Not stored in NVS — a normal reflash is enough.

| Setting | File | Line | Default |
|---|---|---|---|
| `WL_MAX` | `sg_config.h` | 68 | `50` — max watchlist size (see §8) |
| `SG_VERSION` | `StockGraberAlerts.ino` | 36 | `"1.0.0"` — shown at boot and in the startup notice; bump per release |
| `WDT_TIMEOUT_S` | `StockGraberAlerts.ino` | 38 | `120` — watchdog reboot timeout |
| `NTP_RESYNC_MS` | `StockGraberAlerts.ino` | 39 | 24 h between clock resyncs |
| `DUE_CHECK_MS` | `StockGraberAlerts.ino` | 40 | 60 s between "is a market due?" checks |
| `WIFI_TIMEOUT_MS` | `StockGraberAlerts.ino` | 41 | 30 s for the first join before falling back to the AP |
| `WIFI_RETRY_MS` | `StockGraberAlerts.ino` | 42 | 10 s between re-joins while in AP fallback |
| `BOOT_WINDOW_MS` | `StockGraberAlerts.ino` | 43 | 2 s after boot to press BOOT for setup |
| `BODY_RESERVE` | `sg_yahoo.cpp` | 17 | 48 KB — Yahoo body buffer (§1) |
| `FETCH_MIN_FREE` | `sg_yahoo.cpp` | 29 | 120 KB — heap a fetch refuses to start below (§1) |

---

## 5. Compile and flash

### 5.0 Identify the board

Port names follow the USB socket, not the board, so **always check before
uploading** — especially with more than one node on the bench.

```
arduino-cli board list
```

Unlike an S3 (native USB, `/dev/cu.usbmodem*`), this board talks through a
**UART bridge chip**, so the port name depends on which chip your board carries:

| Bridge | macOS | Linux | Windows |
|---|---|---|---|
| CP2102 | `/dev/cu.usbserial-*` or `/dev/cu.SLAB_USBtoUART` | `/dev/ttyUSB*` | `COM*` |
| CH340/CH9102 | `/dev/cu.wchusbserial*` | `/dev/ttyUSB*` | `COM*` |

If no port appears at all, the driver is missing — install Silicon Labs'
CP210x VCP driver or WCH's CH34x driver and replug. (Recent macOS ships a CP210x
driver; CH340 usually still needs WCH's.)

To confirm *which* physical board a port is, read its chip ID and MAC:

```
esptool --port /dev/cu.usbserial-0001 --no-stub flash-id
```

Expect something like:

```
Chip type:          ESP32-D0WD-V3 (revision v3.1)
Features:           Wi-Fi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
Crystal is 40MHz
Detected flash size: 4MB
```

Note the absence of any `Embedded PSRAM` line — that is correct for this board.
Keep a note of each node's MAC — the last four hex digits are also the suffix
of its `StockGraber-Setup-XXXX` network, which tells nodes apart during setup.

### 5.1 Compile

```
arduino-cli compile --fqbn "esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs" StockGraberAlerts
```

A good build ends with roughly:

```
Sketch uses 1158116 bytes (58%) of program storage space. Maximum is 1966080 bytes.
Global variables use 49336 bytes (15%) of dynamic memory, leaving 278344 bytes for local variables.
```

That 278 KB is the number the §1 memory budget is drawn against — Wi-Fi and
lwIP take their share of it at runtime, leaving roughly 200 KB of heap.

If it fails, check §2 — the usual causes are ArduinoJson v6 and ESP32 core 2.x.

### 5.2 Upload

```
arduino-cli upload -p /dev/cu.usbserial-0001 --fqbn "esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs" StockGraberAlerts
```

Substitute your own port. Some DevKitC clones do not auto-reset into the
bootloader: if the upload stalls at `Connecting........_____`, hold **BOOT**,
tap **EN**, release **BOOT**, and retry. A successful upload ends with:

```
Hash of data verified.
Hard resetting via RTS pin...
```

`No changed sectors found` instead means the board already had that exact
binary — not an error.

### 5.3 First flash of a new board

Compile and upload as above — there is nothing to edit first. On boot the
serial log shows:

```
[sg] web: AP "StockGraber-Setup-A1B2" up, portal at http://192.168.4.1 (auth off)
[sg] === SETUP MODE ===
[sg] setup: join wifi "StockGraber-Setup-A1B2" then open http://192.168.4.1
```

Then follow §4.1. After saving, the node restarts and should reach the healthy
boot in §6.

### 5.4 Updating firmware on a running node

Compile, upload. One pass. Settings and watchlist are preserved — ordinary
reflashing does not touch NVS.

### Arduino IDE equivalent

1. Apply the board settings in §3.
2. *Tools → Port* → select the board.
3. *Sketch → Upload*.
4. For a new board, continue with §4.1.

---

## 6. Verify

Serial monitor at **115200**:

```
arduino-cli monitor -p /dev/cu.usbserial-0001 --config baudrate=115200
```

A healthy boot:

```
[sg] StockGraber Alerts v1.0.0 (ESP32-D0WD-V3) starting
[sg] heap: 216 KB free, largest block 110 KB
[sg] press BOOT now to open the setup portal
[sg] wifi: connecting to your-wifi
[sg] wifi: ok 192.168.1.42
[sg] ntp: clock synced
[sg] web: settings page at http://192.168.1.42
[sg] startup notice sent
[sg] running; topic: your-topic-esp; settings: http://192.168.1.42
```

There is deliberately **no PSRAM line** — the sketch only prints one if PSRAM is
actually present, so its absence here is correct, not a misconfiguration.

**The `heap:` line is the one to read on this board.** It is printed before
Wi-Fi comes up, so it is the ceiling rather than the working figure; expect the
free number to drop by roughly 50–60 KB once associated. A fetch needs 120 KB
free and a 48 KB contiguous block (§1), so if that line reports much under
~180 KB free, watch for `low memory` in §Troubleshooting.

The node also **pushes a startup notification to ntfy on every boot**:

```
StockGraber - node started
Reset: power-on
IP: 192.168.1.42
Settings: http://192.168.1.42
Firmware: v1.0.0
Watchlist: 5 of 50 symbols
Heap: 158 KB free
```

The `Reset:` line distinguishes a power cycle from a watchdog trip, a crash, or
a brownout — so an unexplained reboot is diagnosable without a serial cable.
After a Wi-Fi outage the node sends a similar *back online* notice, since its
IP (and so the settings URL) may have changed.

After startup the log is **silent by design**. `loop()` only prints on an event
(report sent, command answered, Wi-Fi lost). Silence means healthy; a repeating
boot banner means it's stuck in a restart loop.

---

## 7. Storage and persistence

Settings and the watchlist live in NVS (flash) and are committed on every
change, so they survive power loss and ordinary reflashing.

To inspect what's actually stored:

```
esptool --port /dev/cu.usbserial-0001 read-flash 0x9000 0x5000 nvs.bin
strings nvs.bin | grep -A1 watch
```

**What wipes NVS:**

- *Tools → Erase All Flash Before Sketch Upload* → **Enabled**. This is a sticky
  IDE setting — turning it on for another project silently arms it for this one.
- `esptool erase-flash`.
- `<passphrase> factoryreset` over ntfy — settings only; the watchlist is kept.

---

## 8. Limits

| Limit | Value | What happens at the edge |
|---|---|---|
| **Watchlist** | **50 symbols** (`WL_MAX` in `sg_config.h`) | `add` replies *"Already holding the maximum of 50 symbols"* and refuses. |
| NVS string | ~700 symbols | Never reached; the 50-symbol cap sits well below it. |
| ntfy message body | 4 KB (ntfy.sh) | **51–136 lines depending on content** — see below. A 50-symbol `status` is 1.5–4.0 KB. |
| `status` runtime | ~3 s per symbol | 50 symbols ≈ 2–3 minutes. The watchdog is fed per symbol, so it won't reboot. |
| Heap during `status` | ~4 KB for the reply | Negligible against the fetch budget (§1); symbol count is essentially free on memory. |

### How close 50 symbols really is to the 4 KB reply limit

Closer than a per-line average suggests, because two things inflate the line:
display names are expanded (`BTC-USD` → `Bitcoin (BTC-USD)`), and `▲ ▼ ⚑ —` are
three bytes each in UTF-8, not one.

| Line | Example | Bytes | 4 KB holds |
|---|---|---|---|
| bare ticker, no flags | `AAPL       303.42  ▲ +1.23%` | 29 | 136 |
| labelled, no flags | `Bitcoin (BTC-USD) 63,472.43  ▼ -2.05%` | 39 | 104 |
| labelled + one flag | …`   ⚑ near 52w high` | 59 | 68 |
| labelled + cross *and* 52w flag | …`   ⚑ golden cross  ⚑ near 52w high` | 76 | **51** |

So a 50-symbol watchlist is fine in normal conditions (~2.3 KB) but reaches
**3,999 bytes — 97 bytes of margin** in the pathological case where nearly every
symbol shows both a cross and a proximity flag at once. ntfy answers 413 rather
than truncating, so the whole `status` reply would fail, not just its tail.

That case needs almost the entire list flagged simultaneously, which is why 50
has held. If you intend to actually run a full list and want real margin, **40**
puts the worst case at 3,199 bytes (78% of the limit) and shortens `status` to
about two minutes.

The cap is a *runtime* limit, not a storage one — NVS could hold far more, and
memory is not what constrains it. `status` fetches every symbol over HTTPS in a
single command, so list length drives response time and reply size. Raise
`WL_MAX` only alongside the reply-size arithmetic above.

---

## 9. Use it

Subscribe to your topic in the ntfy app. Reports arrive ~20 min after each
market's close. From the app you can send:

| Send | Get |
|---|---|
| `<pass> status` | the whole watchlist |
| `<pass> list` | just the symbols |
| `<pass> AAPL` | one symbol |
| `<pass> add TSLA` | add it (verified against Yahoo first) |
| `<pass> remove IBM` | drop it |
| `<pass> factoryreset` | wipe settings, reboot into setup (watchlist kept) |
| `<pass> help` | the command list |

`add`/`remove` **require** the passphrase even though queries don't — ntfy.sh
topics are public, so an ungated topic stays read-only. Aliases work too
(`add gold` → `GC=F`, `add btc` → `BTC-USD`).

**Symbol format is Yahoo's**, which isn't always the obvious one:

| Instrument | Code |
|---|---|
| Gold futures | `GC=F` |
| Bitcoin | `BTC-USD` |
| Berkshire class B | `BRK-B` (hyphen, **not** `BRK.B`) |
| Hang Seng index | `^HSI` |

`add` fetches the symbol before storing it, so a bad code is rejected with
*"No data for X — not added"* rather than silently poisoning the watchlist.

---

## Files

| File | Role |
|---|---|
| `StockGraberAlerts.ino` | boot flow (setup / AP fallback / online), Wi-Fi, NTP, scheduler, dispatch, watchdog, startup notice |
| `sg_market.*` | market classification, close times, DST via POSIX TZ |
| `sg_calc.*` | SMA 50/200 + cross, 52-week, change %, line formatting |
| `sg_yahoo.*` | Yahoo chart API → OHLC |
| `sg_ntfy.*` | publish + poll, with optional ntfy login |
| `sg_commands.*` | command parsing |
| `sg_config.*` | settings + watchlist in NVS |
| `sg_webconfig.*` | setup portal and settings page |

## Design notes

- **Timezones use the C library.** Unlike the MicroPython node (which hand-rolls
  DST), the ESP32 Arduino core has real POSIX TZ support, so New York is just
  `EST5EDT,M3.2.0,M11.1.0` and DST is handled for you.
- **Yahoo is fetched over HTTP/1.0, then buffered.** Two transport quirks forced
  this. Yahoo answers HTTP/1.1 with `Transfer-Encoding: chunked`, and ESP32's
  `HTTPClient::getStream()` hands back the raw socket with the chunk framing
  still in it, which the JSON parser chokes on — hence `useHTTP10(true)`. Yahoo
  also sends no `Content-Length`, so the body ends at connection close; parsing
  straight off the socket mistakes a gap in the TCP stream for EOF and truncates
  the document. Reading the body to completion first avoids both.
- **The body buffer is reserved up front, not grown.** `writeToStream()` into a
  pre-reserved `StreamString` rather than `getString()`: same core code path, but
  one 48 KB allocation instead of ~25 reallocs. On PSRAM this was invisible;
  without it, that churn is the difference between working and fragmenting. See
  §1 for the full budget.
- **An ArduinoJson filter still applies.** Only the four arrays actually needed
  are kept: a 34 KB response parses down to a 20 KB document, and `meta`,
  `events` and the `adjclose` series are dropped as they stream past.
- **Polling, not streaming.** The Pi holds a long-lived ntfy connection; a
  multi-hour TLS session on a microcontroller is the least reliable thing you can
  build, so this polls every ~12 s. `since=<id>` gives dedup for free.
- **No price cache.** A handful of symbols fetched once per close is cheaper than
  managing flash writes.
- **Settings live in NVS, not the binary.** The portal is the WOL-NTFY
  project's design: SoftAP + captive portal before provisioning, SoftAP + STA as
  a fallback while Wi-Fi is down, and a login-protected page on the LAN. DNS is
  only hijacked in AP mode; doing it on the LAN would break name resolution.
- **The `ok` sentinel is written last** when saving, so a power cut mid-save
  reads as unprovisioned rather than half-valid.
- **The watchdog is fed per symbol.** Both the report path and the `status`
  command reset the task WDT inside their fetch loops; without that, a long
  watchlist reboots the board mid-reply.

## Troubleshooting

| Symptom | Cause |
|---|---|
| No serial port appears at all | Missing USB-bridge driver (CP210x or CH34x) — see §5.0. |
| Serial Monitor blank / garbage | Wrong baud (must be 115200), or the wrong `/dev/cu.*` of a pair. |
| Upload stalls at `Connecting......____` | Clone without auto-reset: hold **BOOT**, tap **EN**, release **BOOT**. Or drop Upload Speed to 115200. |
| `Sketch too big` | Partition Scheme left at the 1.2 MB default and something grew — switch to **Minimal SPIFFS** (§3). |
| Boots into `SETUP MODE` every time | No complete settings in NVS (or a save was cut off). Finish the form in §4.1. |
| `wifi unreachable; portal at …` | Router down, wrong password, or a 5 GHz-only network. Fix it from the `StockGraber-Setup-XXXX` network (§4.2); the node also keeps retrying on its own. |
| Settings page says *Settings locked* | Board set up by the old `sg_secrets.h` flow — no admin password yet. See §4.3. |
| BOOT press does nothing / board seems dead after it | BOOT was held through the reset, which starts the serial bootloader. Press EN, release, *then* press BOOT (§4.2). |
| `HTTP 401` / `HTTP 403` from ntfy | The server wants a login, or rejected it. Check username/password/token on the settings page. |
| `fetch failed: … low memory (… KB free …)` | The heap could not cover a fetch (§1). Something else is holding memory, or `FETCH_MIN_FREE` is set above what this board actually has — compare against the boot `heap:` line before lowering it. |
| Sporadic reboots during `status` only | Memory, almost certainly. Check the boot `heap:` line and the reset reason in the ntfy startup notice. |
| No PSRAM line in the boot log | Expected on this board — see §6. |
| Every symbol reports `no data` | Check serial for the `[sg] fetch failed:` line — it names the real error. |
| `HTTP 404` on a symbol | Wrong Yahoo code (e.g. `BRK.B` instead of `BRK-B`). |
| `HTTP 429` | Yahoo rate-limiting. Shorten the watchlist or slow the poll. |
| Compile errors mentioning `JsonDocument` or `Allocator` | ArduinoJson v6 installed; upgrade to v7. |
| Compile error on `esp_task_wdt_init` | ESP32 core 2.x installed; needs 3.x. |

## Known limitations

- **`status` is slow** — every symbol is fetched fresh, ~3 s each.
- **TLS certificates are not verified** (`setInsecure()`); bundle a CA cert if
  that matters to you.
- **The settings page is plain HTTP.** The admin login is HTTP Basic auth, so
  it crosses your LAN (or the setup AP) unencrypted. Fine on a home network;
  don't expose port 80 to the internet.
- **Yahoo access is the fragile part.** `yfinance` quietly handles cookies,
  crumbs and rate limits; here it's a browser-ish User-Agent. If Yahoo starts
  demanding a crumb, `yahooFetch()` is the thing to fix.
- **The daily report path has not been observed running.** It shares the fetch
  code that the `status` command exercises, but it only fires after a market
  close and hasn't yet been seen end-to-end.
- **Memory is the tight resource now, not flash.** With no PSRAM a fetch peaks at
  roughly 90 KB against a ~200 KB heap. It fits with margin, but a future feature
  that holds tens of KB across a fetch would not.

## Verified status

> **This is the ESP32 Dev Module port. It has been verified by build and by
> host-side testing, not yet on this board.** The ESP32-S3 notes at the end
> record what was confirmed on the previous hardware.

Verified for the ESP32 Dev Module:

- Compiles clean for `esp32:esp32:esp32:PSRAM=disabled,FlashSize=4M,PartitionScheme=min_spiffs`
  on ESP32 core 3.3.10 with ArduinoJson 7.4.3 — **1,158,116 bytes, 58% of the
  1.9 MB app partition; 49,336 bytes static RAM** (with the web portal).
- Still compiles unchanged for the S3 FQBN, so the port is not a fork.
- The filter and extraction path in `sg_yahoo.cpp` was run on the host against
  live Yahoo payloads for `AAPL`, `GC=F`, `BTC-USD` and `^HSI` (equity, futures,
  crypto, index): all four parse, bar counts and 52-week ranges are correct, null
  bars are skipped, and the parsed document peaks at 20 KB — the number the
  `FETCH_MIN_FREE` budget is built on.
- The memory budget itself is derived from the core's own
  `MBEDTLS_SSL_MAX_CONTENT_LEN=16384` / `MBEDTLS_INTERNAL_MEM_ALLOC=y`, not
  estimated.

**The web portal is verified by build only.** Setup mode, the captive-portal
pop-up, AP fallback, the BOOT-button window and ntfy login have not yet been run
on hardware.

**Not yet verified on this board:** upload, Wi-Fi association, NTP, NVS
persistence, ntfy polling and replies, and — the one that matters most here —
the real free-heap figure at fetch time. Read the boot `heap:` line (§6) on
first flash and check it against §1.

Previously confirmed on ESP32-S3 N16R8 hardware: build, upload, Wi-Fi, NTP,
8 MB PSRAM, NVS persistence across reflashing, ntfy command polling and replies,
Yahoo fetches for all watchlist symbols, passphrase-gated `add`/`remove`, and the
startup notification.
