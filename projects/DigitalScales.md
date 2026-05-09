# DigitalScales — Project Context

This document brings Claude Code up to speed on the DigitalScales project. Read this before starting any work on this codebase.

**Project name:** DigitalScales
**Repo path:** `/mnt/c/Users/kirkh/github/digiscales/firmware/DigitalScales`
**IDE:** STM32CubeIDE (build and flash done by user — Claude edits source only)

---

## Developer Profile

- Electrical and software engineer, UK
- Tooling: STM32CubeIDE, HAL (acceptable for this project), STLINK
- Bench: Siglent SDS824X HD oscilloscope (SCPI 192.168.0.87:5025), Feeltech FY6800 signal generator (/dev/ttyUSB0)
- Lab references: `lab/SDS824X.md`, `lab/FY6800.md` in this repo

---

## Project Goal

A digital kitchen/postal scale built from scratch. STM32G431KB reads an HX711 load cell ADC and displays the weight on a 128×64 OLED. Target display range: 00.0 – 99.9 (units TBD).

---

## Hardware

| Item | Detail |
|------|--------|
| MCU | STM32G431KB (Nucleo-32 board), 170 MHz Cortex-M4 |
| Display | 128×64 OLED over I2C1 — **SH1106 controller** (sold as SSD1306-compatible) |
| Load cell ADC | HX711F — CLK on PA4, DATA on PA5 |
| Timer | TIM6 — 1 ms software tick (prescaler 170, period 1000) |
| Clock | 170 MHz from HSI via PLL (VOLTAGE_SCALE1_BOOST) |

### HX711 pin assignment (from main.h)
```c
#define HX711_CLK_Pin       GPIO_PIN_4
#define HX711_CLK_GPIO_Port GPIOA
#define HX711_DT_Pin        GPIO_PIN_5
#define HX711_DT_GPIO_Port  GPIOA
```

Datasheet: `datasheets/hx711F.pdf` in the project repo.

---

## Project Structure

```
Core/
  Src/
    main.c          HAL init (CubeMX-generated) — calls app_init() then app_loop()
    app.c           Application logic — init and main loop
    ssd1306.c       OLED driver (page addressing, I2C)
    fonts.c         Standard fonts: Font_7x10, Font_11x18, Font_16x26
    fontx.c         Large 7-segment font for scale readout
    hx711.c         HX711 load cell ADC driver
  Inc/
    main.h          GPIO pin definitions
    app.h
    ssd1306.h
    fonts.h
    fontx.h
    hx711.h
Drivers/            STM32 HAL + CMSIS — CubeMX-managed, do not edit
datasheets/
  hx711F.pdf
DigitalScales.ioc   CubeMX project file
```

Do not edit files inside `Drivers/`. User code belongs in `/* USER CODE BEGIN/END */` blocks in CubeMX files, or in `app.c`, `fontx.c`, `hx711.c` etc.

---

## Display Driver — SH1106 Gotcha

The OLED module is an **SH1106** controller, not a true SSD1306, despite the labelling.

**Key difference:** SH1106 has 132 columns of RAM; only columns 2–129 are mapped to the visible 128 pixels. Writing from column 0 shifts the image left by 2 pixels and leaves 2 columns of random RAM content visible on the right edge.

**Fix applied in `ssd1306_UpdateScreen` (ssd1306.c):**
```c
ssd1306_WriteCommand(hi2c, 0x02);  // SH1106: visible cols start at RAM col 2
ssd1306_WriteCommand(hi2c, 0x10);
```
**Do not revert this to 0x00.** SH1106 only supports page addressing mode — do not attempt horizontal or vertical addressing mode.

---

## fontx — Large Scale Readout Font

`fontx.c` / `fontx.h` — a custom 30×56 pixel 7-segment style bitmap font.

- **Supported characters:** `'.'` (ASCII 46) through `'9'` (ASCII 57) only
- **Cell size:** 30×56 px — four characters ("XX.X") = 120 px, centred on 128 px display
- **Internal margin:** 2 px blank on all four sides; glyph occupies cols 2–27, rows 2–53
- **Bit layout:** `uint32_t` per row, MSB = leftmost pixel (col 0), bits 1..0 always 0

**Segment masks:**
```
SEG_H = 0x03FFFF00   horizontal (cols 6-23)
SEG_L = 0x3C000000   left vertical (cols 2-5)
SEG_R = 0x000000F0   right vertical (cols 24-27)
```

**API:**
```c
uint8_t fontx_DrawChar(char ch, uint8_t x, uint8_t y, SSD1306_COLOR color);
void    fontx_DrawString(const char *str, uint8_t x, uint8_t y, SSD1306_COLOR color);
```

Both write to the screen buffer only — call `ssd1306_UpdateScreen(&hi2c1)` afterwards.

**Centred usage:**
```c
// FONTX_CENTER_X = 4, FONTX_CENTER_Y = 4
fontx_DrawString("12.3", FONTX_CENTER_X, FONTX_CENTER_Y, White);
ssd1306_UpdateScreen(&hi2c1);
```

---

## HX711 Driver

`hx711.c` / `hx711.h` — bit-bang driver using DWT cycle counter for timing.

### Gain enum
```c
typedef enum {
    HX711_GAIN_A_128 = 25,  // Channel A, gain 128 — default for load cells
    HX711_GAIN_B_32  = 26,  // Channel B, gain 32
    HX711_GAIN_A_64  = 27,  // Channel A, gain 64
} HX711_Gain;
```

The gain enum value equals the total PD_SCK pulse count per conversion (datasheet Table 3). The channel/gain for the NEXT conversion is programmed by the pulse count at the end of each read.

### API
```c
HAL_StatusTypeDef hx711_init(HX711_t *hx, HX711_Gain gain);
uint8_t           hx711_is_ready(HX711_t *hx);         // 1 = DOUT low = ready
HAL_StatusTypeDef hx711_get_data(HX711_t *hx, int32_t *outval);  // HAL_BUSY if not ready
void              hx711_power_down(HX711_t *hx);
void              hx711_power_up(HX711_t *hx);
```

### Timing
- 1 µs delays via DWT cycle counter (`CoreDebug->DEMCR`, `DWT->CTRL`, `DWT->CYCCNT`)
- DWT enabled once in `hx711_init()` — do not reset CYCCNT elsewhere
- Data sampled after PD_SCK falling edge (maximum setup margin)
- Power-down: PD_SCK high for >60 µs (HAL_Delay(1) used — 1 ms >> 60 µs threshold)

### Initialisation in app.c
```c
hx711.CLK_GPIO  = HX711_CLK_GPIO_Port;
hx711.CLK_Pin   = HX711_CLK_Pin;
hx711.DATA_GPIO = HX711_DT_GPIO_Port;
hx711.DATA_Pin  = HX711_DT_Pin;
hx711_init(&hx711, HX711_GAIN_A_128);
```

`hx711_init` waits up to 500 ms for the first conversion (settling time after power-up is up to 400 ms at 10 SPS), then performs one dummy read to programme the gain.

### Critical gotcha — PD_SCK must not be toggled between reads
The HX711 enters power-down if PD_SCK is held high for >60 µs. Never toggle or drive CLK high outside of the driver. The original prototype `app_loop` toggled CLK every 1 ms — this was removed when the driver was introduced.

### Return values
- `hx711_get_data` returns `HAL_OK` at most once per conversion period (~100 ms at 10 SPS)
- Returns `HAL_BUSY` while DOUT is high (conversion in progress) — this is normal
- Raw value is 24-bit 2's complement, sign-extended to `int32_t`
- At 10 SPS a new reading is available approximately every 100 ms

---

## app.c — Current Structure

```c
void app_init(void) {
    // Init OLED, init HX711, start TIM6
}

void app_loop(void) {
    for(;;) {
        int32_t raw;
        if (hx711_get_data(&hx711, &raw) == HAL_OK) {
            // convert raw → weight, format with sprintf("%04.1f"), draw with fontx
            // ssd1306_UpdateScreen(&hi2c1)
        }
    }
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim == &htim6) ++millisecond_counter;
}
```

`millisecond_counter` is a `volatile static uint32_t` incremented by TIM6 ISR.

### sprintf float formatting
```c
sprintf(buffer, "%04.1f", value);  // "03.5", "99.9" etc.
```
If output is blank or "?", enable float in linker: **Properties → C/C++ Build → Settings → MCU GCC Linker → Miscellaneous**, add `-u _printf_float`.

---

## Current Status (as of 2026-05-09)

- OLED display working with fontx large font ✓
- SH1106 column offset fixed ✓
- HX711 driver implemented and tested (one read confirmed) ✓
- **Load cell not yet connected** — HX711 currently reading floating inputs
- Calibration (tare offset + scale factor) not yet implemented
- Next steps: connect load cell, implement tare/calibration, display real weight

---

## Coding Style

- HAL is acceptable throughout (unlike ESC/bare-metal projects)
- Tabs for indentation
- Braces on all control flow, body on its own line
- `/* USER CODE BEGIN/END */` blocks for anything in CubeMX-generated files
