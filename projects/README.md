# Projects

Only read when asked specifically about a project.

## CLAUDEDEMO.md 

Created to demostrate Claude building a digitall IIR filter on an STM32G431 and then taking control of hardware (network connected signal generator and oscilloscope) for perform bode plots by writing Python scripts to perform verification.

## TwinTorqueESC.md

The dual ESC (electronic speed controller) developed using the STM32G431KBT6

## DigitalScales.md

STM32G431KB digital scale — HX711 load cell ADC, SH1106 128×64 OLED display, custom large 7-segment font. Covers the SH1106 column-offset fix, fontx font design, and HX711 bit-bang driver.

## gps-staff.md + gps-staff/

DIY RTK GNSS survey staff — base + rover PCBs (STM32F765VIT + ZED-F9P + SX1262), ESP32-S3 Zero comms bridges, ESP32-S3 4.3" handheld controller. `gps-staff.md` is the overview/status index. Sub-files in `gps-staff/` cover focused areas:

| File | Contents |
|------|----------|
| `gps-staff/handheld.md` | LVGL 8.4, widget pattern, IDF 6 build, display setup |
| `gps-staff/stm32-firmware.md` | F765/F767/F446 conventions, CubeMX hooks, FreeRTOS, flags |
| `gps-staff/hardware.md` | PCB v1.0 BOM, pin assignments, all boards |
| `gps-staff/comms.md` | GFSK RTCM, BLE NimBLE, ESP-NOW, two-phase operation |

