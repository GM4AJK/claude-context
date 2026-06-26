# GPS Staff — Hardware & PCB

## PCB Staff Unit (×2 — base and rover, identical)

Custom 4-layer PCB. Ordered JLCPCB 2026-06-25, tagged `PCB-v1.0`. **Cannot be changed** — any issues are ECOs for v1.1. Gerbers: `hardware/rtk/rtk/gerbers_v1_0/`.

**Stackup:** JLC04161H-7628 — L2=GND, L3=3.3V. 50Ω microstrip on L1 over L2 for RF traces.

### Bill of Materials

| Component | Part | Notes |
|-----------|------|-------|
| MCU | STM32F765VIT6 | 216 MHz Cortex-M7, LQFP100 |
| GNSS | u-blox ZED-F9P-05B | RTK-capable, dual-band, ordered DigiKey 2026-06-25 |
| Radio | Semtech SX1262 (Core1262-LF module) | GFSK/LoRa OTA |
| IMU | LSM6DSRXTR | **Last Time Buy** at DigiKey — LSM6DSOXTR is pin-compatible fallback |
| Magnetometer | MMC5603NJ | Via sub-board J11 (no interrupt) |
| OLED | SSD1309 128×64 | Rover-facing status display |
| ESP32 bridge | Waveshare ESP32-S3 Zero | 4MB flash, 2MB PSRAM — comms bridge |
| SD card | SDMMC 4-bit + card detect | FatFS, config + log storage |
| USB PHY | USB3300 ULPI | HS USB, both builds |
| ESD | USBLC6-4SC6 | Replaced out-of-stock USBLC6-2SC6 (pin-compatible) |
| Bias-T choke | LQW15AN8N0G8ZD 8nH wire-wound | RF choke in bias-T — **never swap for ferrite bead** |

### F765VIT Pin Assignments (finalised)

| Signal | Pin | Notes |
|--------|-----|-------|
| F9P nav/config | UART5 PB8/PB9 | |
| F9P RTCM corrections | UART7 PA8/PA15 | |
| F9P D_SEL | PE14 | Default high = UART mode |
| Mode select input | PE2 (GP1) | Pull-up; open=base, GND=rover |
| Mode valid strobe | PE3 (GP2) | F765 output to ESP32 |
| IMU INT1 | PD14 | LSM6DSR |
| IMU INT2 | PD15 | LSM6DSR |
| I2C bus (IMU/mag/display) | I2C4 PD12/PD13 | |
| SX1262 SPI | (see schematic) | |
| SDMMC | (4-bit, see schematic) | |

EEPROM removed — config stored on SD card file.

### Bias-T Detail

VCC_RF (module output) drives bias-T. L2 (8nH wire-wound) is the RF choke. **Never swap the RF choke inductor for a ferrite bead** — a ferrite bead is a resistive/lossy element, not an RF choke; it would attenuate the RF signal.

---

## Bench Nucleo Boards (dev/test only, not shipped)

### Nucleo-F767ZI (Base sim)

On-board ST-LINK is physically cut. Replacement wiring via Morpho connectors:
- CN11: PA13 (SWDIO), PA14 (SWCLK), PD9 (UART RX from ST-LINK)
- CN12: PD8 (UART TX to ST-LINK)
- External ST-LINK V3 + bench PSU required

### Nucleo-F446RE (Rover sim)

Portable — runs from USB battery pack. No UART console when portable; diagnostic UI via SSD1309 OLED only.

---

## Handheld

**Waveshare ESP32-S3-Touch-LCD-4.3**

| Spec | Value |
|------|-------|
| Flash | 16MB |
| PSRAM | 8MB OCT |
| Display | 800×480 RGB LCD, GT911 touch |
| I2C (touch) | GPIO8 (SDA), GPIO9 (SCL), addr 0x5D |
| Console | CH343P UART (`/dev/esp32_handheld`, COM13) |
| Orientation | Fixed portrait — LVGL 270° rotation, 480×800 in software |

Hardware reference: `docs/ESP32-S3-4-3Inch.md` in the gps-staff repo.

---

## ESP32-S3 Zero Staff Modules (×2)

**Waveshare ESP32-S3 Zero**

| Spec | Value |
|------|-------|
| Flash | 4MB |
| PSRAM | 2MB |
| WS2812 LED | GPIO21 |
| UART0 | GPIO43 (TX) / GPIO44 (RX) — connects to F765 |
| GP1–GP5 | Wired to F765 PE2–PE6 |
| Console | USB Serial/JTAG (use `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`) |
| Device nodes | `/dev/esp32_base`, `/dev/esp32_rover` |

Role at boot: GP1 open = base, GP1 GND = rover. Bench: hardwired GP1 link. PCB: F765 PE3 strobe after 500ms timeout.

Single firmware binary: `firmware/esp32-base/`. BLE + WiFi together ~70% flash — 30% for app code, no constraint.

---

## Spare

**Waveshare ESP32-S3-Touch-AMOLED-1.8** — acquired because "it looked cute". Role TBD.
