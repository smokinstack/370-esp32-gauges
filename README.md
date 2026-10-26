# 370Z Telemetry — Current Build

What the system is made of today: hardware, firmware, tools and servers.

---

## 1. Overview

```
 OBD-II port ──(passive Y-splitter)──┬── ECUtek Bluetooth dongle (ECU Connect app on the phone)
   pin 6 CAN-H / pin 14 CAN-L        │
                                     └── BRAIN  Waveshare ESP32-S3-RS485-CAN
                                           │  CAN listen-only, 500 kbit/s
                                           │  sensor I²C bus (Qwiic hub):
                                           │    M5Stack Unit VMeter ── oil pressure sender
                                           │    Adafruit INA3221 ───── battery volts / AFR / MAP inputs
                                           │    DS3231 RTC
                                           │  own Wi-Fi hotspot 
                       ┌───────────────────┼───────────────────┐
                   SCREEN 1             SCREEN 2            SCREEN 3      Waveshare ESP32-S3-Touch-LCD-2.1 ×3
                       └── poll http://ip:8080/ (JSON) ──┘
                                           │
                                         Phone: settings app :8080/app, recording :8081/rec

 Standalone: CAN → SD logger on a Waveshare ESP32-S3-Touch-LCD-4.3B
 Home network: code / build / editor server, GitLab runner + firmware publisher
```

**Ground rules built into everything:**
- **CAN is read-only.** Every device that touches the bus is listen-only and never transmits.
- **No fake readings.** Missing, stale or out-of-range values are `null` and show as `--`.
- **One owner per chip.** Each I²C device is read by exactly one task.

---

## 2. Hardware

### 2.1 Bill of materials

| Part                                                                                                                                                                   | Role                                                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Waveshare **ESP32-S3-RS485-CAN** (ESP32-S3R8, 16 MB flash, 8 MB OPI PSRAM, isolated CAN transceiver, onboard PCF85063 RTC)                                             | The brain                                                                                            |
| Passive OBD-II Y-splitter                                                                                                                                              | Connects the OBD port to the brain                                                                   |
| M5Stack **Unit VMeter** (isolated ADS1115, I²C 0x49, EEPROM 0x53)                                                                                                      | Oil pressure sender input                                                                            |
| Oil pressure sender, 0.5–4.5 V = 0–10 bar                                                                                                                              | Shared with the ECU                                                                                  |
| Adafruit **INA3221** (3 × 0–26 V, I²C 0x40)<br>Still not sure if this is best to wideband and MAP, maybe change to addition M5Stack units                              | Battery volts / wideband AFR / MAP inputs                                                            |
| DS3231 RTC (I²C 0x68)                                                                                                                                                  | Extra clock on the sensor bus                                                                        |
| Qwiic hub board                                                                                                                                                        | Joins the sensor devices to the brain's SH1.0 socket                                                 |
| 12 V socket USB adapter + USB-C lead                                                                                                                                   | Powers the brain                                                                                     |
| 3 × Waveshare **ESP32-S3-Touch-LCD-2.1** (480×480 round IPS, ST7701 RGB panel, CST820 touch, QMI8658 IMU, PCF85063 RTC, TCA9554 I/O expander, 16 MB flash, 8 MB PSRAM) | The gauge screens                                                                                    |
| Waveshare **ESP32-S3-Touch-LCD-4.3B** + microSD card (+ optional MAX17048 fuel gauge at 0x36)                                                                          | Standalone CAN logger                                                                                |
| **12V→5V buck converter, 5A connected using a fuse tap**                                                                                                               | Powers all three screen using the 112PIN Multi-function Interface on the rear of each of the screens |

### 2.2 Brain wiring

| Function               | Connection                                                                                                                                                    |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CAN                    | OBD pin 6 (CAN-H), pin 14 (CAN-L), pins 4/5 ground, via the Y-splitter → brain CAN H/L. Board's 120 Ω terminator **off**: the car's bus is already terminated |
| Power                  | USB-C from the 12 V socket adapter. The 12 V screw terminal is not used while on USB                                                                          |
| Onboard RTC (PCF85063) | I²C GPIO39 SDA / GPIO38 SCL (`Wire1`)                                                                                                                         |
| Sensor bus             | SH1.0 4-pin socket (Qwiic order: GND, 3V3, SDA, SCL) → Qwiic hub. `PIN_SENS_SDA 2`, `PIN_SENS_SCL 1`, 100 kHz                                                 |

**Sensor bus addresses**

| Address | Device |
|---|---|
| `0x40` | INA3221 |
| `0x49` | M5 VMeter ADS1115 |
| `0x53` | M5 VMeter EEPROM (factory calibration, never written) |
| `0x68` | DS3231 |

### 2.3 Sensors

- **Oil pressure:** sender signal → VMeter `+`, sender ground → VMeter `−`.
  - The VMeter's input is isolated, so the difference between engine ground and dash ground doesn't affect the reading.
- **INA3221 channels:** what each channel measures is chosen in the brain app (Inputs tab): not used, battery volts, AFR, or boost (MAP).
  - Battery: a fused 12 V feed from the dash side.
  - AFR: linear 0–5 V = 10–20 AFR (AEM 30-0300 style).
  - MAP: Bosch 3-bar, 0.25 V = 11 kPa, 4.75 V = 307 kPa.
  - Unwired channels are set to "Not used".


---

## 3. Brain firmware (v1.1.0)

Source: PlatformIO tree `brain370z/` and the matching Arduino sketch
`~/waveshare2.1/Arduino/Brain370Z/` (`Brain370Z.ino` = `main.cpp`).

**Build settings**
- Arduino core esp32 3.x.
- FQBN `esp32:esp32:esp32s3:FlashSize=16M,PSRAM=opi,CDCOnBoot=cdc,USBMode=hwcdc`.
- `partitions.csv`: two 4 MB app slots (over-the-air updates) plus storage for the screens' firmware files.
- User settings live in **`config_local.h`** (included first, never replaced by updates). It holds the  the oil calibration, the sensor pins and `CAN_BENCH_ACK 0`.
- `config.h` holds the defaults.

### 3.1 Tasks

| Task | Core / priority | Job |
|---|---|---|
| `can` | 1 / 6 | TWAI receive (listen-only), decoding, feeds the recording buffer |
| `sensors` | 1 / 4 | Sole owner of the sensor I²C bus (M5, INA3221, DS3231) |
| `rec` | 0 / 3 | Recording stream server on :8081 |
| Arduino loop | 1 / 1 | Web server on :8080, Wi-Fi config |

### 3.2 Source files

| File | Does |
|---|---|
| `can_task.cpp` / `can_decode.cpp` | TWAI listen-only receive, CAN decoding, ECUtek oil eavesdrop |
| `sensors_task.cpp` / `inputs.cpp` | Oil (M5), INA3221 channels, DS3231; input assignment |
| `oil_status.cpp` | Oil verdict: pressure judged against rpm and oil temperature |
| `json_build.cpp` | The screens' JSON |
| `web.cpp` / `webapp_gz.h` | HTTP server, settings app, screen firmware files, brain OTA |
| `netcfg.cpp` | Hotspot settings, handing Wi-Fi changes to the screens, bench profile |
| `clock.cpp` | Onboard RTC, DS3231, phone/NTP time sync |
| `rec_stream.cpp` | Recording stream |

### 3.3 CAN decodes (500 kbit/s, 11-bit IDs)

| Signal                     | ID  | Bytes      | Formula                                                               |
| -------------------------- | --- | ---------- | --------------------------------------------------------------------- |
| RPM                        |     | 0:1 BE     | `raw / 8`                                                             |
| Injection pulse width (ms) |     | 2          | `raw × 0.20211 − 7.76`                                                |
| Throttle (%)               |     | 4          | `raw × 0.29195 + 1.97`                                                |
| Road speed (mph)           |     | 0:1 BE     | `raw × 0.0061917 − 0.49`                                              |
| Coolant (°C)               |     | 0          | `raw − 40`                                                            |
| Oil temp (°C)              |     | 4          | `raw − 50`                                                            |
| Injector duty (%)          |     | 0:1 BE     | `raw × 0.0082625 + 0.90`                                              |
| Gear                       |     | 0          | `0x80…0xA8` = 1–6 (`(raw−128)/8+1`), `0x18` = N, `0x10` = R           |
| Tyre pressures (psi)       |     | 2–5        | `raw / 4` (b2 RR, b3 FR, b4 RL, b5 FL); `0x00` = sensor asleep → null |
| Exterior lights            |     | 1 & `0x60` | off / side / head                                                     |
| Parking brake              |     | 0          | 64 off / 68 on                                                        |
| Brake pedal                |     | 6          | 4 off / 20 on                                                         |
| VDC off                    |     | 4 bit 6    |                                                                       |

### 3.4 Oil

- **Direct sensor (M5):**
  - Sampling: every 250 ms, 9 samples, differential, auto-ranging; median then EMA 0.4.
  - Fault: below 0.30 V = sender fault → null.
  - Calibration in `config_local.h`: `OIL_BOARD_CAL_SCALE 1.0186`, `OIL_BOARD_CAL_OFFSET 0.0075`.
- **`oil_psi`:** the direct sensor 
- **`oil_status`:** `ok` / `warn` / `crit` / null.
  - Threshold = an rpm curve (Nissan minimum: 14 psi at ~650 rpm, 43 psi at 2000 rpm) × an oil temperature multiplier.
  - Only judged at ≥ 400 rpm.

### 3.5 Network and endpoints

- **Hotspot:** IP address, channel 6, up to 8 clients. Name and password can be changed from the app's Wi-Fi tab, which passes them to the screens first.

| Address | Purpose |
|---|---|
| `GET :8080/` | The screens' JSON |
| `:8080/health`, `/healthz` | `ok` |
| `:8080/app` | Phone settings app (Preact + htm, built from `webapp/index.html` by `build_webapp.py`) |
| `:8080/firmware/…` | Screen OTA files (`manifest.json`, `gauge.bin`) |
| `:8080/api/…` | `state`, `wifi`, `inputs`, `time`, `files`, `update` |
| `:8081/rec` | Recording stream (separate port so recording never slows the screens) |

**App tabs:**
- **Live:** all readings plus diagnostics.
- **Record:** CAN log plus decoded CSV to the phone.
- **Inputs:** INA3221 channel assignment.
- **Wi-Fi:** hotspot and bench profile.
- **Clock:** set the time from the phone.
- **Firmware:** brain OTA update, and upload the screens' OTA files.

**Recording:**
- Every CAN frame (candump format) plus the JSON 10×/s, buffered in a 2 MB PSRAM ring.
- MARK frames `1FFFFFF0#4D41524B…`.
- Saved on the phone as `370z_YYYYMMDD_HHMMSS.log` and `…_decoded.csv`.

**USB serial (115200):**
- `?` → `BRAIN370Z`.
- candump lines are replayed into the decoder.
- `i2c` → sensor bus report.
- 10 s CAN heartbeat.

### 3.6 JSON contract

```json
{"ts":1792691017.56,"link":true,"bus_link":true,
 "wheels":{"rr":31.0,"fr":33.0,"rl":28.2,"fl":32.2},
 "rpm":1719,"gear":3,"speed_mph":31.2,"coolant_c":86,"oil_temp_c":84,
 "oil_psi":61.4,"oil_status":"ok","oil_psi_direct":61.4,"oil_psi_ecutek":62.0,
 "throttle_pct":12.5,"volts":14.12,"afr":null,"boost_psi":null,"lights":"off",
 "g_lat":null,"g_lon":null,
 "inj_ms":3.12,"inj_duty_pct":6.4,"handbrake":0,"brake":0,"vdc_off":0,"map_kpa":null,
 "clock":{"y":2026,"mo":10,"d":22,"h":18,"mi":43,"s":37,"dow":4,"ok":true}}
```

- **Freshness:** any value may be `null`.
  - CAN values: null after 5 s.
  - Direct oil: null after 3 s.
  - ECUtek oil: null after 5 s.
  - Volts, AFR and MAP: null after 5 s.
- **`link` / `bus_link`:** a TPMS frame / any CAN frame within the last 5 s.
- **`gear`:** 1–6, neutral 0, reverse −1.
- **`clock`:** local time; `ok:false` = not confirmed.
- **G-force:** computed on each screen from its own IMU, so `g_lat` / `g_lon` are always null from the brain.

---

## 4. Screen firmware (3 × ESP32-S3-Touch-LCD-2.1)

Arduino sketch `examples/LVGL_Arduino/` in the screen repo.

**Build settings**
- Arduino core **esp32 3.3.11** (ESP-IDF 5.5.5), LVGL **8.3.10**, ArduinoJson.
- FQBN `esp32:esp32:waveshare_esp32_s3_touch_lcd_21`; 3 MB app slots.
- Per-screen settings are in **`Gauge_Config.h`**: `GA_SCREEN_ID`, `GA_VERSION`, Wi-Fi profiles, limits.

### 4.1 Display stack

| Item | Setting |
|---|---|
| Panel | ST7701 480×480 RGB, 12 MHz pixel clock (`Display_ST7701.cpp`) |
| Frame buffer | One, in PSRAM (`num_fbs 1`, `double_fb false`) |
| Bounce buffer | 10 lines (`DISP_BOUNCE_LINES`, falls back to 4 then 1 if there's no internal RAM); restart-in-vsync on |
| LVGL rendering | Partial refresh into two 16-line draw buffers (15 KB each) in internal RAM (`LVGL_Driver.cpp`) |
| LVGL memory | 512 KB pool in **PSRAM** (`libraries/lvgl/src/lv_conf.h`: `LV_MEM_POOL_ALLOC` → `heap_caps_malloc(…, MALLOC_CAP_SPIRAM)`) |
| Boot log | Prints the bounce buffer size, core/IDF version, restart-in-vsync, internal RAM left, LVGL pool use |

### 4.2 Source files

| File | Does |
|---|---|
| `LVGL_Arduino.ino` | Setup / loop |
| `Gauge.cpp`, `Gauge.h` | Built-in views, settings pages, alerts, swipe order, alert routing |
| `Gauge_Telemetry.*` | JSON → telemetry; built-in view list (`GA_VIEW_LIST`) |
| `Gauge_Net.*` | Wi-Fi profiles, polling, link handling |
| `Gauge_Ota.*` | Firmware update from the manifest |
| `Gauge_Layout.*` | Layout engine for custom views (one LVGL object per view, drawn in a callback) |
| `Gauge_Inputs.*` | Input table shared by the engine and the editor |
| `Gauge_Layouts.h` | Custom views and uploaded images, generated by the layout editor |
| `Gauge_Config.h` | Screen settings (yours, never shipped) |
| `Display_ST7701.*`, `LVGL_Driver.*`, `Touch_CST820.*`, `Gyro_QMI8658.*`, `RTC_PCF85063.*`, `TCA9554PWR.*`, `I2C_Driver.*`, `BAT_Driver.*`, `SD_Card.*`, `Wireless.*` | Board drivers |
| `car_370z.c`, `smokinstack_logo.c` | Built-in images (car 116×168 RGB565 + alpha; splash logo) |

### 4.3 Behaviour

**Networking**
- Polls `GET http://<host>/` every 1 s.
- Two network profiles: **CAR** (brain ip:8080) and **BENCH** (ip:8137, the publisher). Switched on the NETWORK page.
- Picks up the brain's Wi-Fi details every 15 s.

**Views**
- Built-in views: oil pressure, oil temp, oil split, battery, tyres, speed, G-force, clock, AFR, boost, coolant.
- Custom views come from `Gauge_Layouts.h`.
- The layout file sets the swipe order, which screens show which views, and switches views off.

**Settings pages:** DISPLAY, THEME, UNITS, ALERTS, SENSORS, LEVEL, NETWORK, ABOUT, REBOOT.

**Alerts**
- Oil (the brain's `oil_status`) takes priority over tyres. Oil jumps to the oil view, else the oil split view, else a custom view with an oil gauge.
- Tyres jump to the tyre view, else a custom view with tyre inputs.
- A screen never jumps to a view it doesn't show.
- The red face flash for oil is a separate switch on the ALERTS page.

**Other**
- G-force comes from each screen's own IMU.
- Day/night brightness follows the car's lights.
- Clock: RTC set from the brain; analogue/digital switch.
- OTA: `manifest.json` + `gauge.bin` into the spare app slot, MD5 checked.

### 4.4 Layout engine

- **Gauge types:** arc, needle, bar, number, warning light, text, line, clock, image.
- **Per gauge:**
  - Look: input, scale, units, colours (fill, track, reading, label, unit, scale numbers, ticks, no-data), segmented / notched / rounded bands, unlit shade, zone fill, second ring, marks, ticks.
  - Alerts: levels, only-when conditions, the brain's `oil_status`.
  - Texts: status notes, peak text, per-unit scales.
  - Animation: smoothing, opening sweep, peak hold.
- **Per view:** name, screens, on/off, dial face, background colour.
- **Images:** the built-in `car_370z`, plus uploaded images compiled into `Gauge_Layouts.h` (RGB565 + alpha).
- **Clock gauge:** onboard RTC; analogue, digital, or "screen setting".
- **Limits:** 12 custom views, 16 gauges per view, 32 view entries.
- **Format:** JSON `370z-gauge-layout` v1, documented in `LAYOUT_FORMAT.md`.

---

## 5. Layout editor

Browser editor for the screens' custom views. It runs on the code server as a
service.

**Running it**
- Server: `editor.py` (Python 3 standard library only), folder `~/waveshare2.1/Arduino/370z-layout-editor/`.
- Service: installed with `sudo ./install-service.sh` (system service running as the user); `--lan` makes it reachable from other PCs.

**Preview**
- `preview.wasm` is the screens' own LVGL 8.3.10, `lv_conf.h`, layout engine and car image, compiled to WebAssembly (`core/build_wasm.sh`, Zig).
- So the preview is pixel for pixel what a screen draws.

**Features**
- Views list with built-in views (import, reorder, switch off per screen); convert built-in views into editable ones (all except the G-force ball).
- Drag-and-drop gauges with handles, alignment guides and snapping (Alt = free), arrow keys 5 px (Shift = 1 px).
- Image upload (sized on upload), view background colour, favourite looks (saved in `~/.config/370z-layout-editor/styles.json`).
- Preview data: demo motion, set values, live from the brain, or replay of a `_decoded.csv`.
- Validated by the screen's own loader before export.

**Export**
- Writes `Gauge_Layouts.h` (CRLF) into the screen sketch, locally or over SSH (`user@host:path`).
- Reads the sketch's limits from `Gauge_Config.h` (limit `#define`s only) for conversions.
- Backups go to `~/.local/share/370z-layout-editor/backups/`, outside git.

**Settings:** `~/.config/370z-layout-editor/settings.json` (layouts folder, sketch folder, brain address).

---

