# GPS Staff — STM32 Firmware Conventions

## Build & Flash

Built and flashed by the **user in STM32CubeIDE**. Do not run `make` or any build commands from the CLI. Claude edits source; user builds and flashes, then pastes serial output if needed.

Serial console output is viewed via VS Code "Web and Desktop Serial Monitor" extension (tabbed, all four boards monitored simultaneously). WSL cannot read COM ports directly.

## Firmware Directories

```
firmware/
  Nucleo-F767ZI-FreeRTOS/   Bench base (F767ZI, FreeRTOS)
  rtk-base/                  PCB base role target (F765VIT)
  rtk-rover/                 PCB rover role target (F765VIT)
```

## CubeMX Well-Defined Hooks Pattern

Minimise hand-edits inside CubeMX-generated files to survive regeneration.

**`main.c`** (CubeMX-owned) — only these additions in USER CODE blocks:
```c
// USER CODE BEGIN Includes
#include "app.h"
// USER CODE END Includes

// USER CODE BEGIN 2  (after all MX_*_Init calls)
app_init();
app_loop();   // never returns
// USER CODE END 2
```

**`stm32f7xx_it.c`** (CubeMX-owned) — SysTick hook:
```c
// USER CODE BEGIN SysTick_IRQn 1
app_1ms();
// USER CODE END SysTick_IRQn 1
```

**`main.h`** (CubeMX-owned) — extern peripheral handles in USER CODE EC block:
```c
/* USER CODE BEGIN EC */
extern UART_HandleTypeDef huart1;
extern SPI_HandleTypeDef  hspi1;
// ... etc for all peripherals used by app code
/* USER CODE END EC */
```

Real application code lives in `app.c` / `app.h` and per-module files. Never in CubeMX files beyond the hooks above.

## HAL Callback Overrides

HAL weak callbacks go in `main.c` USER CODE, forwarding to a subfunction in the relevant module:

```c
// main.c USER CODE
void HAL_SD_ErrorCallback(SD_HandleTypeDef *hsd) {
    sdcard_on_hal_error(hsd);
}
```

The subfunction is declared in the module header and defined in the module `.c`. This keeps the interrupt topology readable at a glance in one place.

## Module Pattern

Every module has a `module_t` struct defined in its header. All state lives in the struct — no file-scope globals for module state.

```c
// foo.h
typedef struct {
    // all state here
    uint8_t  buf[256];
    uint32_t idx;
    bool     ready;
} foo_t;

void foo_init(foo_t *p);
void foo_loop(foo_t *p);
```

Caller holds a `static foo_t foo;` instance and passes `&foo`. Functions take `module_t *p` as first argument.

`_loop()` functions open with an explicit guard on the ready condition — e.g. `if (!p->ready) return;` as the very first line.

## FreeRTOS Task Naming

Files implementing a FreeRTOS task must be named `task_xxxx.c` / `task_xxxx.h`.

## IRQ → Idle-Loop Bridge (`flags.c` / `flags.h`)

Proven pre-AI utility. Token-pasting macros generate three functions per flag:

```c
// flags.h — declare a flag
MAKE_H(10MS)   // generates flag_set_10MS(), flag_get_10MS(), flag_peek_10MS()

// flags.c — define it
MAKE_C(10MS)
```

- `flag_set_X()` — atomically OR the bit (call from ISR context)
- `flag_get_X()` — consume-once: reads the bit and atomically clears it (call from idle loop)
- `flag_peek_X()` — read-only, non-consuming

Uses `<stdatomic.h>`. In `app_1ms()` (SysTick hook), `COUNTER_TIMER` macros divide the 1ms tick into `flag_set_10MS` / `flag_set_100MS` / `flag_set_1000MS`. `app_loop()` calls `flag_get_*()` to act on each.

Keep the flags enum tight — only add entries for flags actually wired to a real IRQ source.

## HAL vs Registers

HAL is fine throughout. The rule is hand-written code vs CubeMX scaffolding — the user decides per peripheral. No CubeMX BSP abstraction layer — stay close to HAL.

## Coding Style

- **Tabs** for indentation; CubeMX-generated files left as-is (they use spaces)
- Author header on non-CubeMX hand-written files: `/*\n * Author: Andy Kirkham\n */`
- No comments explaining what code does; only comment non-obvious WHY
- API completeness tracks actual need — add functions only when something needs them
- `app_log()` for debug output (UART). On F446RE portable (USB battery), no UART — OLED only.

## Bench Board Roles

- **Nucleo-F767ZI = Base.** Bench-tethered (cut ST-LINK replaced via CN11/CN12 Morpho pins + external ST-LINK V3 + bench PSU). Fixed to bench — fine for a base station.
- **Nucleo-F446RE = Rover.** Can run from USB battery pack. When portable: no UART console — all diagnostic UI goes to SSD1309 OLED.

## Key Drivers (bench-verified)

| Driver | Status | Notes |
|--------|--------|-------|
| SX1262 GFSK/LoRa | Active, interrupt-driven DIO1 | GFSK active bench mode; `SetDIO3AsTCXOCtrl` + `CalibrateImage` still needed for Core1262-LF |
| LSM6DSOX (IMU) | Merged PR #124 | I2C1 on F446RE bench |
| MMC5603NJ (mag) | Driver needed | Replaced discontinued LIS3MDL |
| FatFS SD card | Working | F767ZI bench-proven; `FF_FS_NORTC` set — wire GPS UTC when F9P arrives |
| SSD1309 OLED | Working | I2C, rover status display |

## RTCM3 / OTA

Sample data: `firmware/data/CambridgeSensoriisSample.bin` (39 frames, GPS+GLONASS MSM4) for OTA protocol dev without real F9P.

`RTCM3_BUF_COUNT=4` now — bump to 8 before adding display driver on PCB (display flush ~33ms eats half the overflow budget).
