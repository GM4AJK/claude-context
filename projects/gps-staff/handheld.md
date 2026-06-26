# GPS Staff — Handheld (ESP32-S3 4.3" Touch Display)

## Hardware

**Waveshare ESP32-S3-Touch-LCD-4.3**
- 16MB Flash, 8MB OCT PSRAM, 800×480 RGB LCD, GT911 touch (I2C GPIO8/9, addr 0x5D)
- Two USB-C ports: **UART** (CH343P, `/dev/esp32_handheld`, COM13) for flash/console; USB (OTG GPIO19/20) unused
- Hardware reference: `docs/ESP32-S3-4-3Inch.md` in the gps-staff repo

## Display Setup

Fixed portrait 480×800. Hardware stays landscape; LVGL rotated once at init:
```c
lv_display_set_rotation(disp, LV_DISPLAY_ROTATION_270);  // board mounted inverted — bench verified
```
0,0 is top-left in portrait. No runtime rotation. All UI code works in 480×800 coordinates.

LVGL 8.4.0, tear-avoidance mode 3 (double-buffer + direct mode), LVGL task on core 1.

## Build & Flash

Claude owns ESP32 builds. Use `scripts/idf.sh` from the repo root (no sourcing needed):

```bash
cd firmware/esp32-handheld && /mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh build
/mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh -p /dev/esp32_handheld flash
```

IDF version: **6.2**. `sdkconfig` is not tracked — regenerates from `sdkconfig.defaults`. Delete `sdkconfig` and rebuild whenever `sdkconfig.defaults` changes.

**sdkconfig.defaults** key settings:
```
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESP_CONSOLE_UART_DEFAULT=y      # CH343P UART port, NOT USB_SERIAL_JTAG
CONFIG_EXAMPLE_LVGL_PORT_TASK_CORE=1
CONFIG_EXAMPLE_LVGL_PORT_AVOID_TEAR_ENABLE=y
CONFIG_EXAMPLE_LVGL_PORT_AVOID_TEAR_MODE_3=y
CONFIG_EXAMPLE_LVGL_PORT_ROTATION_270=y
CONFIG_LV_FONT_MONTSERRAT_24=y
CONFIG_LV_FONT_MONTSERRAT_32=y
```

## Source Layout

```
firmware/esp32-handheld/main/
  main.c                    — thin: init board, lock LVGL, create widgets, unlock
  waveshare_rgb_lcd_port.c/h — board init (LCD + touch), pin defs
  lvgl_port.c/h             — LVGL tick/task, lvgl_port_lock / lvgl_port_unlock
  batt_indicator.c/h        — battery widget (first real widget, reference example)
```

## LVGL Widget Pattern

**One widget = one `.c` / `.h` pair.** `main.c` stays thin — init only, no widget logic.

### Header — struct + API declaration
```c
// widget_foo.h
#pragma once
#include "waveshare_rgb_lcd_port.h"

typedef struct {
    lv_obj_t *parent;
    lv_obj_t *body;
    lv_obj_t *fill;
    int       value;
} widget_foo_t;

void widget_foo_init(lv_obj_t *parent, widget_foo_t *w);
// Add widget_foo_dtor() only when something actually needs to destroy it
```

### Implementation — state via user_data, no file-scope globals
```c
// widget_foo.c
static void update_cb(lv_timer_t *timer)
{
    widget_foo_t *w = (widget_foo_t *)timer->user_data;
    // mutate w->* here
}

void widget_foo_init(lv_obj_t *parent, widget_foo_t *w)
{
    w->parent = parent;
    w->value  = 0;
    w->body   = lv_obj_create(parent);
    // ... build widget tree ...
    lv_timer_create(update_cb, 600, w);   // pass struct as user_data
}
```

### main.c
```c
#include "waveshare_rgb_lcd_port.h"
#include "widget_foo.h"

static widget_foo_t foo;

void app_main(void)
{
    ESP_ERROR_CHECK(waveshare_esp32_s3_rgb_lcd_init());
    if (lvgl_port_lock(-1)) {
        lv_obj_t *scr = lv_scr_act();
        lv_obj_set_style_bg_color(scr, lv_color_black(), 0);
        widget_foo_init(scr, &foo);
        lvgl_port_unlock();
    }
}
```

**Rules:**
- Timer callbacks run inside the LVGL task — no extra mutex needed in callbacks
- All LVGL calls outside a callback must be wrapped in `lvgl_port_lock` / `lvgl_port_unlock`
- API completeness tracks actual need — add dtor/extra functions only when something needs them
- No file-scope globals for widget state — it all lives in the struct

## Coding Style

- Tabs for indentation
- No comments explaining what code does; only comment non-obvious WHY
- `LV_SYMBOL_*` built-in symbols available (FontAwesome 4 subset) — `LV_SYMBOL_CHARGE` is a lightning bolt
- Author header on hand-written files: `/*\n * Author: Andy Kirkham\n */`

## VS Code IntelliSense

`firmware/esp32-handheld/.vscode/c_cpp_properties.json` points at `build/compile_commands.json`. Red squiggles return if `build/` is deleted — rebuild to fix.

## IDF 6.2 Gotchas (already fixed, for reference)

- `driver` component split: use `esp_driver_i2c`, `esp_driver_gpio` in CMakeLists REQUIRES — not `driver`
- `esp_log` not needed in REQUIRES (auto-linked)
- RGB panel struct: `bits_per_pixel`, `sram_trans_align`, `psram_trans_align` fields removed
- `on_bounce_frame_finish` callback → `on_vsync`
- Touch: `esp_lcd_touch_get_coordinates` deprecated → `esp_lcd_touch_get_data(tp, &point_data, &cnt, 1)`
- New I2C master API: `i2c_new_master_bus()`, `i2c_master_bus_add_device()`, `i2c_master_transmit()`
