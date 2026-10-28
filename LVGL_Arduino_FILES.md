# What's in `examples/LVGL_Arduino`

The screen sketch: the firmware for the three round gauge screens.

It started as Waveshare's LVGL demo for the ESP32-S3-Touch-LCD-2.1. Their board
drivers are still here, mostly untouched. Their demo content has been replaced
by the gauge code (`Gauge_*` files).

**Key to the "Who" column**
- **Gauge:** written for this project.
- **Waveshare:** came with their demo.
- **Waveshare, changed:** their file, edited for this project.
- **Generated:** made by a tool. Don't edit by hand.
- Mine: my settings.

---

## The gauge code

| File                         | Who                           | What it's for                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `LVGL_Arduino.ino`           | Waveshare, changed            | The sketch's entry point. `setup()` starts the board drivers, the display, LVGL, then the gauge (`Gauge_Init`) and the network (`GaugeNet_Begin`). A background task keeps the IMU, clock and battery readings fresh. Waveshare's demo calls are commented out.                                                                                                                                                                           |
| `Gauge.h`                    | Gauge                         | The few functions the `.ino` calls into the gauge (`Gauge_Init`, `Gauge_Update`).                                                                                                                                                                                                                                                                                                                                                         |
| `Gauge.cpp`                  | Gauge                         | The main gauge program (the biggest file). Covers: <br>• the built-in views (oil dials, tyres, speedo, G-force, clock, AFR, boost, coolant, battery);<br>• the settings pages (display, theme, units, alerts, sensors, level, network, about, reboot);<br>• theme colours and brightness;<br>• swiping and the swipe order;<br>• alerts (which view to jump to, the red oil flash);<br>• showing the custom views from the layout editor. |
| `Gauge_Config.h`             | **Mine**                      | All the screen's settings. Covers: <br>• `GA_SCREEN_ID` (which of the 3 screens this build is for);<br>• `GA_VERSION`;<br>• Wi-Fi profiles (CAR = the brain, BENCH = the test server), with real passwords;<br>• the dial ranges and alert limits (oil, temperatures, volts, tyres…);<br>• defaults (theme, units, clock style). <br>Never overwritten by updates.                                                                        |
| `Gauge_Telemetry.h` / `.cpp` | Gauge                         | Turns the brain's JSON into a `Telemetry` struct (rpm, oil psi, tyres… with "no reading" kept as NaN). The `.h` also holds `GA_VIEW_LIST`, the list of built-in views. The editor reads that list to import them.                                                                                                                                                                                                                         |
| `Gauge_Net.h` / `.cpp`       | Gauge                         | Wi-Fi and data. Covers: <br>• joining the CAR or BENCH network;<br>• asking the brain for data every second;<br>• handling lost links;<br>• picking up a new Wi-Fi password from the brain;<br>• the "connected / not connected" status on the NETWORK page.                                                                                                                                                                              |
| `Gauge_Ota.h` / `.cpp`       | Gauge                         | Firmware updates over Wi-Fi: reads `manifest.json`, downloads `gauge.bin` into the spare flash slot, checks it, then restarts into it.                                                                                                                                                                                                                                                                                                    |
| `Gauge_Inputs.h` / `.cpp`    | Gauge                         | The table of readings a custom gauge can show (`oil_psi`, `coolant_c`, `tyre_fl`…): where each sits in the JSON, its units, label and default range. The layout editor uses the same table, so its menus always match the screen.                                                                                                                                                                                                         |
| `Gauge_Layout.h` / `.cpp`    | Gauge                         | The layout engine that draws the **custom views** from the layout editor: arcs, needles, bars, numbers, lights, text, lines, clock, images. Also handles their alerts and animation. The editor's preview runs this same file, so the preview looks the same as the screen.                                                                                                                                                               |
| `Gauge_Layouts.h`            | **Generated** (layout editor) | Custom views, written by the editor's "Export to screen code". It's the layout as text, plus any uploaded images as data. Edit the views in the editor, not here.                                                                                                                                                                                                                                                                         |
| `car_370z.c`                 | Gauge                         | The top-down 370Z picture for the tyre view and the editor's image gauge (116×168, with transparency).                                                                                                                                                                                                                                                                                                                                    |
| `smokinstack_logo.c`         | Gauge                         | The picture on the boot splash screen.                                                                                                                                                                                                                                                                                                                                                                                                    |

## Board drivers (Waveshare)

These talk to the chips on the screen board. The gauge code uses them as they are.

| File | Who | What it's for |
|---|---|---|
| `Display_ST7701.h` / `.cpp` | Waveshare, changed | Starts the round 480×480 panel and drives its backlight. Changed to use one frame buffer and a bigger bounce buffer, which stopped the judder. It also prints display and memory details at boot. |
| `LVGL_Driver.h` / `.cpp` | Waveshare, changed | Connects LVGL (the graphics library) to the panel and the touch screen. Changed to draw in small strips in fast internal memory (partial refresh). |
| `Touch_CST820.h` / `.cpp` | Waveshare, slightly changed | The touch-screen chip. |
| `TCA9554PWR.h` / `.cpp` | Waveshare | The I/O expander chip, which switches the panel's reset and other board pins. |
| `I2C_Driver.h` / `.cpp` | Waveshare | The shared I²C bus the touch, IMU, clock and expander chips sit on. |
| `Gyro_QMI8658.h` / `.cpp` | Waveshare | The motion sensor (IMU), used for the G-force view and the LEVEL mounting aid. |
| `RTC_PCF85063.h` / `.cpp` | Waveshare | The clock chip, used for the clock view. The gauge sets it from the brain's time. |
| `BAT_Driver.h` / `.cpp` | Waveshare | Reads a battery voltage pin. Still runs, but the gauge doesn't use the value. |
| `SD_Card.h` / `.cpp` | Waveshare | SD card support. The card itself is switched off in `setup()`. The file stays because its `Flash_test()` prints the flash size at boot. |
| `Wireless.h` / `.cpp` | Waveshare | Waveshare's Wi-Fi / Bluetooth scan demo. Not used: the gauge's own Wi-Fi is in `Gauge_Net.cpp`. |


---

## How the pieces fit

```
LVGL_Arduino.ino
 ├─ board drivers: Display_ST7701, LVGL_Driver, Touch_CST820, TCA9554PWR,
 │                 I2C_Driver, Gyro_QMI8658, RTC_PCF85063, BAT_Driver, SD_Card
 ├─ Gauge.cpp ──────── built-in views, settings, alerts, swipe order
 │    ├─ Gauge_Config.h        settings and limits
 │    ├─ Gauge_Telemetry       JSON → readings
 │    ├─ Gauge_Layout          draws the custom views
 │    │    ├─ Gauge_Inputs     what a custom gauge can show
 │    │    └─ Gauge_Layouts.h  custom views (from the editor)
 │    └─ car_370z.c, smokinstack_logo.c   pictures
 ├─ Gauge_Net ─────── Wi-Fi + polling the brain (or the bench server)
 └─ Gauge_Ota ─────── firmware updates
```
