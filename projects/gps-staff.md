# GPS Staff — Project Context

This document brings Claude Code up to speed on the gps-staff project. Read this before starting any work in this repo.

**Project name:** GPS Survey Staff
**Repo path:** `/mnt/c/Users/kirkh/github/gps-staff`
**GitHub:** `https://github.com/GM4AJK/gps-staff`
**Living spec:** `sdd/README.md` (full technical detail, component decisions, open questions — this is the canonical reference, not the root README)

---

## What It Is

A DIY RTK GNSS survey staff (pole-mounted instrument) — base + rover pair, identical PCBs. The user plants the base on a known point, configures it from the handheld, then walks with the rover to survey. The handheld (separate ESP32-S3 device) is the only user interface.

---

## Hardware Overview

### PCB Staff Units (×2 — base and rover)

Custom 4-layer PCB (`hardware/rtk/`), ordered from JLCPCB 2026-06-25, tagged `PCB-v1.0`. **Cannot be changed** — any issues are ECOs for v1.1.

| Component | Part | Notes |
|-----------|------|-------|
| MCU | STM32F765VIT6 | 216 MHz Cortex-M7, LQFP100 |
| GNSS | u-blox ZED-F9P-05B | RTK-capable, ordered |
| Radio | Semtech SX1262 | GFSK/LoRa, for RTCM OTA |
| IMU | LSM6DSRXTR (Last Time Buy) or LSM6DSOXTR fallback | SPI |
| Magnetometer | MMC5603NJ | Via sub-board J11 |
| Display | SSD1309 OLED (rover-facing status) | I2C4 (PD12/PD13) |
| ESP32 | Waveshare ESP32-S3 Zero (4MB flash, 2MB PSRAM) | Comms bridge |
| SD card | SDMMC 4-bit + card detect | FatFS, config + log storage |

**Key pin assignments (finalised, F765VIT):**
- F9P: UART5 (PB8/PB9) nav/config; UART7 (PA8/PA15) RTCM corrections
- Mode select: PE2 (GP1, pull-up, open=base), PE3 (GP2, F765 strobe)
- IMU INT1→PD14, INT2→PD15
- I2C4 (PD12/PD13) for IMU/magnetometer/display
- EEPROM removed — config stored on SD card

### Bench Nucleo Boards (dev/test, not shipped)

| Board | Role | Notes |
|-------|------|-------|
| Nucleo-F767ZI | Base station sim | Bench-tethered; cut ST-LINK replaced via CN11/CN12 Morpho pins with external ST-LINK V3 |
| Nucleo-F446RE | Rover sim | Portable on USB battery; no UART when portable — OLED-only UI |

### Handheld

**Waveshare ESP32-S3-Touch-LCD-4.3** — fixed-portrait 480×800 (hardware landscape, LVGL rotated 270°)

- 16MB Flash, 8MB OCT PSRAM, 800×480 RGB LCD, GT911 touch (I2C GPIO8/9)
- Console via CH343P UART port (`/dev/esp32_handheld`, COM13); use `CONFIG_ESP_CONSOLE_UART_DEFAULT`
- Firmware: `firmware/esp32-handheld/`
- Hardware ref: `docs/ESP32-S3-4-3Inch.md`

### ESP32-S3 Zero Staff Modules (×2)

**Waveshare ESP32-S3 Zero** — 4MB flash, 2MB PSRAM

- Single firmware binary: `firmware/esp32-base/`
- Role selected at boot: GP1 open = base, GP1 GND = rover
- Device nodes: `/dev/esp32_base`, `/dev/esp32_rover`
- Console via USB Serial/JTAG; use `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`

---

## Communications Architecture

```
Base F765 ──UART──► Base ESP32 Zero ──ESP-NOW/WiFi──► Rover ESP32 Zero ──UART──► Rover F765
                         │                                     │
                    SX1262 GFSK                           SX1262 GFSK
                    (RTCM TX)                             (RTCM RX)
                                                               │
                                                            BLE (NimBLE)
                                                               │
                                                        Handheld ESP32-S3
```

- **RTCM corrections:** Base → Rover over GFSK (SX1262). One-way stream. Active bench mode.
- **Rover ↔ Handheld:** BLE (NimBLE). Bidirectional — position/status as notifications (pub/sub), config/mode as GATT write + notify. Rover is peripheral/server; handheld is central/client.
- **Base ↔ Rover ESP32:** ESP-NOW (no infrastructure, ~200m LOS). For config relay: handheld → BLE → rover ESP32 → ESP-NOW → base ESP32 → UART → base F765.
- **Handheld → Base:** Routed through rover as relay. Handheld never connects to base directly.

**Two-phase operation:**
1. **Base setup:** Plant base, configure (enter known coords if on control point, or let F9P self-survey), start streaming.
2. **Survey:** Walk with rover + handheld. Base is now a dumb RTCM transmitter. Rover status visible on handheld ("Base: OK" means RTCM is flowing).

---

## Firmware Structure

### STM32 (F767ZI bench / F446RE bench / F765VIT PCB)

Built and flashed by user in **STM32CubeIDE** — do not run make or build commands from the CLI.

```
firmware/
  Nucleo-F767ZI-FreeRTOS/   Base station (bench F767ZI)
  rtk-base/                 (maps to PCB base role, F765VIT)
  rtk-rover/                (maps to PCB rover role, F765VIT)
```

**App structure pattern** (CubeMX well-defined hooks):
- `main.c`: only `#include "app.h"` + calls `app_init(); app_loop();` in USER CODE
- `stm32f7xx_it.c`: `app_1ms()` called from SysTick USER CODE block
- Real code lives in `app.c`/`app.h` and per-module files
- HAL weak callback overrides go in `main.c` USER CODE, forwarding to module subfunctions

**Module pattern:**
- Every module has a `module_t` struct in its header; all state lives there
- Functions take `module_t *p` as first arg; caller holds `static module_t foo;`
- File-scope globals for module state: never

**FreeRTOS task naming:** `task_xxxx.c` / `task_xxxx.h`

**IRQ → idle-loop bridge:** `flags.c`/`flags.h` (proven pre-AI utility). Token-pasting macros `MAKE_H(x)`/`MAKE_C(x)` generate `flag_set_X()`, `flag_get_X()` (consume-once), `flag_peek_X()`. Uses `<stdatomic.h>`. Tick dividers in `app_1ms()` produce `flag_set_10MS`, `flag_set_100MS`, `flag_set_1000MS`.

**HAL vs registers:** HAL is fine. The rule is hand-written code vs CubeMX scaffolding — the user decides per peripheral. No CubeMX abstraction layer (BSP) — stay close to HAL.

### ESP32 (all devices)

Built and flashed by Claude via `scripts/idf.sh` (no sourcing needed — sets IDF env internally):

```bash
cd firmware/esp32-handheld && /mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh build
/mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh -p /dev/esp32_handheld flash
```

IDF version: **6.2**. Key IDF 6 migrations already done:
- `driver/i2c.h` → `driver/i2c_master.h` (new master bus API)
- RGB panel: `bits_per_pixel`, `sram_trans_align`, `psram_trans_align` removed; `on_bounce_frame_finish` → `on_vsync`
- Touch: `esp_lcd_touch_get_coordinates` → `esp_lcd_touch_get_data` + `esp_lcd_touch_point_data_t`
- Component `esp_log` not needed in REQUIRES (auto-linked)

`sdkconfig` is not tracked in git. Regenerates from `sdkconfig.defaults`. Delete `sdkconfig` and rebuild whenever `sdkconfig.defaults` changes.

---

## LVGL Widget Pattern (esp32-handheld)

**One widget = one `.c` / `.h` pair.** `main.c` stays thin (init only).

State goes in a struct defined in the header:
```c
typedef struct {
    lv_obj_t *body;
    lv_obj_t *fill;
    int       value;
} widget_foo_t;
```

API is namespaced and struct-based:
```c
void widget_foo_init(lv_obj_t *parent, widget_foo_t *w);
// Add widget_foo_dtor() only when something actually needs to destroy it
```

Timer callbacks get their context via `user_data`:
```c
lv_timer_create(cb, interval_ms, w);   // pass struct ptr
// in cb:
widget_foo_t *w = (widget_foo_t *)timer->user_data;
```

`main.c` pattern:
```c
static widget_foo_t foo;

void app_main(void) {
    ESP_ERROR_CHECK(waveshare_esp32_s3_rgb_lcd_init());
    if (lvgl_port_lock(-1)) {
        lv_obj_t *scr = lv_scr_act();
        lv_obj_set_style_bg_color(scr, lv_color_black(), 0);
        widget_foo_init(scr, &foo);
        lvgl_port_unlock();
    }
}
```

LVGL runs on core 1 with tear-avoidance mode 3. Lock/unlock all LVGL calls with `lvgl_port_lock(-1)` / `lvgl_port_unlock()`. Timer callbacks run inside the LVGL task — no extra mutex needed there.

---

## Coding Style (all hand-written C)

- **Tabs** for indentation; CubeMX-generated files left as-is
- HAL callbacks (`HAL_*`) override in `main.c` USER CODE, forwarding to module subfunction
- FreeRTOS task files named `task_xxxx.c/.h`
- Author header on non-CubeMX files: `/* \n * Author: Andy Kirkham\n */`
- No comments explaining what code does; only comment non-obvious WHY
- API completeness tracks actual need — add destructor/extra functions only when needed

---

## GFSK / LoRa (SX1262)

Active bench mode is **GFSK** (~7–8× less airtime than LoRa for 100m LOS link). Interrupt-driven DIO1 for TX/RX done. For the `Core1262-LF` module, `SetDIO3AsTCXOCtrl` + `CalibrateImage` still needed.

RTCM3 framing: `RTCM3_BUF_COUNT=4` (bump to 8 before adding display driver on PCB — display flush ~33ms eats half the overflow budget).

Sample RTCM3 data for OTA dev: `firmware/data/CambridgeSensoriisSample.bin` (39 frames, GPS+GLONASS MSM4).

---

## Current Status (as of 2026-06-26)

| Area | Status |
|------|--------|
| PCB v1.0 | Ordered JLCPCB 2026-06-25, awaiting delivery |
| ZED-F9P | Ordered DigiKey 2026-06-25 |
| BLE serial bridge (Zero boards) | Bench verified PR #187 |
| GFSK RTCM OTA | Active bench mode, interrupt-driven |
| Handheld display + touch | Working, portrait 270°, LVGL 8.4 |
| Handheld UI | Early sandbox — battery indicator widget only |
| MMC5603NJ driver | Needed (replaced LIS3MDL which is no longer purchasable) |
| Tilt fusion / soft-iron cal | Deferred; `float soft_iron[3][3]` reserved in config struct |
| FatFS RTC timestamp | Deferred until real F9P arrives |
| F765 PE2/PE3 GPIO (mode select) | Not yet implemented, waiting for PCB |

---

## GitHub Workflow

- SSH broken from WSL — use HTTPS + `gh` PAT. See `CLAUDE.md` in the repo.
- **PR-only ruleset** — no direct pushes to `main`. Always branch → commit → `gh pr create` → `gh pr merge`.
- Datasheets: `docs/datasheets/` — update `docs/README.md` catalog in same PR.
